# F2FS物理地址寻址的实现
VFS的读写都依赖于物理地址的寻址。经典的读流程，VFS会传入inode以及page index信息给文件系统，然后文件系统需要根据以上信心，找到物理地址，然后访问磁盘将其读取出来。F2FS的物理地址寻址，是通过`f2fs_get_dnode_of_data`函数实现。

在执行这个`f2fs_get_dnode_of_data`函数之前，需要通过`set_new_dnode`函数进行对数据结构`struct dnode_of_data`进行初始化:
```c
struct dnode_of_data {
	struct inode *inode;		/* VFS inode结构 */
	struct page *inode_page;	/* f2fs_inode对应的node page */
	struct page *node_page;		/* 用户需要访问的物理地址所在的node page，有可能跟inode_page一样*/
	nid_t nid;			/* 用户需要访问的物理地址所在的node的nid，与上面的node_page对应*/
	unsigned int ofs_in_node;	/* 用户需要访问的物理地址位于上面的node_page对应的addr数组第几个位置 */
	bool inode_page_locked;		/* inode page is locked or not */
	bool node_changed;		/* is node block changed */
	char cur_level;			/* 当前node_page的层次，按直接访问或者简介访问的深度区分 */
	char max_level;			/* level of current page located */
	block_t	data_blkaddr;		/* 用户需要访问的物理地址 */
};

static inline void set_new_dnode(struct dnode_of_data *dn, struct inode *inode,
		struct page *ipage, struct page *npage, nid_t nid)
{
	memset(dn, 0, sizeof(*dn));
	dn->inode = inode;
	dn->inode_page = ipage;
	dn->node_page = npage;
	dn->nid = nid;
}
```
大部分情况下，仅需要传入inode进行初始化:
```c
set_new_dnode(&dn, inode, NULL, NULL, 0); // 0表示不清楚nid
```
然后根据需要访问的page index，执行`f2fs_get_dnode_of_data`函数寻找:
```c
err = f2fs_get_dnode_of_data(&dn, page->index, type); // type类型影响了寻址的行为
blockt blkaddr = dn.data_blkaddr; // 获得对应位置的物理地址信息
```
接下来分析，函数是如何寻址，由于函数比较长和复杂，先分析一个比较重要的函数`get_node_path`函数的作用，它的用法是:
#### 概念
在分析之前，我们要明确几个概念。f2fs有三种node的类型，`f2fs_inode`，`direct_node`，和`indirect node`。其中`f2fs_inode`和`direct_node`都是直接保存数据的地址指针，因此一般统称为direct node，若有下横线，例如`direct_node`，则表示数据结构`struct direct_node`，如果没有下横线，则表示直接保存数据的地址指针的node，即`f2fs_inode`和`direct_node`。另外`indirect node`保存的是间接寻址的node的nid，因此一般直接称为为indirect node。
#### 函数用法
```c
int level;
int offset[4];
unsigned int noffset[4];
level = get_node_path(inode, page->index, offset, noffset);
```
这里offset和noffset分别表示block offset和node offset，返回的level表示寻址的深度，一共有4个深度，使用0~3表示:
level=0: 表示可以直接在`f2fs_inode`找到物理地址
level=1: 表示可以在`f2fs_inode->i_nid[0~1]`对应的`direct_node`能够找到物理地址
level=2: 表示可以在`f2fs_inode->i_nid[2~3]`对应的`indirect_node`下的nid对应的`direct_node`能够找到物理地址
level=3: 表示只能在`f2fs_inode->i_nid[4]`对应`indirect_node`的nid对应的`indirect_node`的nid对应的`direct_node`才能找到地址

由于offset和noffset，表示的是物理地址寻址信息，分别表示block偏移和direct node偏移来表示，它们是长度为4的数组，代表不同level 0~3 的寻址信息。之后的函数可以通过offset和noffset将数据块计算出来。
#### 寻址原理
给定page->index，计算出level之后，offset[level]表示该page在所对应的direct node里面的block的偏移，noffset[level]表示当前的node是属于这个文件的第几个node(包括f2fs_node, direct_node, indirect_node)，下面用几个例子展示一下 (注意下面计算的是不使用xattr的f2fs版本，如果使用了xattr结果会不同，但是表示的含义是一样的):

**例子1: 物理地址位于f2fs_inode**
例如我们要寻找page->index = 665的数据块所在的位置，显然655是位于`f2fs_inode`内，因此level=0，因此我们只需要看offset[0]以及noffset[0]的信息，如下图。offset[0] = 665表示这个数据块在当前direct node(注意: f2fs_inode也是direct node的一种)的位置；noffset[0]表示当前direct node是属于这个文件的第几个node，由于f2fs_inode是第一个node，所以noffset[0] = 0。
```c
level = 0 // 可以直接在f2fs_inode找到物理地址
offset[0] = 665 // 由于level=0，因此我们只需要看offset[level]=offset[0]的信息，这里offset[0] = 665表示地址位于f2fs_inode->i_addr[665]
noffset[0] = 0 // 对于level=0的情况，即看noffset[0]，因为level=0表示数据在唯一一个的f2fs_inode中，因此这里表示inode。
```

**例子2: 物理地址位于direct_node**
例如我们要寻找page->index = 2113的数据块所在的位置，它位于第二个direct_node，所以level=1。我们只需要看offset[1]以及noffset[1]的信息，如下图。offset[1] = 172表示这个数据块在当前direct node的位置，即`direct_node->addr[172]`；noffset[1]表示当前direct node是属于这个文件的第几个node，由于它位于第二个direct_node，前面还有一个f2fs_inode以及一个direct node，所以这是第三个node，因此noffset[1] = 2。
```c
level = 1 // 表示可以在f2fs_inode->i_nid[0~1]对应的direct_node能够找到物理地址
offset[1] = 172 // 表示物理地址位于对应的node page的i_addr的第172个位置中，即direct_node->addr[172]
noffset[1] = 2 // 数据保存在总共第三个node中 (1个f2fs_inode，2个direct_node)
```

**例子3: 物理地址位于indirect_node**
例如我们要寻找page->index = 4000的数据块所在的位置，它位于第1个indirect_node的第2个direct_node中，所以level=2。我们只需要看offset[2]以及noffset[2]的信息，如下图。offset[2] = 23表示这个数据块在当前direct node的位置；noffset[2]表示当前direct node是属于这个文件的第几个direct node，即这是第6个node。(1 * f2fs_inode + 2 * direct_node + 1 * indirect_node + 2 * direct node)。

```c
offset[2] = 23
noffset[2] = 5
```

**例子4: 物理地址位于indirect_node再indiret_node中 (double indirect node)**
例如我们要寻找page->index = 2075624的数据块所在的位置，它位于第一个double indirect_node的第一个indirect_node的第一个direct_node中，所以level=3。同理我们只需要看offset[3]以及noffset[3]的信息，如下，可以自己计算一下:
```c
offset[3] = 17
noffset[3] = 2043
```

从上面可以知道`get_node_path`函数以后，执行可以根据offset和noffset直接知道page->index对应的物理地址，位于第几个node page的第几个offset对应的物理地址中。下面分析`f2fs_get_dnode_of_data`的原理:

```c
int f2fs_get_dnode_of_data(struct dnode_of_data *dn, pgoff_t index, int mode)
{
	struct f2fs_sb_info *sbi = F2FS_I_SB(dn->inode);
	struct page *npage[4];
	struct page *parent = NULL;
	int offset[4];
	unsigned int noffset[4];
	nid_t nids[4];
	int level, i = 0;
	int err = 0;
	
	// 通过计算得到offset, noffset，从而知道位于第几个node page的第几个offset对应的物理地址中
	level = get_node_path(dn->inode, index, offset, noffset);
	if (level < 0)
		return level;

	nids[0] = dn->inode->i_ino;
	npage[0] = dn->inode_page;

	if (!npage[0]) {
		npage[0] = f2fs_get_node_page(sbi, nids[0]); // 获取inode对应的f2fs_inode的node page
	}

	parent = npage[0];
	if (level != 0)
		nids[1] = get_nid(parent, offset[0], true); // 获取f2fs_inode->i_nid
		
	dn->inode_page = npage[0];
	dn->inode_page_locked = true;

	for (i = 1; i <= level; i++) {
		bool done = false;

		if (!nids[i] && mode == ALLOC_NODE) { 
			// 创建模式，常用，写入文件时，需要node page再写入数据，因此对于较大文件，在这里创建node page
			if (!f2fs_alloc_nid(sbi, &(nids[i]))) { // 分配nid
				err = -ENOSPC;
				goto release_pages;
			}

			dn->nid = nids[i];
			npage[i] = f2fs_new_node_page(dn, noffset[i]); // 分配node page
			//  如果i == 1，表示f2fs_inode->nid[0~1]，即direct node，直接赋值到f2fs_inode->i_nid中
			//  如果i != 1，表示parent是indirect node类型的，要赋值到indirect_node->nid中
			set_nid(parent, offset[i - 1], nids[i], i == 1); 
			f2fs_alloc_nid_done(sbi, nids[i]);
			done = true;
		} else if (mode == LOOKUP_NODE_RA && i == level && level > 1) {
			// 预读模式，少用，将node page全部预读出来
			npage[i] = f2fs_get_node_page_ra(parent, offset[i - 1]);
			done = true;
		}
		if (i == 1) {
			dn->inode_page_locked = false;
			unlock_page(parent);
		} else {
			f2fs_put_page(parent, 1);
		}

		if (!done) {
			npage[i] = f2fs_get_node_page(sbi, nids[i]); // 根据nid获取node page
		}
		if (i < level) {
			parent = npage[i]; // 注意这里parent被递归地赋值，目的是处理direct node和indrect node的赋值问题
			nids[i + 1] = get_nid(parent, offset[i], false); // 计算下一个nid
		}
	}
	// 全部完成后，将结果赋值到dn，然后退出函数
	dn->nid = nids[level];
	dn->ofs_in_node = offset[level];
	dn->node_page = npage[level];
	dn->data_blkaddr = datablock_addr(dn->inode, dn->node_page, dn->ofs_in_node); // 这个就是根据page index所得到的物理地址
	return 0;
}
```
