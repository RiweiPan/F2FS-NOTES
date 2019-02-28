## 文件数据的保存
F2FS的数据是通过node结构组织起来，每一个文件实体包含了至少一个node，这个node记录了具体数据保存在磁盘空间的物理地址。

当系统需要访问文件的某一个page的时候，系统首先会根据文件描述符找到对应的node结构，然后根据page index从node中找到对应的物理地址，然后根据物理地址访问磁盘读取数据。因此node结构的主要作用是根据逻辑地址索引物理地址。

### f2fs_node的结构以及作用
每一个文件都会包含至少一个 `f2fs_node` 结构，它的结构如下：

```c
struct f2fs_node {
	union {
		struct f2fs_inode i;
		struct direct_node dn;
		struct indirect_node in;
	};
	struct node_footer footer; // footer用于记录node的类型
} __packed;
```

`f2fs_node` 结构大小是4KB，单独占用磁盘4KB的空间，因此f2fs_node也会就有自己的物理地址。

其中`f2fs_inode`、`direct_node`、`indirect_inode`、`node_footer` 在 `f2fs_node` 中起到不同的作用:
>**f2fs_inode:** 用于提供给文件进行索引，保存了部分文件的元数据，以及923个页数据的物理地址，同时也保存了5个用于间接寻址的direct_node或indirect_node的node id(nid)，其中2个是direct_node的nid，2个是indirect_node的nid，以及1个double indirect node的nid;
>**direct_node:** 用于直接寻址，保存了1018个页数据的物理地址;
>**indirect_inode:** 用于间接寻址，保存了1018个direct_node的nid或者1018个indirect_inode的nid;
>**node_footer:** 用于区分该f2fs_node的类型

因此我们可以计算F2FS最大允许单个文件大小，我们根据以下步骤计算：
1. `f2fs_inode` 直接保存了923个页的数据的物理地址;
2. `f2fs_inode` 保存了两个 `direct_node` 的地址，因此可以通过二级索引的方式访问数据，因此这里可以保存 2 x 1018个页的数据;
3. `f2fs_inode` 保存了三个 `indirect_node` 的nid，其中2个保存了 `direct_node` 的nid，因此可以通过二级索引的方式访问数据，因此可以保存 2 x 1018 x 1018个页的数据; 剩下的一个保存的是 `indirect_node` 的nid，因此可以通过三级索引的方式访问数据，这里可以保存 1018 x 1018 x 1018个页的数据，因此我们得到如下计算公式: 
**4KB x (923 + 2 x 1018 + 2 x 1018 x 1018 + 1 x 1018 x 1018 x 1018) = 3.93TB**
因此F2FS单个文件最多了保存3.93TB数据。

### f2fs_inode/direct_inode/indirect_inode的结构以及作用
上面提及到文件数据是通过 `f2fs_inode` 索引数据，它数据结构如下:

```c
struct f2fs_inode {
	...
	__le32 i_addr[DEF_ADDRS_PER_INODE];	// DEF_ADDRS_PER_INODE=923，因此f2fs_inode直接用一个数组保存了物理地址的值
	__le32 i_nid[DEF_NIDS_PER_INODE];	// DEF_NIDS_PER_INODE=5，因此同样是用一个数组直接存放了用于间接寻址的direct_node和indirect_node的地址
	...
} __packed;

struct direct_node {
	__le32 addr[ADDRS_PER_BLOCK]; // ADDRS_PER_BLOCK=1018，保存数据的物理地址
} __packed;

struct indirect_node {
	__le32 nid[NIDS_PER_BLOCK]; // NIDS_PER_BLOCK=1018，保存了用于间接寻址的nid
} __packed;
```
其中DEF_ADDRS_PER_INODE、ADDRS_PER_BLOCK、NIDS_PER_BLOCK这几个宏的数值并不是随意得出，而是经过计算出来。F2FS规定一个 `f2fs_node` 尺寸为4KB，对应磁盘上一个page，所以被称为node page。其中它的变量note_footer占用了24个字节，因此可供给 `f2fs_inode`、`direct_node`、`indirect_inode` 使用的空间有4072个字节。而其中的 `f2fs_inode` 元数据占用了360个字节，5个间接索引的nid占用了20个字节，因此可用空间是3692个字节，然后除以物理地址长度4字节，就得出了923这个结果。对于 `direct_node`、`indirect_inode` ，它们所有的空间都可以直接用来储存，因此4072个字节，除以4个字节得到1018这个结果。


下面通过一个例子展示F2FS是如何通过这个数据结构找到想要的数据：
>假设用户需要读取文件第4000个页的数据，那么首先系统会根据文件描述符找到对应的f2fs_inode结构，由于4000 >（923 + 1018 + 1018），直接寻址和二级寻址(f2fs_inode->nid[0]和nid[1])都无法满足需求，因此系统根据f2fs_inode->nid[2]找到对应的 `indirect_node` ，然后通过计算可以得到数据位于 `indirect_node` 第二个node中，因此又根据 indirect_node->nid[1] 访问对应的 `direct_node`，最后通过计算剩下的偏移(4000-923-1018-1018-1018=23)找到对应的数据。



