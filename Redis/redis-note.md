### 取得權限

在有密碼的 redis 裡，你是無力的。所以同志需密碼，密碼真有力。

取得權限的方法有兩種，一是在 `redis-cli` 裡用 `AUTH` 指令

```bash
AUTH 密碼
```

二是在下 `redis-cli` 指令時就帶入密碼參數：

```bash
redis-cli -a 密碼
```

## 查詢 key 的存在時間

### 查詢某特定 key

如果你對某個 key 的快取存在時間有懷疑，可以下這個指令：

```bash
ttl "key"
```

### 列出 redis 上所有不會過期的 key

這個方法是在 bash 裡下 shell script 搭配 `redis-cli ttl` 的指令來查詢，所以會在 bash 裡下這段指令，不需要進到 redis-cli。

```bash
redis-cli keys  "*" | while read LINE ; do TTL=`redis-cli ttl $LINE` ; if [ $TTL -eq -1 ]; then echo "$LINE $TTL"; fi; done;
```

這段在 Stackoverflow 就可以查到，但有些地方會有問題。 `-eq` 有時候無法運作，要換成 `==`。如果 redis 有密碼，就要在指令裡面加上密碼參數 `-a 密碼`。

```bash
redis-cli -a 密碼 keys  "*" | while read LINE ; do TTL=`redis-cli -a 密碼 ttl $LINE` ; if [ $TTL == -1 ]; then echo "$LINE $TTL"; fi; done;
```

踩地雷時間：

```bash
bash: [: missing `]'
```

在上面的 bash script 裡 `[` 和 `]` 需要與括號內容有空格分開。`[ $TTL == -1]` 這樣子是不行的。

```bash
bash: [: too many arguments
```

某些 key 可能有空白，導致 bash 認為這是 key + 參數。

