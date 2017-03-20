# 使用 Kubectl 指令


## Debug

1. 用 Kubernetes Dashboard 查看 cluster 狀態：
```bash
kubectl proxy
```

k8s dashboard 會在 `127.0.0.1:8001/ui` 開啟，以瀏覽器開啟此網址可以使用內建工具檢視 pods、services 等狀態，也可以查看各 container 的 log。

2. 