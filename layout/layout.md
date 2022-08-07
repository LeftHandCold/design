# Layout

## Storage File Format

![Layout.jpg](image/Layout.jpg)

### Header
```
+---------+------------+---------+-----------+----------------+-------------+------------+--------------+
|Magic(8B)| Version(2B)| Size(4B)| Chksum(4B)| MetaOffset (4B)| MetaLen (4B)| MetaCnt(2B)| Reserved(36B)|
+---------+------------+---------+-----------+----------------+-------------+------------+--------------+

Magic = Engine identity (0x01346616). TAE only
Version = Object file version
Size = Header size
Chksum = Header CheckSum(This field is 0 when calculating)
MetaOffset = The offset of Block metadata
MetaLen = The length of Block metadata
MetaCnt = The number of block metadata in Object
Reserved = 36 bytes reserved space
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
+------------+-----------+-----------+--------+----------+-------------+------------+-------------------+
| Version(2B)|TableID(8B)|BlockID(8B)|Algo(1B)|Offset(4B)|MateCnt(128B)|MateSize(4B)|   Reserved(33B)   |
+------------+-----------+-----------+--------+----------+-------------+------------+-------------------+
|                                                ColumnMeta                                             |
+-------------------------------------------------------------------------------------------------------+
|                                                ColumnMeta                                             |
+-------------------------------------------------------------------------------------------------------+
|                                                ColumnMeta                                             |
+-------------------------------------------------------------------------------------------------------+
|                                                ..........                                             |
+-------------------------------------------------------------------------------------------------------+
Header Size = 64B
Version = Meta version
TableID = Table ID for Block
BlockID = Block ID
Algo = Compression algorithm
Offset = Metadata offset of column in block
MateCnt = The number of metadata of the column in the block
MateSize = Column metadata Size (128b)
Reserved = 33 bytes reserved space
```
##### Column Meta
```
+---------------------------------------------------------------------------------------------------------------+
|                                                    ColumnMeta                                                 |
+--------+-------+----------+-------+---------+--------+--------+------------+---------+-----------+------------+
|Type(1B)|Idx(2B)|Offset(4B)|Len(4B)|oSize(4B)|Min(32B)|Max(32B)|BFoffset(4b)|BFlen(4b)|BFoSize(4B)|Chksum(4B)  |
+--------+-------+----------+-------+---------+--------+--------+------------+---------+-----------+------------+
|                                                    Reserved(33B)                                              |
+---------------------------------------------------------------------------------------------------------------+
ColumnMeta Size = 128B
Type = Metadata type, always 0, representing column meta, used for extension.
Idx = Column index
Offset = Offset of column data
Len = Size of column data
oSize = Original data size
Min = Column min value
Max = Column Max value
BFoffset = Bloomfilter data offset
Bflen = Bloomfilter data size
BFoSize = Bloomfilter original data size
Chksum = Data checksum
Reserved = 33 bytes reserved space
```