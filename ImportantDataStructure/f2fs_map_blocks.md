## f2fs_map_blocks的作用与源码分析
函数 `f2fs_map_blocks` 启到了地址映射的作用，主要作用是通过逻辑地址找到可以访问磁盘的物理地址。

### f2fs_map_blocks的读写流程的作用
1. 对读的作用: 通过该函数根据逻辑地址找到物理地址，然后从磁盘读取出数据。
2. 对写的作用: 文件在写入数据之前，会执行一个preallocate的过程，这个过程会调用 `f2fs_map_blocks` 函数对即将要写入数据的逻辑块进行预处理，如果是append的方式写入数据，则将物理地址初始化为NEW_ADDR; 如果是rewrite的方式写入数据，则不作改变。

### f2fs_map_blocks的核心数据结构
`f2fs_map_blocks` 函数的核心是 `f2fs_map_blocks` 数据结构，如下所示，它保存了一系列映射信息。
```c
struct f2fs_map_blocks {
	block_t m_pblk; // 保存的是物理地址，可以通过这个物理地址访问磁盘读取信息
	block_t m_lblk; // 保存的逻辑地址，即文件的page->index
	unsigned int m_len; // 需要读取的长度
	unsigned int m_flags; // flags表示获取数据状态，如F2FS_MAP_MAPPED
	pgoff_t *m_next_pgofs; // 指向下一个offset
};
```

### 读流程下的f2fs_map_blocks的核心逻辑

一般的读流程，会进行如下的数据结构初始化:
```c
map.m_lblk = block_in_file; // 设置逻辑地址page->index
map.m_len = len; // 设置需要读取的长度
f2fs_map_blocks(inode, &map, 0, F2FS_GET_BLOCK_READ); // 0设定非创建模式，F2FS_GET_BLOCK_READ设定搜索模式
```
即通过逻辑地址和读取长度找到对应的物理地址，与**读流程相关的核心逻辑**如下面代码所示:
```c
int f2fs_map_blocks(struct inode *inode, struct f2fs_map_blocks *map, int create, int flag)
{
	unsigned int maxblocks = map->m_len; // 设定最大搜索长度
	int mode = create ? ALLOC_NODE : LOOKUP_NODE_RA; // LOOKUP_NODE_RA模式

	map->m_len = 0; // 将len重新设置为0
	map->m_flags = 0;
	pgofs =	(pgoff_t)map->m_lblk; // page->index

	// 第一步：先从extent找，如果在extent找到，就可以马上返回
	if (!create && f2fs_lookup_extent_cache(inode, pgofs, &ei)) {
		map->m_pblk = ei.blk + pgofs - ei.fofs;
		map->m_len = min((pgoff_t)maxblocks, ei.fofs + ei.len - pgofs);
		map->m_flags = F2FS_MAP_MAPPED;
		goto out;
	}

	// 第二步：根据page->index找到对应的dn，dn是一个包含了物理地址的数据结构
	set_new_dnode(&dn, inode, NULL, NULL, 0);
	err = get_dnode_of_data(&dn, pgofs, mode);

	// 第三步：从dn获取物理地址
	blkaddr = datablock_addr(dn.node_page, dn.ofs_in_node);
	...
	map->m_pblk = blkaddr;
	...
	return err;
}
```

### 写流程下的f2fs_map_blocks的核心逻辑
一般的读流程，会进行如下的数据结构初始化:
```c
map.m_lblk = F2FS_BLK_ALIGN(iocb->ki_pos); // 计算得到页偏移
map.m_len = F2FS_BYTES_TO_BLK(iocb->ki_pos + iov_iter_count(from)); // 计算得到需要读取的页数
f2fs_map_blocks(inode, &map, 1, F2FS_GET_BLOCK_PRE_AIO); // 1设定创建模式，F2FS_GET_BLOCK_PRE_AIO表示用于预分配物理页
```
写流程下的 `f2fs_map_blocks` 函数作用是先根据逻辑地址读取物理地址出来，如果这个物理地址没有被分配过(NULL_ADDR)，则初始化为新地址(NEW_ADDR)，用于下一步的写入磁盘的操作，与**写流程相关的核心逻辑**如下面代码所示:
```c

int f2fs_map_blocks(struct inode *inode, struct f2fs_map_blocks *map,
						int create, int flag)
{
	unsigned int maxblocks = map->m_len;
	int mode = create ? ALLOC_NODE : LOOKUP_NODE;

	map->m_len = 0;
	map->m_flags = 0;

	pgofs =	(pgoff_t)map->m_lblk;
	end = pgofs + maxblocks;

	// 第一步：根据page->index找到对应的dn，dn是一个包含了物理地址的数据结构
	set_new_dnode(&dn, inode, NULL, NULL, 0);
	err = f2fs_get_dnode_of_data(&dn, pgofs, mode);

	// 第二步：从dn获取物理地址
	blkaddr = datablock_addr(dn.inode, dn.node_page, dn.ofs_in_node);

	// 第三步：如果blkaddr == NULL_ADDR表示这个是从来未使用过的物理页，即目前运行的是append写，
	//  因此将其记录下来。
	if (flag == F2FS_GET_BLOCK_PRE_AIO) {
		if (blkaddr == NULL_ADDR) {
			prealloc++; // 记录需要与分配的物理页的数目
			last_ofs_in_node = dn.ofs_in_node;
		}
	} 

	if (flag == F2FS_GET_BLOCK_PRE_AIO &&
			(pgofs == end || dn.ofs_in_node == end_offset)) {

		dn.ofs_in_node = ofs_in_node;
		// 第四步：根据prealloc记录的从未被使用过的块的数目，
		//  通过函数f2fs_reserve_new_blocks，将他们的值由NULL_ADDR转换为NEW_ADDR，用于下一步写入磁盘
		err = f2fs_reserve_new_blocks(&dn, prealloc);
		if (err)
			goto sync_out;

		map->m_len += dn.ofs_in_node - ofs_in_node;
		if (prealloc && dn.ofs_in_node != last_ofs_in_node + 1) {
			err = -ENOSPC;
			goto sync_out;
		}
		dn.ofs_in_node = end_offset;
	}

	...
	return err;
}
```