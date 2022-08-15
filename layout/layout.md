# Layout

## Storage File Format

![Layout.jpg](image/Layout.jpg)

### Header
```
+---------+------------+--------------+
|Magic(8B)| Version(2B)| Reserved(22B)|
+---------+------------+--------------+

Magic = Engine identity (0x01346616). TAE only
Version = Object file version
Reserved = 22 bytes reserved space
```
### Meta

##### Block Meta Header
```
+---------------+---------------+----------------+----
| <BlockMeta-1> | <BlockMeta-2> |  <BlockMeta-3> |....
+---------------+---------------+----------------+----
       |
       |
+-------------------------------------------------------------------------------------------------------+
|                                                 Header                                                |
+-------------+---------------+-------------+----------+---------------+------------+-------------------+
| TableID(8B) | SegmentID(8B) | BlockID(8B) | Algo(1B) | ColumnCnt(2B) | Chksum(4B) |  Reserved(33B)    |
+-------------+---------------+-------------+----------+---------------+------------+-------------------+
|                                                ColumnMeta                                             |
+-------------------------------------------------------------------------------------------------------+
|                                                ColumnMeta                                             |
+-------------------------------------------------------------------------------------------------------+
|                                                ColumnMeta                                             |
+-------------------------------------------------------------------------------------------------------+
|                                                ..........                                             |
+-------------------------------------------------------------------------------------------------------+
Header Size = 64B
TableID = Table ID for Block
SegmentID = Segment ID
BlockID = Block ID
Algo = Type of compression algorithm for block data
ColumnCnt = The number of column in the block
Chksum = Block metadata checksum
Reserved = 41 bytes reserved space
```
##### Column Meta
```
+---------------------------------------------------------------------------------------------------------------+
|                                                    ColumnMeta                                                 |
+--------+-------+----------+--------+---------+--------+--------+------------+---------+-----------+-----------+
|Type(1B)|Idx(2B)|Offset(4B)|Size(4B)|oSize(4B)|Min(32B)|Max(32B)|BFoffset(4b)|BFlen(4b)|BFoSize(4B)|Chksum(4B) |
+--------+-------+----------+--------+---------+--------+--------+------------+---------+-----------+-----------+
|                                                    Reserved(33B)                                              |
+---------------------------------------------------------------------------------------------------------------+
ColumnMeta Size = 128B
Type = Metadata type, always 0, representing column meta, used for extension.
Idx = Column index
Offset = Offset of column data
Size = Size of column data
oSize = Original data size
Min = Column min value
Max = Column Max value
BFoffset = Bloomfilter data offset
Bflen = Bloomfilter data size
BFoSize = Bloomfilter original data size
Chksum = Data checksum
Reserved = 33 bytes reserved space
```
##### Foot
```
+------------+------------+-------------+----+--------------+----------------+------------+
| <Extent-1> | <Extent-2> |  <Extent-3> |....| MetaAlgo(1B) | ExtentsSize(4B)| Magic (8B) |
+------------+------------+-------------+----+--------------+----------------+------------+
        |
        |
+----------------+--------------+-----------------+
| MetaOffset(4B) | MetaSize(4B) |  MetaOSize (4B) |
+----------------+--------------+-----------------+
Magic = Engine identity (0x01346616). TAE only
ExtentsSize = The size of the meta extents area in the foot.
MetaAlgo = Type of compression algorithm for block metadata
```
##### Extent
```
An extent records the address of a block's metadata in the object
MetaOffset = Offset of block metadata
MetaSize = Size of block metadata
MetaOSize = Original metadata size
```
### IO Path
##### Read block
```
          +-------------------+
          |      MetaInfo     |
          +-------------------+                   
                    |
                    |
+--------------------------------------------------------------------+
|                             IO Entry                               |
+--------------------------------------------------------------------+
|                             BlockMeta                              |
+--------+----------------+----------------+----------------+--------+
| Header | <ColumnMeta-1> | <ColumnMeta-2> | <ColumnMeta-3> | ...... |
+--------+----------------+----------------+----------------+--------+
                  |               |               |
                  |               |               |
            +----------+    +----------+    +----------+
            | IO Entry |    | IO Entry |    | IO Entry |  
            +----------+    +----------+    +----------+
            |ColumnData|    |ColumnData|    |ColumnData|
            +----------+    +----------+    +----------+
```
##### Read object
```
          +-------------------------------+
          |             Foot              |
          +----------+-------------+------+  
          | MetaAlgo | ExtentsSize | Magic|
          +----------+-------------+------+
                          |
                          |
+------------------------------------------------------------+
|                         IO Entry                           |
+------------------------------------------------------------+
|                         Extent Area                        |
+------------+------------+------------+------------+--------+
| <Extent-1> | <Extent-2> | <Extent-3> | <Extent-4> | ...... |
+------------+------------+------------+------------+--------+
        |
        |
+--------------------------------------------------------------------+
|                             IO Entry                               |
+--------------------------------------------------------------------+
|                             BlockMeta                              |
+--------+----------------+----------------+----------------+--------+
| Header | <ColumnMeta-1> | <ColumnMeta-2> | <ColumnMeta-3> | ...... |
+--------+----------------+----------------+----------------+--------+
                  |               |               |
                  |               |               |
            +----------+    +----------+    +----------+
            | IO Entry |    | IO Entry |    | IO Entry |  
            +----------+    +----------+    +----------+
            |ColumnData|    |ColumnData|    |ColumnData|
            +----------+    +----------+    +----------+
```