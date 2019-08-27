---
title: explain命令的使用
date: 2019-08-27 20:13:05
tags:
- mysql
- explain
- index
- 索引
categories:
- mysql
---

# 准备条件

创建表

- user_info：name字段有索引，sname字段有唯一索引
- order_info：user_id、product_name、productor三个字段的联合索引

```
CREATE TABLE `user_info` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `name` varchar(50) NOT NULL DEFAULT '',
  `sname` varchar(50) NOT NULL DEFAULT '',
  `age` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `name_index` (`name`),
  UNIQUE KEY `sname_index` (`sname`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8

INSERT INTO user_info (name, sname, age) VALUES ('xys', 'xys', 20);
INSERT INTO user_info (name, sname, age) VALUES ('a', 'a', 21);
INSERT INTO user_info (name, sname, age) VALUES ('b', 'b', 23);
INSERT INTO user_info (name, sname, age) VALUES ('c', 'c', 50);
INSERT INTO user_info (name, sname, age) VALUES ('d', 'd', 15);
INSERT INTO user_info (name, sname, age) VALUES ('e', 'e', 20);
INSERT INTO user_info (name, sname, age) VALUES ('f', 'f', 21);
INSERT INTO user_info (name, sname, age) VALUES ('g', 'g', 23);
INSERT INTO user_info (name, sname, age) VALUES ('h', 'h', 50);
INSERT INTO user_info (name, sname, age) VALUES ('i', 'i', 15);

CREATE TABLE `order_info` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `user_id` bigint(20) DEFAULT NULL,
  `product_name` varchar(50) NOT NULL DEFAULT '',
  `productor` varchar(30) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `user_product_detail_index` (`user_id`,`product_name`,`productor`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8

INSERT INTO order_info (user_id, product_name, productor) VALUES (1, 'p1', 'WHH');
INSERT INTO order_info (user_id, product_name, productor) VALUES (1, 'p2', 'WL');
INSERT INTO order_info (user_id, product_name, productor) VALUES (1, 'p1', 'DX');
INSERT INTO order_info (user_id, product_name, productor) VALUES (2, 'p1', 'WHH');
INSERT INTO order_info (user_id, product_name, productor) VALUES (3, 'p3', 'MA');
INSERT INTO order_info (user_id, product_name, productor) VALUES (4, 'p1', 'WHH');
INSERT INTO order_info (user_id, product_name, productor) VALUES (2, 'p5', 'WL');
INSERT INTO order_info (user_id, product_name, productor) VALUES (6, 'p1', 'WHH');
INSERT INTO order_info (user_id, product_name, productor) VALUES (9, 'p8', 'TE');
```

# explain结果的重要字段

- type：索引的类型，不同类型的索引的查询速度有着本质上的区别
- possible_keys&key：可能用到的索引和实际用到的索引，比如说数据量很少的情况下，即使查询的字段使用了索引，也有可能走全表扫描，此时possible_keys和key会不相同
- extra：查询过程的一些详细信息

重点看下type和extra

## type

性能由差到好的排列为：

- all：全表扫描
  - 例如：`EXPLAIN SELECT * FROM  user_info;`
- index：全索引扫描，按照索引的顺序进行全量扫描，相比all来说不用单独进行排序
  - 例如：`EXPLAIN SELECT name FROM  user_info;`
- range：部分索引扫描，相比index，需要扫描的数据量减少很多
  - 例如：`explain select * from user_info where id between 2 and 8;`
- ref：用到了索引，而且不是范围查找，索引不是唯一索引，如果索引是唯一索引，对应的type是const；join查询过程中，关联条件的字段有索引，且不是唯一索引，如果是唯一索引，对应的type是eq_ref
  - 对于单表查询的情况：`explain select * from user_info where name = 'xxx';`
  - 对于多表join的情况：`EXPLAIN SELECT * FROM user_info, order_info WHERE user_info.id = order_info.user_id AND order_info.user_id = 5;`
    - **`user_info.id`使用的是主键索引`PRIMARY`，所以是const，查询过程中，会首先去user_info表中根据主键索引找到user_info.id = 5的记录**
    - **`order_info.user_id`使用的是普通索引`user_product_detail_index`，所以是ref，查询过程中通过普通索引查找order_info.user=5的记录**
- eq_ref：关联查询中，关联条件
  - 例如：`EXPLAIN SELECT * FROM user_info, order_info WHERE user_info.id = order_info.user_id;`
    - **首先使用`user_product_detail_index`索引，查找表order_info的user_id，因为需要全索引扫描，所以类型为ref**
    - **根据上面的索引找到的user_id，去表user_info中查找user_info.id=?的记录，因为用到的是主键索引，但是又不是只有一条的精确=查询，所以type不是const，而是`eq_ref`**
- const：使用主键索引进行精确=查找
- system：表中只有一条记录的特殊情况

## extra

- using index：说明查询是覆盖了索引的，不需要读取数据文件，从索引树（索引文件）中即可获得信息。如果同时出现using where，表明索引被用来执行索引键值的查找，没有using where，表明索引用来读取数据而非执行查找动作。这是MySQL服务层完成的，但无需再回表查询记录。
  - 例如：`EXPLAIN SELECT * FROM order_info WHERE order_info.user_id = 5;`，用到了`user_product_detail_index`索引，因为`user_product_detail_index`索引包含mysql的所有列，所以获取列的信息直接从改索引中进行查询即可
- using index condition：这是MySQL 5.6出来的新特性，叫做“索引条件推送”。简单说一点就是MySQL原来在索引上是不能执行如like这样的操作的，但是现在可以了，这样减少了不必要的IO操作，但是只能用在二级索引上。
- using where：使用了WHERE从句来限制哪些行将与下一张表匹配或者是返回给用户。注意：Extra列出现Using where表示MySQL服务器将存储引擎返回服务层以后再应用WHERE条件过滤。**这种情况下往往是没有使用到索引的，例如在一个没有任何索引的列上使用where条件进行查询**。

---

### 参考

- [MySQL 性能优化神器 Explain 使用分析](https://segmentfault.com/a/1190000008131735)
- [MySQL EXPLAIN详解](https://www.jianshu.com/p/ea3fc71fdc45)

