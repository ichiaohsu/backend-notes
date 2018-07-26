## 安裝 helm
`helm` 會在 k8s cluster 中安裝一個 `tiller` server 來跟遠端 chart 倉庫溝通，並在本地建立對應的 k8s deployment，service。為此 `tiller` 必須擁有 k8s api-server 的權限。預設是完全開放，但建議使用 RBAC 的方式來賦予 `helm` 權限。

### 建立 RBAC

先建立 `serviceaccount`。預設會把 tiller server 安裝在 `kube-system` 的命名空間中。所以我們也把 `serviceaccount` 建立在 `kube-system` 裡。
```bash
kubectl -n kube-system create serviceaccount tiller
```
然後我們要給予這個服務帳號叢集管理者的角色：
```bash
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
```
最後是初始化 helm。這一步就會把 `tiller` 伺服器建立起來了。
```bash
helm init --service-account tiller
```

## 解除安裝 helm 

### 刪除相關的 role
```bash
kubectl -n kube-system delete deployment tiller-deploy
kubectl delete clusterrolebinding tiller
kubectl -n kube-system delete serviceaccount tiller
```

### 刪除 helm
```bash
helm reset
```

## 使用 helm

### 刪除 release

```bash
helm delete {release_name}
```
要完整刪除必須加上參數 `--purge`

如果要一次刪除所有 release:
```bash
helm list --short | xargs helm delete --purge
```