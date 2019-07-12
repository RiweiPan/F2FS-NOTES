# 文件数据的保存以及物理地址的映射
文件系统一个重要功能是实现文件的保存以及读写。每个文件系统有不同的保存和读写方式，而F2FS则是通过Node-Data的形式将数据组织起来。其中最重要的结构就是`f2fs_node`结构，它实现了文件数据的组织和管理。

## f2fs_node的结构以及作用
F2FS每一个文件，根据文件大小的不同，对应了一个或者多个`f2fs_node`结构，它实际存在于磁盘当中，并每一个`f2fs_node`结构都占据了4KB的磁盘空间，它的结构如下:
```c
struct f2fs_node {
	union {
		struct f2fs_inode i;
		struct direct_node dn;
		struct indirect_node in;
	};
	struct node_footer footer; // footer用于记录node的类型
} __packed;

struct node_footer {
	__le32 nid;		/* node id */
	__le32 ino;		/* inode nunmber */
	__le32 flag;		/* include cold/fsync/dentry marks and offset */
	__le64 cp_ver;		/* checkpoint version */
	__le32 next_blkaddr;	/* next node page block address */
} __packed;
```
其中`f2fs_inode`、`direct_node`、`indirect_inode`、`node_footer` 在 `f2fs_node` 中起到不同的作用:
1. **f2fs_inode:** 每一个文件仅存在一个`f2fs_inode`结构，用于提供给文件进行索引，保存了部分文件的元数据，以及923个页数据的物理地址，同时也保存了5个用于间接寻址的direct_node或indirect_node的node id(nid)，其中2个是direct_node的nid，2个是indirect_node的nid，以及1个double indirect node的nid;
2. **direct_node:** 用于直接寻址，保存了1018个页数据的物理地址;
3. **indirect_inode:** 用于间接寻址，保存了1018个direct_node的nid或者1018个indirect_inode的nid;
4. **node_footer:** 用于区分该f2fs_node的类型

`f2fs_inode`和`direct_inode`以及`indirect_inode`的结构如下所示:
```c
struct f2fs_inode {
	...
	// DEF_ADDRS_PER_INODE=923，因此f2fs_inode直接用一个数组保存了物理地址的值
	__le32 i_addr[DEF_ADDRS_PER_INODE];
	// DEF_NIDS_PER_INODE=5，因此同样是用一个数组直接存放了用于间接寻址的direct_node和indirect_node的nid
	__le32 i_nid[DEF_NIDS_PER_INODE];	
	...
} __packed;

struct direct_node {
	__le32 addr[ADDRS_PER_BLOCK]; // ADDRS_PER_BLOCK=1018，保存数据的物理地址
} __packed;

struct indirect_node {
	__le32 nid[NIDS_PER_BLOCK]; // NIDS_PER_BLOCK=1018，保存了用于间接寻址的nid
} __packed;
```

## 普通文件数据的保存
从上节描述可以知道，一个文件由一个`f2fs_inode`和多个`direct_inode`或者`indirect_inode`所组成。当系统创建一个文件的时候，它会首先创建一个`f2fs_inode`写入到磁盘，然后用户往该文件写入第一个page的时候，会将数据写入到main area的一个block中，然后将该block的物理地址赋值到`f2fs_inode->i_addr[0]`中，这样就完成了Node-Data的管理关系。随着对同一文件写入的数据的增多，会逐渐使用到其他类型的node去保存文件的数据。

根据`f2fs_inode`能够保存的数据的变量`f2fs_inode->i_addr`以及`f2fs_inode->i_nid`,可以计算F2FS最大允许单个文件的大小:
1. `f2fs_inode` 直接保存了923个页的数据的物理地址
2. `f2fs_inode->i_nid[0~1]` 保存了两个 `direct_node` 的地址，因此可以通过二级索引的方式访问数据，因此这里可以保存 2 x 1018个页的数据
3. `f2fs_inode->i_nid[2~3]` 保存了两个`indirect_node` 的地址，这两个其中2个`indirect_node`保存的是 `direct_node` 的nid，因此可以通过二级索引的方式访问数据，因此可以保存 2 x 1018 x 1018个页的数据;
4. `f2fs_inode->i_nid[4]` 保存了一个`indirect_node` 的地址，这个`indirect_node`保存的是 `indirect_node` 的nid，因此可以通过三级索引的方式访问数据，因此可以保存 1018 x 1018 x 1018个页的数据

可以得到如下计算公式: 
**4KB x (923 + 2 x 1018 + 2 x 1018 x 1018 + 1 x 1018 x 1018 x 1018) = 3.93TB**
因此F2FS单个文件最多了保存3.93TB数据。

## 内联文件数据的保存
从上节可以知道，文件的实际数据是保存在`f2fs_inode->i_addr`对应的物理块当中，因此即使一个很小的文件，如1个字节的小文件，也需要一个node和data block才能实现正常的保存和读写，也就是需要8KB的磁盘空间去保存一个尺寸为1字节的小文件。而且`f2fs_inode->i_addr[923]`里面除了`f2fs_inode->i_addr[0]`保存了一个物理地址，其余的922个i_addr都被闲置，造成了空间的浪费。

因此F2FS为了减少空间的使用量，使用内联(inline)文件减少这些空间的浪费。它的核心思想是当文件足够小的时候，使用`f2fs_inode->i_addr`数组直接保存数据本身，而不单独写入一个block中，再进行寻址。因此，如上面的例子，只有1个字节大小的文件，只需要一个`f2fs_inode`结构，即4KB，就可以同时将node信息和data信息同时保存，减少了一半的空间使用量。

根据上述定义，可以计算得到多大的文件可以使用内联的方式进行保存，`f2fs_inode`有尺寸为923的用于保存数据物理地址的数组i_addr，它的数据类型是__le32，即4个字节。保留一个数组成员另做它用，因此内联文件最大尺寸为: 922 * 4 = 3688字节。

## 文件读写与物理地址的映射

**文件的组织方式以及一般情况下的读写流程**
Linux的文件是通过page进行组织起来的，默认page的size是4KB，使用index作为编号。下面是一个size=10KB的文件的组织方式:
```
 +-----------------------+
  | page index0         |
  |------------------------|
  | page index1         |
  |------------------------|
  | page index2         |
 +-----------------------+
```
当用户写入/访问2~6kb的数据的时候，系统就会计算出数据保存在page index = 0和1的page中，然后将inode和page index传入到文件系统中，然后文件系统通过inode和page index找到对应物理地址，将数据写入磁盘或者从磁盘中读取数据出来。

**F2FS的物理地址寻址流程**
上面提及到，VFS会将inode和page index的信息传入到文件系统找到对应的物理地址后，进行写入或者读取。下面通过一个例子，描述F2FS如何根据inode和page index找到物理地址:

假设用户需要读取文件第4000个页(page index = 3999)的数据，
第一步: 那么首先系统会根据inode找到对应的f2fs_inode结构
第二步: 由于4000 >（923 + 1018 + 1018），`f2fs_inode->i_addr`和`f2fs_inode->nid[0]和nid[1]`都无法满足需求，因此系统根据`f2fs_inode->nid[2]`找到对应的 `indirect_node`的地址
第三步: `indirect_node`保存的是`direct_node`的nid数组，由于 4000 - 923 - 1018 - 1018 = 1041，而一个`direct_node`只能保存1018个block，因此可以知道数据位于`indirect_node->nid[1]`对应的`direct_node`中
第四步: 计算剩下的的偏移(4000-923-1018-1018-1018=23)找到数据的物理地址位于该`direct_node`的`direct_node->addr[23]`中。

F2FS中常用函数`f2fs_get_dnode_of_data`完成这一步骤，可以参考[物理地址寻址的实现](https://github.com/RiweiPan/F2FS-NOTES/blob/master/ImportantDataStructure/get_dnode.md) 。







