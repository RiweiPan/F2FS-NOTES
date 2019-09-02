# victim segment的选择
在[垃圾回收流程分析](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS-GC/GC%E6%B5%81%E7%A8%8B%E4%BB%8B%E7%BB%8D.md)这一节提及到了gc会通过`__get_victim`函数去选择一个需要被gc的segment。怎么样的segment才适合被gc，而F2FS是如何实现这一个流程的呢? 本节对这部分源码进行分析。

我们首先分析`__get_victim`函数，如下所示:
```c
static int __get_victim(struct f2fs_sb_info *sbi, unsigned int *victim,
			int gc_type)
{
	struct sit_info *sit_i = SIT_I(sbi);
	int ret;

	mutex_lock(&sit_i->sentry_lock);
	ret = DIRTY_I(sbi)->v_ops->get_victim(sbi, victim, gc_type,
					      NO_CHECK_TYPE, LFS);
	mutex_unlock(&sit_i->sentry_lock);
	return ret;
}
```
这里f2fs通过接口的方式提供给用户一个实现自己的垃圾回收算法能力。在默认情况下f2fs通过`get_victim_by_default`实现victim segment的选择:
```c
static const struct victim_selection default_v_ops = {
	.get_victim = get_victim_by_default,
};

```

我们先浏览一个简化的`get_victim_by_default`函数的结构，再逐步分析:

```c
static int get_victim_by_default(struct f2fs_sb_info *sbi,
		unsigned int *result, int gc_type, int type, char alloc_mode)
{
	struct dirty_seglist_info *dirty_i = DIRTY_I(sbi);
	struct victim_sel_policy p;
	unsigned int secno, max_cost;
	int nsearched = 0;

	/* 初始化选择策略 */
	p.alloc_mode = alloc_mode;
	select_policy(sbi, gc_type, type, &p);

	p.min_segno = NULL_SEGNO;
	p.min_cost = max_cost = get_max_cost(sbi, &p);

	/* 前台gc模式要求快速释放空间，因此不做循环寻找，直接找到之前BG GC的时候所记录下来适合gc的的segment进行gc */
	if (p.alloc_mode == LFS && gc_type == FG_GC) {
		p.min_segno = check_bg_victims(sbi);
		if (p.min_segno != NULL_SEGNO)
			goto got_it;
	}

	/* 进入循环，找到一个最小cost的segment */
	while (1) {
		unsigned long cost;
		unsigned int segno;

		/* 从map里面找到一个dirty的segment所以对应segno出来 */
		segno = find_next_bit(p.dirty_segmap, MAIN_SEGS(sbi), p.offset);

		p.offset = segno + p.ofs_unit;
		if (p.ofs_unit > 1)
			p.offset -= segno % p.ofs_unit;

		secno = GET_SECNO(sbi, segno);
			
		/* 计算当前segment的cost */
		cost = get_gc_cost(sbi, segno, &p);

		/* 判断更新最小cost */
		if (p.min_cost > cost) {
			p.min_segno = segno;
			p.min_cost = cost;
		} else if (unlikely(cost == max_cost)) {
			continue;
		}

		/* 达到了最大搜索次数即退出 */
		if (nsearched++ >= p.max_search) {
			sbi->last_victim[p.gc_mode] = segno;
			break;
		}
	}
	if (p.min_segno != NULL_SEGNO) {
got_it:
		if (p.alloc_mode == LFS) {
			secno = GET_SECNO(sbi, p.min_segno);
			if (gc_type == FG_GC)
				sbi->cur_victim_sec = secno;
			else
				set_bit(secno, dirty_i->victim_secmap); // BG_GC的情况下设定这个map，给FG_GC快速寻找segment进行gc
		}
		*result = (p.min_segno / p.ofs_unit) * p.ofs_unit; // 返回结果
	}

	return (p.min_segno == NULL_SEGNO) ? 0 : 1;
}
```
### 设置victim segment选择策略
第一步设置`victim_sel_policy`，即victim segment选择策略:
```c
struct victim_sel_policy p;
p.alloc_mode = alloc_mode; // 设置分配模式，一般为LFS
select_policy(sbi, gc_type, type, &p); // 根据gc类型等信息，设定选择策略
p.min_segno = NULL_SEGNO;
p.min_cost = max_cost = get_max_cost(sbi, &p); // 根据不同的policy，设定最大回收cost
```
其中
```c
static void select_policy(struct f2fs_sb_info *sbi, int gc_type,
			int type, struct victim_sel_policy *p)
{
	struct dirty_seglist_info *dirty_i = DIRTY_I(sbi);

	if (p->alloc_mode == SSR) {
		p->gc_mode = GC_GREEDY;
		p->dirty_segmap = dirty_i->dirty_segmap[type];
		p->max_search = dirty_i->nr_dirty[type];
		p->ofs_unit = 1;
	} else {
		p->gc_mode = select_gc_type(sbi->gc_thread, gc_type); // 赋予gc算法类型，默认有两种算法，即greedy算法(GC_GREEDY)，以及cost-benefit算法(GC_CB)
		p->dirty_segmap = dirty_i->dirty_segmap[DIRTY]; // 即脏的segment所对应的segmap，它的作用是标记了这个segmen里面的block有效无效信息
		p->max_search = dirty_i->nr_dirty[DIRTY]; // 表示最大搜索次数，等于当前有多少个dirty的segment
		p->ofs_unit = sbi->segs_per_sec; // 1 sec = 1 seg ，所以等于1
	}

	if (p->max_search > sbi->max_victim_search)
		p->max_search = sbi->max_victim_search;

	p->offset = sbi->last_victim[p->gc_mode]; // 上一个被回收的segment
}
```

### 查找最小cost的segment
在分析下一步循环之前，我们要明确循环的目的是什么。f2fs通过变量**cost**表示gc这个segment所带来的开销，cost的值根据不同的算法也不一样。`get_victim_by_default`函数目的是找到一个cost最低的segment进行回收，因此在找到之前需要设定一个最大cost，用于一步步遍历降低cost。
```c
	while (1) {
		unsigned long cost;
		unsigned int segno;

		/* 从map里面找到一个dirty的segment所以对应segno出来 */
		segno = find_next_bit(p.dirty_segmap, MAIN_SEGS(sbi), p.offset);

		p.offset = segno + p.ofs_unit;
		if (p.ofs_unit > 1)
			p.offset -= segno % p.ofs_unit;

		secno = GET_SECNO(sbi, segno);
			
		/* 计算当前segment的cost */
		cost = get_gc_cost(sbi, segno, &p);

		/* 判断更新最小cost */
		if (p.min_cost > cost) {
			p.min_segno = segno;
			p.min_cost = cost;
		} else if (unlikely(cost == max_cost)) {
			continue;
		}

		/* 达到了最大搜索次数即退出 */
		if (nsearched++ >= p.max_search) {
			sbi->last_victim[p.gc_mode] = segno;
			break;
		}
	}
```
首先通过`find_next_bit` 函数，从`p.dirty_segmap`找到第一个dirty的segment出来，然后通过`get_gc_cost`函数计算这个segment的cost。然后判断是否少于`p.min_cost`，然后赋值。最后循环到达了最大搜索次数以后旧退出。此时的segment及时本次选择过程中得到的cost最小的segment，最后返回到*result中，完成这个victim segment选择流程。

### cost算法
F2FS使用了两种计算cost的算法，分别是Greedy算法和Cost-Benefit算法。

```c
static inline unsigned int get_gc_cost(struct f2fs_sb_info *sbi,
			unsigned int segno, struct victim_sel_policy *p)
{
	if (p->alloc_mode == SSR)
		return get_seg_entry(sbi, segno)->ckpt_valid_blocks;

	/* alloc_mode == LFS */
	if (p->gc_mode == GC_GREEDY)
		return get_valid_blocks(sbi, segno, sbi->segs_per_sec); // Greedy算法，valid block越多表示cost越大，越不值得gc
	else
		return get_cb_cost(sbi, segno); // Cost-Benefit算法，这个是考虑了访问时间和valid block开销的算法
}
```
由于是通过cost值表示是否值得被gc，因此cost值是越小越好。

**Greedy算法**
顾名思义，会选择invalid block最多(valid block最少)的segment进行gc，实现如下:
```c
static inline unsigned int get_valid_blocks(struct f2fs_sb_info *sbi,
				unsigned int segno, int section)
{
	if (section > 1)
		return get_sec_entry(sbi, segno)->valid_blocks;
	else
		return get_seg_entry(sbi, segno)->valid_blocks;
}
```
由于是通过cost来描述开销，因此这里valid_block越多，表示开销越大。

**Cost-Benefit算法**
Cost-Benefit算法是一个同时考虑最近一次修改时间以及invalid block个数的算法。因为相当于频繁修改的数据而言，不值得进行GC，因为GC完很快就修改了，同时由于异地更新的特性，导致继续产生invalid block。较长时间未作修改的数据，可以认为迁移以后也相对没那么频繁继续产生invalid block。Cost-Benefit算法的核心是:
```
cost = (1 - u) / 2u * age
其中
u: 表示valid block在该section中的比例，age代表该section最近一次修改的时间
1-u: 表示对这个section进行gc后的收益
2u: 则表示开销(读+写)
age: 表示上一次修改时间
```
因此我们可以将Cost-Benefit算法理解为一个平衡invalid block数目以及修改时间的的一个算法，在f2fs的实现如下:
```c
static unsigned int get_cb_cost(struct f2fs_sb_info *sbi, unsigned int segno)
{
	struct sit_info *sit_i = SIT_I(sbi);
	unsigned int secno = GET_SECNO(sbi, segno);
	unsigned int start = secno * sbi->segs_per_sec;
	unsigned long long mtime = 0;
	unsigned int vblocks;
	unsigned char age = 0;
	unsigned char u;
	unsigned int i;

	for (i = 0; i < sbi->segs_per_sec; i++)
		mtime += get_seg_entry(sbi, start + i)->mtime; // 计算section里面的每一个segment最近一次访问时间
	vblocks = get_valid_blocks(sbi, segno, sbi->segs_per_sec); // 获取当前的section有多少个valid block

	mtime = div_u64(mtime, sbi->segs_per_sec); // 计算平均每一segment的最近一次访问时间
	vblocks = div_u64(vblocks, sbi->segs_per_sec); // 计算平均每一个segment的valid block个数

	u = (vblocks * 100) >> sbi->log_blocks_per_seg; // 百分比计算所以乘以100，然后计算得到了valid block的比例

	/* sit_i->min_mtime以及sit_i->max_mtime计算的是每一次gc的时候的最小最大的修改时间，因此通过这个比例计算这个section的修改时间在总体的情况下的表现 */
	if (mtime < sit_i->min_mtime)
		sit_i->min_mtime = mtime;
	if (mtime > sit_i->max_mtime)
		sit_i->max_mtime = mtime;
	if (sit_i->max_mtime != sit_i->min_mtime)
		age = 100 - div64_u64(100 * (mtime - sit_i->min_mtime),
				sit_i->max_mtime - sit_i->min_mtime);

	return UINT_MAX - ((100 * (100 - u) * age) / (100 + u));
}
```
第一步先计算一个section平均每一个segment的valid block数目是多少，然后计算valid block的比例。
第二步计算上一次修改时间占据最大最小的修改时间的比例。
最后一步，公式`((100 * (100 - u) * age) / (100 + u))`即对应`(1 - u) / 2u * age`，使用UINT_MAX减去这个值的原因是f2fs要维持cost越高，越不值得被gc的特征。















