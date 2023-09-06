# MatrixOne Layout

现在的Layout，它由Header、数据区、索引区、元数据区和Footer组成。

这些区域的作用如下：

Header：记录了当前Layout的版本、元数据区的位置等信息。

数据区：存储了数据对象的数据信息。

索引区：存储了数据对象的索引信息。

元数据区：存储了数据对象的元数据信息，这些元数据包括数据对象的类型、数据对象的大小、行数、列数、压缩算法等信息。

Footer：可以看作是一个Header的镜像。

## 数据结构
#### Extent
在MatrixOne中，Extent负责记录一个IOEntry的位置信息。
```
+------------+----------+-------------+-------------+
| Offset(4B) | Size(4B) |  OSize (4B) |  Algo (1B)  |
+------------+----------+-------------+-------------+

Offset = IOEntry的起始地址
Size = IOEntry存储在对象中的大小
oSize = IOEntry压缩前的原始大小
Algo = 压缩算法类型
```
#### ObjectMeta

MatrixOne的查询业务是从ObjectMeta开始的。
MatrixOne向S3写入数据成功后，会返回一个记录ObjectMeta位置的Extent，并保存到Catalog中。
当MatrixOne执行查询操作需要读取数据时，可以通过这个Extent拿到ObjectMeta，从而拿到Block的真实数据。

ObjectMeta由一个MetaHeader，一个DataMeta、一个TombstoneMeta、多个SubMeta组成。
每一个Meta由多个BlockMeta、一个MetaHeader、一个BlockIndex组成。
MetaHeader和BlockMeta结构一致，用来记录整个Object的全局信息，比如dbID，TableID，ObjectName和Object全局数据的的ZoneMap、Ndv、nullCut等等。

BlockIndex记录后面每一个BlockMeta的地址。ObjectMeta在MatrixOne系统中使用非常频繁，
虽然每一个BlockMeta的地址可以遍历算出来，但是为了节省开销和提高性能，还是记录了BlockIndex。

BlockMeta记录了每一个Block的元数据信息。

#### BlockMeta&MetaHeader
BlockMeta由一个Header和多个ColumnMeta组成。

Header记录了BlockID、Block的Rows、Column Count等信息。

ColumnMeta记录了每一列的数据地址、Null Count（当前列Null值的数量）、Ndv（当前列有多少不同的值）等信息。
##### Block Meta Header
```
+----------------------------------------------------------------------------------------------------+
|                                              Header                                                |
+----------+------------+-------------+-------------+------------+--------------+--------------------+
| DBID(8B) | TableID(8B)|AccountID(4B)|BlockID(20B) |  Rows(4B)  | ColumnCnt(2B)| BloomFilter(13B)   |
+----------+------------+-------------+-------------+------------+--------------+--------------------+
|                                            Reserved(37B)                                           |
+----------------------------------------------------------------------------------------------------+
|                                             ColumnMeta                                             |
+----------------------------------------------------------------------------------------------------+
|                                             ColumnMeta                                             |
+----------------------------------------------------------------------------------------------------+
|                                             ColumnMeta                                             |
+----------------------------------------------------------------------------------------------------+
|                                             ..........                                             |
+----------------------------------------------------------------------------------------------------+

DBID = Database id
TableID = Table id
AccountID = Account id
BlockID = Block id。
Rows = MetaHeader中为对象的行数，BlockMeta中为当前Block的行数
ColumnCnt = 在对象或Block中有多少列
BloomFilter = 存储BloomFilter区域的地址，只在MetaHeader中有效
```
##### Column Meta
```
+--------------------------------------------------------------------------------+
|                                    ColumnMeta                                  |
+--------+---------+-------+-----------+---------------+-----------+-------------+
|Idx(2B) |Type(1B) |Ndv(4B)|NullCnt(4B)|DataExtent(13B)|Chksum(4B) |ZoneMap(64B) |   
+--------+---------+-------+-----------+---------------+-----------+-------------+
|                                   Reserved(32B)                                |
+--------------------------------------------------------------------------------+

Idx = Column的序号
Ndv = Column中有多少不同的值
NullCnt = Column中有多少Null值
DataExtent = Column数据的位置
Chksum = Column数据的checksum
ZoneMap = Column的ZoneMap，大小固定为64B
```

##### Header
Header和Footer记录信息一致，只不过位置不同。
```
+---------+------------+---------------+----------+
|Magic(8B)| Version(2B)|MetaExtent(13B)|Chksum(4B)|
+---------+------------+---------------+----------+

Magic = Engine identity (0x0xFFFFFFFF)
Version = 对象文件的版本号
MetaExtent = ObjectMeta的位置信息
Chksum = ObjectMeta的checksum
```

##### Footer
```
+----------+----------------+-----------+----------+
|Chksum(4B)| MetaExtent(13B)|Version(2B)| Magic(8B)|
+----------+----------------+-----------+----------+
```
## 通过Extent读数据

在介绍[ObjectMeta](layout.md#objectmeta)篇章时，我们知道，MatrixOne向S3写入数据成功后，
会返回一个记录ObjectMeta位置的Extent。这里时候我们会通过Extent来执行读数据的操作。

首先，通过Extent中的地址，向s3请求读一个IO Entry，并且放入MatrixOne的cache中。
这个IO Entry就是ObjectMeta的全部内容。通过位移计算可以拿到BlockIndex，根据BlockIndex记录的Extent可以得到被读的
BlockMeta，到此为止我们元数据操作已经结束，剩下就是向s3请求读一个个Column Data了。
```
      +-----------+
      |   Extent  |
      +-----------+                   
            |
            |
+-------------------------+            
|        IO Entry         |
+-------------------------+
|       ObjectMeta        |
+-------------------------+
            |
            |
+---------------------------------------------------------------------
|                            BlockIndex                              |
+--------+------------+------------+------------+-----------+--------+
| Count  | <Extent-1> | <Extent-2> | <Extent-3> | Extent-4> | ...... |
+--------+------------+------------+------------+-----------+--------+
               |                                       |
               |                                       |
+------------------------------------------------+     |
|             Block1(BlockMeta)                  |     |
+--------------+--------------+--------------+---+     |
|<ColumnMeta-1>|<ColumnMeta-2>|<ColumnMeta-3>|...|     |
+--------------+--------------+--------------+---+     |   
        |               |               |              | 
        |               |               |       +------------------+
  +----------+    +----------+    +----------+  | Block4(BlockMeta)|
  | IO Entry |    | IO Entry |    | IO Entry |  +------------------+
  +----------+    +----------+    +----------+  | <ColumnMeta>...  |
  |ColumnData|    |ColumnData|    |ColumnData|  +------------------+
  +----------+    +----------+    +----------+          |
                                                        |
                                                +------------------+
                                                |    IO Entry...   |
                                                +------------------+       
``` 

## 版本兼容
#### IOEntry
IOEntry表示一个IO单元，具体对应在Layout中就是：ObjectMeta、每一个Column的数据、BloomFilter区还有Header和Footer。
除了Header和Footer，我们需要在每一个IOEntry头添加两个flag：Type&Version。

每一个结构或模块需要实现Encode/Decode函数，然后注册到MatrixOne中，MatrixOne读出IOEntry后会根据这两个flag来选择对应的代码。

```go

const (
	IOET_ObjectMeta_V1  = 1
	IOET_ColumnData_V1  = 1
	IOET_BloomFilter_V1 = 1
	...

	IOET_ObjectMeta_CurrVer  = IOET_ObjectMeta_V1
	IOET_ColumnData_CurrVer  = IOET_ColumnData_V1
	IOET_BloomFilter_CurrVer = IOET_BloomFilter_V1
)

const (
        IOET_Empty   = 0
        IOET_ObjMeta = 1
        IOET_ColData = 2
        IOET_BF      = 3
        ...
)
```
以ObjectMeta为例。我们需注册V1的Encode/Decode函数代码，设置IOET_ObjectMeta_CurrVer为V1,并写入数据。
```go
func EncodeObjectMetaV1(meta *ObjectMeta) []byte {
	...
}
func DecodeObjectMetaV1(buf []byte) *ObjectMeta {
        ...
}
RegisterIOEnrtyCodec(IOET_ObjMeta,IOET_ObjectMeta_V1,EncodeObjectMetaV1,DecodeObjectMetaV1)
ObjectMeta.Write(IOET_ObjMeta, IOET_ObjectMeta_CurrVer)
```
当系统运行了一段时间，也累积不少数据，我们需要修改当前ObjectMeta的结构。这时候需要增加一个版本IOET_ObjectMeta_V2。
这样MatrixOne执行读操作时可以根据Type&Version来选择对应版本的Encode/Decode函数代码
```go
const (
    IOET_ObjectMeta_V1  = 1
    IOET_ObjectMeta_V2  = 2
    ...

    IOET_ObjectMeta_CurrVer  = IOET_ObjectMeta_V2
    ...
)

func EncodeObjectMetaV2(meta *ObjectMeta) []byte {
	...
}
func DecodeObjectMetaV2(buf []byte) *ObjectMeta {
        ...
}
RegisterIOEnrtyCodec(IOET_ObjMeta,IOET_ObjectMeta_V2,EncodeObjectMetaV2,DecodeObjectMetaV2)
ObjectMeta.Write(IOET_ObjMeta, IOET_ObjectMeta_CurrVer)
```
