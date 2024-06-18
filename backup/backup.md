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

//base_id: 上次一次备份的ID，主要用于确定上次备份的时间戳。
          这个时间戳用于备份时判断那些数据需要备份，那些已经备份过了。
          if object create ts > base_id ts, then backup this object.
          else skip this object.
```

```shell

//备份结果 
bakcup
|   |  // BackupData
│   ├── full_backup:b147d8e-3136-11ee-b513-acde48001122
│   ├── incremental_backup:ab1e7ff0-3136-11ee-b513-acde48001122
│   ├── incremental_backup:ab1e7f0a-3136-11ee-b513-acde48001122
│   └── ....
|   |  // BackupMeta
│   ├── b147d8e-3136-11ee-b513-acde48001122.full
│   ├── ab1e7ff0-3136-11ee-b513-acde48001122.inc
│   ├── ab1e7f0a-3136-11ee-b513-acde48001122.inc
│   └── ....

BackupData: 需要修改tae_list文件，增加object create ts，用于确定object的数据位置存在哪一个备份集中。
BackupMeta: 保存备份的元数据信息，包括数据位置，时间戳等。
```

### 恢复

恢复时，需要提供增量备份的ID，和之前的全量恢复操作一致。

只是工具会解析增量备份的ID，确定该ID之前的所有增量和全量全量，根据tae_list依次恢复数据。
```shell
./mo_br restore ab1e7f0a-3136-11ee-b513-acde48001122 
--restore_dir filesystem --restore_path "/restore"
```

```shell
// 新增的恢复逻辑
├── BackupData
│   ├── full_backup:b147d8e-3136-11ee-b513-acde48001122 : full_t30
│   ├── incremental_backup:ab1e7ff0-3136-11ee-b513-acde48001122 : inc_t50
│   ├── incremental_backup:ab1e7f0a-3136-11ee-b513-acde48001122 : inc_t100
│   └── ....
├── BackupMeta
│   ├── b147d8e-3136-11ee-b513-acde48001122.full : t30
│   ├── ab1e7ff0-3136-11ee-b513-acde48001122.inc : t50
│   ├── ab1e7f0a-3136-11ee-b513-acde48001122.inc : t100
│   └── ....

inc_t100的tae_list:
object1|597766|checksum|t40|false  -> inc_t50
object2|597766|checksum|t53|true  -> inc_t100
object3|597766|checksum|t80|true  -> inc_t100
object4|597766|checksum|t23|false  -> full_t30
object5|597766|checksum|t13|false  -> full_t30
object6|597766|checksum|t90|true  -> inc_t100
```
