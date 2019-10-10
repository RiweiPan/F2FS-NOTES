## seg_entry 和 sit_info

### seg_entry结构
#### 介绍
因为每一个segment需要管理512个Block的地址，因此需要通过某种方式去标记一个segment下的block，哪些是已经使用的，哪些block是处于无效状态等待回收。在F2FS中，通过结构体 `seg_entry` 去管理一个segment下的所有block的使用信息:

```c
struct seg_entry {
	unsigned int type:6;		/* 这个segment的类型 */
	unsigned int valid_blocks:10;	/* 已经使用的块的数目 */
	unsigned int ckpt_valid_blocks:10;	/* 上一次执行CP时，使用的块的数目 */
	unsigned int padding:6;		/* padding */
	unsigned char *cur_valid_map;	/* 通过bitmap(512位)表示这个segment哪些被使用，哪些没使用 */
#ifdef CONFIG_F2FS_CHECK_FS
	unsigned char *cur_valid_map_mir;	/* mirror of current valid bitmap */
#endif
	/*
	 * # of valid blocks and the validity bitmap stored in the the last
	 * checkpoint pack. This information is used by the SSR mode.
	 */
	unsigned char *ckpt_valid_map;	/* 上次CP时的bitmap状态 */
	unsigned char *discard_map; /* 标记哪些block需要discard的bitmap */
	unsigned long long mtime;	/* 修改时间 */
};
```

`seg_entry` 由于跟磁盘空间大小有关，因此初始化时以动态分配的方式，保存在元数据区域的**SIT**区域当中，代码的具体实现为`sbi->sit_info->sentries` 中。

#### 应用场景
**写流程:** 当文件的修改某一个block的数据时，需要经过的流程是：
1) 分配一个新的block; 
2) 将数据写入到新分配的block中; 
3) 将旧block置为无效，等待回收; 
4) 将新block写入到磁盘中。

这一个流程需要更新的segment的管理信息，因为新block和旧block可能来自不同的segment，因此需要更新segment的统计信息，具体流程是: 根据新block的地址，找到对应segment number和seg_entry，然后在 `seg_entry `的根据新block在segment的bitmap对应位置设为1，然后给 `seg_entry->valid_blocks` 加一，表示这个segment新增加了一个被使用block;对于旧block，一样是根据block地址找到segment number和seg_entry，然后执行相反操作对bitmap设为0，然后 `seg_entry->valid_blocks` 减一。

### curseg_info结构
#### 介绍
`curseg_info` 在F2FS中表示的是当前使用的segment的信息。一般情况下，F2FS同时运行着6个 `curseg_info` ，分别表示**(NODE,DATA) X (HOT,WARM,COLD)**这些不同类型的segment。 它的基本结构和关联数据结构是:
```c
struct curseg_info {
	struct mutex curseg_mutex;		/* lock for consistency */
	struct f2fs_summary_block *sum_blk;	/* cached summary block */
	struct rw_semaphore journal_rwsem;	/* protect journal area */
	struct f2fs_journal *journal;		/* cached journal info */
	unsigned char alloc_type;		/* current allocation type */
	unsigned int segno;			/* current segment number */
	unsigned short next_blkoff;		/* next block offset to write */
	unsigned int zone;			/* current zone number */
	unsigned int next_segno;		/* preallocated segment */
};
```
它的主要成员的含义是:
**f2fs_summary_block:** `curseg_info` 表示一个segment，因此通过 `f2fs_summary_block` 管理这个segment下的所有block。 `f2fs_summary_block` 包含512个 `f2fs_summary`，每个summary代表一个这个segment里面的一个block，它的结构是:
```c
struct f2fs_summary_block {
	struct f2fs_summary entries[ENTRIES_IN_SUM]; /* ENTRIES_IN_SUM = 512 表示被管理的512个块 */
	struct f2fs_journal journal;
	struct summary_footer footer; /* 指示这个segment的类型 */
} __packed;

struct f2fs_summary {
	__le32 nid;		/* 属主node id */
	union {
		__u8 reserved[3];
		struct {
			__u8 version;		/* node version number */
			__le16 ofs_in_node;	/* 属主node里面的第几个block */
		} __packed;
	};
} __packed;

```
可以看到每一个 `f2fs_summary` 用来描述这个segment里面的block是属于哪一个node，而且是这个node里面的第几个block。

**f2fs_journal:** `curseg_info` 管理着512个block，需要一种机制去记录每一个它所管理的block是否已经被分配出去。 因此 `f2fs_journal` 的作用就是记录每一个block是否是有效。它的结构如下:
```c
struct f2fs_journal {
	union {
		__le16 n_nats;
		__le16 n_sits; /* 这个journal里面包含多少个sit_journal对象 */
	};
	/* spare area is used by NAT or SIT journals or extra info */
	union {
		struct nat_journal nat_j;
		struct sit_journal sit_j;
		struct f2fs_extra_info info;
	};
} __packed;

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
`f2fs_journal` 可以记录NAT和SIT的journal，这一节只关注SIT的作用。 通过 `f2fs_sit_entry` 可以发现，`f2fs_journal` 保存的是有效block的数目 `vblocks` 以及 它的bitmap `valid_map`。 

#### curseg_info的作用
`curseg_info` 的作用主要是当一个Node或者Data需要分配一个新的block的时候，就会根据这个block的类型，在 `curseg_info` 取出一个segment，然后在这个segment分配出一个新的block，然后将新的block的映射信息，写入 `curseg_info` 的 `f2fs_summary_block` 和 `f2fs_journal` 中。这样设计的原因是，将大部分更新元数据的操作都放在 `curseg_info` 完成，避免了频繁读写磁盘。
