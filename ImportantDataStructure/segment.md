## seg_entry 和 sit_info

### 介绍
因为每一个segment需要管理512个Block的地址，因此需要通过某种方式去标记一个segment下的block，哪些是已经使用的，哪些block是处于无效状态等待回收。在F2FS中，通过结构体 `seg_entry` 去管理一个segment下的所有block的使用信息:

```c
struct seg_entry {
	unsigned int type:6;		/* 这个segment的类型 */
	unsigned int valid_blocks:10;	/* 已经使用的块的数目 */
	unsigned int ckpt_valid_blocks:10;	/* 上一次执行CP时，使用的块的数目 */
	unsigned int padding:6;		/* padding */
	unsigned char *cur_valid_map;	/* 通过bitmap(512位)表示这个segment哪些被使用，哪些没使用 */
#ifdef CONFIG_F2FS_CHECK_FS
	unsigned char *cur_valid_map_mir;	/* mirror of current valid bitmap */
#endif
	/*
	 * # of valid blocks and the validity bitmap stored in the the last
	 * checkpoint pack. This information is used by the SSR mode.
	 */
	unsigned char *ckpt_valid_map;	/* 上次CP时的bitmap状态 */
	unsigned char *discard_map; /* 标记哪些block需要discard的bitmap */
	unsigned long long mtime;	/* 修改时间 */
};
```

`seg_entry` 由于跟磁盘空间大小有关，因此初始化时以动态分配的方式，保存在元数据区域的**SIT**区域当中，代码的具体实现为`sbi->sit_info->sentries` 中。

### 应用场景
1. **写流程:** 当文件的修改某一个block的数据时，需要经过的流程是：**1) 分配一个新的block; 2) 将数据写入到新分配的block中; 3) 将旧block置为无效，等待回收; 4) 将新block写入到磁盘中。**这一个流程需要更新的segment的管理信息，因为新block和旧block可能来自不同的segment，因此需要更新segment的统计信息，具体流程是: 根据新block的地址，找到对应segment number和seg_entry，然后在 `seg_entry `的根据新block在segment的bitmap对应位置设为1，然后给 `seg_entry->valid_blocks` 加一，表示这个segment新增加了一个被使用block;对于旧block，一样是根据block地址找到segment number和seg_entry，然后执行相反操作对bitmap设为0，然后 `seg_entry->valid_blocks` 减一。