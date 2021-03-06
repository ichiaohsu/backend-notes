# 如何在 Kubernetes 上建立 jenkins pipeline

## 權限設定

* jenkins 必須有存取 Google Cloud Storage 的權限，在建立 cluster 時必須將 `Storage` 的權限設定成 `Read Write`。

已經建立好的 VM instances 無法改變權限設定，但可以重起新的 Node pool，

* 照 Google 的說明，也必須開啟 Cloud Source Repositories 的權限。

## 開始部署 Jenkins

一般來說，我們把 jenkins 的相關服務都利用 namespace 切開。第一步，我們要先建立一個 namespace 給 jenkins 用。
```bash
kubectl create ns jenkins
```

### 建立 Persistent Disk

因為 pods 的 ephermeral 的特性，jenkins 需要一個穩定的儲存空間來存放 xml 檔和各種設定，我們必須先建立一個 GCP persistent Disk，然後 mount 上我們的 jenkins pod。因此在建立 deployment 之前，我們要先來建 persistent Disk。

利用以下的 `gcloud` 指令來建立 persistent Disk，會先把某些必要的設定塞進這個磁碟空間，非常必要。（沒有會建立 pod 失敗的意味)

```bash
gcloud compute images create jenkins-home-image --source-uri https://storage.googleapis.com/solutions-public-assets/jenkins-cd/jenkins-home-v3.tar.gz
gcloud compute disks create jenkins-home --image jenkins-home-image --zone asia-east1-a
```

註：根據 google 的教學，persistant Disk 在使用以前必須先格式化，也就是要:
1. 先 mount 到任一虛擬機器
2. 下格式化指令
3. unmount persistent Disk
但個人兩個都做過，其實不格式化好像才可以......記得要用 jenkins images 靈光充滿這個磁碟就是了。

### 設定 jenkins 的密碼

我們要利用 k8s secret 的功能來設定 jenkins 的密碼。密碼的設定寫在 jenkins/options 這個檔案裡。把檔案中的 CHANGE_ME 改成你想要的密碼。

如果你沒有想要自己設定密碼，也可以用 `openssl` 指令來產生密碼:
```bash
PASSWORD=`openssl rand -base64 15`; echo "Your password is $PASSWORD"; sed -i.bak s#CHANGE_ME#$PASSWORD# jenkins/options
```

改好以後，產生 k8s secret 物件:
```bash
kubectl create secret generic jenkins --from-file=jenkins/k8s/options --namespace=jenkins
```

### 部署 jenkins services & deployments

我們需要兩個 service 來運作 jenkins:

1. 一個設定為 `NodePort` 的 service，可以讓使用者通過 port 8080 來存取 jenkins 的使用者介面做設定。
2. 一個內部的 service，使用 `ClusterIP` 在 port 50000 讓 jenkins executor 與 jenkins master 溝通用。

可以利用 /jenkins/k8s 資料夾裡的 yaml 設定檔來建立。
```bash
kubectl apply -f /jenkins/k8s
```

### 為 jenkins 建立 Load Balancer

為了讓使用者連進 jenkins 的使用者介面，我們要建立 ingress 物件來橋接剛剛建立的 `jenkins-ui` service。

1. 確定 service 正常運作。

```bash
kubectl get svc -n jenkins
```

2. 如果沒有可用的安全連線 tls 鍵值組，我們就要建立自己的鍵值組給 jenkins 用

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /tmp/tls.key -out /tmp/tls.crt -subj "/CN=jenkins/O=jenkins"

kubectl create secret generic tls --from-file=/tmp/tls.crt --from-file=/tmp/tls.key --namespace jenkins
```

3. 建立 jenkins 自己的 ingress

利用 /jenkins/k8s/ingress.yml，建立 ingress 給 jenkins 用。

```bash
kubectl apply -f /jenkins/k8s/ingress.yml
```

**注意：為了讓 jenkins ui 擁有固定 IP，我們先在 GCP 上保留了一個固定 IP，然後在 annotation 處設定了 `kubernetes.io/ingress.global-static-ip-name`**

觀察 ingress 物件的狀況:

```bash
kubectl get ing jenkins -n jenkins
```

如果 `backend` 中的物件狀況已經成為 HEALTHY，就可以將 `Address` 中的位址輸入瀏覽器，開始設定 jenkins 了！記得，登入密碼與帳號，就是你剛剛在 /jenkins/options 中設定的帳號密碼。

## show me the 權限

建立好 jenkins server 以後，要讓 jenkins 可以拿存在 google container registry 裡的 docker image 來更新 k8s 的 docker image，我們必須先幫 jenkins 拿到 google 與 kubernetes 的權限。

### 通往 google container registry 之路

1. 點 Credentials 後再點擊子選項 System
2. 點擊 Global credentials
3. 點 Add Credentials
4. 選擇 Google Service Account from metadata 後按 OK

### Kubernetes 之鑰

如果 jenkins 跑在 GKE 裡面，就可以使用 google service account 來做認證。

1. 點 Credentials 後再點擊子選項 System
2. 點擊 Global credentials
3. 點 Add Credentials
4. 選擇 Kubernetes Service Account 後按 OK

## 設定替身使者

jenkins 被觸發以後，會喚起一個實際來做事的 container，稱為 executor。在 build 專案以前，要先設定用哪種 docker image 來當我們的 executor。

1. 點 Manage Jenkins，然後點 Configure System
2. `# of executor` 這項要輸入 `0`，確定我們會用 executor 來跑各項任務。
3. 拉到 Cloud: Kubernetes 這段，確定 Kubernetes URL 這段用的: `https://kubernetes.default`
4. Kubernetes Namespace 設成 default，對於要更新在 default 下面的 pods 比較方便
5. Jenkins URL: `http://jenkins-ui.jenkins.svc.cluster.local:8080`
6. Jenkins tunnel: `jenkins-discovery.jenkins.svc.cluster.local:50000`
7. 最重要的來了，設定 pod template！jenkins 被 trigger 以後，會根據這邊的模板設定喚醒一個 executor 來做事。按照以上的步驟設定到這邊，應該 Kubernetes Cloud 這個選項都已經被建立好了，裡面也已經有一個預設的 jnlp slave image。但這個映像檔的 kubectl 版本可能太舊，若是你的 k8s server 版本在 1.5 以上，會無法取得資源。這邊要自己做一版映像檔，放在 google container registry 上再掛進去。

**注意！！這個映像檔名字，一定要叫 jenkins-k8s-slave**

8. 設定兩組 Host Path Volume，讓我們的 jnlp slave 可以連到 docker daemon(但這應該已經預設好了，用預設就好)
```bash 
Host Path  /usr/bin/docker
Mount Path /usr/bin/docker
Host Path  /var/run/docker.sock
Mount Path /var/run/docker.sock
```

大功告成！
你可以開始建立各種 jenkins job 了！