---
layout: post
title: "漫谈HBase FilterList"
description: ""
category: 
tags: []
---

__初衷__

对数据库来说，满足业务多样化的查询方式非常重要。如果说有人设计了一个KV数据库，只提供了Get/Put/Scan这三种接口，估计要被用户吐槽到死，毕竟现实的业务场景并不简单。就以订单系统来说，查询给定用户最近三个月的历史订单，这里面的过滤条件就至少有2个：1. 查指定用户的订单；2. 订单必须是最近是三个月的。此外，这里的过滤条件还必须是用AND来连接的。如果通过Scan先把整个订单表信息加载到客户端，再按照条件过滤，这会给数据库系统造成极大压力。因此，在服务端实现一个数据过滤器是必须的。

除了上例查询需求，类似小明或小黄最近三个月的历史订单这样的查询需求，同样很常见。这两个查询需求，本质上前者是一个AND连接的多条件查询，后者是一个OR连接的多条件查询，现实场景中AND和OR混合连接的多条件查询需求也很多。因此，HBase设计了用AND或OR来连接过滤条件的FilterList。

例如，如下FilterList，表示将读到rowkey以abc为前缀且值为test的那些cell。

```java
fl = new FilterList(MUST_PASS_ALL,
                new PrefixFilter("abc"),
                new ValueFilter(EQUAL, new BinaryComparator("testA")));
```
实际上，FilterList内部的子filter也可以是一个FilterList。例如下面表示将读到那些rowkey以abc为前缀且值为testA或testB的f列cell列表。

```java
fl = new FilterList(MUST_PASS_ALL,
                new PrefixFilter("abc"),
                new FamilyFilter(EQUAL, new BinaryComparator("f")),
                new FilterList(MUST_PASS_ONE, 
                    new ValueFilter(EQUAL, new BinaryComparator("testA")),
                    new ValueFilter(EQUAL, new BinaryComparator("testB"))
                ))
```

因此，FilterList的结构其实是一颗多叉树。每一个叶子节点都是一个具体的Filter，例如PrefixFilter、ValueFilter等；所有的非叶子节点都是一个FilterList，各个子树对应各自的子filter逻辑。

__问题__

1. 各个Filter优化的问题，有的返回NEXT_COL,有的返回NEXT_ROW，有的返回SEEK_NEXT_HINT。
2. 用AND连接的FilterList，必须选最大的跳跃步数；
3. 用OR连接的FilterList，必须选最小的跳跃步数。
4. FilterList是Region级别内状态有效的。
5. Filter中返回的NEXT_ROW，其实是一个CF级别的NEXT_ROW。

__一些优化__

1. 优化PrefixFilter
2. 用MultipleColumnPrefixFilter来替换掉FilterList(OR, ColumnPrefixFilter, ColumnPrefixFilter, ...)。
