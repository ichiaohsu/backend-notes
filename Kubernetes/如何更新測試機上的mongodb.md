# 用 consumer 上的 mongo 備份檔同步測試機

有時候，測試機上的database因為沒有與產品資料庫同步，可能會造成問題。這時候我們要用 `kubectl` 和 `gcloud` 指令們來手動同步測試機上的資料庫。

簡單說，流程如下：
1. 拿到 consumer 上的資料庫備份
2. 把拿到的備份檔傳到 kubernetes pod 裡
3. 刪除舊的資料庫
4. 用備份檔案恢復資料庫

## 工欲更新DB，必先有DB

首先我們必須先拿到 mongodb 的資料，才有辦法用最新的資料庫來更新我們的測試機。這邊我們採用 consumer 機上面每日會拉下來的備份檔，所以任務變成：如何從 consumer 上面下載這個備份檔？答案是 `scp`，`gcloud` 都有內建。

```bash
gcloud compute scp consumer-1:/tmp/dump/db/ ~/Downloads/db/ --recurse
```

`~/Downloads/db/` 可以替換成你想要的路徑，因為我們是 copy 一整個資料夾，所以 `--recurse` 也是必須的。如果指令報錯 `scp: /tmp/dump/db: not a regular file`，多半必須加上 `--recurse`。

## 上傳備份檔

這邊用 `kubectl` 內建的 `cp` 來把備份檔通通傳到 mongodb 所在的 pod 裡面：

```bash
kubectl cp ~/Downloads/db mongo-0:/tmp/db
```

暫時放一下，用 `/tmp` 就好。如果你不想放在 db（或是之前手動更新的時候用過了)，你也可以改一下資料夾的名字。

## 刪掉不要的資料庫

不把舊的資料庫刪掉，到時候 import 新的資料庫就要取個新名字，這樣後端、keystone 全部都要改相對應的名字，很不方便。這邊我們先把它刪掉。先連到 mongodb 所在的 pod：

```bash
kubectl exec -it mongo-0 bash
```

然後用 mongo shell 把資料庫刪掉：

```bash
mongo
> use db
> db.dropDatabase()
```

## 重生吧資料庫

先退出 mongo shell，然後用 `mongorestore` 重建資料庫。

```bash
mongorestore /tmp/db -d db
```

`/tmp/db` 就是我們剛剛上傳備份檔的路徑，`db` 是我們要 import 的資料庫名字，所以我們都用 `db`。

到這邊就手動同步資料庫完成。