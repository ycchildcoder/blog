---
title: 数据库基础
date: 2022-02-11 10:41:46
tags:
---

数据库基础

什么是数据库：数据库是一个以某种有组织的方式存储的数据集合。

表

表是一种结构化的文件，可用来存储某种特定类型的数据。

列

表由列组成，所有表都是由一个或多个列组成

行

表中的数据是按行存储的，所保存的每个记录存储在自己的行内。

主键

每一行都应该有可以唯一标识自己的列（或一组列）

什么是SQL

SQL是结构查询语言（structured query language），专门用来与数据库通信的语言。

什么是MYSQL

数据的所有存储，检索，管理和处理是由数据库软件-DBMS（数据库管理系统）完成的。MYSQL是一种DBMS

mysql命令行

mysql -u ben（用户名） -p -h myserver（server） -P 9999（端口）

选择数据库

use databases;

了解数据库和表

show databases; 展示数据库

show tables; 展示表

展示表字段

show columes from table; 或 desc table;

select 语句

select prod_name from products; 查询 prod_name 列

select prod_id, prod_name, prod_price from products; 查询多列

select * from products; 查询所有

检索不通的行

select distinct vend_id from products;  DISTINCT 关键字指示mysql返回不同的值

限制结果

select prod_name from products limit 5; 限制展示5条

使用完全限定的表明

select products.prod_name from products;

select products.prod_name from crashcourse.products;

排序检索数据

select prod_name from products order by prod_name; ORDER BY 字句对输出进行排序

多列排序

select prid_id, prod_price, prod_name from products order by prod_price, prod_name; 首先按价格，然后再按名称排序。

指定排序方向 DESC（降序），ASC（升序，默认）

select prod_id, prod_price, prod_name from products order by prod_price DESC; 

select prod_id, prod_price, prod_name from products order by prod_price desc, prod_name;  prod_price 降序， prod_name 升序。

过滤数据 where

select prod_name, prod_price from products where prod_price = 2.5;  查找prod_price 等于2.5 的记录。

select prod_name, prod_price from products where prod_price < 10; 查找prod_price 小于10 的记录；

范围值查询

select prod_name, prod_price from products where prod_price between 5 and 10; 检索价格在5和10之间的所有产品。

空值检查

select prod_name, prod_price from products where prod_price is NULL; 

组合where 过滤

AND操作符 - 指示检索满足所有给定条件的行

select prod_id, prod_price, proc_name from products where vend_id = 1003 and prod_price <=10;

OR 操作符 - 指示检索匹配任一条件的行

select prod_name, prod_price from products where vend_id=1002 and vend_id = 1003;

明确顺序

select prod_name, prod_price from products where (vend_id=1002 or vend_id=1003) and prod_price >= 10;

IN 操作符 - 用来指定条件范围

select prod_name, prod_price from products where vend_id in (1002,1003) order by prod_name;

NOT 操作符 - 否定它之后的条件

select prod_name, prod_price from products where vend_id not in (1002,1003) order by prod_name;

通配符进行过滤

LIKE操作符 - 搜索中，%标识任何字符出现任意次数

select prod_id, prod_name from products where prod_name like 'jet%';

select prod_id, prod_name from products where prod_name like '%anvil%'; 

下划线（_ ）通配符 ，只匹配单个字符而不是多个字符

select prod_id, prod_name from products where prod_name like '_ ton anvil'; 

正则表达式 进行搜索

基本字符匹配 - regexp 

select prod_name from products where prod_name REGEXP '1000' order by prod_name; 检索列prod_name 包含文本1000的所有行

select prod_name from products where prod_name regexp '.000' order by prod_name;

进行OR 匹配

select prod_name from products where prod_name regexp '1000|2000' order by prod_name;

匹配几个字符之一

select prod_name from products where prod_name regexp '[123] Ton' order by prod_name; [123] 意思匹配1或2或3.

匹配范围

select prod_name from products where prod_name regexp '[1-5] ton' order by prod_name; [1-5] 匹配1到5

匹配特殊字符  - 需带转义，匹配特殊字符，必须用\\\\为前导。 \\\\.表示查找.

select vend_name from vendors where vend_name regexp '\\\\.' order by vend_name;

匹配字符类

![image-20220211182759817](https://raw.githubusercontent.com/ycchildcoder/markdown/main/image-20220211182759817.png)

匹配多个实例

![image-20220211182925651](https://raw.githubusercontent.com/ycchildcoder/markdown/main/image-20220211182925651.png)



select prod_name from products where prod_name regexp '\\\\([0-9] sticks?\\\\)'

定位符

![image-20220214104639250](https://raw.githubusercontent.com/ycchildcoder/markdown/main/image-20220214104639250.png)

创建计算字段

拼接字段

select concat(vend_name, '(', vend_country, ')') from vendors order by vend_name;

使用别名

select concat(RTrim(vend_name),'(', RTrim(vend_country),')') AS vend_title from vendors order by vend_name;

执行算数计算

select prod_id, quantity, item_price from orderitems where order_num = 20005;

select prod_id, quantity, item_price, quantity*item_price as expanded_price from orderiems where order_num = 20005;

文本处理函数

select vend_name, upper(vend_name) as vend_name_upcase from vendors order by vend_name;

![image-20220214110835711](https://raw.githubusercontent.com/ycchildcoder/markdown/main/image-20220214110835711.png)

![image-20220214111200218](https://raw.githubusercontent.com/ycchildcoder/markdown/main/image-20220214111200218.png)

日期和时间处理函数

![image-20220214111330748](https://raw.githubusercontent.com/ycchildcoder/markdown/main/image-20220214111330748.png)



select cust_id, order_num from orders where order_date = '2005-09-01';

select cust_id, order_num, from orders where date(order_date) = '2005-09-01';

检索2005年9月下的所有订单

select cust_id, order_num from orders where date(order_date) between '2005-09-01' and '2005-09-30';

select cust_id, order_num from orders where year(order_date) = 2005 and month(order_date) = 9;

数值处理函数

![image-20220214112426231](https://raw.githubusercontent.com/ycchildcoder/markdown/main/image-20220214112426231.png)

汇总数据

聚集数据

![image-20220214135350694](https://raw.githubusercontent.com/ycchildcoder/markdown/main/image-20220214135350694.png)

AVG() 函数- 求得该列的平均值

select avg(prod_price) as avg_price from products;

select avg(prod_price) as avg_price from products where vend_id = 1003;

COUNT() 函数- 确定表中行的数目

select count(*) as num_cust from customers;

select count(cust_email) as num_cust from customers;

MAX() 函数- 返回列中最大值

select max(prod_price) as max_price from products;

MIN() 函数- 返回指定列最小值

select min(prod_price) as min_price from products;

SUM() 返回指定列的和

select sum(item_price*quantity) as total_price from orderitems where order_num = 20004;

聚焦不同值 - DISTINCT 平均只考虑各个不同的价格

select avg(distinct prod_price) as avg_price from products where vend_Id = 1003;

组合聚集函数

select coun(*) as num_items, min(prod_price) as price_min, max(prod_price) as price_max, avg(prod_price) as price_avg from products;

分组数据-group by and having

数组分组

select count(*) as num_prods from products where vend_id=10003;

创建分组

select vend_id, count(*) as num_prods from products group by vend_id;

