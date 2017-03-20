# 基本說明

## Clusters

一個 cluster 由 master node 與被其操控的 nodes 所組成。Master 包含：

* Etcd:
* API server:
* Controller Manager Server:

## Nodes

Nodes 是一台虛擬主機(VM)或實體主機資源。分享相同設定的 node 稱為 node-pool，預設為 `default`。一個 node 上可運行多個 pods，不同 pods 透過 intranet 溝通。每個 node 上皆運作有兩個元件：

* kubelet:
* kube-proxy:

## Pods

Kubernetes 的部署以 pod 為最小單元，一個 pod 可以包含一到多個 docker container。Pod 內的 container 可以共享相同的 volume，network 和 namespace，因此可透過 `localhost` 彼此溝通。不同的 container 不可使用相同的 port。

Pods理論上會持續運作直至被人為或控制器(controller)命令所終止。Pods被設計為短暫的(ephemeral)。





