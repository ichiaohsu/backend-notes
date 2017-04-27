# 基本說明

## Clusters

一個 cluster 由 master node 與被其操控的 nodes 所組成。Master 包含：

* Etcd: 儲存 cluster 狀態
* API server:
* Controller Manager Server:

## Nodes

Nodes 是一台虛擬主機(VM)或實體主機資源。分享相同設定的 node 稱為 node-pool，預設為 `default`。一個 node 上可運行多個 pods，不同 pods 透過 intranet 溝通。每個 node 上皆運作有兩個元件：

* kubelet:
* kube-proxy:

可以兩種模式運作：`userspace` 和 `iptables`

    * userspace: In the userspace mode, kube-proxy is running as a userspace process i.e. regular application. It terminates all incoming service connections and creates a new connection to a particular service endpoint. The advantage of the userspace mode is that because the connections are created from userspace process, if the connection fails, kube-proxy can retry to a different endpoint. 

    * iptables: In iptables mode, the traffic routing is done entirely through kernelspace via quite complex iptables kung-fu. Feel free to check the iptables rules on each node. This is way more efficient than moving the packets from the kernel to userspace and then back to the kernel. So you get higher throughput and better latency. The downside is that the service can be more difficult to debug, because you need to inspect logs from iptables and maybe do some tcpdumping or what not.

## Autoscaler

Cluster Autoscaler 運作在 k8s master 節點上。當 k8s 找不到足夠資源的節點來運行 pods，這些 pods 的狀態會成為 `pending`，這些 pods 也被稱為 unschedulable pods。Autoscaler 會由 api server 尋找所有的 pods，每十秒就檢視是否有 unschedulable pods，如果有，此時就必須進行 scale up。

每十秒 autoscaler 也會檢查是否有閒置的節點，這些節點要滿足以下條件：
* 所有 pods 使用的 CPU 和記憶體總和低於節點資源的 50%
* 節點上有不受 deployment、replica set、replication controller及 job 管理的 pod，因為刪除節點，這些 pods 並不會被重建，所以 node 不會被刪除。
* 結點不存在非 k8s 部署的 pod。
* 不存在有自己儲存空間的 pod。

被認為不被需要的節點，十分鐘之後將會被刪除。Scale down 最快也只會發生在最近一次 scale up 的十分鐘後。

## Pods

Kubernetes 的部署以 pod 為最小單元，一個 pod 可以包含一到多個 docker container。Pod 內的 container 可以共享相同的 volume，network 和 namespace，因此可透過 `localhost` 彼此溝通，毋需透過 NAT。也因此不同的 container 無法使用相同的 port。各 pod 擁有自己的 cluster IP。與不同 node 之間的溝通，可以透過此 cluster IP(此段待證明)。

Pods 理論上會持續運作直至被人為或控制器 (controller) 命令所終止。Pods 的壽命被設計為短暫的(ephemeral)。

當節點因為錯誤或維護而重開，獨立而不被控制器所操控的 pods 也不會被重新部署在這些節點上。因此最好使用 Replication center 或 Deployment 來建立新的 pods。除了指定 `restartPolicy: Never` 的情況，這些控制器會試圖在你的機器上運作指定數量的 pods。

## Services

Pods 的生命是短暫的。即使我們以人工或控制器的方式，用同樣的模板建立一個 pod，每次建立的 pod 都會被賦予不同的名字與 cluster IP。問題來了：如果 pods 的參數是如此不穩定，每次重新產生、被 autoscale 產生的 pods 都擁有不一樣的 IP，我們要怎麼樣保證能夠存取到我們想要的 pods 呢？

k8s 引入了 service 抽象層。每個節點上的 kube-proxy 會持續監視 k8s master server 上的 endpoint，一旦有 service 被建立，kube-proxy 會建立 iptable，將 service 的名字綁定在 selector 區塊中所指定的 pods。未來指向這些 service 的流量都會被導引到這些 pods，也因此 service name 可以作為 cluster 中的 DNS 使用。舉例而言，在我們建立了 RESTful server 的 service: rest-service 後，前端設定中就可以用 rest-service 代替 Restful server 的 IP。

自 k8s v1.0 起，services 可被視為一種第四層結構。從 v1.2 開始，k8s 加入了第七層 (HTTP) 版本的 service: Ingress。如果將 service 宣告為 LoadBalancer 時，並指定 IP 時，就會看到此 IP 出現在 service 中的 LoadBalancer Ingress。

Services 分四種:
* ClusterIP: 只會分配 cluster IP，不能從外部訪問。這是預設的 service 類別。
* NodePort: 讓各節點的 IP 在 NodePort 開放。即使不在叢集內也可以利用 `<NodeIP>:<NodePort>` 連接。
* LoadBalancer: 使用雲端服務提供者的 load balancer 使位於 cluster 之外的使用者可以存取這個 service。事實上，這個選項會自動建立 NodePort 及 ClusterIP，並且產生一個 TCP 負載平衡器。如果想使用 HTTP 負載平衡器，必須引入 Ingress API，並將之指向對應的 service。
* ExternalName: Map 至 `externalName` 的內容所指。

p.s. 一般來說 service type 指定為 `type:LoadBalancer` 毋須指定 `NodePort`，但在 AWS 上必須這麼做。

參考: https://kubernetes.io/docs/user-guide/services/

## Ingress

即使在 service 設定中利用 `type:LoadBalancer` 引入平台提供者的負載平衡器，這也只會是一個 TCP 類型的負載平衡器。要建立第七層的負載平衡，k8s 利用 Ingress 物件來建立路由規則，將外部流量導引至對應的 service。

要使用 ingress，必須建立 ingress 物件以及 ingress controller。

- Run Ingress controller

nginx-controller is just a simple application running in a K8s pod which first register themselves into the list of controllers via API service and store some configuration there. 

(An Ingress Controller is a daemon, deployed as a Kubernetes Pod, that watches the apiserver's /ingresses endpoint for updates to the Ingress resource. Its job is to satisfy requests for ingress.)

- Create Ingress (API object)

the ingress controllers are actually K8s API watcher applications which watch particular /registry/ingress endpoints changes to keep an eye on particular K8s service endpoints.

ingress controller monitor service endpoints eg. Ingress controllers don’t route traffic to the service, but rather to the actual service endpoints i.e. pods.

K8s service is like we should know by now, just another API watcher which simply watches /registry/endpoints. This endpoint is updated by K8s controller-manager. Even if the K8s controller-manager does pick up the endpoints change, there is no guarantee kube-proxy has picked it up and updated the iptables rules accordingly - kube-proxy can still route the traffic to the dead pods. So it’s safer for the ingress controllers to watch the endpoints themselves and update their routing tables as soon as controller-manager updates the list of endpoints.

(traffic from outside K8s cluster) -> (ingress-controller) -> (kube-dns lookup for particular service) -> (kube-proxy/iptables) -> service endpoint (pod)

ref:

1. http://containerops.org/2017/01/30/kubernetes-services-and-ingress-under-x-ray/
2. https://github.com/kelseyhightower/kubernetes-the-hard-way