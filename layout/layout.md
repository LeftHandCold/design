# Layout Reference
- [Format](layout.md#format)
  - [Type](layout.md#type)
  - [Extent](layout.md#extent)
  - [Header](layout.md#header)
  - [Metadata Area](layout.md#metadata-area)
  - [Object Meta](layout.md#object-meta)
  - [Block Meta](layout.md#block-meta-header)
- [Data Structure](layout.md#data-structure)
  - [Appendable Block](layout.md#appendable-block)
- [Index Structure](layout.md#index-structure)
  - [Zone Map](layout.md#zone-map)
  - [Bloom Filter](layout.md#bloom-filter)
- [IO Path](layout.md#io-path)
  - [Block](layout.md#read-block)
  - [Object](layout.md#read-block)
##  Format
### Storage File Format
![Layout.jpg](image/Layout.jpg)
##### Type
```
+---------------+
|   MetaType    |
+---------------+
| * ObjectMeta  |
| * BlockMeta   |
| * ColumnData  |
| * ZoneMap     |
+---------------+

MetaType:       Meta enumeration type
ObjectMeta    = Object metadata
BlockMeta     = Block metadata (a batch is one block)
ColumnData    = Column data metadata
ZoneMap       = Zonemap metadata

+---------------+
|  ObjectType   |
|  (DataType)   |
+---------------+
| * Data        |
| * Checkpoint  |
| * GCMeta      |
| * ETL         |
| * QueryResult |
+---------------+

ObjectType:     Object enumeration type
Data          = Database data
Checkpoint    = Checkpoint data
GCMeta        = Metadata of Disk cleaner
ETL           = ETL data
QueryResult   = Cache data of frontend query results
```

#### Extent

```
+------------+----------+-------------+-------------+
| Offset(4B) | Size(4B) |  OSize (4B) |  Algo (1B)  |
+------------+----------+-------------+-------------+

Extent Size = 13B
An extent records the address of a data/meta unit in the object
Offset = Offset of Metadata/ColumnData/BloomFilter
Size = Size of Metadata/ColumnData/BloomFilter
oSize = Original Metadata/ColumnData/BloomFilter size
Algo = Compression algorithm type for Data
```

#### Header
```
+---------+------------+---------------+-----------+--------------+
|Magic(8B)| Version(2B)|MetaExtent(13B)| Chksum(4B)| Reserved(21B)|
+---------+------------+---------------+-----------+--------------+

Header Size = 64B
Magic = Engine identity (0x01346616). TAE only
Version = Object file version
MetaExtent = Extent of Metadata
Chksum = Metadata checksum
Reserved = 21 bytes reserved space
```
#### Metadata Area

```
+----------------------------------------------------------------------------------------------+
|                                         <Object Meta>                                        |
+----------------------------------------------------------------------------------------------+
|                                         <BlockMeta-1>                                        |
+----------------------------------------------------------------------------------------------+
|                                         <BlockMeta-2>                                        |
+----------------------------------------------------------------------------------------------+
|                                          ..........                                          |
+----------------------------------------------------------------------------------------------+
|                                     <Block Zonemap Area>                                     |
+----------------------------------------------------------------------------------------------+
```
##### Object Meta
An object can only have one ObjectMeta item
```
+--------------+---------+---------+------------+--------------+-------------+--------+
| MetaType(1B) | Type(1B)| DBID(8B)| TableID(8B)| SegmentID(4B)|AccountID(4B)|Rows(4B)|
+--------------+-------------------+---------------------------+-------------+--------+
| ColumnCnt(2B)| BlkMetaExtent(13B)|    ZmMetaExtent(13B)      |      Resered(24B)    |
+--------------+-------------------+---------------------------+----------------------+
| <Col1>|<Col2>|<Col3>|<Col4>|<Col5>|<Col6>|...
                          |
                          |
           +--------+------------+------------+-------------+
           | Ndv(4B)| NullCnt(4B)| Zonemap(64)| Resered(24B)|
           +--------+------------+------------+-------------+
                                     
MetaType = 00
Type = Object enumeration type
DBID = Database id
TableID = Table id
SegmentID = Segment id
AccountID = Account id
Rows = How many rows are contained in object
ColumnCnt = The number of column in the object zonemap
BlkMetaExtent = Extent of block metada
ZmMetaExtent = Extent of block zonemap area

Ndv = How many distinct values in the column
NullCnt = How many Null values in the column
Zonemap = Contains tow 32B values: min and max

```
##### Block Meta Header
```
+---------------+---------------+----------------+----+----------------------+
| <BlockMeta-1> | <BlockMeta-2> |  <BlockMeta-3> |....| <Block Zonemap Area> |
+---------------+---------------+----------------+----+----------------------+
       |
       |
+----------------------------------------------------------------------------------------------------+
|                                              Header                                                |
+------------+--------+-----------+-----------+-------------+-----------+----------------------------+
|MetaType(1B)|Type(1B)|Version(2B)|BlockID(4B)|ColumnCnt(2B)|ExistZM(1B)|        Reserved(21B)       |
+------------+--------+-----------+-----------+-------------+-----------+----------------------------+
|                                             ColumnMeta                                             |
+----------------------------------------------------------------------------------------------------+
|                                             ColumnMeta                                             |
+----------------------------------------------------------------------------------------------------+
|                                             ColumnMeta                                             |
+----------------------------------------------------------------------------------------------------+
|                                             ..........                                             |
+----------------------------------------------------------------------------------------------------+

BlockMetaHeader Size = 32B
MetaType = 01
Type = Object enumeration type
Version = version of block data(vector)
BlockID = Block id
ColumnCnt = The number of column in the block
ExistZM = Whether to write zonemap
```
##### Column Meta
```
+------------------------------------------------------------------------------------------------------+
|                                            DataColumnMeta                                            |
+-------------+--------+--------+----------------+----------+----------------+----------+--------------+
|MetaType(1B) |Type(1B)| Idx(2B)| DataExtent(13B)|Chksum(4B)|  BFExtent(13B) |Chksum(4B)| Reserved(26B)|
+-------------+--------+--------+----------------+----------+----------------+----------+--------------+

ColumnMeta Size = 64B
MetaType = 02
Type = The data type of the Column
Idx = Column index
DataExtent = Exten of Column Data
Chksum = Column Data checksum
BFExtent = Exten of BloomFilter
```
##### Foot
```
+----------+----------------+-----------+----------+
|Chksum(4B)| MetaExtent(13B)|Version(2B)| Magic(8B)|
+----------+----------------+-----------+----------+

Magic = Engine identity (0x01346616). TAE only
Version = Object file version
MetaExtent = Extent of Metadata
Chksum = Metadata checksum
```
## Data Structure
### Appendable Block
```
```

## Index Structure
### Zone Map
### Bloom Filter

## IO Path
#### Read block
```
          +-------------------+
          |     MetaLoction   |
          +-------------------+                   
                    |
                    |
+--------------------------------------------------------------------+
|                             IO Entry                               |
+--------------------------------------------------------------------+
|        Meta(ObjectMeta/BlockMetaHeader/ColumnMeta/ZoneMap)         |
+--------+----------------+----------------+----------------+--------+
| Block  | <ColumnMeta-1> | <ColumnMeta-2> | <ColumnMeta-3> | ...... |
+--------+----------------+----------------+----------------+--------+
                  |               |               |
                  |               |               |
            +----------+    +----------+    +----------+
            | IO Entry |    | IO Entry |    | IO Entry |  
            +----------+    +----------+    +----------+
            |ColumnData|    |ColumnData|    |ColumnData|
            +----------+    +----------+    +----------+
```
#### Read object
```
          +-----------------------------+
          |            Header           |
          +--------+-------------+------+  
          | ...... | MetaExtent  |......|
          +--------+-------------+------+
                          |
                          |
+--------------------------------------------------------------------+
|                             IO Entry                               |
+--------------------------------------------------------------------+
|        Meta(ObjectMeta/BlockMetaHeader/ColumnMeta/ZoneMap)         |
+--------+----------------+----------------+----------------+--------+
| Block  | <ColumnMeta-1> | <ColumnMeta-2> | <ColumnMeta-3> | ...... |
+--------+----------------+----------------+----------------+--------+
                  |               |               |
                  |               |               |
            +----------+    +----------+    +----------+
            | IO Entry |    | IO Entry |    | IO Entry |  
            +----------+    +----------+    +----------+
            |ColumnData|    |ColumnData|    |ColumnData|
            +----------+    +----------+    +----------+
```