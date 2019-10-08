# Segment Infomation Tabel-SIT结构
Segment Infomation Tabel，简称SIT，是F2FS用于集中管理segment状态的结构。它的主要作用是维护的segment的分配信息，它的作用使用两个常见例子进行阐述:

- 用户进行写操作，那么segment会根据用户写入的数据量分配特定数目的block给用户进行数据写入，SIT会将这些已经被分配的block标记为"已经使用"，那么之后的写操作就不会再使用这些block。
- 用户进行了**覆盖写**操作以后，由于F2FS异地更新的特性，F2FS会分配新block给用户写入，同时会将旧block置为无效状态，这样gc的时候可以根据segment无效的block的数目，采取某种策略进行回收。

综上所述，SIT的作用是维护每一个segment的block的使用状态以及有效无效状态。

## SIT在元数据区域的物理结构
![cp_layout](../img/F2FS-Layout/sit_layout.png)

从结构体可以知道，SIT区域默认情况下包含了2个segment(1024个block)数目的`struct f2fs_sit_block`组成，每一个`struct f2fs_sit_block`包含了55个`struct f2fs_sit_entry`，每一个entry对应了一个segment的管理状态。每一个entry包含了三个变量: vblocks(记录这个segment有多少个block已经被使用了)，valid_map(记录这个segment里面的哪一些block是无效的)，mtime(表示修改时间)。

### SIT物理存放区域结构

从上图所示，SIT的基本存放单元是`struct f2fs_sit_block`，它结构如下:

```c
struct f2fs_sit_block {
	struct f2fs_sit_entry entries[SIT_ENTRY_PER_BLOCK];
} __packed;
```

由于一个block的尺寸是4KB，因此跟根据`sizeof(struct f2fs_sit_entry entries)`的值，得到`SIT_ENTRY_PER_BLOCK`的值为55。`struct f2fs_sit_entry entries`用来表示每一个segment的状态信息，它的结构如下:

```c
struct f2fs_sit_entry {
	__le16 vblocks;				/* reference above */
	__u8 valid_map[SIT_VBLOCK_MAP_SIZE];	/* bitmap for valid blocks */
	__le64 mtime;				/* segment age for cleaning */
} __packed;
```

第一个参数`vblocks`表示当前segment有多少个block已经被使用，第二个参数`valid_map`表示segment内的每一个block的有效无效信息; 由于一个segment包含了512个block，因此需要用512个bit去表示每一个block的有效无效状态，因此`SIT_VBLOCK_MAP_SIZE`的值是64(8*64=512)。最后一个参数`mtime`表示这个entry被修改的时间。



## SIT内存管理结构

SIT在内存中对应的管理结构是`struct sm_info`，它在` build_segment_manager`函数进行初始化:

```c
int build_segment_manager(struct f2fs_sb_info *sbi)
{
	struct f2fs_super_block *raw_super = F2FS_RAW_SUPER(sbi);
	struct f2fs_checkpoint *ckpt = F2FS_CKPT(sbi);
	struct f2fs_sm_info *sm_info;
	int err;

    /* 分配空间 */
	sm_info = kzalloc(sizeof(struct f2fs_sm_info), GFP_KERNEL);

	/* 初始化一些地址信息，基础信息 */
	sbi->sm_info = sm_info;
	INIT_LIST_HEAD(&sm_info->wblist_head);
	spin_lock_init(&sm_info->wblist_lock);
	sm_info->seg0_blkaddr = le32_to_cpu(raw_super->segment0_blkaddr);
	sm_info->main_blkaddr = le32_to_cpu(raw_super->main_blkaddr);
	sm_info->segment_count = le32_to_cpu(raw_super->segment_count);
	sm_info->reserved_segments = le32_to_cpu(ckpt->rsvd_segment_count);
	sm_info->ovp_segments = le32_to_cpu(ckpt->overprov_segment_count);
	sm_info->main_segments = le32_to_cpu(raw_super->segment_count_main);
	sm_info->ssa_blkaddr = le32_to_cpu(raw_super->ssa_blkaddr);

    /* 初始化内存中的entry数据结构 */
	err = build_sit_info(sbi);
    
    /* 初始化可用segment的数据结构 */
	err = build_free_segmap(sbi);

    /* 恢复checkpoint active segment区域的信息，参考checkpoint结构那一节 */
	err = build_curseg(sbi);

	/* 从磁盘中将SIT物理区域记录的 物理区域sit_entry与只存在于内存的sit_entry建立联系 */
	build_sit_entries(sbi);

    /* 根据checkpoint记录的恢复信息，恢复可用segment的映射关系 */
	init_free_segmap(sbi);
    
    /* 恢复脏segment的映射关系 */
	err = build_dirty_segmap(sbi);

    /* 初始化最大最小的修改时间 */
	init_min_max_mtime(sbi);
	return 0;
}
```

`build_sit_info`用于初始化内存区域的entry，这里需要注意的是注意区分内存entry以及物理区域的entry:

```c
static int build_sit_info(struct f2fs_sb_info *sbi)
{
	struct f2fs_super_block *raw_super = F2FS_RAW_SUPER(sbi);
	struct f2fs_checkpoint *ckpt = F2FS_CKPT(sbi);
	struct sit_info *sit_i;
	unsigned int sit_segs, start;
	char *src_bitmap, *dst_bitmap;
	unsigned int bitmap_size;

	/* 分配空间给sit_info */
	sit_i = kzalloc(sizeof(struct sit_info), GFP_KERNEL);

    /* 将sit_info归于sbi->sm_info进行管理 */
	SM_I(sbi)->sit_info = sit_i;

    /* 根据main area的segment的数目,给每一个segment在内存中分配一个entry结构 */
	sit_i->sentries = vzalloc(TOTAL_SEGS(sbi) * sizeof(struct seg_entry));

    /* 这个bitmap是segment的bitmap,作用是当segment全部block都没有使用过,
     * 这个segment就需要标记free
     */
	bitmap_size = f2fs_bitmap_size(TOTAL_SEGS(sbi));
    
    /* 这个bitmap是记录segment是否为脏的bitmap,作用是当segment分配了一个block之后,
     * 这个segment对应的entry信息就会改变,因此将这个segment标记为脏,之后需要通过某种策略
     * 将数据写回到SIT区域
     */
	sit_i->dirty_sentries_bitmap = kzalloc(bitmap_size, GFP_KERNEL);

    /* 这里给每一个内存entry的记录block状态的bitmap分配空间，SIT_VBLOCK_MAP_SIZE=64 */
	for (start = 0; start < TOTAL_SEGS(sbi); start++) {
		sit_i->sentries[start].cur_valid_map
			= kzalloc(SIT_VBLOCK_MAP_SIZE, GFP_KERNEL);
		sit_i->sentries[start].ckpt_valid_map
			= kzalloc(SIT_VBLOCK_MAP_SIZE, GFP_KERNEL);
	}

	/* 获取SIT区域包含了多少个segment去存放f2fs_sit_block */
	sit_segs = le32_to_cpu(raw_super->segment_count_sit) >> 1;

	/* 从checkpoint中恢复bitmap的状态 */
	bitmap_size = __bitmap_size(sbi, SIT_BITMAP);
	src_bitmap = __bitmap_ptr(sbi, SIT_BITMAP);

	dst_bitmap = kmemdup(src_bitmap, bitmap_size, GFP_KERNEL);

	/* 初始化其他信息 */
	sit_i->s_ops = &default_salloc_ops;

	sit_i->sit_base_addr = le32_to_cpu(raw_super->sit_blkaddr);
	sit_i->sit_blocks = sit_segs << sbi->log_blocks_per_seg;
	sit_i->written_valid_blocks = le64_to_cpu(ckpt->valid_block_count);
	sit_i->sit_bitmap = dst_bitmap;
	sit_i->bitmap_size = bitmap_size;
	sit_i->dirty_sentries = 0;
	sit_i->sents_per_block = SIT_ENTRY_PER_BLOCK;
	sit_i->elapsed_time = le64_to_cpu(sbi->ckpt->elapsed_time);
	sit_i->mounted_time = CURRENT_TIME_SEC.tv_sec;
	mutex_init(&sit_i->sentry_lock);
	return 0;
}
```

`build_free_segmap`用于初始化segment的分配状态: 

```c
static int build_free_segmap(struct f2fs_sb_info *sbi)
{
	struct f2fs_sm_info *sm_info = SM_I(sbi);
	struct free_segmap_info *free_i;
	unsigned int bitmap_size, sec_bitmap_size;

	/* 给管理segment分配状态的free_segmap_info分配内存空间 */
	free_i = kzalloc(sizeof(struct free_segmap_info), GFP_KERNEL);

    /* 将sit_info归于sbi->sm_info进行管理 */
	SM_I(sbi)->free_info = free_i;

    /* 根据segment的数目初始化free map的大小 */
	bitmap_size = f2fs_bitmap_size(TOTAL_SEGS(sbi));
	free_i->free_segmap = kmalloc(bitmap_size, GFP_KERNEL);

    /* 由于1 section = 1 segment，将sec map看作为根据segment map同等作用就好 */
	sec_bitmap_size = f2fs_bitmap_size(TOTAL_SECS(sbi));
	free_i->free_secmap = kmalloc(sec_bitmap_size, GFP_KERNEL);

	/* 在从checkpoint恢复数据之前，将所有的segment设置为dirty */
	memset(free_i->free_segmap, 0xff, bitmap_size);
	memset(free_i->free_secmap, 0xff, sec_bitmap_size);

	/* 初始化其他信息 */
	free_i->start_segno =
		(unsigned int) GET_SEGNO_FROM_SEG0(sbi, sm_info->main_blkaddr);
	free_i->free_segments = 0;
	free_i->free_sections = 0;
	rwlock_init(&free_i->segmap_lock);
	return 0;
}
```

`build_sit_entries`的作用是从SIT的物理区域存放的物理entry与内存的entry建立联系，首先看看物理entry和内存entry的差异在哪里。

```c
// 物理entry
struct f2fs_sit_entry {
	__le16 vblocks;				/* reference above */
	__u8 valid_map[SIT_VBLOCK_MAP_SIZE];	/* bitmap for valid blocks */
	__le64 mtime;				/* segment age for cleaning */
} __packed;

// 内存entry
struct seg_entry {
	unsigned short valid_blocks;	/* # of valid blocks */
	unsigned char *cur_valid_map;	/* validity bitmap of blocks */
	unsigned short ckpt_valid_blocks;
	unsigned char *ckpt_valid_map;
	unsigned char type;		/* segment type like CURSEG_XXX_TYPE */
	unsigned long long mtime;	/* modification time of the segment */
};
```

两者之间的差异主要是多了表示segment类型的type变量，以及多了两个与checkpoint相关的内容。



其实物理entry也包含了type的信息，但是为了节省空间，将type于vblocks存放在了一起，及vblocks的前10位表示数目，后6位表示type，他们的关系可以用`f2fs_fs.h`找到:

```c
#define SIT_VBLOCKS_SHIFT	10
#define SIT_VBLOCKS_MASK	((1 << SIT_VBLOCKS_SHIFT) - 1)
#define GET_SIT_VBLOCKS(raw_sit)				\
	(le16_to_cpu((raw_sit)->vblocks) & SIT_VBLOCKS_MASK)
#define GET_SIT_TYPE(raw_sit)					\
	((le16_to_cpu((raw_sit)->vblocks) & ~SIT_VBLOCKS_MASK)	\
	 >> SIT_VBLOCKS_SHIFT)
```



因此，内存entry实际上仅仅多了2个与checkpoint相关的信息，即`ckpt_valid_blocks`与`ckpt_valid_map`。在系统执行checkpoint的时候，会将`valid_blocks`以及`cur_valid_map`的值分别写入`ckpt_valid_blocks`与`ckpt_valid_map`，当系统出现宕机的时候根据这个值恢复映射信息。



继续分析`build_sit_entries`的代码，

```c
static void build_sit_entries(struct f2fs_sb_info *sbi)
{
	struct sit_info *sit_i = SIT_I(sbi);
	struct curseg_info *curseg = CURSEG_I(sbi, CURSEG_COLD_DATA);
	struct f2fs_summary_block *sum = curseg->sum_blk;
	unsigned int start;

    /* 建立物理entry以及内存entry的关系 */
	for (start = 0; start < TOTAL_SEGS(sbi); start++) {
		struct seg_entry *se = &sit_i->sentries[start]; // 内存entry
		struct f2fs_sit_block *sit_blk;
		struct f2fs_sit_entry sit;
		struct page *page;
		int i;

        // 先尝试在journal恢复
		mutex_lock(&curseg->curseg_mutex);
		for (i = 0; i < sits_in_cursum(sum); i++) {
			if (le32_to_cpu(segno_in_journal(sum, i)) == start) {
				sit = sit_in_journal(sum, i);
				mutex_unlock(&curseg->curseg_mutex);
				goto got_it;
			}
		}
		mutex_unlock(&curseg->curseg_mutex);
        
        // 如果恢复不了就从SIT恢复
		page = get_current_sit_page(sbi, start); // 读取 f2fs_sit_block
		sit_blk = (struct f2fs_sit_block *)page_address(page); // 转换为block
		sit = sit_blk->entries[SIT_ENTRY_OFFSET(sit_i, start)]; // 物理entry
		f2fs_put_page(page, 1);
got_it:
		check_block_count(sbi, start, &sit);
		seg_info_from_raw_sit(se, &sit); // 将物理entry的数据赋予到内存entry
	}
}

```

`init_free_segmap` 从内存entry以及checkpoint中恢复free segment的信息:

```c
static void init_free_segmap(struct f2fs_sb_info *sbi)
{
	unsigned int start;
	int type;

	for (start = 0; start < TOTAL_SEGS(sbi); start++) { // 根据segment编号遍历每一个内存entry
		struct seg_entry *sentry = get_seg_entry(sbi, start);
		if (!sentry->valid_blocks) // 如果这个segment一个block都没有用过，则设置为free
			__set_free(sbi, start);
	}

	/* 从checkpoint的curseg中恢复可用信息 */
	for (type = CURSEG_HOT_DATA; type <= CURSEG_COLD_NODE; type++) {
		struct curseg_info *curseg_t = CURSEG_I(sbi, type);
		__set_test_and_inuse(sbi, curseg_t->segno); // 设置为正在使用的状态
	}
}
```

`init_dirty_segmap`恢复脏segment的信息

```c
static void init_dirty_segmap(struct f2fs_sb_info *sbi)
{
	struct dirty_seglist_info *dirty_i = DIRTY_I(sbi);
	struct free_segmap_info *free_i = FREE_I(sbi);
	unsigned int segno = 0, offset = 0;
	unsigned short valid_blocks;

	while (segno < TOTAL_SEGS(sbi)) {
		/* find dirty segment based on free segmap */
		segno = find_next_inuse(free_i, TOTAL_SEGS(sbi), offset); // 找出所有已经使用过的seg
		if (segno >= TOTAL_SEGS(sbi))
			break;
		offset = segno + 1;
		valid_blocks = get_valid_blocks(sbi, segno, 0); // 得到了使用了多少个block
		if (valid_blocks >= sbi->blocks_per_seg || !valid_blocks)
			continue;
		mutex_lock(&dirty_i->seglist_lock);
		__locate_dirty_segment(sbi, segno, DIRTY); // 将其设置为dirty
		mutex_unlock(&dirty_i->seglist_lock);
	}
}
```







