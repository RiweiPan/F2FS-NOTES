# F2FS笔记、备忘录

### 一、F2FS文件系统结构
1. 总体结构
2. Superblock结构
3. Checkpoint结构
4. Segment Infomation Table结构(SIT)
5. Node Address Table结构(NAT)
6. Segment Summary Area结构(SSA)
7. 对f2fs_fill_super的分析

### 二、文件和目录数据是如何保存在F2FS中
1. 与文件数据保存相关的数据结构以及它们的关系
2. 几个常用的跟数据保存相关的函数

### 三、文件与目录的创建以及删除
1. 一般文件的创建
2. 一般目录的创建
3. 一般文件的删除
4. 一般目录的删除

### 四、文件的读写
1. 读流程
2. 写流程

### 五、F2FS的GC流程
1. GC的一般流程
2. 如何选择victim segment
3. 如何对页进行迁移

### 六、F2FS的Checkpoint流程

### 七、F2FS的Recovery的流程

