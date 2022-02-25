---
title: 事务隔离级别
date: 2022-02-13 10:41:46
tags:
---



### 事务的四大特性



![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/1712b5213446a402%7Etplv-t2oaga2asx-watermark.awebp)



-   **原子性：** 事务作为一个整体被执行，包含在其中的对数据库的操作要么全部都执行，要么都不执行。
-   **一致性：** 指在事务开始之前和事务结束以后，数据不会被破坏，假如A账户给B账户转10块钱，不管成功与否，A和B的总金额是不变的。
-   **隔离性：** 多个事务并发访问时，事务之间是相互隔离的，一个事务不应该被其他事务干扰，多个并发事务之间要相互隔离。。
-   **持久性：** 表示事务完成提交后，该事务对数据库所作的操作更改，将持久地保存在数据库之中。

## 事务并发存在的问题

事务并发执行存在什么问题呢，换句话说就是，一个事务是怎么干扰到其他事务的呢？看例子吧~

假设现在有表：

```
CREATE TABLE `account` (
  `id` int(11) NOT NULL,
  `name` varchar(255) DEFAULT NULL,
  `balance` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `un_name_idx` (`name`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
复制代码
```

表中有数据：



![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/171381960fd0f119%7Etplv-t2oaga2asx-watermark.awebp)



### 脏读（dirty read）

假设现在有两个事务A、B：

-   假设现在A的余额是100，事务A正在准备查询Jay的余额
-   这时候，事务B先扣减Jay的余额，扣了10
-   最后A 读到的是扣减后的余额



![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/1713824c77723cd4%7Etplv-t2oaga2asx-watermark.awebp)



由上图可以发现，事务A、B交替执行，事务A被事务B干扰到了，因为事务A读取到事务B未提交的数据,这就是**脏读**。

### 不可重复读（unrepeatable read）

假设现在有两个事务A和B：

-   事务A先查询Jay的余额，查到结果是100
-   这时候事务B 对Jay的账户余额进行扣减，扣去10后，提交事务
-   事务A再去查询Jay的账户余额发现变成了90



![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/1713829b86401900%7Etplv-t2oaga2asx-watermark.awebp)



事务A又被事务B干扰到了！在事务A范围内，两个相同的查询，读取同一条记录，却返回了不同的数据，这就是**不可重复读**。

### 幻读

假设现在有两个事务A、B：

-   事务A先查询id大于2的账户记录，得到记录id=2和id=3的两条记录
-   这时候，事务B开启，插入一条id=4的记录，并且提交了
-   事务A再去执行相同的查询，却得到了id=2,3,4的3条记录了。



![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/171382b2bdd28029%7Etplv-t2oaga2asx-watermark.awebp)



事务A查询一个范围的结果集，另一个并发事务B往这个范围中插入/删除了数据，并静悄悄地提交，然后事务A再次查询相同的范围，两次读取得到的结果集不一样了，这就是**幻读**。

这里，有朋友会问，不可重复读和幻读这不一样吗？

**答案为：不一样。有啥区别？**

幻读和不可重复读都是读取了另一条已经提交的事务；**但不可重复读查询的都是同一个数据项，而幻读针对的是一批数据整体**（比如数据的个数）。

不可重复读和幻读是初学者不易分清的概念；简单来说，解决不可重复读的方法是大家常说的**加行锁**，解决幻读方式是**加表锁**。

## 事务的四大隔离级别实践

既然并发事务存在**脏读、不可重复、幻读**等问题，InnoDB实现了哪几种事务的隔离级别应对呢？

-   **读未提交（Read Uncommitted）**
-   **读已提交（Read Committed）**
-   **可重复读（Repeatable Read）**
-   **串行化（Serializable）**

在MySQL中，实现了这四种隔离级别，分别有可能产生问题如下所示：

![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/1627454-eceded962ef591d1.png)

### 读未提交（Read Uncommitted）

想学习一个知识点，最好的方式就是实践之。好了，我们去数据库给它设置**读未提交**隔离级别，实践一下吧~



![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/17148fd0f161aea8%7Etplv-t2oaga2asx-watermark.awebp)



先把事务隔离级别设置为read uncommitted，开启事务A，查询id=1的数据

```
set session transaction isolation level read uncommitted;
begin;
select * from account where id =1;
复制代码
```

结果如下：



![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/17148f9415ffc166%7Etplv-t2oaga2asx-watermark.awebp)

这时候，另开一个窗口打开mysql，也把当前事务隔离级别设置为read uncommitted，开启事务B，执行更新操作



```
set session transaction isolation level read uncommitted;
begin;
update account set balance=balance+20 where id =1;
复制代码
```

接着回事务A的窗口，再查account表id=1的数据，结果如下：



![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/17148faf1e5d5f48%7Etplv-t2oaga2asx-watermark.awebp)



可以发现，在**读未提交（Read Uncommitted）** 隔离级别下，一个事务会读到其他事务未提交的数据的，即存在**脏读**问题。事务B都还没commit到数据库呢，事务A就读到了，感觉都乱套了。。。实际上，读未提交是隔离级别最低的一种。

### 已提交读（READ COMMITTED）

为了避免脏读，数据库有了比**读未提交**更高的隔离级别，即**已提交读**。



![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/17148c908b12084b%7Etplv-t2oaga2asx-watermark.awebp)



把当前事务隔离级别设置为已提交读（READ COMMITTED），开启事务A，查询account中id=1的数据

```
set session transaction isolation level read committed;
begin;
select * from account where id =1;
复制代码
```

另开一个窗口打开mysql，也把事务隔离级别设置为read committed，开启事务B，执行以下操作

```
set session transaction isolation level read committed;
begin;
update account set balance=balance+20 where id =1;
复制代码
```

接着回事务A的窗口，再查account数据，发现数据没变：



![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/1713d69e118832d2%7Etplv-t2oaga2asx-watermark.awebp)



我们再去到事务B的窗口执行commit操作：

```
commit;
复制代码
```

最后回到事务A窗口查询，发现数据变了：



![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/1713d68ad5a2fd47%7Etplv-t2oaga2asx-watermark.awebp)



由此可以得出结论，隔离级别设置为**已提交读（READ COMMITTED）** 时，已经不会出现脏读问题了，当前事务只能读取到其他事务提交的数据。但是，你站在事务A的角度想想，存在其他问题吗？

**提交读的隔离级别会有什么问题呢？**

在同一个事务A里，相同的查询sql，读取同一条记录（id=1），读到的结果是不一样的，即**不可重复读**。所以，隔离级别设置为read committed的时候，还会存在**不可重复读**的并发问题。

### 可重复读（Repeatable Read）

如果你的老板要求，在同个事务中，查询结果必须是一致的，即老板要求你解决不可重复的并发问题，怎么办呢？老板，臣妾办不到？来实践一下**可重复读（Repeatable Read）** 这个隔离级别吧~



![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/17144b324064255a%7Etplv-t2oaga2asx-watermark.awebp)



哈哈，步骤1、2、6的查询结果都是一样的，即**repeatable read解决了不可重复读问题**，是不是心里美滋滋的呢，终于解决老板的难题了~

**RR级别是否解决了幻读问题呢？**

再来看看网上的一个热点问题，有关于RR级别下，是否解决了幻读问题？我们来实践一下：



![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/1714911d7b42c350%7Etplv-t2oaga2asx-watermark.awebp)

由图可得，步骤2和步骤6查询结果集没有变化，看起来RR级别是已经解决幻读问题了~ 但是呢，**RR级别还是存在这种现象**：

![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/17142571375bd709%7Etplv-t2oaga2asx-watermark.awebp)



其实，**上图如果事务A中，没有`update account set balance=200 where id=5;`这步操作，`select * from account where id>2`查询到的结果集确实是不变**，这种情况没有**幻读**问题。但是，有了update这个骚操作，同一个事务，相同的sql，查出的结果集不同，这个是符合了**幻读**的定义~

这个问题，亲爱的朋友，你觉得它算幻读问题吗？

### 串行化（Serializable）

前面三种数据库隔离级别，都有一定的并发问题，现在放大招吧，实践SERIALIZABLE隔离级别。

把事务隔离级别设置为Serializable，开启事务A，查询account表数据

```
set session transaction isolation level serializable;
select @@tx_isolation;
begin;
select * from account;
复制代码
```

另开一个窗口打开mysql，也把事务隔离级别设置为Serializable，开启事务B，执行插入一条数据：

```
set session transaction isolation level serializable;
select @@tx_isolation;
begin;
insert into account(id,name,balance) value(6,'Li',100);
复制代码
```

执行结果如下：



![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/1714282f3cb7f7fa%7Etplv-t2oaga2asx-watermark.awebp)



由图可得，当数据库隔离级别设置为serializable的时候，事务B对表的写操作，在等事务A的读操作。其实，这是隔离级别中最严格的，读写都不允许并发。它保证了最好的安全性，性能却是个问题~

## 查看当前会话隔离级别

#### 方式1

```
命令：SHOW VARIABLES LIKE 'transaction_isolation';

mysql> show variables like 'transaction_isolation';
+-----------------------+--------------+
| Variable_name  | Value |
+-----------------------+--------------+
| transaction_isolation | SERIALIZABLE |
+-----------------------+--------------+
```

#### 方式2

```
命令：SELECT @@transaction_isolation;

mysql> select @@transaction_isolation;
+-------------------------+
| @@transaction_isolation |
+-------------------------+
| SERIALIZABLE            |
+-------------------------+
```

## 设置隔离级别

#### 方式1：通过set命令

```
SET [GLOBAL|SESSION] TRANSACTION ISOLATION LEVEL level;
其中level有4种值：
level: {
     REPEATABLE READ
   | READ COMMITTED
   | READ UNCOMMITTED
   | SERIALIZABLE
}
```

##### 关键词：GLOBAL

```
SET GLOBAL TRANSACTION ISOLATION LEVEL level;
* 只对执行完该语句之后产生的会话起作用
* 当前已经存在的会话无效
```

##### 关键词：SESSION

```
SET SESSION TRANSACTION ISOLATION LEVEL level;
* 对当前会话的所有后续的事务有效
* 该语句可以在已经开启的事务中间执行，但不会影响当前正在执行的事务
* 如果在事务之间执行，则对后续的事务有效。
```

##### 无关键词

```
SET TRANSACTION ISOLATION LEVEL level;
* 只对当前会话中下一个即将开启的事务有效
* 下一个事务执行完后，后续事务将恢复到之前的隔离级别
* 该语句不能在已经开启的事务中间执行，会报错的
```

#### 方式2：通过服务启动项命令

>   可以修改启动参数transaction-isolation的值
>
>   比方说我们在启动服务器时指定了--transaction-isolation=READ UNCOMMITTED，那么事务的默认隔离级别就从原来的REPEATABLE READ变成了READ UNCOMMITTED。