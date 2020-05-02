---
layout:     post
title:      MapReduce论文阅读记录
date:       2018-01-30
author:     ChayCao
header-img: img/post-bg-2015.jpg 
catalog: true
tags:  MapReduce
---


本文为阅读MapReduce论文的记录，内容主要是论文的第三部分——实现。方便本人今后查看。

## 1. 运行概述

下图展示了 MapReduce 过程的整体情况

![MapReduce过程](https://chaycao-1302020836.cos.ap-shenzhen-fsi.myqcloud.com/chaycao%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2/2018/MapReduce%E8%BF%87%E7%A8%8B.png)

当用户程序执行 MapReduce 时，会依次发生以下动作（对应图中的标号）：

1. 用户程序中的 MapReduce 库将输入文件分成 *M* 个分片，每片有16M-64M（由用户决定），MapReduce 库还会将程序拷贝到集群机器上。

2. 集群中有一个 master，多个 worker。在拷贝程序过程中，其中 master 获得的程序是特殊的。master 将分配工作给 worker。现在有 *M* 个 map 任务和 *R* 个 reduce 任务需要被分配。master 会选择空闲的 worker，分配给 map 任务或 reduce 任务。一个 worker 只能担任一个 map 任务或 一个reduce 任务。

3. 被分配 map 任务的 worker 接下来会读取相应的输入块，将输入数据解析成 k-v 对，并将 k-v 对传给用户定义的 *Map* 函数。由 *Map* 函数生成的中间结果 k-v 对缓存在内存中。

4. 缓存的中间结果将定期地被写到本地磁盘上。分区函数（例如，hash(key) mod R）将中间结果分割成 *R* 个分区。然后，中间结果在本地磁盘的位置将传回给 master，接着 master 将负责把这些位置传给reduce worker。

5. 当 reduce worker 被 master 通知了中间结果的位置，它将通过 RPC 读取 map worker 本地磁盘上的中间结果。当完成读取工作，它会对中间结果进行排序，让具有相同 key 的对被分组在一块。

   排序工作的重要性在于：通常具有不同 key 的对会被分到同一个 *reduce* 任务中（与分区函数有关）。如果由于中间结果过大，无法装进内存进行排序，需要使用外部排序。

6. reduce worker 对已排序的数据进行遍历，每遇到一个不同的 key，便将 key 与对应的一系列 value 传给用户定义的 reduce 函数。其输出将作为该 reduce 分区的结果，追加到最终的输出文件中。

7. 当所有的 map 、reduce 任务完成， master 将唤醒用户程序。同时，用户程序中的mapreduce 调用得到返回。

在执行完成后，mapreduce 的输出将是 *R* 个文件（每个 reduce 任务一个）。通常，用户不需要将这 *R* 个文件合并成一个，可作为输入传给另一个 mapreduce 调用，或另一个分布式程序。

---

## 2. Master 数据结构

对于每个 map、reduce 任务，master 都会存储其状态（idle、in-progress、completed）和 non-idle的 worker 的信息。

master 在 map 任务到 reduce 任务之间传输中间结果的位置。对于每个完成的 map 任务，master 会存储其 *R* 个分区的位置和大小，并将该信息逐渐传输给处于 in-progress的reduce  worker。

---

## 3. 容错

### 3.1 worker 故障

master 定期地 ping 所有 worker。如果一个 worker 长时间没有响应， master 认为该 worker 已故障。该worker 上，以下任务，将被重置为 idle 状态，并将该任务重新分配到其他 worker 上

- 处于 completed 状态的 map 任务
- 处于 in-progress 状态的 map、reduce 任务
- 处于 in-progress 状态的 reduce 任务

completed 状态的 map 任务需要重新执行的原因：输出存储在故障机器的本地磁盘上，已经不可访问了。

completed 状态的 reduce 任务不需要重新执行的原因：输出存储在全局文件系统（GFS）上。

worker A 执行 map 任务，由于 A 故障了，接着由 worker B 执行该 map 任务。所有在运行 reduce 任务的 worker 都将被通知重新执行，而还没有从 worker A 读数据的 reduce 任务，将转为 worker B。

## 3.2 master 故障

master 定期检查点记录状态，当 master 任务死亡时，从最近的检查点状态开始执行。

## 3.3 本地性

网络带宽是我们的计算环境中相对稀缺的资源。 我们通过利用输入数据（由 GFS 管理）存储在组成我们集群的机器的本地磁盘上的来节省网络带宽。 GFS 将每个文件分成 64MB 块，并在不同的机器上存储每个块的多个副本（通常是3个副本）。 mapReduce master 将输入文件的位置信息考虑在内，并尝试在包含相应输入数据副本的机器上分配 map 任务。否则，它将尝试在该任务的输入数据副本附近（例如，在与包含数据的计算机处于同一网络交换机上的机器上）安排一个 map 任务。 在集群中大部分 worker 上运行大型MapReduce操作时，大多数输入数据都是本地读取的，不会消耗网络带宽。

## 3.4 任务粒度

从上文我们可以得知，map 阶段被划分成 *M* 个 task，reduce 阶段被划分成 *R* 个 task，*M* 和 *R* 一般会比集群中节点数大得多。每个节点运行多个 task 有利于动态的负载均衡，加速 worker 从失败中恢复。

在具体的实现中，*M* 和 *R* 的大小是有实际限制的，因为 master 至少要做 O(*M*＋*R*) 次的调度决策，并且需要保持O(*M* * *R*)个状态（使用的内存并不大，一条 M-R 记录需要 1 字节）。

通常情况下，*R* 的大小是由用户指定的，而对 *M* 的选择要保证每个任务的输入数据大小，即一个输入分片在 16MB～64MB 之间（数据本地性最优）。*R* 的大小是 worker 数量的一个较小的倍数。

## 3.5 备份任务

一种最常见的延长 mapreduce 运行总时间的原因是 “straggler”：一台机器花费异常时间完成最后一个 map 或 reduce 任务。“straggler” 出现的原因有很多，例如：磁盘有问题，读取速度下降；集群调度在机器上安排了其他任务，由于竞争CPU、内存、本地磁盘或网络带宽，导致其更慢地执行 mapreduce 代码。

解决“straggler”的机制：当 mapreduce 操作快完成时， master 会备份剩余的 in-progress 状态的任务。无论主程序或备份程序执行完成，该任务都会被标记为已完成。