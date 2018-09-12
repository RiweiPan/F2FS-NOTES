# F2FS笔记、备忘录

### 一、F2FS文件系统结构
1. [总体结构](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS%20Layout/%E6%80%BB%E4%BD%93%E7%BB%93%E6%9E%84.md)
2. [Superblock结构](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS%20Layout/Superblock%E7%BB%93%E6%9E%84.md)
3. [Checkpoint结构](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS%20Layout/Checkpoint%E7%BB%93%E6%9E%84.md)
4. [Segment Infomation Table结构(SIT)](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS%20Layout/Segment%20Infomation%20Table%E7%BB%93%E6%9E%84.md)
5. [Node Address Table结构(NAT)](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS%20Layout/Node%20Address%20Table%E7%BB%93%E6%9E%84.md)
6. [Segment Summary Area结构(SSA)](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS%20Layout/Segment%20Summary%20Area%E7%BB%93%E6%9E%84.md)
7. [对f2fs_fill_super的分析](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS%20Layout/%E5%AF%B9f2fs_fill_super%E7%9A%84%E5%88%86%E6%9E%90.md)

### 二、文件和目录数据是如何保存在F2FS中
1. [与文件数据保存相关的数据结构以及它们的关系](https://github.com/RiweiPan/F2FS-NOTES/blob/master/File%20Data%20Saving/%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E5%8F%8A%E5%85%B3%E7%B3%BB.md)
2. [几个常用的跟数据保存相关的函数](https://github.com/RiweiPan/F2FS-NOTES/blob/master/File%20Data%20Saving/%E5%B8%B8%E7%94%A8%E5%87%BD%E6%95%B0.md)

### 三、文件与目录的创建以及删除
1. [一般文件的创建](https://github.com/RiweiPan/F2FS-NOTES/blob/master/File%20Creation%20and%20Deletion/%E6%96%87%E4%BB%B6%E5%88%9B%E5%BB%BA.md)
2. [一般目录的创建](https://github.com/RiweiPan/F2FS-NOTES/blob/master/File%20Creation%20and%20Deletion/%E7%9B%AE%E5%BD%95%E5%88%9B%E5%BB%BA.md)
3. [一般文件的删除](https://github.com/RiweiPan/F2FS-NOTES/blob/master/File%20Creation%20and%20Deletion/%E6%96%87%E4%BB%B6%E5%88%A0%E9%99%A4.md)
4. [一般目录的删除](https://github.com/RiweiPan/F2FS-NOTES/blob/master/File%20Creation%20and%20Deletion/%E7%9B%AE%E5%BD%95%E5%88%A0%E9%99%A4.md)

### 四、文件的读写
1. [读流程](https://github.com/RiweiPan/F2FS-NOTES/blob/master/Reading%20and%20Writing/%E8%AF%BB%E6%B5%81%E7%A8%8B.md)
2. [写流程](https://github.com/RiweiPan/F2FS-NOTES/blob/master/Reading%20and%20Writing/%E5%86%99%E6%B5%81%E7%A8%8B.md)

### 五、F2FS的GC流程
1. [GC的一般流程](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS-GC/GC%E6%B5%81%E7%A8%8B%E4%BB%8B%E7%BB%8D.md)
2. [如何选择victim segment](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS-GC/%E9%80%89%E6%8B%A9victim%20segment.md)
3. [如何对页进行迁移](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS-GC/%E5%A6%82%E4%BD%95%E8%BF%9B%E8%A1%8C%E9%A1%B5%E8%BF%81%E7%A7%BB.md)

### 六、F2FS的Checkpoint流程

### 七、F2FS的Recovery的流程

