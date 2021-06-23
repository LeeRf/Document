## 1、MySql 的整体架构



### 1、MySQL 的架构图



如果能在头脑中构建出一幅 MySQL 各组件之间如何协同工作的架构图，就会有助于深入理解 MySQL 服务器。如下图展示了 MySQL 的逻辑架构图

![image-20210622193701363](MySQL 进阶.assets/image-20210622193701363.png)

### 2、最上层 <客户端>



最上层的服务并不是 MySQL 所独有的，大多数基于网络的客户端/服务器的工具或者服务都有类似的架构。比如连接处理、授权认证、安全等等。



### 3、第二层 <查询缓存 | 解析器>



括连接器、查询缓存、分析器、优化器、执行器等，涵盖 mysql 的大多数核心服务功能，以及所有的内置函数（例如日期、世家、数 学和加密函数等），所有跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图等



### 4、第三层 <存储引擎>



负责数据的存储和提取，是真正与底层物理文件打交道的组件。 数据本质是存储在磁盘上的，通过特定的存储引擎对数据进行有组织的存放并根据业务需要对数据进行提取。存储引擎的架构模式是插件式的，支持 Innodb，MyIASM、Memory 等多个存储引擎。现在最常用的存储引擎是 Innodb，它从 mysql5.5.5 版本开始成为了默认存储引擎



### 5、第四层 <物理层面>



存储数据库真正的表数据、日志等。物理文件包括：redolog、undolog、binlog、errorlog、querylog、slowlog、data、index等



## 2、MySql 索引结构详解：



### 1、索引的通俗介绍？



#### 1、索引是什么？



>   索引是帮住 MySql（数据库） 高校获取数据的 **排好序 的 数据结构**

![image-20210622150545531](MySQL 进阶.assets/image-20210622150545531.png)



#### 2、索引为什么不是其它数据结构？



>   思考：索引数据结构为什么不是：二叉树、红黑树、Hash表、B - Tree ？
>

因为二叉树首先极端情况下会退化为链表（比如插入依次递增的数据）并且高度不可控就意味者磁盘 IO 次数增加消耗性能、平衡二叉树和红黑树虽然解决了退化成了链表的问题、但是高度依然不可控、在大数据量的情况下磁盘 IO 性能拉跨、正好 B Tree 结构横向扩展了节点的存储空间、就可以放更多的索引元素，既节省了 IO 次数、又极大的提升了整体性能



### 2、B-Tree 类的数据结构



#### 1、B - Tree 介绍



-   叶子几点具有相同的深度
-   叶子节点的指针为空
-   所有索引元素不重复
-   节点中的数据索引从左到右递增排列

![image-20210622151026695](MySQL 进阶.assets/image-20210622151026695.png)



#### 2、B + Tree 介绍



-   非叶子节点不存储 data 、只存储索引、可以放更多的索引
-   叶子节点包含所有索引字段
-   叶子节点用指针连接、提高区间访问的性能

![image-20210622151321324](MySQL 进阶.assets/image-20210622151321324.png)

B+ Tree 其实在叶子节点还有前后指向两边的指针、指针指向下一个叶子节点的磁盘上的位置信息，一个节点的索引数据 MySql  默认分配的是 16 kb 大小、一般 B+ Tree 的一个节点 Load 到内存等于做一次磁盘 IO 和 在内存中定位数据所在节点中的位置要慢很多，因为在内存中节点数据也是从左到右依次递增的、所以可以采取一些算法（比如折半查找）耗费的性能几乎可以忽略不记

>   那么 16 kb 的页大小可以放多少个索引呢？下面以 BigInt（ 在 MySql 中占用 8 个 Byte ） 类型来算
>

每个节点元素里还会存储下一个节点的磁盘指针地址、大概占用 6 Byte，那么就是 16kb / (8 + 6) Byte ≈ 1170 个索引元素、这是一个页大小还不是叶子节点、假如叶子节点数据很大、一行的记录几十个字段一般不会超过 1 kb 的大小加上前后指针顶多 12 Byte 、一张 B+ Tree 数据结构能存储 1170 * 1170 * 16 ≈ 2190 万条记录、那么通过 3 次磁盘 IO 就可以在两千万多条记录定位到一条指定的数据

可通过以下 Sql 查看页码大小 (不推荐修改) 

```java
Show Global Status Like 'Innodb_page_size';
```



#### 3、索引在磁盘上的存储位置



>   默认在 MySql 的安装目录中的 data 目录中

![image-20210622151610387](MySQL 进阶.assets/image-20210622151610387.png)

>   一般来说一个库对应一个文件夹：如下图直观所示

![image-20210622151636230](MySQL 进阶.assets/image-20210622151636230.png)



>   一个库里的数据库磁盘文件就在这个文件夹下，存储了包括数据、表结构、索引等信息

![image-20210622185657432](MySQL 进阶.assets/image-20210622185657432.png)

#### 4、B+ Tree 和 B - Tree 的区别 ？



>   B+ Tree 和 B - Tree 的区别 ？以及 MySql 为什么用 B+ Tree 作为索引的数据结构 ?



首先最大的区别是 B - Tree 的节点每个都是存储数据的，而 B+ Tree 只有叶子节点才存储数据

这样会有什么问题呢？最大的问题就是假如每行的索引数据 1 kb 大小放到 B - Tree 中，每页的大小 16kb、那么每个节点只能存储 16 个元素、那么 2000 万的数据存储到这里高度不可控、高度会达到一个很大的数字、这时候你在做磁盘IO 一次一次定位性能严重拉跨、再一个 B+ Tree 支持叶子节点的区间访问、因为存储了前后两个节点的磁盘指针地址



第二：就是 MySql 在 B+ Tree 的叶子节点实现了类似双向循环链表一样的指针，相互指向前后的元素，这样 B+ Tree 在做范围查询的时候，在叶子节点可以直接获取相邻节点的数据，无需从根节点重新查找，而 B - Tree 就不支持这样的特性



### 3、 MylSAM 和 InnoDB 存储引擎



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

因为 InnoDB 模式下一张表的构建本来就是按照 B+ Tree 结构来构建的整一张表、有主键的话就按照主键来建造索引、如果没有主键 MySql 会从第一列开始往后选每个元素都不想等的列作为索引，如果找不到 MySql 会自己造一个隐藏列来当索引。



>   2、为什么推荐使用整型的自增主键 ？

因为在通过索引查找一个元素的时候，需要比对数据的大小和区间来定位数据的所在位置，整形的比对速度要比字符串的速度要高很多

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



### 4、MySql 的 Hash 索引结构



#### 1、Hash 索引结构介绍



默认来说 MySql 的索引结构是 B+Tree 结构的，下面还有 HASH 结构的索引

![image-20210622194909077](MySQL 进阶.assets/image-20210622194909077.png)

>   对索引的 key 进行一次 hash 激素那就可以定位出数据存储的位置，很多时候 hash 索引要比 B+ 树索引更高效、缺点仅能满足 “=”、“IN”，不支持范围查询，hash 冲突问题、具体存储结构如下

![image-20210622194934476](MySQL 进阶.assets/image-20210622194934476.png)



### 5、索引的最左前缀原理



#### 1、联合索引底层结构是怎样的 ？



如下图所示的联合索引的存储结构，以下是联合主键索引的示例

![image-20210622195056651](MySQL 进阶.assets/image-20210622195056651.png)

建立联合索引的时候 MySql 会根据建立索引字段的先后顺序来建立索引的，在跳过了 name 字段，在 name 字段的值相等的区间是有序的，但是在整张表中跳过了第一个有序字段，后续字段无法保证其顺序性，所以走不了索引。例如以下的 Sql 会走索引的是第一条，因为你不能跳过第一个联合索引直接查 age 或者 position 字段就破坏了索引的有序性，这样是不会走索引的



>   跳过第一个联合索引的字段就不能走索引？

因为 MySql 构建 B+ Tree 联合索引的时候实先保证第一个索引的有序性进行构建，这样一来，剩余的索引字段就不一定有序，既然不能保证后续索引的有序性在跳过了第一个索引字段后因为无序性就不能通过索引查询、意味着只有通过全表扫描才能不遗漏的找到查询数据



## 3、Explain 分析与索引最佳实践



### 1、Explain 分析工具的简介



#### 1、Explain 官方文档介绍



使用 EXPLAIN 关键字可以模拟优化器执行 SQL 语句，分析你的查询语句或是结构的性能瓶颈. 在 Select 语句之前增加 Explain 关键字，MySQL 会在查询上设置一个标记执行查询会返回执行计划的信息，而不是执行这条SQL、注意：如果 from 中包含子查询，仍会执行该子查询，将结果放入临时表中

>   MySql 推荐版本 5.7.29 
>
>   Explain 官方使用文档链接：https://dev.mysql.com/doc/refman/5.7/en/explain-output.html



#### 2、Explain 的两个变种



explain extended：会在 explain 的基础上额外提供一些查询优化的信息，紧随其后通过 Show warnings 命令可以得到优化后的查询语句，从而看出优化器优化了什么额外还有 filtered 列，是一个半分比的值，rows * filtered / 100 可以估算出将要和 explain 中的前一个表进行连接的行数（前一个表指 explain 中的值比当前表 id 值小的表）

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



### 2、Explain 分析中的列



接下来我们将展示 explain 中每个列的信息



#### 1、id 列



id 列的编号是 select 的序列号、有几个 select 就有几个 id，并且 id 的顺序是按 select 出现的顺序增长的。**id 列越大执行优先级越高**、id 相同则从上往下依次执行、其中 id 为 NULL 意味者最后执行



#### 2、select_type 列



>   表示对应行是简单还是复杂的查询其中包含以下类型（常用）：

-   **simple**：简单查询，查询不包含子查询和 union、例如 Sql 语句

```sql
mysql > explain select * from film where id = 2
```

-   **primary**：复杂查询中最外层的 select
-   **subquery**：包含在 select 中的子查询（不在 from 子句中）
-   **derived**：包含在 from 自居中的子查询、MySql 会将结果存放在一个临时表中，也成为衍生表（derived 的英文含义）
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



这一列表示 explain 的一行正在访问哪个表、当 from 子句中有子查询时，table 列是 <derivedN> 格式，表示当前查询依赖 id = N 的查询

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



MySql 能对查询的某部分进行优化并将其转化成一个常量，**用于 primary key 或 unique key 的所有列与常数比较时**，所以表最多有一个匹配行，读取 1 次，速度比较快，**system 是 const 的特例，即表里只有一条元素匹配时为 system**

```sql
mysql > explain extended select * from (select * from film where id = 1) tmp;
```

>   分析结果如下：

![image-20210622205502369](MySQL 进阶.assets/image-20210622205502369.png)



##### 3、eq_ref



primary key 或者 unique key 索引的所有部分被连接使用、最多只会返回一条符合条件的记录，这可能是在 const 之外最好的联接类型了，**大白话就是用到了表的主键进行关联联合查询的类型**

```sql
mysql> explain select * from film_actor left join film on film_actor.film_id = film.id;
```

>   分析结果如下：

![image-20210622205815659](MySQL 进阶.assets/image-20210622205815659.png)



##### 4、ref



相比 rq_ref、不使用唯一索引，而是**使用普通索引或者唯一索引的部分前缀，索引要和某个值比较等**，可能会找到多个符合条件的行、如以下简单的查询语句，name 是普通索引（非唯一索引）

```sql
mysql > explain select * from film where name = 'film1';
```

![image-20210622210015578](MySQL 进阶.assets/image-20210622210015578.png)



>   关联表查询，possible_keys 的值 idx_film_actor_id 是 film_id 和 actor_id 的联合索引，这里使用到了 film_actor 的左边前缀 film_id 部分

```sql
mysql >  explain select film_id from film 
                     left join film_actor on film.id = film_actor.film_id;
```

![image-20210622210302431](MySQL 进阶.assets/image-20210622210302431.png)



##### 5、range



**范围扫描**通 常出现在 in()、between、>、<、>= 等操作中，使用一个索引来检索给定范围的行

```sql
mysql > explain select * from actor where id > 1
```

![image-20210622210429693](MySQL 进阶.assets/image-20210622210429693.png)



##### 6、index



**扫描全索引就能拿到结果**，一般是扫描某个二级索引、**这种扫描不会从索引树根节点开始快速查找，而是直接对二级索引的叶子节点遍历和扫描**，速度还是还是比较慢的、这种查询一般为使用覆盖索引、二级索引一般比较小、所以这种通常比 ALL 快一些

```sql
mysql > explain select * from film;
```

![image-20210622210712318](MySQL 进阶.assets/image-20210622210712318.png)



film 表只有 id 和 name 两个字段、其中 id 是 主键索引、name 是普通（二级）索引，即该表所有字段都是索引，此时 key 列的值 是 idx_name 的二级索引

>   MySql 会进行优化，优先选择二级索引，这个叫做覆盖索引，因为二级索引的叶子节点存储的是主键索引的值 而不是 主键索引那样存储整一行的数据，这样不同扫全部列



##### 7、ALL



即全表扫描，扫描你的聚集索引的所有叶子节点、通常情况下这需要增加索引来进行优化了

```sql
mysql > explain select * from actor;
```

![image-20210622210854197](MySQL 进阶.assets/image-20210622210854197.png)



#### 5、possible_key 列



**这一列显示查询可能使用哪些索引来查找**

explain 时可能出现 possible_keys 有列，而 key 显示 NULL 的情况，这种情况是因为表中数据不多，mysql 认为索引对此查询帮助不大，选择了全表查询如果该列是 NULL，则没有相关的索引、在这种情况下，可以通过检查 where 子句看是否可以创造一个适当的索引来提高查询性能，然后用 explain 查看效果



#### 6、key 列



**这一列显示 mySql 实际采用了哪个索引来优化对该表的访问**，如果没有使用索引，则该列是 NULL，如果想强制 MySql 使用或者忽视 possible_keys 列中的索引，在查询中使用 forceindex、ignore index



#### 7、key_len 列



>   这一列显示了 mysql  在索引里使用的字节数、通过这个值可以算出具体使用了索引中的哪些列
>

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



>   这一列显示了在 key 列记录的索引中、表查找的时候实际所用到的列或常量、常见的有：const（常量）、字段名（例：film.id）



#### 9、rows 列



>   这一列是 mysql 估计要读取并检测的行数、注意这个不是结果集里的行数



#### 10、extra 列 (常见的)



这一列展示的是额外信息、常见的重要值如下：



##### 1、Using index：



覆盖索引定义：MYSQL 执行计划 explain 结果里的 key 有使用索引、如果 select 后面查询的字段都可以从这个索引的树中获取、这种情况一般可以说是用到了覆盖索引、extra 里一般都有 using index；覆盖索引一般针对的是辅助索引，整个查询结果只通过辅助索引就能拿到结果，不需要通过辅助索引树找到主键，再通过主键去主键索引树里获取其它字段值

```sql
mysql > explain select film_id from film_actor where film_id = 1;
```

![image-20210622212433639](MySQL 进阶.assets/image-20210622212433639.png)



##### 2、Using where：



使用 where 语句来处理结果、并且查询的列未被索引覆盖，也没有用到索引（需要优化的）

```sql
mysql> explain select * from actor where name = 'a';
```

![image-20210622212555637](MySQL 进阶.assets/image-20210622212555637.png)



##### 3、Using index condition：



查询的列不完全被索引覆盖、where 条件中是一个前导列的范围（比如联合索引只用到了第一个做范围查找）

```sql
mysql> explain select * from film_actor where film_id > 1;
```



##### 4、Using temporary：



>   MySql 需要创建一张临时表来处理查询结果、出现这种情况一般是要进行优化的，首先是想到用索引来优化



如下 SQL 语句 actor.name 没有建立索引、此时创建了张临时表来 distinct 进行去重对比

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



>   将使用外部排序而不是索引排序，数据较小时从内存排序，否则需要在磁盘完成排序、这种情况下一般也是要考虑使用索引来优化的、如下SQL示例 

actor.name 未创建索引，会浏览 actor 整个表，保存关键字 name 和对应的 id，然后排序 name 并检索记录

```sql
mysql > explain select * from actor order by name;
```

![image-20210622212956898](MySQL 进阶.assets/image-20210622212956898.png)

如果将 actor.name 建立索引，此时执行计划 Extra 的值变成了 Using Index



##### 6、Select tables optimized away



使用某些聚合函数（ 比如 max、min ）来访问存在索引的某个字段时

```sql
mysql> explain select min(id) from film;
```

![image-20210622213127967](MySQL 进阶.assets/image-20210622213127967.png)



### 3、初识索引的最佳实践



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



>   例如：不要在索引上做 计算、函数、自动 OR 手动 类型转换，会导致索引失效而转向全表扫描

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

>   我们看到该 SQL 分析的结果，possible_keys 列 mysql 认为可以走索引，但是最终没走索引，mysql 在走索引的时候，会对索引的效率做一些评估，但是如果全表扫描比索引书还快一些（二级索引查询还需要回表扫描，效率会慢些），实际就不会走索引。

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



## 4、MySql 索引优化实战篇



### 0、测试数据准备阶段



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



### 1、MySQL选择合适索引的策略？



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
>       -   预估全表扫描情况、扫描行数、查询成本
>       -   预估使用到的索引类型、索引的适用范围、是否使用覆盖索引，索引扫描行数、索引使用成本
>       -   除此之外还会记录表的最优访问记录、最终选择的访问路径等策略

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



### 2、联合索引常见优化方案



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



对于辅助的联合索引 (name, age, position)，正常情况按照最左前缀原则，以上的 SQL 只会走 name 字段索引，因为根据 name 字段过滤完，得到的索引行里的 age 和 position 是无序的，无法很好的利用索引

-   在 MySQL5.6 之前的版本，这个查询只能在 联合索引里匹配到名字是 'Lee' 开头的索引
-   然后**拿这些索引对应的主键逐个回表**，到主键索引上找出相应的记录
-   最后比对 age 和 position 这两个字段的值是否符合

MySQL 5.6 引入了索引下推优化，可以在索引遍历过程中，对索引中包含的所有字段先做判断，过滤掉不符合条件的记录之后再回表，可以有效的减少回表次数。使用了索引下推优化后，上面那个查询在联合索引里匹配到名字是 'Lee' 开头的索引之后，同时还会在索引里过滤 age 和 position 这两个字段，拿着过滤完剩下的索引对应的主键 id 再回表查整行数据

弄懂了索引下推，其实还不如起个大白话名字 叫做索引条件提前过滤

>   注意：索引下推会减少回表次数，对于 innodb 引擎的表**索引下推只能用于二级索引**，innodb 的主键索引（聚簇索引）树叶子节点上保存的是全行数据，所以这个时候索引下推并不会起到减少查询全行数据的效果



##### 4、为什么范围查找没有用到索引下推优化？



估计应该是 MySql 认为范围查找过滤的结果集过大，like Lee% 在绝大多数情况来看，过滤后的结果集比较小，所以这里 Mysql 选择给 like Lee% 用了索引下推优化，当然这也不是绝对的，有时 like Lee% 也不一定就会走索引下推



### 3、Using filesort 文件排序原理



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



### 4、Order By 和 Group By 优化



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

分析：查找只用到了 name, 在 name 有序的情况下，后两位也可以保持有序，所以 age 和 position 用于排序、无 Using filesort



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



### 5、分页查询优化实例



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



该 SQL 表示查询从第 90001 开始的五行数据，没添加单独 order by，表示通过主键排序。我们再看表 employees ，因为主键是自增并且连续的，所以可以改写成按照主键去查询从第 90001 开始的五行数据，如下





##### 2、根据非主键字段排序的分页查询





### 6、Join 关联查询优化





### 7、索引的设计、原则







### 8、索引设计实战







### 9、 MySQL 规范手册（Alibaba）













