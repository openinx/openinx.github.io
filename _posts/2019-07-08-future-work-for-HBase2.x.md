---
layout: post
title: "社区HBase未来值得做的一些工作"
description: ""
category: 
tags: []
---

HBase2.0.0版本自2018年4月30日正式发布起， 到现在已经过了接近15个月。现在的状态是HBase2.0.x已经EOL了，后面不会再发新的Release版本了，HBase2.1已经发布到HBase2.1.6了，预计将来也不会维护太长的时间。今后的HBase2.x的稳定版本将会是HBase2.2.x和HBase2.3.x，尤其是HBase2.3.x，可能成为未来真正意义上经过大厂考验的版本。

具有里程碑意义的HBase2.x相比HBase1.x，实现了众多有吸引力的功能和性能优化：

1.ProcedureV2和AssignmentV2的引入，能通过框架的方式保证分布式任务流的原子性。这在HBase1.x上曾经是一个非常令人困惑的麻烦。举个简单的例子，在建表流程中，会分成几步：a. 在zk上加个znode；b. 在文件系统上新增表的目录；c. 生成Assign的任务，并分发到具体的RegionServer，让其执行online region的操作。在HBase1.x中任何一步异常了，都可能造成各状态不一致的问题发生，极端情况下可能需要通过类似HBCK这样的工具来进行修复。但在HBase2.x中，已经通过框架来解决了这个问题。需要人操行的地方少了，那代码需要操心的地方就很多了，由于各个任务流都采用Procedure V2进行重写，中间难免会一些bug，所以，后续将这块功能变得更加稳定，是一个非常优先级非常高的工作。

2.In-Memory Compaction功能。可以说这是一个性能优化进步很大的功能，在我们大数据集(100亿行数据)的测试情况下，写入操作的P999延迟可以严格控制在令人惊讶的50ms以内，而且延迟非常稳定。但是社区考虑到其功能的稳定性，暂时没有把它设为默认的Memstore，也就是说默认的Mmestore仍然是延迟控制较差的ConcurrentSkipListMap的DefaultMemstore。在我们的测试环境，确实也发现了一些很难定位的BUG，例如HBASE-XXXX。因此，将这个功能弄稳定也是优先级特别高的一个事情。

3.MOB这个功能很好，可以通过同一个API处理各个Value大小的Cell，而且原子语义等跟正常的Cell完全一致。但当前的方案仍然有一些缺陷，例如MOB的大Value compaction现在是由Master端来负责跑的，这种Compaction的数据量会是一个巨大的量，单点来做会非常耗时，毕竟单机网卡流量和CPU资源都非常有限。理想的方案是分担到各个RegionServer去做，但目前还没有实现，这也就是一个必须要做的工作。

4.在读写路径上引入Offheap后，有时候目前会碰到一些字节错乱的bug。这种bug只在特定条件下才触发，不易复现极大地增加了定位问题的难度，而且预计未来可能会碰到一些Memory Leak的问题，毕竟自己管理内存之后，就有这种可能。所以，这块也需要考虑。

5.在HBase2.x中，除了Flush和Snapshot两个流程之外，其他的管理流程全部都Procedure-V2化。所以将Flush和Snapshot搞成Procedure-V2的写法，也是一个非常必要的工作。毕竟现在既有ProcedureV1的写法，又有Procedure-V2的写法，让代码显得冗余，搞定了Flush和Snapshot之后，ProcedureV1的框架就可以完全清楚掉了。 

6.Replicaiton现在仍然是走ZK的，开启串行复制之后，每个Region都会在zk上维护一个znode。这在大集群上可能会对ZK造成很大的压力。所以Replication从存ZK改成存Meta，也会是一个很必要的工作。之前我尝试去做这方面的研发，后面发现一个比较重要的问题，就是启动时Master和RegionServer死锁的问题，要解决这个问题可能需要对Master启动流程做一些调整，会有一些额外的工作。当时有其他优先级更高的事情，就干其他事情去了，从长远来看，改成走Meta是必须的。

7. CCSMap插件化，搞成一个独立的依赖。
