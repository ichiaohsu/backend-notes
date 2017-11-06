### 列出所有 constraints

```bash
SELECT COLUMN_NAME, CONSTRAINT_NAME, REFERENCED_COLUMN_NAME, 
REFERENCED_TABLE_NAME FROM information_schema.KEY_COLUMN_USAGE 
WHERE TABLE_NAME = 'YOUR TABLENAME';
```

### 拿掉 constraints

mySQL 沒有 `DROP CONSTRIANT`，要用 `DROP FOREIGN KEY`。

```bash
ALTER TABLE `TABLE_NAME` DROP FOREIGN KEY `FOREIGN KEY`;
```

### 新增 constraints

```bash
ALTER TABLE `TABLE_NAME` ADD CONSTRAINT `CONSTRAINT_NAME` FOREIGN KEY `FOREIGN_KEY_NAME` REFERENCES `REFERENCED_TABLE_NAME (REFERENCED_COLUMN_NAME)` ON CASCADE DELETE ON CASCADE UPDATE;
```

`ON CASCADE DELETE` 和 `ON CASCADE UPDATE` 可以讓母表有變動時，在子表中把相對應的值刪掉或更新。沒有設定這兩項的話，預設為 `RESTRICT`，當要刪除母表中的項目就會被拒絕了。

### 查詢 INNODB ENGINE 的狀態

有時候在做 mySQL 操作時，產生錯誤卻無法從中得到真正的問題(比如說 ADD CONSTRAINT 失敗的錯誤就沒什麼訊息可用)。這時候我們可以藉由以下的指令來查詢 INNODB ENGINE 的狀態，得到一些有用的資訊。

```bash 
SHOW ENGINE INNODB STATUS
```

像上面所講的 ADD CONSTRAINT 失敗，就可以去看 `LATEST FOREIGN KEY ERROR` 部分，看能不能有什麼有用的資訊。

### 查詢表格的 schema

```bash
desc [TABLE NAME]
```