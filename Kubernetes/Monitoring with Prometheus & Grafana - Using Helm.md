### 安裝 Prometheus
將 Prometheus 安裝至 `monitoring` 命名空間，設定 `scrape_interval=15s`，並且把 `prometheus-server` expose。
```bash
helm install stable/prometheus --name=prometheus --namespace=monitoring --set server.global.scrape_interval=15s,server.service.type=LoadBalancer
```

大多數 dashboard 預設的時間區間都使用一分鐘，如果不自行設定 `scrape_interval`，而使用預設的 `1m`，通常會因為 sampling rate 不夠造成儀表板空白的現象。因此這裡把 `scrape_interval` 設成 15 秒避免麻煩。

### 安裝 Grafana
ㄧ樣把 Grafana 安裝到 `monitoring` 空間，並且把其 server expose。
```bash
helm install stable/grafana --name=grafana --namespace=monitoring --set service.type=LoadBalancer
```

利用以下指令來得到預設的 Grafana 密碼：
```bash
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```
接著只要取得 grafana server 的 service ip，就可以透過 `http://{grafana-service-ip:80}` 來訪問 grafana server 了！

登入時使用 `admin/{剛剛用 kubectl 指令得到的密碼}` 就可以用管理者的身份登入到 grafana server。

在 `Create` 選單裡選擇 `Import`，就可以把 Grafana 官網上的 dashboard 給引入到你的 server。
