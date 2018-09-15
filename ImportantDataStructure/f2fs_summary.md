## f2fs_summary 和 f2fs_summary_block

### 介绍
因为每一个segment需要管理512个Block的地址，而且很多场合需要通过block地址找到这个block是属于哪一个node，以及属于这个node的第几个block。 `f2fs_summary` 主要保存了block->node的映射信息：

```c
struct f2fs_summary {
	__le32 nid;		/* parent node id */
	union {
		__u8 reserved[3];
		struct {
			__u8 version;		/* node version number */
			__le16 ofs_in_node;	/* block index in parent node */
		} __packed;
	};
} __packed;
```
一个segment对应的512个 `f2fs_summary` 是通过一个4KB的block保存，`f2fs_summary_block` 保存在元数据区域的**SSA**区域: 

```c
struct f2fs_summary_block {
	struct f2fs_summary entries[ENTRIES_IN_SUM];
	struct f2fs_journal journal;
	struct summary_footer footer;
} __packed;

struct summary_footer {
	unsigned char entry_type;	/* SUM_TYPE_XXX */
	__le32 check_sum;		/* summary checksum */
} __packed;
```

其中 `summary_footer` 记录了这个 `f2fs_summary_block` 的一些属性，如校验信息，以及这个 `f2fs_summary_block` 对应的segment所管理的block是属于node还是data。

### 应用场景

#### GC流程
1. **GC基本流程:** 选一个无效block最多的当选择出需要gc的victim segment，然后将这个victim segment的block迁移插入到其他segment中，这样就可以制造出一个全部block都可以用的segment。
2. **f2fs_summary在GC的作用:** 当选择出需要gc的victim segment之后，可以通过这个victim segment的segno，在SSA区域找到 `f2fs_summary_block`。对victim segment的每一个block进行迁移的时候，会根据block的地址在 `f2fs_summary_block` 找到 它所对应的`f2fs_summary` 然后根据它所记录的 `f2fs_summary->nid` 以及 `f2fs_summary->ofs_in_node` 找到对应的具体的block的数据，然后将这些数据设置为dirty，然后等待vfs的writeback机制完成页迁移。