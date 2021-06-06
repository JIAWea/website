---
title: MySQL优化之SQL优化
date: 2020-05-08 09:30:18
categories: 数据库
tags: MySQL
---

### 1 查询SQL尽量不要使用 `select *` ,而是`select`具体字段。

反例：
```
select * from employee;
```

正例：
```
select id, name from employee;
```
说明:
1. 只取需要的字段，节省资源、减少网络开销。
2. select * 进行查询时，很可能就不会使用到覆盖索引了，就会造成回表查询。

<!--more-->

### 2 如果只查询结果只有一条的数据或者最大、最小一条记录，使用limit
假如有employee员工表，要找出一个名字叫ray的人
```
CREATE TABLE  `employee`(
  `id` int (11) NOT NULL,
  `name` varchar(255) DEFAULT NULL,
  `age` int(11) DEFAULT NULL,
  `date` datetime DEFAULT NULL,
  `sex` int (1) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

反例：
```
select id, name from employee where name='ray'
```

正例：
```
select id, name from employee where name='ray' limit 1;
```

说明：
1. 加上limit 1后,只要找到了对应的一条记录,就不会继续向下扫描了,效率将会大大提高。
2. 当然，如果name是唯一索引的话，是不必要加上limit 1了，因为limit的存在主要就是为了防止全表扫描，从而提高性能,如果一个语句本身可以预知不用全表扫描，有没有limit ，性能的差别并不大。


### 3 批量插入

反例
```
INSERT into person(name,age) values('A',24)
INSERT into person(name,age) values('B',24)
INSERT into person(name,age) values('C',24)
```

正例
```
INSERT into person(name,age) values('A',24),('B',24),('C',24);
```
说明:
1. 比较常规，就不多做说明了


### 4 优化like语句
尽量使用右模糊`xxx%`代替左模糊或者全模糊

反例：
```
select userId, name from user where user_id like '%123';
select userId, name from user where user_id like '%123%';
```

正例：
```
select userId, name from user where user_id like '123%';
```


### 5 避免SQL中对where字段进行函数转换或表达式计算

反例：
```
select * from employee where user_id + 1 = 15551;
```

正例：
```
select * from employee where user_id  = 15550;
```

说明：
1. 将导致系统放弃使用索引而进行全表扫
2. 其实在知道了有SQL优化器之后，我个人感觉这种普通的表达式转换应该可以提前进行处理再进行查询，这样一来就可以用到索引了，但是问题又来了，如果mysql优化器可以提前计算出结果，那么写sql语句的人也一定可以提前计算出结果，所以矛盾点在这个地方，导致5.7版本以前的此种情况都无法使用索引吧，未来可能会对其进行优化


### 6 超大分页场景解决方案(limit)

我们日常做分页需求时，一般会用 limit 实现，但是当偏移量特别大的时候，查询效率就变得低下。

反例：
```
select id, name, age from employee LIMIT 1000000, 10;
```

正例：
```
// 方案一：返回上次查询的最大记录（偏移量）
select id, name, age from employee id > 1000000 limit 10;

// 方案二：先通过索引查询需要的id
select id, name, age from employee a, (select id from employee limit 1000000, 10) b where a.id = b.id;

// 方案三：order by + 索引
select id, name, age from employee order by id limit 1000000, 10
```

说明：
1. MySQL 并不是跳过 offset 行，而是取 offset+N 行，然后返回放弃前 offset 行，返回 N 行，那当 offset 特别大的时候，效率就非常的低下，要么控制返回的总页数，要么对超过特定阈值的页数进行 SQL 改写


### 7 尽量避免在where子句中使用!=或<>操作符，否则将引擎放弃使用索引而进行全表扫描。

反例：
```
select age, name from user where age <> 18;
```

正例：
```
// 可以考虑分开两条sql写
select age, name from user where age < 18;
select age, name from user where age > 18;
```

说明：
1. 使用!=和<>很可能会让索引失效


### 8 对查询进行优化，应考虑在where及order by涉及的列上建立索引，尽量避免全表扫描。
对经常使用where和order by的字段加上索引


### 9 where子句中考虑使用默认值代替null。

反例：
```
select * from user where age is not null;
```

正例：
```
// 设置默认值为 0 
select * from user where age > 0;
```

说明：
1. 并不是说使用了is null 或者 is not null 就会不走索引了，这个跟mysql版本以及查询成本都有关。
2. 如果把null值，换成默认值，很多时候让走索引成为可能，同时，表达意思会相对清晰一点。


### 10 索引不宜太多，一般5个以内

说明：
1. 索引并不是越多越好，索引虽然提高了查询的效率，但是也降低了插入和更新的效率。
2. insert或update时有可能会重建索引，所以建索引需要慎重考虑，视具体情况来定。
3. 一个表的索引数最好不要超过5个，若太多需要考虑一些索引是否没有存在的必要。

###　11 索引不适合建在有大量重复数据的字段上，如性别这类型数据库字段

说明：
1. 因为SQL优化器是根据表中数据量来进行查询优化的，如果索引列有大量重复数据，Mysql查询优化器推算发现不走索引的成本更低，很可能就放弃索引了。
