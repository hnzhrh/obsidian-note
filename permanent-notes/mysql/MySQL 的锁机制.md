---
title: MySQL 的锁机制
tags:
  - permanent-note
  - middleware/database/mysql/lock
date: 2025-03-13
time: 18:23
aliases: 
done: false
---


# 1 全局锁

```SQL
-- 加锁
FLUSH TABLES WITH READ LOCK;

-- 解锁
UNLOCK TABLES;
```

锁住整个 MySQL，所有的库变成只读，一般只用在停机迁移数据库，避免数据不一致。
一般来说，数据库迁移不能简简单单的锁库、dump ，参考[如何在不停机的情况下迁移数据库？](如何在不停机的情况下迁移数据库？.md)



# 2 表级锁

## 2.1 表锁
可以对表加读锁和写锁，读锁阻塞所有写操作，写锁阻塞所有读和写操作。

```sql
LOCK TABLES {table1} READ, {table2} WRITE ;  
  
UNLOCK TABLES ;
```

有一个点需要注意，如果线程 A 对表 T 加锁，则线程 A 接下来的操作只能访问表 T，不能访问其他表，报错：
```shell
[HY000][1100] Table '{table}' was not locked with LOCK TABLES
```
## 2.2 元数据锁 Metadata Locks
MDL 锁不需要显示调用，对数据库表操作时自动加锁：
* CRUD 加 MDL S 锁
* 变更表结构时加 X 锁
MDL 锁的目的是为了防止 CURD 操作时，其他事务变更表的结构。
## 2.3 意向锁 Intention Locks

意向锁的目的是为了判断表中有没有记录被加锁，如果没有意向锁，加表级 X 锁时需要查看所有记录有没有加 X 锁，效率太低。

* 事务请求在记录上加 S 锁时需要先在表上加 IS 锁
* 事务请求在记录上加 X 锁时需要现在表上加 IX 锁

意向锁之间是不会冲突的，比如事务 A 加了 IX 锁，事务 B 也能加 IX 锁，IX 锁只和表锁有冲突，冲突矩阵如下：
表级锁的兼容性如下（行表示已持有锁，列表示新请求锁）：

| |X|IX|S|IS|
|---|---|---|---|---|
|**X**|❌|❌|❌|❌|
|**IX**|❌|✔️|❌|✔️|
|**S**|❌|❌|✔️|✔️|
|**IS**|❌|✔️|✔️|✔️|

可以使用语句 `SHOW ENGINE INNODB STATUS` 查看意向锁信息：
```sql
TABLE LOCK table `test`.`t` trx id 10080 lock mode IX
```
表示事务 10080 在表 `test.t` 上持有 IX 锁。
## 2.4 自增锁 `AUTO-INC Locks`

`AUTO-INC` 锁是一种特殊的表级锁，用于 `AUTO_INCREMENT` 列的并发控制。当事务向带有自增列的表插入数据时，InnoDB 会获取此锁，其他事务需要等待，插入结束后释放锁，以此来保证自增 ID 的连续性。

配置 `innodb_autoinc_lock_mode` 提供了三种模式：

* 0：使用 `AUTO-INC` 表级锁，插入后释放，自增值完全连续，插入串行化性能最差
* 1：使用 `AUTO-INC` 表锁和轻量级锁
	* 单行插入：使用轻量级锁（分配自增值后立即释放）
	* 多行插入：使用 `AUTO-INC` 锁，保证同一语句的自增值连续
* 2：使用轻量级锁，自增值可能不连续，同一语句中的自增值也可能不连续

模式 2 轻量级锁可能带来自增是不连续，例如：
```SQL
-- 事务 A 插入多行
INSERT INTO t (data) VALUES ('A1'), ('A2'), ('A3');

-- 事务 B 同时插入多行
INSERT INTO t (data) VALUES ('B1'), ('B2'), ('B3');
```
事务 A 和事务 B 的自增值分配可能交替进行（如 `1,3,5` 和 `2,4,6`）。

使用模式 2 需要注意主从复制，不能使用 `SBR（Statement-Based Replication）`，要使用 ` RBR（Row-Based Replication）`，因为自增值的顺序和 SQL 执行顺序不一样，` SBR ` 无法保证数据一致性。
# 3 行级锁

**有一个很重要的点，锁是加在索引上的，如果 MySQL 没有走索引，遍历了表，则相当于加了表锁。**

## 3.1 S 锁和 X 锁

`InnoDB` 实现了两种标准的锁：
* S 锁，shared locks
* X 锁，exclusive locks

| 锁类型     | 简称  | 权限说明                             |
| ------- | --- | -------------------------------- |
| **共享锁** | S锁  | 允许持有锁的事务读取行数据（Read）              |
| **排他锁** | X锁  | 允许持有锁的事务修改（Update）或删除（Delete）行数据 |

S 锁和 X 锁兼容性：

|     | S   | X   |
| --- | --- | --- |
| S   | ✅   | ❌   |
| X   | ❌   | ❌   |

场景一：假设事务 T1 已持有 S 锁

| 新请求者 T2 的锁类型 | 是否立即授予？ | 结果状态                  |
| ------------ | ------- | --------------------- |
| S 锁          | ✅ 是     | T1 和 T2 同时持有 S 锁（并行读） |
| X 锁          | ❌ 否     | T2 必须等待 T1 释放 S 锁     |

场景二：假设事务 T1 已持有 X 锁

|新请求者 T2 的锁类型|是否立即授予？|结果状态|
|---|---|---|
|S 锁 或 X 锁|❌ 否|T2 必须等待 T1 释放 X 锁|

```SQL
-- 事务 T1
SELECT * FROM users WHERE id = 1 LOCK IN SHARE MODE; -- 获取 S 锁

-- 事务 T2
SELECT * FROM users WHERE id = 1 LOCK IN SHARE MODE; -- 立即获取 S 锁 ✅

-- 事务 T1
SELECT * FROM users WHERE id = 1 FOR UPDATE; -- 获取 X 锁

-- 事务 T2
UPDATE users SET name = 'Alice' WHERE id = 1; -- 等待 T1 释放 X 锁 ❌
```
## 3.2 记录锁 Record Locks

记录锁是**针对索引记录的行级锁**，如果记录中没有索引，则使用隐藏的索引，使用：
- **显式锁定**：
	- `X` 锁：`SELECT ... FOR UPDATE` 
	- `S` 锁：`SELECT ... LOCK IN SHARE MODE`
- **隐式锁定**：执行 `UPDATE`/`DELETE` 时，InnoDB 自动对受影响的行加记录锁。

可以使用命令 `SHOW ENGINE INNODB STATUS` 查看锁情况：
```SQL
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t` trx id 10078 lock_mode X locks rec but not gap Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0 0: len 4; hex 8000000a; asc ;; 1: len 6; hex 00000000274f; asc 'O;; 2: len 7; hex b60000019d0110; asc ;;
```

含义：
- **space id 58**：表空间 ID，标识数据存储位置。
- **page no 3**：数据页编号，锁所在的物理页。
- **index `PRIMARY`**：锁作用的索引（此处是主键索引）。
- **trx id 10078**：持有锁的事务 ID。
- **lock_mode X**：锁模式为排他锁（X 锁）。
- **locks rec but not gap**：仅锁定记录本身，不锁定间隙（区别于间隙锁）。
- **PHYSICAL RECORD**：记录的物理存储信息：
    - `hex 8000000a`：主键值（十六进制，实际值为 `10`，`8000000a` 是 InnoDB 内部编码格式）。   
    - `hex 00000000274f`：事务 ID 和回滚指针（用于 MVCC）。     
    - 其他字段：行数据的其他部分（如时间戳、隐藏列等）。
## 3.3 间隙锁 Gap Locks 和临键锁 Next-Key Locks

Gap Locks 是在 `RR` 隔离级别下防止幻读的关键机制，是锁定索引记录区间的锁，一般由**等值查询未命**或范围查询中触发，锁定范围为左开右开区间，可以防止其他事务插入数据，且间隙锁之间是兼容的。

Next-Key Locks 是 `RR` 隔离级别下 Record Locks + Gap Locks 的组合，一般由范围查询触发。

加 Gap 还是 Next-Key 锁主要看查询调节的边界点是 Gap 的开区间还是闭区间。

Gap 是满足查询条件的区间的上一个记录和下一个记录组成 Gap。

开始实验，使用 `SELECT * FROM performance_schema.data_locks;  ` 观察加锁情况
* `X/S`：Next-Key Locks
* `X,GAP`: Gap Locks
* `X,REC_NOT_GAP`: Record Locks


数据表：

| id  | balance | name |
| :-- | :------ | :--- |
| 1   | 1       | 1    |
| 100 | 100     | 100  |
| 200 | 200     | 200  |
| 300 | 300     | 300  |
| 400 | 400     | 400  |

**情景一：唯一索引列上等值查询未命中加间隙锁**

```SQL
-- 事务 A
set autocommit=false;  
START TRANSACTION;    
SELECT * FROM account WHERE id = 150 FOR UPDATE ;  
SELECT * FROM performance_schema.data_locks;  
  
-- 事务 B
-- 不阻塞
UPDATE account SET name = 'change' WHERE id = 100;  
UPDATE account SET name = 'change' WHERE id = 200;
-- 阻塞
INSERT INTO account(id,balance, name) VALUES(110,110,'100');


```

| ENGINE | ENGINE\_LOCK\_ID | ENGINE\_TRANSACTION\_ID | THREAD\_ID | EVENT\_ID | OBJECT\_SCHEMA | OBJECT\_NAME | PARTITION\_NAME | SUBPARTITION\_NAME | INDEX\_NAME | OBJECT\_INSTANCE\_BEGIN | LOCK\_TYPE | LOCK\_MODE | LOCK\_STATUS | LOCK\_DATA |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| INNODB | 281473506054952:3906:1102:281473433273296 | 3184 | 28193 | 596 | test\_db | account | null | null | null | 281473433273296 | TABLE | IX | GRANTED | null |
| INNODB | 281473506054952:3906:36:4:4:281473433270384 | 3184 | 28193 | 596 | test\_db | account | null | null | PRIMARY | 281473433270384 | RECORD | X,GAP | GRANTED | 200 |
锁类型为 GAP Locks，锁定的范围为 `（100,200）`

**情景二：唯一索引列上等值查询命中退化为记录锁**

```SQL
-- 事务 A
set autocommit=false;  
START TRANSACTION;   
SELECT * FROM account WHERE id = 100 FOR UPDATE ;  
SELECT * FROM performance_schema.data_locks;

-- 事务 B
-- 阻塞
UPDATE account SET name = 'change' WHERE id = 100;

-- 事务 C
-- 不会阻塞
INSERT INTO account(id,balance, name) VALUES(120,120,'120');
```

| ENGINE | ENGINE\_LOCK\_ID | ENGINE\_TRANSACTION\_ID | THREAD\_ID | EVENT\_ID | OBJECT\_SCHEMA | OBJECT\_NAME | PARTITION\_NAME | SUBPARTITION\_NAME | INDEX\_NAME | OBJECT\_INSTANCE\_BEGIN | LOCK\_TYPE | LOCK\_MODE | LOCK\_STATUS | LOCK\_DATA |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| INNODB | 281473506054952:3890:1100:281473433273296 | 3096 | 28193 | 165 | test\_db | account | null | null | null | 281473433273296 | TABLE | IX | GRANTED | null |
| INNODB | 281473506054952:3890:34:4:3:281473433270384 | 3096 | 28193 | 165 | test\_db | account | null | null | PRIMARY | 281473433270384 | RECORD | X,REC\_NOT\_GAP | GRANTED | 100 |

**情景三：唯一索引列上范围查询 Next-Key 锁**
```SQL
set autocommit=false;  
START TRANSACTION;  
SELECT * FROM account WHERE id < 250 FOR UPDATE ;  
SELECT * FROM performance_schema.data_locks;
```


| ENGINE | ENGINE\_LOCK\_ID                            | ENGINE\_TRANSACTION\_ID | THREAD\_ID | EVENT\_ID | OBJECT\_SCHEMA | OBJECT\_NAME | PARTITION\_NAME | SUBPARTITION\_NAME | INDEX\_NAME | OBJECT\_INSTANCE\_BEGIN | LOCK\_TYPE | LOCK\_MODE | LOCK\_STATUS | LOCK\_DATA |
| :----- | :------------------------------------------ | :---------------------- | :--------- | :-------- | :------------- | :----------- | :-------------- | :----------------- | :---------- | :---------------------- | :--------- | :--------- | :----------- | :--------- |
| INNODB | 281473506054952:3908:1102:281473433273296   | 3189                    | 28193      | 640       | test\_db       | account      | null            | null               | null        | 281473433273296         | TABLE      | IX         | GRANTED      | null       |
| INNODB | 281473506054952:3908:36:4:5:281473433270728 | 3189                    | 28193      | 640       | test\_db       | account      | null            | null               | PRIMARY     | 281473433270728         | RECORD     | X,GAP      | GRANTED      | 300        |
| INNODB | 281473506054952:3908:36:4:7:281473433270384 | 3189                    | 28193      | 640       | test\_db       | account      | null            | null               | PRIMARY     | 281473433270384         | RECORD     | X          | GRANTED      | 1          |
| INNODB | 281473506054952:3908:36:4:8:281473433270384 | 3189                    | 28193      | 640       | test\_db       | account      | null            | null               | PRIMARY     | 281473433270384         | RECORD     | X          | GRANTED      | 100        |
| INNODB | 281473506054952:3908:36:4:9:281473433270384 | 3189                    | 28193      | 640       | test\_db       | account      | null            | null               | PRIMARY     | 281473433270384         | RECORD     | X          | GRANTED      | 200        |
为什么会加 `（200,300）` 的 GAP 锁是为了防止幻读出现，比如其他事务插入了 ID 为 210 的数据。

```SQL
set autocommit=false;  
START TRANSACTION;  
SELECT * FROM account WHERE id < 200 FOR UPDATE ;  
SELECT * FROM performance_schema.data_locks;
```

| ENGINE | ENGINE\_LOCK\_ID | ENGINE\_TRANSACTION\_ID | THREAD\_ID | EVENT\_ID | OBJECT\_SCHEMA | OBJECT\_NAME | PARTITION\_NAME | SUBPARTITION\_NAME | INDEX\_NAME | OBJECT\_INSTANCE\_BEGIN | LOCK\_TYPE | LOCK\_MODE | LOCK\_STATUS | LOCK\_DATA |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| INNODB | 281473506054952:3920:1103:281473433273296 | 3232 | 28193 | 894 | test\_db | account | null | null | null | 281473433273296 | TABLE | IX | GRANTED | null |
| INNODB | 281473506054952:3920:37:4:8:281473433270728 | 3232 | 28193 | 894 | test\_db | account | null | null | PRIMARY | 281473433270728 | RECORD | X,GAP | GRANTED | 200 |
| INNODB | 281473506054952:3920:37:4:2:281473433270384 | 3232 | 28193 | 894 | test\_db | account | null | null | PRIMARY | 281473433270384 | RECORD | X | GRANTED | 1 |
| INNODB | 281473506054952:3920:37:4:3:281473433270384 | 3232 | 28193 | 894 | test\_db | account | null | null | PRIMARY | 281473433270384 | RECORD | X | GRANTED | 100 |

当小于 200 后，即使其他事务插入 200 ～ 300 之间的数据也不会造成幻读。

```SQL
set autocommit=false;  
START TRANSACTION;  
SELECT * FROM account WHERE id < 300 AND id > 150 FOR UPDATE ;  
SELECT * FROM performance_schema.data_locks;
```

| ENGINE | ENGINE\_LOCK\_ID | ENGINE\_TRANSACTION\_ID | THREAD\_ID | EVENT\_ID | OBJECT\_SCHEMA | OBJECT\_NAME | PARTITION\_NAME | SUBPARTITION\_NAME | INDEX\_NAME | OBJECT\_INSTANCE\_BEGIN | LOCK\_TYPE | LOCK\_MODE | LOCK\_STATUS | LOCK\_DATA |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| INNODB | 281473506054952:3924:1103:281473433273296 | 3234 | 28193 | 962 | test\_db | account | null | null | null | 281473433273296 | TABLE | IX | GRANTED | null |
| INNODB | 281473506054952:3924:37:4:7:281473433270728 | 3234 | 28193 | 962 | test\_db | account | null | null | PRIMARY | 281473433270728 | RECORD | X,GAP | GRANTED | 300 |
| INNODB | 281473506054952:3924:37:4:8:281473433270384 | 3234 | 28193 | 962 | test\_db | account | null | null | PRIMARY | 281473433270384 | RECORD | X | GRANTED | 200 |

如果是 `<=300`，则 300 可以直接加 Next-Key 锁，右闭合区间。
```SQL
set autocommit=false;  
START TRANSACTION;  
SELECT * FROM account WHERE id <= 300 AND id > 150 FOR UPDATE ;  
SELECT * FROM performance_schema.data_locks;
```

| ENGINE | ENGINE\_LOCK\_ID | ENGINE\_TRANSACTION\_ID | THREAD\_ID | EVENT\_ID | OBJECT\_SCHEMA | OBJECT\_NAME | PARTITION\_NAME | SUBPARTITION\_NAME | INDEX\_NAME | OBJECT\_INSTANCE\_BEGIN | LOCK\_TYPE | LOCK\_MODE | LOCK\_STATUS | LOCK\_DATA |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| INNODB | 281473506054952:3926:1103:281473433273296 | 3235 | 28193 | 1006 | test\_db | account | null | null | null | 281473433273296 | TABLE | IX | GRANTED | null |
| INNODB | 281473506054952:3926:37:4:7:281473433270384 | 3235 | 28193 | 1006 | test\_db | account | null | null | PRIMARY | 281473433270384 | RECORD | X | GRANTED | 300 |
| INNODB | 281473506054952:3926:37:4:8:281473433270384 | 3235 | 28193 | 1006 | test\_db | account | null | null | PRIMARY | 281473433270384 | RECORD | X | GRANTED | 200 |

**情景四：非唯一索引范围查询加锁可能会扩大范围**

```SQL
-- 事务 A
set autocommit=false;  
START TRANSACTION;  
SELECT * FROM account WHERE balance <= 300 AND balance >= 150 FOR UPDATE ;  
SELECT * FROM performance_schema.data_locks;  

```

| ENGINE | ENGINE\_LOCK\_ID | ENGINE\_TRANSACTION\_ID | THREAD\_ID | EVENT\_ID | OBJECT\_SCHEMA | OBJECT\_NAME | PARTITION\_NAME | SUBPARTITION\_NAME | INDEX\_NAME | OBJECT\_INSTANCE\_BEGIN | LOCK\_TYPE | LOCK\_MODE | LOCK\_STATUS | LOCK\_DATA |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| INNODB | 281473506054952:3946:1103:281473433273296 | 3247 | 28193 | 1478 | test\_db | account | null | null | null | 281473433273296 | TABLE | IX | GRANTED | null |
| INNODB | 281473506054952:3946:37:5:4:281473433270384 | 3247 | 28193 | 1478 | test\_db | account | null | null | account\_balance\_index | 281473433270384 | RECORD | X | GRANTED | 200, 200 |
| INNODB | 281473506054952:3946:37:5:5:281473433270384 | 3247 | 28193 | 1478 | test\_db | account | null | null | account\_balance\_index | 281473433270384 | RECORD | X | GRANTED | 300, 300 |
| INNODB | 281473506054952:3946:37:5:6:281473433270384 | 3247 | 28193 | 1478 | test\_db | account | null | null | account\_balance\_index | 281473433270384 | RECORD | X | GRANTED | 400, 400 |
| INNODB | 281473506054952:3946:37:4:7:281473433270728 | 3247 | 28193 | 1478 | test\_db | account | null | null | PRIMARY | 281473433270728 | RECORD | X,REC\_NOT\_GAP | GRANTED | 300 |
| INNODB | 281473506054952:3946:37:4:8:281473433270728 | 3247 | 28193 | 1478 | test\_db | account | null | null | PRIMARY | 281473433270728 | RECORD | X,REC\_NOT\_GAP | GRANTED | 200 |

## 3.4 插入意向锁 Insert Intention Locks

插入意向锁是指事务想插入已经拥有间隙锁的位置，会加一个插入意向锁。

例如：
```SQL
-- 事务 A
set autocommit=false;  
START TRANSACTION;  
SELECT * FROM account WHERE id <= 300 AND id >= 150 FOR UPDATE ;  
SELECT * FROM performance_schema.data_locks;  

-- 事务 B
-- 阻塞
INSERT INTO account(id,balance, name) VALUES(240,240,'240');

-- 事务 A
SELECT * FROM performance_schema.data_locks;
-- 可以看到 WAITING 状态的 INSERT_INTENTION锁
```

| ENGINE | ENGINE\_LOCK\_ID                            | ENGINE\_TRANSACTION\_ID | THREAD\_ID | EVENT\_ID | OBJECT\_SCHEMA | OBJECT\_NAME | PARTITION\_NAME | SUBPARTITION\_NAME | INDEX\_NAME | OBJECT\_INSTANCE\_BEGIN | LOCK\_TYPE | LOCK\_MODE              | LOCK\_STATUS | LOCK\_DATA |
| :----- | :------------------------------------------ | :---------------------- | :--------- | :-------- | :------------- | :----------- | :-------------- | :----------------- | :---------- | :---------------------- | :--------- | :---------------------- | :----------- | :--------- |
| INNODB | 281473506054952:3948:1103:281473433273296   | 3248                    | 28193      | 1525      | test\_db       | account      | null            | null               | null        | 281473433273296         | TABLE      | IX                      | GRANTED      | null       |
| INNODB | 281473506055760:166:1103:281473433279312    | 3249                    | 28280      | 307       | test\_db       | account      | null            | null               | null        | 281473433279312         | TABLE      | IX                      | GRANTED      | null       |
| INNODB | 281473506054952:3948:37:4:7:281473433270384 | 3248                    | 28193      | 1525      | test\_db       | account      | null            | null               | PRIMARY     | 281473433270384         | RECORD     | X                       | GRANTED      | 300        |
| INNODB | 281473506054952:3948:37:4:8:281473433270384 | 3248                    | 28193      | 1525      | test\_db       | account      | null            | null               | PRIMARY     | 281473433270384         | RECORD     | X                       | GRANTED      | 200        |
| INNODB | 281473506055760:166:37:4:7:281473433276400  | 3249                    | 28280      | 307       | test\_db       | account      | null            | null               | PRIMARY     | 281473433276400         | RECORD     | X,GAP,INSERT\_INTENTION | WAITING      | 300        |

# 4 Reference
* [MySQL :: MySQL 8.4 Reference Manual :: 17.7.1 InnoDB Locking](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)
* [MySQL 有哪些锁？ \| 小林coding](https://xiaolincoding.com/mysql/lock/mysql_lock.html)
* [MySQL 是怎么加锁的？ \| 小林coding](https://xiaolincoding.com/mysql/lock/how_to_lock.html#%E4%BB%80%E4%B9%88-sql-%E8%AF%AD%E5%8F%A5%E4%BC%9A%E5%8A%A0%E8%A1%8C%E7%BA%A7%E9%94%81)
* [MySQL 是怎样运行的：从根儿上理解 MySQL - 小孩子4919 - 掘金小册](https://juejin.cn/book/6844733769996304392/section/6844733770071801864)
* [MySQL :: MySQL 8.4 Reference Manual :: 10.11.4 Metadata Locking](https://dev.mysql.com/doc/refman/8.4/en/metadata-locking.html)
* [MySQL :: MySQL 8.4 Reference Manual :: 10.11.1 Internal Locking Methods](https://dev.mysql.com/doc/refman/8.4/en/internal-locking.html)