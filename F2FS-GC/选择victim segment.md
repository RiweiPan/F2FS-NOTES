## Victim Segment的选择
#### 选择函数的初始化
F2FS将victim segment的选择函数设计为函数指针，用户可以根据自己的需求，初始化这个函数指针:
```c
struct victim_selection {
	/*
	 * result: 用于保存被选择的segment number，可以通过segno找到对应的segment
	 * gc_type: 前台GC或者后台GC
	 * type: 标记的是数据类型，也就是 NODE DATA HOT WARM COLD那些，针对获取不同的segment等
	 * alloc_mode: 分配模式SSR或者LFS
	 */
	int (*get_victim)(struct f2fs_sb_info *sbi, unsigned int *result, int gc_type, int type, char alloc_mode);
};
```
LFS模式和SSR模式是指block的分配模式，他们的区别在:
>LFS:顺序分配，即创建一个新block。
>SSR:重用被抛弃的block，直接在这个block里面更新。

在F2FS中，默认的get victim函数是:
```c
static const struct victim_selection default_v_ops = {
	.get_victim = get_victim_by_default,
};
```

#### get_victim_by_default的分析
`get_victim_by_default` 的核心数据结构是 `victim_sel_policy`，它的作用是保存中间变量，记录segno的变换等，它的结构是:
```c
struct victim_sel_policy {
	int alloc_mode;			/* LFS or SSR 分配模式 */
	int gc_mode;			/* GC_CB or GC_GREEDY GC的模式 */
	unsigned long *dirty_segmap;	/* 记录的是脏segment的bitmap */
	unsigned int max_search;	/* 最大搜索次数 */
	unsigned int offset;		/* 当前正在搜索的segno */
	unsigned int ofs_unit;		/* bitmap search unit */
	unsigned int min_cost;		/* minimum cost COST表示的是如果迁移这个segment，要迁移多少个非有效快 */
	unsigned int min_segno;		/* 记录拥有最少COST的那个segno */
};
```

接下来分析 `victim_sel_policy` 是如何初始化的，它的初始化 `get_victim_by_default` 函数的开始的十几行:
```c
static int get_victim_by_default(struct f2fs_sb_info *sbi,
		unsigned int *result, int gc_type, int type, char alloc_mode)
{
	struct dirty_seglist_info *dirty_i = DIRTY_I(sbi); // 从sbi获取记录segment的dirty信息的bitmap
	struct sit_info *sm = SIT_I(sbi);
	struct victim_sel_policy p; // 这个变量起到记录segno，和结束循环的作用
	unsigned int secno, last_victim;
	unsigned int last_segment = MAIN_SEGS(sbi); // 获取main area的第一个segment的地址，segno是根据第一个segment地址计算的
	unsigned int nsearched = 0;

	mutex_lock(&dirty_i->seglist_lock);

	p.alloc_mode = alloc_mode;
	select_policy(sbi, gc_type, type, &p); // 根据gc_type和alloc type初始化victim_sel_policy

	p.min_segno = NULL_SEGNO; // min_segno记录的是cost最小的segno，初始化NULL_SEGNO
	p.min_cost = get_max_cost(sbi, &p); // 初始化为迁移的最大开销，之后会在循环中慢慢降低，直到找到最小开销

	if (*result != NULL_SEGNO) { // 特殊情况就退出
		if (IS_DATASEG(get_seg_entry(sbi, *result)->type) &&
			get_valid_blocks(sbi, *result, false) &&
			!sec_usage_check(sbi, GET_SEC_FROM_SEG(sbi, *result)))
			p.min_segno = *result;
		goto out;
	}

	if (p.max_search == 0)
		goto out;

	last_victim = sm->last_victim[p.gc_mode]; // 获取上次被gc的victim的segno
	/**
	 * 如果是LFS模式+前台GC模式，就选择上次被bg gc的segment，
	 * 这样可以保证拥有最少的有效块，因此在迁移这些块的时候时间消耗比较少，能够较快获得空间，同时gc速度比较快
	 */
	if (p.alloc_mode == LFS && gc_type == FG_GC) {
		p.min_segno = check_bg_victims(sbi);
		if (p.min_segno != NULL_SEGNO)
			goto got_it;
	}

	...


}
```
获取victim segment的核心是如何找到一个合适的segment，F2FS使用cost表示迁移一个segment时的开销，也就是统计当前segment有多少个有效块(只有有效块才会被迁移)。

F2FS会选择一个cost最低的segment作为victim segment，目前有三种方式计算开销，分别是SSR模式，和LFS分配模式情况下的GREEDY模式和COST-BENEFIT模式。

SSR模式一般在空间使用过高的时候才会发生，因此一般情况是使用GREEDY模式和COST-BENEFIT模式选择victim。


挑选victim的时候，会经历一个cost从高到低的选择过程，因此第一步是将cost初始化为最大cost，以便从高变低。
```c
/*
 * 获得不同模式的最大损耗
 */
static unsigned int get_max_cost(struct f2fs_sb_info *sbi,
				struct victim_sel_policy *p)
{
	/* SSR allocates in a segment unit */
	if (p->alloc_mode == SSR)
		return sbi->blocks_per_seg; // SSR模式最大损耗为这个segment所有的块都是有效块，要迁移512个块
	if (p->gc_mode == GC_GREEDY)
		return 2 * sbi->blocks_per_seg * p->ofs_unit; // GC_GREEDY模式最大损耗为512 * 2，因为p->ofs_unit=1
	else if (p->gc_mode == GC_CB) // CB模式是无符号整型最大值
		return UINT_MAX;
	else /* No other gc_mode */
		return 0;
}

```

第二步就是计算当前segment的COST
```c
static inline unsigned int get_gc_cost(struct f2fs_sb_info *sbi,
			unsigned int segno, struct victim_sel_policy *p)
{
	if (p->alloc_mode == SSR)
		return get_seg_entry(sbi, segno)->ckpt_valid_blocks; // SSR模式计算上次Checkpoint的有效块数

	/* alloc_mode == LFS */
	if (p->gc_mode == GC_GREEDY)
		return get_valid_blocks(sbi, segno, true); // greedy模式直接获取segno对应的segment里面的有效块数
	else
		return get_cb_cost(sbi, segno);
}
```

COST-BENEFIT模式所计算的cost会考虑到时间信息，它的计算公式及实现如下:

<a href="https://www.codecogs.com/eqnedit.php?latex=cost&space;=&space;\frac{1-\mu&space;}{2\mu&space;}&space;\times&space;age" target="_blank"><img src="https://latex.codecogs.com/gif.latex?cost&space;=&space;\frac{1-\mu&space;}{2\mu&space;}&space;\times&space;age" title="cost = \frac{1-\mu }{2\mu } \times age" /></a> 

<a href="https://www.codecogs.com/eqnedit.php?latex=\mu" target="_blank"><img src="https://latex.codecogs.com/gif.latex?\mu" title="\mu" /></a>代表有效块占segment的比例，<a href="https://www.codecogs.com/eqnedit.php?latex=age" target="_blank"><img src="https://latex.codecogs.com/gif.latex?age" title="age" /></a>表示当前segment的上次修改时间。
<a href="https://www.codecogs.com/eqnedit.php?latex=1-\mu" target="_blank"><img src="https://latex.codecogs.com/gif.latex?1-\mu" title="1-\mu" /></a>表示对当前segment进行gc之后，能够获取可用块的比例。
<a href="https://www.codecogs.com/eqnedit.php?latex=2\mu" target="_blank"><img src="https://latex.codecogs.com/gif.latex?2\mu" title="2\mu" /></a>表示处理当前segment的开销，开销包括了读开销和写开销，因此使用<a href="https://www.codecogs.com/eqnedit.php?latex=2\mu" target="_blank"><img src="https://latex.codecogs.com/gif.latex?2\mu" title="2\mu" /></a>表示总体开销。
<a href="https://www.codecogs.com/eqnedit.php?latex=(1-\mu)/2\mu" target="_blank"><img src="https://latex.codecogs.com/gif.latex?(1-\mu)/2\mu" title="(1-\mu)/2\mu" /></a>因此可以理解为投入产出比。
<a href="https://www.codecogs.com/eqnedit.php?latex=age" target="_blank"><img src="https://latex.codecogs.com/gif.latex?age" title="age" /></a>的加入是更进一步考虑时间的影响，对于系统而言，肯定迁移长时间没有改变的数据，收益更高，因为这表示该segment刷写次数少，相对健康。因此COST-BENEFIT算法加入对时间的考虑，取两者相乘的结果作为cost返回，因此当cost计算的结果越大，表示这个segment越应该被gc。
在F2FS的算法实现中，使用了UINT_MAX减去上面计算的cost值，因此在F2FS中，`get_cb_cost` 的值越小越容易被gc。

```c

static unsigned int get_cb_cost(struct f2fs_sb_info *sbi, unsigned int segno)
{
	struct sit_info *sit_i = SIT_I(sbi);
	unsigned int secno = GET_SEC_FROM_SEG(sbi, segno);
	unsigned int start = GET_SEG_FROM_SEC(sbi, secno);
	unsigned long long mtime = 0;
	unsigned int vblocks;
	unsigned char age = 0;
	unsigned char u;
	unsigned int i;

	for (i = 0; i < sbi->segs_per_sec; i++)
		mtime += get_seg_entry(sbi, start + i)->mtime;
	vblocks = get_valid_blocks(sbi, segno, true); // 获得有效块数

	mtime = div_u64(mtime, sbi->segs_per_sec); // 获得修改时间
	vblocks = div_u64(vblocks, sbi->segs_per_sec); // 因为1sec=1seg所以结果一样

	u = (vblocks * 100) >> sbi->log_blocks_per_seg; // 除以512，表示有效块占据segment的比例

	/* Handle if the system time has changed by the user */
	if (mtime < sit_i->min_mtime) // 更新SIT的数据
		sit_i->min_mtime = mtime;
	if (mtime > sit_i->max_mtime)
		sit_i->max_mtime = mtime;
	if (sit_i->max_mtime != sit_i->min_mtime)
		age = 100 - div64_u64(100 * (mtime - sit_i->min_mtime),
				sit_i->max_mtime - sit_i->min_mtime); // 计算age的时间

	return UINT_MAX - ((100 * (100 - u) * age) / (100 + u)); // 注意这里使用了UINT_MAX减去计算值，因此cost值越小越好
}
```

了解了cost怎么计算之后，可以知道cost的值越低，这个segment越应该被gc。
然后继续回到对 `get_victim_by_default` 的分析，初始化了victim_sel_policy之后，开始进入while循环，它的逻辑很简单，从 `MAIN_SEGS(sbi) + p.offset` 开始遍历每一个dirty的segment，然后计算它的cost，最后达到了最大搜索次数，就跳出while循环。然后返回遍历过程的cost最小segment对应的segno，作为victim。
```c
static int get_victim_by_default(struct f2fs_sb_info *sbi,
		unsigned int *result, int gc_type, int type, char alloc_mode)
{
	struct dirty_seglist_info *dirty_i = DIRTY_I(sbi); // 从sbi获取记录segment的dirty信息的bitmap
	struct sit_info *sm = SIT_I(sbi);
	struct victim_sel_policy p; // 这个变量起到记录segno，和结束循环的作用
	unsigned int secno, last_victim;
	unsigned int last_segment = MAIN_SEGS(sbi); // 获取main area的第一个segment的地址，segno是根据第一个segment地址计算的
	unsigned int nsearched = 0;

	mutex_lock(&dirty_i->seglist_lock);

	p.alloc_mode = alloc_mode;
	select_policy(sbi, gc_type, type, &p); // 根据gc_type和alloc type初始化victim_sel_policy

	p.min_segno = NULL_SEGNO; // min_segno记录的是cost最小的segno，初始化NULL_SEGNO
	p.min_cost = get_max_cost(sbi, &p); // 初始化为迁移的最大开销，之后会在循环中慢慢降低，直到找到最小开销

	if (*result != NULL_SEGNO) { // 特殊情况就退出
		if (IS_DATASEG(get_seg_entry(sbi, *result)->type) &&
			get_valid_blocks(sbi, *result, false) &&
			!sec_usage_check(sbi, GET_SEC_FROM_SEG(sbi, *result)))
			p.min_segno = *result;
		goto out;
	}

	if (p.max_search == 0)
		goto out;

	last_victim = sm->last_victim[p.gc_mode]; // 获取上次被gc的victim的segno
	/**
	 * 如果是LFS模式+前台GC模式，就选择上次被bg gc的segment，
	 * 这样可以保证拥有最少的有效块，因此在迁移这些块的时候时间消耗比较少，能够较快获得空间，同时gc速度比较快
	 */
	if (p.alloc_mode == LFS && gc_type == FG_GC) {
		p.min_segno = check_bg_victims(sbi);
		if (p.min_segno != NULL_SEGNO)
			goto got_it;
	}

	// 从main area的第一个segment，开始遍历每一个dirty的segment，计算cost
	while (1) {
		unsigned long cost;
		unsigned int segno;

		segno = find_next_bit(p.dirty_segmap, last_segment, p.offset); // 从dirty bitmap获取下一个dirty的segno
		if (segno >= last_segment) {
			if (sm->last_victim[p.gc_mode]) {
				last_segment =
					sm->last_victim[p.gc_mode];
				sm->last_victim[p.gc_mode] = 0;
				p.offset = 0;
				continue;
			}
			break;
		}

		p.offset = segno + p.ofs_unit;
		if (p.ofs_unit > 1) {
			p.offset -= segno % p.ofs_unit;
			nsearched += count_bits(p.dirty_segmap,
						p.offset - p.ofs_unit,
						p.ofs_unit);
		} else {
			nsearched++;
		}

		secno = GET_SEC_FROM_SEG(sbi, segno);

		if (sec_usage_check(sbi, secno))
			goto next;
		if (gc_type == BG_GC && test_bit(secno, dirty_i->victim_secmap))
			goto next;

		cost = get_gc_cost(sbi, segno, &p); // 计算cost，cost越小，越值得被gc

		if (p.min_cost > cost) { // 如果这次的cost小于上次的cost，那就更新最小cost，记录segno
			p.min_segno = segno;
			p.min_cost = cost;
		}
next:
		if (nsearched >= p.max_search) { // 达到了最大搜索次数，就跳出循环
			if (!sm->last_victim[p.gc_mode] && segno <= last_victim)
				sm->last_victim[p.gc_mode] = last_victim + 1;
			else
				sm->last_victim[p.gc_mode] = segno + 1;
			sm->last_victim[p.gc_mode] %= MAIN_SEGS(sbi);
			break;
		}
	}
	if (p.min_segno != NULL_SEGNO) {
got_it:
		if (p.alloc_mode == LFS) {
			secno = GET_SEC_FROM_SEG(sbi, p.min_segno);
			if (gc_type == FG_GC)
				sbi->cur_victim_sec = secno;
			else
				set_bit(secno, dirty_i->victim_secmap);
		}
		*result = (p.min_segno / p.ofs_unit) * p.ofs_unit; // 返回最小cost的segno，也就是victim segno

		trace_f2fs_get_victim(sbi->sb, type, gc_type, &p,
				sbi->cur_victim_sec,
				prefree_segments(sbi), free_segments(sbi));
	}
out:
	mutex_unlock(&dirty_i->seglist_lock);

	return (p.min_segno == NULL_SEGNO) ? 0 : 1;
}
```

