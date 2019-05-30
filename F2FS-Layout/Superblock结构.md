# Superblock结构
Superblock保存了F2FS的核心元数据的结构，包括磁盘大小，元区域的各个部分的起始地址等。`f2fs_super_block`则是F2FS对Superblock的具体数据结构实现，它保存在磁盘的最开始的位置，F2FS进行启动的时候从磁盘的前端直接读取出来。

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

`f2fs_super_block`只在文件系统初始化的时候使用，表示实际存在于磁盘中的数据。大部分情况下系统使用的都是superblock的另外一个结构`f2fs_sb_info`，简称`sbi`，这个结构在文件系统初始化时侯，通过读取`f2fs_super_block`的数据进行初始化，只存于内存当中。这个结构是F2FS文件系统使用最多的数据结构，因为它包含了SIT、NAT、SSA、Checkpoint等多个重要的元数据结构信息，因此几乎F2FS中所有的动作都需要通过`sbi`进行处理。