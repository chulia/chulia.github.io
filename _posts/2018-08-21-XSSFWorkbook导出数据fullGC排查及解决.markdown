---
layout:     post
title:      "XSSFWorkbook导出数据fullGC排查及解决"
subtitle:   " \"解决一点, 知道一线\""
date:       2018-08-21 21:00:00
author:     "Chulia"
header-img: "img/post-bg-fullGC.jpg"
catalog: true
tags:
    - fullGC
    - JVM
---

# 1.出问题啦
这篇文章只会告诉你要使用SXSSFWorkbook导出数据，不要用XSSFWorkbook，跟fullGC没有半毛钱关系，写个大点的标题吸引下关注。为什么一定要用SXSSFWorkbook，简单来说就是少一个S的XSSFWorkbook要所有数据写入之后才会刷到磁盘，内存占用非常严重。

前段时间，发现商家系统时不时连续发生fullGC，有时候老年代直接占满，连续fullGC也不能释放内存。从哨兵上看起来是这样的，老年代释放不了，全部时间都在gc，机器不会响应其它请求。
![fullGC情况](https://ws1.sinaimg.cn/large/708e88fely1ft5vy77kz2j20r80x2456.jpg)
![虚拟机内存占用情况](https://ws1.sinaimg.cn/large/708e88fegy1ft5vl69dlkj20r80fwais.jpg)

通过jmap -histo:live  pid| less命令打印虚拟机内存中对象分布情况，排在第一第二位的对象是POI导出使用的两个对象，分别占用了985M和903M，确认是导出导致了fullGC。查看GC开始时间的日志，发现同时有两个商家在导出商品信息，导出的记录大概有7000行，每行40多个字段，初略估算导出任务内存占用在百兆级别，为什么会导致fullGC而释放不了内存呢？
![jmap打印内存分布情况](https://ws1.sinaimg.cn/large/708e88fegy1ft5w1owuqyj20r808z7fj.jpg)

再观察了一段时间，发现单次导出也会导致fullGC，GC及内存情况如下：
![单次导出GC情况](https://ws1.sinaimg.cn/large/708e88fegy1ft5w3vk3jej20r80ir43p.jpg)
![单次导出内存及CPU情况](https://ws1.sinaimg.cn/large/708e88fegy1ft5w4ea39wj20r808141w.jpg)

老年代在单次导出时依然有一个急剧上升的过程，并在多次连续fullGC之后恢复正常，CPU利用率明显上升。

随便google一下POI fullGC，网上很多说XSSFWorkbook占用内存十分严重的情况，因为XSSFWorkbook是要等所有的单元格写入之后，再一次性刷磁盘，这种方式的好处是可以一边写入一边读取，但是我们通常导出Excel仅仅是写入而不会同时读取，所以完全可以用SXSSFWorkbook代替，这种类型的workbook会设置一定的窗口大小，写入行数多余窗口大小则立即刷入磁盘以释放内存。

构造函数如下，可以直接从XSSFWorkbook构造，一行代码即可改造为SXSSFWorkbook对象：
```
public SXSSFWorkbook(XSSFWorkbook workbook, int rowAccessWindowSize) {
    this(workbook, rowAccessWindowSize, false);
}
```

# 2.性能对比

使用SXSSFWorkbook之后，尝试单次导出和连续多次导出Excel并观察GC情况和内存分配情况。

- **单次导出**

导出前老年代占用727M
![导出前老年代占用727M](https://ws1.sinaimg.cn/large/708e88fely1ft5w7l7t1ij20r80f4qc1.jpg)

导出后老年代增长到895M
![导出Excel后老年代占用895M](https://ws1.sinaimg.cn/large/708e88fely1ft5wgqljo9j20r80ez11x.jpg)

GC情况如下，没有出现fullGC
![导出的GC情况](https://ws1.sinaimg.cn/large/708e88fely1ft5w8cic58j20r80h8gp0.jpg)

CPU占用如下，使用率正常
![CPU利用率](https://ws1.sinaimg.cn/large/708e88fegy1ft5w8pso8ij20r80f241x.jpg)

- **连续多次导出**

连续进行了5次导出，基本每次老年代内存增长不到200M
![开始前内存占用](https://ws1.sinaimg.cn/large/708e88fegy1ft5waeb9s1j20r80iiwo4.jpg)
![第一次导出内存占用](https://ws1.sinaimg.cn/large/708e88fegy1ft5wb1f5vej20r80intip.jpg)
![多次导出内存占用](https://ws1.sinaimg.cn/large/708e88fegy1ft5wba4ohzj20r80iigvh.jpg)

# 3.总结
使用SXSSFWorkbook导出Excel表格能明显降低内存使用率，防止频繁导出导致机器fullGC，快使用它吧。

