### Superblock结构

superblock是F2FS保存元数据最重要的数据结构，几乎每一个操作都要用到superblock的变量信息，首先先分析第一个superblock的相关数据结构

```c

struct f2fs_super_block {
	__le32 magic;			/* Magic Number */
	__le16 major_ver;		/* Major Version */
	__le16 minor_ver;		/* Minor Version */
	__le32 log_sectorsize;		/* log2 sector size in bytes */
	__le32 log_sectors_per_block;	/* log2 # of sectors per block 一般是3，因为 1 << 3 = 8 */
	__le32 log_blocksize;		/* log2 block size in bytes 一般是12，因为 1 << 12 = 4096 */
	__le32 log_blocks_per_seg;	/* log2 # of blocks per segment 一般是8，因为 1 << 8 = 512 */
	__le32 segs_per_sec;		/* # of segments per section */
	__le32 secs_per_zone;		/* # of sections per zone */
	__le32 checksum_offset;		/* checksum offset inside super block */
	__le64 block_count;		/* total # of user blocks */
	__le32 section_count;		/* total # of sections */
	__le32 segment_count;		/* total # of segments */
	__le32 segment_count_ckpt;	/* # of segments for checkpoint */
	__le32 segment_count_sit;	/* # of segments for SIT */
	__le32 segment_count_nat;	/* # of segments for NAT */
	__le32 segment_count_ssa;	/* # of segments for SSA */
	__le32 segment_count_main;	/* # of segments for main area */
	__le32 segment0_blkaddr;	/* start block address of segment 0 */
	__le32 cp_blkaddr;		/* start block address of checkpoint */
	__le32 sit_blkaddr;		/* start block address of SIT */
	__le32 nat_blkaddr;		/* start block address of NAT */
	__le32 ssa_blkaddr;		/* start block address of SSA */
	__le32 main_blkaddr;		/* start block address of main area */
	__le32 root_ino;		/* root inode number */
	__le32 node_ino;		/* node inode number */
	__le32 meta_ino;		/* meta inode number */
	__u8 uuid[16];			/* 128-bit uuid for volume */
	__le16 volume_name[MAX_VOLUME_NAME];	/* volume name */
	__le32 extension_count;		/* # of extensions below */
	__u8 extension_list[F2FS_MAX_EXTENSION][F2FS_EXTENSION_LEN];/* extension array */
	__le32 cp_payload;
	__u8 version[VERSION_LEN];	/* the kernel version */
	__u8 init_version[VERSION_LEN];	/* the initial kernel version */
	__le32 feature;			/* defined features */
	__u8 encryption_level;		/* versioning level for encryption */
	__u8 encrypt_pw_salt[16];	/* Salt used for string2key algorithm */
	struct f2fs_device devs[MAX_DEVICES];	/* device list */
	__le32 qf_ino[F2FS_MAX_QUOTAS];	/* quota inode numbers */
	__u8 hot_ext_count;		/* # of hot file extension */
	__u8 reserved[314];		/* valid reserved region */
} __packed;

```


对于一个50MB大小的磁盘，格式化后的 `f2fs_super_block` 的信息如下:
```
magic = -218816496
major_ver = 1
minor_ver = 10
log_sectorsize = 9
log_sectors_per_block = 3
log_blocksize = 12
log_blocks_per_seg = 9
segs_per_sec = 1
secs_per_zone = 1
checksum_offset = 0
block_count = 12800 # 50MB / 4KB = 12800
section_count = 17     # section只在main area应用，因此和main area一样
segment_count = 24
segment_count_ckpt = 2 # checkpoint用了2个segment
segment_count_sit = 2  # SIT也用了2个segment
segment_count_nat = 2  # NAT也用了2个segment
segment_count_ssa = 1  # SSA用了1个segment
segment_count_main = 17 # main area一共有17个可用的segment
segment0_blkaddr = 512
cp_blkaddr = 512       # checkpoint的地址
sit_blkaddr = 1536     # sit的地址
nat_blkaddr = 2560	   # nat的地址
ssa_blkaddr = 3584     # ssa的地址
main_blkaddr = 4096    # main area的地址
root_ino = 3
node_ino = 1
meta_ino = 2
extension_count = 27
cp_payload = 0
feature = 0
encryption_level = 
```

`f2fs_super_block` 结构保存的是最基础的元信息，包括磁盘大小，元区域的各个部分的起始地址等。这个数据结构保存在磁盘的前端的位置，F2FS进行启动的时候从磁盘的前端直接读取出来。而磁盘本身的数据，则是通过 `mkfs.f2fs` 进行初始化。这个数据结构只在初始化的时候使用，大部分情况下系统使用的都是superblock的另外一个结构:

```c

struct f2fs_sb_info {
	struct super_block *sb;			/* pointer to VFS super block */
	struct proc_dir_entry *s_proc;		/* proc entry */
	struct f2fs_super_block *raw_super;	/* raw super block pointer */
	struct rw_semaphore sb_lock;		/* lock for raw super block */
	int valid_super_block;			/* valid super block no */
	unsigned long s_flag;				/* flags for sbi */

#ifdef CONFIG_BLK_DEV_ZONED
	unsigned int blocks_per_blkz;		/* F2FS blocks per zone */
	unsigned int log_blocks_per_blkz;	/* log2 F2FS blocks per zone */
#endif

	/* for node-related operations */
	struct f2fs_nm_info *nm_info;		/* node manager */
	struct inode *node_inode;		/* cache node blocks */

	/* for segment-related operations */
	struct f2fs_sm_info *sm_info;		/* segment manager */

	/* for bio operations */
	/*
	 * sbi->write_io指针数组保存的是不同类型(NODE\DATA\META)的bio数据，每个指针指向一个大小为3的f2fs_bio_info数组，代表不同的冷热程度
	 * sbi->write_io[0][0~2] = DATA-HOT DATA-WARM DATA-COLD
	 * sbi->write_io[1][0~2] = NODE-HOT NODE-WARM NODE-COLD
	 * sbi->write_io[2][0~2] = META-HOT META-WARM META-COLD
	 * 它的作用是：当磁盘写入多个页的时候，可以通过这个bio暂存要写入到磁盘的页，等到满足一定的条件，就批量刷写入磁盘，提高IO效率
	 * */
	struct f2fs_bio_info *write_io[NR_PAGE_TYPE];	/* for write bios */
	struct mutex wio_mutex[NR_PAGE_TYPE - 1][NR_TEMP_TYPE];
						/* bio ordering for NODE/DATA */
	/* keep migration IO order for LFS mode */
	struct rw_semaphore io_order_lock;
	mempool_t *write_io_dummy;		/* Dummy pages */

	/* for checkpoint */
	struct f2fs_checkpoint *ckpt;		/* raw checkpoint pointer */
	int cur_cp_pack;			/* remain current cp pack */
	spinlock_t cp_lock;			/* for flag in ckpt */
	struct inode *meta_inode;		/* cache meta blocks */
	struct mutex cp_mutex;			/* checkpoint procedure lock */
	struct rw_semaphore cp_rwsem;		/* blocking FS operations */
	struct rw_semaphore node_write;		/* locking node writes */
	struct rw_semaphore node_change;	/* locking node change */
	wait_queue_head_t cp_wait;
	unsigned long last_time[MAX_TIME];	/* to store time in jiffies */
	long interval_time[MAX_TIME];		/* to store thresholds */

	struct inode_management im[MAX_INO_ENTRY];      /* manage inode cache */

	/* for orphan inode, use 0'th array */
	unsigned int max_orphans;		/* max orphan inodes */

	/* for inode management */
	struct list_head inode_list[NR_INODE_TYPE];	/* dirty inode list */
	spinlock_t inode_lock[NR_INODE_TYPE];	/* for dirty inode list lock */

	/* for extent tree cache */
	struct radix_tree_root extent_tree_root;/* cache extent cache entries */
	struct mutex extent_tree_lock;	/* locking extent radix tree */
	struct list_head extent_list;		/* lru list for shrinker */
	spinlock_t extent_lock;			/* locking extent lru list */
	atomic_t total_ext_tree;		/* extent tree count */
	struct list_head zombie_list;		/* extent zombie tree list */
	atomic_t total_zombie_tree;		/* extent zombie tree count */
	atomic_t total_ext_node;		/* extent info count */

	/* basic filesystem units */
	unsigned int log_sectors_per_block;	/* log2 sectors per block */
	unsigned int log_blocksize;		/* log2 block size */
	unsigned int blocksize;			/* block size */
	unsigned int root_ino_num;		/* root inode number*/
	unsigned int node_ino_num;		/* node inode number*/
	unsigned int meta_ino_num;		/* meta inode number*/
	unsigned int log_blocks_per_seg;	/* log2 blocks per segment */
	unsigned int blocks_per_seg;		/* blocks per segment */
	unsigned int segs_per_sec;		/* segments per section */
	unsigned int secs_per_zone;		/* sections per zone */
	unsigned int total_sections;		/* total section count */
	unsigned int total_node_count;		/* total node block count */
	unsigned int total_valid_node_count;	/* valid node block count */
	loff_t max_file_blocks;			/* max block index of file */
	int dir_level;				/* directory level */
	unsigned int trigger_ssr_threshold;	/* threshold to trigger ssr */
	int readdir_ra;				/* readahead inode in readdir */

	block_t user_block_count;		/* # of user blocks */
	block_t total_valid_block_count;	/* # of valid blocks */
	block_t discard_blks;			/* discard command candidats */
	block_t last_valid_block_count;		/* for recovery */
	block_t reserved_blocks;		/* configurable reserved blocks */
	block_t current_reserved_blocks;	/* current reserved blocks */

	unsigned int nquota_files;		/* # of quota sysfile */

	u32 s_next_generation;			/* for NFS support */

	/* # of pages, see count_type */
	atomic_t nr_pages[NR_COUNT_TYPE];
	/* # of allocated blocks */
	struct percpu_counter alloc_valid_block_count;

	/* writeback control */
	atomic_t wb_sync_req[META];	/* count # of WB_SYNC threads */

	/* valid inode count */
	struct percpu_counter total_valid_inode_count;

	struct f2fs_mount_info mount_opt;	/* mount options */

	/* for cleaning operations */
	struct mutex gc_mutex;			/* mutex for GC */
	struct f2fs_gc_kthread	*gc_thread;	/* GC thread */
	unsigned int cur_victim_sec;		/* current victim section num */
	unsigned int gc_mode;			/* current GC state */
	/* for skip statistic */
	unsigned long long skipped_atomic_files[2];	/* FG_GC and BG_GC */

	/* threshold for gc trials on pinned files */
	u64 gc_pin_file_threshold;

	/* maximum # of trials to find a victim segment for SSR and GC */
	unsigned int max_victim_search;

	/*
	 * for stat information.
	 * one is for the LFS mode, and the other is for the SSR mode.
	 */
#ifdef CONFIG_F2FS_STAT_FS
	struct f2fs_stat_info *stat_info;	/* FS status information */
	unsigned int segment_count[2];		/* # of allocated segments */
	unsigned int block_count[2];		/* # of allocated blocks */
	atomic_t inplace_count;		/* # of inplace update */
	atomic64_t total_hit_ext;		/* # of lookup extent cache */
	atomic64_t read_hit_rbtree;		/* # of hit rbtree extent node */
	atomic64_t read_hit_largest;		/* # of hit largest extent node */
	atomic64_t read_hit_cached;		/* # of hit cached extent node */
	atomic_t inline_xattr;			/* # of inline_xattr inodes */
	atomic_t inline_inode;			/* # of inline_data inodes */
	atomic_t inline_dir;			/* # of inline_dentry inodes */
	atomic_t aw_cnt;			/* # of atomic writes */
	atomic_t vw_cnt;			/* # of volatile writes */
	atomic_t max_aw_cnt;			/* max # of atomic writes */
	atomic_t max_vw_cnt;			/* max # of volatile writes */
	int bg_gc;				/* background gc calls */
	unsigned int ndirty_inode[NR_INODE_TYPE];	/* # of dirty inodes */
#endif
	spinlock_t stat_lock;			/* lock for stat operations */

	/* For app/fs IO statistics */
	spinlock_t iostat_lock;
	unsigned long long write_iostat[NR_IO_TYPE];
	bool iostat_enable;

	/* For sysfs suppport */
	struct kobject s_kobj;
	struct completion s_kobj_unregister;

	/* For shrinker support */
	struct list_head s_list;
	int s_ndevs;				/* number of devices */
	struct f2fs_dev_info *devs;		/* for device list */
	unsigned int dirty_device;		/* for checkpoint data flush */
	spinlock_t dev_lock;			/* protect dirty_device */
	struct mutex umount_mutex;
	unsigned int shrinker_run_no;

	/* For write statistics */
	u64 sectors_written_start;
	u64 kbytes_written;

	/* Reference to checksum algorithm driver via cryptoapi */
	struct crypto_shash *s_chksum_driver;

	/* Precomputed FS UUID checksum for seeding other checksums */
	__u32 s_chksum_seed;
};

```

`f2fs_sb_info` 这个数据结构在F2FS被应用到很多地方，而且有多种方式可以访问到这个结构，可见其重要性。

```c

static inline struct f2fs_sb_info *F2FS_SB(struct super_block *sb);
static inline struct f2fs_sb_info *F2FS_I_SB(struct inode *inode);
static inline struct f2fs_sb_info *F2FS_M_SB(struct address_space *mapping);
static inline struct f2fs_sb_info *F2FS_P_SB(struct page *page);

```

在F2FS几乎所有的行为都跟 `f2fs_sb_info` 有关，因此具体作用在使用到再提及。