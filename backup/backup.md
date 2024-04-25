# Incremental Backup Feature

为了保证Prod环境数据的安全性，已经提供备份功能。但由于数据量较大，每次备份都是全量备份，导致备份时间较长，且占用大量存储空间。
为了解决这个问题，需要提供增量备份功能。

### 备份

增量备份需要执行在全量备份之后，每次备份只备份发生变化的数据。备份数据的格式和全量备份一致，只是数据量较小。
**例子**

```shell
./mo_br backup --host "127.0.0.1" 
--port 6001 --user "dump" --password "111" 
--backup_dir "filesystem"  --path "/backup"
--backup_type "incremental" --base_id "ab1e7f0a-3136-11ee-b513-acde48001122"
```

```shell

//备份结果 
bakcup
├── BackupData
│   ├── full_backup:b147d8e-3136-11ee-b513-acde48001122
│   ├── incremental_backup:ab1e7ff0-3136-11ee-b513-acde48001122
│   ├── incremental_backup:ab1e7f0a-3136-11ee-b513-acde48001122
│   └── ....
├── BackupMeta
│   ├── b147d8e-3136-11ee-b513-acde48001122.full
│   ├── ab1e7ff0-3136-11ee-b513-acde48001122.inc
│   ├── ab1e7f0a-3136-11ee-b513-acde48001122.inc
│   └── ....
```

### 恢复

恢复时，需要提供增量备份的ID，和之前的全量恢复操作一致。

只是工具会解析增量备份的ID，确定该ID之前的所有增量和全量全量，依次恢复有效的数据。
```shell
./mo_br restore ab1e7f0a-3136-11ee-b513-acde48001122 
--restore_dir filesystem --restore_path "/restore"
```
