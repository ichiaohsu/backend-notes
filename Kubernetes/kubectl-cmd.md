# 用得到的 Kubectl 指令

## 查看、指定 context
```bash
kubectl config get-contexts
```
可以列出所有 context，`CURRNET` 欄位帶 `*` 的則是目前 contexts

切換 context:
```bash
kubectl config use-context [CONTEXT NAME]
```

## 建立 services

## Pods
* 列出詳細 pods 資訊:
```bash
kubectl get pods -o wide
```

## ConfigMaps

* 用檔案建立 ConfigMaps:

```bash
kubectl create configmap [CONFIGMAP NAME] --from-file=[CONFIG FILE PATH]
```

* 以 YAML 格式檢視已經建立的 ConfigMaps:
```bash
kubectl get configmap [CONFIGMAP NAME] -o yaml
```

* 用更改過的檔案建立相同名字的 ConfigMap 並取代原有的 ConfigMap:

先用 `dry-run` 的方式建立 ConfigMap，取得 yaml 檔的內容，然後在 pipeline 裡用這個 yaml 來取代原本的 ConfigMap。

```bash
kubectl create configmap [CONFIGMAP NAME] --from-file [CONFIG FILE PATH] -o yaml --dry-run | kubectl replace -f -
```

## 更新 image
```bash
kubectl set image deployment/[DEPLOYMENT NAME] [POD NAME]=[IMAGE SRC]
```
image 位置如果沒變，試著加上`:latest` tag。
*使用一次之後就沒用了，用以下方法更新！
據說rolling-update會在背景運作

```bash
kubectl apply -f [DEPLOYMENT FILE]
```

## 重開 pod (用取代的方式):

```bash
kubectl get pod [PODNAME] -n [NAMESPACE] -o yaml | kubectl replace --force -f -
```

## Force delete a pod on a dead node
Sometimes when a pod was scheduled to a node with questionable status,
its status would become "Unknown", and unable to be deleted through normal `kubectl delete pod`. Under such circumstances, you should use `grace-period=0` to force delete such pods.

```bash
kubectl delete pod foo --grace-period=0 --force
```

## Debug

* 用 k8s Dashboard 查看 cluster 狀態：
```bash
kubectl proxy
```

k8s dashboard 會在 `127.0.0.1:8001/ui` 開啟，以瀏覽器開啟此網址可以使用內建工具檢視 pods、services 等狀態，也可以查看各 container 的 log。

* 使用終端機輸出log:
```bash
kubectl log [POD NAME] 
```

* 連進 pod 執行特殊指令：
```bash
kubectl exec -it [POD NAME] [COMMAND]
```
