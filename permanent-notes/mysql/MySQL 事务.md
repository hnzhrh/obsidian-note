---
title: MySQL 事务
tags:
  - permanent-note
  - middleware/database/mysql/transaction
date: 2025-03-13
time: 17:59
aliases: 
done: false
---
# 1 事务的 ACID 特性
## 1.1 概览

| 特性  | 实现方式        |
| --- | ----------- |
| 原子性 | Undo log    |
| 持久性 | Redo log    |
| 隔离性 | MVCC/锁      |
| 一致性 | 原子性+持久性+隔离性 |

## 1.2 原子性
一个事务只会全部成功或者全部失败，不会出现部分成功部分失败的情况，假设 ABC 三条 SQL，如果 A 执行成功后 B 执行失败了，则一定会回滚到 A 执行之前的状态。

Demo:
```sql
create table test_db.account
(
    id      bigint unsigned auto_increment
        primary key,
    balance int         not null,
    name    varchar(63) not null
);

INSERT INTO test_db.account (balance, name)  
VALUES (1000, 'Alice');
INSERT INTO test_db.account (balance, name)  
VALUES (1000, 'Bob');
```

事务正常执行：

```sql
-- 开启事务  
START TRANSACTION;  
  
-- 从Alice账户扣除500元  
UPDATE account SET balance = balance - 500 WHERE id = 1;  
  
SELECT * FROM account;
  
-- 向Bob账户增加500元  
UPDATE account SET balance = balance + 500 WHERE id = 2;  
  
-- 提交事务（所有操作生效）  
COMMIT;
```

事务回滚：

```sql
-- 开启事务  
START TRANSACTION;  
  
-- 从Alice账户扣除500元  
UPDATE account SET balance = balance - 500 WHERE id = 1;  
  
SELECT * FROM account;  
  
-- 向Bob账户增加500元  
UPDATE account SET balance = balance + 500 WHERE id = 999;  
    
-- 因错误主动回滚（或MySQL自动回滚）  
ROLLBACK;
```

`SELECT * FROM account` 的结果：

| id | balance | name |
| :--- | :--- | :--- |
| 1 | 500 | Alice |
| 2 | 1000 | Bob |
当执行了 `ROLLBACK` 后：

| id | balance | name |
| :--- | :--- | :--- |
| 1 | 1000 | Alice |
| 2 | 1000 | Bob |
数据回滚到初始状态。
## 1.3 一致性
AID 的结果是 C
## 1.4 隔离性
多事务并发执行对数据的影响程度，隔离性过高会导致性能越低。
## 1.5 持久性
事务处理结束后，对数据的修改是永久的，即使系统故障也不会丢失。
# 2 事务并发产生的问题
## 2.1 脏读
A 事务读取到了 B 事务**修改但没有提交**的数据就是脏读。

| 时序  | 事务 A                                                       | 事务 B                                                | 备注             |
| --- | ---------------------------------------------------------- | --------------------------------------------------- | -------------- |
| 1   |                                                            | `SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;` |                |
| 2   | `START TRANSACTION;`                                       | `START TRANSACTION;`                                |                |
| 3   | `UPDATE account SET balance = balance - 500 WHERE id = 1;` |                                                     |                |
| 4   |                                                            | `SELECT * FROM account;`                            | 读取到 500，业务开始处理 |
| 5   | `ROLLBACK;`                                                |                                                     |                |
| 6   |                                                            | `COMMIT`                                            | 错误数据           |

## 2.2 不可重复读
同一个事务内多**次读取同一个数据**的值不一致的情况就是不可重复读。

| 时序  | 事务 A                                                       | 事务 B                                              | 备注      |
| --- | ---------------------------------------------------------- | ------------------------------------------------- | ------- |
| 1   |                                                            | `SET TRANSACTION ISOLATION LEVEL READ COMMITTED;` |         |
| 2   | `START TRANSACTION;`                                       | `START TRANSACTION;`                              |         |
| 3   | `UPDATE account SET balance = balance - 500 WHERE id = 1;` |                                                   |         |
| 4   |                                                            | `SELECT * FROM account;`                          | 依然 1000 |
| 5   | `COMMIT;`                                                  |                                                   |         |
| 6   |                                                            | `SELECT * FROM account;`                          | 读取到 500 |

## 2.3 幻读

同一个事务内同一条件下**多次查询查出来的记录**不相同的情况就是幻读。

| 时序  | 事务 A                                                        | 事务 B                                              | 备注   |
| --- | ----------------------------------------------------------- | ------------------------------------------------- | ---- |
| 1   |                                                             | `SET TRANSACTION ISOLATION LEVEL READ COMMITTED;` |      |
| 2   | `START TRANSACTION;`                                        | `START TRANSACTION;`                              |      |
| 3   |                                                             | `SELECT * FROM account;`                          | 两条记录 |
| 4   | `INSERT INTO account(balance, name) VALUES(1000,'erpang');` |                                                   |      |
| 5   |                                                             | `SELECT * FROM account;`                          | 两条记录 |
| 6   | `COMMIT`                                                    |                                                   |      |
| 7   |                                                             | `SELECT * FROM account;`                          | 三条记录 |

# 3 事务隔离级别

| 隔离级别 | 解决问题             | 实现方式                         |
| ---- | ---------------- | ---------------------------- |
| 读未提交 |                  |                              |
| 读已提交 | 脏读               | MVCC Read View，每个语句执行前重新生成   |
| 可重复读 | 不可重复读、幻读（没有完全规避） | Read View（启动事务时生成）/MVCC/LOCK |
| 串行化  | 脏读、不可重复读、幻读      | 读写锁                          |

## 3.1 读未提交
会产生脏读、不可重复读、幻读
## 3.2 读已提交
解决了脏读问题
## 3.3 可重复读
MySQL 默认隔离级别
解决了脏读、不可重复读和幻读（没有完全避免）

| 时序  | 事务 A                                                       | 事务 B                                               | 备注       |
| --- | ---------------------------------------------------------- | -------------------------------------------------- | -------- |
| 1   |                                                            | `SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;` |          |
| 2   | `START TRANSACTION;`                                       | `START TRANSACTION;`                               |          |
| 3   | `UPDATE account SET balance = balance - 500 WHERE id = 1;` |                                                    |          |
| 4   |                                                            | `SELECT * FROM account;`                           | 依然 1000  |
| 5   | `COMMIT;`                                                  |                                                    |          |
| 6   |                                                            | `SELECT * FROM account;`                           |  依然 1000 |

## 3.4 序列化
`SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;`
对记录加读写锁，解决并发问题。

# 4 MySQL 如何解决幻读？

* 针对快照读（普通 `select`），通过 [MySQL MVCC 机制](MySQL%20MVCC%20机制.md) 解决
* 针对当前读, 通过 `next-key lock`（记录锁+间隙锁），读取的是记录的**最新版本**，读取时还需要保证其他事务不能修改该记录，**会对其加锁**
	* ``select * from xxx for update`
	* `select lock in share mode`
	* `update`
	* `insert`
	* `delete`

MySQL 可重复读很大程度上避免了幻读，但是没有完全解决幻读：

场景一：当前事务执行了 `UPDATE/DELETE` 等触发了当前读，且修改了记录的事务 ID，导致后续事务内普通查找幻读

| 时序  | 事务 A                                                          | 事务 B                                               | 备注                        |
| --- | ------------------------------------------------------------- | -------------------------------------------------- | ------------------------- |
| 1   |                                                               | `SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;` |                           |
| 2   | `START TRANSACTION;`                                          | `START TRANSACTION;`                               |                           |
| 3   |                                                               | `SELECT * FROM account where id > 4;`              | 查不到数据                     |
| 4   | `INSERT INTO account(id,balance, name) VALUES(5,1000,'new');` |                                                    |                           |
| 5   |                                                               | `SELECT * FROM account where id > 4;`              | 查不到数据                     |
| 6   | `COMMIT;`                                                     |                                                    |                           |
|     |                                                               | `SELECT * FROM account where id > 4;`              | 查不到数据                     |
|     |                                                               | `UPDATE account set name='change' WHERE id = 5;`   | 更新不存在的数据，事务 B 更新了记录的事务 ID |
|     |                                                               | `SELECT * FROM account where id > 4;`              | 查到数据幻读                    |
场景二：使用 `select for update` 当前读幻读

| 时序  | 事务 A                                                          | 事务 B                                               | 备注                      |
| --- | ------------------------------------------------------------- | -------------------------------------------------- | ----------------------- |
| 1   |                                                               | `SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;` |                         |
| 2   | `START TRANSACTION;`                                          | `START TRANSACTION;`                               |                         |
| 3   |                                                               | `SELECT * FROM account where id > 4;`              | 查不到数据                   |
| 4   | `INSERT INTO account(id,balance, name) VALUES(5,1000,'new');` |                                                    |                         |
| 5   |                                                               | `SELECT * FROM account where id > 4;`              | 查不到数据                   |
| 6   | `COMMIT;`                                                     |                                                    |                         |
|     |                                                               | `SELECT * FROM account where id > 4;`              | 查不到数据                   |
|     |                                                               | `SELECT * FROM account where id > 4 for update ;`  | `for update` 触发当前读，造成幻读 |
# 5 什么是快照读？什么是当前读？

MySQL 的 **快照读（Snapshot Read）** 和 **当前读（Current Read）** 是两种不同的数据读取机制，主要用于解决并发事务中的一致性和锁冲突问题。它们的核心区别在于 **读取数据时是否基于最新版本**，以及是否涉及锁机制。

---

## 5.1 快照读（Snapshot Read）

快照读是基于 **多版本并发控制（MVCC）** 的机制，读取的是数据的某个历史版本（快照），而不是当前最新的数据。这种读取方式通常是无锁的，因此性能较高。

### 5.1.1 触发场景
1. **普通 `SELECT` 语句**（非显式加锁的查询）。
2. 在 **可重复读（REPEATABLE READ）** 或 **读已提交（READ COMMITTED）** 隔离级别下。

### 5.1.2 工作机制
- **可重复读（REPEATABLE READ）**：事务启动时生成一个全局一致性快照，后续所有快照读都基于这个快照。
- **读已提交（READ COMMITTED）**：每次快照读都会生成新的快照，读取已提交的最新数据。

### 5.1.3 示例
```sql
-- 普通 SELECT 语句（快照读）
SELECT * FROM users WHERE id = 1;
```

### 5.1.4 特点
- **无锁**：不阻塞其他事务的写操作。
- **一致性视图**：读取的是某个时间点的数据版本，保证事务内一致性。
- **适用于读多写少场景**：如报表查询、历史数据分析。

## 5.2 当前读（Current READ）

当前读直接读取数据的最新提交版本，并且会对数据加锁（显式或隐式），确保其他事务无法修改当前数据。当前读的目的是保证数据的强一致性。

### 5.2.1 触发场景
1. **显式加锁的查询**：
   ```sql
   SELECT ... FOR UPDATE;      -- 加排他锁（X 锁）
   SELECT ... LOCK IN SHARE MODE; -- 加共享锁（S 锁）
   ```
2. **数据修改操作**：
   ```sql
   UPDATE、DELETE、INSERT  -- 隐式加锁，读取最新数据后修改。
   ```

### 5.2.2 工作机制
- 当前读会读取数据的最新提交版本，并加锁阻止其他事务修改。
- 在 **可重复读（REPEATABLE READ）** 隔离级别下，当前读还会触发 **间隙锁（Gap Lock）** 或 **临键锁（Next-Key Lock）**，防止幻读。

### 5.2.3 示例

```sql
-- 显式加锁的当前读
SELECT * FROM users WHERE id = 1 FOR UPDATE;
```

### 5.2.4 特点
- **加锁**：可能阻塞其他事务的写操作。
- **强一致性**：读取的是最新的已提交数据。
- **适用于写操作**：如事务需要基于最新数据做修改。

## 5.3 快照读 vs 当前读

| 特性               | 快照读                          | 当前读                          |
|--------------------|--------------------------------|--------------------------------|
| **数据版本**        | 历史快照版本                    | 最新已提交版本                  |
| **是否加锁**        | 无锁                           | 加锁（共享锁或排他锁）          |
| **触发语句**        | 普通 `SELECT`                  | `SELECT ... FOR UPDATE/LOCK IN SHARE MODE`、`UPDATE`、`DELETE`、`INSERT` |
| **隔离级别支持**    | READ COMMITTED、REPEATABLE READ | 所有隔离级别                    |
| **并发性能**        | 高（无锁）                     | 低（可能阻塞其他事务）          |

---

## 5.4 实际应用场景
1. **快照读**：
   - 需要高并发查询的业务（如用户浏览商品详情）。
   - 事务需要多次读取一致的数据（如对账场景）。

2. **当前读**：
   - 需要保证数据强一致性的修改操作（如库存扣减）。
   - 防止并发写冲突（如订单支付）。

---

## 5.5 注意事项
1. 在 **可重复读（REPEATABLE READ）** 隔离级别下，快照读和当前读可能在同一事务中混合使用，需注意数据一致性问题。
2. 当前读可能因锁冲突导致事务超时或死锁，需合理设计事务逻辑。
3. 使用 `FOR UPDATE` 或 `LOCK IN SHARE MODE` 时，尽量缩小锁的范围（如通过索引精准锁定行）。

通过理解快照读和当前读的区别，可以更好地设计事务逻辑，平衡性能与一致性需求。

# 6 Reference
* [事务隔离级别是怎么实现的？ \| 小林coding](https://xiaolincoding.com/mysql/transaction/mvcc.html)
* [MySQL 可重复读隔离级别，完全解决幻读了吗？ \| 小林coding](https://xiaolincoding.com/mysql/transaction/phantom.html)