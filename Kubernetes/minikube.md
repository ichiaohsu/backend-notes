# Minikube

## 安裝

## 啟動、關閉 Minikube:
```bash
minikube start

minikube stop
```

## 使用 Google Container Registry 上的 docker images

minikube 有自己的 docker daemon，所以上面看到的 docker images 跟本機端的不同。`minikube ssh` 連至 minikube VM 下 `docker images`，即可發現跟自己看得到的 docker image 們並不相同。

這裡採用一個偷懶的作法，讓我們可以拿到 gcr 上面的 images：在 minikube 上下載 gcr images。

* 首先一樣 `minikube ssh` 連進 minikube VM。

* 接著要拿到 gcr 的授權。minikube並不像裝有 gloud 工具的機器一樣可以直接 pull/push ，所以我們要先拿到 gcloud token。

```bash
gcloud auth print-access-token
```

把印出的 token 複製下來，用在以下的指令：

```bash
docker login -e [YOUR EMAIL] -u oauth2accesstoken -p [ACCESS TOKEN] https://gcr.io
```

* `docker pull` images !

參考：https://cloud.google.com/container-registry/docs/advanced-authentication