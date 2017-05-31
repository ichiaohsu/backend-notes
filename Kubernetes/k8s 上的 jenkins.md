# 如何在 Kubernetes 上建立 jenkins pipeline

## 權限設定

* jenkins 必須有存取 Google Cloud Storage 的權限，在建立 cluster 時必須將 `Storage` 的權限設定成 `Read Write`。
* 照 Google 的說明，也必須開啟 Cloud Source Repositories 的權限。

## 開始部署 Jenkins

### 設定 jenkins services

我們需要兩個 service 來運作 jenkins:

1. 一個設定為 `NodePort` 的 service，可以讓使用者通過 port 8080 來存取 jenkins 的使用者介面做設定。
2. 一個內部的 service，使用 `ClusterIP` 在 port 50000 讓 jenkins executor 與 jenkins master 溝通用。

