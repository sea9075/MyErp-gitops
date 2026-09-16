# MyERP-gitops

MyERP 專案的 GitOps 部署倉庫（Helm chart，路徑 `myerp/`）。ArgoCD 監看這個 repo，
一有變更就自動 sync 到叢集；`MyERP` repo 的 GitHub Actions 只會自動更新
`myerp/values.yaml` 裡的 `image.api.tag` / `image.worker.tag` 兩個欄位，其餘設定都是
一次性的環境設定，改了才需要手動 commit + push。

## 這個 repo 管什麼、不管什麼

管：`myerp-api` / `myerp-worker` 的 Deployment、Service、HPA、KEDA ScaledObject、
外部路由（HTTPRoute）、ExternalSecret / ClusterSecretStore。

不管：Gateway（`myerp-gateway`）、Cloudflare Tunnel、Cilium/cert-manager/ArgoCD/KEDA/
External Secrets Operator 本身的安裝——這些是叢集平台層，已經用 `helm install` 手動裝好
（見 `MyERP` repo 的 `Infra-Progress.md`）。

## 叢集端一次性設定（部署前必須手動做，不會進 git）

以下步驟只需要做一次，之後 CI/CD 就會自動運作，不用每次部署都重做。

### 1. 建立 imagePullSecret（讓叢集能從 ACR 拉私有映像）

```bash
kubectl create namespace myerp   # 如果還沒建立
kubectl create secret docker-registry myerp-acr-credentials \
  --namespace=myerp \
  --docker-server=myerpacr.azurecr.io \
  --docker-username=<ACR Admin username> \
  --docker-password=<ACR Admin password>
```

帳密位置：Azure Portal → `myerpacr` → 存取金鑰。跟 GitHub Secrets 裡的
`ACR_USERNAME`/`ACR_PASSWORD` 是同一組。

### 2. 安裝 metrics-server（HPA 依 CPU/記憶體縮放的前提）

```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update
helm install metrics-server metrics-server/metrics-server -n kube-system
```

裸機 kubeadm 叢集常見一個坑：metrics-server 預設用 kubelet 憑證做 TLS 驗證，
自簽憑證環境下會抓不到 metrics（`kubectl top nodes` 顯示不出數字）。如果裝完發現
`kubectl top nodes` 沒有資料，補一個參數：

```bash
helm upgrade metrics-server metrics-server/metrics-server -n kube-system \
  --set args={--kubelet-insecure-tls}
```

### 3. 安裝 External Secrets Operator（ESO）

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace
```

### 4. 建立 Azure Service Principal，給 Key Vault 讀取權限

```bash
az ad sp create-for-rbac --name myerp-eso-sp --skip-assignment
```

記下輸出的 `appId`（= Client ID）、`password`（= Client Secret）、`tenant`。接著到
Azure Portal → `myerp-kv` → 存取控制 (IAM) → 新增角色指派，把 **Key Vault Secrets User**
這個角色指派給剛建立的 `myerp-eso-sp`（唯讀，符合最小權限原則——ESO 只需要讀，不需要寫）。

### 5. 把 Service Principal 憑證存成 K8s Secret（整條機密鏈唯一手動建立、不進 git 的機密）

```bash
kubectl create secret generic azure-sp-credentials \
  --namespace=myerp \
  --from-literal=ClientID=<appId> \
  --from-literal=ClientSecret=<password>
```

這是所謂「雞生蛋問題」的解法：ESO 要連 Key Vault 才能把其他機密（DB 連線字串、JWT Key、
Service Bus 連線字串）動態同步進叢集，但 ESO 自己連 Key Vault 用的這組認證，沒辦法又
從 Key Vault 拉——只能手動建一次，之後就不用再管了。

### 6. 把應用程式機密實際存進 Azure Key Vault

```bash
az keyvault secret set --vault-name myerp-kv --name SqlConnectionString --value "<Azure SQL 連線字串>"
az keyvault secret set --vault-name myerp-kv --name JwtSigningKey --value "<Jwt Key>"
az keyvault secret set --vault-name myerp-kv --name ServiceBusConnectionString --value "<Service Bus 連線字串>"
```

（這三個 secret 名稱要跟 `myerp/values.yaml` 裡 `externalSecrets.mappings` 的
`remoteKey` 完全一致，改名字要兩邊一起改。）

### 7. 填入 `myerp/values.yaml` 裡的 `azureKeyVault.tenantId`

Azure Portal → Microsoft Entra ID → 概觀 → 租用戶 ID，或執行 `az account show --query tenantId -o tsv`，
填入後 commit + push 這個 repo。

### 8. 設定 ArgoCD Application（讓 ArgoCD 開始監看這個 repo）

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myerp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/sea9075/MyERP-gitops.git
    targetRevision: main
    path: myerp
  destination:
    server: https://kubernetes.default.svc
    namespace: myerp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

存成檔案後 `kubectl apply -f <檔名> -n argocd`，或直接在 ArgoCD 網頁介面手動建立同樣設定
的 Application。`prune: true` 代表之後如果從這個 repo 刪掉某個資源，ArgoCD 會自動把叢集上
對應的資源也刪掉，`selfHeal: true` 代表有人手動改了叢集上的資源，ArgoCD 會自動改回 git 裡
的版本——這兩個是 GitOps「git 是唯一事實來源」精神的關鍵設定。

### 9. 部署成功、確認外部路由正常後，清掉測試用的 placeholder 資源

```bash
kubectl delete httproute placeholder-route -n myerp
kubectl delete deployment placeholder -n myerp
kubectl delete service placeholder -n myerp
```

## 為什麼用 Service Principal，不是 Workload Identity

Hybrid-Cloud.md 原規劃是等 Azure Arc onboarding（叢集註冊進 Azure Resource Manager）
定案後，用 Workload Identity Federation 讓 ESO 免帳密連 Key Vault。但查證後確認 ESO 連
Key Vault 這件事，跟 Azure Arc onboarding 完全是兩回事、互不依賴，用 Service Principal
現在就能做，不需要卡在 Azure Arc 認證方式定案這個更大、還沒排期的前置工作上。之後如果真的
完成 Azure Arc onboarding，要升級成 Workload Identity 只需要改 `secretstore.yaml` 的
`authType` 跟拿掉 `authSecretRef`，其他資源（ExternalSecret、Deployment）完全不用動。
