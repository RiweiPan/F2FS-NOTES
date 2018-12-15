## Checkpoint流程
### Checkpoint的介绍
从F2FS的磁盘布局可以了解到，F2FS有一个元数据区域，用于集中管理磁盘所有的block的信息，因此F2FS使用了检查点(checkpointing)机制去维护文件系统的恢复点(recovery point)，这个恢复点用于在系统突然崩溃的时候，元数据区域依然能够正确地将数据重新读取出来。因此保证元数据区域的有效性以及恢复性。当F2FS需要通过 `fsync` 或 `umount` 等命令对系统进行同步的时候，F2FS会触发一次Checkpoint机制，它会完成以下的工作:
1. 页缓存的脏node和dentry block会刷写回到磁盘;
2. 挂起系统其他的写行为，如create，unlink，mkdir；
3. 将系统的meta data，如NAT、SIT、SSA的数据写回磁盘;
4. 更新checkpoint的状态，包括checkpoint的版本，NAT和SIT的bitmaps以及journals，SSA，Orphan inode


### Checkpoint的时机
从上面的描述可以可以知道，CP是一个开销很大的操作，因此合理选取CP时机，能够很好地提高性能。CP的触发时机有:
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

### Checkpoint的具体流程
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
	f2fs_flush_nat_entries(sbi, cpc); // 刷写所有nat entries到磁盘
	f2fs_flush_sit_entries(sbi, cpc); // 刷写所有sit entries到磁盘，处理dirty prefree segments
	err = do_checkpoint(sbi, cpc); // checkpoint核心流程
	f2fs_clear_prefree_segments(sbi, cpc); // 清除dirty prefree segments的dirty标记
	unblock_operations(sbi); //恢复文件系统的操作
	...
	f2fs_update_time(sbi, CP_TIME); // 更新CP的时间
	...
}

```

#### NAT和SIT的刷写
`f2fs_flush_nat_entries` 和 `f2fs_flush_sit_entries` 的作用是将暂存在ram的nat entry合sit entry都回写到Journa或磁盘l当中:
##### f2fs_flush_nat_entries函数
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
	 * __gang_lookup_nat_set 这个函数就是从radix tree读取set_idx开始，连续读取SETVEC_SIZE这么多个nat_entry_set，保存在setvec
	 * 因此这个while循环的作用就是读取SIT缓存的所有nat_entry_set，然后加入到sets这个链表当中
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

/*
 * 将nat_entry_set对应的nat_entry信息，写入到curseg->journal中
 */
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
		!__has_cursum_space(journal, set->entry_cnt, NAT_JOURNAL)) /* 当journal空间不够了，旧刷写到磁盘中 */
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
		if (nat_get_blkaddr(ne) == NULL_ADDR) {
			add_free_nid(sbi, nid, false, true);
		} else {
			spin_lock(&NM_I(sbi)->nid_list_lock);
			update_free_nid_bitmap(sbi, nid, false, false);
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
##### f2fs_flush_sit_entries函数

主要过程跟 `f2fs_flush_nat_entries` 类似，将sbi缓存的seg_entry刷写到journal或磁盘中当中

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

#### do_checkpoint函数
```c
static int do_checkpoint(struct f2fs_sb_info *sbi, struct cp_control *cpc)
{
	struct f2fs_checkpoint *ckpt = F2FS_CKPT(sbi);
	struct f2fs_nm_info *nm_i = NM_I(sbi);
	unsigned long orphan_num = sbi->im[ORPHAN_INO].ino_num, flags;
	block_t start_blk;
	unsigned int data_sum_blocks, orphan_blocks;
	__u32 crc32 = 0;
	int i;
	int cp_payload_blks = __cp_payload(sbi);
	struct super_block *sb = sbi->sb;
	struct curseg_info *seg_i = CURSEG_I(sbi, CURSEG_HOT_NODE);
	u64 kbytes_written;
	int err;

	/*
	 * Flush all the NAT/SIT pages
	 * 将缓存的META区域的所有dirty page全部刷写回磁盘
	 * */
	while (get_pages(sbi, F2FS_DIRTY_META)) {
		f2fs_sync_meta_pages(sbi, META, LONG_MAX, FS_CP_META_IO);
		if (unlikely(f2fs_cp_error(sbi)))
			return -EIO;
	}

	/*
	 * modify checkpoint
	 * version number is already updated
	 *
	 * 更新checkpoint结构的数据
	 *
	 */
	ckpt->elapsed_time = cpu_to_le64(get_mtime(sbi, true)); // 更新修改时间
	ckpt->free_segment_count = cpu_to_le32(free_segments(sbi)); // 已经使用的segment数目
	for (i = 0; i < NR_CURSEG_NODE_TYPE; i++) { // 遍历HOW WARM COLD的NODE，以及更新cur_seg对应的segno，blkoff，alloc_type到CP
		ckpt->cur_node_segno[i] =
			cpu_to_le32(curseg_segno(sbi, i + CURSEG_HOT_NODE));
		ckpt->cur_node_blkoff[i] =
			cpu_to_le16(curseg_blkoff(sbi, i + CURSEG_HOT_NODE));
		ckpt->alloc_type[i + CURSEG_HOT_NODE] =
				curseg_alloc_type(sbi, i + CURSEG_HOT_NODE);
	}
	for (i = 0; i < NR_CURSEG_DATA_TYPE; i++) { // 遍历HOW WARM COLD的DATA，以及更新cur_seg对应的segno，blkoff，alloc_type到CP
		ckpt->cur_data_segno[i] =
			cpu_to_le32(curseg_segno(sbi, i + CURSEG_HOT_DATA));
		ckpt->cur_data_blkoff[i] =
			cpu_to_le16(curseg_blkoff(sbi, i + CURSEG_HOT_DATA));
		ckpt->alloc_type[i + CURSEG_HOT_DATA] =
				curseg_alloc_type(sbi, i + CURSEG_HOT_DATA);
	}

	/* 2 cp  + n data seg summary + orphan inode blocks */
	data_sum_blocks = f2fs_npages_for_summary_flush(sbi, false);
	spin_lock_irqsave(&sbi->cp_lock, flags);
	if (data_sum_blocks < NR_CURSEG_DATA_TYPE) // 如果三种WARM HOT COLD都可以在一个page里面完成，那么就设置COMPACT标志
		__set_ckpt_flags(ckpt, CP_COMPACT_SUM_FLAG);
	else
		__clear_ckpt_flags(ckpt, CP_COMPACT_SUM_FLAG);
	spin_unlock_irqrestore(&sbi->cp_lock, flags);

	orphan_blocks = GET_ORPHAN_BLOCKS(orphan_num); // 恢复孤儿节点
	ckpt->cp_pack_start_sum = cpu_to_le32(1 + cp_payload_blks +
			orphan_blocks);

	if (__remain_node_summaries(cpc->reason))
		ckpt->cp_pack_total_block_count = cpu_to_le32(F2FS_CP_PACKS+
				cp_payload_blks + data_sum_blocks +
				orphan_blocks + NR_CURSEG_NODE_TYPE);
	else
		ckpt->cp_pack_total_block_count = cpu_to_le32(F2FS_CP_PACKS +
				cp_payload_blks + data_sum_blocks +
				orphan_blocks);

	/* update ckpt flag for checkpoint */
	update_ckpt_flags(sbi, cpc); // 更新checkpoint的标志

	/*
	 * update SIT/NAT bitmap
	 * 更新bitmap
	 * */
	get_sit_bitmap(sbi, __bitmap_ptr(sbi, SIT_BITMAP));
	get_nat_bitmap(sbi, __bitmap_ptr(sbi, NAT_BITMAP));

	crc32 = f2fs_crc32(sbi, ckpt, le32_to_cpu(ckpt->checksum_offset));
	*((__le32 *)((unsigned char *)ckpt +
				le32_to_cpu(ckpt->checksum_offset)))
				= cpu_to_le32(crc32);

	start_blk = __start_cp_next_addr(sbi); // 获取cp的起始地址，一共有两个

	/* write nat bits */
	if (enabled_nat_bits(sbi, cpc)) {
		__u64 cp_ver = cur_cp_version(ckpt); // 获取CP的version
		block_t blk;

		cp_ver |= ((__u64)crc32 << 32);
		*(__le64 *)nm_i->nat_bits = cpu_to_le64(cp_ver);

		blk = start_blk + sbi->blocks_per_seg - nm_i->nat_bits_blocks;
		for (i = 0; i < nm_i->nat_bits_blocks; i++)
			f2fs_update_meta_page(sbi, nm_i->nat_bits +
					(i << F2FS_BLKSIZE_BITS), blk + i);

		/* Flush all the NAT BITS pages */
		while (get_pages(sbi, F2FS_DIRTY_META)) {
			f2fs_sync_meta_pages(sbi, META, LONG_MAX,
							FS_CP_META_IO);
			if (unlikely(f2fs_cp_error(sbi)))
				return -EIO;
		}
	}

	/*
	 * write out checkpoint buffer at block 0
	 * 更新另外一个CP
	 * */
	f2fs_update_meta_page(sbi, ckpt, start_blk++);

	/*
	 * 刷写目前的内存中的checkpoint信息到磁盘
	 * */
	for (i = 1; i < 1 + cp_payload_blks; i++)
		f2fs_update_meta_page(sbi, (char *)ckpt + i * F2FS_BLKSIZE,
							start_blk++);

	if (orphan_num) {
		write_orphan_inodes(sbi, start_blk);
		start_blk += orphan_blocks;
	}

	// start_blk经过多次的累加后，进入到summary区域
	f2fs_write_data_summaries(sbi, start_blk); // 将data summary以及里面的journal写入磁盘
	start_blk += data_sum_blocks;

	/* Record write statistics in the hot node summary */
	kbytes_written = sbi->kbytes_written;
	if (sb->s_bdev->bd_part)
		kbytes_written += BD_PART_WRITTEN(sbi);

	seg_i->journal->info.kbytes_written = cpu_to_le64(kbytes_written);

	if (__remain_node_summaries(cpc->reason)) {
		f2fs_write_node_summaries(sbi, start_blk); // 将node summary以及里面的journal写入磁盘
		start_blk += NR_CURSEG_NODE_TYPE;
	}

	/* update user_block_counts */
	sbi->last_valid_block_count = sbi->total_valid_block_count;
	percpu_counter_set(&sbi->alloc_valid_block_count, 0);

	/* Here, we have one bio having CP pack except cp pack 2 page */
	f2fs_sync_meta_pages(sbi, META, LONG_MAX, FS_CP_META_IO);

	/* wait for previous submitted meta pages writeback */
	wait_on_all_pages_writeback(sbi);

	if (unlikely(f2fs_cp_error(sbi)))
		return -EIO;

	/* flush all device cache */
	err = f2fs_flush_device_cache(sbi);
	if (err)
		return err;

	/* barrier and flush checkpoint cp pack 2 page if it can */
	commit_checkpoint(sbi, ckpt, start_blk);
	wait_on_all_pages_writeback(sbi);

	f2fs_release_ino_entry(sbi, false);

	if (unlikely(f2fs_cp_error(sbi)))
		return -EIO;

	clear_sbi_flag(sbi, SBI_IS_DIRTY);
	clear_sbi_flag(sbi, SBI_NEED_CP);
	__set_cp_next_pack(sbi); // 切换cp？

	/*
	 * redirty superblock if metadata like node page or inode cache is
	 * updated during writing checkpoint.
	 */
	if (get_pages(sbi, F2FS_DIRTY_NODES) ||
			get_pages(sbi, F2FS_DIRTY_IMETA))
		set_sbi_flag(sbi, SBI_IS_DIRTY);

	f2fs_bug_on(sbi, get_pages(sbi, F2FS_DIRTY_DENTS));

	return 0;
}

/*
 * 将cur_seg的journal和entries和footer，刷写入CP-CURSEG区域对应的f2fs_summary_block中
 * 注意CP区域的保存的只有CURSEG的数据，而其他SUMMARY_BLOCK，保存在SSA区域当中
 * */
void f2fs_write_data_summaries(struct f2fs_sb_info *sbi, block_t start_blk)
{
	if (is_set_ckpt_flags(sbi, CP_COMPACT_SUM_FLAG))
		write_compacted_summaries(sbi, start_blk);
	else
		write_normal_summaries(sbi, start_blk, CURSEG_HOT_DATA);
}

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

/*
 * blk_addr HOW/WARM/COLD的summary的起始地址
 * */
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
	memcpy(&dst->journal, curseg->journal, SUM_JOURNAL_SIZE); // 将cur_seg->journal的数据刷写如f2fs_summary_block->journal中
	up_read(&curseg->journal_rwsem);

	memcpy(dst->entries, src->entries, SUM_ENTRY_SIZE); // 将cur_seg->entries的数据刷写如f2fs_summary_block->entries中
	memcpy(&dst->footer, &src->footer, SUM_FOOTER_SIZE); // 将cur_seg->footer的数据刷写如f2fs_summary_block->footer中

	mutex_unlock(&curseg->curseg_mutex);

	set_page_dirty(page);
	f2fs_put_page(page, 1);
}
```




























