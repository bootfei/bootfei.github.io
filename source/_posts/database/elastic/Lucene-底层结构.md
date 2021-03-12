---
title: Lucene-底层结构
date: 2021-03-12 20:04:10
tags:
---



# 倒排索引

 Elasticsearch最强大的就是为每个字段提供了倒排索引，当查询的时候不用担心没有索引可以利用，什么是倒排索引，举个简单例子：

| 文档ID | 年龄 | 性别 |
| ------ | ---- | ---- |
| 1      | 25   | 女   |
| 2      | 32   | 女   |
| 3      | 25   | 男   |

 每一行是一个文档（document），每个document都有一个文档ID。那么给这些文档建立的倒排索引就是：

年龄的索引：

| 年龄 | 文档ID列表：posting list |
| ---- | ------------------------ |
| 25   | [1,3]                    |
| 32   | [2]                      |

性别的索引：

| 性别 | 文档ID列表：posting list |
| ---- | ------------------------ |
| 男   | [3]                      |
| 女   | [1,2]                    |

可以看到，倒排索引是针对每个字段的，每个字段都有自己的倒排索引，25、32这些叫做term，[1,3]这种叫做posting list，

它是一个int的数组，存储了所有符合某个term的文档id，这时候我们想找出年龄为25的人，就会很快速。

但是这里只有两个term，如果有成百上千个term呢，那找出某个term就会很慢，因为term还没有排序，解决这个问题需要了解两个概念：Term Dictionary 和 Term Index。<!--这就是ES比Mysql的原因-->


# 索引查询

## Term Dictionary

Elasticsearch为了能快速找到某个term，将所有的term进行了排序，然后二分法查找term，类似于上学时候老师教我们的翻新华字典的方式，所以这叫做Term Dictionary，这种查询方式其实和传统关系型数据库的B-Tree的方式很相似，所以这并不是Elasticsearch快的原因。

| Term（已经排过序了a -> z） | Posting List |
| -------------------------- | ------------ |
| ab                         | [....]       |
| abc                        | [....]       |
| b                          | [....]       |

查询abc的时候，使用二分查找，hi = term.size(), lo = 0;

可以看到，整个Term列 = Term Dictionary

## Term Index

如果说Term Dictionary是直接去二分法翻字典，那么Term Index就是字典的目录页。

假设我们的term如果全是英文单词，那么Term Index就是26个字母表，但是通常term未必都是英文，而可以是任意的byte数组。因为就算26个英文字符也不一定都有对应的term，比如：a开头的term只有一个，c开头的term有一百万个，x开头的term一个也没有，这样查询到c的时候又会很慢了。所以通常情况下Term Index 是包含term的一些前缀的一棵树，例如这样的一个Term Index：

<img src="https://img-blog.csdn.net/20181010140158769?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2Mjg5Mzc3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70" alt="img" style="zoom:80%;" />

这样的情况下通过Term Index据可以快速定位到某个offset(分支的开端)，然后以此位置向下查找，再加上[FST]()（Finite-State Transducer，Lucene4.0开始使用该算法来查找Term 在Dictionary中的位置）的[压缩技术]()，将Term Index 缓存到内存中，通过Term Index 找到对应的Term Dictionary的 block，然后再去磁盘直接找到term，减少磁盘的随机读写次数，大大的提升查询效率。（FST在下个章节单独介绍）

# Posting List压缩技术 Roaring Bitmap

谈到roaring bitmaps就要先了解bitset 或者bitmap，Bitset是一种的数据结构，对应posting list如果是：[2,3,5,7,9] 那么对于对应的bitset就是：[0,1,1,0,1,0,1,0,1,0]，用0和1来表示该位置的数值的有无，这种做法就是一个byte可以代表8个文档，当大数据量时，仍然会消耗很多内存，所以直接将bitset结构存入内存不太理想。

Elasticsearch不仅压缩了Term Index，还对posting list 进行了压缩，posting list虽然只存储了文档id，但是当文档id很大的时候，比PB级的数据，Elasticsearch对posting list的压缩做了两件事：排序和大数变小数，引用一张被引用无数次的图：

![Alt text](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9yYXcuZ2l0aHVidXNlcmNvbnRlbnQuY29tL05ld2F5NjY1NS9uZXdheTY2NTUuZ2l0aHViLmNvbS9tYXN0ZXIvaW1hZ2VzL2VsYXN0aWNzZWFyY2gtc3R1ZHkvZnJhbWVPZlJlZmVyZW5jZS5wbmc?x-oss-process=image/format,png)

简单解读一下这种压缩技巧：

step1：在对posting list进行压缩时进行了正序排序。
step2：通过增量将73后面的大数变成小数存储增量值。
step3:  转换成二进制，取占最大位的数，227占8位，前三个占八位，30占五位，后三个数每个占五位。
从第三步可以看出，这种压缩方式仍然不够高效，所以Lucene使用的数据结构叫做Roaring Bitmap，其压缩的原理可以理解为，与其保存100个0，占用100个bit，还不如保存0一次，然后声明这个0有100个，它以两个自己可以表示的最大数65535为界，将posting list分块，比如第一块是0-65535，第二块是65536-131071，如图:

![Alt text](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9yYXcuZ2l0aHVidXNlcmNvbnRlbnQuY29tL05ld2F5NjY1NS9uZXdheTY2NTUuZ2l0aHViLmNvbS9tYXN0ZXIvaW1hZ2VzL2VsYXN0aWNzZWFyY2gtc3R1ZHkvUm9hcmluZ2JpdG1hcHMucG5n?x-oss-process=image/format,png)

压缩技巧解读：

step1：从小到大进行排序。
step2：将大数除以65536，用除得的结果和余数来表示这个大数。
step3:：以65535为界进行分块。
注意：如果一块超过了4096 个值，直接用bitset存，2个字节就用个简单的数组存放好了，比如short[]，修正一下：1KB=1024B=1024byte=8192bit，每个值一个bytes，4096*2bytes = 8192bytes，刚好达到每一个block的界限 4096 = 65536 / 2 / 8