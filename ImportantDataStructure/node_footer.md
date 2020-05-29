## Node Footer的作用

`footer`是F2FS中，记录node的属性的一个数据，它源码如下

```c
struct f2fs_node {
	union {
		struct f2fs_inode i;
		struct direct_node dn;
		struct indirect_node in;
	};
	struct node_footer footer;
} __packed;

struct node_footer {
	__le32 nid;		/* node id */
	__le32 ino;		/* inode nunmber */
	__le32 flag;		/* include cold/fsync/dentry marks and offset */
	__le64 cp_ver;		/* checkpoint version */
	__le32 next_blkaddr;	/* next node page block address */
} __packed;
```

F2FS有三种类型的node，分别是`f2fs_inode`、`direct_node`、`indirect_node`，每一种类型的node都有对应的footer。



### footer->nid和footer->ino

每一个node都有一个独特的`nid`，它被记录在`footer`中，如果是`direct_node`或者`indirect_node`，它们都有一个对应的`f2fs_inode`，因此为了记录从属关系，还需要`footer`记录它所属于的`f2fs_inode`的`nid`，即`ino`。因此，如果`footer->nid == footer->ino`，那么这个node就是inode，反正这个`node`是`direct_node`或者`indirect_node`。



### footer->flag

`footer->flag`的作用是标记当前的node的属性。目前F2FS给node定义了三种属性:

```c
enum {
	COLD_BIT_SHIFT = 0,
	FSYNC_BIT_SHIFT,
	DENT_BIT_SHIFT,
	OFFSET_BIT_SHIFT
};

#define OFFSET_BIT_MASK		(0x07)	/* (0x01 << OFFSET_BIT_SHIFT) - 1 */
```

其中`footer->flag`的

第0位表示这个node是否是cold node。

第1位表示这个node是否执行了完整的fsync。F2FS为了`fsync`的效率做了一些改进，F2FS不会在`fsync`刷写所有脏的node page进去磁盘，只会刷写一些根据data直接相关的node page进入磁盘，例如`f2fs_inode`和`direct_node`。因此这个标志位是用来记录这个node是否执行了完整的fsync，以便系统在crash中恢复。

第3位表示这个node是是用来保存文件数据，还是目录数据的，也是用于数据恢复



##　footer->cp_ver和footer->next_blkaddr

`footer->cp_ver`分别用来记录当前的checkpoint的version，恢复的时候比较version版本确定如何进行数据恢复。

`footer->next_blkaddr`则是用来记录这个node对应下一个node page的地址，也是用来恢复数据

