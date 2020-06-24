# F2FS的GC流程
垃圾回收(Garbage Collection)在F2FS中，主要作用是回收无效的block，以供用户重新使用。在详细介绍GC之前，需要先分析一下为什么需要GC。

## Log-structured文件系统的特性
F2FS是一个基于Log-structured的文件系统，而垃圾回收则是Log-structured文件系统一个非常重要的特征，因为其独特的数据分配方式:
1. 这种类型的文件系统运行时会一直维护一个segment manager的元数据结构。
2. 用户使用在写入流程中，分配的每一个block都是从一个segment manager中取出来，并根据block的地址更新segment manager对应位置的数据，标记该block为已使用。
3. 当用户需要进行更新文件数据时，文件系统会通过异地更新的方式进行数据更新，即文件系统将用户更新后的数据写入一个新的block中，并且将旧的block无效掉(invalid)。

## F2FS GC的简介
基于Log-structured文件系统的特征，GC的主要作用是回收这些invalid的block，以供文件系统继续使用。F2FS的GC分为前台GC和后台GC: 前台GC一般在系统空间紧张的情况下运行，目的是尽快回收空间; 而后台GC则是在系统空闲的情况下进行，目的是在不影响用户体验的情况回收一定的空间。前台GC一般情况下是在checkpoint或者写流程的时候触发，因为F2FS能够感知空间的使用率，如果空间不够了会常触发前台GC加快回收空间，这意味着文件系统空间不足的时候，性能可能会下降。后台GC则是被一个线程间隔一段时间进行触发。而接下来我们主要讨论的都是后台GC。

## GC线程创建以及GC的触发时机
GC的启动函数是`start_gc_thread`，它在f2fs进行挂载的时候执行，作用是创建一个gc线程。
```c
int start_gc_thread(struct f2fs_sb_info *sbi)
{
	struct f2fs_gc_kthread *gc_th;
	dev_t dev = sbi->sb->s_bdev->bd_dev;
	int err = 0;

	// 分配gc线程所需要的内存空间
	gc_th = kmalloc(sizeof(struct f2fs_gc_kthread), GFP_KERNEL);
	if (!gc_th) {
		err = -ENOMEM;
		goto out;
	}

	// 设置最小后台gc触发间隔，DEF_GC_THREAD_MIN_SLEEP_TIME=30秒
	gc_th->min_sleep_time = DEF_GC_THREAD_MIN_SLEEP_TIME;
	// 设置最大后台gc触发间隔，DEF_GC_THREAD_MAX_SLEEP_TIME=60秒
	gc_th->max_sleep_time = DEF_GC_THREAD_MAX_SLEEP_TIME;
	// 设置没有gc的间隔，DEF_GC_THREAD_NOGC_SLEEP_TIME=300秒
	gc_th->no_gc_sleep_time = DEF_GC_THREAD_NOGC_SLEEP_TIME;

	// 判断系统是否为空间状态(idle)
	gc_th->gc_idle = 0;

	// 启动线程
	sbi->gc_thread = gc_th;
	init_waitqueue_head(&sbi->gc_thread->gc_wait_queue_head);
	sbi->gc_thread->f2fs_gc_task = kthread_run(gc_thread_func, sbi,
			"f2fs_gc-%u:%u", MAJOR(dev), MINOR(dev));
	if (IS_ERR(gc_th->f2fs_gc_task)) {
		err = PTR_ERR(gc_th->f2fs_gc_task);
		kfree(gc_th);
		sbi->gc_thread = NULL;
	}
out:
	return err;
}
```
从上面分析可以知道，gc的触发间隔会根据实际情况进行变化，下面根据gc线程的关于时间的变化的代码，分析是如何进行间隔变化的:
```c
static inline void increase_sleep_time(struct f2fs_gc_kthread *gc_th,
								long *wait)
{
	if (*wait == gc_th->no_gc_sleep_time)
		return;

	*wait += gc_th->min_sleep_time;
	if (*wait > gc_th->max_sleep_time)
		*wait = gc_th->max_sleep_time;
}

static inline void decrease_sleep_time(struct f2fs_gc_kthread *gc_th,
								long *wait)
{
	if (*wait == gc_th->no_gc_sleep_time)
		*wait = gc_th->max_sleep_time;

	*wait -= gc_th->min_sleep_time;
	if (*wait <= gc_th->min_sleep_time)
		*wait = gc_th->min_sleep_time;
}
```
`increase_sleep_time`以及`decrease_sleep_time`是调整gc间隔的函数，其中入参`long *wait`即为gc的间隔时间，而指针类型的原因是等待时间可以在该函数进行变化。

对于`increase_sleep_time`函数而言，如果目前的等待时间等于`no_gc_sleep_time`，则不做变化，表示系统处于不需要频繁做后台GC的情况，继续维持这种状态。如果不是，则增加30秒的gc间隔时间。

对于`decrease_sleep_time`函数而言，如果目前的等待时间等于`no_gc_sleep_time`，则不做变化，表示系统处于不需要频繁做后台GC的情况，继续维持这种状态。如果不是，则减少30秒的gc间隔时间。

这里有一个疑问，如果两个函数的末尾都增加限制范围的判断，限制了gc的间隔时间在30秒~60秒之间，为什么gc间隔可以增加到300秒以满足第一个if的条件呢? 解答这个问题，我们需要分析gc线程的主函数`gc_thread_func`，如下所示，我们可以知道关键位置是`f2fs_gc`的if判断条件。当f2fs_gc返回值不为0的时候，表示系统无法找到可以找到足够的invalid的block，因此间隔一段较长的时间，积累多一点invalid block再进行gc。

```c
static int gc_thread_func(void *data)
{
	struct f2fs_sb_info *sbi = data;
	struct f2fs_gc_kthread *gc_th = sbi->gc_thread;
	wait_queue_head_t *wq = &sbi->gc_thread->gc_wait_queue_head;
	long wait_ms;

	wait_ms = gc_th->min_sleep_time; // wait_ms初始化为最小的gc间隔，即30秒

	do {

		if (try_to_freeze()) // 如果线程被挂起了，则continue
			continue;
		else
			wait_event_interruptible_timeout(*wq,
						kthread_should_stop(),
						msecs_to_jiffies(wait_ms)); // 阻塞while循环wait_ms毫秒

		...
		
		if (sbi->sb->s_writers.frozen >= SB_FREEZE_WRITE) { // 如果f2fs冻结了写操作，则表示没有新分配block，因此增加gc间隔，不做gc
			increase_sleep_time(gc_th, &wait_ms);
			continue;
		}

		if (!is_idle(sbi)) { // 如果系统处于不是idle的状态，则表示系统忙，因此为了不影响用户体验，增加间隔的同时也不进行gc
			increase_sleep_time(gc_th, &wait_ms);
			continue;
		}

		if (has_enough_invalid_blocks(sbi)) // 如果系统空间有很多无效(invalid)的block，则减少触发间隔，增加gc的次数
			decrease_sleep_time(gc_th, &wait_ms);
		else
			increase_sleep_time(gc_th, &wait_ms); // 反之则减少gc的次数
		/**
		 * 这里解答了上面的疑问，什么时候会间隔300秒后再gc
		 * 因为当f2fs_gc返回值不为0的时候，表示系统无法找到可以找到足够的invalid的block，因此间隔一段较长的时间，
		 * 积累多一点invalid block再进行gc。
		 **/
		if (f2fs_gc(sbi))
			wait_ms = gc_th->no_gc_sleep_time; 

		...

	} while (!kthread_should_stop());
	return 0;
}
```

## GC的主流程
在分析如何回收之前，我们需要知道gc是以什么单位进行回收的。从第一章的f2fs layout的分析可以知道，f2fs的最小单位是block，往上的是segment，再往上是section，最高是zone。gc的回收单位是section，在默认情况下，一个section等于一个segment，因此每回收一个section就回收了512个block。

从上面的gc线程主函数`gc_thread_func`可以知道，gc的核心是`f2fs_gc`函数的执行，代码如下。


变量初始化中需要特别关注的是`struct gc_inode_list gc_list `，它的作用是将被gc的section所影响到的inode加入到这个list里面。因为一个section里面并不是所有的block都是invalid的，f2fs也只是会挑选相对比较多的invalid block的section进行gc。因此有一些valid的block需要进行迁移时会影响到它的inode，因此需要将这些inode串起来，集中处理。

需要关注的是函数是`__get_victim`函数以及`do_garbage_collect`函数，其中`__get_victim`是根据invalid block的数目以及其它因素等挑选出最适合进行gc的segment，然后传入到`do_garbage_collect`进行gc。`__get_victim`我们用单独一个小节[如何选择victim segment](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS-GC/%E9%80%89%E6%8B%A9victim%20segment.md)进行描述，这里先不做分析，只需要理解为挑选出一个合适的segment进行gc。



```c
int f2fs_gc(struct f2fs_sb_info *sbi)
{
	unsigned int segno, i;
	int gc_type = BG_GC;
	int nfree = 0;
	int ret = -1;
	struct cp_control cpc;
	struct gc_inode_list gc_list = {
		.ilist = LIST_HEAD_INIT(gc_list.ilist),
		.iroot = RADIX_TREE_INIT(GFP_NOFS),
	};

gc_more:


	if (!__get_victim(sbi, &segno, gc_type)) // 挑选出合适的segment，通过segno段号的方式返回
		goto stop; // 注意ret现在的值是-1，因此如果f2fs_gc返回不是0的结果，意味着找不出适合的victim segment，对应了上面300秒等待时间的分析
	ret = 0;

	for (i = 0; i < sbi->segs_per_sec; i++) // 将整个section里面的segment进行gc，其实1 sec = 1 seg，只会执行一次
		do_garbage_collect(sbi, segno + i, &gc_list, gc_type); // 进行gc的主要操作


	if (has_not_enough_free_secs(sbi, nfree))
		goto gc_more;

stop:
	put_gc_inode(&gc_list);
	return ret;
}
```
`do_garbage_collect`函数是gc流程的主要操作，作用是根据入参的段号segno找到对应的segment，然后将整个segment读取出来，通过异地更新的方式写入迁移到其他segment中。这样操作以后，被gc的segment会变为一个全新的segment进而可以被系统重新使用，如代码所示:

上面提及到f2fs需要使用一个inode list去记录被影响到的inode，那么是如何根据物理地址找到对应的inode呢? 答案是通过`f2fs_summary_block`结构。每一个segment都对应一个`f2fs_summary_block`结构。segment中每一个block都对应了`f2fs_summary_block`结构中的一个entry，记录了这个block地址属于哪个node(通过node id)以及属于这个node的第几个block，更详细的描述参看[f2fs_summary和f2fs_summary_block的介绍和应用](https://github.com/RiweiPan/F2FS-NOTES/blob/master/ImportantDataStructure/f2fs_summary.md)。

因此，`do_garbage_collect`函数的第一步是根据segno找到对应的`f2fs_summary_block`结构。第二步则是根据gc的数据类型选择`gc_node_segment`函数或者`gc_data_segment`函数实现数据的迁移。

```c
static void do_garbage_collect(struct f2fs_sb_info *sbi, unsigned int segno,
				struct gc_inode_list *gc_list, int gc_type)
{
	struct page *sum_page;
	struct f2fs_summary_block *sum;
	struct blk_plug plug;

	sum_page = get_sum_page(sbi, segno); // 根据segno获取f2fs_summary_block

	blk_start_plug(&plug);

	sum = page_address(sum_page);

	switch (GET_SUM_TYPE((&sum->footer))) { // 根据类型迁移数据
	case SUM_TYPE_NODE:
		gc_node_segment(sbi, sum->entries, segno, gc_type);
		break;
	case SUM_TYPE_DATA:
		gc_data_segment(sbi, sum->entries, gc_list, segno, gc_type);
		break;
	}
	blk_finish_plug(&plug);

	stat_inc_seg_count(sbi, GET_SUM_TYPE((&sum->footer)));
	stat_inc_call_count(sbi->stat_info);

	f2fs_put_page(sum_page, 1);
}
```
`gc_node_segment`函数或者`gc_data_segment`函数的数据迁移步骤大同小异，因此这里只分析`gc_data_segment`函数。

函数的第一步是根据segno获取这个segment里面的第一个block的地址，保存在`start_addr`中。然后从这个地址开始，便利这个segment所有的block。`entry = sum`同理，表示第一个block对应的第一个entry。

核心循环由于`next_step`的跳转，执行了4次，每一次循环都对应了一个阶段，分别是phase=0~3，通过continue关键词使每一次循环只执行一部分操作。我们每一阶段进行分析。

**第一阶段(phase=0):** 根据`entry`记录的nid，通过`ra_node_page`函数可以将这个nid对应的node page读入到内存当中。

**第二阶段(phase=1):** 根据`start_addr`以及`entry`，通过`check_dnode`函数，找到了对应的`struct node_info *dni`，它记录这个block是属于哪一个inode(inode no)，然后将对应的inode page读入到内存当中。

**第三阶段(phase=2):** 首先通过`entry->ofs_in_node`获取到当前block属于node的第几个block，然后通过`start_bidx_of_node`函数获取到当前block是属于从`inode page`开始的第几个block，其实本质上就是`start_bidx + ofs_in_node = page->index`的值。然后根据`page->index`找到对应的`data page`，读入到内存中以便后续使用。最后就是将该inode加入到上面提及过的inode list中。

**第三阶段(phase=3):** 从inode list中取出一个inode，然后根据`start_bidx + ofs_in_node`找到对应的`page->index`，然后通过`move_data_page`函数，将数据写入到其他segment中。

需要注意的是，上述很多的操作都是为了将数据读入内存中，这样系统可以快速地进行下一个步骤。

经过上述四个步骤，一个segment的所有数据被迁移了出去，系统就可以将segment回收(即从segment manager中设置这个segment对应的标志位为全新可用状态)。


```c
static void gc_data_segment(struct f2fs_sb_info *sbi, struct f2fs_summary *sum,
		struct gc_inode_list *gc_list, unsigned int segno, int gc_type)
{
	struct super_block *sb = sbi->sb;
	struct f2fs_summary *entry;
	block_t start_addr;
	int off;
	int phase = 0;

	start_addr = START_BLOCK(sbi, segno);

next_step:
	entry = sum;

	for (off = 0; off < sbi->blocks_per_seg; off++, entry++) {
		struct page *data_page;
		struct inode *inode;
		struct node_info dni; /* dnode info for the data */
		unsigned int ofs_in_node, nofs;
		block_t start_bidx;

		if (phase == 0) {
			ra_node_page(sbi, le32_to_cpu(entry->nid)); // 预读node page
			continue;
		}

		if (check_dnode(sbi, entry, &dni, start_addr + off, &nofs) == 0) // 找到ino
			continue;

		if (phase == 1) {
			ra_node_page(sbi, dni.ino); // 预读inode page
			continue;
		}

		ofs_in_node = le16_to_cpu(entry->ofs_in_node);

		if (phase == 2) {
			inode = f2fs_iget(sb, dni.ino);

			start_bidx = start_bidx_of_node(nofs, F2FS_I(inode));

			data_page = find_data_page(inode,
					start_bidx + ofs_in_node, false); // 预读data page

			f2fs_put_page(data_page, 0);
			add_gc_inode(gc_list, inode); // 加入到inode list
			continue;
		}

		/* phase 3 */
		inode = find_gc_inode(gc_list, dni.ino);
		if (inode) {
			start_bidx = start_bidx_of_node(nofs, F2FS_I(inode));
			data_page = get_lock_data_page(inode,
						start_bidx + ofs_in_node);
			move_data_page(inode, data_page, gc_type); // 迁移数据
		}
	}

	if (++phase < 4)
		goto next_step;
}
```











