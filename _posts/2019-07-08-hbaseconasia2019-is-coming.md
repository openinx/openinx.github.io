---
layout: post
title: "HBaseConAsia2019 盛会即将来袭"
description: ""
category: 
tags: []
---

第三届Apache HBaseConAsia 峰会将于7月20日在北京举行。作为Apache基金会旗下HBase社区的顶级用户峰会，HBaseCon大会是Apache HBase™官方从2012年开始发起和延续至今的技术会议。届时将有超20位来自亚洲一线互联网和大数据生态相关企业的技术专家和社区领袖亮相，带来HBase及大数据技术生态的最新洞察和行业实践。

Apache HBase是基于Apache Hadoop构建的一个高可用、高性能、多版本的分布式NoSQL数据库，是Google Big table的开源实现，通过在廉价PC Server上搭建起大规模结构化存储集群，提供海量数据高性能的随机读写能力。

伴随着移动互联网和物联网时代数据的爆炸性增长，HBase作为基础存储系统得到了快速发展与应用。阿里、Facebook、雅虎、小米、华为、腾讯、京东、滴滴、网易、360、快手等众多国内外顶级互联网公司先后成为HBase的重度用户，并深度参与项目优化与改进。目前，中国力量已成为HBase生态积极壮大的核心源动力，国内共有5位PMC成员和17位HBase Committer。其中小米公司累计培养2位PMC成员和9位HBase Committer。

### 精彩演讲，先睹为快

__开场演讲__  
__演讲嘉宾__：崔宝秋（小米集团副总裁、技术委员会主席）  

#### HBase现状与未来方向

__演讲主题__：HBase现状   
__内容简介__：具有里程碑意义的HBase2.0.0发布不久，HBase3.0.0已经呼之欲出。资深PMC张铎将与您一起讨论HBase2.x以及HBase3.x的现状和核心改进。分享将包括Procedure-V2、Assignment-V2、HBCK2、跨机房同步复制、异步客户端等核心主题，干货十足。   
__演讲嘉宾__：张铎（HBase PMC，小米存储团队负责人，小米开源委员会秘书长）   

__演讲主题__：HBase在云上的优势及技术趋势  
__内容简介__：与传统的物理数据中心相比，HBase在云上的优势是什么？构建云HBase的挑战是什么？未来的技术趋势是什么？这些都将是本次演讲要讨论的重点。除此之外，还将包括以下内容：  
1. 为何HBase架构天然适用云环境  
2. HDFS构建在云盘上的挑战  
3. HBase如何充分利用不同的云存储介质  
4. HBase Serverless的实现和价值  
5. 借助云端虚拟机的拓展能力，HBase还能可以做些什么？  
6. 云端HBase如何从GPU，FPGA等新硬件中获益？  

__演讲嘉宾__：沈春辉（HBase PMC、阿里巴巴资深技术专家）  
__演讲主题__：HBase BucketCache with Persistency Memory  
__内容简介__：Intel的DCPMM (Date Centre Persistent Memory devices) 是一种新型的非易失内存技术。该设备支持更大内存容量的同时，还能保证数据的持久性。英特尔的资深工程师团队将分享如何将HBase BucketCache构建在这些大容量的非易失内存上，同时将给出具体的性能对比数据。  
__演讲嘉宾__：Anoop Jam John （HBase PMC）、Ramkrishna S Vasudevan (HBase PMC)、Xu Kai ( Intel 工程师）  


#### HBase2.x内核改进

__演讲主题__：Further GC optimization: Reading HFileBlock into offheap directly  
__内容简介__：HBase2.0.0版本已经将最核心的读写路径做了offheap化，极大的降低了GC对读写请求延迟的影响。但在性能测试中，我们发现当cache命中率不高时，读请求的P999延迟几乎和GC的Stop The World耗时一致。本次分享，将讲述Intel工程师和小米工程师如何一起携手展开一场极致的GC优化之旅。  
__演讲嘉宾__：Anoop Jam John （HBase PMC），胡争（小米HBase工程师，HBase Committer）  

__演讲主题__：HBCK2: Concepts, trends and recipes for fixing issues within HBase 2  
__内容简介__：面向开发和运维介绍HBCK2的概念、细节和最佳实践。  
__演讲嘉宾__：Wellington Chevreuil（Cloudera HBase工程师，HBase Committer）  
 
__演讲主题__：基于Procedure V2实现的WAL Splitting和ACL功能。  
__内容简介__：HBase Procedure-V2提供了统一的方式来协调在机器间交互的事务等功能，本次分享将介绍小米基于Procedure-V2的split WAL和ACL的设计。  
__演讲嘉宾__：梅祎（小米HBase工程师，HBase Committer） 

#### HBase业内实践

__演讲主题__：HBase在Pinterest的最新进展  
__内容简介__：关于HBase在事务、流式消息、二级索引、SQL查询方面的增强和优化。  
1.通过Apache Omid实现HBase对ACID事务支持  
2.自研实现的Sparrow对比Omid在事务上提升2倍吞吐，降低20%延迟  
3.Argus和Kafka结合使用，提供WAL通知机制  
4.基于开源Lily实现Ixia，用于实时构建HBase二级索引  
5.Ixia同时整合了Muse，实现类SQL查询              
__演讲嘉宾__：Lianghong Xu（Pinterest工程师）  

__演讲主题__：HBase在腾讯的应用和改进  
__内容简介__：在腾讯内部，包括社交、支付、微信在内的众多业务都有依赖HBase。来自腾讯的HBase Committer程广旭将分享HBase在腾讯的应用场景，以及功能和性能方面的改进。  
__演讲嘉宾__：程广旭（腾讯HBase工程师，HBase Committer）  

__演讲主题__: HBase在快手万亿级短视频存储和百亿级用户画像分析场景中的应用  
__内容简介__: 快手每天都会产生大量的短视频数据以及用户画像数据. 本次演讲将为大家介绍快手如何基于HBase和HDFS来存储这些海量的短视频数据, 以及如何存储和分析海量的用户画像数据.  
__演讲嘉宾__: 徐明(快手大数据架构研发工程师)  

更多精彩演讲，请参考：https://open.mi.com/conference/hbasecon-asia-2019

### 报名参会

扫描下面二维码，立即免费报名，去大会现场和大咖一起畅聊HBase。此外，报名参会者将现场收获HBaseCon Asia 2019纪念T恤一件。



T恤样品



### 网络直播

本次大会分为3个分会场：  
__会场1__: 介绍HBase内核设计和改进，小米直播ID为：3004163  
__会场2__: 介绍HBase生态产品和解决方案，小米直播ID为：2329658282  
__会场3__: 介绍HBase在各公司应用场景，小米直播ID为：2312959070  

小米直播的正确使用姿势：  
1.手机下载【小米直播APP】；  
2.在小米直播APP中搜索上述分会场的直播ID，例如：3004163  
3.进入直播房间，观看对应分会场直播。  
