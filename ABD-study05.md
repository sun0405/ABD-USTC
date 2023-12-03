---
title: ADB-Study05
date: 2023-10-29 21:11:36
tags:
- ABD
categories:
- ABD-USTC
---

# Buffer Management

旨在: 最小化磁盘的I/O次数, 保持最可能被访问的在内存中.

## 缓冲区结构

![图二](/figures/05-02.png)

### frame的参数

- Dirty: Frame中的块是否已经被修改
- Pin-count: Frame的块以及被请求并且还未释放的计数, 即当前的用户数
- Others: latch, 是否加锁

### 请求块

如果请求块不在缓冲池中, 则选择一个frame来替代, 如果这个frame是dirty的, 将它写入磁盘, 读取请求块进入所选择的frame. Pin这个块并返回地址.



## 缓冲区替换算法

### 理论最优算法: OPT算法

OPT算法也称为Belady's算法

每次都置换以后永远也用不到的页面, 若没有则淘汰最久以后再用到的页面; OPT算法必须预先知道全部的页面访问序列, 但再实际的DBMS/OS中是无法实现的, 因此仅有理论意义.

作用: 可以在实验中作为算法性能的上界加以对比

### LRU

所以的frame按照最近一次访问时间排列成一个链表

LRU----()----()----()-- <---MRU

基于时间局部性假设: 越是最近访问的在未来被访问的概率越高, 总是替换LRU端的frame

优点:

- 能很好地作用于流行的页面
- 在置换时的复杂度是O(1)

缺点:

- 缓存污染, LRU+重复序列扫描, 因为它只考虑最近的一次访问时间.
- 开销: 每次都要修改LRU列表
- 如果访问不满足时间局部性则性能较差
- 如果仅考虑最近一次访问, 则不考虑访问频率

### LRU-K

在LRU基础上考虑了频率问题

如果某个frame的访问次数达到了k次以上, 则应当尽量不置换

- 维护2个LRU链表, 一个是访问次数小于K的, 一个是访问次数K次以上的
- 优先安装LRU策略置换小于K次的链表
- 保证高频访问的页能够尽量在BUFFER中
- K并非越大越好, 一般为2即可
- 缺点: 需要额外记录访问次数

### 2Q

与LRU-2类似,但不同的地方在于访问一次的队列使用FIFO, 而不是LRU.

<img src="/figures/05-01.png" style="zoom:43%;" />

### Second-Chance FIFO

相当于给了每个frame两次置换机会, 避免高频访问但最近一轮没有被访问的frame被置换出buffer

每个frame仅需要一个额外的Bit, 空间代价低

缺点: 置换时需要移动多个元素，理论性能比LRU差

### CLOCK

也是Second-Chance的变种, 将Second-Chance FIFO组织成环形

![图三](/figures/05-03.png)

N个frame组成环形，current指针指向当前 frame；每个frame有一个referenced位，初始为1； 当需要置换页时，从current开始检查，若 pin-count>0，current增加1；若referenced 已启动（=1），则关闭它（=0）并增加current（保证最近的不被替换）；若pin-count=0并且referenced关闭（=0），则替换该frame，同时current加1 

注意：Current指针只在置换时更新，访问命中时不改变Current指针

### SSD上的置换算法

传统的置换算法强调命中率, 但是SSD上的略有区别.

- 闪存: 读快写慢, 写次数有限(寿命限制)

减少缓存置换中对闪存的写是一个重要指标

- SSD-aware缓存算法

首先考虑改进**LRU**, 即有CFLRU算法(clean-first), 分为clean page, dirty page

LRU-WSR: clean-first+cold flag, 置换: clean > cold dirty > hot dirty, 冷热数据分类, 即考虑访问频率. 但是这样做对空间的占用比较大. 还有新鲜程度, 即时间和频率都要注意, 用一个行数来进行"老化", 

AD-LRU:

### 为何不使用OS缓冲区管理

DBMS经常能预测访问模强制式:

- 可以使用更专门的缓冲区替换策略
- 有利于pre-fetch(预取)策略的有效使用

但文件系统是难以预测的, 以文件为基础进行管理.

DBMS需要强制写回磁盘能力, OS的缓冲写回一般通过记录写请求来实现, 实际的磁盘修改推迟, 因此不能保证写顺序.

## 缓冲区管理器的实现

将key的请求转化为Page请求.

record请求 -> frame请求 -> page请求 ->  disk file I/O

### 错误的记录操作实现例子

````python
```
fopen();
fseek();
fwrite();
```
````

基于文件系统, 使用了FS的缓冲管理

- 不能保证WAL
- 不利于查询优化
- 不适应应用需求

### Block Vs Disk File

文件存储在磁盘上的物理形式是 bits/bytes，block是由OS或DBMS软件对文件所做的抽象，这一抽象是通过控制数据在文件中的起止offset来实现的

通常frame大小=page大小

Buffer = set of frames, File = set of pages

### Buffer的存储结构

Buffer是一个frame的列表，每个frame用于表示和存放一个磁盘块

### Buffer中Frame的查找

读磁盘块时: 根据page_id来确定在Buffer中是否已经存在frame

写磁盘块时: 要根据frame_id来快速找到文件中对应的page_id

`y = hash(page)`

要解决fram(page_id)中page_id的大空间问题, 用hash表

首先, 要维护Buffer中所有frame的维护信息, 如:

```C
struct BCB {
	BCB();
    int page_id;
    int frame_id;
    int count;
    int time;
    int dirty;
    BCB * next;
};
```

建立frame-page之间的索引

若用Hash Table, 需要建立2个

- BCB HTable
- int HTable

### Buffer Manager的基本功能

- FixPage(int page_id): 将对应page_id的page读入到buffer中。如果 buffer已满，则需要选择换出的frame。
- FixNewPage(): 在插入数据时申请一个新page并将其放在buffer中
- SelectVictim(): 选择换出的frame_id
- FindFrame(int page_id): 查找给定的page是否已经在某个frame中了
- SetDirty(int frame_id): 设定frame为dirty

### 数据库文件的一般组成

数据文件: 首块在Insert_Record时创建, 一般块号从0开始(FixnewPage)

系统目录文件: 首块在Create_Table时创建(FixNewPage)

*所有数据和元数据操作都唯一通过 Buffer Manager来请求page*

