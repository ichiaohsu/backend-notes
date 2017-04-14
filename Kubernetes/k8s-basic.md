# 基本說明

## Clusters

一個 cluster 由 master node 與被其操控的 nodes 所組成。Master 包含：

* Etcd:
* API server:
* Controller Manager Server:

### Autoscaler

Cluster Autoscaler 運作在 k8s master 節點上。Autoscaler 會由 api server 尋找所有的 pods，每十秒就檢視是否有 unschedulable pod。當 k8s 找不到節點來運行 pods，就會產生所謂的 unschedulable pods，此時就必須進行 scale up。

每十秒 autoscaler 也會檢查是否有閒置的節點，這些節點要滿足以下條件：
* 所有 pods 使用的 CPU 和記憶體總和低於節點資源的 50%
* 節點上有不受 deployment、replica set、replication controller及 job 管理的 pod，因為刪除節點，這些 pods 並不會被重建，所以 node 不會被刪除。
* 結點不存在非 k8s 部署的 pod。
* 不存在有自己儲存空間的 pod。

被認為不被需要的節點，十分鐘之後將會被刪除。Scale down 最快也只會發生在最近一次 scale up 的十分鐘後。


## Nodes

Nodes 是一台虛擬主機(VM)或實體主機資源。分享相同設定的 node 稱為 node-pool，預設為 `default`。一個 node 上可運行多個 pods，不同 pods 透過 intranet 溝通。每個 node 上皆運作有兩個元件：

* kubelet:
* kube-proxy:

> 可以兩種模式運作：`userspace` 和 `iptables`

> * userspace: 
> In the userspace mode, kube-proxy is running > as a userspace process i.e. regular application. It 
> terminates all incoming service connections and creates a new connection to a particular service 
> endpoint. The advantage of the userspace mode is that because the connections are created from 
> userspace process, if the connection fails, kube-proxy can retry to a different endpoint. 

> * iptables:
> In iptables mode, the traffic routing is done entirely through kernelspace via quite complex iptables 
> kung-fu. Feel free to check the iptables rules on each node. This is way more efficient than moving 
> the packets from the kernel to userspace and then back to the kernel. So you get higher throughput and 
> better latency. The downside is that the service can be more difficult to debug, because you need to 
> inspect logs from iptables and maybe do some tcpdumping or what not.

## Pods

Kubernetes 的部署以 pod 為最小單元，一個 pod 可以包含一到多個 docker container。Pod 內的 container 可以共享相同的 volume，network 和 namespace，因此可透過 `localhost` 彼此溝通。也因此不同的 container 無法使用相同的 port。各 pod 擁有自己的 cluster IP。與不同 node 之間的溝通，可以透過此 cluster IP(此段待證明)。

*(containers within a Pod can all reach each other’s ports on localhost, and all pods in a cluster can see each other without NAT)*

Pods 理論上會持續運作直至被人為或控制器 (controller) 命令所終止。Pods 的壽命被設計為短暫的(ephemeral)。

*  Naked pods will not be rescheduled in the event of node failure.
Replication controllers are almost always preferable to creating pods, except for some explicit restartPolicy: Never scenarios

## Services

Service name 會被當作 DNS name，必須與程式碼中的 api server host 統一。

自 k8s v1.0 起，services 可被視為一種第四層結構。從 v1.1 開始，Ingress API 以 service 第七層結構(HTTP)版本加入 k8s。如果將 service 宣告為 LoadBalancer 時，並指定 ip 時，就會看到此 ip 出現在 service 中的 LoadBalancer Ingress。

Services 分四種:
* ClusterIP: 只會分配 cluster ip，不能從外部訪問。5這是預設的服務類別。
* NodePort: 讓各節點的 ip 在 NodePort 開放。即使不在叢集內也可以利用 `<NodeIP>:<NodePort>` 連接。
* LoadBalancer: 使用雲端服務提供者的 load balancer 使服務可以被取得。NodePort 及 ClusterIP 會被自動建立。
* ExternalName: Map 至 `externalName` 的內容所指。

p.s. 一般來說 service type 指定為 `type:LoadBalancer` 毋須指定 `NodePort`，但在 AWS 上必須這麼做。

參考: https://kubernetes.io/docs/user-guide/services/

## Ingress

即使在 service 設定中利用 `type:LoadBalancer` 引入平台提供者的負載平衡器，這也只會是一個 tcp 類型的負載平衡器。要建立第七層的負載平衡，k8s 利用 ingress 物件來建立路由規則，將外部流量導引至對應的 service。

(Ingress is an API resource which represents a set of traffic routing rules that map external traffic to K8s services.)

要使用 ingress，必須建立 ingress 物件以及 ingress controller。

1. Run Ingress controller
2. Create Ingress (API object)


ref:

1. http://containerops.org/2017/01/30/kubernetes-services-and-ingress-under-x-ray/