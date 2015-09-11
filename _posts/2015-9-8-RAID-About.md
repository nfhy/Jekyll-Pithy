---
layout: post
title: RAID小结
category: 杂七杂八
excerpt: Raid的小总结
---
**独立硬盘冗余阵列**（RAID, Redundant Array of Independent Disks），旧称**廉价磁盘冗余阵列**（Redundant Array of Inexpensive Disks），简称磁盘阵列。其基本思想就是把多个相对便宜的硬盘组合起来，成为一个硬盘阵列组，使性能达到甚至超过一个价格昂贵、容量巨大的硬盘。根据选择的版本不同，RAID比单颗硬盘有以下一个或多个方面的好处：增强数据集成度，增强容错功能，增加处理量或容量。另外，磁盘阵列对于电脑来说，看起来就像一个单独的硬盘或逻辑存储单元。分为RAID-0，RAID-1，RAID-1E，RAID-5，RAID-6，RAID-7，RAID-10，RAID-50，RAID-60。

```
以上摘自wiki中文
```

##RAID0
-------------
所有硬盘并联，读写并行处理，在所有RAID中具有最快的读写速度。任何一块一盘离线，所有数据都无法恢复。

##RAID1
-------------
所有硬盘互为镜像，理论上读取速度与硬盘数线性相关，写入速度与单块硬盘速度相当。只要有一块硬盘在线就可以正常工作。在所有RAID中安全性最高。硬盘越多，存储空间越浪费，在所有RAID中空间利用率最低。

##RAID2/RAID3/RAID4
-------------
较少实际应用，主要用于研究领域。原理与RAID5类似，性能不如RAID5.

##RAID5
-------------
![alt text](https://raw.githubusercontent.com/nfhy/medias/master/images/raidAbout/raid5.png "RAID5")

RAID5是一种储存性能、数据安全和存储成本兼顾的存储解决方案。每块硬盘有一个奇偶校验区，最少需要三块硬盘。对于阵列中的硬盘A、B、C，每个硬盘分三个区：A1、A2、A3、B1、B2、B3、C1、C2、C3,其中A3、B2、C1为奇偶校验区，A3=B3⊕C3，B2=A2⊕C2,C1=A1⊕B1，根据异或操作的特性，若a⊕b=c则a=b⊕c且b=a⊕c，当硬盘B故障时，只需计算B3=A3⊕C3、B1=A1⊕C1，B2=A2⊕C2即可完成数据恢复。可以理解为，当一块硬盘故障时，需要解n个一元一次异或方程进行数据恢复。

##RAID6
-------------
![alt text](https://raw.githubusercontent.com/nfhy/medias/master/images/raidAbout/raid6.png "RAID6")

与RAID5类似，但每块硬盘有两个独立的校验区，最少需要四块硬盘。两个校验区PQ，P与RAID5类似，进行异或运算；Q需要应用场理论（Galois Filed伽罗华域，尝试去查询这方面的资料，但是要求数学专业知识，本人数学知识有限，文档读起来非常吃力，没有理解）。与RAID5类似，可以理解为当有两块硬盘故障时，需要解n个二元一次异或方程组进行数据恢复。

##RAID10
-------------
先做镜像再做分区，只要每个分区中至少有一块硬盘在线就可以正常工作，可靠性高。

##RAID01
-------------
先做分区再做镜像，相当于多个RAID0阵列做镜像，只要一个RAID0中有一块硬盘故障，整个RAID0就会失效，相对于RAID10可靠性低。

##RAID50
-------------
先做RAID5再做RAID0，每个RAID5分区都允许至少一块硬盘故障，当有一个RAID5分区中有至少两块硬盘故障时，整个RAID50将会失效。

##RAID60
-------------
先做RAID6再做RAID0，每个RAID6分区都允许至少两块硬盘故障，当有一个RAID6分区中有至少三块硬盘故障时，整个RAID60将会失效。
关于RAID选择建议：

*	如果存储空间足够，选择RAID1+0

*	单独的数据库服务器其磁盘阵列最好选用RAID1+0

*	如选择RAID1+0空间不够，选择RAID 5

*	如果磁盘阵列包含的磁盘超过了8块，最好选择RAID 6

 

最后附上wiki上关于RAID的词条链接：

[英文wiki](https://en.wikipedia.org/wiki/Standard_RAID_levels)

[中文wiki](https://zh.wikipedia.org/wiki/RAID)

