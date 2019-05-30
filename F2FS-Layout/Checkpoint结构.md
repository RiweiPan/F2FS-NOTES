# Checkpoint结构
Checkpoint是维护F2FS的数据一致性的结构，它维护了系统当前的状态，例如segment的分配情况，node的分配情况，以及当前的active segment的状态等。F2FS在满足一定的条件的情况下，将当前系统的状态写入Checkpoint中，万一系统出现突然宕机，这个是F2FS可以从Checkpoint中恢复到上次回写时的状态，以保证数据的可恢复性。F2FS维护了两个Checkpoint结构，互为备份，其中一个是当前正在使用的Checkpoint，另外一个上次回写的稳定的Chcekpoint。如果系统出现了宕机，那么当前的Checkpoint就会变得不可信任，进而使用备份Checkpoint进行恢复。


Checkpoint区域由几个部分构成，分别是checkpoint元数据区域、orphan node区域、active segment区域：

```
+-------------------+-------------+----------------+-------------------+
| f2fs_checkpoint 1 | orphan node | active segment | f2fs_checkpoint 2 |
+-------------------+-------------+----------------+-------------------+
```

## Checkpoint元数据区域
F2FS使用数据结构`f2fs_checkpoint`表示Checkpoint结构，它保存在磁盘中`f2fs_super_block`之后区域中，数据结构如下：
```c
struct f2fs_checkpoint {
	__le64 checkpoint_ver;		/* CP版本，用于比较新旧版本进行恢复 */
	__le64 user_block_count;	/* # of user blocks */
	__le64 valid_block_count;	/* # of valid blocks in main area */
	__le32 rsvd_segment_count;	/* # of reserved segments for gc */
	__le32 overprov_segment_count;	/* # of overprovision segments */
	__le32 free_segment_count;	/* # of free segments in main area */

	/* information of current node segments */
	__le32 cur_node_segno[MAX_ACTIVE_NODE_LOGS];
	__le16 cur_node_blkoff[MAX_ACTIVE_NODE_LOGS];
	/* information of current data segments */
	__le32 cur_data_segno[MAX_ACTIVE_DATA_LOGS];
	__le16 cur_data_blkoff[MAX_ACTIVE_DATA_LOGS];
	__le32 ckpt_flags;		/* Flags : umount and journal_present */
	__le32 cp_pack_total_block_count;	/* total # of one cp pack */
	__le32 cp_pack_start_sum;	/* start block number of data summary */
	__le32 valid_node_count;	/* Total number of valid nodes */
	__le32 valid_inode_count;	/* Total number of valid inodes */
	__le32 next_free_nid;		/* Next free node number */
	__le32 sit_ver_bitmap_bytesize;	/* Default value 64 */
	__le32 nat_ver_bitmap_bytesize; /* Default value 256 */
	__le32 checksum_offset;		/* checksum offset inside cp block */
	__le64 elapsed_time;		/* mounted time */
	/* allocation type of current segment */
	unsigned char alloc_type[MAX_ACTIVE_LOGS];

	/* SIT and NAT version bitmap */
	unsigned char sit_nat_version_bitmap[1];
} __packed;
```

## Orphan node区域
这是一个动态的区域，如果没有orphan node list则不会占用空间


## Active Segment区域
### Active Segment的定义
Active Segment即当前正在用于分配的segment，如用户需要写入8KB数据，那么就会从active segment分配两个block提供给用户写入到磁盘中。F2FS为了提高数据分配的效率，根据数据的特性，一共定义了6个active segment。如[总体结构](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS-Layout/%E6%80%BB%E4%BD%93%E7%BB%93%E6%9E%84.md)这一节的multi-head logging特性所描述，这6个active segment对应了(HOT, WARM, COLD) X (NODE, DATA)的数据。

### Active Segment与恢复相关的数据结构
CP的主要任务是维护数据一致性，因此CP的Active Segment区域的主要任务是维护Active Segment的分配状态，使系统宕机时候可以恢复正常。维护Active Segment需要维护三种信息，分别是`f2fs_checkpoint`的信息，以及该segment对应的journal和summary的信息。

- **f2fs_checkpoint中Active Segment信息**：从上面给出的`f2fs_checkpoint`定义，`cur_node_segno[MAX_ACTIVE_NODE_LOGS]`和`cur_data_segno[MAX_ACTIVE_DATA_LOGS]`表示node和data当前的Active Segment的编号，系统可以通过这个编号找到对应的segment。`MAX_ACTIVE_NODE_LOGS`以及`MAX_ACTIVE_NODE_LOGS`分别表示data和node有多少种类型，F2FS默认情况下都等于3，表示HOT、WARM、COLD类型数据。`cur_node_blkoff[MAX_ACTIVE_NODE_LOGS]`以及`cur_data_blkoff[MAX_ACTIVE_DATA_LOGS]`则分别表示当前Active Segment分配到哪一个block(一个segment包含了512个block)。

- **Segment对应的Journal信息**：Journal在两处地方都有出现，分别使CP区域以及SSA区域。F2FS定义的journal结构如下，它主要保存了NODE以及SEGMENT的修改信息。如系统分配出一个block给用户，那么就要将这个block在bitmap中标记为已分配，防止其他请求使用。分两个区域存放journal使为了减轻频繁更新导致的系统性能下降。例如，当系统写压力很大的时候，bitmap就会频繁被更新，如果这个时候频繁将bitmap写入SSA，就会加重写压力。因此CP区域的Journal的作用就是维护这些经常修改的数据，等待CP被触发的时候才吸入磁盘，从而减少写压力。
```c
struct f2fs_journal {
	union {
		__le16 n_nats;
		__le16 n_sits;
	};
	/* spare area is used by NAT or SIT journals or extra info */
	union {
		struct nat_journal nat_j;
		struct sit_journal sit_j;
		struct f2fs_extra_info info;
	};
} __packed;
```
- **Segment对应的Summary信息**：Summary同样在CP区域和SSA区域有出现，它表示的是逻辑地址和物理地址的映射关系，这个映射关系会使用到GC流程中。Summary与segment是一对一的关系，一个summary保存了一个segment所有的block的物理地址和逻辑地址的映射关系。Summary保存在CP区域中同样是出于减少IO的写入.