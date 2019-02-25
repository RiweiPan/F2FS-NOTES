## 前滚恢复和Recovery的作用与实现
后滚恢复的主要作用是恢复已经写入到磁盘，但是上一次Checkpoint没有记录的数据

### Recovery的代码分析
F2FS启动的时候，会调用 `f2fs_recover_fsync_data` 函数开始恢复数据，代码如下:
```c
int f2fs_recover_fsync_data(struct f2fs_sb_info *sbi, bool check_only)
{
	struct list_head inode_list;
	struct list_head dir_list;
	int err;
	int ret = 0;
	unsigned long s_flags = sbi->sb->s_flags;
	bool need_writecp = false;
#ifdef CONFIG_QUOTA
	int quota_enabled;
#endif

	if (s_flags & SB_RDONLY) {
		f2fs_msg(sbi->sb, KERN_INFO, "orphan cleanup on readonly fs");
		sbi->sb->s_flags &= ~SB_RDONLY;
	}

#ifdef CONFIG_QUOTA
	/* Needed for iput() to work correctly and not trash data */
	sbi->sb->s_flags |= SB_ACTIVE;
	/* Turn on quotas so that they are updated correctly */
	quota_enabled = f2fs_enable_quota_files(sbi, s_flags & SB_RDONLY);
#endif

	fsync_entry_slab = f2fs_kmem_cache_create("f2fs_fsync_inode_entry",
			sizeof(struct fsync_inode_entry));
	if (!fsync_entry_slab) {
		err = -ENOMEM;
		goto out;
	}

	INIT_LIST_HEAD(&inode_list);
	INIT_LIST_HEAD(&dir_list);

	/* prevent checkpoint */
	mutex_lock(&sbi->cp_mutex);

	/* step #1: find fsynced inode numbers */
	err = find_fsync_dnodes(sbi, &inode_list, check_only);
	if (err || list_empty(&inode_list))
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
	destroy_fsync_dnodes(&inode_list);

	/* truncate meta pages to be used by the recovery */
	truncate_inode_pages_range(META_MAPPING(sbi),
			(loff_t)MAIN_BLKADDR(sbi) << PAGE_SHIFT, -1);

	if (err) {
		truncate_inode_pages_final(NODE_MAPPING(sbi));
		truncate_inode_pages_final(META_MAPPING(sbi));
	}

	clear_sbi_flag(sbi, SBI_POR_DOING);
	mutex_unlock(&sbi->cp_mutex);

	/* let's drop all the directory inodes for clean checkpoint */
	destroy_fsync_dnodes(&dir_list);

	if (!err && need_writecp) {
		struct cp_control cpc = {
			.reason = CP_RECOVERY,
		};
		err = f2fs_write_checkpoint(sbi, &cpc);
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
	curseg = CURSEG_I(sbi, CURSEG_WARM_NODE);
	/**
	 * 考虑一个场景: 数据进行写入，数据已经落入了磁盘，但是在元数据更新前出现了宕机
	 * NEXT_FREE_BLKADDR指的是宕机前的，正在使用的block，但是由于宕机了，不得不回滚到上一个CP点
	 * 因此这里可以对已经写入了磁盘的数据，但是丢失的元数据(因此是CURSEG_WARM_NODE)进行恢复
	 */
	blkaddr = NEXT_FREE_BLKADDR(sbi, curseg);
	while (1) {
		struct fsync_inode_entry *entry;

		if (!f2fs_is_valid_meta_blkaddr(sbi, blkaddr, META_POR))
			return 0;

		page = f2fs_get_tmp_page(sbi, blkaddr);

		if (!is_recoverable_dnode(page)) // 比较node_page和cp的ver是否一样，如果一样则继续，不一样则挑出(跳出后会发生什么？)
			break;

		if (!is_fsync_dnode(page)) // 具有检查是否进行FSYNC的标志，这个标志会在f2fs_sync_node_pages和f2fs_fsync_node_pages执行后进行标记
			goto next;

		entry = get_fsync_inode(head, ino_of_node(page));
		if (!entry) {
			bool quota_inode = false;

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
			entry->last_dentry = blkaddr;
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

		/* check next segment */
		blkaddr = next_blkaddr_of_node(page);
		f2fs_put_page(page, 1);

		f2fs_ra_meta_pages_cond(sbi, blkaddr);
	}
	f2fs_put_page(page, 1);
	return err;
}
```
