## 前滚恢复和Recovery的作用与实现
前滚恢复的主要作用是恢复那些执行了fsync等同步操作写入了磁盘，但是在CP回写之前就宕机的数据(F2FS为了提高效率，不会在fsync之后马上就回写CP，而是加上了某些标志然后回写fsync关联的数据，用于前滚恢复，从而避免大开销的写CP操作)。

考虑一个场景: 数据进行写入，数据已经写入了磁盘(执行了类似fsync的操作)，但是CP回写之前出现了宕机，那么系统首先会使用后滚恢复恢复到上一个CP点，但是无法索引到回写前fsync的数据，但是可以通过fsync标志进行前滚恢复。

前滚恢复的具体实现在recovery.c，核心函数是 `f2fs_recover_fsync_data` ，下面进行分析。

### Recovery的代码分析
F2FS启动的时候，会调用 `f2fs_recover_fsync_data` 函数开始恢复数据，代码如下:
```c
int f2fs_recover_fsync_data(struct f2fs_sb_info *sbi, bool check_only)
{
	...
	/**
	 * step #1: find fsynced inode numbers 
	 * inode_list保存了需要恢复的inode entry
	 * */
	err = find_fsync_dnodes(sbi, &inode_list, check_only); // 找到上次被fsynced的inode，保存在inode list中
	if (err || list_empty(&inode_list)) // 如果找不到可以recover的dnode，则跳到skip
		goto skip;

	if (check_only) {
		ret = 1;
		goto skip;
	}

	need_writecp = true;

	/* step #2: recover data */
	err = recover_data(sbi, &inode_list, &dir_list);
	if (!err)
		f2fs_bug_on(sbi, !list_empty(&inode_list));
skip:
	destroy_fsync_dnodes(&inode_list); // 删除保存在inode list保存的dnode

	/* truncate meta pages to be used by the recovery */
	truncate_inode_pages_range(META_MAPPING(sbi),
			(loff_t)MAIN_BLKADDR(sbi) << PAGE_SHIFT, -1); // 删除缓存

	if (err) {
		truncate_inode_pages_final(NODE_MAPPING(sbi));
		truncate_inode_pages_final(META_MAPPING(sbi));
	}

	clear_sbi_flag(sbi, SBI_POR_DOING); // 清除标记
	mutex_unlock(&sbi->cp_mutex);

	/* let's drop all the directory inodes for clean checkpoint */
	destroy_fsync_dnodes(&dir_list);

	if (!err && need_writecp) {
		struct cp_control cpc = {
			.reason = CP_RECOVERY,
		};
		err = f2fs_write_checkpoint(sbi, &cpc); // write checkpoint保存recover完的信息
	}

	kmem_cache_destroy(fsync_entry_slab);
out:
#ifdef CONFIG_QUOTA
	/* Turn quotas off */
	if (quota_enabled)
		f2fs_quota_off_umount(sbi->sb);
#endif
	sbi->sb->s_flags = s_flags; /* Restore SB_RDONLY status */

	return ret ? ret: err;
}
```


```c
static int find_fsync_dnodes(struct f2fs_sb_info *sbi, struct list_head *head,
				bool check_only)
{
	struct curseg_info *curseg;
	struct page *page = NULL;
	block_t blkaddr;
	unsigned int loop_cnt = 0;
	unsigned int free_blocks = sbi->user_block_count -
					valid_user_blocks(sbi);
	int err = 0;

	/* get node pages in the current segment */
	curseg = CURSEG_I(sbi, CURSEG_WARM_NODE); // 所有的恢复都是基于CP所管理的CURSEG进行恢复，因为它维护了宕机前一刻的系统使用状态

	/**
	 * NEXT_FREE_BLKADDR指的是宕机前的，可能已经分配使用的block地址，
	 * 但是由于宕机了，不得不回滚到上一个CP点，而没有被索引到这个已经写入数据的块，因此尝试恢复
	 */
	blkaddr = NEXT_FREE_BLKADDR(sbi, curseg);
	while (1) {
		struct fsync_inode_entry *entry;

		if (!f2fs_is_valid_meta_blkaddr(sbi, blkaddr, META_POR))
			return 0;

		page = f2fs_get_tmp_page(sbi, blkaddr); // 根据blkaddr读取该地址对应的node page的数据
		
		/**
		 * 根据上述的场景，如果突然宕机，
		 * node page已经写入了磁盘，但是恢复后的CP没有检测到，
		 * 因此比较node的ver和cp的ver是否一致，如果一致就可以恢复，否则只能舍弃掉
		 */
		if (!is_recoverable_dnode(page))
			break;

		/**
         * 具有检查是否进行FSYNC的标志
		 * 这个标志会在f2fs_sync_node_pages和f2fs_fsync_node_pages执行后进行标记
         */
		if (!is_fsync_dnode(page))
			goto next;

		/**
		 * 这里如果找到了entry则表示了已经对这个inode进行了恢复，避免多次恢复
		 */
		entry = get_fsync_inode(head, ino_of_node(page));
		if (!entry) { // 找不到表示还没有恢复这个inode
			bool quota_inode = false;

			/**
			 * 如果node page是，inode则执行恢复inode的函数
			 */
			if (!check_only &&
					IS_INODE(page) && is_dent_dnode(page)) { 
				err = f2fs_recover_inode_page(sbi, page);
				if (err)
					break;
				quota_inode = true;
			}

			/*
			 * CP | dnode(F) | inode(DF)
			 * For this case, we should not give up now.
			 * 
			 * 如果page不是inode，则表示该inode不需要继续恢复(已经写到dnode了，按顺序来说必须是先改inode再改dnode)
			 * 将已经恢复得inode加入到list中，避免多次恢复
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

		entry = get_fsync_inode(inode_list, ino_of_node(page));
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
		err = do_recover_data(sbi, entry->inode, page);
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
