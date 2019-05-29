## Checkpoint的作用与实现
后滚恢复即恢复到上一个Checkpoint点的元数据状态，因此F2FS需要在特定的时刻将Checkpoint的数据写入到磁盘中。

### Checkpoint的时机
CP是一个开销很大的操作，因此合理选取CP时机，能够很好地提高性能。CP的触发时机有:
>前台GC(FG_GC)
>FASTBOOT
>UMOUNT
>DISCARD
>RECOVERY
>TRIM
>周期进行

因此F2FS有几个宏表示CP的触发原因:

```c
#define	CP_UMOUNT	0x00000001
#define	CP_FASTBOOT	0x00000002
#define	CP_SYNC		0x00000004
#define	CP_RECOVERY	0x00000008
#define	CP_DISCARD	0x00000010
#define CP_TRIMMED	0x00000020
```

大部分情况下，都是触发 `CP_SYNC` 这个宏的CP。

### Checkpoint的核心流程
Checkpoint的入口函数以及核心数据结构
```c
struct cp_control {
	int reason;         /* Checkpoint的原因，大部分情况下是CP_SYNC */
	__u64 trim_start;   /* CP处理后的数据的block起始地址 */
	__u64 trim_end;     /* CP处理后的数据的block结束地址 */
	__u64 trim_minlen;  /* CP处理后的最小长度 */
};

int f2fs_write_checkpoint(struct f2fs_sb_info *sbi, struct cp_control *cpc)
```
下面分段分析 `CP_SYNC` 原因的 `f2fs_write_checkpoint` 函数的核心流程。

```c
f2fs_write_checkpoint(struct f2fs_sb_info *sbi, struct cp_control *cpc)
{
	struct f2fs_checkpoint *ckpt = F2FS_CKPT(sbi); //从sbi读取当前CP的数据结构
	...
	err = block_operations(sbi); //将文件系统的所有操作都停止
	...
	f2fs_flush_merged_writes(sbi); // 将暂存的所有BIO刷写到磁盘
	...
	ckpt_ver = cur_cp_version(ckpt); // 获取当前CP的version
	ckpt->checkpoint_ver = cpu_to_le64(++ckpt_ver); // 给当前CP version加1

	// 更新元数据的NAT区域
	f2fs_flush_nat_entries(sbi, cpc); // 刷写所有nat entries到磁盘

	// 更新元数据的SIT区域
	f2fs_flush_sit_entries(sbi, cpc); // 刷写所有sit entries到磁盘，处理dirty prefree segments

	// 更新元数据的Checkpoint区域以及Summary区域
	err = do_checkpoint(sbi, cpc); // checkpoint核心流程

	
	f2fs_clear_prefree_segments(sbi, cpc); // 清除dirty prefree segments的dirty标记
	unblock_operations(sbi); //恢复文件系统的操作
	...
	f2fs_update_time(sbi, CP_TIME); // 更新CP的时间
	...
}
```
### Checkpoint涉及的子函数的分析

#### 暂存BIO的回写
一般情况下，文件系统与设备的交互的开销是比较大的，因此一些文件系统为了减少交互的开销，都会尽可能将更多的page合并在一个bio中，再提交到设备，进而减少交互的次数。F2FS中，在sbi中使用了`struct f2fs_bio_info`结构用于减少交互次数，它的核心是缓存一个bio，将即将回写的page都保存到这个bio中，等到bio尽可能满再回写进入磁盘。它在sbi的声明如下:
```c
struct f2fs_sb_info {
	...
	struct f2fs_bio_info *write_io[NR_PAGE_TYPE]; // NR_PAGE_TYPE表示HOW/WARM/COLD不同类型的数据
	...
}
```
在Checkpoint流程中，必须要回写暂存的page，以获得系统最新的稳定状态信息，它调用了函数是 `f2fs_flush_merged_writes`。`f2fs_flush_merged_writes` 函数调用了`f2fs_submit_merged_write`分别回写了DATA、NODE、META的信息。然后会调用`__submit_merged_write_cond`函数，这个函数会遍历HOW/WARM/COLD对应的`sbi->write_io`进行回写，最后调用`__submit_merged_bio`函数，从`sbi->write_io`得到bio，submit到设备中。
```c
void f2fs_flush_merged_writes(struct f2fs_sb_info *sbi)
{
	f2fs_submit_merged_write(sbi, DATA);
	f2fs_submit_merged_write(sbi, NODE);
	f2fs_submit_merged_write(sbi, META);
}

void f2fs_submit_merged_write(struct f2fs_sb_info *sbi, enum page_type type)
{
	__submit_merged_write_cond(sbi, NULL, 0, 0, type, true);
}

static void __submit_merged_write_cond(struct f2fs_sb_info *sbi,
				struct inode *inode, nid_t ino, pgoff_t idx,
				enum page_type type, bool force)
{
	enum temp_type temp;

	if (!force && !has_merged_page(sbi, inode, ino, idx, type))
		return;

	for (temp = HOT; temp < NR_TEMP_TYPE; temp++) { // 遍历不同的HOT/WARM/COLD类型就行回写

		__f2fs_submit_merged_write(sbi, type, temp);

		/* TODO: use HOT temp only for meta pages now. */
		if (type >= META)
			break;
	}
}

static void __f2fs_submit_merged_write(struct f2fs_sb_info *sbi,
				enum page_type type, enum temp_type temp)
{
	enum page_type btype = PAGE_TYPE_OF_BIO(type);
	struct f2fs_bio_info *io = sbi->write_io[btype] + temp; // temp可以计算属于HOT/WARM/COLD对应的sbi->write_io

	down_write(&io->io_rwsem);

	/* change META to META_FLUSH in the checkpoint procedure */
	if (type >= META_FLUSH) {
		io->fio.type = META_FLUSH;
		io->fio.op = REQ_OP_WRITE;
		io->fio.op_flags = REQ_META | REQ_PRIO | REQ_SYNC;
		if (!test_opt(sbi, NOBARRIER))
			io->fio.op_flags |= REQ_PREFLUSH | REQ_FUA;
	}
	__submit_merged_bio(io);
	up_write(&io->io_rwsem);
}

static void __submit_merged_bio(struct f2fs_bio_info *io)
{
	struct f2fs_io_info *fio = &io->fio;

	if (!io->bio)
		return;

	bio_set_op_attrs(io->bio, fio->op, fio->op_flags);

	if (is_read_io(fio->op))
		trace_f2fs_prepare_read_bio(io->sbi->sb, fio->type, io->bio);
	else
		trace_f2fs_prepare_write_bio(io->sbi->sb, fio->type, io->bio);

	__submit_bio(io->sbi, io->bio, fio->type); // 从f2fs_io_info得到bio，提交到设备
	io->bio = NULL;
}
```


#### NAT区域的脏数据回写
`f2fs_flush_nat_entries` 和 `f2fs_flush_sit_entries` 的作用是将暂存在ram的nat entry合sit entry都回写到Journal或磁盘当中:
##### f2fs_flush_nat_entries函数
修改node的信息会对对应的`nat_entry`进行修改，同时`nat_entry`会被设置为脏，加入到`nm_i->nat_set_root`的radix tree中。Checkpoint会对脏的`nat_entry`进行回写，完成元数据的更新。

首先声明了一个list变量`LIST_HEAD(sets)`，然后通过一个while循环，将`nat_entry_set`按一个set为单位，对脏的`nat_entry`进行提取，每次提取SETVEC_SIZE个，然后保存到`setvec[SETVEC_SIZE]`中，然后对`setvec`中的每一个`nat_entry_set`，按照一定条件加入到`LIST_HEAD(sets)`的链表中。最后针对`LIST_HEAD(sets)`的`nat_entry_set`，执行`__flush_nat_entry_set`函数，对脏数据进行回写。`__flush_nat_entry_set`有两种回写方法，第一种是写入到curseg的journal中，第二种是直接找到对应的nat block，回写到磁盘中。
```c
void f2fs_flush_nat_entries(struct f2fs_sb_info *sbi, struct cp_control *cpc)
{
	struct f2fs_nm_info *nm_i = NM_I(sbi);
	struct curseg_info *curseg = CURSEG_I(sbi, CURSEG_HOT_DATA);
	struct f2fs_journal *journal = curseg->journal;
	struct nat_entry_set *setvec[SETVEC_SIZE];
	struct nat_entry_set *set, *tmp;
	unsigned int found;
	nid_t set_idx = 0;
	LIST_HEAD(sets);

	if (!nm_i->dirty_nat_cnt)
		return;

	down_write(&nm_i->nat_tree_lock);

	/*
	 * __gang_lookup_nat_set 这个函数就是从radix tree读取set_idx开始，
	 * 连续读取SETVEC_SIZE这么多个nat_entry_set，保存在setvec中
	 * 然后按照一定条件，通过__adjust_nat_entry_set函数加入到LIST_HEAD(sets)链表中
	 * */
	while ((found = __gang_lookup_nat_set(nm_i,
					set_idx, SETVEC_SIZE, setvec))) {
		unsigned idx;
		set_idx = setvec[found - 1]->set + 1;
		for (idx = 0; idx < found; idx++)
			__adjust_nat_entry_set(setvec[idx], &sets,
						MAX_NAT_JENTRIES(journal));
	}

	/*
	 * flush dirty nats in nat entry set
	 * 遍历这个list所有的nat_entry_set，然后写入到curseg->journal中
	 * */
	list_for_each_entry_safe(set, tmp, &sets, set_list)
		__flush_nat_entry_set(sbi, set, cpc);

	up_write(&nm_i->nat_tree_lock);
	/* Allow dirty nats by node block allocation in write_begin */
}
```

`__flush_nat_entry_set`有两种回写的方式，第一种是写入到curseg的journal中，第二种是回写到nat block中。

第一种写入方式通常是由于curseg有足够的journal的情况下的写入，首先遍历`nat_entry_set`中的所有`nat_entry`，然后根据nid找到curseg->journal中对应的nat_entry的位置，跟着将被遍历的`nat_entry`的值赋予给curseg->journal的`nat_entry`，通过`raw_nat_from_node_info`完成curseg的nat_entry的更新。

第二种写入方式在curseg没有足够的journal的时候触发，首先根据nid找到NAT区域的对应的`f2fs_nat_block`，然后通过`get_next_nat_page`读取出来，然后通过`raw_nat_from_node_info`进行更新。


```c
static void __flush_nat_entry_set(struct f2fs_sb_info *sbi,
		struct nat_entry_set *set, struct cp_control *cpc)
{
	struct curseg_info *curseg = CURSEG_I(sbi, CURSEG_HOT_DATA);
	struct f2fs_journal *journal = curseg->journal;
	nid_t start_nid = set->set * NAT_ENTRY_PER_BLOCK; // 根据set number找到对应f2fs_nat_block
	bool to_journal = true;
	struct f2fs_nat_block *nat_blk;
	struct nat_entry *ne, *cur;
	struct page *page = NULL;

	/*
	 * there are two steps to flush nat entries:
	 * #1, flush nat entries to journal in current hot data summary block.
	 * #2, flush nat entries to nat page.
	 */
	if (enabled_nat_bits(sbi, cpc) ||
		!__has_cursum_space(journal, set->entry_cnt, NAT_JOURNAL)) //当curseg的journal空间不够了，就刷写到磁盘中
		to_journal = false;

	if (to_journal) {
		down_write(&curseg->journal_rwsem);
	} else {
		page = get_next_nat_page(sbi, start_nid); /* 根据nid找到管理这个nid的f2fs_nat_block */
		nat_blk = page_address(page);
		f2fs_bug_on(sbi, !nat_blk);
	}

	/*
	 * flush dirty nats in nat entry set
	 * 遍历所有的nat_entry
	 *
	 * nat_entry只存在于内存当中，具体在磁盘保存的是f2fs_entry_block
	 * */
	list_for_each_entry_safe(ne, cur, &set->entry_list, list) {
		struct f2fs_nat_entry *raw_ne;
		nid_t nid = nat_get_nid(ne);
		int offset;

		f2fs_bug_on(sbi, nat_get_blkaddr(ne) == NEW_ADDR);

		if (to_journal) {
			// 搜索当前的journal中nid所在的位置
			offset = f2fs_lookup_journal_in_cursum(journal,
							NAT_JOURNAL, nid, 1);
			f2fs_bug_on(sbi, offset < 0);
			raw_ne = &nat_in_journal(journal, offset); // 从journal中取出f2fs_nat_entry的信息
			nid_in_journal(journal, offset) = cpu_to_le32(nid); // 更新journal的nid
		} else {
			raw_ne = &nat_blk->entries[nid - start_nid]; /* 拿到nid对应的nat_entry地址，下面开始填数据 */
		}
		raw_nat_from_node_info(raw_ne, &ne->ni); // 将node info的信息更新到journal中后者磁盘中
		nat_reset_flag(ne); // 清除需要CP的标志
		__clear_nat_cache_dirty(NM_I(sbi), set, ne); // 从dirty list清除处理后的entry
		if (nat_get_blkaddr(ne) == NULL_ADDR) { // 如果对应nid已经是被无效化了，则释放
			add_free_nid(sbi, nid, false, true);
		} else {
			spin_lock(&NM_I(sbi)->nid_list_lock);
			update_free_nid_bitmap(sbi, nid, false, false); // 更新可用的nat的bitmap
			spin_unlock(&NM_I(sbi)->nid_list_lock);
		}
	}

	if (to_journal) {
		up_write(&curseg->journal_rwsem);
	} else {
		__update_nat_bits(sbi, start_nid, page);
		f2fs_put_page(page, 1);
	}

	/* Allow dirty nats by node block allocation in write_begin */
	if (!set->entry_cnt) {
		radix_tree_delete(&NM_I(sbi)->nat_set_root, set->set);
		kmem_cache_free(nat_entry_set_slab, set);
	}
}

```

#### SIT区域的脏数据回写
##### f2fs_flush_sit_entries函数

主要过程跟 `f2fs_flush_nat_entries` 类似，将dirty的seg_entry刷写到journal或sit block中

```c
void f2fs_flush_sit_entries(struct f2fs_sb_info *sbi, struct cp_control *cpc)
{
	struct sit_info *sit_i = SIT_I(sbi);
	unsigned long *bitmap = sit_i->dirty_sentries_bitmap;
	struct curseg_info *curseg = CURSEG_I(sbi, CURSEG_COLD_DATA);
	struct f2fs_journal *journal = curseg->journal;
	struct sit_entry_set *ses, *tmp;
	struct list_head *head = &SM_I(sbi)->sit_entry_set;
	bool to_journal = true;
	struct seg_entry *se;

	down_write(&sit_i->sentry_lock);

	if (!sit_i->dirty_sentries)
		goto out;

	/*
	 * add and account sit entries of dirty bitmap in sit entry
	 * set temporarily
	 *
	 * 遍历所有dirty的segment的segno，
	 * 找到对应的sit_entry_set，然后保存到sbi->sm_info->sit_entry_set
	 */
	add_sits_in_set(sbi);

	/*
	 * if there are no enough space in journal to store dirty sit
	 * entries, remove all entries from journal and add and account
	 * them in sit entry set.
	 */
	if (!__has_cursum_space(journal, sit_i->dirty_sentries, SIT_JOURNAL))
		remove_sits_in_journal(sbi);

	/*
	 * there are two steps to flush sit entries:
	 * #1, flush sit entries to journal in current cold data summary block.
	 * #2, flush sit entries to sit page.
	 * 遍历list中的所有segno对应的sit_entry_set
	 */
	list_for_each_entry_safe(ses, tmp, head, set_list) {
		struct page *page = NULL;
		struct f2fs_sit_block *raw_sit = NULL;
		unsigned int start_segno = ses->start_segno;
		unsigned int end = min(start_segno + SIT_ENTRY_PER_BLOCK,
						(unsigned long)MAIN_SEGS(sbi));
		unsigned int segno = start_segno; /* 找到 */

		if (to_journal &&
			!__has_cursum_space(journal, ses->entry_cnt, SIT_JOURNAL))
			to_journal = false;

		if (to_journal) {
			down_write(&curseg->journal_rwsem);
		} else {
			page = get_next_sit_page(sbi, start_segno); /* 访问磁盘，从磁盘获取到f2fs_sit_block */
			raw_sit = page_address(page); /* 根据segno获得f2fs_sit_block，然后下一步将数据写入这个block当中 */
		}

		/*
		 * flush dirty sit entries in region of current sit set
		 * 遍历segno~end所有dirty的seg_entry
		 * */
		for_each_set_bit_from(segno, bitmap, end) {
			int offset, sit_offset;

			se = get_seg_entry(sbi, segno); /* 根据segno从SIT缓存中获取到seg_entry，这个缓存是F2FS初始化的时候，将全部seg_entry读入创建的 */

			if (to_journal) {
				offset = f2fs_lookup_journal_in_cursum(journal,
							SIT_JOURNAL, segno, 1);
				f2fs_bug_on(sbi, offset < 0);
				segno_in_journal(journal, offset) =
							cpu_to_le32(segno);
				seg_info_to_raw_sit(se,
					&sit_in_journal(journal, offset)); // 更新journal的数据
				check_block_count(sbi, segno,
					&sit_in_journal(journal, offset));
			} else {
				sit_offset = SIT_ENTRY_OFFSET(sit_i, segno);
				seg_info_to_raw_sit(se,
						&raw_sit->entries[sit_offset]); // 更新f2fs_sit_block的数据
				check_block_count(sbi, segno,
						&raw_sit->entries[sit_offset]);
			}

			__clear_bit(segno, bitmap); // 从dirty map中除名
			sit_i->dirty_sentries--;
			ses->entry_cnt--;
		}

		if (to_journal)
			up_write(&curseg->journal_rwsem);
		else
			f2fs_put_page(page, 1);

		f2fs_bug_on(sbi, ses->entry_cnt);
		release_sit_entry_set(ses);
	}

	f2fs_bug_on(sbi, !list_empty(head));
	f2fs_bug_on(sbi, sit_i->dirty_sentries);
out:
	up_write(&sit_i->sentry_lock);

	/*
	 * 通过CP的时机，将暂存在dirty_segmap的dirty的segment信息，更新到free_segmap中
	 * 而且与接下来的do_checkpoint完成的f2fs_clear_prefree_segments有关系，因为 这里处理完了
	 * dirty prefree segments，所以在f2fs_clear_prefree_segments这个函数将它的dirty标记清除
	 * */
	set_prefree_as_free_segments(sbi);
}

static inline struct seg_entry *get_seg_entry(struct f2fs_sb_info *sbi,
						unsigned int segno)
{
	struct sit_info *sit_i = SIT_I(sbi);
	return &sit_i->sentries[segno];
}

static void set_prefree_as_free_segments(struct f2fs_sb_info *sbi)
{
	struct dirty_seglist_info *dirty_i = DIRTY_I(sbi);
	unsigned int segno;

	mutex_lock(&dirty_i->seglist_lock);
	/*
	 * 遍历dirty_seglist_info->dirty_segmap[PRE]，然后执行__set_test_and_free
	 * */
	for_each_set_bit(segno, dirty_i->dirty_segmap[PRE], MAIN_SEGS(sbi))
		__set_test_and_free(sbi, segno); /* 根据segno更新free_segmap的可用信息 */
	mutex_unlock(&dirty_i->seglist_lock);
}

static inline void __set_test_and_free(struct f2fs_sb_info *sbi,
		unsigned int segno)
{
	struct free_segmap_info *free_i = FREE_I(sbi);
	unsigned int secno = GET_SEC_FROM_SEG(sbi, segno);
	unsigned int start_segno = GET_SEG_FROM_SEC(sbi, secno);
	unsigned int next;

	spin_lock(&free_i->segmap_lock);
	/*
	 * free_i->free_segmap用这个bitmap表示这个segment是否是dirty
	 * 如果这个segno对应的segment位置等于0，代表不是dirty，不作处理
	 * 如果这个segno对应的位置等于1，表示这个segment是dirty的，那么在当前的free_segment+1，更新最新的free_segment信息
	 * */
	if (test_and_clear_bit(segno, free_i->free_segmap)) {
		free_i->free_segments++;

		next = find_next_bit(free_i->free_segmap,
				start_segno + sbi->segs_per_sec, start_segno);
		if (next >= start_segno + sbi->segs_per_sec) {
			if (test_and_clear_bit(secno, free_i->free_secmap))
				free_i->free_sections++;
		}
	}
	spin_unlock(&free_i->segmap_lock);
}

```

#### Checkpoint区域的回写
上述分别描述了对NAT和SIT的回写与更新，而`do_checkpoint`是针对Checkpoint区域的更新。Checkpoint主要涉及两部分，第一部分`f2fs_checkpoint`结构的更新，第二部分是curseg的summary数据的回写。在分析这个函数之前，需要知道元数据的Checkpoint区域在磁盘中是如何保存的，磁盘的保存结构如下：
```
             +---------------------------------------------------------------------------------------------------+
             | f2fs_checkpoint | data summaries | hot node summaries | warm node summaries | cold node summaries |
             +---------------------------------------------------------------------------------------------------+
                              .                 .             
                       .                                   .               
                 .                 compacted summaries                 .        
                 +----------------+-------------------+----------------+
                 |  nat journal   |    sit journal    | data summaries |
                 +----------------+-------------------+----------------+

                 .                  normal summaries                   .        
                 +----------------+-------------------+----------------+
                 |                    data summaries                   |
                 +----------------+-------------------+----------------+
```
其中f2fs_checkpoint、hot/warm/cold node summaries都分别占用一个block的空间。f2fs为了减少Checkpoint的写入开销，将data summaries被设计为可变的。它包含两种写入方式，一种是compacted summaries写入，另一种是normal summaries写入。compacted summaries可以在一次Checkpoint中，减少1~2个page的写入。


##### do_checkpoint函数
下面是简化的`do_checkpoint`函数核心流程，如下所示：
```c
static int do_checkpoint(struct f2fs_sb_info *sbi, struct cp_control *cpc)
{
	// 第一部分，根据curseg，修改f2fs_checkpoint结构的信息
	...
	for (i = 0; i < NR_CURSEG_NODE_TYPE; i++) {
		ckpt->cur_node_segno[i] =
			cpu_to_le32(curseg_segno(sbi, i + CURSEG_HOT_NODE));
		ckpt->cur_node_blkoff[i] =
			cpu_to_le16(curseg_blkoff(sbi, i + CURSEG_HOT_NODE));
		ckpt->alloc_type[i + CURSEG_HOT_NODE] =
				curseg_alloc_type(sbi, i + CURSEG_HOT_NODE);
	}
	//printk("[do-checkpoint] point 3\n");
	for (i = 0; i < NR_CURSEG_DATA_TYPE; i++) {
		ckpt->cur_data_segno[i] =
			cpu_to_le32(curseg_segno(sbi, i + CURSEG_HOT_DATA));
		ckpt->cur_data_blkoff[i] =
			cpu_to_le16(curseg_blkoff(sbi, i + CURSEG_HOT_DATA));
		ckpt->alloc_type[i + CURSEG_HOT_DATA] =
				curseg_alloc_type(sbi, i + CURSEG_HOT_DATA);
	}

	// 第二部分，根据curseg，修改summary的信息
	...

	data_sum_blocks = f2fs_npages_for_summary_flush(sbi, false);

	if (data_sum_blocks < NR_CURSEG_DATA_TYPE)
		__set_ckpt_flags(ckpt, CP_COMPACT_SUM_FLAG);
	else
		__clear_ckpt_flags(ckpt, CP_COMPACT_SUM_FLAG);

	f2fs_write_data_summaries(sbi, start_blk); // 将data summary以及里面的journal写入磁盘

	/* 
	 * node summaries的写回只有在启动和关闭F2FS的时候才会执行，
	 * 如果出现的宕机的情况下，就会失去了UMOUNT的标志，也会失去了所有的NODE SUMMARY
	 * F2FS会进行根据上次checkpoint的情况进行恢复
	 */
	if (__remain_node_summaries(cpc->reason)) {
		f2fs_write_node_summaries(sbi, start_blk); // 将node summary以及里面的journal写入磁盘
		start_blk += NR_CURSEG_NODE_TYPE;
	}

	commit_checkpoint(sbi, ckpt, start_blk); // 将修改后的checkpoint区域的数据提交到设备，对磁盘的元数据进行更新
	
	...

	return 0;
}
```

首先，第一部分主要是针对元数据区域的`f2fs_checkpoint`结构的修改，其实包括将curseg的当前segno，blkoff等写入到`f2fs_checkpoint`中，以便下次重启时可以根据这些信息，重建curseg。

接下来重点讨论，Checkpoint区域的summary的回写，在分析流程之前，需要分析compacted summaries和normal summaries的差别。

**compacted summaries和normal summaries**
通过查看curseg的结构可以知道，curseg管理了(NODE,DATA) X (HOT,WARM,COLD)总共6个的segment，因此也需要管理这6个segment对应的`f2fs_summary_block`。

因此一般情况下，每一次checkpoint时候，应该需要回写6种类型的`f2fs_summary_block`，即6个block到磁盘。

为了减少这部分回写的开销，f2fs针对**DATA**类型`f2fs_summary_block`设计了一种compacted summary block。一般情况下，DATA需要回写3个`f2fs_summary_block`到磁盘(HOT,WARM,COLD)，但是如果使用了compacted summary block，大部分情况下只需要回写1~2个block。

compacted summary block被设计为通过1~2个block保存当前curseg所有的元信息，它的核心设计是**将HOW WARM COLD DATA的元信息混合保存**:

**混合类型Journal保存**
compacted summary block分别维护了一个公用的nat journal，以及sit journal，HOT WARM COLD类型的Journal都会混合保存进入两个journal结构中。

在满足COMPACTED的条件下，系统启动时，F2FS会从磁盘中读取这两个Journal到内存中，分别保存在HOT以及COLD所在的curseg->journal中。

不同类型的journal会在CP时刻，通过`f2fs_flush_sit_entries`函数写入到HOT或者COLD对应的curseg->journal区域中。如果HOT或者COLD对应的curseg->journal区域的空间不够了，就将不同类型的journal保存的segment的信息，直接写入到对应的sit entry block中。

接下来将HOT或者COLD对应的curseg->journal包装为compacted block回写到cp区域中。

**混合类型Summary保存**
b) 以及将HOT,WARM,COLD三种类型的summary混合保存同一个data summaries数组中，它们的差别如下:
```
compacted summary block (4KB)
+------------------+
|nat journal       |
|sit journal       |
|data sum[439]     | data summaries数组大小是439
+------------------+
|                  | 如果需要，会接上一个纯summary数组的block
| data sum[584]    | data summaries数组大小是584
|                  |
+------------------+


normal summary block，表示三种类型的DATA的summary
+--------------------+
|hot data journal    |
|hot data summaries  | data summaries数组大小是512
|                    |
+--------------------+
|warm data journal   |
|warm data summaries | data summaries数组大小是512
|                    |
+--------------------+
|cold data journal   |
|cold data summaries | data summaries数组大小是512
|                    |
+--------------------+
```

根据上面的描述，不同类型的summary block的可以保存的summary的大小，可以得到
HOT,WARM,COLD DATA这三种类型，如果目前**加起来**仅使用了
1. 少于439的block(只修改了439个f2fs_summary)，那么可以通过compacted回写方式进行回写，即通过一个compacted summary block完成回写，需要回写1个block。
2. 大于439，少于439+584=1023个block，那么可以通过compacted回写方式进行回写，即可以通过compacted summary block加一个纯summary block的方式保存所有信息，需要回写2个block。
3. 大于1023的情况下，即和normal summary block同样的回写情况，那么就会使用normal summary block的回写方式完成回写，即回写3个block。(因为大于1023情况下，如果继续使用compacted回写，最差的情况下要回写4个block)

接下来进行代码分析：
```c

// 根据需要回写的summary的数目，返回需要写回的block的数目，返回值有1、2、3
data_sum_blocks = f2fs_npages_for_summary_flush(sbi, false); 

// 如果data_sum_blocks = 1 或者 2，则表示回写1个或者2个block，则设置CP_COMPACT_SUM_FLAG标志
if (data_sum_blocks < NR_CURSEG_DATA_TYPE) // NR_CURSEG_DATA_TYPE = 3
		__set_ckpt_flags(ckpt, CP_COMPACT_SUM_FLAG);
else
	__clear_ckpt_flags(ckpt, CP_COMPACT_SUM_FLAG);

// 然后将summary写入磁盘
f2fs_write_data_summaries(sbi, start_blk); // 将data summary以及里面的journal写入磁盘
```
`f2fs_write_data_summaries`函数会判断一下是否设置了CP_COMPACT_SUM_FLAG标志，采取不同的方法写入磁盘
```c
void f2fs_write_data_summaries(struct f2fs_sb_info *sbi, block_t start_blk)
{
	if (is_set_ckpt_flags(sbi, CP_COMPACT_SUM_FLAG))
		write_compacted_summaries(sbi, start_blk);
	else
		write_normal_summaries(sbi, start_blk, CURSEG_HOT_DATA);
}
```
`write_compacted_summaries`函数会根据上述的compacted block的数据分布，将数据写入到磁盘中
```c
static void write_compacted_summaries(struct f2fs_sb_info *sbi, block_t blkaddr)
{
	struct page *page;
	unsigned char *kaddr;
	struct f2fs_summary *summary;
	struct curseg_info *seg_i;
	int written_size = 0;
	int i, j;
	int datatypes = CURSEG_COLD_DATA;
#ifdef CONFIG_F2FS_COMPRESSION
	datatypes = CURSEG_BG_COMPR_DATA;
#endif

	page = f2fs_grab_meta_page(sbi, blkaddr++);
	kaddr = (unsigned char *)page_address(page);
	memset(kaddr, 0, PAGE_SIZE);

	/* Step 1: write nat cache */
	seg_i = CURSEG_I(sbi, CURSEG_HOT_DATA); // 第一步写nat的journal
	memcpy(kaddr, seg_i->journal, SUM_JOURNAL_SIZE);
	written_size += SUM_JOURNAL_SIZE;

	/* Step 2: write sit cache */
	seg_i = CURSEG_I(sbi, CURSEG_COLD_DATA);
	memcpy(kaddr + written_size, seg_i->journal, SUM_JOURNAL_SIZE); // 第二步写sit的journal
	written_size += SUM_JOURNAL_SIZE;

	/* Step 3: write summary entries */
	for (i = CURSEG_HOT_DATA; i <= datatypes; i++) { // 开始写summary
		unsigned short blkoff;
		seg_i = CURSEG_I(sbi, i);
		if (sbi->ckpt->alloc_type[i] == SSR)
			blkoff = sbi->blocks_per_seg;
		else
			blkoff = curseg_blkoff(sbi, i);

		for (j = 0; j < blkoff; j++) {
			if (!page) { // 如果f2fs compacted block写不下，则创建一个纯summary的block
				page = f2fs_grab_meta_page(sbi, blkaddr++);
				kaddr = (unsigned char *)page_address(page);
				memset(kaddr, 0, PAGE_SIZE);
				written_size = 0;
			}
			summary = (struct f2fs_summary *)(kaddr + written_size);
			*summary = seg_i->sum_blk->entries[j];
			written_size += SUMMARY_SIZE;

			if (written_size + SUMMARY_SIZE <= PAGE_SIZE -
							SUM_FOOTER_SIZE)
				continue;

			set_page_dirty(page); // 如果超过了compaced sum block可以承载的极限，就设置这个block是脏，等待回写
			f2fs_put_page(page, 1);
			page = NULL;
		}
	}
	if (page) {
		set_page_dirty(page);
		f2fs_put_page(page, 1);
	}
}
```

`write_normal_summaries`函数则是简单地将按照HOT/WARM/COLD的顺序写入到checkpoint区域中
```c
static void write_normal_summaries(struct f2fs_sb_info *sbi,
					block_t blkaddr, int type)
{
	int i, end;
	if (IS_DATASEG(type))
		end = type + NR_CURSEG_DATA_TYPE;
	else
		end = type + NR_CURSEG_NODE_TYPE;

	for (i = type; i < end; i++) // 根据 HOW WARM COLD 都写入磁盘
		write_current_sum_page(sbi, i, blkaddr + (i - type));
}

static void write_current_sum_page(struct f2fs_sb_info *sbi,
						int type, block_t blk_addr)
{
	struct curseg_info *curseg = CURSEG_I(sbi, type);
	struct page *page = f2fs_grab_meta_page(sbi, blk_addr);
	struct f2fs_summary_block *src = curseg->sum_blk;
	struct f2fs_summary_block *dst;

	dst = (struct f2fs_summary_block *)page_address(page);
	memset(dst, 0, PAGE_SIZE);

	mutex_lock(&curseg->curseg_mutex);

	down_read(&curseg->journal_rwsem);
	memcpy(&dst->journal, curseg->journal, SUM_JOURNAL_SIZE);
	up_read(&curseg->journal_rwsem);

	memcpy(dst->entries, src->entries, SUM_ENTRY_SIZE);
	memcpy(&dst->footer, &src->footer, SUM_FOOTER_SIZE);

	mutex_unlock(&curseg->curseg_mutex);

	set_page_dirty(page);
	f2fs_put_page(page, 1);
}
```




























