---
layout: post
title: "Further GC optimization: Reading HFileBlock into offheap directly"
description: ""
category: 
tags: []
---

In [HBASE-21879](https://issues.apache.org/jira/browse/HBASE-21879), we redesigned the offheap read path: read the HFileBlock from HDFS to pooled offheap 
ByteBuffers directly, while before [HBASE-21879](https://issues.apache.org/jira/browse/HBASE-21879) we just read the HFileBlock to heap which would still lead 
to high GC pressure.

After few months of development and testing, all subtasks have been resovled now except the [HBASE-21946](https://issues.apache.org/jira/browse/HBASE-21946)
(It depends on [HDFS-14483](https://issues.apache.org/jira/browse/HDFS-14483) and our HDFS teams are working on this, we expect the HDFS-14483 to be included
in hadoop 2.9.3 and after that the HBASE-21946 will get resolved). we think the feature is stable enough now. 

We have designed 3 test cases to prove the performance improvment with HBASE-21879: 
1. Disabled BlockCache, which means the cacheHitRatio is 0%;
2. CacheHitRatio~65%;
3. CachehHitRatio~100%;

In our performance results[4], we can see that: the case#1 have an great performance improvement
(throughput increased about 17%, heap allocation decreased about 95%, Young generaion size decreased 
about 81.7%), that's because after HBASE-21879 all reads will allocate from pooled offheap bytebuffers 
and almost no heap allocation, while before HBASE-21879 the read path will create so many heap allocations.
On the other hand, from the testing results of case#2 and case#3 we can also see that: As the cacheHitRatio
increasing, the difference between before-HBASE-21879 and after-HBASE-21879 will decrease, when 
cacheHitRatio is 100%,  they almost have no much difference in both throughput and latency.

For more details please see the [document](/ppt/HBASE-21879-document.pdf).  Thanks Anoop/Ram/DuoZhang/Stack/GuanghaoZhang very much for your meticulous work (Suggession, discussion, patch reviewing, doc reviewing etc).