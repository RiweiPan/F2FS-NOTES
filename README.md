# F2FS笔记、备忘录

### 实验环境的搭建
[实验环境的搭建](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS-Experiment/实验环境搭建.md)

### 一、文件系统布局以及结构
1. [总体结构](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS-Layout/%E6%80%BB%E4%BD%93%E7%BB%93%E6%9E%84.md)
2. [Superblock结构](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS-Layout/Superblock%E7%BB%93%E6%9E%84.md)
3. [Checkpoint结构](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS-Layout/Checkpoint%E7%BB%93%E6%9E%84.md)
4. [Segment Infomation Table结构(SIT)](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS-Layout/Segment%20Infomation%20Table%E7%BB%93%E6%9E%84.md)
5. [Node Address Table结构(NAT)](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS-Layout/Node%20Address%20Table%E7%BB%93%E6%9E%84.md)
6. [Segment Summary Area结构(SSA)](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS-Layout/Segment%20Summary%20Area%E7%BB%93%E6%9E%84.md)

### 二、文件数据的存储以及读写
1. [文件读写与物理地址的映射](https://github.com/RiweiPan/F2FS-NOTES/blob/master/Reading-and-Writing/file_data_structure.md)
2. [读流程](https://github.com/RiweiPan/F2FS-NOTES/blob/master/Reading-and-Writing/%E8%AF%BB%E6%B5%81%E7%A8%8B.md)
3. [写流程](https://github.com/RiweiPan/F2FS-NOTES/blob/master/Reading-and-Writing/%E5%86%99%E6%B5%81%E7%A8%8B.md)

### 三、文件与目录的创建以及删除
1. [一般文件的创建](https://github.com/RiweiPan/F2FS-NOTES/blob/master/File-Creation-and-Deletion/%E6%96%87%E4%BB%B6%E5%88%9B%E5%BB%BA.md)
2. [一般目录的创建](https://github.com/RiweiPan/F2FS-NOTES/blob/master/File-Creation-and-Deletion/%E7%9B%AE%E5%BD%95%E5%88%9B%E5%BB%BA.md)
3. [一般文件的删除](https://github.com/RiweiPan/F2FS-NOTES/blob/master/File-Creation-and-Deletion/%E7%9B%AE%E5%BD%95%E5%88%9B%E5%BB%BA.md)
4. [一般目录的删除](https://github.com/RiweiPan/F2FS-NOTES/blob/master/File-Creation-and-Deletion/%E7%9B%AE%E5%BD%95%E5%88%A0%E9%99%A4.md)

### 四、垃圾回收流程
1. [垃圾回收流程分析](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS-GC/GC%E6%B5%81%E7%A8%8B%E4%BB%8B%E7%BB%8D.md)
2. [如何选择victim segment](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS-GC/%E9%80%89%E6%8B%A9victim%20segment.md)
3. [如何对页进行迁移](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS-GC/%E5%A6%82%E4%BD%95%E8%BF%9B%E8%A1%8C%E9%A1%B5%E8%BF%81%E7%A7%BB.md)

### 五、数据恢复流程
1. [数据恢复的原理以及方式](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS-Data-Recovery/%E6%95%B0%E6%8D%AE%E6%81%A2%E5%A4%8D%E7%9A%84%E5%8E%9F%E7%90%86%E4%BB%A5%E5%8F%8A%E6%96%B9%E5%BC%8F.md)
2. [后滚恢复和Checkpoint的作用与实现](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS-Data-Recovery/Checkpoint%E6%B5%81%E7%A8%8B.md)
3. [前滚恢复和Recovery的作用与实现](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS-Data-Recovery/Recovery%E7%9A%84%E6%B5%81%E7%A8%8B.md)

### 六、重要数据结构或者函数的分析
1. [f2fs_summary和f2fs_summary_block的介绍和应用](https://github.com/RiweiPan/F2FS-NOTES/blob/master/ImportantDataStructure/f2fs_summary.md)
2. [seg_entry和sit_info的作用](https://github.com/RiweiPan/F2FS-NOTES/blob/master/ImportantDataStructure/segment.md) 
3. [f2fs_journal的作用](https://github.com/RiweiPan/F2FS-NOTES/blob/master/ImportantDataStructure/f2fs_journal.md)
4. [f2fs_fill_super的分析](https://github.com/RiweiPan/F2FS-NOTES/blob/master/ImportantDataStructure/f2fs_fill_super.md)
5. [f2fs_map_block的作用](https://github.com/RiweiPan/F2FS-NOTES/blob/master/ImportantDataStructure/f2fs_map_blocks.md)
6. [CURSEG的作用](https://github.com/RiweiPan/F2FS-NOTES/blob/master/ImportantDataStructure/curseg.md)
7. [物理地址寻址的实现](https://github.com/RiweiPan/F2FS-NOTES/blob/master/ImportantDataStructure/get_dnode.md) 