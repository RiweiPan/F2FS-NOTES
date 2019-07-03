# Checkpoint结构
Checkpoint是维护F2FS的数据一致性的结构，它维护了系统当前的状态，例如segment的分配情况，node的分配情况，以及当前的active segment的状态等。F2FS在满足一定的条件的情况下，将当前系统的状态写入Checkpoint中，万一系统出现突然宕机，这个是F2FS可以从Checkpoint中恢复到上次回写时的状态，以保证数据的可恢复性。F2FS维护了两个Checkpoint结构，互为备份，其中一个是当前正在使用的Checkpoint，另外一个上次回写的稳定的Chcekpoint。如果系统出现了宕机，那么当前的Checkpoint就会变得不可信任，进而使用备份Checkpoint进行恢复。


## Checkpoint物理存放区域结构
Checkpoint区域由几个部分构成，分别是checkpoint元数据区域、orphan node区域、active segments区域：

```
+-------------------+-------------+----------------+-------------------+
| f2fs_checkpoint 1 | orphan node | active segments | f2fs_checkpoint 2 |
+-------------------+-------------+----------------+-------------------+
```

### Checkpoint元数据区域
F2FS使用数据结构`f2fs_checkpoint`表示Checkpoint结构，它保存在磁盘中`f2fs_super_block`之后区域中，数据结构如下：
```c
struct f2fs_checkpoint {
	__le64 checkpoint_ver;		/* CP版本，用于比较新旧版本进行恢复 */
	__le64 user_block_count;	/* # of user blocks */
	__le64 valid_block_count;	/* # of valid blocks in main area */
	__le32 rsvd_segment_count;	/* # of reserved segments for gc */
	__le32 overprov_segment_count;	/* # of overprovision segments */
	__le32 free_segment_count;	/* # of free segments in main area */

	/* information of current node segments */
	__le32 cur_node_segno[MAX_ACTIVE_NODE_LOGS];
	__le16 cur_node_blkoff[MAX_ACTIVE_NODE_LOGS];
	/* information of current data segments */
	__le32 cur_data_segno[MAX_ACTIVE_DATA_LOGS];
	__le16 cur_data_blkoff[MAX_ACTIVE_DATA_LOGS];
	__le32 ckpt_flags;		/* Flags : umount and journal_present */
	__le32 cp_pack_total_block_count;	/* total # of one cp pack */
	__le32 cp_pack_start_sum;	/* start block number of data summary */
	__le32 valid_node_count;	/* Total number of valid nodes */
	__le32 valid_inode_count;	/* Total number of valid inodes */
	__le32 next_free_nid;		/* Next free node number */
	__le32 sit_ver_bitmap_bytesize;	/* Default value 64 */
	__le32 nat_ver_bitmap_bytesize; /* Default value 256 */
	__le32 checksum_offset;		/* checksum offset inside cp block */
	__le64 elapsed_time;		/* mounted time */
	/* allocation type of current segment */
	unsigned char alloc_type[MAX_ACTIVE_LOGS];

	/* SIT and NAT version bitmap */
	unsigned char sit_nat_version_bitmap[1];
} __packed;
```

### Orphan node区域
这是一个动态的区域，如果没有orphan node list则不会占用空间


### Active Segments区域
#### Active Segments的定义
Active Segments，又称current segment(CURSEG)，即当前正在用于分配的segment，如用户需要写入8KB数据，那么就会从active segments分配两个block提供给用户写入到磁盘中。F2FS为了提高数据分配的效率，根据数据的特性，一共定义了6个active segment。如[总体结构](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS-Layout/%E6%80%BB%E4%BD%93%E7%BB%93%E6%9E%84.md)这一节的multi-head logging特性所描述，这6个active segments对应了(HOT, WARM, COLD) X (NODE, DATA)的数据。

#### Active Segment与恢复相关的数据结构
CP的主要任务是维护数据一致性，因此CP的Active Segment区域的主要任务是维护Active Segment的分配状态，使系统宕机时候可以恢复正常。维护Active Segment需要维护三种信息，分别是`f2fs_checkpoint`的信息，以及该segment对应的journal和summary的信息。

- **f2fs_checkpoint中Active Segment信息**：从上面给出的`f2fs_checkpoint`定义，`cur_node_segno[MAX_ACTIVE_NODE_LOGS]`和`cur_data_segno[MAX_ACTIVE_DATA_LOGS]`表示node和data当前的Active Segment的编号，系统可以通过这个编号找到对应的segment。`MAX_ACTIVE_NODE_LOGS`以及`MAX_ACTIVE_NODE_LOGS`分别表示data和node有多少种类型，F2FS默认情况下都等于3，表示HOT、WARM、COLD类型数据。`cur_node_blkoff[MAX_ACTIVE_NODE_LOGS]`以及`cur_data_blkoff[MAX_ACTIVE_DATA_LOGS]`则分别表示当前Active Segment分配到哪一个block(一个segment包含了512个block)。

- **Segment对应的Journal信息**：Journal在两处地方都有出现，分别是CP区域以及SSA区域。F2FS定义的journal结构如下，它主要保存了NODE以及SEGMENT的修改信息。如系统分配出一个block给用户，那么就要将这个block在bitmap中标记为已分配，防止其他请求使用。分两个区域存放journal使为了减轻频繁更新导致的系统性能下降。例如，当系统写压力很大的时候，bitmap就会频繁被更新，如果这个时候频繁将bitmap写入SSA，就会加重写压力。因此CP区域的Journal的作用就是维护这些经常修改的数据，等待CP被触发的时候才吸入磁盘，从而减少写压力。
```c
struct f2fs_journal {
	union {
		__le16 n_nats;
		__le16 n_sits;
	};
	/* spare area is used by NAT or SIT journals or extra info */
	union {
		struct nat_journal nat_j;
		struct sit_journal sit_j;
		struct f2fs_extra_info info;
	};
} __packed;
```
- **Segment对应的Summary信息**：Summary同样在CP区域和SSA区域有出现，它表示的是逻辑地址和物理地址的映射关系，这个映射关系会使用到GC流程中。Summary与segment是一对一的关系，一个summary保存了一个segment所有的block的物理地址和逻辑地址的映射关系。Summary保存在CP区域中同样是出于减少IO的写入。


## Checkpoint内存存管理结构
Checkpoint的内存管理结构是`struct f2fs_checkpoint`本身，因为Checkpoint一般只在F2FS启动的时候被读取数据，用于数据恢复，而在运行过程中大部分情况都是被写，用于记录恢复信息。因此，Checkpoint不需要过于复杂的内存管理结构，因此使用`struct f2fs_checkpoint`本身即可以满足需求。

另一方面，Active Segments区域的信息涉及到系统block地址的分配，因此需要特定的管理结构`struct curseg_info`进行管理，它的定义如下:
```c
struct curseg_info {
	struct mutex curseg_mutex;
	struct f2fs_summary_block *sum_blk;	/* 每一个segment对应一个summary block */
	struct rw_semaphore journal_rwsem;
	struct f2fs_journal *journal;		/*每一个segment对应一个 info */
	unsigned char alloc_type;
	unsigned int segno;			/* 当前segno */
	unsigned short next_blkoff;		/* 记录当前segment用于分配的下一个给block号 */
	unsigned int zone;			/* current zone number */
	unsigned int next_segno;		/* 当前segno用完以后，下个即将用来分配的segno */
};
```
从结构分析可以直到，`curseg_info`记录当前的segment的分配信息，当系统出现宕机的时候，可以从CP记录的`curseg_info`恢复当上一次CP点的状态。

每一种类型的active segment就对应一个`struct curseg_info`结构。在F2FS中，使用一个数组来表示:
```c
struct f2fs_sm_info {
	...
	struct curseg_info *curseg_array; // 默认是分配6个curseg_info，分别对应不同类型
	...
}
```
`struct f2fs_sm_info`是SIT的管理结构，它也管理了CP最终的active segment的信息，是一个跨区域的管理结构。


`struct f2fs_checkpoint`通过`get_checkpoint_version`函数从磁盘读取出来:
```c
static int get_checkpoint_version(struct f2fs_sb_info *sbi, block_t cp_addr,
		struct f2fs_checkpoint **cp_block, struct page **cp_page,
		unsigned long long *version)
{
	unsigned long blk_size = sbi->blocksize;
	size_t crc_offset = 0;
	__u32 crc = 0;

	*cp_page = f2fs_get_meta_page(sbi, cp_addr); // 根据CP所在的地址cp_addr从磁盘读取一个block
	*cp_block = (struct f2fs_checkpoint *)page_address(*cp_page); // 直接转换为数据结构

	crc_offset = le32_to_cpu((*cp_block)->checksum_offset);
	if (crc_offset > (blk_size - sizeof(__le32))) {
		f2fs_msg(sbi->sb, KERN_WARNING,
			"invalid crc_offset: %zu", crc_offset);
		return -EINVAL;
	}

	crc = cur_cp_crc(*cp_block);
	if (!f2fs_crc_valid(sbi, crc, *cp_block, crc_offset)) { // 比较CRC的值，进而知道是否成功读取出来
		f2fs_msg(sbi->sb, KERN_WARNING, "invalid crc value");
		return -EINVAL;
	}

	*version = cur_cp_version(*cp_block);
	return 0;
}
```

`struct curseg_info`则是通过`build_curseg`函数进行初始化:
```c
static int build_curseg(struct f2fs_sb_info *sbi)
{
	struct curseg_info *array;
	int i;

	array = f2fs_kzalloc(sbi, array_size(NR_CURSEG_TYPE, sizeof(*array)),
			     GFP_KERNEL); // 根据active segment类型的数目分配空间
	if (!array)
		return -ENOMEM;

	SM_I(sbi)->curseg_array = array; // 赋值到f2fs_sm_info->curseg_array

	for (i = 0; i < NR_CURSEG_TYPE; i++) { // 为curseg的其他信息分配空间
		mutex_init(&array[i].curseg_mutex);
		array[i].sum_blk = f2fs_kzalloc(sbi, PAGE_SIZE, GFP_KERNEL);
		if (!array[i].sum_blk)
			return -ENOMEM;
		init_rwsem(&array[i].journal_rwsem);
		array[i].journal = f2fs_kzalloc(sbi,
				sizeof(struct f2fs_journal), GFP_KERNEL);
		if (!array[i].journal)
			return -ENOMEM;
		array[i].segno = NULL_SEGNO;
		array[i].next_blkoff = 0;
	}
	return restore_curseg_summaries(sbi); // 从f2fs_checkpoint恢复上一个CP点CURSEG的状态
}

static int restore_curseg_summaries(struct f2fs_sb_info *sbi)
{
	struct f2fs_journal *sit_j = CURSEG_I(sbi, CURSEG_COLD_DATA)->journal;
	struct f2fs_journal *nat_j = CURSEG_I(sbi, CURSEG_HOT_DATA)->journal;
	int type = CURSEG_HOT_DATA;
	int err;

	...
	for (; type <= CURSEG_COLD_NODE; type++) { // 按类型逐个恢复active segment的信息
		err = read_normal_summaries(sbi, type);
		if (err)
			return err;
	}
	...

	return 0;
}

static int read_normal_summaries(struct f2fs_sb_info *sbi, int type)
{
	struct f2fs_checkpoint *ckpt = F2FS_CKPT(sbi);
	struct f2fs_summary_block *sum;
	struct curseg_info *curseg;
	struct page *new;
	unsigned short blk_off;
	unsigned int segno = 0;
	block_t blk_addr = 0;

	...
	segno = le32_to_cpu(ckpt->cur_data_segno[type]); // 从CP读取segno
	blk_off = le16_to_cpu(ckpt->cur_data_blkoff[type - CURSEG_HOT_DATA]); // 从CP读取blk_off
	blk_addr = sum_blk_addr(sbi, NR_CURSEG_DATA_TYPE, type); // 获取summary block地址	

	// 读取&转换结构
	new = f2fs_get_meta_page(sbi, blk_addr);
	sum = (struct f2fs_summary_block *)page_address(new);

	curseg = CURSEG_I(sbi, type); // 根据type找到对应的curseg
	mutex_lock(&curseg->curseg_mutex);

	/* 复制&恢复数据 */
	down_write(&curseg->journal_rwsem);
	memcpy(curseg->journal, &sum->journal, SUM_JOURNAL_SIZE);
	up_write(&curseg->journal_rwsem);

	memcpy(curseg->sum_blk->entries, sum->entries, SUM_ENTRY_SIZE);
	memcpy(&curseg->sum_blk->footer, &sum->footer, SUM_FOOTER_SIZE);
	curseg->next_segno = segno;
	reset_curseg(sbi, type, 0);
	curseg->alloc_type = ckpt->alloc_type[type];
	curseg->next_blkoff = blk_off; // 恢复上次的分配状态
	mutex_unlock(&curseg->curseg_mutex);
	f2fs_put_page(new, 1);
	return 0;
}
```


