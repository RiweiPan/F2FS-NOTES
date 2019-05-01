## F2FS的sync流程
应用程序很多时候会调用fsync同步文件，在f2fs中会调用`f2fs_do_sync_file`函数完成文件的同步过程，简化后的代码代码如下，fsync的过程涉及了很多inode的flag相关的信息，因此接下来分开几个部分分析：

```c
static int f2fs_do_sync_file(struct file *file, loff_t start, loff_t end,
						int datasync, bool atomic)
{
	enum cp_reason_type cp_reason = 0;
	struct writeback_control wbc = {
		.sync_mode = WB_SYNC_ALL,
		.nr_to_write = LONG_MAX,
		.for_reclaim = 0,
	};

	/* 1.   FI_NEED_IPU-fdatasyn以及少量文件改动使用就地更新 */
	if (datasync || get_dirty_pages(inode) <= SM_I(sbi)->min_fsync_blocks)
		set_inode_flag(inode, FI_NEED_IPU);
	ret = file_write_and_wait_range(file, start, end);
	clear_inode_flag(inode, FI_NEED_IPU);

	/* 2.   f2fs_skip_inode_update这个函数用于判断inode是否为脏，如果是脏，这个函数会返回false */
	if (!f2fs_skip_inode_update(inode, datasync)) {
		f2fs_write_inode(inode, NULL);
		goto go_write;
	}

	if (!is_inode_flag_set(inode, FI_APPEND_WRITE) &&
			!f2fs_exist_written_data(sbi, ino, APPEND_INO)) {

		/* it may call write_inode just prior to fsync */
		if (need_inode_page_update(sbi, ino))
			goto go_write;

		if (is_inode_flag_set(inode, FI_UPDATE_WRITE) ||
				f2fs_exist_written_data(sbi, ino, UPDATE_INO))
			goto flush_out;
		goto out;
	}
go_write:
	down_read(&F2FS_I(inode)->i_sem);
	cp_reason = need_do_checkpoint(inode);
	up_read(&F2FS_I(inode)->i_sem);

	if (cp_reason) {
		ret = f2fs_sync_fs(inode->i_sb, 1);
		try_to_fix_pino(inode);
		clear_inode_flag(inode, FI_APPEND_WRITE);
		clear_inode_flag(inode, FI_UPDATE_WRITE);
		goto out;
	}

sync_nodes:
	atomic_inc(&sbi->wb_sync_req[NODE]);
	ret = f2fs_fsync_node_pages(sbi, inode, &wbc, atomic);
	atomic_dec(&sbi->wb_sync_req[NODE]);
	if (ret)
		goto out;

	if (f2fs_need_inode_block_update(sbi, ino)) {
		f2fs_mark_inode_dirty_sync(inode, true);
		f2fs_write_inode(inode, NULL);
		goto sync_nodes;
	}

	if (!atomic) {
		ret = f2fs_wait_on_node_pages_writeback(sbi, ino);
		if (ret)
			goto out;
	}

	f2fs_remove_ino_entry(sbi, ino, APPEND_INO);
	clear_inode_flag(inode, FI_APPEND_WRITE);
flush_out:
	if (!atomic && F2FS_OPTION(sbi).fsync_mode != FSYNC_MODE_NOBARRIER)
		ret = f2fs_issue_flush(sbi, inode->i_ino);
	if (!ret) {
		f2fs_remove_ino_entry(sbi, ino, UPDATE_INO);
		clear_inode_flag(inode, FI_UPDATE_WRITE);
		f2fs_remove_ino_entry(sbi, ino, FLUSH_INO);
	}
	f2fs_update_time(sbi, REQ_TIME);
out:
	trace_f2fs_sync_file_exit(inode, cp_reason, datasync, ret);
	f2fs_trace_ios(NULL, 1);
	printk("[hubery] end f2fs_do_sync_file (%d,%d)thread=%s\n", current->pid, current->tgid, current->comm);
	return ret;
}
```

### FI_NEED_IPU-fdatasyn以及少量文件改动使用就地更新
当改动文件少量的数据时，为了减少IO量，使用了会使用就地更新的策略：在fsync函数中，给inode设置`FI_NEED_IPU`标记，再writepages函数中，检测到该标记会使用就地更新的策略。
```c
if (datasync || get_dirty_pages(inode) <= SM_I(sbi)->min_fsync_blocks)
	set_inode_flag(inode, FI_NEED_IPU); // 当dirty pages少于min_fsync_blocks时，设置标记
ret = file_write_and_wait_range(file, start, end); // 调用writepages刷写入磁盘
clear_inode_flag(inode, FI_NEED_IPU); // 再清除标记
```
writepages会调用`f2fs_do_write_data_page`进行写入，该函数有会调用`need_inplace_update`判断是否包含`FI_NEED_IPU`标记，如果包含则进行就地更新。
```c
int f2fs_do_write_data_page(struct f2fs_io_info *fio)
{
	...

	if (ipu_force || (is_valid_blkaddr(fio->old_blkaddr) && need_inplace_update(fio))) {
		...
		err = f2fs_inplace_write_data(fio); // 进行就地写入
		...
		return err;
	}
	...
}
```

### FI_AUTO_RECOVER-可以判断inode是否为脏
如果对一个不是没有`FI_DIRTY_INODE`标记的inode进行size等信息进行修改，会加上`FI_AUTO_RECOVER`等信息，当inode被设置`FI_DIRTY_INODE`标记的时候或者inode被fsync之后，就会被清除。
```c
static inline bool f2fs_skip_inode_update(struct inode *inode, int dsync)
{
	...
	if (!is_inode_flag_set(inode, FI_AUTO_RECOVER) ||
			file_keep_isize(inode) ||
			i_size_read(inode) & ~PAGE_MASK) // 这个if判断了没有设置FI_AUTO_RECOVER标记，则认为是脏的
		return false;
}

```


### FI_APPEND_WRITE & FI_UPDATE_WRITE-用于判断inode已经没有脏数据可以写
FI_APPEND_WRITE----->在异地更新的时候设置
FI_UPDATE_WRITE----->在就地更新的时候设置
这两个标记在fsync的时候的都会清除，因此fsync会使用这两个标记判断是否已经没有dirty pages
```c
if (!is_inode_flag_set(inode, FI_APPEND_WRITE) &&
		!f2fs_exist_written_data(sbi, ino, APPEND_INO)) { // 判断FI_APPEND_WRITE标记

	if (need_inode_page_update(sbi, ino))
		goto go_write;

	if (is_inode_flag_set(inode, FI_UPDATE_WRITE) ||
			f2fs_exist_written_data(sbi, ino, UPDATE_INO)) // 判断FI_UPDATE_WRITE标记，如果还是存在就要flush
		goto flush_out; // 如果
	goto out; // 如果都没有就直接退出fsync函数
}
```

### fsync的回写
如果无法在`f2fs_skip_inode_update`函数跳过，则要进行各个page的回写

第一步检查是否需要回写checkpoint，如果需要则写完checkpoint后退出fsync

```c
down_read(&F2FS_I(inode)->i_sem);
cp_reason = need_do_checkpoint(inode); 
up_read(&F2FS_I(inode)->i_sem);

if (cp_reason) {
	ret = f2fs_sync_fs(inode->i_sb, 1); // 这个函数用于调用write_checkpoint，回写一次checkpoint
	try_to_fix_pino(inode);
	clear_inode_flag(inode, FI_APPEND_WRITE);
	clear_inode_flag(inode, FI_UPDATE_WRITE);
	goto out;
}
```

如果不需要回写checkpoint，则回写所有dirty的node pages。
```c
atomic_inc(&sbi->wb_sync_req[NODE]);
ret = f2fs_fsync_node_pages(sbi, inode, &wbc, atomic); // 回写所有的dirty node pages
atomic_dec(&sbi->wb_sync_req[NODE]);

if (f2fs_need_inode_block_update(sbi, ino)) {
	f2fs_mark_inode_dirty_sync(inode, true);
	f2fs_write_inode(inode, NULL);
	goto sync_nodes;
}

if (!atomic) {
	ret = f2fs_wait_on_node_pages_writeback(sbi, ino);
	if (ret)
		goto out;
}

f2fs_remove_ino_entry(sbi, ino, APPEND_INO);
clear_inode_flag(inode, FI_APPEND_WRITE);
```

