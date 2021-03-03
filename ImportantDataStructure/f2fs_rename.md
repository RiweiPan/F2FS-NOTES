# F2FS的rename流程

## rename流程介绍

1. sys_rename函数
2. do_renameat2函数
3. vfs_rename函数
4. f2fs_rename函数



## sys_rename函数

sys_rename函数是一个系统调用，是rename函数进入内核层的第一个函数:

```c
SYSCALL_DEFINE2(rename, const char __user *, oldname, const char __user *, newname)
{
    // AT_FDCWD表示以相对路径的方法找oldname和newname这个文件，flags=0
	return do_renameat2(AT_FDCWD, oldname, AT_FDCWD, newname, 0);
}
```

## do_renameat2函数

do_renameat2函数比较长，考虑多个输入flag的作用，这里只考虑sys_rename函数rename一个文件的情形，即flag=0，并以此精简函数。

```c
static int do_renameat2(int olddfd, const char __user *oldname, int newdfd,
			const char __user *newname, unsigned int flags)
{
	struct dentry *old_dentry, *new_dentry;
	struct dentry *trap;
	struct path old_path, new_path;
	struct qstr old_last, new_last;
	int old_type, new_type;
	struct inode *delegated_inode = NULL;
	struct filename *from;
	struct filename *to;
	unsigned int lookup_flags = 0, target_flags = LOOKUP_RENAME_TARGET;
	bool should_retry = false;
	int error;

retry:
    // 接下来两个函数最重要的作用是根据oldname和newname找到父目录的dentry结构
    // 这两个dentry结构保存在old_path和new_path中(注意是父目录的dentry)
	from = filename_parentat(olddfd, getname(oldname), lookup_flags,
				&old_path, &old_last, &old_type);

	to = filename_parentat(newdfd, getname(newname), lookup_flags,
				&new_path, &new_last, &new_type);
    
retry_deleg:
    // 这个函数会触发一个全局的rename的互斥锁，然后锁两个父目录inode结构
	trap = lock_rename(new_path.dentry, old_path.dentry);
	// 根据old path的父目录找到需要被rename的文件的dentry
	old_dentry = __lookup_hash(&old_last, old_path.dentry, lookup_flags);
	// 根据new path的父目录找到或创建新的dentry
	new_dentry = __lookup_hash(&new_last, new_path.dentry, lookup_flags | target_flags);
	// 调用vfs_rename函数进行重命名
    // 传入的是新旧两个目录的inode，以及需要重命名的两个dentry， flags = 0
	error = vfs_rename(old_path.dentry->d_inode, old_dentry,
			   new_path.dentry->d_inode, new_dentry,
			   &delegated_inode, flags);

	dput(new_dentry);

	dput(old_dentry);
    // 解锁全局rename互斥锁，释放两个inode锁
	unlock_rename(new_path.dentry, old_path.dentry);

	path_put(&new_path);
	putname(to);

	path_put(&old_path);
	putname(from);
exit:
	return error;
}
```

## vfs_rename函数

vfs_rename函数也会做简化，简化的情形是将文件A重命名到文件B (B可能已经存在，或者不存在)，flags=0。

```c
int vfs_rename(struct inode *old_dir, struct dentry *old_dentry,
	       struct inode *new_dir, struct dentry *new_dentry,
	       struct inode **delegated_inode, unsigned int flags)
{
	int error;
	bool is_dir = d_is_dir(old_dentry);
	struct inode *source = old_dentry->d_inode; // 旧文件inode
	struct inode *target = new_dentry->d_inode; // 新文件inode
	bool new_is_dir = false;
	unsigned max_links = new_dir->i_sb->s_max_links;
	struct name_snapshot old_name;

	dget(new_dentry); // 对新文件的引用计数+1
	if (target)
		inode_lock(target); // 如果新文件已经存在，则上锁


	error = old_dir->i_op->rename(old_dir, old_dentry,
				       new_dir, new_dentry, flags);


out:
	if (target)
		inode_unlock(target); // 如果新文件已经存在，则解锁
	dput(new_dentry); // 对新文件的引用计数-1
	return error;
}
```

## f2fs_rename函数

f2fs_rename函数也会做简化，简化的情形是将文件A重命名到文件B (B可能已经存在，或者不存在)，flags=0。

```c
static int f2fs_rename(struct inode *old_dir, struct dentry *old_dentry,
			struct inode *new_dir, struct dentry *new_dentry,
			unsigned int flags)
{
	struct f2fs_sb_info *sbi = F2FS_I_SB(old_dir);
	struct inode *old_inode = d_inode(old_dentry);
	struct inode *new_inode = d_inode(new_dentry);
	struct inode *whiteout = NULL;
	struct page *old_dir_page;
	struct page *old_page, *new_page = NULL;
	struct f2fs_dir_entry *old_dir_entry = NULL;
	struct f2fs_dir_entry *old_entry;
	struct f2fs_dir_entry *new_entry;
	bool is_old_inline = f2fs_has_inline_dentry(old_dir);
	int err;
	
	// 输入显然是
    // 旧的父目录old_dir，旧的文件old_dentry
    // 新的父目录new_dir，新的文件new_dentry
    
    // 根据旧文件的名字找到对应的f2fs_dir_entry，old_page保存的是磁盘上的dir_entry数据
	old_entry = f2fs_find_entry(old_dir, &old_dentry->d_name, &old_page);

	if (new_inode) { // 如果新文件已经存在

        // 根据新文件的名字找到对应的f2fs_dir_entry，new_page保存的是磁盘上的数据
		new_entry = f2fs_find_entry(new_dir, &new_dentry->d_name,
						&new_page);

        // F2FS获取一个全局读信号量
		f2fs_lock_op(sbi);

        // 在管理orphan inode的全局结构中，将orphan inode的数目+1。
		err = f2fs_acquire_orphan_inode(sbi);

        // 这里进行新旧inode的link的变化:
        // 将new_dentry所属的inode指向old_inode
        // 因为rename的时候新inode是已经存在了，因此rename的操作就是将
        // 新路径原来的inode无效掉，然后替换为旧路径的inode
		f2fs_set_link(new_dir, new_entry, new_page, old_inode);

		new_inode->i_ctime = current_time(new_inode);
        
        
		down_write(&F2FS_I(new_inode)->i_sem); // 拿写信号量
		// 减少新inode一个引用计数，因为被rename了
		f2fs_i_links_write(new_inode, false);
		up_write(&F2FS_I(new_inode)->i_sem); // 释放写信号量

        // 如果引用计数下降到0，则添加到orphan inode中，在checkpoint管理
		if (!new_inode->i_nlink)
			f2fs_add_orphan_inode(new_inode);
		else
			f2fs_release_orphan_inode(sbi); // 否则管理结构将orphan inode的数目-1。
	} else {
        
        // 这个情况是新路径的Inode不存在

		// F2FS获取一个全局读信号量
		f2fs_lock_op(sbi);

        // 由于新inode是不存在的，因此直接将旧inode添加到新的f2fs_dir_entry中
		err = f2fs_add_link(new_dentry, old_inode);

	}

    
	down_write(&F2FS_I(old_inode)->i_sem);
	if (!old_dir_entry || whiteout)
		file_lost_pino(old_inode);  // 这个操作要保留着用于数据恢复
	else
		F2FS_I(old_inode)->i_pino = new_dir->i_ino;
	up_write(&F2FS_I(old_inode)->i_sem);

	old_inode->i_ctime = current_time(old_inode);
	f2fs_mark_inode_dirty_sync(old_inode, false);

    // 新的数据已经加入到新的f2fs_dir_entry，因此旧entry就去去除掉
	f2fs_delete_entry(old_entry, old_page, old_dir, NULL);

	// F2FS释放全局读信号量
	f2fs_unlock_op(sbi);

	f2fs_update_time(sbi, REQ_TIME);
	return 0;
}
```

