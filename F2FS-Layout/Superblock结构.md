# Superblock结构
Superblock保存了F2FS的核心元数据的结构，包括磁盘大小，元区域的各个部分的起始地址等。

## Superblock物理存放区域结构
`f2fs_super_block`是F2FS对Superblock的具体数据结构实现，它保存在磁盘的最开始的位置，F2FS进行启动的时候从磁盘的前端直接读取出来。

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


对于一个50MB大小的磁盘，格式化后的`f2fs_super_block` 的信息如下:
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

## Superblock内存存管理结构
如上一节所述，`f2fs_super_block`在内存中的对应的结构是`struct f2fs_sb_info`，它除了包含了`struct f2fs_super_block`的信息以外，还包含了一些额外的功能，如锁、SIT、NAT对应的内存管理结构等，简单如下所述：
```c
struct f2fs_sb_info {
	struct super_block *sb;			/* pointer to VFS super block */
	struct f2fs_super_block *raw_super;	/* raw super block pointer */
	struct rw_semaphore sb_lock;		/* lock for raw super block */

	/* for node-related operations */
	struct f2fs_nm_info *nm_info;		/* node manager */
	struct inode *node_inode;		/* cache node blocks */

	/* for segment-related operations */
	struct f2fs_sm_info *sm_info;		/* segment manager */

	/* for checkpoint */
	struct f2fs_checkpoint *ckpt;		/* raw checkpoint pointer */

	/* for orphan inode, use 0'th array */
	unsigned int max_orphans;		/* max orphan inodes */

	struct f2fs_mount_info mount_opt;	/* mount options */

	/* for cleaning operations */
	struct mutex gc_mutex;			/* mutex for GC */
	struct f2fs_gc_kthread	*gc_thread;	/* GC thread */
	unsigned int cur_victim_sec;		/* current victim section num */
	unsigned int gc_mode;			/* current GC state */
};
```
它的初始化在`init_sb_info`函数完成:
```c
static void init_sb_info(struct f2fs_sb_info *sbi)
{
	struct f2fs_super_block *raw_super = sbi->raw_super;
	int i, j;

	sbi->log_sectors_per_block =
		le32_to_cpu(raw_super->log_sectors_per_block);
	sbi->log_blocksize = le32_to_cpu(raw_super->log_blocksize);
	sbi->blocksize = 1 << sbi->log_blocksize;
	sbi->log_blocks_per_seg = le32_to_cpu(raw_super->log_blocks_per_seg);
	sbi->blocks_per_seg = 1 << sbi->log_blocks_per_seg;
	sbi->segs_per_sec = le32_to_cpu(raw_super->segs_per_sec);
	sbi->secs_per_zone = le32_to_cpu(raw_super->secs_per_zone);
	sbi->total_sections = le32_to_cpu(raw_super->section_count);
	sbi->total_node_count =
		(le32_to_cpu(raw_super->segment_count_nat) / 2)
			* sbi->blocks_per_seg * NAT_ENTRY_PER_BLOCK;
	sbi->root_ino_num = le32_to_cpu(raw_super->root_ino);
	sbi->node_ino_num = le32_to_cpu(raw_super->node_ino);
	sbi->meta_ino_num = le32_to_cpu(raw_super->meta_ino);
	sbi->cur_victim_sec = NULL_SECNO;
	sbi->max_victim_search = DEF_MAX_VICTIM_SEARCH;

	sbi->dir_level = DEF_DIR_LEVEL;
	sbi->interval_time[CP_TIME] = DEF_CP_INTERVAL;
	sbi->interval_time[REQ_TIME] = DEF_IDLE_INTERVAL;
	clear_sbi_flag(sbi, SBI_NEED_FSCK);

	for (i = 0; i < NR_COUNT_TYPE; i++)
		atomic_set(&sbi->nr_pages[i], 0);

	for (i = 0; i < META; i++)
		atomic_set(&sbi->wb_sync_req[i], 0);

	INIT_LIST_HEAD(&sbi->s_list);
	mutex_init(&sbi->umount_mutex);
	for (i = 0; i < NR_PAGE_TYPE - 1; i++)
		for (j = HOT; j < NR_TEMP_TYPE; j++)
			mutex_init(&sbi->wio_mutex[i][j]);
	init_rwsem(&sbi->io_order_lock);
	spin_lock_init(&sbi->cp_lock);

	sbi->dirty_device = 0;
	spin_lock_init(&sbi->dev_lock);

	init_rwsem(&sbi->sb_lock);
}

```