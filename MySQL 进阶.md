## 1. MySql 的整体架构



### 1. MySQL 的架构图



如果能在头脑中构建出一幅 MySQL 各组件之间如何协同工作的架构图，就会有助于深入理解 MySQL 服务器。如下图展示了 MySQL 的逻辑架构图

![image-20260311153935340](MySQL 进阶.assets/image-20260311153935340.png)

### 2. 最上层 <客户端>



最上层的服务并不是 MySQL 所独有的，大多数基于网络的客户端/服务器的工具或者服务都有类似的架构。比如连接处理、授权认证、安全等等。



### 3. 第二层 <查询缓存 | 解析器>



括连接器、查询缓存、分析器、优化器、执行器等，涵盖 mysql 的大多数核心服务功能，以及所有的内置函数（例如日期、世家、数学和加密函数等），所有跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图等



### 4. 第三层 <存储引擎>



负责数据的存储和提取，是真正与底层物理文件打交道的组件。 数据本质是存储在磁盘上的，通过特定的存储引擎对数据进行有组织的存放并根据业务需要对数据进行提取。存储引擎的架构模式是插件式的，支持 Innodb，MyIASM、Memory 等多个存储引擎。现在最常用的存储引擎是 Innodb，它从 mysql5.5.5 版本开始成为了默认存储引擎



### 5. 第四层 <物理层面>



存储数据库真正的表数据、日志等。物理文件包括：redolog、undolog、binlog、errorlog、querylog、slowlog、data、index等



## 2. MySql 索引结构详解：



### 1. 索引的通俗介绍？



#### 1. 索引是什么？



>   索引是帮助 MySql（数据库）高校获取数据的 **排好序 的 数据结构**

![image-20210622150545531](MySQL 进阶.assets/image-20210622150545531.png)



#### 2. 索引为什么不是其它数据结构？



>   思考：索引数据结构为什么不是：二叉树、红黑树、Hash表、B - Tree ？
>

因为二叉树首先极端情况下会**退化为链表**（比如插入依次递增的数据）并且高度不可控就意味者磁盘 IO 次数增加消耗性能、平衡二叉树和红黑树虽然解决了退化成了链表的问题、但是高度依然不可控、在大数据量的情况下磁盘 IO 性能拉跨、正好 B Tree 结构横向扩展了节点的存储空间、就可以放更多的索引元素，既节省了 IO 次数、又极大的提升了整体性能



### 2. B-Tree 类的数据结构



#### 1. B - Tree 介绍



-   叶子节点具有相同的深度
-   叶子节点的指针为空
-   所有索引元素不重复
-   节点中的数据索引从左到右递增排列

![image-20210622151026695](MySQL 进阶.assets/image-20210622151026695.png)



#### 2. B + Tree 介绍



-   非叶子节点不存储 data 、只存储索引、可以放更多的索引
-   叶子节点包含所有索引字段
-   叶子节点用指针连接、提高区间访问的性能

![image-20210622151321324](MySQL 进阶.assets/image-20210622151321324.png)

B+ Tree 其实在叶子节点还有前后指向两边的指针、指针指向下一个叶子节点的磁盘上的位置信息，一个节点的索引数据 MySql  默认分配的是 16 kb 大小、一般 B+ Tree 的一个节点 Load 到内存等于做一次磁盘 IO、比直接在磁盘中定位数据所在的节点要快很多，因为在内存中节点数据也是从左到右依次递增的、所以可以采取一些算法（比如折半查找）耗费的性能几乎可以忽略不记

>   那么 16 kb 的页大小可以放多少个索引呢？下面以 BigInt（ 在 MySql 中占用 8 个 Byte ） 类型来算
>

每个节点元素里还会存储下一个节点的磁盘指针地址、大概占用 6 Byte，那么就是 16kb / (8 + 6) Byte ≈ 1170 个索引元素、这是一个页大小还不是叶子节点、假如叶子节点数据很大、一行的记录几十个字段一般不会超过 1 kb 的大小加上前后指针顶多 12 Byte 、一张 B+ Tree 数据结构能存储 1170 * 1170 * 16 ≈ 2190 万条记录、那么通过 3 次磁盘 IO 就可以在两千万多条记录定位到一条指定的数据

可通过以下 Sql 查看页码大小 (不推荐修改) 

```java
Show Global Status Like 'Innodb_page_size';
```



#### 3. 索引在磁盘上的存储位置



>   默认在 MySql 的安装目录中的 data 目录中

![image-20210622151610387](MySQL 进阶.assets/image-20210622151610387.png)

>   一般来说一个库对应一个文件夹：如下图直观所示

![image-20210622151636230](MySQL 进阶.assets/image-20210622151636230.png)



>   一个库里的数据库磁盘文件就在这个文件夹下，存储了包括数据、表结构、索引等信息

![image-20210622185657432](MySQL 进阶.assets/image-20210622185657432.png)

#### 4. B+ Tree 和 B - Tree 的区别 ？



>   B+ Tree 和 B - Tree 的区别 ？以及 MySql 为什么用 B+ Tree 作为索引的数据结构 ?

首先最大的区别是 B - Tree 的节点每个都是存储数据的，而 B+ Tree 只有叶子节点才存储数据

| 特性             | **B-Tree (B树)**                                             | **B+Tree (B+树)**                                            |
| :--------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **数据存储位置** | **<u>所有节点都存储数据</u>**                                | **<u>仅叶子节点存储数据；非叶子节点只存储索引键</u>**        |
| **叶子节点结构** | 叶子节点之间**没有链表连接**                                 | 所有叶子节点通过**指针连接成一个有序的单向/双向链表**        |
| **冗余键值**     | 没有冗余，键值在整棵树中唯一                                 | **存在冗余**，非叶子节点的键值一定会出现在叶子节点中         |
| **查找效率**     | 查找路径可能提前结束（在非叶子节点命中）                     | 查找路径**必须走到叶子节点**才能命中数据，路径长度固定（树高稳定） |
| **范围查询**     | **效率较低**。找到下界后，需要通过中序遍历（回溯父节点）来找下一个值。 | **效率极高**。找到下界后，直接通过叶子节点的链表顺序遍历即可。 |

对于一张千万级别的表，B+Tree 的树高通常只有 3-4 层、极强的范围查询能力、**B-Tree**：需要不断地在父子节点之间回溯，进行中序遍历，这会带来大量随机 I/O。而 **B+ Tree 叶子节点本身就有区间指针**



### 3. MylSAM 和 InnoDB 存储引擎



#### 1、MylSAM 存储引擎



##### 1、MylSAM 索引实现



>   MylSAM （非聚集索引）索引文件和数据文件是分离的、如下图所示



![image-20210622185911019](MySQL 进阶.assets/image-20210622185911019.png)

MylSAM 引擎的索引文件是存储在 .MYI 后缀的文件中的、表的数据在 .MYD 后缀的文件中的. 当通过索引来找一条数据时，先通过 B+ Tree 找到数据所在的叶子节点、再拿到叶子节点存储的磁盘指针指向 MYD 文件中数据所在的行. 如下图

![image-20210622190000265](MySQL 进阶.assets/image-20210622190000265.png)



#### 2、InnoDB 存储引擎



##### 1、InnoDB 索引实现



InnoDB（聚集索引） 引擎下的表数据文件本身就是一个按照 B+ Tree 组织的一个索引结构文件

聚集索引：叶节点包含了完整的表数据记录

![image-20210622190135987](MySQL 进阶.assets/image-20210622190135987.png)

InnoDB 的磁盘文件

![image-20210622193046733](MySQL 进阶.assets/image-20210622193046733.png)



##### 2、思考一些问题



>   1、为什么建议 InnoDB 表必须建立主键？

因为 InnoDB 模式下一张表的构建本来就是按照 B+ Tree 结构来构建的整一张表、有主键的话就按照主键来建造索引、如果没有主键 MySql 会从第一列开始往后选、每个元素都不相同的列、作为索引，如果找不到 MySql 会自己造一个隐藏列来当索引。



>   2、为什么推荐使用整型的自增主键 ？

因为在通过索引查找一个元素的时候，需要比对数据的大小和区间来定位数据的所在位置，整型的比对速度要比字符串的速度要高很多

推荐自增主键的原因：B+ Tree 本身就会将插入的元素排好序维护一个结构，如果非自增的索引插入假如正好插入到节点区间，可能会不断的造成 B+ Tree 为了维护性质而做过多的分裂、融合等耗时操作、会有一部分性能损耗，自增索引正好都是插入到 B+ Tree 的右端不断新增节点，无需分裂



>   3、InnoDB 的主键（聚集）索引 和 非主键（二级）索引有什么区别呢？

MylSAM 的没什么区别，而 InnoDB 模式下，非主键索引的叶子节点里的值存储的就不是整整一列的数据了，而是这列的主键索引的值、如下图所示因为为了 **保持一致性和节约存储空间**。不然就有两个副本的列数据，插入更新操作都要相应的维护两套数据的成本大大增加

![image-20210622193014560](MySQL 进阶.assets/image-20210622193014560.png)



#### 3、两种存储引擎的区别



-   InnoDB 支持事务，MyISAM 不支持
-   InnoDB 支持外键，MyISAM 不支持
-   InnoDB 不支持 FULLTEXT 类型的索引 
-   InnoDB 中不保存表的行数
    -   如 select count() from table 时，InnoDB 需要扫描一遍整个表来计算有多少行，但是 MyISAM 只要简单的读出保存好的行数即可，注意的是，当 count() 语句包含 where 条件时 MyISAM 也需要扫描整个表 

-   对于自增字段，InnoDB 中必须包含只有该字段的索引，但是在 MyISAM 表中可以和其他字段一起建立联合索引

-   清空整个表时，InnoDB 是一行一行的删除，效率非常慢。MyISAM 则会重建表 

-   InnoDB 支持行锁（某些情况下还是锁整表，如 update table set a=1 where user like ‘％lee％’) MyISAM 是表锁

| -        | Innodb                                   | Myisam                                                 |
| -------- | ---------------------------------------- | ------------------------------------------------------ |
| 存储文件 | （1） .frm 表定义文件（2） .ibd 数据文件 | （1）.frm 表定义文（2）.myd 数据文件（3）.myi 索引文件 |
| 锁       | 表锁、行锁                               | 表锁                                                   |
| 事务     | ACID                                     | 不支持                                                 |
| CRDU     | 读、写                                   | 读多                                                   |
| count    | 扫表                                     | 专门存储的地方                                         |
| 索引结构 | B+ Tree                                  | B+ Tree                                                |

> 提问：为什么  InnoDB 是一行一行的删除，而 MyISAM 则会重建表 ？
>
> 核心原因在于**InnoDB** 和 **MyISAM** 的架构理念完全不同。InnoDB 支持 **事务、日志、以及存储结构设计的差异**，InnoDB 必须保证事务一致性和可回滚。所以 InnoDB 必须：
>
> 1. **为每一行记录写 Undo Log**
> 2. 记录 Redo Log
> 3. 更新索引
> 4. 支持事务回滚
>
> **而 MyISAM 因为没有事务可以直接重建表**





### 4. MySql 的 Hash 索引结构



#### 1、Hash 索引结构介绍



默认来说 MySql 的索引结构是 B+Tree 结构的，下面还有 HASH 结构的索引

![image-20210622194909077](MySQL 进阶.assets/image-20210622194909077.png)

>   对索引的 key 进行一次 hash 计算那就可以定位出数据存储的位置，很多时候 hash 索引要比 B+ 树索引更高效、缺点仅能满足 “=”、“IN”，不支持范围查询，hash 冲突问题、具体存储结构如下

![image-20210622194934476](MySQL 进阶.assets/image-20210622194934476.png)



### 5. 索引的最左前缀原理



#### 1、联合索引底层结构是怎样的 ？



如下图所示的联合索引的存储结构，以下是联合主键索引的示例

![image-20210622195056651](MySQL 进阶.assets/image-20210622195056651.png)

建立联合索引的时候 MySql 会根据建立索引字段的先后顺序来建立索引的，在跳过了 name 字段，在 name 字段的值相等的区间是有序的，但是在整张表中跳过了第一个有序字段，后续字段无法保证其顺序性，所以走不了索引。例如以下的 Sql 会走索引的是第一条，因为你不能跳过第一个联合索引直接查 age 或者 position 字段就破坏了索引的有序性，这样是不会走索引的



>   跳过第一个联合索引的字段就不能走索引？

因为 MySql 构建 B+ Tree 联合索引的时候实先保证第一个索引的有序性进行构建，这样一来，剩余的索引字段就不一定有序，既然不能保证后续索引的有序性在跳过了第一个索引字段后因为无序性就不能通过索引查询、意味着只有通过全表扫描才能不遗漏的找到查询数据



## 3. Explain 分析与索引最佳实践



### 1. Explain 分析工具的简介



#### 1、Explain 官方文档介绍



使用 EXPLAIN 关键字可以模拟优化器执行 SQL 语句，分析你的查询语句或是结构的性能瓶颈. 在 Select 语句之前增加 Explain 关键字，MySQL 会在查询上设置一个标记执行查询会返回执行计划的信息，而不是执行这条SQL、注意：如果 from 中包含子查询，仍会执行该子查询，将结果放入临时表中

>   MySql 推荐版本 5.7.29 
>
>   Explain 官方使用文档链接：https://dev.mysql.com/doc/refman/5.7/en/explain-output.html



#### 2、Explain 的两个变种



explain：会在 explain 的基础上额外提供一些查询优化的信息，紧随其后通过 Show warnings 命令可以得到优化后的查询语句，从而看出优化器优化了什么额外还有 filtered 列，是一个半分比的值，rows * filtered / 100 可以估算出将要和 explain 中的前一个表进行连接的行数（前一个表指 explain 中的值比当前表 id 值小的表）

如下 Sql 语句

```sql
mysql> explain extended select * from film where id = 1;
```

![image-20210622200431800](MySQL 进阶.assets/image-20210622200431800.png)



```sql
mysql > show warnings
```

![image-20210622203846320](MySQL 进阶.assets/image-20210622203846320.png)



explain partitions：相比 explain 多了 partitions 字段、如果查询是基于分区表的话，会显示查询将访问的分区



### 2. Explain 分析中的列



接下来我们将展示 explain 中每个列的信息



#### 1、id 列



id 列的编号是 select 的序列号、有几个 select 就有几个 id，并且 id 的顺序是按 select 出现的顺序增长的。**id 列越大执行优先级越高**、id 相同则从上往下依次执行、其中 id 为 NULL 意味者最后执行



#### 2、select_type 列



>   表示对应行是简单还是复杂的查询其中包含以下类型（常用）：

-   **simple**：简单查询，查询不包含子查询和 union、例如 Sql 语句

```sql
mysql > explain select * from film where id = 2
```

-   **primary**：复杂查询中括号最外层的 select

```sql
mysql > EXPLAIN SELECT * FROM users WHERE id IN (SELECT user_id FROM orders);
-- 执行计划
-- PRIMARY        users
-- SUBQUERY       orders
```

-   **subquery**：包含在 select 中的子查询（不在 from 子句中、出现在 `WHERE`、`SELECT`、`HAVING` 等位置）

```sql
mysql > EXPLAIN SELECT * FROM users WHERE id IN (SELECT user_id FROM orders);
-- 执行计划
-- PRIMARY        users
-- SUBQUERY       orders
```

-   **derived**：包含在 from 子句中的子查询、MySql 会将结果存放在一个临时表中，也成为衍生表（derived 的英文含义）

```sql
SELECT *
FROM (
    SELECT user_id, COUNT(*) c
    FROM orders
    GROUP BY user_id
) t
WHERE t.c > 10;

-- 执行计划
-- PRIMARY   <derived2>
-- DERIVED   orders
```



-   **union**：在 union 中的第二个和随后的 select . 如下：

```sql
mysql > explain select 1 union all select 1;
```

![image-20210622204636343](MySQL 进阶.assets/image-20210622204636343.png)

>   用以下例子来了解 primary 、 subquery 和 derived 类型的含义：

```sql
#关闭 mySql5.7 新特性对衍生表的合并优化
mysql > set session optimizer_switch = 'derived_merge = off';

mysql > 
explain select (select 1 from actor where id = 1) from (
    select * from film where id = 1 
) der;

# 还原默认配置
mysql > set session optimizer_switch = 'derived_merge=on'; 
```



#### 3、table 列



这一列表示 explain 的行正在访问哪个表、当 from 子句中有子查询时，table 列是 <derivedN> 格式，表示当前查询依赖 id = N 的查询

于是先执行 id = N 的查询当有 union 时， UNION RESULT 的 table 列的值为 <union 1, 2>。1 和 2 表示参与 untion 的 select 行 id



#### 4、type 列 (常用的)



这一列表示关联类型或访问类型，即 MySql 决定如何查找表中的列，查找数据行记录的大概范围依照性能优先级从最优到最差分别为：

```sql
system > const > eq_ref > ref > range > index > ALL
```



>   **一般来说得保证查询达到 range 级别、最好达到 ref 级别**



##### 1、NULL（性能最优）：



表示 mysql 能够在优化阶段分解查询语句，在执行阶段用不着再访问表或索引，**例如查询索引列的最小值，可以单独查找索引来完成**，并不需要在执行时访问表

```sql
mysql > explain select min(id) from film;
```

>   分析结果如下：

![image-20210622205338891](MySQL 进阶.assets/image-20210622205338891.png)



##### 2、const、system



MySql 能对查询的某部分进行优化并将其转化成一个常量，**用 primary key 或 unique key 的列与常数比较时**，**仅返回一条数据**，读取 1 次，速度比较快，**system 是 const 的特例，即表里只有一条元素时为 system**

```sql
mysql > explain extended select * from (select * from film where id = 1) tmp;
```

>   分析结果如下：

![image-20210622205502369](MySQL 进阶.assets/image-20210622205502369.png)



##### 3、eq_ref



**使用 primary key 或者 unique key 索引进行 Join 联合查询**、每次 JOIN **只返回一行**。这可能是在 const 之外最好的联接类型了，**大白话就是用到了表的主键进行关联联合查询的类型**

```sql
mysql> explain select * from film_actor left join film on film_actor.film_id = film.id;
```

>   分析结果如下：

![image-20210622205815659](MySQL 进阶.assets/image-20210622205815659.png)



##### 4、ref



使用 **普通索引等值匹配 或者 联合索引前缀匹配**，可能返回多行数据、如以下简单的查询语句，name 是普通索引

```sql
-- 假设(name, age) 是联合索引
mysql> EXPLAIN SELECT name, age FROM users;
-- 假设 age 为普通索引
mysql> SELECT * FROM users WHERE age = 20;
```

![image-20210622210015578](MySQL 进阶.assets/image-20210622210015578.png)



>   关联表查询，possible_keys 的值 idx_film_actor_id 是 film_id 和 actor_id 的联合索引，这里使用到了 film_actor 的左边前缀 film_id 部分

```sql
mysql >  explain select film_id from film 
                     left join film_actor on film.id = film_actor.film_id;
```

![image-20210622210302431](MySQL 进阶.assets/image-20210622210302431.png)



##### 5、range



**使用任何索引进行范围扫描**、通常出现在 in()、between、>、< 等操作中，需扫描部分索引，优于全索引扫描

```sql
mysql > explain select * from actor where id > 1
```

![image-20210622210429693](MySQL 进阶.assets/image-20210622210429693.png)



##### 6、index



**全索引扫描（**遍历整个索引**，但不会访问表数据**）、它比 `ALL`（全表扫描）要 **快**，因为索引通常比表数据小，可以减少 I/O 操作。**当查询的字段全部在索引中** 时，MySQL 可以**直接使用索引** 而不访问表（**索引覆盖**）

```sql
-- id 是主键。查询所有id
mysql > explain SELECT id FROM users;
-- age 是普通索引，查询所有 age
mysql > explain SELECT age FROM users;
```

![image-20210622210712318](MySQL 进阶.assets/image-20210622210712318.png)



film 表只有 id 和 name 两个字段、其中 id 是 主键索引、name 是普通（二级）索引，即该表所有字段都是索引，此时 key 列的值 是 idx_name 的二级索引

>   MySql 会进行优化，优先选择二级索引，这个叫做覆盖索引，因为二级索引的叶子节点存储的是主键索引的值 而不是 主键索引那样存储整一行的数据，这样不用扫全部列



##### 7、ALL

即全表扫描，扫描你的聚集索引的所有叶子节点、通常情况下这需要增加索引来进行优化了

```sql
-- 没有用索引做条件查询
mysql > explain select * from actor;
-- name 没有加索引. 全表扫描
mysql > SELECT * FROM users WHERE name = 'Tom';
```

![image-20210622210854197](MySQL 进阶.assets/image-20210622210854197.png)



#### 5、possible_key 列



**这一列显示查询可能使用哪些索引来查找**

explain 时可能出现 possible_keys 有列，而 key 显示 NULL 的情况，这种情况是因为表中数据不多，mysql 认为走索引对此提醒查询性能帮助不大，选择了全表查询;  如果该列是 NULL，则没有相关的索引、在这种情况下，可以通过检查 where 子句看是否可以创造一个适当的索引来提高查询性能，然后用 explain 查看效果



#### 6、key 列



**这一列显示 mySql 实际采用了哪个索引来优化对表的查询**，如果没有使用索引，则该列是 NULL，如果想强制 MySql 使用或者忽视 possible_keys 列中的索引，在查询中使用 forceindex、ignore index



#### 7、key_len 列



>   **MySQL 在查询中实际使用的索引长度（字节数）**、通过这个值可以算出具体使用了索引中的哪些列
>
>   注意：不是实际数据长度**而是索引字段的最大长度**(如果字段允许 `NULL` 额外增加 1 byte)

举例来说，film_actor 的联合索引 idx_film_actor_id 由 film_id 和 actor_id 两个 int 列组成，并且每个 int 是 4 字节、通过结果中的 key_len = 4、可推断出查询使用了第一个列：film_id 列来执行索引查找，如果都用到的话 key_len 就是 8 了。SQL 还要加个actor_id 的条件，把这个索引也用上

```sql
mysql > explain select * from film_actor where film_id = 2;
```

![image-20210622211109764](MySQL 进阶.assets/image-20210622211109764.png)



**key_len 计算规则如下：**

-   **字符串：**
    -   char(n)：n 个 byte长度
    -   varchar(n)：如果是 utf-8，则长度计算公式为 3n + 2 byte、加的 2 byte用来存储字符串

-   **数值类型：**
    -   tinyint：1 byte
    -   smallint：2 byte
    -   int：4 byte
    -   bigint：8 byte

-   **时间类型：**

    -   date：3 byte

    -   timestamp：4 byte

    -   datetime：8 byte

-   如果字段允许有 NULL 值，则额外需要 1 字节记录是否为 NULL


索引最大长度是 768 byte、当字符串过长时、MYSQL 会做一个类似左前缀索引的处理，将前半部分的字符串提取出来做索引



#### 8、ref 列



>   这一列显示了在 key 列记录的索引中、**表查找的时候实际所用到的列或常量**、常见的有：const（常量）、字段名（例：film.id）



#### 9、rows 列



>   这一列是 mysql **预估需要读取扫描的行数**、注意这个不是结果集里的行数



#### 10、extra 列 (常见的)



这一列展示的是额外信息、常见的重要值如下：



##### 1、Using index：



**索引覆盖定义**： **索引覆盖**（Covering Index），即查询所需的列全部包含在索引中，无需访问表数据（避免回表查询）。**覆盖索引一般针对的是辅助索引，整个查询结果只通过辅助索引就能拿到结果，不需要通过辅助索引树找到主键**，再通过主键去主键索引树里获取其它字段值

```sql
mysql > explain select film_id from film_actor where film_id = 1;
```

![image-20210622212433639](MySQL 进阶.assets/image-20210622212433639.png)



##### 2、Using where：



表示 **WHERE 过滤条件** 没有使用索引，而是在查询过程中 **逐行扫描并进行过滤**（可能导致全表扫描）（需要优化的）

```sql
mysql> explain select * from actor where name = 'a';
```

![image-20210622212555637](MySQL 进阶.assets/image-20210622212555637.png)



##### 3、Using index condition：



**索引条件推送（ICP，Index Condition Pushdown）** 优化：索引不能完全覆盖查询，但可以 **先通过索引过滤部分数据**，然后再回表查询剩余数据

```sql
EXPLAIN SELECT * FROM users WHERE age > 30 AND name = 'Alice';
```

> 如果 `age` 没有索引，而 `name` 有索引，可能会出现 `Using index condition`



##### 4、Using temporary：



>   - **查询使用了临时表**，通常出现在 **GROUP BY、ORDER BY 和 DISTINCT** 查询中
>   - **临时表可能存放于内存（HEAP）或磁盘（MyISAM/InnoDB）**，当数据过大时可能会写入磁盘，影响性能



如下 SQL 语句 name 没有建立索引、此时创建了张临时表来 distinct 进行去重对比

```sql
mysql> explain select distinct name from actor;
```

![image-20210622212738314](MySQL 进阶.assets/image-20210622212738314.png)



如下 SQL 语句 film.name 建立了 idx_name 索引，此时查询时 extra 时 using index, 没有用到临时表

```sql
mysql> explain select distinct name from film;
```

![image-20210622212814418](MySQL 进阶.assets/image-20210622212814418.png)



##### 5、Using filesort：



表示 **ORDER BY 语句未命中索引**，MySQL 需要使用外部排序（内存/磁盘），数据较小时从内存排序，否则需要在磁盘完成排序、这种情况下一般也是要考虑使用索引来优化的、如下SQL示例 

```sql
mysql > explain select * from actor order by name;
```

![image-20210622212956898](MySQL 进阶.assets/image-20210622212956898.png)

> name 未创建索引，会浏览 actor 整个表，保存关键字 name 和对应的 id，然后排序 name 并检索记录。如果将 actor.name 建立索引，此时执行计划 Extra 的值变成了 Using Index
>



##### 6、Select tables optimized away



使用某些聚合函数（ 比如 max、min ）来访问存在索引的某个字段时

```sql
mysql> explain select min(id) from film;
```

![image-20210622213127967](MySQL 进阶.assets/image-20210622213127967.png)



##### 7、Using join buffer (Block Nested Loop)



> - 适用 **JOIN 连接**，MySQL 需 **使用临时缓冲区存储数据** 以进行 **嵌套循环连接（BNL，Block Nested Loop）**
> - **当无法使用索引进行关联查询** 时，MySQL 会创建 join buffer，导致性能下降

```sql
-- 如果 customers.id 没有索引，则可能会出现 Using join buffer
EXPLAIN SELECT * FROM orders JOIN customers ON orders.customer_id = customers.id;
```



### 3. Explain Analyze (8.0)



> Explain Analyze(艾呢来兹) 是 MySQL 8.0 在性能分析方面最重要的增强，首次引入于 8.0.18 版本

- `EXPLAIN ANALYZE` 是一个用于调试的工具，它会**实际执行**你的查询语句，然后输出每个执行步骤（迭代器/算子）的实际执行信息和预估信息的对比
- **和传统 EXPLAIN 的区别**：
  - 传统的 `EXPLAIN` （包括 `FORMAT=JSON`）只输出优化器的**预估**成本（cost）和扫描行数（rows）。这是基于统计信息的估算，可能不准
  - `EXPLAIN ANALYZE` 则提供**实际**的执行时间（actual time）、实际处理的行数（actual rows）以及该步骤被循环执行的次数（loops）



下面将用 Explain Analyze 进行分析

```sql
EXPLAIN ANALYZE SELECT u.name, o.order_no 
FROM users u 
JOIN orders o ON u.id = o.user_id 
WHERE u.age > 25\G
```

输出

```sql
*************************** 1. row ***************************
EXPLAIN
-> Nested loop inner join  (cost=4.95 rows=9) (actual time=0.153..0.200 rows=9 loops=1)
    -> Filter: (u.age > 25)  (cost=2.83 rows=9) (actual time=0.097..0.100 rows=9 loops=1)
        -> Index scan on u using idx_age  (cost=2.83 rows=50) (actual time=0.045..2.345 rows=5000 loops=1)
    -> Index lookup on o using idx_user_id (user_id=u.id)  (cost=2.35 rows=1) (actual time=0.010..0.010 rows=1 loops=9)
```

**核心指标解析**：

- **`actual time=0.153..0.200`**：这里的两个数值，第一个是返回**第一行**的平均实际时间（毫秒），第二个是返回**所有行**的平均实际时间。这对于分析网络传输或延迟很有帮助。
- **`rows=9`** vs **`rows=5000`**：可以清晰看到，预估扫描行数（rows=9）和实际扫描行数（rows=5000）可能存在巨大差异，这就是为什么实际执行时间远超预期的根本原因
- **`loops=9`**：这个指标至关重要！它表明该步骤（Index lookup on o）被执行了 9 次。那么，这一步的实际总耗时就是 `0.010ms * 9 = 0.09ms`，总处理行数为 `1 * 9 = 9` 行



>  总结：MySQL 8.0 对 `EXPLAIN` 的升级，可以概括为从**“静态预估”**走向了**“动态诊断”**

| 功能点           | MySQL 5.7                                         | MySQL 8.0                                 | 核心价值                                 |
| :--------------- | :------------------------------------------------ | :---------------------------------------- | :--------------------------------------- |
| **核心命令**     | `EXPLAIN`                                         | `EXPLAIN` + **`EXPLAIN ANALYZE`**         | 提供实际执行时间与行数，精准定位性能瓶颈 |
| **输出格式**     | `TRADITIONAL`, `JSON`                             | `TRADITIONAL`, `JSON`, **`TREE`**         | 树状结构更清晰展示复杂查询执行流程       |
| **默认格式控制** | 无                                                | **`explain_format` 系统变量** （8.0.32+） | 可设置默认输出格式，提升使用便捷性       |
| **支持语句**     | `SELECT`, `DELETE`, `INSERT`, `REPLACE`, `UPDATE` | **+ `TABLE`** （8.0.19+）                 | 覆盖更多语句类型                         |



### 4. 初识索引的最佳实践



#### 1、准备阶段：创建表



-   创建员工表

```sql
-- ----------------------------
-- Table structure for employees
-- ----------------------------
DROP TABLE IF EXISTS `employees`;

CREATE TABLE `employees`  (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT 'ID',
  `name` varchar(24) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '姓名',
  `age` int(11) NOT NULL DEFAULT 0 COMMENT '年龄',
  `position` varchar(20) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '职位',
  `hire_time` timestamp(0) NOT NULL DEFAULT CURRENT_TIMESTAMP(0) COMMENT '入职日期',
  PRIMARY KEY (`id`) USING BTREE,
  INDEX `idx_name_age_position`(`name`, `age`, `position`) USING BTREE COMMENT '姓名_年龄_职业'
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

SET FOREIGN_KEY_CHECKS = 1;
```

-   测试数据

```sql
INSERT INTO employees(name, age, position, hire_time) values('Lee', 24, 'Java', NOW());
INSERT INTO employees(name, age, position, hire_time) values('ZhangSan', 22, 'C++', NOW());
INSERT INTO employees(name, age, position, hire_time) values('LiSi', 20, 'C#', NOW());
```

-   主键索引 id
-   联合索引 idx_name_age_position



>   其中 id 是主键索引、name、age、position 是联合索引



#### 2、全值匹配



```sql
# 走了联合索引的 name
EXPLAIN SELECT * FROM employees WHERE name = 'LiLei';
```

![image-20210623122525371](MySQL 进阶.assets/image-20210623122525371.png)



```sql
# 走了联合索引的 name 、age 索引
EXPLAIN SELECT * FROM employees WHERE name = 'LiLei' AND age = 22; 
```

![image-20210623122557208](MySQL 进阶.assets/image-20210623122557208.png)



```sql
# 走了联合索引的 name 、age、position 索引
EXPLAIN SELECT * FROM employees WHERE name = 'LiLei' AND age = 22 AND position='manager'; 
```

![image-20210623122636180](MySQL 进阶.assets/image-20210623122636180.png)



#### 3、最左前缀法则



>   如果索引了多列，要遵循最左前缀法则、指的是查询从索引的最左前列开始并且不跳过索引中的列

```sql
EXPLAIN SELECT * FROM employees WHERE name = 'Bill' and age = 31;    # 走了 name、age 索引
EXPLAIN SELECT * FROM employees WHERE age = 30 AND position = 'dev'; # 没走索引
EXPLAIN SELECT * FROM employees WHERE position = 'manager';          # 没走索引
```



#### 4、不在索引上做任何函数操作



>   例如：**不要在索引上做 计算、函数、自动或手动的类型转换**，会导致索引失效而转向全表扫描

```sql
# 正常走了 name 索引
EXPLAIN SELECT * FROM employees WHERE name = 'LiLei';

# 因为 left 函数截取了索引列的某些位数，导致无序可言，故不走索引
EXPLAIN SELECT * FROM employees WHERE left(name, 3) = 'LiLei';
```



>   我们看这样一个例子、先给 hire_time 加一个普通（二级）索引：

```sql
ALTER TABLE `employees` ADD INDEX `idx_hire_time` (`hire_time`) USING BTREE ;
```

**1、然后再去执行 explain 分析执行计划**

```sql
# 加了 date 函数之后未走任何索引
# 因为字段 hire_tie 经过data函数运算、破坏了原有的类型，在索引树中找不到原有的类型
EXPLAIN select * from employees where date(hire_time) = '2018‐09‐30';
```

![image-20210623123559482](MySQL 进阶.assets/image-20210623123559482.png)



**2、此时在以上基础上做一些优化、转化为日期范围查询，有可能会走索引**

-   至于优化后续章节会讲到 MYSQL 的一些优化策略，这里先有个印象

```sql
EXPLAIN select * from employees where hire_time >= '2018‐09‐30 00:00:00' and hire_time <= '2018‐09‐30 23:59:59' ;
```

>   我们看到该 SQL 分析的结果，possible_keys 列 mysql 认为可以走索引，但是最终没走索引，mysql 在走索引的时候，会对索引的效率做一些评估，但是如果全表扫描比索引还快一些（二级索引查询还需要回表扫描，效率会慢些），实际就不会走索引。

![image-20210623123625455](MySQL 进阶.assets/image-20210623123625455.png)



**3、删除刚刚加的 hire_time 上的索引、还原最初的状态**

```sql
ALTER TABLE `employees` DROP INDEX `idx_hire_time`;
```



#### 5、存储引擎不能使用索引中范围条件右边的列



```sql
# 正常走全部的三个联合索引
EXPLAIN SELECT * FROM employees WHERE name= 'LiLei' AND age = 22 AND position = 'manager' ;
```

```sql
EXPLAIN SELECT * FROM employees WHERE name= 'LiLei' AND age > 22 AND position = 'manager';
```

以上为什么以上只走了 name 和 age 两个索引？**因为第一个索引 name 相等的情况下，去做第二个索引的范围匹配，就破坏了第三个索引的有序性**，所以第三个不走索引

![image-20210623131131364](MySQL 进阶.assets/image-20210623131131364.png)



#### 6、尽量使用覆盖索引、减少 select * 的使用



>   尽量使用覆盖索引（只访问索引的查询（索引列包含查询列）），减少 select * 语句的使用



以下是不当用例：

```sql
EXPLAIN SELECT * FROM employees WHERE name= 'LiLei' AND age = 23 AND position = 'manager';
```

![image-20210623131419651](MySQL 进阶.assets/image-20210623131419651.png)



优化方案：

>   使用覆盖索引，就是 select 的时候如果需要的字段正好是索引字段，那么就尽量查询具体指定的列
>
>   覆盖索引的好处：我们直到二级索引叶子节点除了存储索引的值外，还存储了主键索引，如果查询到了 二级索引之外的字段，就需要回表扫描，这时如果只用到了二级索引或者联合索引的字段，就无需回表扫描了

```sql
EXPLAIN SELECT name, age FROM employees WHERE name = 'LiLei' AND age = 23 AND position = 'manager' ;
```

![image-20210623131651820](MySQL 进阶.assets/image-20210623131651820.png)



#### 7、减少 !=、<>、not in、not exists 的使用



>   MySql 在使用不等于 ( != 或者 <> )、not in、not exists、的时候无法使用索引会导致全表扫描，<、>、<=、>=、这些操作符 MySql 内部会根据检索比例，表大小等多个因素评估是否使用索引

```sql
# 该SQL 的 type 列 为 ALL、意味着走了全表扫描
EXPLAIN SELECT * FROM employees WHERE name != 'LiLei';
```

MySql 在使用以上操作符时，因为仅排除了一个值，剩下的数据都要返回，这样和全表扫描没有什么区别，走索引后的结果说不定还不如全表扫描的性能快、尤其是二级索引没有索引覆盖的时候还需要回表扫描

![image-20210623132215704](MySQL 进阶.assets/image-20210623132215704.png)



#### 8、Like 以通配符开头导致全表扫描



##### 1、Like 通配符问题



like 以通配符开头 （ '%abc...'）会导致 MySql 索引失效会变成全表扫描操作

```sql
EXPLAIN SELECT * FROM employees WHERE name like '%Lei';
```

虽然 name 是联合索引的最左前缀，但是模糊匹配 name 的后面几位截取破坏了索引的有序性，所以无法走索引

![image-20210623132638436](MySQL 进阶.assets/image-20210623132638436.png)



##### 2、优化方案



```sql
EXPLAIN SELECT * FROM employees WHERE name like 'Lei%';
```

将模糊查询的 % 号放在后面进行查找，这样在联合索引 name 列 就算取前几位也依然能保证有序性，继而可以走索引

![image-20210623132754168](MySQL 进阶.assets/image-20210623132754168.png)



##### 3、继续优化 Like '%字符串%' ？



使用覆盖索引、查询字段必须是建立覆盖索引字段，**如果不能使用覆盖索引则可能需要借助搜索引擎**，这样顶多全表扫描一遍二级索引树，无需回表扫描。

```sql
EXPLAIN SELECT name, age, position FROM employees WHERE name like '%Lei%';
```

![image-20210623132912281](MySQL 进阶.assets/image-20210623132912281.png)



#### 10、字符串不加单引号索引失效



##### 1、字符串不加单引号问题重现



```sql
# 我们发现 type = ALL，走了全表扫描
EXPLAIN SELECT * FROM employees WHERE name = 1000;
```

![image-20210623133241159](MySQL 进阶.assets/image-20210623133241159.png)



##### 2、优化方案



```sql
EXPLAIN SELECT * FROM employees WHERE name = '1000' ;
```

用到索引列进行条件匹配时使用索引列类型一样的类型，否则 MySql 后台会使用函数将索引列转换类型，导致在B+ Tree 中找不到对应类型，一旦后台索引列加了函数就会失效

![image-20210623133453577](MySQL 进阶.assets/image-20210623133453577.png)



#### 11、少用 or 或 in



>   用它查询时，mysql 不一定使用索引，mysql 内部优化器会根据检索比例、表大小等多个因素整体评估是否使用索引，详见范围查询优化

```sql
EXPLAIN SELECT * FROM employees WHERE name = 'LiLei' or name = 'HanMeimei';
```

![image-20210623133653854](MySQL 进阶.assets/image-20210623133653854.png)



#### 12、范围查找优化



##### 1、范围查找问题在现



给年龄添加单个（普通）索引

```sql
ALTER TABLE `employees` ADD INDEX `idx_age` (`age`) USING BTREE ;
```



执行 explain 分析执行计划

```sql
explain select * from employees where age >= 1 and age <= 2000;
```

没走索引原因：mysql 内部优化器会根据检索比例.表大小等多个因素整体评估是否使用索引. 比如这个例子, 可能是由于单次数据量查询过大导致优化器最终选择不走索引

![image-20210623134121107](MySQL 进阶.assets/image-20210623134121107.png)



##### 2、优化方案



>   可以将大的范围拆分成多个范围

```sql
explain select * from employees where age >=1 and age <=1000;
explain select * from employees where age >=1001 and age <=2000;
```

![image-20210623134320816](MySQL 进阶.assets/image-20210623134320816.png)



还原最初索引状态

```sql
ALTER TABLE `employees` DROP INDEX `idx_age`;
```



#### 13、索引使用总结



![image-20210623134357747](MySQL 进阶.assets/image-20210623134357747.png)



>   like KK% 相当于 = 常量，%KK 和 %KK% 相当于范围
>

‐‐ mysql 5.7 关闭 ONLY_FULL_GROUP_BY 报错

```sql
select version(), @@sql_mode; SET sql_mode = (
    SELECT REPLACE (@@sql_mode , 'ONLY_FULL_GROUP_BY' , '' )
);
```



## 4. MySql 索引优化实战篇



### 0. 测试数据准备阶段



#### 1、联合索引测试脚本

```sql
-- 创建测试表 employees
CREATE TABLE `employees` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(24) NOT NULL DEFAULT '' COMMENT '姓名',
  `age` int(11) NOT NULL DEFAULT '0' COMMENT '年龄',
  `position` varchar(20) NOT NULL DEFAULT '' COMMENT '职位',
  `hire_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '入职时间',
  PRIMARY KEY (`id`),
  KEY `idx_name_age_position` (`name`,`age`,`position`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8 COMMENT='员工记录表';

-- 插入测试数据
INSERT INTO employees(name,age,position,hire_time) VALUES('LiLei',22,'manager',NOW());
INSERT INTO employees(name,age,position,hire_time) VALUES('HanMeimei', 23,'dev',NOW());
INSERT INTO employees(name,age,position,hire_time) VALUES('Lucy',23,'dev',NOW());

-- 循环插入 10 万条示例数据
drop procedure if exists insert_emp; 
delimiter ;;
create procedure insert_emp()        
begin
  declare i int;                    
  set i=1;                          
  while(i<=100000)do                 
    insert into employees(name,age,position) values(CONCAT('lee', i), i, 'dev');  
    set i=i+1;                       
  end while;
end;;
delimiter ;
call insert_emp();
```



#### 2、Join 关联查询脚本

```sql
-- 示例表：
CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_a` (`a`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

create table t2 like t1;

-- 插入一些示例数据
-- 往t1表插入1万行记录
drop procedure if exists insert_t1; 
delimiter ;;
create procedure insert_t1()        
begin
  declare i int;                    
  set i=1;                          
  while(i<=10000)do                 
    insert into t1(a,b) values(i,i);  
    set i=i+1;                       
  end while;
end;;
delimiter ;
call insert_t1();

-- 往t2表插入100行记录
drop procedure if exists insert_t2; 
delimiter ;;
create procedure insert_t2()        
begin
  declare i int;                    
  set i=1;                          
  while(i<=100)do                 
    insert into t2(a,b) values(i,i);  
    set i=i+1;                       
  end while;
end;;
delimiter ;
call insert_t2();
```



### 1. MySQL选择合适索引的策略？



#### 1、大范围索引失效情况分析

```sql
# 走了全表扫描
EXPLAIN select * from employees where name > 'a';
```

![image-20210623141113133](MySQL 进阶.assets/image-20210623141113133.png)



如果用 name 索引需要遍历 name 字段联合索引树，并且这个范围太大，除了比字母 a 其余26个字母的字段排序都需要扫描，然后还需要根据遍历出来的主键值去主键索引树里再去查出最终数据，成本比全表扫描还高，可以用覆盖索引优化这样只需要遍历 name 字段的联合索引树就能拿到所有结果，如下



#### 2、优化方案



```sql
# 使用联合索引的索引覆盖字段
EXPLAIN select name,age,position from employees where name > 'a';
```

![image-20210623141535571](MySQL 进阶.assets/image-20210623141535571.png)



```sql
# 缩小查找范围
EXPLAIN select * from employees where name > 'zzz' ;
```

![image-20210623141555659](MySQL 进阶.assets/image-20210623141555659.png)



#### 3、MySQL Trace 分析工具



对于上面这两种 name > 'a' 和 name > 'zzz' 的执行结果，mysql 最终是否选择走索引或者一张表涉及多个索引，mysql 最终如何选择索引，我们可以用 trace工具来一查究竟，开启 trace 工具会影响 mysql 性能，所以只能临时分析 sql 使用，用完之后立即关闭

>   **trace 工具详解：**
>
>   Trace 工具可以很好的帮助我们分析 MySQL 究竟如何选择索引，
>
>   -   第一阶段时，MySQL 会格式化 SQL
>   -   第二阶段时，MySQL 会对我们写的SQL语句进行优化
>   -   之后 MySQL 会分析表依赖关系、预估表的整体访问成本、这个阶段大致分为
>       -   **预估全表扫描情况、扫描行数、查询成本**
>       -   **预估使用到的索引类型、索引的适用范围、是否使用覆盖索引，索引扫描行数、索引使用成本**
>       -   **除此之外还会记录表的最优访问记录、最终选择的访问路径等策略**

```sql
mysql> set session optimizer_trace="enabled=on",end_markers_in_json=on;  --开启trace
mysql> select * from employees where name > 'a' order by position;
mysql> SELECT * FROM information_schema.OPTIMIZER_TRACE;
```

>   查看 trace 字段详情：

```sql
{
  "steps": [
    {
      "join_preparation": {    --第一阶段：SQL准备阶段，格式化sql
        "select#": 1,
        "steps": [
          {
            "expanded_query": "/* select#1 */ select `employees`.`id` AS `id`,`employees`.`name` AS `name`,`employees`.`age` AS `age`,`employees`.`position` AS `position`,`employees`.`hire_time` AS `hire_time` from `employees` where (`employees`.`name` > 'a') order by `employees`.`position`"
          }
        ] /* steps */
      } /* join_preparation */
    },
    {
      "join_optimization": {    --第二阶段：SQL优化阶段
        "select#": 1,
        "steps": [
          {
            "condition_processing": {    --条件处理
              "condition": "WHERE",
              "original_condition": "(`employees`.`name` > 'a')",
              "steps": [
                {
                  "transformation": "equality_propagation",
                  "resulting_condition": "(`employees`.`name` > 'a')"
                },
                {
                  "transformation": "constant_propagation",
                  "resulting_condition": "(`employees`.`name` > 'a')"
                },
                {
                  "transformation": "trivial_condition_removal",
                  "resulting_condition": "(`employees`.`name` > 'a')"
                }
              ] /* steps */
            } /* condition_processing */
          },
          {
            "substitute_generated_columns": {
            } /* substitute_generated_columns */
          },
          {
            "table_dependencies": [    --表依赖详情
              {
                "table": "`employees`",
                "row_may_be_null": false,
                "map_bit": 0,
                "depends_on_map_bits": [
                ] /* depends_on_map_bits */
              }
            ] /* table_dependencies */
          },
          {
            "ref_optimizer_key_uses": [
            ] /* ref_optimizer_key_uses */
          },
          {
            "rows_estimation": [    --预估表的访问成本
              {
                "table": "`employees`",
                "range_analysis": {
                  "table_scan": {     --全表扫描情况
                    "rows": 10123,    --扫描行数
                    "cost": 2054.7    --查询成本
                  } /* table_scan */,
                  "potential_range_indexes": [    --查询可能使用的索引
                    {
                      "index": "PRIMARY",    --主键索引
                      "usable": false,
                      "cause": "not_applicable"
                    },
                    {
                      "index": "idx_name_age_position",    --辅助索引
                      "usable": true,
                      "key_parts": [
                        "name",
                        "age",
                        "position",
                        "id"
                      ] /* key_parts */
                    }
                  ] /* potential_range_indexes */,
                  "setup_range_conditions": [
                  ] /* setup_range_conditions */,
                  "group_index_range": {
                    "chosen": false,
                    "cause": "not_group_by_or_distinct"
                  } /* group_index_range */,
                  "analyzing_range_alternatives": {    --分析各个索引使用成本
                    "range_scan_alternatives": [
                      {
                        "index": "idx_name_age_position",
                        "ranges": [
                          "a < name"      --索引使用范围
                        ] /* ranges */,
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": false,    --使用该索引获取的记录是否按照主键排序
                        "using_mrr": false,
                        "index_only": false,       --是否使用覆盖索引
                        "rows": 5061,              --索引扫描行数
                        "cost": 6074.2,            --索引使用成本
                        "chosen": false,           --是否选择该索引
                        "cause": "cost"
                      }
                    ] /* range_scan_alternatives */,
                    "analyzing_roworder_intersect": {
                      "usable": false,
                      "cause": "too_few_roworder_scans"
                    } /* analyzing_roworder_intersect */
                  } /* analyzing_range_alternatives */
                } /* range_analysis */
              }
            ] /* rows_estimation */
          },
          {
            "considered_execution_plans": [
              {
                "plan_prefix": [
                ] /* plan_prefix */,
                "table": "`employees`",
                "best_access_path": {    --最优访问路径
                  "considered_access_paths": [   --最终选择的访问路径
                    {
                      "rows_to_scan": 10123,
                      "access_type": "scan",     --访问类型：为scan，全表扫描
                      "resulting_rows": 10123,
                      "cost": 2052.6,
                      "chosen": true,            --确定选择
                      "use_tmp_table": true
                    }
                  ] /* considered_access_paths */
                } /* best_access_path */,
                "condition_filtering_pct": 100,
                "rows_for_plan": 10123,
                "cost_for_plan": 2052.6,
                "sort_cost": 10123,
                "new_cost_for_plan": 12176,
                "chosen": true
              }
            ] /* considered_execution_plans */
          },
          {
            "attaching_conditions_to_tables": {
              "original_condition": "(`employees`.`name` > 'a')",
              "attached_conditions_computation": [
              ] /* attached_conditions_computation */,
              "attached_conditions_summary": [
                {
                  "table": "`employees`",
                  "attached": "(`employees`.`name` > 'a')"
                }
              ] /* attached_conditions_summary */
            } /* attaching_conditions_to_tables */
          },
          {
            "clause_processing": {
              "clause": "ORDER BY",
              "original_clause": "`employees`.`position`",
              "items": [
                {
                  "item": "`employees`.`position`"
                }
              ] /* items */,
              "resulting_clause_is_simple": true,
              "resulting_clause": "`employees`.`position`"
            } /* clause_processing */
          },
          {
            "reconsidering_access_paths_for_index_ordering": {
              "clause": "ORDER BY",
              "steps": [
              ] /* steps */,
              "index_order_summary": {
                "table": "`employees`",
                "index_provides_order": false,
                "order_direction": "undefined",
                "index": "unknown",
                "plan_changed": false
              } /* index_order_summary */
            } /* reconsidering_access_paths_for_index_ordering */
          },
          {
            "refine_plan": [
              {
                "table": "`employees`"
              }
            ] /* refine_plan */
          }
        ] /* steps */
      } /* join_optimization */
    },
    {
      "join_execution": {    --第三阶段：SQL执行阶段
        "select#": 1,
        "steps": [
        ] /* steps */
      } /* join_execution */
    }
  ] /* steps */
}
```

>   结论：全表扫描的成本低于索引扫描，所以 mysql 最终选择全表扫描

```sql
mysql> select * from employees where name > 'zzz' order by position;
mysql> SELECT * FROM information_schema.OPTIMIZER_TRACE;

查看trace字段可知索引扫描的成本低于全表扫描，所以mysql最终选择索引扫描

mysql> set session optimizer_trace="enabled=off";    -- 关闭trace
```



### 2. 联合索引常见优化方案



#### 1、联合索引首字段用范围不走索引



```sql
EXPLAIN SELECT * FROM employees WHERE name > 'LiLei' AND age = 22 AND position ='manager';
```

![image-20210623143641849](MySQL 进阶.assets/image-20210623143641849.png)



>   结论：联合索引第一个字段用范围查找不会走索引，MySQL 内部可能觉得第一个字段就用范围、结果集应该很大、回表率很高、还不如就用全表扫描



#### 2、强制走索引



-   使用 force index(索引名) 来强制走索引

```sql
EXPLAIN SELECT * FROM employees force 
    index(idx_name_age_position) WHERE name > 'LiLei' AND age = 22 AND position ='manager';
```



>   结论：虽然使用了强制走索引让联合索引第一个字段范围查找也走索引，扫描的行 rows 看上去也少了点，但是最终查找效率不一定比全表扫描高，因为回表效率不高

做了一个小实验：

```sql
-- 关闭查询缓存
set global query_cache_size=0;  
set global query_cache_type=0;
-- 执行时间0.333s
SELECT * FROM employees WHERE name > 'LiLei';
-- 执行时间0.444s
SELECT * FROM employees force index(idx_name_age_position) WHERE name > 'LiLei';
```



#### 3、索引覆盖优化



```sql
EXPLAIN SELECT name, age, position FROM employees 
           WHERE name > 'LiLei' AND age = 22 AND position ='manager';
```

![image-20210623144955400](MySQL 进阶.assets/image-20210623144955400.png)



#### 4、in 和 or 在表数据量比较大会走索引



>   in 和 or 在表数据量比较大的情况会走索引，在表记录不多的情况下会选择全表扫描

```sql
EXPLAIN SELECT * FROM employees 
    WHERE name in ('LiLei','HanMeimei','Lucy') AND age = 22 AND position='manager';
```

![image-20210623145107467](MySQL 进阶.assets/image-20210623145107467.png)



```sql
EXPLAIN SELECT * FROM employees 
    WHERE (name = 'LiLei' or name = 'HanMeimei') AND age = 22 AND position='manager';
```

![image-20210623145152709](MySQL 进阶.assets/image-20210623145152709.png)



做一个小实验、如果将 employees 表只留三条记录、在执行以上的SQL、数据量不大就会走全表扫描：如下

-   继续执行上面两条 SQL 的分析、我们发现 MySQL 最终走了全表扫描

![image-20210623145326640](MySQL 进阶.assets/image-20210623145326640.png)



#### 5、Like 'Lee%' 一般情况会走索引



##### 1、Like 走索引的案例再现



```sql
EXPLAIN SELECT * FROM employees 
     WHERE name like 'Lee%' AND age = 22 AND position ='manager';
```

![image-20210623145511979](MySQL 进阶.assets/image-20210623145511979.png)



##### 2、重要的知识点：索引下推



>   这里涉及一个知识点：索引下推（Index Condition Pushdown, ICP）、Like Lee% 其实就是用到了索引下推优化



##### 3、那么什么是索引下推？



> 索引下推（ICP，Index Condition Pushdown）是一种 MySQL **查询优化技术**，用于 **减少回表（回表查询）**，提高查询效率。它的核心思想是：
> **在索引遍历过程中尽可能多地过滤数据，减少回表（访问原始数据行）的次数**
>
> 注意：索引下推会减少回表次数，对于 innodb 引擎的表 **索引下推只能用于二级、联合索引 **，对于 innodb 的主键索引（聚簇索引）叶子节点上保存的是全行数据，这时候索引下推意义不大

```sqlite
EXPLAIN SELECT * FROM users WHERE name = 'Alice' AND age > 25;
```

**MySQL 5.5 及以前（未使用索引下推）：**

- 先使用索引 `idx_name_age` 找到 **所有 name='Alice' 的记录**
- **回表** 获取 `age` 的数据
- 过滤 `age > 25` 的记录

**问题：**

- 如果 `name = 'Alice'` 这个条件匹配了 **1000 行数据**，但 `age > 25` 只有 **10 行满足**，那么 **MySQL 仍然需要回表 1000 次**，导致大量 **IO 操作**，查询性能下降



**MySQL 5.6 及以上版本支持索引下推优化，优化后查询过程如下：**

- **先使用索引 `idx_name_age` 找到 name='Alice' 的记录**
- **在索引层直接应用 `age > 25` 过滤条件**，减少不必要的回表操作
- **只有真正符合 name='Alice' AND age>25 的数据才会回表**

**优化结果：**

- **减少回表次数，提高查询速度**
- **索引被充分利用，减少磁盘 IO**



##### 4、为什么范围查找没有用到索引下推优化？



估计应该是 MySql 认为范围查找过滤的结果集过大，like Lee% 在绝大多数情况来看，过滤后的结果集比较小，所以这里 Mysql 选择给 like Lee% 用了索引下推优化，当然这也不是绝对的，有时 like Lee% 也不一定就会走索引下推



### 3. Count(*) 的查询优化方案



#### 1、Count(*) 的优化尽量遵循以下



```sql
-- 临时关闭 mysql 查询缓存，为了查看 sql 多次执行的真实时间
mysql> set global query_cache_size=0;
mysql> set global query_cache_type=0;

mysql> EXPLAIN select count(1) from employees;
mysql> EXPLAIN select count(id) from employees;
mysql> EXPLAIN select count(name) from employees;
mysql> EXPLAIN select count(*) from employees;
```

注意：以上 4 条 Sql 只有根据某个字段的 Count 不会统计字段为 Null 值的数据行

对于以上几种情况要分开来讲：



>   字段有索引：count(*) ≈ count(1) > count(字段) > count(主键 id)
>

解析：字段有索引 count(字段) 统计走二级索引、二级索引存储数据比主键索引少、所以 count(字段) > count(主键 id)

>   字段无索引：count(*) ≈ count(1) > count(主键 id) > count(字段)
>

解析：字段没有索引 count(字段) 统计走不了索引、count(主键 id) 还可以走主键索引、所以 count(主键 id) > count(字段)



Count(1) 跟 Count(字段) 执行过程类似、不过 Count(1) 不需要取字段统计 / 就用常量 1 做统计。Count(字段) 还需要取出来字段，所以理论上 Count(1) 比 Count(字段) 会快一点

Count(*) 是例外、MySql 并不会把全部字段取出来、而是专门做了优化，不取值，按行累加、效率很高、所以不需要用 Count(列名) 或者 Count(常量) 来替换 Count(\*)



>   为什么对于 Count(id)、MySQL 最终选择辅助索引而不是主键聚集索引？
>

因为二级索引相对主键索引存储数据更少、检索性能应该更高。MySQL 内部做了点优化（因该是5.7版本才优化）



#### 2、常见的优化办法



##### 1、查询 MySQL 自己维护的总行数



对于 myisam 存储引擎的表做不带 where 条件的 count 查询性能是很高的，因为 myisam 存储引擎的表的总行数会被 mysql 存储在磁盘上，查询不需要计算

![image-20210623164101159](MySQL 进阶.assets/image-20210623164101159.png)



对于 innodb 存储引擎的表 mysql 不会存储表的总记录行数 (因为有MVCC机制，后面会讲)，查询 count 需要实时计算



##### 2、show table status



如果只需要知道表总行数的估计值可以用以下 SQL 查询、性能很高

![image-20210623164143767](MySQL 进阶.assets/image-20210623164143767.png)



##### 3、将总数维护到 Redis 中



插入或删除表数据行的时候同时维护 redis 里的表总行数 key 的计数值 (用 incr 或 decr 命令)，但是这种方式可能不准，很难保证表操作和 redis 操作的事务一致性



##### 4、增加数据库计数表



插入或删除表数据行的时候同时维护计数表，让他们在同一个事务里操作



### 4. Using filesort 文件排序原理



#### 1、filesort 文件排序方式



##### 1、单路排序：



**是一次性取出满足条件行的所有字段，然后在 sort buffer 中进行排序**；用 trace工具可以看到 sort_mode 信息里显示 < sort_key, additional_fields > 或者 < sort_key,packed_additional_fields >



##### 2、双路排序(回表排序)：



是首先根据相应的条件取出相应的 **排序字段** 和 **可以直接定位行数据的行 ID** ，然后在 sort buffer 中进行排序，排序完后需要再次取回其它需要的字段；用 trace工具可以看到 sort_mode 信息里显示 <sort_key, rowid>



#### 2、如何得知 MySql 实际使用哪种排序



>   MYSQL 通过比较系统变量 max_leng_for_sort_data（默认 1024 byte）的大小和查询的字段总大小判断使用哪种排序模式

-   如果 字段的总长度小于 max_length_for_sort_data，那么使用 单路排序模式
-   如果 字段的总长度大于 max_length_for_sort_data，那么使用 双路排序模式



查看下这条 SQL 对应的 Trace 结果如下（只展示排序部分）：

```sql
mysql> set session optimizer_trace="enabled=on",end_markers_in_json=on;  --开启trace
mysql> select * from employees where name = 'lee' order by position;
mysql> select * from information_schema.OPTIMIZER_TRACE;

trace排序部分结果：
"join_execution": {    --Sql执行阶段
"select#": 1,
"steps": [
  {
    "filesort_information": [
      {
        "direction": "asc",
        "table": "`employees`",
        "field": "position"
      }
    ] /* filesort_information */,
    "filesort_priority_queue_optimization": {
      "usable": false,
      "cause": "not applicable (no LIMIT)"
    } /* filesort_priority_queue_optimization */,
    "filesort_execution": [
    ] /* filesort_execution */,
    "filesort_summary": {                      --文件排序信息
      "rows": 10000,                           --预计扫描行数
      "examined_rows": 10000,                  --参与排序的行
      "number_of_tmp_files": 3,                --使用临时文件的个数，这个值如果为0代表全部使用的sort_buffer内存排序，否则使用的磁盘文件排序
      "sort_buffer_size": 262056,              --排序缓存的大小，单位Byte
      "sort_mode": "<sort_key, packed_additional_fields>"       --排序方式，这里用的单路排序
    } /* filesort_summary */
  }
] /* steps */
} /* join_execution */

mysql> set max_length_for_sort_data = 10;    --employees表所有字段长度总和肯定大于10字节
mysql> select * from employees where name = 'lee' order by position;
mysql> select * from information_schema.OPTIMIZER_TRACE;

trace排序部分结果：
"join_execution": {
"select#": 1,
"steps": [
  {
    "filesort_information": [
      {
        "direction": "asc",
        "table": "`employees`",
        "field": "position"
      }
    ] /* filesort_information */,
    "filesort_priority_queue_optimization": {
      "usable": false,
      "cause": "not applicable (no LIMIT)"
    } /* filesort_priority_queue_optimization */,
    "filesort_execution": [
    ] /* filesort_execution */,
    "filesort_summary": {
      "rows": 10000,
      "examined_rows": 10000,
      "number_of_tmp_files": 2,
      "sort_buffer_size": 262136,   
      "sort_mode": "<sort_key, rowid>"         --排序方式，这里用的双路排序
    } /* filesort_summary */
  }
] /* steps */
} /* join_execution */


mysql> set session optimizer_trace="enabled=off";    --关闭trace
```



#### 3、单路排序详细过程



-   从索引 name 找到第一个满足 name = 'lee' 条件中的主键 id
-   根据主键 id 取出整行，取出所有字段的值，存入 sort_buffer 中
-   从索引name找到下一个满足 name = ‘lee’ 条件的主键 id
-   重复步骤 2、3 直到不满足 name = ‘lee’
-   对 sort_buffer 中的数据按照字段 position 进行排序
-   返回结果给客户端



#### 4、双路排序的详细过程



-   从索引 name 找到第一个满足 name = ‘lee’ 的主键id
-   根据主键 id 取出整行，把排序字段 position 和主键 id 这两个字段放到 sort buffer 中
-   从索引 name 取下一个满足 name = ‘lee’ 记录的主键 id
-   重复 3、4 直到不满足 name = ‘lee’
-   对 sort_buffer 中的字段 position 和主键 id 按照字段 position 进行排序
-   遍历排序好的 id 和字段 position，按照 id 的值回到原表中取出 所有字段的值返回给客户端



#### 5、两个排序模式的对比



其实对比两个排序模式，单路排序会把所有需要查询的字段都放到 sort buffer 中，而双路排序只会把主键和需要排序的字段放到 sort buffer 中进行排序，然后再通过主键回到原表查询需要的字段。如果 MySQL 排序内存 sort_buffer 配置的比较小并且没有条件继续增加了，可以适当把 max_length_for_sort_data 配置小点，让优化器选择使用双路排序算法，可以在 sort_buffer 中一次排序更多的行，只是需要再根据主键回到原表取数据

如果 MySQL 排序内存有条件可以配置比较大，可以适当增大 max_length_for_sort_data 的值，让优化器优先选择全字段排序(单路排序)，把需要的字段放到 sort_buffer 中这样排序后就会直接从内存里返回查询结果了

所以，MySQL 通过 max_length_for_sort_data 这个参数来控制排序，在不同场景使用不同的排序模式，从而提升排序效率。注意，如果全部使用 sort_buffer 内存排序一般情况下效率会高于磁盘文件排序，但不能因为这个就随便增大sort_buffer (默认1M)，**mysql 很多参数设置都是做过优化的，不要轻易调整**



### 5. Order By 和 Group By 优化



#### 1、Case 1



```sql
EXPLAIN select * from employees where name = 'lee' and position = 'dev' order by age;
```

![image-20210623150937460](MySQL 进阶.assets/image-20210623150937460.png)

利用最左前缀法则：

中间字段不能断，因此查询只用到了 name 索引（在 name 索引相等的情况下第二个 age 是有序的，所以可以走索引），从 key_len = 74 **也能看出 age 索引列用在排序过程中**，因为 Extra 字段里没有 using filesort



#### 2、Case 2



```sql
EXPLAIN select * from employees where name = 'lee' order by position;
```

![image-20210623152259340](MySQL 进阶.assets/image-20210623152259340.png)



分析：从分析结果来看、key_len = 74，查询使用了 name 索引，由于用了 position 进行排序，跳过了 age，所以出现了 Using filesort



#### 3、Case 3



```sql
EXPLAIN select * from employees where name = 'lee' order by age, position;
```

![image-20210623152449519](MySQL 进阶.assets/image-20210623152449519.png)

分析：查找只用到了 name, 在 name 有序的情况下，后两位也可以保持有序，所以 age 和 position 用于排序、**无 Using filesort**



#### 4、Case 4



```sql
EXPLAIN select * from employees where name = 'lee' order by position, age;
```

![image-20210623153738040](MySQL 进阶.assets/image-20210623153738040.png)

分析：

和 Case 3 中的 explain 的结果一样、但是出现了 Using filesort、因为索引的创建顺序为 name、age、position、但是排序的时候 age 和 position 颠倒位置了



#### 5、Case 5



```sql
EXPLAIN Select * from employees where name = 'lee' and age = 18 order by position,age;
```

![image-20210623153842599](MySQL 进阶.assets/image-20210623153842599.png)

分析：与 Case 4 对比，在 Extra 中并未出现 Using filesort，因为age为常量（年龄都是 18 还根据 age 排什么序？），在排序中被优化，所以索引未颠倒，不会出现 Using filesor



#### 6、Case 6



```sql
EXPLAIN Select * from employees where name = 'lee' order by age asc, position desc;
```

![image-20210623154035437](MySQL 进阶.assets/image-20210623154035437.png)

分析：虽然排序的字段列与索引顺序一样，且 order by 默认升序，这里 position desc 变成了降序，导致与索引的排序方式不同导致结果集无序，从而产生 Using filesort、Mysql 8 以上版本有降序索引可以支持该种查询方式



#### 7、Case 7



```sql
EXPLAIN Select * from employees where name in ('LiLei', 'zhuge') order by age, position;
```

![image-20210623154120548](MySQL 进阶.assets/image-20210623154120548.png)

分析：对于排序来说，多个相等条件也是范围查询



#### 8、Case 8



```sql
EXPLAIN Select * from employees where name > 'a' order by name;
```

![image-20210623154412873](MySQL 进阶.assets/image-20210623154412873.png)



分析：范围查找一般 MySql 经过分析结果集大还不如走全表扫描、一般的优化方案是用覆盖索引优化

```sql
EXPLAIN Select name, age, position from employees where name > 'a' order by name;
```



#### 9、Order by 和 Group by 优化总结



1.  MySql支持两种方式的排序 filesort 和 index、Using index 是指 MYSQL 扫描索引本身完成排序、性能很好，而 filesort 是指不能通过索引进行排序、性能低下
2.  order by 满足两种情况会使用 Using Index 排序
    -   order by 语句使用索引最左前列
    -   使用 where 子句与 order by 子句条件列组合满足索引最左前列
3.  尽量在索引树上完成排序、遵循索引建立（索引创建的顺序）时的最左前缀法则
4.  如果 order by 的条件不在索引列上，就会产生 Using filesort
5.  能用覆盖索引尽量用覆盖索引
6.  group by 和 order by 很类似、实质就是先排序后分组、遵循索引创建顺序的最左前缀法则、对于 group by 的优化如果不需要排序的可以加上 order by null 禁止排序

>   注意，where 高于 having，能写在 where 中的限定条件就不要去 having 限定了



### 6. 分页查询优化实例



#### 1、分页查询如何提高性能？



>   很多时候我们业务系统实现分页功能可能会用到如下SQL

```sql
EXPLAIN select * from employees Limit 10000, 10;
```

表示从表 employees 中取出从 100001 开始的 10 行记录，看似只查询了 10 条记录，实际这条 SQL 是先读取 10010 条记录，然后抛弃前 10000 条记录，然后读到后面的 10 条想要的数据，因此查询一张数据量大的表比较靠后的数据、执行效率是非常低的、那么如何进行优化呢？请看如下



#### 2、常见的分页场景优化技巧



##### 1、根据自增且连续的主键排序的分页查询



首先来看一个根据自增连续主键排序的分页查询的例子：

```sql
Select * from employees limit 90000, 5;
```



该 SQL 表示查询从第 90001 开始的五行数据，没添加单独 order by，表示通过主键排序。我们再看表 employees ，因为主键是自增并且连续的，所以可以改写成 **按照主键去查询从第 90001 开始的五行数据** ，如下

```sql
Select * from employees where id > 90000 limit 5;
```



查询结果是一致的、我们在对比一下执行计划：

>   第一条SQL的执行计划：

```sql
mysql> EXPLAIN select * from employees limit 90000,5;
```

![image-20210623160852680](MySQL 进阶.assets/image-20210623160852680.png)



>   第二条SQL的执行计划：

```sql
mysql> EXPLAIN select * from employees where id > 90000 limit 5;
```

![image-20210623160902314](MySQL 进阶.assets/image-20210623160902314.png)

显然改写后的SQL走了索引、而且扫描的行数大大减少、但是，这条改写的 SQL 在很多场景并不实用，因为表中可能某些记录被删后，主键空缺，导致结果不一致。所以这种改写得满足以下两个条件：

-   主键自增且连续
-   结果是按照主键排序的



##### 2、根据非主键字段排序的分页查询



再看一个根据非主键字段排序的分页查询，SQL 如下：

```sql
EXPLAIN Select * from employees ORDER BY name limit 90000, 5;
```

![image-20210623161111347](MySQL 进阶.assets/image-20210623161111347.png)

发现并没有使用 name 字段的索引（key 字段对应的值为 null），具体原因上节课讲过：扫描整个索引并查找到没索引的行 ( 可能要遍历多个索引树 ) 的成本比扫描全表的成本更高，所以优化器放弃使用索引



>   知道不走索引的原因，那么怎么优化呢？

其实关键是让 **排序时返回的字段尽可能少**，所以可以让 排序 和 分页操作 先查出主键，然后根据主键查到对应的记录，SQL 改写如下

```sql
Select * from employees e inner join (
    select id from employees order by name limit 90000, 5
) f on e.id = f.id;
```

![image-20210623161450944](MySQL 进阶.assets/image-20210623161450944.png)



需要的结果与原 SQL 一致、执行时间减少了一半以上、我们再对比优化前后的 SQL 的执行计划、可以发现原来的 SQL 使用的是 filesort 排序，而优化后的 SQL 使用的是索引排序



##### 3、使用索引优化分页查询



- **问题**：分页查询通常涉及到 `ORDER BY` 操作，如果排序字段没有索引，会导致全表扫描，性能大幅下降



##### 4、**避免使用 `OFFSET`**



- **问题**：使用 `OFFSET` 作为偏移量时，数据库仍然会扫描之前的数据，这会导致性能逐渐变差，尤其是分页页数较大时



##### 5、分页查询的数据预加载与缓存



- **问题**：对于数据量非常大的场景，分页查询可能会导致响应时间变长
- **解决方案**：若数据修改不频繁、可以在后台定期预加载数据，将分页查询的结果缓存起来，减少数据库压力



##### 6、限制查询的最大页数



- **问题**：如果用户可以选择任意页数进行查询，在数据量非常大的情况下，查询的性能可能极其低下
- **解决方案**：可以设置一个**最大页数限制**，防止用户查询超大页数导致性能问题
- **实际场景**：比如只展示近半年的数据、近几个月的数据等等



##### 7、数据分区与分表



- **问题**：对于非常大的表，分页查询时可能会遇到性能瓶颈
- **解决方案**：使用数据分区（Partitioning）或分表技术，将数据分割成多个较小的表，减少查询的数据量



##### 8、小结：



![image-20250328171013072](MySQL 进阶.assets/image-20250328171013072.png)



### 7. Join 关联查询优化



#### 1、关联查询数据准备



>   创建表 t1 插入1万条数据 和 表 t2 插入 100 条数据、创建表的脚本见 <测试数据准备阶段>

mySql 的表关联常见有两种算法：

-   Nested-Loop Join 算法
-   Block Nested-Loop Join 算法



#### 2、嵌套循环连接 Nested - Loop Join（NLJ）算法



一次一行循环地从第一张表（称为驱动表）中读取行、在这行数据中取到关联字段，根据关联字段在另一张表（被驱动表）里取出满足条件的行、然后取出两张表的结果合集

```sql
EXPLAIN select * from t1 inner join t2 on t1.a= t2.a; # 走索引
```

![image-20210623164319877](MySQL 进阶.assets/image-20210623164319877.png)



**从执行计划中可以看到这些信息：**

驱动表 t2、被驱动表是 t1、先执行的就是驱动表（执行计划结果的 id 如果一样则按从上到下顺序执行SQL）、优化器一般会优先选择小表做驱动，所以使用 **inner join 时**、排在前面的表并不一定就是驱动表

当使用 **left join 时、左表是驱动表、右表是被驱动表**、当使用 **right join 时、右表时驱动表，左表是被驱动表**，当使用 join 时，MySql 会选择数据量比较小的表作为驱动表、大表作为被驱动表

使用 NLJ 算法、一般 join 语句中、如果执行计划 Extra 中出现 Using Join buffer 则表示使用的 join 算法是 NLJ



**上面SQL的大致流程如下：**

1.  从表 t2 中读取一行数据（如果 t2 表有查询过滤条件的、会从过滤结果里取出一行数据）
2.  从第 1 步的数据中、取出关联字段 a、到表 t1 中查找
3.  取出表 t1 中满足条件的行、跟 t2 中获取到的结果合并、作为结果返回客户端
4.  重复上面 3 步



整个过程会读取 t2 表的所有数据（扫描 100 行），然后遍历这每行数据中字段 a 的值、根据 t2 表中 a 的值索引扫描 t1 表中的对应行（扫描100次 t1 表的索引、1次扫描可以认为最终只扫描 t1 表一行完整数据、也就是总共 t1 表也扫描了 100 行）、因此整个过程扫描了 200 行

>   如果被驱动表的关联字没索引、使用 NLJ 算法性能会比较低（下面有详细解释），MySQL 会选择 Block Nested-Loop Join 算法



#### 3、基于块的嵌套循环连接 Block Nested - Loop Join(BNL)算法



>   把驱动表的数据读入到 join_buffer 中、然后扫描被驱动表、把被驱动表每一行取出来跟 join_buffer 中的数据做对比

```sql
EXPLAIN select * from t1 inner join t2 on t1.b= t2.b; # 无索引
```

![image-20210623164537480](MySQL 进阶.assets/image-20210623164537480.png)

Extra 中 的 Using join buffer (Block Nested Loop) 说明该关联查询使用的是 BNL 算法



**上面的大致流程如下：**

1.  把 t2 的所有数据放入到 join_buffer 中
2.  把表 t1 中每一行取出来，跟 join_buffer 中的数据做对比
3.  返回满足 join 条件的数据

整个过程对表 t1 和 t2 做了一次全表扫描、因此扫描的总行数为 10000 （表 t1 的数据总量） + 100（表 t2 的数据总量）= 10100、并且 join_buffer 里的数据是无序的、因此对表 t1 中的每一行都要做 100 次判断，所以内存中的判断次数是 100 * 10000 = 100 万次



>   **这个例子里表 t2 才 100 行，要是表 t2 是一个大表，join_buffer 放不下怎么办呢？**

join_buffer 的大小是由参数 join_buffer_size 设定的，默认值是 256k。如果放不下表 t2 的所有数据话，策略很简单，就是分段放、比如 t2 表有 1000 行录、join_buffer 一次只能放 800 行数据、那么执行过程就是先往 join_buffer 里放 800 行记录、然后从 t1 表里取数据跟 join_buffer 中数据对比得到部分结果，然后清空 join_buffer ，再放入 t2 表剩余200 行记录，再次从 t1 表里取数据跟 join_buffer 中数据对比。所以就多扫了一次 t1 表



>   **被驱动表的关联字段没索引为什么要选择使用 BNL 算法而不是用 Nested - Loop Join 呢？**

如果上面第二条 sql 使用 Nested-Loop Join，那么扫描行数为 100 * 10000 = 100万次，这个是磁盘扫描很显然，用 BNL 磁盘扫描次数少很多，相比于磁盘扫描，BNL 的内存计算会快得多因此 MySQL 对于被驱动表的关联字段没索引的关联查询，一般都会使用 BNL 算法。如果有索引一般选择 NLJ 算法，有索引的情况下 NLJ 算法比 BNL 算法性能更高



#### 4、对关联 SQL 的优化



##### 1、优化方案如下：



关联字段加索引：让 MYSQL 做 JOIN 操作时尽量选择 NLJ 算法

小表驱动大表：写多表连接 SQL 时 如果明确知道哪张表是小表可以用 **straight_join 写法固定连接驱动方式**、省去**MySql 优化器自己的时间**



##### 2、straight_join 解释：



straight_join 功能同 join 类似，但能让左边的表（驱动表）来驱动右边的表（被驱动表）、能改表优化器对于联表查询的执行顺序

```sql
# 代表指定 mysql 选着 t2 表作为驱动表
Select * from t2 straight_join t1 on t2.a = t1.a;
```



##### 3、优化注意事项



**straight_join 只适用于 Inner join**，并不适用于 Left Join，Right Join、因为 Left Join，Right Join 已经代表指定了表的执行顺序

尽可能让优化器去判断，因为大部分情况下 MySql 优化器是比人要聪明的. 使用 Straight_join 一定要慎重，因为部分情况下人为指定的执行顺序并不一定会比优化引擎要靠谱



##### 4、对于小表定义的明确



在决定哪个表做驱动表的时候、应该是两个表按照各自的条件过滤、过滤完成之后、计算参与 Join 的各个字段的总数居量、数据量小的那个表、就是“小表”应该作为驱动表



##### 5、in 和 exsits 优化：



>   原则：小表驱动大表、即小的数据集驱动大的数据集
>

in：当 B 表的数据集小于 A 表的数据集时、in 优于 exists   如右SQL语句：

```sql
select * from A where id in (select id from B)
```

等价于（伪代码）：

```sql
for(select id from B){
     select * from A where A.id = B.id
}
```



>   exists：当 A 表的数据集小于 B 表的数据集时、exists 优于 in

将主查询 A 的数据、放到子查询 B 中做条件验证、根据验证结果（true 或 false）来决定主查询的数据是否保留

```sql
select * from A where exists (select 1 from B where B.id = A.id)
# 等价于:
for(select * from A){
   select * from B where B.id = A.id
}
# A 表与 B 表的 ID 字段应建立索引
```

-   EXISTS (subquery) 只返回 true 或 false, 因此子查询中的 SELECT * 也可以用 SELECT 1 替换, 官方说法是实际执行时会忽略 SELECT 清单, 因此没有区别
-   EXISTS 子查询的实际执行过程可能经过了优化而不是我们理解上的逐条对比
-   EXISTS 子查询往往也可以用 JOIN 来代替，何种最优需要具体问题具体分析



### 8. 索引的设计、原则



#### 1、代码先行、索引后上



不知大家一般是怎么给数据表建立索引的，是建完表马上就建立索引吗？这其实是不对的，一般应该**等到主体业务功能开发完毕，把涉及到该表相关 Sql 都要拿出来分析之后、再建立索引**



#### 2、联合索引尽量覆盖条件



比如可以设计一个或者两三个联合索引 (尽量少建单值索引)，让每一个联合索引都尽量去包含 Sql 语句里的 where、order by、group by 的字段，还要确保这些联合索引的字段顺序尽量满足 Sql 查询的最左前缀原则



#### 3、不要在小基数字段上建立索引



索引基数是指这个字段在表里总共有多少个不同的值，比如一张表总共 100 万行记录，其中有个性别字段，其值不是男就是女，那么该字段的基数就是 2 如果对这种小基数字段建立索引的话，还不如全表扫描了，因为你的索引树里就包含男和女两种值，根本没法进行快速的二分查找，那用索引就没有太大的意义了一般建立索引，尽量使用那些基数比较大的字段，就是值比较多的字段，那么才能发挥出 B+ 树快速二分查找的优势来



#### 4、长字符串我们可以用前缀索引



尽量对字段类型较小的列设计索引，比如说什么 tinyint 之类的，因为字段类型较小的话，占用磁盘空间也会比较小，此时你在搜索的时候性能也会比较好一点当然，这个所谓的字段类型小一点的列，也不是绝对的。

很多时候你就是要针对 varchar(255) 这种字段建立索引，哪怕多占用一些磁盘空间也是有必要的对于这种varchar(255) 的大字段可能会比较占用磁盘空间，可以稍微优化下，比如针对这个字段的前 20 个字符建立索引，就是说，对这个字段里的每个值的前 20 个字符放在索引树里，类似于 KEY index(name(20),age,position)



>   此时你在 where 条件里搜索的时候，如果是根据 name字段来搜索，那么此时就会先到索引树里根据 name 字段的前 20 个字符去搜索，定位到之后前 20 个字符的前缀匹配的部分数据之后。
>
>   再回到聚簇索引提取出来完整的 name 字段值进行比对但是假如你要是 order by name，那么此时你的name因为在索引树里仅仅包含了前 20 个字符，所以这个排序是没法用上索引的， group by 也是同理。所以这里大家要对前缀索引有一个了解



#### 5、Where 和 Order by 冲突时优先 Where



>   在 where 和 order by 出现索引设计冲突时，到底是针对 where 去设计索引，还是针对 order by 设计索引？到底是让where 去用上索引，还是让 order by 用上索引?

一般这种时候往往都是让 where 条件去使用索引来快速筛选出来一部分指定的数据，接着再进行排序因为大多数情况基于索引进行 where 筛选往往可以最快速度筛选出你要的少部分数据，然后做排序的成本可能会小很多



#### 6、基于慢SQL查询做优化



可以根据监控后台的一些慢 Sql，针对这些慢 Sql 查询做特定的索引优化
关于慢 Sql 查询不清楚的可以参考这篇文章：https://blog.csdn.net/qq_40884473/article/details/89455740



### 9. 索引设计实战部分



#### 1. 索引设计实战案例一



##### 1. 项目背景简介



以社交场景 APP来举例，我们一般会去搜索一些好友，这里面就涉及到对用户信息的筛选，这里肯定就是对用户 User表搜索了、这个表一般来说数据量会比较大。我们先不考虑分库分表的情况，比如，我们一般会筛选地区(省市)，性别，年龄，身高，爱好之类的，有的 APP 可能用户还有评分，比如用户的受欢迎程度评分，我们可能还会根据评分来排序等等



##### 2. 需求分析和索引设计



对于后台程序来说除了过滤用户的各种条件，还需要分页之类的处理，可能会生成类似 Sql 语句执行：

```sql
Select xx from user where xx=xx and xx=xx order by xx limit xx,xx
```

对于这种情况如何合理设计索引了，比如用户可能经常会根据省市优先筛选同城的用户，还有根据性别去筛选，那我们是否应该设计一个联合索引  (province, city, sex) 了？这些字段好像基数都不大，其实是应该的，因为这些字段查询太频繁了



>   假设又有用户根据年龄范围去筛选了，比如 

```sql
where province = xx and city = xx and age >= xx and age <= xx
```

我们尝试着把 age 字段加入联合索引 (province, city, sex, age)，注意，一般这种范围查找的条件都要放在最后，之前讲过联合索引范围之后条件的是不能用索引的但是对于当前这种情况依然用不到 age 这个索引字段，因为用户没有筛选 sex 字段，那怎么优化了？



其实我们可以这么来优化下 Sql 的写法：

```sql
where province=xx and city=xx and sex in ('female','male') and age>=xx and age<=xx
```

对于爱好之类的字段也可以类似 sex 字段处理，所以可以把爱好字段也加入索引 (province, city, sex, hobby, age)



>   假设可能还有一个筛选条件



比如要筛选最近一周登录过的用户，一般大家肯定希望跟活跃用户交友了，这样能尽快收到反馈，对应后台 Sql 可能是这样：

```sql
where province = xx and city = xx and sex in ('female','male') 
   and age >= xx and age <= xx and latest_login_time >= xx
```

那我们是否能把 latest_login_time 字段也加入索引了？比如 (province, city, sex, hobby, age, latest_login_time) ，显然是不行的



**那怎么来优化这种情况了？**

其实我们可以试着再设计一个字段 is_login_in_latest_7_days，用户如果一周内有登录值就为 1，否则为 0，那么我们就可以把索引设计成以下来满足上面那种场景了 (province, city, sex, hobby, is_login_in_latest_7_days, age) 

一般来说，通过这么一个多字段的索引是能够过滤掉绝大部分数据的，就保留小部分数据下来基于磁盘文件进行 order by 语句的排序，最后基于limit进行分页那么一般性能还是比较高的



>   不过有时可能用户会这么来查询，就查下受欢迎度较高的女性，比如

```sql
where sex = 'female' order by score limit xx,xx
```

那么上面那个索引是很难用上的，不能把太多的字段以及太多的值都用 In 语句拼接到 Sql 里的，那怎么办了？其实我们可以再设计一个辅助的联合索引比如  (sex, score) 这样就能满足查询要求了



##### 3. 索引设计和优化总结



以上就是给大家讲的一些索引设计的思路了，核心思想就是，尽量利用3、4个复杂的多字段 联合索引，抗下你  80% 以上的查询，然后用一两个辅助索引尽量抗下剩余的一些非典型查询，保证这种大数据量表的查询尽可能多的都能充分利用索引。**剩余20%的非典型查询，如果查询模式差异很大，可能需要多个辅助索引**



索引设计的核心思路是：**根据实际查询模式，合理规划索引组合**。

1. **识别核心查询**：分析慢查询日志，找出**频率最高**、**业务最关键**的查询类型
2. **为每种核心查询设计最优索引**：针对每种查询模式，设计最合适的联合索引（注意字段顺序）
3. **合并相似索引**：如果一个索引能覆盖多个相似查询，就保留它；否则需要多个
4. **控制索引总数**：在查询性能和写入开销之间找到平衡点，通常单表索引数建议**控制在5-7个以内**
5. **持续观察优化**：随着业务变化，定期检查索引使用情况，清理无用索引



**某社交APP消息表（10亿+数据）**

| 方案       | 索引设计                      | 效果                                       |
| :--------- | :---------------------------- | :----------------------------------------- |
| **原方案** | 按"一两个联合索引抗下80%"设计 | 剩余20%查询频繁全表扫描，数据库CPU经常100% |
| **优化后** | 4个核心联合索引 + 2个辅助索引 | 99%查询走索引，CPU降到30%，响应时间<100ms  |

**关键洞察**：那20%的长尾查询，虽然频率低，但往往是大数据量查询，一旦不走索引，对系统的伤害可能比高频查询更大。



### 10. MySQL 规范手册（Alibaba）



#### 1. MySQL 数据类型选择



在MySQL中，选择正确的数据类型，对于性能至关重要。一般应该遵循下面两步：

-   确定合适的大类型：数字、字符串、时间、二进制
-   确定具体的类型：有无符号、取值范围、变长定长等

在 MySQL 数据类型设置方面，尽量用更小的数据类型，因为它们通常有更好的性能，花费更少的硬件资源。并且，尽量把字段定义为 NOT NULL，避免使用 NULL



#### 2. MySQL 数据类型表

| 类型         | 大小                                     | 范围（有符号）                                               | 范围（无符号）                                               | 用途           |
| ------------ | ---------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | -------------- |
| TINYINT      | 1 字节                                   | (-128, 127)                                                  | (0, 255)                                                     | 小整数值       |
| SMALLINT     | 2 字节                                   | (-32 768, 32 767)                                            | (0, 65 535)                                                  | 大整数值       |
| MEDIUMINT    | 3 字节                                   | (-8 388 608, 8 388 607)                                      | (0, 16 777 215)                                              | 大整数值       |
| INT或INTEGER | 4 字节                                   | (-2 147 483 648, 2 147 483 647)                              | (0, 4 294 967 295)                                           | 大整数值       |
| BIGINT       | 8 字节                                   | (-9 233 372 036 854 775 808, 9 223 372 036 854 775 807)      | (0, 18 446 744 073 709 551 615)                              | 极大整数值     |
| FLOAT        | 4 字节                                   | (-3.402 823 466 E+38, 1.175 494 351 E-38)，0，(1.175 494 351 E-38，3.402 823 466 351 E+38) | 0, (1.175 494 351 E-38, 3.402 823 466 E+38)                  | 单精度浮点数值 |
| DOUBLE       | 8 字节                                   | (1.797 693 134 862 315 7 E+308, 2.225 073 858 507 201 4 E-308), 0, (2.225 073 858 507 201 4 E-308, 1.797 693 134 862 315 7 E+308) | 0, (2.225 073 858 507 201 4 E-308, 1.797 693 134 862 315 7 E+308) | 双精度浮点数值 |
| DECIMAL      | 对DECIMAL(M,D) ，如果M>D，为M+2否则为D+2 | 依赖于M和D的值                                               | 依赖于M和D的值                                               | 小数值         |

**优化建议**

1.  如果整形数据没有负数，如ID号，建议指定为UNSIGNED无符号类型，容量可以扩大一倍。
2.  建议使用 TINYINT 代替 ENUM、BITENUM、SET。
3.  避免使用整数的显示宽度(参看文档最后)，也就是说，不要用 INT(10) 类似的方法指定字段显示宽度，直接用INT。
4.  DECIMAL 最适合保存准确度要求高，而且用于计算的数据，比如价格。但是在使用DECIMAL类型的时候，注意长度设置。
5.  建议使用整形类型来运算和存储实数，方法是，实数乘以相应的倍数后再操作。
6.  整数通常是最佳的数据类型，因为它速度快，并且能使用 AUTO_INCREMENT。



#### 3. 日期和时间类型



| 类型      | 大小(字节) | 范围                                       | 格式                | 用途                     |
| --------- | ---------- | ------------------------------------------ | ------------------- | ------------------------ |
| DATE      | 3          | 1000-01-01 到 9999-12-31                   | YYYY-MM-DD          | 日期值                   |
| TIME      | 3          | '-838:59:59' 到 '838:59:59'                | HH:MM:SS            | 时间值或持续时间         |
| YEAR      | 1          | 1901 到 2155                               | YYYY                | 年份值                   |
| DATETIME  | 8          | 1000-01-01 00:00:00 到 9999-12-31 23:59:59 | YYYY-MM-DD HH:MM:SS | 混合日期和时间值         |
| TIMESTAMP | 4          | 1970-01-01 00:00:00 到 2038-01-19 03:14:07 | YYYYMMDDhhmmss      | 混合日期和时间值，时间戳 |

**优化建议**

1.  MySQL 能存储的最小时间粒度为秒
2.  建议用 DATE 数据类型来保存日期。MySQL中默认的日期格式是 yyyy-mm-dd
3.  用 MySQL 的内建类型 DATE、TIME、DATETIME 来存储时间，而不是使用字符串
4.  当数据格式为 TIMESTAMP 和 DATETIME 时，可以用 CURRENT_TIMESTAMP 作为默认（ MySQL5.6 以后），MySQL 会自动返回记录插入的确切时间
5.  TIMESTAMP 是 UTC 时间戳，与时区相关
6.  DATETIME 的存储格式是一个 YYYYMMDD HH:MM:SS的整数，与时区无关，你存了什么，读出来就是什么
7.  除非有特殊需求，一般的公司建议使用 TIMESTAMP，它比 DATETIME 更节约空间，但是像阿里这样的公司一般会用 DATETIME，因为不用考虑 TIMESTAMP 将来的时间上限问题
8.  有时人们把 Unix 的时间戳保存为整数值，但是这通常没有任何好处，这种格式处理起来不太方便，我们并不推荐它



#### 4. 字符串类型



| 类型       | 大小                | 用途                                                         |
| ---------- | ------------------- | ------------------------------------------------------------ |
| CHAR       | 0-255字节           | 定长字符串，char(n)当插入的字符数不足n时(n代表字符数)，插入空格进行补充保存。在进行检索时，尾部的空格会被去掉。 |
| VARCHAR    | 0-65535 字节        | 变长字符串，varchar(n)中的n代表最大字符数，插入的字符数不足n时不会补充空格 |
| TINYBLOB   | 0-255字节           | 不超过 255 个字符的二进制字符串                              |
| TINYTEXT   | 0-255字节           | 短文本字符串                                                 |
| BLOB       | 0-65 535字节        | 二进制形式的长文本数据                                       |
| TEXT       | 0-65 535字节        | 长文本数据                                                   |
| MEDIUMBLOB | 0-16 777 215字节    | 二进制形式的中等长度文本数据                                 |
| MEDIUMTEXT | 0-16 777 215字节    | 中等长度文本数据                                             |
| LONGBLOB   | 0-4 294 967 295字节 | 二进制形式的极大文本数据                                     |
| LONGTEXT   | 0-4 294 967 295字节 | 极大文本数据                                                 |

**优化建议**

1.  字符串的长度相差较大用VARCHAR；字符串短，且所有值都接近一个长度用CHAR
2.  CHAR 和 VARCHAR 适用于包括人名、邮政编码、电话号码和不超过 255 个字符长度的任意字母数字组合。那些要用来计算的数字不要用 VARCHAR 类型保存，因为可能会导致一些与计算相关的问题。换句话说，可能影响到计算的准确性和完整性
3.  尽量少用 BLOB 和 TEXT，如果实在要用可以考虑将 BLOB 和 TEXT 字段单独存一张表，用 id 关联
4.  BLOB 系列存储二进制字符串，与字符集无关。TEXT 系列存储非二进制字符串，与字符集相关
5.  BLOB 和 TEXT 都不能有默认值。



**PS：INT 显示宽度**

我们经常会使用命令来创建数据表，而且同时会指定一个长度，如下。但是，这里的长度并非是 TINYINT 类型存储的最大长度，而是显示的最大长度

```sql
CREATE TABLE `user`(
    `id` TINYINT(2) UNSIGNED
);
```



这里表示 user 表的 id 字段的类型是 TINYINT，可以存储的最大数值是 255。所以，在存储数据时，如果存入值小于等于 255，如 200，虽然超过 2 位，但是没有超出 TINYINT 类型长度，所以可以正常保存；如果存入值大于 255，如500，那么 MySQL 会自动保存为 TINYINT 类型的最大值 255。

在查询数据时，不管查询结果为何值，都按实际输出。这里 TINYINT(2) 中 2 的作用就是，当需要在查询结果前填充 0时，命令中加上 ZEROFILL 就可以实现，如

```sql
`id` TINYINT(2) UNSIGNED ZEROFILL
```

这样，查询结果如果是 5，那输出就是 05。如果指定 TINYINT(5)，那输出就是 00005，其实实际存储的值还是 5，而且存储的数据不会超过 255，只是 MySQL 输出数据时在前面填充了 0

换句话说，在 MySQL 命令中，字段的类型长度 TINYINT(2)、INT(11) 不会影响数据的插入，只会在使用 ZEROFILL 时有用，让查询结果前填充 0



## 5. MySQL 底层原理探究



### 1. InnoDB 存储引擎概述



InnoDB 存储引擎最早由 Innobase Oy 公司开发（创始人HeikkiTuuri，芬兰赫尔辛基），于2016年被甲骨文并购，被包括在 MySQL 数据库所有的二进制发行版本中，从 MySQL5.5 版本开始是**默认存储引擎**

InnoDB存储引擎是第一个完整支持ACID事务的MySQL存储引擎（BDB是第一个支持事务的MySQL存储引擎，现在已经停止开发），其特点是行锁设计、支持MVCC、支持外键、提供一致性非锁定读，同时被设计用来最有效地利用以及使用内存和CPU

> [MySQL InnoDB存储引擎详细介绍之底层架构实现原理-CSDN博客](https://blog.csdn.net/shuiziliu518/article/details/151577168)



### 2. InnoDB 存储引擎架构



InnoDB 的架构分为 **后台线程**、**内存结构** 和 **磁盘结构** 三大部分，协同完成数据的高效存储、事务处理和并发控制

![image-20260311155257748](MySQL 进阶.assets/image-20260311155257748.png)

### 3. InnoDB 后台线程



后台线程的主要工作是负责刷新内存池中的数据，保证缓冲池中的缓存是最近的数据。此外将已修改的数据文件刷新到磁盘文件，同时保证在数据库发生异常的情况下InnoDB能恢复到正常运行状态。

> InnoDB存储引擎是多线程的模型，其后台有多个不同的后台线程，负责处理不同的任务

![image-20260311155524278](MySQL 进阶.assets/image-20260311155524278.png)

#### 3.1 Master Thread

`Master Thread` 是一个非常核心的后台线程，具有最高的线程优先级别。主要负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性，包括脏页的刷新、变更缓冲（Change Buffer）、undo页的回收等

MasterThread 内部由多个循环组成：主循环（loop ）、后台循环（backgroup loop）、刷新循环（flush loop）、暂停循环（suspend loop）



> **每1秒执行一次的操作：**
>

- 刷新 Redo Log Buffer (重做日志缓冲区) 中的内容写入磁盘，即使事务没有提交
- 将变更缓冲区中的数据合并到辅助索引页中（减少随机I/O）。（可能执行，在前1秒 IO 操作次数小于5次触发执行）
- 刷新 100个（默认）脏页数据到磁盘。（可能执行，在脏页数据数量大于参数 `innodb_max_dirty_pages_pct` 值触发执行）
- 切换为后台循环（Background Loop）。（可能执行，当检测到无用户活动或数据库关闭）

> **每10秒执行一次的操作：**

- 刷新100个（默认）脏页数据到磁盘。（可能执行，在前10秒 IO 操作次数小于 200 次触发）
- 合并变更缓冲区数据 （总会执行）
- 刷新日志缓冲区数据到磁盘 （总会执行）
- 删除无用的 undo 页 （总会执行）
- 刷新 100个 或 10个（默认）脏页数据到磁盘。（总会执行，根据缓冲池中脏页比例 `buf_get_modified_ratio_pct` 判断，大于该比例则刷新100个，小于则10个脏页数据）
- 检查并优化自适应哈希索引的性能
- 收集并更新表的统计信息（如行数、索引分布）



#### 3.2 IO Thread

> 使用 AIO（Async IO）来处理异步 I/O 操作，支持多线程并行操作，AIO 通过合并多个 IO 操作为单个操作，极大的提升吞吐量

主要有四种线程类型：

- `change buffer thread`：负责将变更缓冲内容刷新到磁盘
- `log thread`：负责将日志缓冲区内容刷新到磁盘
- `read thread`：负责读取操作，将数据从磁盘加载到缓存page页
- `write thread`：负责写操作，将缓存脏页刷新到磁盘。在 MySQL 5.6 版本后，脏页刷新操作由独立 Page Cleaner Thread 完成

InnoDB 1.0 版本之前固定 4 个 IO Thread，在 InnoDB1.0 版本开始可通过参数 `innodb_read_io_threads` 和`innodb_write_io_threads` 来设置读写线程数



#### 3.3 Purge Thread



- 事务提交后，回收无用的 Undo 日志（不再需要的版本）
- InnoDB 1.1 版本引入，在 InnoDB 1.2 版本可通过参数 `innodb_purge_threads` 来配置启动线程数。目的是为了减轻 Master Thread 的工作，从而提高 CPU 的使用率以及提升存储引擎的性能



#### 3.4 Page Cleaner Thread



- 将脏页数据放入到单独的线程中刷新到磁盘，脏页数据刷盘后相应的 redo log 也就可以覆盖，即可以同步数据，又能达到 redo log 循环使用的目的
- InnoDB 1.2 版本引入，可通过参数 innodb_page_cleaners 来配置启动线程数。目的是为了减轻 Master Thread 的工作及对于用户查询线程的阻塞，进一步提高 InnoDB 的性能



### 4. InnoDB 内存结构



> InnoDB 的内存架构以 **缓冲池（Buffer Pool）** 为核心
>
> 辅以日志缓冲区、数据字典缓存、锁系统、事务信息等模块，共同实现高性能的数据访问、事务处理及并发控制

![image-20260311164809580](MySQL 进阶.assets/image-20260311164809580.png)



#### 4.1 Buffer Pool 缓冲池



> 缓冲池（Buffer Pool）是 InnoDB 最核心的内存区域，主要用于缓存磁盘上的数据页（默认 16KB），包括数据页、索引页、undo 页、插入缓冲页等，以达到减少磁盘 I/O，提升查询和更新操作的性能



##### 4.1 存储结构



###### **4.1.1 (数据页 Data Page)**

存储表的实际数据行，每个数据页包含多行数据，默认页大小为16KB



###### **4.1.2 (索引页 Index Page)**

存储表的索引结构，包括主键索引（聚集索引）和辅助索引（二级索引）



###### **4.1.3 (回滚页 Undo Page)**

**很多资料会以为是先写入 Undo Log Buffer、其实没有这个概念，也不在 Log Buffer 结构中。而是在 Buffer Pool 缓冲区的数据页当中，也作为缓冲的一部分。所谓写入 Undo Log 其实就是先写入 Undo Page 缓冲区，再由 checkpoint 检查点触发刷脏机制落盘。**



###### **4.1.4 (变更缓冲区 Change Buffer)**

变更缓冲区（Change Buffer）是由插入缓冲区（Insert Buffer）演变而来，主要缓存了对 **二级索引页** 的修改操作（如插入、更新、删除），避免直接访问磁盘

适用场景：

- 适用于写密集型操作（如批量插入）
- 当目标页不在缓冲池中时，合并多次写入操作，减少随机 I/O

工作流程：

1. 修改操作发生时，若目标页未命中缓冲池，则将变更操作记录到 Change Buffer
2. 后台线程定期将 Change Buffer 中的变更合并到目标页（Merge 操作）
3. 目标页被访问时，自动触发 Merge 操作

```cmd
# 核心参数
innodb_change_buffer_max_size = 25%  # 默认最大占用缓冲池的 25%
```



###### **4.1.5  (自适应哈希索引 Adaptive Hash Index)**

自适应哈希索引（Adaptive Hash Index）就是当某个键值被频繁访问时，InnoDB 会自动为其创建生成哈希索引，加速对 **热点数据的等值查询**。**局限性**：仅支持等值查询（如 `WHERE id=1`），不支持范围查询

```cmd
# 禁用方式：通过配置
innodb_adaptive_hash_index = OFF #禁用自适应哈希索引
```



###### **4.1.6 (数据字典 Data Dictionary)**

- 存储表、列、索引、外键等元数据（MySQL 8.0后由独立数据字典管理，支持更灵活的元数据操作）
- 通过`information_schema`或`mysql.innodb_table_stats`等系统表访问



###### **4.1.7 (锁信息 Lock Info)**

- **行锁（Record Lock）**：锁定索引记录，支持乐观锁和悲观锁
- **间隙锁（Gap Lock）**：锁定索引间隙，防止幻读（RR隔离级别下生效）
- **表锁（Table Lock）**：用于DDL操作或全局锁（如`FLUSH TABLES WITH READ LOCK`）



###### **4.1.8 事务信息**

- **事务ID（TRX_ID）**: 唯一标识活跃事务，支持MVCC（多版本并发控制）
- **回滚段（Rollback Segment）**: 存储Undo日志，用于事务回滚和快照读（通过`innodb_rollback_segments`配置）
- **Undo 表空间**：存储历史版本数据，支持长事务和崩溃恢复



##### 4.2 管理策略

> 缓冲池通过采用改进的 **LRU 算法（`Least Recently Used`，最近最少使用）** 进行管理。即最频繁使用的页保存在LRU 列表的前端，而最少使用的页保存在 LRU 列表的尾端



###### **4.2.1 缓冲池三大链表**

- `LRU List 最常访问 `
  - 管理缓存的数据页，确保最常访问的数据在缓存中可用
  - 当一个数据页被访问时，该数据页会被添加到 LRU List
  - 当一个数据页被 LRU List 逐出时，该数据页会被添加到 Free List

- `Free List 空闲列表`
  - 管理内存中的空闲页面帧，确保有足够的空间加载新的数据页
  
  - 当系统启动时，所有的数据页都被添加到 Free List。当需要使用某个数据页时会优先从 Free List 中查找，如果存在则删除，添加到 LRU list
  
- `Flush List 刷盘列表`
- 管理需要刷新到磁盘的脏页 ( `Dirty Page`，即被修改但还未写入磁盘的数据页)，按修改时间排序，确保数据的一致性和持久性
  
- 当一个数据页被修改后，它会被标记为脏页，并被加入到 Flush List 中
  
- InnoDB 后台线程定期检查 Flush List，通过 Checkpoint 机制 将脏页写入磁盘，防止数据丢失



**4.2.2 LRU 列表两个分区**

- **new 列表区**：存放频繁访问的热点数据（占 63%，约5/8）
- **old 列表区**：存放使用较少的数据（占 37%，约3/8）



**4.2.3 LRU 列表如何运行**

- 新页插入 old 列表区

  - 将新页插入到 LRU 列表的 **midpoint（默认 5/8 位置）** old 列表区，而非头部，避免热点数据被频繁淘汰。

  - midpoint 位置可以由参数 `innodb_old_blocks_pct` 配置

- old 列表区数据淘汰

  - 当 LRU 列表空间不足，会先淘汰 LRU 列表尾部的数据，再将新页数据插入到 midpoint 位置

  - 当数据处于 old 列表区的时间未达到参数 `innodb_old_blocks_time` 配置的时间时，会将该数据直接淘汰`

- old 列表区转移至 new 列表区：当 old 列表区数据时长超过参数 `innodb_old_blocks_time` 配置的时间时，会将该数据转移到 new 列表区，即插入到 LRU 列表首位成为热点数据
- new 列表区转移至 old 列表区：由于 old 列表区的数据不断的刷新转移到 new 列表区，原来处于 new 列表区的数据会被持续的推后降级，直到降级到 old 列表区，最后移出 LRU 列表



##### 4.3 核心参数



| 参数                          | 描述                                           |
| ----------------------------- | :--------------------------------------------- |
| innodb_buffer_pool_size       | 指定缓冲池大小（建议设置为系统内存的50%~80%）  |
| innodb_buffer_pool_instances  | 指定缓冲池实例个数，多实例可减少锁竞争         |
| innodb_buffer_pool_chunk_size | 指定每个缓冲池实例的大小                       |
| innodb_old_blocks_pct         | 指定 LRU List 列表的 midpoint 位置             |
| innodb_old_blocks_time        | 指定 old 列表区的存活时间                      |
| innodb_buffer_pool_dump_pct   | 指定每个缓冲池最近使用的页面读取和转储的百分比 |



#### 4.2 Log Buffer 日志缓冲区

日志缓冲区（Log Buffer）仅仅是 Redo Log 内存日志的专属缓冲区，减少频繁的磁盘 I/O。注意：并不包括 Undo Log 和 Binlog，Undo Log 的缓冲区位于 Buffer Pool 当中的 Undo Page 数据页。**Binlog Cache 是 MySQL Server 层的结构**，不属于 InnoDB 层、更不属于 Log Buffer。



##### 4.2.1 存储结构

- **Redo Log**：记录物理修改操作（如页的更新），用于崩溃恢复



##### 4.2.2 刷新策略

- 事务提交时：根据 `innodb_flush_log_at_trx_commit` 参数决定是否强制刷盘

  - **1（默认）：每次提交都刷盘（保证 ACID、最安全）**

  - 2：每秒刷盘一次（性能优先，可能丢失 1 秒数据）

  - 0：依赖操作系统定时刷盘（风险较高）

- 后台线程：定期将日志缓冲区内容刷新到 Redo Log 文件（如每秒一次）



##### 4.2.3 核心参数

| 参数                           | 描述                         |
| ------------------------------ | :--------------------------- |
| innodb_log_buffer_size         | 指定日志缓冲区大小           |
| innodb_log_file_size           | 指定 redo log 日志文件的大小 |
| innodb_flush_log_at_trx_commit | 指定事务提交时是否刷盘       |



#### 4.3 Additional Memory Pool 额外内存池



> 额外内存池（Additional Memory Pool）主要是为 InnoDB 的元数据（metadata）和内部管理结构（如表结构信息、索引元数据、缓冲池（Buffer Pool）的控制块、锁信息等）提供内存空间，避免这些结构直接从操作系统动态分配内存，从而减少内存分配的开销（如系统调用、内存碎片等）



##### 4.3.1 存储结构

- 主要存储 InnoDB 字典缓存（dictionary cache）中的数据
- 表和列的定义信息（如 CREATE TABLE 语句定义的结构）
- 索引的元数据（如索引类型、字段顺序、页分布等）
- 缓冲池的页描述符（每个缓冲池页对应的控制信息）
- 锁管理器中的锁结构、事务相关的元数据等



##### 4.3.2 性能问题

当缓冲池非常大时（如几十GB甚至更大），这些内部数据结构所需的内存也会相应增加。如果这块内存不足，InnoDB可能会尝试从操作系统的内存分配器申请更多内存，可能导致性能下降



##### 4.3.3 核心参数

额外内存池的大小可以由参数 `innodb_additional_mem_pool_size` 配置。在 5.6+ 版本中，InnoDB 通常会自行在Buffer Pool 之外分配需要的内存，这个参数已不再重要



### 5. MySQL 磁盘结构



> MySQL 的磁盘文件系统是 MySQL 数据库中用于存储数据、日志、配置信息和运行时状态的核心组件。主要包括 **参数文件**、**日志文件**、**Socket 文件**、**PID 文件**、**MySQL 表结构文件** 和 **InnoDB 存储引擎文件** 等

![image-20260311190911424](MySQL 进阶.assets/image-20260311190911424.png)



#### 5.1 参数文件



参数文件是 MySQL 的配置文件，用于定义数据库的运行时行为，包括内存分配、日志路径、存储引擎配置等

- 控制 InnoDB 的内存分配（如 `innodb_buffer_pool_size`）
- 定义表空间文件和日志文件的存储路径及大小
- 调整事务日志刷新策略以平衡性能与数据安全



**常见配置文件**：

- **Linux 系统**：`/etc/my.cnf` 或 `/etc/mysql/my.cnf`
- **Windows 系统**：`my.ini`

**配置文件简单示例**

```sql
[mysqld]
# InnoDB 缓冲池大小（核心参数）
innodb_buffer_pool_size = 20G

# 共享表空间文件路径
innodb_data_home_dir = /var/lib/mysql
innodb_data_file_path = ibdata1:2000M:autoextend

# 独立表空间配置
innodb_file_per_table = 1

# 重做日志文件配置
innodb_log_group_home_dir = /var/lib/mysql
innodb_log_file_size = 1G
innodb_log_files_in_group = 2

# 日志刷新策略
innodb_flush_log_at_trx_commit = 1

# 线程并发度
innodb_thread_concurrency = 16
————————————————
版权声明：本文为CSDN博主「五老新」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/shuiziliu518/article/details/151577168
```



#### 5.2 日志文件



> InnoDB 的日志文件主要用来记录MySQL实例对某种条件做出响应时写入的文件，如错误日志文件、二进制日志文件、慢查询日志文件、查询日志文件等



##### 5.2.1 错误日志 Error Log



- **所属层级：Server 层**
- **核心作用**：记录 MySQL 启动、运行、关闭过程中的错误信息、警告及关键状态信息，是故障排查的首要依据
- **主要记录**：
  - MySQL 启动 / 关闭
  - 服务器运行错误
  - 崩溃信息
  - InnoDB错误
  - 主从复制错误
- **配置参数**：
  - log_error：指定日志路径（如 `/var/log/mysql/error.log`）
  - log_timestamps：控制时间戳格式（如 SYSTEM 或 UTC）
- **查看方法**：
  - 使用 `SHOW VARIABLES LIKE 'log_error'` 定位路径
  - 通过 `grep "error" /var/log/mysql/error.log` 快速筛选错误信息
- **管理建议**：
  - 定期检查，尤其是启动失败时优先分析此日志
  - 结合 logrotate 实现日志轮转，避免占用过多磁盘空间



##### 5.2.2 慢查询日志 Slow Query Log



- **所属层级：Server 层**
- **核心作用**：记录执行时间超过阈值的SQL语句，用于性能优化和SQL调优
- **配置参数**：
  - `slow_query_log`：开启/关闭慢查询日志，默认关闭
  - `long_query_time`：设置慢查询阈值（默认10秒，高并发场景建议设为0.5秒）
  - `log_queries_not_using_indexes`：记录未使用索引的查询
  - `log_throttle_queries_not_using_indexes`：限制每分钟记录的未使用索引的SQL数量（如100条/分钟）
  - `long_query_io`：记录逻辑读取次数超过阈值的查询
- **查看与分析**：
  - 使用 `SHOW VARIABLES LIKE '%slow%';` 定位路径
  - 使用 `SHOW STATUS LIKE 'Slow_queries';` 查看慢查询 sql 数量
  - 使用 `mysqldumpslow` 工具分析日志 `mysqldumpslow -s al -n 10 /var/log/mysql/slow.log` 获取执行时间最长的10条SQL
  - 可将日志输出到表（mysql.slow_log），便于SQL查询分析
- **生产环境建议**：
  - 结合业务场景调整阈值，避免记录过多无关查询
  - 定期分析，识别频繁执行或资源消耗大的SQL



##### 5.2.3 查询日志 General Query Log



- **所属层级：Server 层**
- **核心作用**：记录所有客户端连接和执行的SQL语句（包括 SELECT ），用于审计和调试
- **配置参数**：
  - `general_log`：开启/关闭查询日志
  - `general_log_file`：指定日志路径（如 /var/log/mysql/query.log）
  - `log_output`：设置输出格式（FILE 或 TABLE）
- **注意事项**：
  - 生产环境通常关闭，因日志量极大，可能影响性能
  - 开启时需谨慎，避免泄露敏感信息
- **查看方法**：
  - 通过 `SHOW VARIABLES LIKE 'general_log%';` 查看配置
  - 使用 `SELECT * FROM mysql.general_log;` 查询表格式日志



##### 5.2.4 **中继日志 (Relay Log)**



在主从复制架构中，从服务器的 I/O 线程从主服务器拉取 binlog 后，会先写入本地的 Relay Log，然后由 SQL 线程读取并执行。其格式与 binlog 相同



#### 5.3 Socket 文件



- **所属层级：Server 层**
- **作用**：本地客户端（如 mysql 命令行）与 MySQL 服务进程通信的 Unix 域套接字文件
- **默认路径**：Linux 通常为 `/var/run/mysqld/mysqld.sock`，Windows 为 C:\Program Files\MySQL\Data\mysql.sock
- **配置方法**：在 `my.cnf` 中通过 `socket=/path/to/socket` 自定义路径
- **常见问题**：路径不匹配或权限不足会导致“无法连接”错误，需检查文件存在性及权限（如chmod 660）



#### 5.4 Pid 文件



- **所属层级：Server 层**
- **作用**：存储 MySQL 进程 ID（PID），用于进程管理、防止多重启动及监控
- **默认路径**：通常位于数据目录（如 `/var/lib/mysql/mysqld.pid`）
- **配置方法**：通过 `pid-file=/path/to/pid` 指定路径
- **管理场景**：
  - 启动失败时检查 PID 文件是否存在，避免重复启动
  - 自动化脚本通过读取 PID 监控服务状态



#### 5.5 MySQL 表结构文件



- **作用**：
  - 存储表的 元数据（如列定义、索引信息、视图定义等）
  - 文件格式为 .frm，每个表对应一个文件（如 `table_name.frm`）
- **文件路径**：
  - 默认存储在数据库目录下（如 /var/lib/mysql/database_name/）
- **文件类型**：
  - .frm 文件：早期版本存储表结构（列、索引、约束等），MySQL 8.0后逐渐被系统表空间替代
  - 系统表空间（ibdata1）：存储数据字典、撤销日志、变更缓冲等元数据，默认包含所有表结构
  - 独立表空间（.ibd）：启用 `innodb_file_per_table=ON` 后，每张表对应一个.ibd 文件，存储数据、索引及表结构元数据



#### 5.6 InnoDB 表存储引擎文件



> InnoDB 的存储引擎文件包括 **表空间文件** 和 **重做日志文件**，是 InnoDB 数据存储的核心



##### 5.6 表空间文件 Tablespace



> InnoDB 的所有数据都被逻辑地存放在一个空间中，称之为表空间（tablespace）。表空间又由段（segment）、区（extent）、页（page）结构组成

InnoDB 表空间分类是 MySQL 数据库存储管理的核心架构，包含**系统表空间、独立表空间、通用表空间、临时表空间、回滚表空间**五大核心类型，主要存储表数据、索引和事务信息等

![image-20260311192545835](MySQL 进阶.assets/image-20260311192545835.png)



###### ***5.6.1 系统表空间（System Tablespace）***

- **核心作用**：存储元数据（表/索引/列定义）、双写缓冲区、回滚日志、变更缓冲、系统事务数据等全局数据，是InnoDB的默认共享存储区
- **文件结构**：默认文件为 ibdata1，可扩展为多个文件（如 ibdata1:100M;ibdata2:200M:autoextend ），支持裸设备分区提升 I/O 效率。MySQL 8.0 后，数据字典不再保存在系统表空间，但双写缓冲、插入缓冲仍存储于此
- **配置参数**：
  - `innodb_data_file_path`：定义文件路径、大小及自动扩展策略（如末文件可扩展至最大 1 GB）
  - `innodb_data_home_dir`：指定表空间目录（默认 datadir）
- **版本差异**：MySQL 8.0 前用户表数据可能存储于此，之后默认启用独立表空间；双写缓冲区（16KB 页）防止部分写入失效，确保数据页可靠性
- **管理要点**：碎片化空间无法自动收缩，需通过重建表空间回收；高并发下可能成 I/O 瓶颈



###### ***5.6.2 独立表空间（File-Per-Table Tablespace）***

- **核心作用**：每张表独占一个.`ibd` 文件，存储表数据、索引及元数据，实现表级隔离
- **配置逻辑**：
  - 默认启用（`innodb_file_per_table=ON`），新表自动创建独立文件；支持 `DATA DIRECTORY` 子句指定存储路径（如外部磁盘）
  - 支持数据在线迁移（表空间传输功能），需满足版本一致、表结构相同等条件
- **优势**：
- **空间回收**：`TRUNCATE TABLE` 或 `ALTER TABLE ... ENGINE=InnoDB` 释放空间至文件系统
- **迁移灵活**：可复制 .ibd 文件至其他实例，通过 `ALTER TABLE ... IMPORT TABLESPACE` 导入
- **并发优化**：多表写入分散至不同文件，减少共享表空间 I/O 竞争
- **限制**：只存储了表的索引和数据，但是其他类的数据，如回滚（undo）信息，变更缓冲、索引页、事务信息，二次写缓冲（Double write buffer）等全局数据还是存放在系统表空间内



###### ***5.6.3 通用表空间（General Tablespace）***

- **核心作用**：跨数据库共享存储，允许多个表存储于同一 `.ibd` 文件，减少元数据内存开销
- **创建方式**：
  - 语法：`CREATE TABLESPACE ts ADD DATAFILE '/path/ts.ibd' FILE_BLOCK_SIZE=16K ENGINE=InnoDB`
  - 支持自定义路径（需通过 `innodb_directories` 配置目录），避免与独立表空间冲突
- **优势**：
  - 性能优化：减少 `DROP/TRUNCATE` 的文件系统开销，提升大批量操作效率
  - 存储管理：支持表压缩（需统一压缩格式），减少存储占用
- **管理要点**：删除表空间前需先删除所有表；通过 `ALTER TABLE ... TABLESPACE=` 转移表



###### ***5.6.4 临时表空间（Temporary Tablespace）***

- **核心作用**：存储临时表、排序数据、哈希连接中间结果等临时数据，避免占用永久存储
- **文件结构**：默认文件为 `ibtmp1`（12MB自动扩展），位于数据目录，服务器关闭时自动删除并重建
- **配置参数**：`innodb_temp_data_file_path` 控制路径、大小及扩展策略（如 `ibtmp1:12M:autoextend:max:1G`）
- **应用场景**：支持 `ORDER BY`、`GROUP BY`、索引创建等操作的临时存储，提升查询效率
- **监控**：通过 `INFORMATION_SCHEMA.INNODB_TEMP_TABLESPACES` 查看使用情况



###### ***5.6.5 回滚表空间（Undo Log Tablespace）***

- **核心作用**：存储撤销日志，支撑事务回滚、MVCC（多版本并发控制）及崩溃恢复
- **存储逻辑**：
  - MySQL 8.0 后默认独立于系统表空间，文件名为 `undo_001`、`undo_002` 等，支持动态添加（`CREATE UNDO TABLESPACE ... ADD DATAFILE`）
  - 每个表空间支持 128 个回滚段（`innodb_rollback_segments`），通过 `innodb_undo_tablespaces` 控制数量
- **配置参数**：
  - `innodb_undo_directory`：指定存储目录（默认 `datadir`）
  - `innodb_max_undo_log_size`：设置截断阈值（默认 1GB），超量时自动回收空间
- **管理要点**：监控工具 `INNODB_METRICS` 查看截断状态，`SET GLOBAL innodb_monitor_enable=module_undo` 启用日志



##### 5.7 重做日志文件 Redo Log



- 记录物理页的修改操作，用于崩溃恢复（确保事务的持久性）
- 采用 **循环写入** 和 **检查点（Checkpoint）** 机制，避免日志文件无限增长

默认存储在 `innodb_log_group_home_dir` 指定的目录下（如 `/var/lib/mysql`），文件名为 `ib_logfile0`、`ib_logfile1` 等

```sql
# 关键参数
innodb_log_file_size = 1G     # 每个日志文件大小
innodb_log_files_in_group = 2 # 日志文件组数量
innodb_log_buffer_size = 64M  # 日志缓冲区大小
innodb_loggroup_home_dir = ./  #指定了日志文件组所在路径，默认为./
```

**特点**：

- **物理日志**：记录数据页的物理修改（如页的更新）
- **崩溃恢复**：在系统重启时，通过 Redo Log 恢复未写入磁盘的已提交事务



### 6. MySQL 重要二进制日志小结



> MySQL中的日志种类不少，但从架构上可以清晰地分为两大阵营：**Server层日志**（**所有存储引擎共享**）和**引擎层日志**（以InnoDB为例）
>
> 其中，**二进制日志(binlog)、重做日志(redo log)和回滚日志(undo log)** 是实现事务、崩溃恢复和主从复制的基石，最为关键



![image-20260312145214801](MySQL 进阶.assets/image-20260312145214801.png)



#### 6.1 Binary Log (二进制日志 )



##### 5.2.4 二进制日志 Binary Log

- **所属层级**: **MySQL Server 层**、并不和 Redo Log 在同一层级
- **核心功能**: 它记录了所有对数据库执行更改的操作（如`UPDATE`、`INSERT`、`DELETE`、`CREATE TABLE`等），但不包括`SELECT`和`SHOW`这样的查询语句。它的主要作用是**数据恢复**和**主从复制**。
- **日志格式**: 这是binlog最核心的特性，有三种记录格式：
  - **STATEMENT 格式 (基于语句的复制, SBR)**：记录的是原始的SQL语句。优点是日志量小。缺点是某些非确定性函数（如`NOW()`）在主从复制时可能导致数据不一致
  - **ROW 格式 (基于行的复制, RBR)**：不记录SQL语句，而是记录每一行数据被修改成什么样了。优点是复制最安全，不会出现数据不一致问题。缺点是日志量可能非常大，尤其是批量操作时
  - **MIXED 格式 (混合格式)**：MySQL自动判断，默认使用STATEMENT格式，当遇到可能引起不一致的情况时，自动切换为ROW格式。从MySQL 8.0开始，**默认格式为ROW**
- **配置参数**：

- `log_bin`：指定二进制日志路径（如 `/var/lib/mysql/binlog`）
- `binlog_format`：设置日志格式（ ROW 为默认推荐）
- `expire_logs_days`：自动清理日志天数（如 7 天）
- `max_binlog_size`：控制单个日志文件大小（默认 1 GB）
- `sync_binlog`：每次事务提交时同步写入磁盘（安全性高）

- **查看与分析**：
  - 通过 `SHOW VARIABLES LIKE 'log_bin%';` 查看配置
  - 使用 `mysqlbinlog --start-datetime="2025-09-01 00:00:00" binlog.000001` 解析日志
  - 结合 `pt-query-digest` 进行性能分析
- **生产环境建议**：
  - **开启 `sync_binlog=1` 、一般和 Redo Log 的配置一样。都在事务提交时刷盘，提升数据安全性**
  - 定期备份和清理旧日志，避免磁盘空间不足



#### 6.2 Redo Log (重做日志)



- **所属层级**: **InnoDB 引擎层 (Log Buffer)**
- **核心功能**: 这是实现 WAL 机制的核心，主要用于**保证事务的持久性**。当数据库发生崩溃时，用它来恢复那些已经提交但还没来得及写入磁盘的数据页的修改
- **物理格式**: Redo Log是**物理日志**，记录的是“在某个数据页的某个偏移量上做了什么修改”。例如，它记录的内容类似于“将表空间X的第100号页面的第1024字节开始的数据，更新为Y”。
- **文件格式**:
  - **MySQL 8.0.30之前**: 默认在数据目录下生成两个文件 `ib_logfile0` 和 `ib_logfile1`，循环写入。
  - **MySQL 8.0.30及之后**: 日志文件位于 `#innodb_redo` 目录下，有多个`#ib_redoNNN`文件，总容量由 `innodb_redo_log_capacity` 控制。
- **内部结构**: Redo Log文件由**日志文件头**和多个**日志块**组成，每个块通常为 512 byte，并包含头部信息（如块号、数据长度）和尾部校验和。日志通过不断增长的 **LSN（日志序列号）**来标记写入位置



#### 6.3 Undo Log (回滚日志)



- **所属层级**: **InnoDB 引擎层 (Buffer Pool 的 Undo Page)**
- **核心功能**: 主要用于**保证事务的原子性**（事务回滚）和 **实现 MVCC（多版本并发控制）**。当执行修改时，Undo Log会记录修改前的数据，以便在需要回滚时恢复
- **逻辑格式**: 与 Redo Log不同，Undo Log 是**逻辑日志**。可以把它理解为记录的是反向操作的SQL语句，或者说是数据的历史版本。例如，执行`UPDATE ... SET name='new' WHERE id=1`，Undo Log会记录一条信息，告诉你这条记录原来的名字是'old'。这些旧版本通过版本链组织起来，供不同的事务读取，从而实现高并发下的数据一致性



#### 6.4 日志分类小结



| 日志类型            | 层级   | 作用                     | 是否重要 |
| ------------------- | ------ | ------------------------ | -------- |
| Error Log           | Server | 记录错误、启动、崩溃信息 | ⭐⭐⭐      |
| General Query Log   | Server | 记录所有SQL              | ⭐        |
| Slow Query Log      | Server | 记录慢SQL                | ⭐⭐       |
| Binary Log (binlog) | Server | 数据复制、恢复           | ⭐⭐⭐⭐     |
| Relay Log           | Server | 主从复制使用             | ⭐⭐⭐      |
| Redo Log            | InnoDB | 崩溃恢复                 | ⭐⭐⭐⭐     |
| Undo Log            | InnoDB | 事务回滚、MVCC           | ⭐⭐⭐      |



#### 6.5 什么是 WAL



> MYSQL InnoDB 的事务遵循 **WAL（Write-Ahead Logging）** ，也是数据库中最核心的机制之一。WAL 的原则是：**在修改数据页之前，必须先把对应的日志写入持久化存储**，在 MySQL 中就是，数据页刷脏到磁盘前，Redo Log Buffer 中的缓冲数据必须先写入到磁盘。而对 Undo 页没有要求

WAL机制带来的好处：

1. **写入性能提升**：写 Redo Log 比直接在磁盘上随机写快 100 倍以上
2. **数据安全**：数据库崩溃时的恢复能力
3. **一致性保证**：ACID 特性的基础

> **为什么写 Redo Log 是顺序 I/O ？而直接修改磁盘是随机 I/O？**

```ceylon
Redo Log文件（ib_logfile0）是预分配的文件：

┌──────────────────────────────────────────────────────┐
│ 写入位置 →  记录1 | 记录2 | 记录3 | 记录4 | 记录5...  │
└──────────────────────────────────────────────────────┘
           ↑                                        ↑
       当前写入点                             文件末尾
    
Redo Log顺序写入（无需移动磁头）：

[写入前]
ib_logfile0: [位置0] 空闲
             [位置1] 空闲
             [位置2] 空闲
             ...

[提交时，顺序写入三条记录]
ib_logfile0: [位置0] 记录1: "页1的偏移量1024改成100"  ← 写入
             [位置1] 记录2: "页3的偏移量2048改成200"  ← 继续写入
             [位置2] 记录3: "页2的偏移量1024改成300"  ← 继续写入
             [位置3] 空闲

写入指针从0 → 1 → 2 顺序移动
```

虽然 Redo Log 预分配了文件，将同步写改为了异步写，为高并发写入预留了缓冲空间，大大提高了并发量，像是数据库层面的中间件。**但并不代表数据最终的落盘就不是随机 I/O了**，最后还是由异步线程 `Page Cleaner Thread` 刷脏页落盘、随机写并未消失。但是 InnoDB 做了优化

- **延迟写**：把必须同步的随机写，变成了可以异步执行的随机写
- **合并写入 (Group Commit)**：多次修改同一个页，只需要写一次磁盘
- **削峰填谷**：把突发的写入压力，平摊到后台慢慢处理

```less
有WAL：
┌─────────────────────────────────────────┐
│ 事务1: 顺序写日志（记录修改页1）           │
│ 事务2: 顺序写日志（记录修改页2）           │
│ 事务3: 顺序写日志（记录修改页1）           │
│ ...（所有事务快速返回）                    │
│                                         │
│ 后台刷脏（每秒一次）：                     │
│ 发现页1被改了2次，只需要刷1次              │
│ 发现页2被改了1次，刷1次                    │
│ ...                                      │
│ 总IO次数：500次随机写（合并了重复页）       │
│ 用户等待时间：只等日志写入，不用等刷盘       │
└─────────────────────────────────────────┘
```



#### 6.6 Buffer Pool 结构图



> 下图清晰地展示了一个事务从开始到提交，Redo Log、Undo Log 和 Binlog 的完整协作流程，这也是 InnoDB 存储引擎能够保证数据一致性的核心所在
>

![e2978a21c6bec28f7e3c37f1fe6fdb7d](MySQL 进阶.assets/e2978a21c6bec28f7e3c37f1fe6fdb7d.png)

以下为 MySQL官方的 **Innodb Buffer Pool结构 与磁盘交互图** MySQL 8.4 版本(未来主流)

[MySQL ：： MySQL 8.4 参考手册 ：： 17.4 InnoDB 架构](https://dev.mysql.com/doc/refman/8.4/en/innodb-architecture.html)

![innodb-architecture-8-0](MySQL 进阶.assets/innodb-architecture-8-0.png)



#### 6.7 InnoDB 一次事务的完整流程



> 要理解 Redo Log、Undo Log 和 Binlog 是如何协同工作的，最好的方式就是追踪一个完整的事务生命周期。这三者通过一个被称为**“两阶段提交”**的精巧机制紧密配合，共同保证了 MySQL 的数据一致性、原子性和持久性

```sql
-- 示例事务
BEGIN;
UPDATE user SET age = 26 WHERE id = 1;
COMMIT;
```

> 一次事务、Innodb 内部的执行流程如下



**1、客户端发送 UPDATE SQL 语句**

```sql
# InnoDB 检查修改数据是否在 Buffer Pool 数据页中、若没有从磁盘加载数据页到内存
```

**2、写 Undo Log 到 Undo Page 中**

- 在修改数据前，InnoDB 会将修改数据的 **旧版本**（例如，age = 25 (old)）写入 Undo Log。它就像是数据修改前的“快照”，支撑着事务的 MVCC 功能为其他并发事务提供一致性读视图
- `Uodo Page` 其实是在 Buffer Pool 内存中的数据页，跟 Buffer 类似。有着真正的刷盘时机，在没刷盘前崩溃，**也有丢失 Uodo Log 的风险，所以写 Uodo Log 的时候会额外在写一份 Redo Log 来重做 Undo Log (用于MVCC机制和一致性视图)**
  - 刷盘时机由 `Checkpoint` 检查点机制触发

**3、修改 Buffer Pool 中的数据页变为脏页 (Dirty Page)**

- 一次 Update 可能修改、Data Index、Data Page、以及 Change Buffer 等等

**4、同时写 Redo Log 到 Log Buffer**

- 并非一个 Update 产生一条 Redo Log，而是每一处值的变化都是一条 Redo Log。可能产生十几条
  - uodo 页的更改 也会记录到 Redo Log.
  - Data Index 页的更改
  - Data Page 页的更改
  - Change Buffer 页的更改
  - System Page 系统页

**5、发送 Commit 指令**

- 客户端开始发送 Commit 指令

**6、执行`(二阶段提交)` Redo Log `prepare`**

- 将 Log Buffer 的 Redo Log 刷盘拆分为 **Prepare** 和 **Commit** 两个阶段，目的是让 Redo Log（物理日志）和 Server层的 Binlog（逻辑日志）保持状态一致

- **Prepare 阶段**：事务执行 COMMIT 后，InnoDB 首先将处于 Prepare 状态的 Redo Log 持久化到磁盘。此时的日志已经包含了所有的修改信息，但还处于一种 `Prepare` 状态。**XID（事务ID）** 会被同时记在 Redo Log 和即将写入的 Binlog 中，作为两者配对的 “信物”

- **Log Buffer 的刷盘策略配置如下(重要)**：

  ```sql
  innodb_flush_log_at_trx_commit = 1（默认，最安全、金融级）
  -- 每次事务提交时，都将 redo log buffer fsync() 到磁盘 [性能最差，但保证不丢已提交事务]
  -- 事务提交指 写入 Prepare 阶段开始刷盘了
  
  # OS 代表操作系统缓存（OS Page Cache）
  innodb_flush_log_at_trx_commit = 2（大多数互联网项目）
  -- 每次事务提交时，只写入 OS Cache，每秒 fsync() 到磁盘 [性能好，MySQL崩溃不丢。但OS崩溃丢1秒数据]
  
  innodb_flush_log_at_trx_commit = 0 
  -- 每秒才写入 OS Cache 并 fsync 性能最好，MySQL崩溃也丢、OS崩也丢1秒数据
  ```

**7、Server 层写 Binlog 到 Binlog Cache**

- InnoDB 通知 Server 层将事务写入 `Binlog Cache`。但是也在缓存中，Binlog 刷盘时机策略如下

  ```sql
  sync_binlog = 1 (最安全)
  -- 每次事务提交时，都强制将 binlog cache fsync 到磁盘
  
  sync_binlog = 0 (默认)
  -- 由操作系统决定何时刷盘，性能最好，但风险最大
  
  sync_binlog = N (如 sync_binlog=100)
  -- 每N次事务提交才刷盘一次、性能和数据安全之间的折中
  ```

**8、将 Redo Log 状态改为 `commit`**

- 只有在 Binlog 成功写入磁盘后，InnoDB 才会在 Redo Log 中写入一个最终的 **COMMIT 标记**，并认为事务真正完成
- 通知客户端事务提交成功（**注意：如果 Redo log 的刷盘策略不是1、也会返回成功**）

**9、后台 `Page Cleaner Thread` 线程异步刷脏 (`checkpoint`)** 

- 只有当 Redo Log 已经刷回磁盘的情况下 (通过校验 LSN)，Page Cleaner Thread 才会对 Buffer Pool 中的数据页的更改刷盘

  ```sql
  Redo Log 全局刷盘位置 LSN = 1800
  ------------------------------------------------------
  Page Cleaner 扫描 Buffer Pool 中的 Flush List：
  ┌─────────────────────────────────────┐
  │ 脏页链表 (按LSN从小到大)               │
  ├─────────────────────────────────────┤
  │ 页A LSN=1000 → 页B LSN=1500 → 页C LSN=2000 → 页D LSN=2500
  └─────────────────────────────────────┘
  ✓ 页A (1000 ≤ 1800) → 可以刷
  ✓ 页B (1500 ≤ 1800) → 可以刷
  ✗ 页C (2000 > 1800) → 不能刷，停止扫描
    后面的页 D 更大，都不用看 这样只需要检查到第一个不满足条件的页
  ```

- Checkpoint 由多种情况触发、触发后将促进 Page Cleaner Thread 线程管理和分类 Buffer Pool 中的不同脏页，并分配到正确的三个链表当中

| 触发条件             | 说明                         | 相关参数                                |
| :------------------- | :--------------------------- | :-------------------------------------- |
| **定时触发**         | Master Thread每秒/每10秒执行 | 默认机制                                |
| **脏页比例过高**     | 脏页占比超过阈值             | `innodb_max_dirty_pages_pct`（默认75%） |
| **Redo Log空间压力** | 未刷盘的Redo Log占比过高     | 异步/同步阈值                           |
| **LRU列表空间不足**  | 需要淘汰页时                 | 默认机制                                |
| **正常关闭**         | 数据库关闭时                 | `innodb_fast_shutdown`                  |



> **事务崩溃恢复的保证范围**

| 事务状态   | 崩溃时机            | Redo Log状态         | 能否恢复   | 原因           |
| :--------- | :------------------ | :------------------- | :--------- | :------------- |
| **已提交** | 返回成功给客户端后  | 已持久化到磁盘       | ✅ 能恢复   | Redo Log完整   |
| **已提交** | fsync完成，但返回前 | 已持久化到磁盘       | ✅ 能恢复   | Redo Log完整   |
| **已提交** | fsync进行中         | 可能部分写入         | ⚠️ 可能损坏 | 需checksum检测 |
| **已提交** | 刚进OS Cache        | 仅在OS Cache         | ⚠️ 取决于OS | OS可能丢数据   |
| **未提交** | 任何时刻            | 未持久化或未标记提交 | ❌ 不能恢复 | 事务本就没提交 |
| **未提交** | 刚写Redo Log Buffer | 仅在MySQL内存        | ❌ 不能恢复 | 内存数据丢失   |

 

当 MySQL 意外重启，**崩溃恢复：如何裁决“半成功”事务**？它会扫描 Redo Log 中所有处于 **Prepare** 状态但未完成 Commit 的事务，然后拿着这些事务的 XID 去 Binlog 中查找

| 崩溃发生的时刻                                          | Redo Log状态 | Binlog状态    | 恢复时的裁决 | 原因                                                         |
| :------------------------------------------------------ | :----------- | :------------ | :----------- | :----------------------------------------------------------- |
| **场景A**：刚写完 Redo Log Prepare，还没写Binlog 时崩溃 | PREPARE      | 不存在对应XID | **事务回滚** | Binlog 里没有记录，意味这个事务对从库和备份来说是不存在的，所以主库也必须回滚以保证数据一致 |
| **场景B**：刚写完Binlog，还没写 Redo Log Commit 时崩溃  | PREPARE      | 存在对应XID   | **事务提交** | Binlog 已经写成功了，说明这个事务被认可（可以被同步到从库）。如果此时主库回滚，就会导致主从不一致。因此，InnoDB会“补完”这个事务，将其提交 |

但是当以下情况。redo log 已经完成并且被标记为 Commit 状态，并且已经落盘，但是 binlog 还在 cache 中需要等够 1000 次才落盘。此时发生崩溃，binlog 丢失，但 redo log 落盘。**这个事务会被提交，不会回滚**

```sql
innodb_flush_log_at_trx_commit = 1  # redo log 每次事务都刷盘
sync_binlog = 1000                  # binlog cache 每1000次刷盘

# MySQL 在崩溃恢复时会按照以下规则判断事务的命运：
[扫描 Redo Log]
    ↓
[检查每个事务的状态]
    ↓
├─ 如果 Redo Log 有完整的 Prepare + Commit 标记
│   └─ ✅ 直接提交（无论 Binlog 状态）
│
├─ 如果 Redo Log 只有 Prepare，没有 Commit 标记
│   ├─ 检查对应的 Binlog 是否存在且完整
│   ├─ 如果是 → ✅ 提交事务
│   └─ 如果否 → ❌ 回滚事务
│
└─ 如果只有部分 Prepare（不完整）
    └─ ❌ 回滚事务
```

**那么 Binlog 没刷盘影响在哪？**

- 对主库无影响
- 但是对从库/备份的库有风险。如果使用 binlog 恢复从库，且事务丢失了，导致主从不一致



> **Redo Log 与 Binlog 的缓冲区实际配置建议**
>
> **不过强烈建议使用`innodb_flush_log_at_trx_commit=1`**，这是保证 ACID 特性的底线配置

```sql
#金融级配置 (最安全，任何崩溃都不会丢数据，但性能最差)
innodb_flush_log_at_trx_commit = 1  -- 每次提交都刷盘 redolog
sync_binlog = 1                     -- 每次提交都刷盘 binlog
```

```sql
# 平衡配置（大多数业务）
innodb_flush_log_at_trx_commit = 2  -- 提交时 RedoLog 只写 OS Cache，每秒刷盘
sync_binlog = 1000                   -- 每1000次事务提交刷一次binlog
```

```sql
# 高性能配置（可接受丢数据）
innodb_flush_log_at_trx_commit = 0  -- 每秒刷一次redo log
sync_binlog = 0                     -- 由操作系统决定binlog刷盘
```



> **事务回滚时还需要先生成 redo log 吗？**

- 回滚也是一种修改。也面临失败崩溃的风险。只能通过另一个事务反向操作。**回滚生成的Redo Log和普通更新没有区别**。如下示意

  ```sql
  时间线：
  
  [原始事务执行时]
  1. 写入Undo Log: "id=1的原值是25"
  2. 修改内存页: age=26
  3. 写入Redo Log(Prepare): "将id=1改为26"
  
  [执行ROLLBACK时]
  4. 从Undo Log读取原值: 25
  5. 修改内存页: age=25 (这是新的修改！)
  6. 写入新的Redo Log: "将id=1改为25" ← 回滚也需要Redo Log！
  7. 写入Redo Log Commit标记
  8. 返回客户端"回滚成功"
  ```



> **重要问题：事务提交成功了，数据还有丢失的风险吗？**
>
> **有：的保证以下配置**

**1、保证 binlog 和 redo log 事务提交即刷盘**、否则客户端通知成功了。但是 Redo log 还在 Log Buffer 没刷盘。崩溃也丢数据

```sql
-- 双 1 设置（金融级安全）
sync_binlog = 1
innodb_flush_log_at_trx_commit = 1
```

**2、硬件层面的措施**

- 硬件层面的欺骗、例如有些硬盘的 fsync() 实现，仅仅写入了硬盘的缓存就返回成功。突然断电导致数据丢失

- 使用 UPS（不间断电源）：

  - 防止突然断电导致 OS Cache 丢失、给服务器足够时间完成 fsync()

- 使用带电容的 RAID卡：
  - 写缓存有电池保护

  - 即使断电，缓存数据也能持久化



### 7. MVCC 机制与事务机制









