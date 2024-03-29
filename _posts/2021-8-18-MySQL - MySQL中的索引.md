---
layout: post
title:  'MySQL - MySQL中的索引'
date:  2021-8-18
tags: [mysql]
categories: [数据库,mysql]
---

## 1. 概述

**定义**：**索引是存储引擎用于快速找到记录的一种数据结构**。举例说明：如果查找一本书中的某个特定主题，一般会先看书的目录（类似索引），找到对应页面。在MySQL，存储引擎采用类似的方法使用索引，高效获取查找的数据。

**索引的分类**

- 从存储结构上来划分
  - Btree 索引（B+tree，B-tree)
  - 哈希索引
  - full-index 全文索引

- 从应用层次上来划分
  - 普通索引：即一个索引只包含单个列，一个表可以有多个单列索引。
  - 唯一索引：索引列的值必须唯一，但允许有空值。
  - 复合索引：一个索引包含多个列。

- 从表记录的排列顺序和索引的排列顺序是否一致来划分
  - 聚簇索引（主键）：表记录的排列顺序和索引的排列顺序一致。
  - 非聚集索引：表记录的排列顺序和索引的排列顺序不一致。



## 2.  InnoDB中的索引方案

### 2.2 聚簇索引

1. 使用记录主键值的大小进行记录和页的排序，这包括三个方面的含义：

   1. 页内的记录是按照主键的大小顺序排成一个单向链表。
   2. 各个存放用户记录的页也是根据页中用户记录的主键大小顺序排成一个双向链表。
   3. 存放目录项记录的页分为不同的层次，在同一层次中的页也是根据页中目录项记录的主键大小顺序排成一个双向链表。

2. B+树的叶子节点存储的是完整的用户记录。

   所谓完整的用户记录，就是指这个记录中存储了所有列的值。

         我们把具有这两种特性的B+树称为聚簇索引，所有完整的用户记录都存放在这个聚簇索引的叶子节点处。这种聚簇索引并不需要我们在MySQL语句中显式的使用INDEX语句去创建，InnoDB存储引擎会**自动的为我们创建聚簇索引**。另外有趣的一点是，在InnoDB存储引擎中，聚簇索引就是数据的存储方式（所有的用户记录都存储在了叶子节点），也就是所谓的索引即数据，数据即索引。
       
          ps：聚簇索引默认情况下为准建，如果主键不存在，则使用自动生成隐藏的主键。

### 2.2 二级索引

        上边介绍的聚簇索引只能在搜索条件是主键值时才能发挥作用，因为B+树中的数据都是按照主键进行排序的。那如果我们想以别的列作为搜索条件该咋办呢？这时就可以多建几棵B+树，不同的B+树中的数据采用不同的排序规则，这就是二级索引。

比方说我们用c2列的大小作为数据页、页中记录的排序规则，再建一棵B+树，效果如下图所示：

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/picSnipaste_2021-08-18_18-31-09.jpg)

这个B+树与上边介绍的聚簇索引有几处不同：

1. 使用记录c2列的大小进行记录和页的排序，这包括三个方面的含义：
   1. 页内的记录是按照c2列的大小顺序排成一个单向链表。
   2. 各个存放用户记录的页也是根据页中记录的c2列大小顺序排成一个双向链表。
   3. 存放目录项记录的页分为不同的层次，在同一层次中的页也是根据页中目录项记录的c2列大小顺序排成一个双向链表。
2. B+树的叶子节点存储的并不是完整的用户记录，而只是c2列+主键这两个列的值。
3. 目录项记录中不再是主键+页号的搭配，而变成了c2列+页号的搭配。

使用二级索引与聚簇索引时的区别：由于聚簇索引即数据，所以在使用时可以直接找到数据信息，而二级索引由于只包含索引值（上图的c2）和聚簇索引（主键）信息，所以根据二级索引查找到信息时，必须再根据主键值去聚簇索引中再查找一遍完整的用户记录，当然如果只需要返回索引包含的字段信息，是可以直接返回的（例如，select c2）。

### 2.3 联合索引

我们也可以同时以多个列的大小作为排序规则，也就是同时为多个列建立索引，比方说我们想让B+树按照c2和c3列的大小进行排序，这个包含两层含义：

1. 先把各个记录和页按照c2列进行排序。
2. 在记录的c2列相同的情况下，采用c3列进行排序

**以c2和c3列的大小为排序规则建立的B+树称为联合索引，本质上也是一个二级索引**。它的意思与分别为c2和c3列分别建立索引的表述是不同的，不同点如下：

- 建立联合索引只会建立一样的1棵B+树。
- 为c2和c3列分别建立索引会分别以c2和c3列的大小为排序规则建立2棵B+树。

## 3. MyISAM中的索引方案

         我们知道InnoDB中索引即数据，也就是聚簇索引的那棵B+树的叶子节点中已经把所有完整的用户记录都包含了，而MyISAM的索引方案虽然也使用树形结构，但是却将索引和数据分开存储：

- 将表中的记录按照记录的插入顺序单独存储在一个文件中，称之为数据文件。这个文件并不划分为若干个数据页，有多少记录就往这个文件中塞多少记录就成了。

- 使用MyISAM存储引擎的表会把索引信息另外存储到一个称为索引文件的另一个文件中。MyISAM会单独为表的主键创建一个索引，只不过在索引的叶子节点中存储的不是完整的用户记录，而是主键值 + 行号的组合。也就是先通过索引找到对应的行号，再通过行号去找对应的记录！

  这一点和InnoDB是完全不相同的，在InnoDB存储引擎中，我们只需要根据主键值对聚簇索引进行一次查找就能找到对应的记录，而在MyISAM中却需要进行一次回表操作，意味着MyISAM中建立的索引相当于全部都是二级索引！

- 如果有需要的话，我们也可以对其它的列分别建立索引或者建立联合索引，原理和InnoDB中的索引差不多，不过在叶子节点处存储的是相应的列 + 行号。这些索引也全部都是二级索引。



## 4. MySql中的索引的使用条件

- **全值匹配**：如果我们的搜索条件中的列和索引列一致的话，这种情况就称为全值匹配
- **匹配左边的列**：在我们的搜索语句中也可以不用包含全部联合索引中的列，只包含左边的就行。例如索引字段为`c1+c2+c3`，我们使用`c1`或`c1+c2`都可以使用到索引，但是`c2+c3`不能使用到索引
- **匹配列前缀**：索引`c1`，类型为字符串,当我们使用`like 'a%'`时可以使用到索引，但是匹配的中间，或者后面则不能,例如`like '%a%'`、`like %a`
- **匹配范围值**：所有记录都是按照索引列的值从小到大的顺序排好序的，所以这极大的方便我们查找索引列的值在某个范围内的记录。例如，`where 'A'<c1 and c1<'C'`

还有更多的使用情况就不一一列举，都大同小异


## 5. 索引的访问方式

在MySql中执行查询语句时，查询的执行方式大致分为两种：

- 使用全表扫描进行查询

  这种执行方式很好理解，就是把表的每一行记录都扫一遍嘛，把符合搜索条件的记录加入到结果集就完了。不管是啥查询都可以使用这种方式执行。

- 使用索引进行查询

  因为直接使用全表扫描的方式执行查询要遍历好多记录，所以代价可能太大了。如果查询语句中的搜索条件可以使用到某个索引，那直接使用索引来执行查询可能会加快查询执行的时间。

### 5.1 const

有的时候我们可以通过主键列来定位一条记录，比方说这个查询：`SELECT * FROM single_table WHERE id = 1438;`

<img src="https://blog-1253533258.cos.ap-shanghai.myqcloud.com/picSnipaste_2021-08-18_19-39-55.jpg" style="zoom:50%;" />

类似的，我们根据唯一二级索引列来定位一条记录，比如下边这个查询：`SELECT * FROM single_table WHERE key2 = 3841;`

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/picSnipaste_2021-08-18_19-42-25.jpg)

对于唯一二级索引来说，查询该列为NULL值的情况比较特殊，比如这样：`SELECT * FROM single_table WHERE key2 IS NULL;`因为唯一二级索引列并不限制 NULL 值的数量，所以上述语句可能访问到多条记录，也就是说上边这个语句不可以使用const访问方法来执行

### 5.2 ref

有时候我们对某个普通的二级索引列与常数进行等值比较，比如这样：`SELECT * FROM single_table WHERE key1 = 'abc';`

由于普通二级索引并不限制索引列值的唯一性，所以可能找到多条对应的记录，也就是说使用二级索引来执行查询的代价取决于等值匹配到的二级索引记录条数。如果匹配的记录较少，则回表的代价还是比较低的，所以MySQL可能选择使用索引而不是全表扫描的方式来执行查询。

这种搜索条件为二级索引列与常数等值比较，采用二级索引来执行查询的访问方法称为：ref。

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/pic20210818194603.png)

特殊情况：

- 二级索引列值为NULL的情况

  不论是普通的二级索引，还是唯一二级索引，它们的索引列对包含NULL值的数量并不限制，所以我们采用key IS NULL这种形式的搜索条件最多只能使用ref的访问方法，而不是const的访问方法。

- 如果最左边的连续索引列并不全部是等值比较的话，它的访问方法就不能称为ref了，比方说这样

  `SELECT * FROM single_table WHERE key_part1 = 'god like' AND key_part2 > 'legendary';`

### 5.3 ref_or_null

不仅想找出某个二级索引列的值等于某个常数的记录，还想把该列的值为NULL的记录也找出来，就像下边这个查询：`SELECT * FROM single_demo WHERE key1 = 'abc' OR key1 IS NULL;`

当使用二级索引而不是全表扫描的方式执行该查询时，这种类型的查询使用的访问方法就称为ref_or_null

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/pic20210818195048.png)

### 5.4 range

之前介绍的几种访问方法都是在对索引列与某一个常数进行等值比较的时候才可能使用到，但是有时候我们面对的搜索条件更复杂，比如下边这个查询：`SELECT * FROM single_table WHERE key2 IN (1438, 6328) OR (key2 >= 38 AND key2 <= 79);`

这种利用索引进行范围匹配的访问方法称之为：range

其实对于B+树索引来说，只要索引列和常数使用=、<=>、IN、NOT IN、IS NULL、IS NOT NULL、>、<、>=、<=、BETWEEN、!=（不等于也可以写成<>）或者LIKE操作符连接起来，就可以产生一个所谓的区间。

### 5.5 index

`SELECT key_part1, key_part2, key_part3 FROM single_table WHERE key_part2 = 'abc';`

该索引为` key_part1, key_part2, key_part3 `

由于key_part2并不是联合索引idx_key_part最左索引列，所以我们无法使用ref或者range访问方法来执行这个语句。但是这个查询符合下边这两个条件：

- 它的查询列表只有3个列：key_part1, key_part2, key_part3，而索引idx_key_part又包含这三个列。
- 搜索条件中只有key_part2列。这个列也包含在索引idx_key_part中。

也就是说我们可以直接通过遍历idx_key_part索引的叶子节点的记录来比较key_part2 = 'abc'这个条件是否成立，把匹配成功的二级索引记录的key_part1, key_part2, key_part3列的值直接加到结果集中就行了。由于二级索引记录比聚簇索记录小的多（聚簇索引记录要存储所有用户定义的列以及所谓的隐藏列，而二级索引记录只需要存放索引列和主键），而且这个过程也不用进行回表操作，所以直接遍历二级索引比直接遍历聚簇索引的成本要小很多，把这种采用遍历二级索引记录的执行方式称之为：index。

### 5.6 all

最直接的查询执行方式就是全表扫描，对于InnoDB表来说也就是直接扫描聚簇索引，把这种使用全表扫描执行查询的方式称之为：all。

ps:以上所有访问方式速度大部分情况下是依次递减的

## 6. 总结

以上是最近学习MySql索引相关内容后的一个简单的总结

*参考*

- 《MySql是怎么运行的》
- [MySQL：索引详解](https://juejin.cn/post/6844904134261342216)