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

2. 如果沒有可用的安全連線 tls 鍵值組，我們就要建立自己的鍵值組給 jenkins 用。

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /tmp/tls.key -out /tmp/tls.crt -subj "/CN=jenkins/O=jenkins"

kubectl create secret generic tls --from-file=/tmp/tls.crt --from-file=/tmp/tls.key --namespace jenkins
```

3. 建立 jenkins 自己的 ingress

利用 /jenkins/k8s/ingress.yml，建立 ingress 給 jenkins 用。

```bash
kubectl apply -f /jenkins/k8s/ingress.yml
```

**注意：為了讓 jenkins ui 擁有固定 IP，我們先在 GCP 上保留了一個固定 IP，然後在 annotation 處設定了 `kubernetes.io/ingress.global-static-ip-name`**

觀察 ingress 物件的狀況:

```bash
kubectl get ing jenkins -n jenkins
```

如果 `backend` 中的物件狀況已經成為 HEALTHY，就可以將 `Address` 中的位址輸入瀏覽器，開始設定 jenkins 了！記得，登入密碼與帳號，就是你剛剛在 /jenkins/options 中設定的帳號密碼。