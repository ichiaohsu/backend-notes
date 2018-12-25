# 深入淺出教你棒棒地管理鏡叢集

萬一你面前有個大洞叫 k8s，然後有人從後面把你推下去，怎麼辦？莫急莫慌莫請假，這時候請假你同事也不會，趕快往下看，教你如何好棒棒地從零開始把 k8s 接回來自己做。ＹＥＳ ＹＯＵ ＣＡＮ！

## 你也棒棒我也棒棒，棒棒的 k8s 大家一起來

### 欲練神功，必先安裝 terraform

terraform 是一套管理雲端資源的工具，由 DevOps 界大名鼎鼎的 HashiCorp 出品。他可以幫忙你新增 Google Compute Instance, Google Storage bucket, Google Container cluster ... 等等族繁不及備載。由此我們可以達成 Infrastructure as Code 的目標。由於在 kubernetes 升級時，我們是採用 blue-green 模式。因此常常需要建立、刪除 container cluster，所以利用 terraform 來把需要建立的東西程式碼化，不用慢慢去那邊對權限。

#### 安裝 terraform

(這邊我忘了，跳過)

#### 開始利用 terraform 管理 GCP 資源

要新增 container cluster 前，我們要先把正式機的 terraform 配置複製一份出來，然後再做 terraform 的前置作業：

```bash
terraform init
```
這會把 terraform 的 "provider" google 下載到這個資料夾來。接下來我們才可以用 terraform 來操控 GCP 資源。

改一下 cluster name，利用 `terraform plan` 確認要建立的 k8s 資源是想要的東西沒有問題以後
```bash
terraform apply
```

稍等一下， terraform 就會把我們的 k8s cluster 建立出來了。接下來是要得到管理 kube-apiserver 的權限。

### 安裝 `kubectl` 工具，取得權限

1. 安裝 `kubectl` 指令工具:

k8s 有一個 API Server，負責監控著你的叢集。月黑風高的晚上，摔在坑裡的你，想要得救第一件事是要摸到這個伺服器。它有開放 RESTful 服務，透過 `kubectl` 指令進行溝通。所以我們的第一件事是要把 `kubectl` 指令工具裝起來。`gcloud` 指令工具已經有整合版，我們透過 `gcloud` 工具來安裝。

```bash
gcloud components install kubectl
```

2. 把你的 GKE 叢集放到 kube.config 裡面:

你必須登入才可以存取 GKE 的各種資源，對 `kubectl` 指令也是一樣的，我們必須為它取得權限。把你的叢集名稱打在以下命令的 `[]` 之間。
```bash
gcloud container clusters get-credentials [CLUSTER NAME]
```

k8s 會建立一個叢集物件，把叢集的基本資料，和一些登入資訊放在這裡。所以即使你從瀏覽器把叢集刪掉，這裡的資訊還是會留著喔！必須手動刪除。好，剛剛幫指令工具拿到登入權限以後，我們就可以開始啦！

### k8s k8s 你擦亮眼

### 更改你想要的更改
