---
layout: post
title: HDFS 和 MapReduce
key: 20170109
tags: HDFS MapReduce
---

### HDFS 和 MapReduce

```
HDFS 负责存储
MapReduce 负责任务的分解与结果的汇总
```


----------


**HDFS 部分**

 - HDFS 中的角色：
**名称结点**：【NameNode】HDFS 中的管理者，负责文件系统的命名空间，集群配置信息，存储块信息的复制，保存每个块文件的 Metadata（文件信息保存在内存中）
**数据结点**：【DataNode】HDFS 中的文件存储基本单元，存储每个块的metadata ,周期性的报告所存块的信息给 NameNode（心跳包）
**客户端**：【】需要获取分布式文件系统的应用程序

**HDFS 数据存储【 数据的读取 / 数据的写入 】**

 - HDFS 文件读取过程:
![这里写图片描述](http://img.blog.csdn.net/20170109110550124?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hlbmcxNDgz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
```
Client 向NameNode 发起文件读取的请求。
NameNode 返回文件存储的 DataNode 的信息【元数据 Metadata 】。
Client 读取文件信息
```

HDFS 写入文件过程
![这里写图片描述](http://img.blog.csdn.net/20170109110619500?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hlbmcxNDgz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

```
client 向 NameNode 发起文件写入请求。
NameNode 根据文件大小和文件块配置情况，向client返回它所管理的 DataNamed 的信息【元数据 Metadata 】
Client 将文件划分为多个Block ，根据DataName 的地址信息，按顺序写入DataNode中
DataNode直接进行流水线复制，完成收相NameNode 更新元数据
```

关于分块的大小，可以通过设置 dfs.block.size的值进行分块，不设置，使用默认大小


----------


**MapReduce 模型部分**

Hadoop 通过自动分割将要执行的问题（程序）、拆解成Map(映射) 和 Reduce(化简)
     在自动分割后通过Map 程序将数据映射成为不相关的区块，分配（调度）给大量的计算机进行处理以达到分散运算的效果，再通过Reduce程序将结果汇合整合，输出开发者需要的结果。

     用户只需编写Map 和 Reduce 函数，这两个函数运行在 键-值对基础上的，
     数据切分，节点之间的通信调度都是由Hadoop框架本身负责。

MapReduce 软件实现：
     指定一个map函数，把键值对（key/value）映射成新的键值对（key/value ），形成一系列中间结果形式的键值对（key/value）,
     然后把他们传给Reduce(归约)函数，把具有相同中间形式key的value 合并在一起。
Map  和 Reduce 函数具有一定的关联性，其算法描述为：

```
Map(k,v) -> list(k1,v1)
Reduce(k1,list(v1)) ->list(v1)
```

在Map过程中将数据并行【把数据用映射函数规则分开】
在Reduce 把分开的数据用归约函数规则合在一起。
（Map 是一个分的过程，reduce 是一个合的过程）