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

由于offset和noffset，表示的是物理地址寻址信息，它的表示比较复杂，分别使用block offset和node offset来表示，因此用3个例子来描述

**例子1: 物理地址位于f2fs_inode**
如page->index = 665的page，它位于`f2fs_inode->i_addr[665]`对应的物理地址处，它的输出值为:
```c
level = 0 // 可以直接在f2fs_inode找到物理地址
offset[0] = 665 // 表示位于f2fs_inode->i_addr[665]
// 以下3个offset没什么用，因为用不到，也没有初始化，都是随机值
offset[1] = 32592 
offset[2] = 0 
offset[3] = 0
noffset[0] = 0 // 对于level=0的情况，noffset没什么用
// 以下3个noffset没什么用，因为用不到，也没有初始化，都是随机值
noffset[1] = 21862 
noffset[2] = 1894499744 
noffset[3] = 21862
```

**例子2: 物理地址位于direct_node**
如page->index = 2113的page，它位于`f2fs_inode->i_nid[1]`=>`direct_node->addr[172]`对应的物理地址处，它的输出值为:
```c
level = 1 // 表示可以在f2fs_inode->i_nid[0~1]对应的direct_node能够找到物理地址
offset[0] = NODE_DIR2_BLOCK // 除了level=0以外，offset[0]都是表示同一个level的不同node，如这里DIR2表示这是第二个direct_node，即f2fs_inode->i_nid[1]
offset[1] = 172 // 表示物理地址位于对应的node page的i_addr的第172个位置中，即direct_node->addr[172]
// 以下2个offset没什么用，因为用不到，也没有初始化，都是随机值
offset[2] = 0
offset[3] = 0
noffset[0] = 0 // 默认是0，没什么用
noffset[1] = 2 // 表示除f2fs_inode以外的第二个node page，即f2fs_inode->i_nid[1]对应的direct_node
// 以下2个noffset没什么用，因为用不到，也没有初始化，都是随机值
noffset[2] = 2694272416
noffset[3] = 21990
```

**例子3: 物理地址位于indirect_node**
如page->index = 4000的page，位于`f2fs_inode->i_nid[2]`=>`indirect_node->nid[1]`=>`direct_node->addr[23]`当中，它的level以及offset和noffset的值如下:
level = 2 # 由于是f2fs_inode->i_nid[3]完成了寻址，因此level=2
```c
offset[0] = NODE_IND1_BLOCK // 除了level=0以外，offset[0]都是表示同一个level的不同node，如这里IND1表示这是第一个indirect_node，即f2fs_inode->i_nid[3]
offset[1] = 1  // 表示indirect_node的偏移，即indirect_node->nid[1]对应的nid
offset[2] = 23 // 上面找到对应的indirect_node->nid[1]的nid后，对应的direct_node->addr[23]
offset[3] = 0
noffset[0] = 0 // 默认是0，没什么用
noffset[1] = 3 // 暂时不清楚
noffset[2] = 5 // 一共用了多少个node page，f2fs_inode + 2 * direct_node + 1 * indirect_node + 1 * indirect_node->direct_node = 5
noffset[3] = 21909
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
