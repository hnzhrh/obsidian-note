---
title: MySQL-索引
tags:
  - fleet-note
  - middleware/database/mysql/index
date: 2024-12-02
time: 16:20
aliases:
---
**索引是提升查询速度的一种数据结构**。

**那为什么关系型数据库都热衷支持 B+树索引呢**？因为它是目前为止排序最有效率的数据结构。像二叉树，哈希索引、红黑树、SkipList，在海量数据基于磁盘存储效率方面远不如 B+ 树索引高效。

**B+树索引的特点是：** 基于磁盘的平衡树，但树非常矮，通常为 3~4 层，能存放千万到上亿的排序数据。树矮意味着访问效率高，从千万或上亿数据里查询一条数据，只用 3、4 次 I/O。

所有 B+ 树都是从高度为 1 的树开始，然后根据数据的插入，慢慢增加树的高度。**你要牢记：索引是对记录进行排序，** 高度为 1 的 B+ 树索引中，存放的记录都已经排序好了，若要在一个叶子节点内再进行查询，只进行二叉查找，就能快速定位数据。

可随着插入 B+ 树索引的记录变多，1个页（16K）无法存放这么多数据，所以会发生 B+ 树的分裂，B+ 树的高度变为 2，当 B+ 树的高度大于等于 2 时，根节点和中间节点存放的是索引键对，由（索引键、指针）组成。

索引键就是排序的列，而指针是指向下一层的地址，在 MySQL 的 InnoDB 存储引擎中占用 6 个字节。下图显示了 B+ 树高度为 2 时，B+ 树索引的样子：

![image.png](https://images.hnzhrh.com/note/20241205102705.png)

可以看到，在上面的B+树索引中，若要查询索引键值为 5 的记录，则首先查找根节点，查到键值对（20，地址），这表示小于 20 的记录在地址指向的下一层叶子节点中。接着根据下一层地址就可以找到最左边的叶子节点，在叶子节点中根据二叉查找就能找到索引键值为 5 的记录。

那一个高度为 2 的 B+ 树索引，理论上最多能存放多少行记录呢?

在 MySQL InnoDB 存储引擎中，一个页的大小为 16K，在上面的表 User 中，键值 userId 是BIGINT 类型，则：

```yaml
根节点能最多存放以下多个键值对 = 16K / 键值对大小(8+6) ≈ 1100
```

再假设表 User 中，每条记录的大小为 500 字节，则：

```undefined
叶子节点能存放的最多记录为 = 16K / 每条记录大小 ≈ 32
```

综上所述，树高度为 2 的 B+ 树索引，最多能存放的记录数为：

```yaml
总记录数 = 1100 * 32 =  35,200
```

也就是说，35200 条记录排序后，生成的 B+ 树索引高度为 2。在 35200 条记录中根据索引键查询一条记录只需要查询 2 个页，一个根叶，一个叶子节点，就能定位到记录所在的页。

高度为 3 的 B+ 树索引本质上与高度 2 的索引一致，如下图所示，不再赘述：
![image.png](https://images.hnzhrh.com/note/20241205102730.png)

同理，树高度为 3 的 B+ 树索引，最多能存放的记录数为：

```yaml
总记录数 = 1100（根节点） * 1100（中间节点） * 32 =  38,720,000
```

讲到这儿，你会发现，高度为 3 的 B+ 树索引竟然能存放 3800W 条记录。在 3800W 条记录中定位一条记录，只需要查询 3 个页。**那么 B+ 树索引的优势是否逐步体现出来了呢**？

不过，在真实环境中，每个页其实利用率并没有这么高，还会存在一些碎片的情况，我们假设每个页的使用率为60%，则：
![image.png](https://images.hnzhrh.com/note/20241205102750.png)


表格显示了 B+ 树的威力，即在 50 多亿的数据中，根据索引键值查询记录，只需要 4 次 I/O，大概仅需 0.004 秒。如果这些查询的页已经被缓存在内存缓冲池中，查询性能会更快。

**真正的开销在于 B+ 树索引的维护，保证数据排序，这里存在两种不同数据类型的插入情况**。

- **数据顺序（或逆序）插入：** B+ 树索引的维护代价非常小，叶子节点都是从左往右进行插入，比较典型的是自增 ID 的插入、时间的插入（若在自增 ID 上创建索引，时间列上创建索引，则 B+ 树插入通常是比较快的）。
- **数据无序插入：** B+ 树为了维护排序，需要对页进行分裂、旋转等开销较大的操作，另外，即便对于固态硬盘，随机写的性能也不如顺序写，所以磁盘性能也会收到较大影响。比较典型的是用户昵称，每个用户注册时，昵称是随意取的，若在昵称上创建索引，插入是无序的，索引维护需要的开销会比较大。

你不可能要求所有插入的数据都是有序的，因为索引的本身就是用于数据的排序，插入数据都已经是排序的，那么你就不需要 B+ 树索引进行数据查询了。

所以对于 B+ 树索引，在 MySQL 数据库设计中，仅要求主键的索引设计为顺序，比如使用自增，或使用函数 UUID_TO_BIN 排序的 UUID，而不用无序值做主键。


# MySQL 中 B+ 树索引的设计与管理

在 MySQL 数据库中，可以通过查询表 mysql.innodb_index_stats 查看每个索引的大致情况：

```sql
SELECT 

table_name,index_name,stat_name,

stat_value,stat_description 

FROM innodb_index_stats 

WHERE table_name = 'orders' and index_name = 'PRIMARY';

+----------+------------+-----------+------------+------------------+

|table_name| index_name | stat_name | stat_value |stat_description  |

+----------+-------------------+------------+------------+----------+

| orders | PRIMARY|n_diff_pfx01|5778522     | O_ORDERKEY            |

| orders | PRIMARY|n_leaf_pages|48867 | Number of leaf pages        |

| orders | PRIMARY|size        |49024 | Number of pages in the index|

+--------+--------+------------+------+-----------------------------+

3 rows in set (0.00 sec)
```

从上面的结果中可以看到，表 orders 中的主键索引，大约有 5778522 条记录，其中叶子节点一共有 48867 个页，索引所有页的数量为 49024。根据上面的介绍，你可以推理出非叶节点的数量为 49024 ~ 48867，等于 157 个页。

另外，我看见网上一些所谓的 MySQL“军规”中写道“一张表的索引不能超过 5 个”。**根本没有这样的说法，完全是无稽之谈。**

在我看来，如果业务的确需要很多不同维度进行查询，那么就该创建对应多索引，这是没有任何值得商讨的地方。

**真正在业务上遇到的问题是：** 由于业务开发同学对数据库不熟悉，创建 N 多索引，但实际这些索引从创建之初到现在根本就没有使用过！因为优化器并不会选择这些低效的索引，这些无效索引占用了空间，又影响了插入的性能。

**那你怎么知道哪些 B+树索引未被使用过呢**？在 MySQL 数据库中，可以通过查询表sys.schema_unused_indexes，查看有哪些索引一直未被使用过，可以被废弃：

```sql
SELECT * FROM schema_unused_indexes

WHERE object_schema != 'performance_schema';

+---------------+-------------+--------------+

| object_schema | object_name | index_name   |

+---------------+-------------+--------------+

| sbtest        | sbtest1     | k_1          |

| sbtest        | sbtest2     | k_2          |

| sbtest        | sbtest3     | k_3          |

| sbtest        | sbtest4     | k_4          |

| tpch          | customer    | CUSTOMER_FK1 |

| tpch          | lineitem    | LINEITEM_FK2 |

| tpch          | nation      | NATION_FK1   |

| tpch          | orders      | ORDERS_FK1   |

| tpch          | partsupp    | PARTSUPP_FK1 |

| tpch          | supplier    | SUPPLIER_FK1 |

+---------------+-------------+--------------+
```

如果数据库运行时间比较长，而且索引的创建时间也比较久，索引还出现在上述结果中，DBA 就可以考虑删除这些没有用的索引。

而 MySQL 8.0 版本推出了索引不可见（Invisible）功能。在删除废弃索引前，用户可以将索引设置为对优化器不可见，然后观察业务是否有影响。若无，DBA 可以更安心地删除这些索引：

```sql
ALTER TABLE t1 

ALTER INDEX idx_name INVISIBLE/VISIBLE;
```

# 函数索引

到目前为止，我们的索引都是创建在列上，从 MySQL 5.7 版本开始，MySQL 就开始支持创建函数索引 **（即索引键是一个函数表达式）。** 函数索引有两大用处：

- 优化业务 SQL 性能；
- 配合虚拟列（Generated Column）。

**先来看第一个好处，优化业务 SQL 性能。**

我们知道，不是每个开发人员都能比较深入地了解索引的原理，有时他们的表结构设计和编写 SQL 语句会存在“错误”，比如对于上面的表 User，要查询 2021 年1 月注册的用户，有些开发同学会错误地写成如下所示的 SQL：

```sql
SELECT * FROM User 

WHERE DATE_FORMAT(register_date,'%Y-%m') = '2021-01'
```

或许开发同学认为在 register_date 创建了索引，所以所有的 SQL 都可以使用该索引。**但索引的本质是排序，** 索引 idx_register_date 只对 register_date 的数据排序，又没有对DATE_FORMAT(register_date) 排序，因此上述 SQL 无法使用二级索引idx_register_date。

**数据库规范要求查询条件中函数写在等式右边，而不能写在左边**，就是这个原因。

我们通过命令 EXPLAIN 查看上述 SQL 的执行计划，会更为直观地发现索引 idx_register_date没有被使用到：

```markdown
EXPLAIN SELECT * FROM User

WHERE DATE_FORMAT(register_date,'%Y-%m') = '2021-01'

*************************** 1. row ***************************

           id: 1

  select_type: SIMPLE

        table: User

   partitions: NULL

         type: ALL

possible_keys: NULL

          key: NULL

      key_len: NULL

          ref: NULL

         rows: 1

     filtered: 100.00

        Extra: Using where
```

上述需求正确的 SQL 写法应该是，其中变化在第 2 行，主要将函数 DATE_FORMAT 插接为了一个范围查询：

```markdown
EXPLAIN SELECT * FROM User

WHERE register_date > '2021-01-01' 

AND register_date < '2021-02-01'

*************************** 1. row ***************************

           id: 1

  select_type: SIMPLE

        table: User

   partitions: NULL

         type: range

possible_keys: idx_register_date

          key: idx_register_date

      key_len: 8

          ref: NULL

         rows: 1

     filtered: 100.00

        Extra: Using index condition
```

如果线上业务真的没有按正确的 SQL 编写，那么可能造成数据库存在很多慢查询 SQL，导致业务缓慢甚至发生雪崩的场景。**要尽快解决这个问题，可以使用函数索引，** 创建一个DATE_FORMAT(register_date) 的索引，这样就能利用排序数据快速定位了：

```sql
ALTER TABLE User 

ADD INDEX 

idx_func_register_date((DATE_FORMAT(register_date,'%Y-%m')));
```

接着用命令 EXPLAIN 查看执行计划，就会发现 SQL 可以使用到新建的索引idx_func_register_date：

```markdown
EXPLAIN SELECT * FROM User 

WHERE DATE_FORMAT(register_date,'%Y-%m') = '2021-01'

*************************** 1. row ***************************

           id: 1

  select_type: SIMPLE

        table: User

   partitions: NULL

         type: ref

possible_keys: idx_func_register_date

          key: idx_func_register_date

      key_len: 31

          ref: const

         rows: 1

     filtered: 100.00

        Extra: NULL
```

上述创建的函数索引可以解决业务线上的燃眉之急，但强烈建议业务开发同学在下一个版本中优化 SQL，否则这会导致对同一份数据做了两份索引，索引需要排序，排序多了就会影响性能。

**函数索引第二大用处是结合虚拟列使用。**

在前面的 JSON 小节中，我们已经创建了表 UserLogin：

```sql
CREATE TABLE UserLogin (

    userId BIGINT,

    loginInfo JSON,

    cellphone VARCHAR(255) AS (loginInfo->>"$.cellphone"),

    PRIMARY KEY(userId),

    UNIQUE KEY idx_cellphone(cellphone)

);
```

其中的列 cellphone 就是一个虚拟列，它是由后面的函数表达式计算而成，本身这个列不占用任何的存储空间，而索引 idx_cellphone 实质是一个函数索引。这样做得好处是在写 SQL 时可以直接使用这个虚拟列，而不用写冗长的函数：

```sql
-- 不用虚拟列

SELECT  *  FROM UserLogin

WHERE loginInfo->>"$.cellphone" = '13918888888'

-- 使用虚拟列

SELECT  *  FROM UserLogin 

WHERE cellphone = '13918888888'
```

对于爬虫类的业务，我们会从网上先爬取很多数据，其中有些是我们关心的数据，有些是不关心的数据。通过虚拟列技术，可以展示我们想要的那部分数据，再通过虚拟列上创建索引，就是对爬取的数据进行快速的访问和搜索。

# 组合索引




# Reference
* [08 索引：排序的艺术](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%98%E5%AE%9D%E5%85%B8/08%20%20%E7%B4%A2%E5%BC%95%EF%BC%9A%E6%8E%92%E5%BA%8F%E7%9A%84%E8%89%BA%E6%9C%AF.md)