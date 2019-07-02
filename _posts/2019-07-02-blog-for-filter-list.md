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

例如，如下FilterList，表示将读到rowkey以abc为前缀且值为test的那些cell。

```java
fl = new FilterList(MUST_PASS_ALL,
                new PrefixFilter("abc"),
                new ValueFilter(EQUAL, new BinaryComparator(Bytes.toBytes("testA")))
);
```
实际上，FilterList内部的子filter也可以是一个FilterList。例如下面表示将读到那些rowkey以abc为前缀且值为testA或testB的f列cell列表。

```java
fl = new FilterList(MUST_PASS_ALL,
                new PrefixFilter("abc"),
                new FamilyFilter(EQUAL, new BinaryComparator(Bytes.toBytes("f"))),
                new FilterList(MUST_PASS_ONE, 
                    new ValueFilter(EQUAL, new BinaryComparator(Bytes.toBytes("testA"))),
                    new ValueFilter(EQUAL, new BinaryComparator(Bytes.toBytes("testB")))
                )
);
```

因此，FilterList的结构其实是一颗多叉树。每一个叶子节点都是一个具体的Filter，例如PrefixFilter、ValueFilter等；所有的非叶子节点都是一个FilterList，各个子树对应各自的子filter逻辑。对应的图示如下：

<img src="/images/filter-list-tree-structure.png" width="50%">

当然，HBase还提供了NOT语义的SkipFilter，例如用户想拿到那些rowkey以abc为前缀但value既不等于testA又不等于testB的f列的cell列表，可用如下FilterList来表示：

```java
fl = new FilterList(MUST_PASS_ALL,
               new PrefixFilter("abc"),
               new FamilyFilter(EQUAL, new BinaryComparator("f")),
               new SkipFilter(
                   new FilterList(MUST_PASS_ONE,
                        new ValueFilter(EQUAL, new BinaryComparator(Bytes.toBytes("testA"))),
                        new ValueFilter(EQUAL, new BinaryComparator(Bytes.toBytes("testB")))
                   )
               ));
```

__问题__

1.各个Filter优化的问题，有的返回NEXT_COL,有的返回NEXT_ROW，有的返回SEEK_NEXT_HINT。

2.用AND连接的FilterList，必须选最大的跳跃步数；

3.用OR连接的FilterList，必须选最小的跳跃步数。

4.FilterList是Region级别内状态有效的。

5.Filter中返回的NEXT_ROW，其实是一个CF级别的NEXT_ROW。

__一些优化__

_1.通过设置StartRow和StopRow替换PrefixFilter_

PrefixFilter是将rowkey前缀为指定字节串的数据都过滤出来并返回给用户。例如，如下scan会返回所有rowkey前缀为'def'的数据。

```java
Scan scan = new Scan();
 scan.setFilter(new PrefixFilter(Bytes.toBytes("def")));
```

注意，这个scan虽然能拿到预期的效果，但却并不高效。因为对于rowkey在区间(-oo, def)的数据，scan会一条条依次扫描一次，发现前缀不为def，就读下一行，直到找到第一个rowkey前缀为def的行为止。

这主要是因为目前HBase的PrefixFilter设计的相对简单粗暴，没有根据具体的Filter做过多的查询优化。这种问题其实很好解决，在scan中简单加一个startRow即可，RegionServer在发现scan设了StartRow，首先寻址定位到这个StartRow，然后从这个位置开始扫描数据，这样就跳过了大量的(-oo, def)的数据。代码如下：

```java
Scan scan = new Scan();
 scan.setStartRow(Bytes.toBytes("def"));
 scan.setFilter(new PrefixFilter(Bytes.toBytes("def")));
```

当然，更简单直接的方式，就是将PrefixFilter直接展开成扫描[def, deg)这个区间的数据，这样效率是最高的，代码如下：

```java
Scan scan = new Scan();
 scan.setStartRow(Bytes.toBytes("def"));
 scan.setStopRow(Bytes.toBytes("deg"));
```

_2.用MultipleColumnPrefixFilter来替换掉FilterList(OR, ColumnPrefixFilter, ColumnPrefixFilter, ...)_

在[HBASE-22448](https://issues.apache.org/jira/browse/HBASE-22448)，有用户提到，写一个如下的FilterList会显的特别慢：

```java
fl = new FilterList(MUST_PASS_ONE, 
    new ColumnPrefixFilter(Bytes.toBytes("aaa")),
    new ColumnPrefixFilter(Bytes.toBytes("bbb")),
    ...
    new ColumnPrefixFilter(Bytes.toBytes("zzz"))
)
```

这是因为采用FilterList(OR, ColumnPrefixFilter,...)的比较次数如下图所示，每个橙色的实心圆圈表示一次Cell的Compare操作。
<img src="/images/filterlist-with-column-prefix-filter.png" width="70%">


经过讨论和测试后，发现其实是可以用MultipleColumnPrefixFilter来替换上述FilterList(OR,ColumnPrefixFilter,...)的。代码如下：

```java
fl = new MultipleColumnPrefixFilter(byte[][] {
    Bytes.toBytes("aaa"),
    Bytes.toBytes("bbb"),
    ...,
    Bytes.toBytes("zzz")
 });
```
通过评估，我们发现Cell的比较次数如下图所示：
<img src="/images/filterlist-with-multiple-column-prefix-filter.png" width="70%">

二者对比发现，采用MultipleColumnPrefixFilter之后可以减少大量的比较次数。事实上，用[HBASE-22448](https://issues.apache.org/jira/browse/HBASE-22448)上的测试数据对比，发现优化后的性能快20倍：

这个案例带给我们的启发是，如果发现某些场景下采用通用的FilterList框架无法满足业务的性能需求，那么实际上可以尝试采用自定义Filter的方式来满足更高的性能需求。因为在自定义的Filter中，我们可以通过更少的比较次数来实现优化，而FilterList框架为了保证通用逻辑的正确性则无法实现。

_3.关于SingleColumnValueFilter的语义_

这个Filter的定义比较复杂，让人有点难以理解。举例来说：

```java
Scan scan = new Scan();
SingleColumnValueFilter scvf = new SingleColumnValueFilter(
    Bytes.toBytes("family"),
    Bytes.toBytes("qualifier"), 
    CompareOp.EQUAL, 
    Bytes.toBytes("value")
);
scan.setFilter(scvf);
```

这个例子表面上是将列簇为family、列为qualifier且值为value的cell返回给用户。但事实上，__对那些不包含family:qualifier这一列的行，也会被默认返回给用户__。如果用户不希望读取那些不包含family:qualifier的数据，需要设计如下scan：

```java
Scan scan = new Scan();
SingleColumnValueFilter scvf = new SingleColumnValueFilter(
    Bytes.toBytes("family"),
    Bytes.toBytes("qualifier"), 
    CompareOp.EQUAL, 
    Bytes.toBytes("value")
);
scvf.setFilterIfMissing(true); // 跳过不包含对应列的数据
scan.setFilter(scvf);
```

另外，当SingleColumnValueFilter设置filterIfMissing为true时，和其他Filter组合成FilterList时，可能导致返回结果不正确（参见[HBASE-20151](https://issues.apache.org/jira/browse/HBASE-20151)）。因为filterIfMissing设为true时，SingleColumnValueFilter必须要遍历一行数据中的每一个cell， 才能确定是否过滤，但在filterList中，如果其他的Filter返回NEXT_ROW会直接跳过某个列簇的数据，导致SingleColumnValueFilter无法遍历一行所有的cell，从而导致返回结果不符合预期。
对于这个问题，个人建议是：不要使用SingleColumnValueFilter和其他Filter组合成FilterList。尽量通过ValueFilter来替换掉SingleColumnValueFilter。


_4.关于PageFilter_

在[HBASE-21332](https://issues.apache.org/jira/browse/HBASE-21332)中，有一位用户说，有一个表，表里面有5个Region，分别为(-oo, 111), [111, 222), [222, 333), [333, 444), [444, +oo)。 表中这5个Region，每个Region都有超过10000行的数据。他发现通过如下scan扫描出来的数据居然超过了3000行：

```java
Scan scan = new Scan();
scan.withStartRow(Bytes.toBytes("111"));
scan.withStopRow(Bytes.toBytes("4444"));
scan.setFilter(new PageFilter(3000));
```

乍一看确实很诡异，因为PageFilter就是用来做数据分页功能的，应该要保证每一次扫描最多返回不超过3000行。但是需要注意的是，HBase里面Filter状态全部都是Region内有效的，也就是说，Scan一旦从一个Region切换到另一个Region之后， 之前那个Filter的内部状态就无效了，新Region内用的其实是一个全新的Filter。具体这个问题来说，就是PageFilter内部计数器从一个Region切换到另一个Region之后，计数器已经被清0。
因此，这个Scan扫描出来的数据将会是：

在[111,222)区间内扫描3000行数据，切换到下一个region [222, 333)。 

在[222,333)区间内扫描3000行数据，切换到下一个region [333, 444)。 

在[333,444)区间内扫描3000行数据，发现已经到达stopRow，终止。 

因此，最终将返回9000行数据。理论上说，这应该算是HBase的一个缺陷，PageFilter并没有实现全局的分页功能，因为Filter没有全局的状态。我个人认为，HBase也是考虑到了全局Filter的复杂性，所以暂时没有提供这样的实现。
当然如果想实现分页功能，可以不通过Filter，而直接通过limit来实现，代码如下：

```java
Scan scan = new Scan();
scan.withStartRow(Bytes.toBytes("111"));
scan.withStopRow(Bytes.toBytes("4444"));
scan.setLimit(1000);
```

所以，正常情况下对用户来说，PageFilter并没有太多存在的价值。