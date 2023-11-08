---
title: ABD-Study03
date: 2023-10-15 21:11:36
tags:
- ABD
categories:
- ABD-USTC

---

# Data Storage

## 存储器结构

由上而下分别为:

main memory

extended memory

solid state disks, disk cache

magnetic disks/disk arrays

magnetic tapes, optical disks

其所用时间逐步增加

### 典型磁盘结构

![](/figures/03-01.png)

其中, 盘片platter, 盘面surface, 磁头R/W head, 磁道track, 柱面cylinder, 扇区sector.

## 磁盘块存取时间

###  块

- OS/DBMS进行磁盘数据存取的最小逻辑单元, **块由若干连续扇区构成**
- 块是**DBMS**中数据存取的最小单元
- 扇区是**磁盘**中数据存储的最小单元

### 读块时间

$$
Time = 寻道时间S + 旋转延迟R + 传输时间T + 其他延迟
$$

- 寻道时间: 磁头定位到对应柱面所花费的时间, 与横跨柱面个数正相关
- 平均寻道时间: 假设随机跨柱面的情况下,寻道时间的期望值,一般为10ms-40ms之间
- 旋转延迟: 磁盘传动到块的第一个扇区到达磁头所需的时间, 假设磁盘转速为x RPM, 那么对应的平均旋转延迟为$\frac{60000}{2x}$(ms)
- 平均时间维旋转1/2周的时间
- 传输延迟:定位到对应扇区后,读取扇区内数据所需要的延时(以及转过块间间隙所需要的时间).假设单个磁道可以存储M字节的数据,且每t时间转一周,那么假设我们需要读取m大小的数据,所需要的传输时间为$\frac{mt}{M}$

### 如何读取下一块?

1. 下一块在同一柱面上: time = 旋转延迟+传输延迟+其他
2. 不在同一柱面: 寻道+旋转+传输+其他

### 写块

- 与读块类似
- 如果需要校验块是否正确写入, 则需要加上一次旋转时间和一次块传输时间.

### 块修改

将块读入主存->在主存中完成修改->将块重新写入磁盘

### 块地址

物理设备号, 柱面号, 盘面号, 扇区号

## 磁盘例子: Megatron747

以megatron747为例子来对存取时间等进行分析

### 参数

- 3.5 inch
- 3840RPM
- 8 surfaces
- 8192 tracks/surface
- 256 sevtors/track
- 512 bytes/sector
- 大小: 8*8192*256*512 = 8GB

### 计算



