## 数据恢复的代码的实现与分析


### 后滚恢复
后滚恢复的代码实现主要是对Checkpoint数据的读取与恢复。Checkpoint的数据会在F2FS启动的时候，分两阶段进行恢复，第一阶段恢复`f2fs_checkpoint`相关的数据，第二阶段恢复与curseg相关的数据，下面进行分析。

F2FS的初始化函数`f2fs_fill_super`中与后滚恢复相关的代码有
```
static int f2fs_fill_super(struct super_block *sb, void *data, int silent)
{
	...
	err = f2fs_get_valid_checkpoint(sbi); // 恢复f2fs_checkpoint
	...
	err = f2fs_build_segment_manager(sbi); // 恢复curseg
	...
}
```

#### 恢复f2fs_checkpoint
具体的恢复流程是:
1. 分配f2fs_checkpoint的堆空间
2. 然后根据sbi记录的checkpoint的起始地址，读取cp1，cp2的数据出来。

```c
int f2fs_get_valid_checkpoint(struct f2fs_sb_info *sbi)
{

	sbi->ckpt = f2fs_kzalloc(sbi, array_size(blk_size, cp_blks),
				 GFP_KERNEL); // 分配空间
	if (!sbi->ckpt)
		return -ENOMEM;

	cp_start_blk_no = le32_to_cpu(fsb->cp_blkaddr); // 从sbi获得checkpoint的起始地址
	cp1 = validate_checkpoint(sbi, cp_start_blk_no, &cp1_version); // 读取 & 检查合法性

	/* The second checkpoint pack should start at the next segment */
	cp_start_blk_no += ((unsigned long long)1) <<
				le32_to_cpu(fsb->log_blocks_per_seg);
	cp2 = validate_checkpoint(sbi, cp_start_blk_no, &cp2_version);

	if (cp1 && cp2) { // 根据版本的老旧决定使用哪个版本的cp
		if (ver_after(cp2_version, cp1_version))
			cur_page = cp2;
		else
			cur_page = cp1;
	} else if (cp1) {
		cur_page = cp1;
	} else if (cp2) {
		cur_page = cp2;
	} else {
		goto fail_no_cp;
	}

	cp_block = (struct f2fs_checkpoint *)page_address(cur_page); // 这个cur_page表示正在使用的cp
	memcpy(sbi->ckpt, cp_block, blk_size); // 复制数据到sbi中，用于运行中的管理

	/* Sanity checking of checkpoint */
	if (f2fs_sanity_check_ckpt(sbi))
		goto free_fail_no_cp;

	if (cur_page == cp1)
		sbi->cur_cp_pack = 1;
	else
		sbi->cur_cp_pack = 2;

	if (cp_blks <= 1) // 一般情况下，cp_blks=1，所以这里就是完成了cp的初始化了
		goto done;

	cp_blk_no = le32_to_cpu(fsb->cp_blkaddr);
	if (cur_page == cp2)
		cp_blk_no += 1 << le32_to_cpu(fsb->log_blocks_per_seg);

	for (i = 1; i < cp_blks; i++) { // 注意这里构建bitmap，存疑，如何构建，理论上会占用journal
		void *sit_bitmap_ptr;
		unsigned char *ckpt = (unsigned char *)sbi->ckpt;

		cur_page = f2fs_get_meta_page(sbi, cp_blk_no + i);
		sit_bitmap_ptr = page_address(cur_page);
		memcpy(ckpt + i * blk_size, sit_bitmap_ptr, blk_size);
		f2fs_put_page(cur_page, 1);
	}
done:
	f2fs_put_page(cp1, 1);
	f2fs_put_page(cp2, 1);
	return 0;

free_fail_no_cp:
	f2fs_put_page(cp1, 1);
	f2fs_put_page(cp2, 1);
fail_no_cp:
	kfree(sbi->ckpt);
	return -EINVAL;
}
```

#### 恢复curseg
这一个步骤主要在`f2fs_build_segment_manager`函数的`build_curseg`函数中完成:
1. 分配空间给curseg_info数组，默认是6个，表示(HOT,WARM,COLD) X (DATA,NODE)的6种分配方式。
2. 执行`restore_curseg_summaries`函数，从`f2fs_checkpoint`中读取数据，以及根据compacted summary block和normal summary block恢复数据。

```c
static int build_curseg(struct f2fs_sb_info *sbi)
{
	struct curseg_info *array;
	int i;

	array = f2fs_kzalloc(sbi, array_size(NR_CURSEG_TYPE, sizeof(*array)),
			     GFP_KERNEL);
	if (!array)
		return -ENOMEM;

	SM_I(sbi)->curseg_array = array;

	// 初始化curseg的空间
	for (i = 0; i < NR_CURSEG_TYPE; i++) {
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

	// 从磁盘中，读取恢复curseg的信息
	return restore_curseg_summaries(sbi);
}
```

**restore_curseg_summaries函数**

`restore_curseg_summaries`第一步先检查CP_COMPACT_SUM_FLAG的标志，这个标志用于检查是否按COMPACTED的方式读取data summary。第二步就是通过NORMAL的方式读取NODE的summary。
```c
static int restore_curseg_summaries(struct f2fs_sb_info *sbi)
{
	struct f2fs_journal *sit_j = CURSEG_I(sbi, CURSEG_COLD_DATA)->journal;
	struct f2fs_journal *nat_j = CURSEG_I(sbi, CURSEG_HOT_DATA)->journal;
	int type = CURSEG_HOT_DATA;
	int err;

	if (is_set_ckpt_flags(sbi, CP_COMPACT_SUM_FLAG)) {
		int npages = f2fs_npages_for_summary_flush(sbi, true); // 检查需要读取一个block还是2个block

		if (npages >= 2)
			f2fs_ra_meta_pages(sbi, start_sum_block(sbi), npages,
							META_CP, true); // 先预读出来

		/* restore for compacted data summary */
		read_compacted_summaries(sbi); //恢复 compacted summary
		type = CURSEG_HOT_NODE;
	}

	if (__exist_node_summaries(sbi)) // 如果没有出现宕机，则预测这几个block
		f2fs_ra_meta_pages(sbi, sum_blk_addr(sbi, NR_CURSEG_TYPE, type),
					NR_CURSEG_TYPE - type, META_CP, true);

	/*
	 * 如果没有COMPACTED标识，则DATA和NODE都使用NORMAL的方式进行恢复
	 * */
	for (; type <= CURSEG_COLD_NODE; type++) {
		err = read_normal_summaries(sbi, type);
		if (err)
			return err;
	}

	/* sanity check for summary blocks */
	if (nats_in_cursum(nat_j) > NAT_JOURNAL_ENTRIES ||
			sits_in_cursum(sit_j) > SIT_JOURNAL_ENTRIES)
		return -EINVAL;

	return 0;
}
```
**read_normal_summaries函数**
这个函数对于F2FS的正常关闭，重新启动时读取的summary的方式都是类似的，都是根据HOT/WARM/COLD的顺序，读取对应的block，然后将数据保存到curseg对应的类型当中。这里**重点考虑出现了宕机的情况的恢复**。

F2FS可以通过判断`f2fs_checkpoint`的`CP_UMOUNT_FLAG`标志，可以知道系统是否出现了宕机的情况，**由于F2FS只会在启动和关闭的时候回写node summaries，因此宕机下毫无疑问会丢失所有的node summary**。对函数的DATA部分进行简化后，函数如下。

```c
static int read_normal_summaries(struct f2fs_sb_info *sbi, int type)
{

	segno = le32_to_cpu(ckpt->cur_node_segno[type -
							CURSEG_HOT_NODE]); // 获取上次最后时刻使用的segno
	blk_off = le16_to_cpu(ckpt->cur_node_blkoff[type -
							CURSEG_HOT_NODE]); // 以及用到这个segno的第几个block

	if (__exist_node_summaries(sbi)) 
		blk_addr = sum_blk_addr(sbi, NR_CURSEG_NODE_TYPE,
						type - CURSEG_HOT_NODE); // 如果没有宕机的情况下，从checkpoint中恢复
	else
		blk_addr = GET_SUM_BLOCK(sbi, segno); // 出现了宕机，则从SSA中恢复

	new = f2fs_get_meta_page(sbi, blk_addr); // 根据地址读取出f2fs_summary_block
	sum = (struct f2fs_summary_block *)page_address(new); // 转换结构

	if (__exist_node_summaries(sbi)) { // 如果没有宕机的情况下，将每一个ns重新置为0
		struct f2fs_summary *ns = &sum->entries[0];
		int i;
		for (i = 0; i < sbi->blocks_per_seg; i++, ns++) {
			ns->version = 0;
			ns->ofs_in_node = 0;
		}
	} else {
		f2fs_restore_node_summary(sbi, segno, sum); // 如果出现了宕机，则执行这个这个函数
	}

	/* set uncompleted segment to curseg */
	curseg = CURSEG_I(sbi, type);
	mutex_lock(&curseg->curseg_mutex);

	/* 复制journal到curseg */
	/* update journal info */
	down_write(&curseg->journal_rwsem);
	memcpy(curseg->journal, &sum->journal, SUM_JOURNAL_SIZE); 
	up_write(&curseg->journal_rwsem);

	/* 复制summaries到curseg */
	memcpy(curseg->sum_blk->entries, sum->entries, SUM_ENTRY_SIZE);
	memcpy(&curseg->sum_blk->footer, &sum->footer, SUM_FOOTER_SIZE);
	curseg->next_segno = segno;
	reset_curseg(sbi, type, 0);
	curseg->alloc_type = ckpt->alloc_type[type];
	curseg->next_blkoff = blk_off;
	mutex_unlock(&curseg->curseg_mutex);
	f2fs_put_page(new, 1);
	return 0;
}
```

然后分析 `f2fs_restore_node_summary` 函数，这个函数主要是根据宕机时，根据最后一次checkpoint的node正在使用的segno找到对应的保存在这个segno的所有node page。进而遍历整个segment的node page恢复改summary对应的nid。

！！！需要注意的时候这种恢复方式仅仅可以对node使用，不能对data使用，这是因为node summary仅需要记录一个nid，但是对于data的summary，需要同时记录ofs_in_node，这个信息无法从node中直接恢复出来，因此不能使用这种方式。
```c
void f2fs_restore_node_summary(struct f2fs_sb_info *sbi,
			unsigned int segno, struct f2fs_summary_block *sum)
{
	struct f2fs_node *rn;
	struct f2fs_summary *sum_entry;
	block_t addr;
	int i, idx, last_offset, nrpages;

	/* scan the node segment */
	last_offset = sbi->blocks_per_seg;
	addr = START_BLOCK(sbi, segno); // node block所在的segment
	sum_entry = &sum->entries[0];

	for (i = 0; i < last_offset; i += nrpages, addr += nrpages) {
		nrpages = min(last_offset - i, BIO_MAX_PAGES);

		/* readahead node pages */
		f2fs_ra_meta_pages(sbi, addr, nrpages, META_POR, true); // 预读NODE PAGES

		for (idx = addr; idx < addr + nrpages; idx++) { // 遍历这个segment的node，然后将nid赋值改curseg
			struct page *page = f2fs_get_tmp_page(sbi, idx);

			rn = F2FS_NODE(page);
			sum_entry->nid = rn->footer.nid;
			sum_entry->version = 0;
			sum_entry->ofs_in_node = 0;
			sum_entry++;
			f2fs_put_page(page, 1);
		}

		invalidate_mapping_pages(META_MAPPING(sbi), addr,
							addr + nrpages);
	}
}
```

## 原理未名
### 前滚恢复
前滚恢复的主要作用是恢复那些执行了fsync等同步操作写入了磁盘，但是在CP回写之前就宕机的数据(F2FS为了提高效率，不会在fsync之后马上就回写CP，而是加上了某些标志然后回写fsync关联的数据，用于前滚恢复，从而避免大开销的写CP操作)。

**典型Crash场景分析：**dnode block指的是直接保存data block的node page，包含`f2fs_inode`以及`direct_node`这两种node page。参考下图，在blkaddr=2和3之间发生了checkpoint，cpver表示的checkpoint版本号由11增加为12，nat bitmap也写入到checkpoint中。一个文件进行修改了之后，执行fsync，使用了blkaddr=4和blkaddr=5的dnode block用于准备node page数据的写入，但是此时出现了Crash，导致目前的nat bitmap丢失了。因此node page的索引都会丢失，即使node page已经在crash之前写入了磁盘。

```
 DNODE BLOCK
+-----------+
|blkaddr=2  | <-node.footer->cpver=11， curseg->next_blkoff=3
+-----------+ <-Checkpoint, cpver  11=>12，cp是先增加版本号再回写到磁盘，因此stable cpver=12
|blkaddr=3  | <-node.footer->cpver=12， curseg->next_blkoff=4
+-----------+ <-fsync
|blkaddr=4  | <-node.footer->cpver=12， curseg->next_blkoff=5
+-----------+ 
|blkaddr=5  | <-node.footer->cpver=12， curseg->next_blkoff=6
+-----------+ <-Crash
|blkaddr=6  | 
+-----------+
```
**典型前滚恢复流程分析：**系统在宕机后重新启动时，恢复到上一次的稳定的checkpoint点，即cpver=12的时刻。此时根据curseg->next_blkoff的值即=，即3，找到下一个即将使用的blkaddr。然后找到blkaddr=3，然后读取出来，然后根据一定的条件判断(如通常情况，必须node和checkpoint的cpver一致才可以恢复)，是否是可以恢复的node page，然后继续判断下一个curseg->next_blkoff的数据，直到为空。


前滚恢复的具体实现在recovery.c，核心函数是 `f2fs_recover_fsync_data` ，下面进行分析。

### Recovery的代码分析
F2FS启动的时候，会调用 `f2fs_recover_fsync_data` 函数开始恢复数据，步骤如下：
1. 先通过`find_fsync_dnodes`找出所有的可以恢复的dnode对应的inode(有可能dnode就是inode本身)，放入到一个list里面。
2. 恢复inode list里面的所有的node page。

```c
int f2fs_recover_fsync_data(struct f2fs_sb_info *sbi, bool check_only)
{
	...
	// 找出所有的可以恢复的dnode对应的inode(有可能dnode就是inode本身)，放入到一个list里面
	err = find_fsync_dnodes(sbi, &inode_list, check_only); 
	
	...
	// 恢复inode list里面的数据
	err = recover_data(sbi, &inode_list, &dir_list);
	...
}
```

**f2fs_recover_fsync_data函数**
```c
static int find_fsync_dnodes(struct f2fs_sb_info *sbi, struct list_head *head, bool check_only)
{
	...
	curseg = CURSEG_I(sbi, CURSEG_WARM_NODE); // 因为是基于dnode进行恢复，因此是WARM NODE

	blkaddr = NEXT_FREE_BLKADDR(sbi, curseg); // 遍历next_blkoff组成的node list
	while (1) {

		page = f2fs_get_tmp_page(sbi, blkaddr); // 根据blkaddr读取该地址对应的node page的数据
		
		if (!is_recoverable_dnode(page)) // 比较cpver的版本
			break;

		if (!is_fsync_dnode(page)) // 前滚恢复只能恢复被fsync的node page
			goto next;

		/**
		 * 如果是NULL，则表示inode不在list中
		 * 如不是NULL，则表示这个inode已经在list中，不需要加入了
		 **/
		entry = get_fsync_inode(head, ino_of_node(page)); // 
		if (!entry) { 
			bool quota_inode = false;

			if (!check_only &&
					IS_INODE(page) && is_dent_dnode(page)) {  // 如果是dentry的inode，则先恢复
				err = f2fs_recover_inode_page(sbi, page);
				if (err)
					break;
				quota_inode = true;
			}

			/*
			 * CP | dnode(F) | inode(DF)
			 * For this case, we should not give up now.
			 */
			entry = add_fsync_inode(sbi, head, ino_of_node(page),
								quota_inode);
			if (IS_ERR(entry)) {
				err = PTR_ERR(entry);
				if (err == -ENOENT) {
					err = 0;
					goto next;
				}
				break;
			}
		}
		entry->blkaddr = blkaddr;

		if (IS_INODE(page) && is_dent_dnode(page))
			entry->last_dentry = blkaddr; // 将恢复得inode串成一个列
next:
		/* sanity check in order to detect looped node chain */
		if (++loop_cnt >= free_blocks ||
			blkaddr == next_blkaddr_of_node(page)) {
			f2fs_msg(sbi->sb, KERN_NOTICE,
				"%s: detect looped node chain, "
				"blkaddr:%u, next:%u",
				__func__, blkaddr, next_blkaddr_of_node(page));
			err = -EINVAL;
			break;
		}

		/** 
		  * check next segment
		  * 
		  * F2FS分配物理地址的时候，会将本次分配的blkaddr和下一次分配的blkaddr，通过footer连接成一个list，
		  * 前滚恢复通过这个list找到下一个被分配的blkaddr，直到没有分配为止
		  * */
		blkaddr = next_blkaddr_of_node(page);
		f2fs_put_page(page, 1);

		f2fs_ra_meta_pages_cond(sbi, blkaddr); // 预读下一个blkaddr
	}
	f2fs_put_page(page, 1);
	return err;
}
```

**recover_data函数**
```c
static int recover_data(struct f2fs_sb_info *sbi, struct list_head *inode_list,
						struct list_head *dir_list)
{
	struct curseg_info *curseg;
	struct page *page = NULL;
	int err = 0;
	block_t blkaddr;

	/* get node pages in the current segment */
	curseg = CURSEG_I(sbi, CURSEG_WARM_NODE);
	blkaddr = NEXT_FREE_BLKADDR(sbi, curseg);

	while (1) {
		struct fsync_inode_entry *entry;

		if (!f2fs_is_valid_meta_blkaddr(sbi, blkaddr, META_POR))
			break;

		f2fs_ra_meta_pages_cond(sbi, blkaddr);

		page = f2fs_get_tmp_page(sbi, blkaddr);

		if (!is_recoverable_dnode(page)) {
			f2fs_put_page(page, 1);
			break;
		}
		
		entry = get_fsync_inode(inode_list, ino_of_node(page)); // 从inodelist中取出一个entry
		if (!entry)
			goto next;
		/*
		 * inode(x) | CP | inode(x) | dnode(F)
		 * In this case, we can lose the latest inode(x).
		 * So, call recover_inode for the inode update.
		 */
		if (IS_INODE(page))
			recover_inode(entry->inode, page);
		if (entry->last_dentry == blkaddr) {
			err = recover_dentry(entry->inode, page, dir_list);
			if (err) {
				f2fs_put_page(page, 1);
				break;
			}
		}
		err = do_recover_data(sbi, entry->inode, page); // 进行恢复
		if (err) {
			f2fs_put_page(page, 1);
			break;
		}

		if (entry->blkaddr == blkaddr)
			del_fsync_inode(entry);
next:
		/* check next segment */
		blkaddr = next_blkaddr_of_node(page);
		f2fs_put_page(page, 1);
	}
	if (!err)
		f2fs_allocate_new_segments(sbi);
	return err;
}

```
