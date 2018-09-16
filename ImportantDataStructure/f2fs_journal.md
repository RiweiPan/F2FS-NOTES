## F2FS Journal机制
### Journal机制的介绍
当F2FS进行文件读写的时候，根据 `f2fs_node` 的设计以及闪存设备异地更新的特性，每修改一个数据块，都需要改动 `f2fs_node` 的地址映射，以及NAT，SIT等信息。但是如果仅仅因为一个小改动，例如修改一个块，就需要改动这么多数据，然后再写入磁盘，这样既会导致性能下降，也会导致SSD寿命的下降。因此F2FS设计了journal机制，用于将这些对数据的修改会暂存在 `f2fs_journal` ，等待系统进行checkpoint的时候，再写入磁盘当中。
部分内容参考: https://blog.csdn.net/sunwukong54/article/details/45669017
### Journal涉及到的数据结构
```c
struct f2fs_journal {
	union {
		__le16 n_nats; /* 这个journal里面包含多少个nat_journal对象 */
		__le16 n_sits; /* 这个journal里面包含多少个sit_journal对象 */
	};
	/* spare area is used by NAT or SIT journals or extra info */
	union {
		struct nat_journal nat_j;
		struct sit_journal sit_j;
		struct f2fs_extra_info info;
	};
} __packed;
```
`f2fs_journal` 可以保存NAT的journal也可以保存SIT的journal，以下分别分析:

**NAT Journal**
NAT类型的journal主要保存的每一个node是属于哪一个inode，以及它的地址是什么，这样设计的原始访问某一个node的时候，只要根据nid找到对应的 `nat_journal_entry` ，然后就可以找到 `f2fs_nat_entry` ，最后找到blkaddr。
```c
struct nat_journal {
	struct nat_journal_entry entries[NAT_JOURNAL_ENTRIES];
	__u8 reserved[NAT_JOURNAL_RESERVED];
} __packed;

struct nat_journal_entry {
	__le32 nid;
	struct f2fs_nat_entry ne;
} __packed;

struct f2fs_nat_entry {
	__u8 version;		/* latest version of cached nat entry */
	__le32 ino;		/* inode number */
	__le32 block_addr;	/* block address */
} __packed;
```
**SIT Journal**
SIT类型的Journal和segment一一对应。segment管理着512个block，需要一种机制去记录每一个它所管理的block是否已经被分配出去。 通过 `f2fs_sit_entry` 可以发现，`f2fs_journal` 保存的是有效block的数目 `vblocks` 以及 它的bitmap `valid_map`。 
```c
struct sit_journal {
	struct sit_journal_entry entries[SIT_JOURNAL_ENTRIES];
	__u8 reserved[SIT_JOURNAL_RESERVED];
} __packed;

struct sit_journal_entry {
	__le32 segno;
	struct f2fs_sit_entry se;
} __packed;

struct f2fs_sit_entry {
	__le16 vblocks;				/* reference above */
	__u8 valid_map[SIT_VBLOCK_MAP_SIZE];	/* SIT_VBLOCK_MAP_SIZE = 64，64 * 8 = 512 可以表示每一个块的valid状态 */
	__le64 mtime;				/* segment age for cleaning */
} __packed;
```

### Journal一些机制的具体实现

#### 通过Journal获取Node的地址
```c
void f2fs_get_node_info(struct f2fs_sb_info *sbi, nid_t nid,
						struct node_info *ni)
{
	struct f2fs_nm_info *nm_i = NM_I(sbi);
	struct curseg_info *curseg = CURSEG_I(sbi, CURSEG_HOT_DATA);
	struct f2fs_journal *journal = curseg->journal;
	nid_t start_nid = START_NID(nid);
	struct f2fs_nat_block *nat_blk;
	struct page *page = NULL;
	struct f2fs_nat_entry ne;
	struct nat_entry *e;
	pgoff_t index;
	int i;

	ni->nid = nid;

	/* Check nat cache */
	down_read(&nm_i->nat_tree_lock);
	e = __lookup_nat_cache(nm_i, nid); // 从cache里面找nid
	if (e) { // 如果有就返回
		ni->ino = nat_get_ino(e);
		ni->blk_addr = nat_get_blkaddr(e);
		ni->version = nat_get_version(e);
		up_read(&nm_i->nat_tree_lock);
		return;
	}

	memset(&ne, 0, sizeof(struct f2fs_nat_entry)); // 初始化为0

	/* Check current segment summary */
	down_read(&curseg->journal_rwsem);
	i = f2fs_lookup_journal_in_cursum(journal, NAT_JOURNAL, nid, 0); // 从NAT_JOURNAL里面找这个nid在journal中的offset
	if (i >= 0) {
		ne = nat_in_journal(journal, i); // 将nat_entry返回出来
		node_info_from_raw_nat(ni, &ne); // 读到node_info中
	}
	up_read(&curseg->journal_rwsem);
	if (i >= 0) {
		up_read(&nm_i->nat_tree_lock);
		goto cache;
	}

	/*
	 * Fill node_info from nat page
	 * start_nid是根据nid找到管理这个nid的nat block偏移
	 * */
	index = current_nat_addr(sbi, nid);
	up_read(&nm_i->nat_tree_lock);

	page = f2fs_get_meta_page(sbi, index); // 从磁盘读取出f2fs_nat_block
	nat_blk = (struct f2fs_nat_block *)page_address(page);
	ne = nat_blk->entries[nid - start_nid];
	node_info_from_raw_nat(ni, &ne);
	f2fs_put_page(page, 1);
cache:
	/* cache nat entry */
	cache_nat_entry(sbi, nid, &ne); // 缓存这个node_entry
}
```

#### 通过Checkpoint将journal的信息写入到磁盘中
[具体参考Checkpoint的流程](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS-Checkpoint/Checkpoint%E6%B5%81%E7%A8%8B.md), 简略的流程如下: 
1. `f2fs_flush_nat_entries` 和 `f2fs_flush_sit_entries` 函数将entry都写入到 `curseg_info->f2fs_summary->journal` 的变量中。
2. `do_checkpoint函数` 读取 ``curseg_info->f2fs_summary` ，然后通过函数 `f2fs_write_node_summaries` 或 `f2fs_write_data_summaries` 刷写到磁盘中。