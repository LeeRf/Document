

## 1、Redis 的安装



##### 1、下载安装包

去Redis官网下载 https://redis.io/ 下载安装包、我们下载稳定版本的 5.0 的最新版本



##### 2、用 Xftp 将下载好的按转包拷入到 /opt 目录下

##### 3、解压 Redis 的安装包

进入 /opt 目录下后、输入 Linux 解压命令：

```Linux
tar xzf redis-5.0.12.tar.gz
```



##### 4、安装 Redis 的运行环境

执行以下命令安装环境

```
yum install gcc-c++
```

如果使用该命令报错 

```
make
```

jemalloc/jemalloc.h：没有那个文件或目录、就换成以下命令

```
make MALLOC=libc
```

 之后在进行安装

```
make install
```



##### 5、redis默认不是后台启动的

​		我们需要手动修改配置文件 redis-conf

![image-20210325132811597](C:\Users\19365\AppData\Roaming\Typora\typora-user-images\image-20210325132811597.png)

##### 6、启动 Redis

```
redis-server #指定启动的redis的配置文件  
redis-server lee/redis.conf
```

并用命令检查Redis 服务是否启动

```
ps -ef | grep redis
```

并使用 redis-cli 进行连接测试

![image-20210325133023221](C:\Users\19365\AppData\Roaming\Typora\typora-user-images\image-20210325133023221.png)

7、退出Redis

```
输入 shutdown  然后 exit
```



## 2、Redis 原生命令的使用



Redis五种核心数结构<img src="C:\Users\19365\AppData\Roaming\Typora\typora-user-images\image-20210325164931259.png" alt="image-20210325164931259" style="zoom: 80%;" />

### 1、String 结构

+++

#### String常用命令

-   ###### 字符串常用命令

    ```
    set key value      //存入字符串键值对
    ```

    >   例：set leekey leevalue

    ```
    mset key value [key value...]   //批量存储字符串键值对
    
    ```

setnx key value   //存入一个不存在的字符串键值对
    
    get key           //获取一个字符串键值

mget key [key ...] //批量获取字符串键值
    
    del key [key ...]  //删除一个键

expire key seconds //设置一个键的过期时间(秒)
    ```
    
    

-   原子加减

    ```
    incr key          //将 key 中存储的数字值加1
    ```

    ```
    decr key          //将 key 中存储的数字值减1
    ```

    ```
    incrby key increment //将 key 所存储的值加上 increment
    ```

    >   例 incrby leekey 10 

    ```
    decrby key decrement //将 key 所存储的值减去 decrement
    ```

    

#### String应用场景

##### 1、set key value

​	可以将简单的数据表通过 key value 存储在 redis 中

​	get key                               ![image-20210325182157795](C:\Users\19365\AppData\Roaming\Typora\typora-user-images\image-20210325182157795.png)

##### 2、对象缓存

```
set user:1 value(json格式)
```

>   例如我们经常使用的将 user 用户整体结构转换为 json 格式存入 redis

也可以批量将部分字段存入redis

```
mset user:1:name lee user:1:age 24
```

然后通过批量读取取出

```
mget user:1name user:1:age
```

##### 3、分布式锁（简单实现、现实业务中有更复杂的实现）

```
setnx product:10001 true  //返回1代表获取锁成功
```

```
setnx product:10001 true  //返回0代表获取锁失败
```

执行业务操作代码块

```
del product:10001   //执行完业务释放锁
```

```
set product:10001 true ex 10 nx  //防止程序意外终止导致死锁
```

##### 4、计数器

可以用于文章阅读量的计数

```
incr article:readcount:{文章id}
```

![image-20210325183745269](C:\Users\19365\AppData\Roaming\Typora\typora-user-images\image-20210325183745269.png)

```
get article:readcount:{文章id}
```

##### 5、Web集群 session共享

>   spring session + redis实现session共享

##### 6、分布式系统全局序列号

```
incrby orderId 1000 //redis 批量生成序列号提升性能
```



### 2、Hash 结构

---

#### hash常用的命令

```
hset key field value      //存储一个哈希表key的键值

hsetnx key field value    //在存储一个不存在的哈希表 key 的键值

hmset key field value [field value ...]  //在一个哈希表key中存储多个键值对

hget key  field 		 //获取哈希表key对应的field键值

hmget key field [field ...]  //批量获取哈希表 key 中的多个 field 键值

hdel key field [field ...]   //删除哈希表 key 中的 field 键值

hlen key                  //返回哈希表 key中的 field 的数量

hgetall key               //返回哈希表 key 中的所有的键值

hincrby key field increment //为哈希表 key 中 field 键的值加上增量 increment
```



#### hash的应用场景

##### 对象缓存

```
hmset user {userId}:name zhuge {userId}:balance 1888

hmset user 1:name zhuge 1:balance 1888

hmset user 1:name 1:balance  
```

![image-20210326155645693](C:\Users\19365\AppData\Roaming\Typora\typora-user-images\image-20210326155645693.png)

##### 电商购物车

​	以用户的 id 为 Key、商品 id 为 field、商品数量为 value

​	**购物车操作**

1.  添加商品  hset cart:1001 10088 1
2.  增加数量  hincrby cart:1001 10088 1
3.  商品总数  hlen cart:1001
4.  删除商品  hdel cart:1001 10088
5.  获取购物车所有商品  hgetall cart:1001



#### Hash的优缺点

优点：

1.  同类数据归类整合存储、方便数据管理
2.  相比string操作消耗内存与cpu更小
3.  相比string存储更节省控件

缺点:

1.  过期功能不能使用在 field上、只能用在 key 上
2.  Redis 集群架构下不适合大规模使用

### 3、List 结构

---

#### List 的常用命令

```
lpush key value[value ...]      //将一个或多个值value插入到key列表的表头（最左边）
rpush key value[value ...]      //将一个或多个值value插入到key列表的表尾 (最右边)
lpop  key			            //移除并返回key列表的头元素
rpop  key			            //移除并返回key列表的尾元素
lrange key start stop		    //返回列表key中指定区间内的元素，区间以偏移量start和stop指定
```

```
BLPOP  key  [key ...]  timeout	//从key列表表头弹出一个元素，若列表中没有元素，阻塞等待
```

>    timeout秒,如果timeout=0,一直阻塞等待

```
BRPOP  key  [key ...]  timeout 	//从key列表表尾弹出一个元素，若列表中没有元素，阻塞等待
```

>    timeout秒,如果timeout=0,一直阻塞等待

![image-20210326161246479](C:\Users\19365\AppData\Roaming\Typora\typora-user-images\image-20210326161246479.png)

#### List 应用场景

常用数据结构

>   Stack（栈）= lpush + lpop
>
>   Queue（队列）= lpush rpop
>
>   Blocking MQ（阻塞队列） = lpush + brpop

![image-20210326161541678](C:\Users\19365\AppData\Roaming\Typora\typora-user-images\image-20210326161541678.png)

##### 微博消息和微信公号消息

zhuge老师关注了MacTalk，备胎说车等大V



1）MacTalk发微博，消息ID为10018

>   ​	LPUSH msg:{诸葛老师-ID} 10018



2）备胎说车发微博，消息ID为10086

>   ​	LPUSH msg:{诸葛老师-ID} 10086



3）查看最新微博消息

>   ​	LRANGE msg:{诸葛老师-ID} 0 4



### 4、Set 结构

---

#### Set的常用命令

```
sadd  key  member  [member ...]			//往集合key中存入元素，元素存在则忽略，若key不存在则新建
srem  key  member  [member ...]			//从集合key中删除元素
smembers  key					  //获取集合key中所有元素
scard  key					      //获取集合key的元素个数
sismember  key  member			  //判断member元素是否存在于集合key中
srandmember  key  [count]	      //从集合key中选出count个元素，元素不从key中删除
spop  key  [count]				  //从集合key中选出count个元素，元素从key中删除
```

#### Set运算操作

```
sinter  key  [key ...] 				        //交集运算
sinterstore  destination  key  [key ..]		//将交集结果存入新集合destination中
sunion  key  [key ..] 				        //并集运算
sunionstore  destination  key  [key ...]	//将并集结果存入新集合destination中
sdiff key  [key ...] 				        //差集运算
sdiffstore  destination  key  [key ...]		//将差集结果存入新集合destination中
```

#### Set应用场景

##### 微信抽奖小程序

1、点击参与抽奖加入集合

>   ​		sadd key {userlD}

2、查看参与抽奖所有用户

>   ​		smembers key  

3、抽取count名中奖者

>   ​		srandmember key [count] / spop key [count]

![image-20210326162225665](C:\Users\19365\AppData\Roaming\Typora\typora-user-images\image-20210326162225665.png)

##### 微博微信点赞、收藏、标签

1、点赞

>   ​		SADD like:{消息ID} {用户ID}

2、取消点赞

>   ​		SREM like:{消息ID} {用户ID}

3、检查用户是否点过赞

>   ​		SISMEMBER like:{消息ID} {用户ID}

4、获取点赞的用户列表

>   ​		SMEMBERS like:{消息ID}

5、获取点赞用户数 

>   ​		SCARD like:{消息ID}



##### Set集合操作

SINTER set1 set2 set3 -> { c }

SUNION set1 set2 set3 -> { a,b,c,d,e }

SDIFF set1 set2 set3 -> { a }

![image-20210326162703402](C:\Users\19365\AppData\Roaming\Typora\typora-user-images\image-20210326162703402.png)



##### 集合操作实现微博微信关注模型

1、诸葛老师关注的人: 

>   ​		zhugeSet-> {guojia, xushu}

2、 杨过老师关注的人:

>    		yangguoSet--> {zhuge, baiqi, guojia, xushu}

3、郭嘉老师关注的人: 

>   ​		guojiaSet-> {zhuge, yangguo, baiqi, xushu, xunyu)

4、我和杨过老师共同关注: 

>   ​		SINTER zhugeSet yangguoSet--> {guojia, xushu}

5、我关注的人也关注他(杨过老师): 

>   ​		SISMEMBER guojiaSet yangguo 

>   ​		SISMEMBER xushuSet yangguo

6、我可能认识的人: 

>   ​		SDIFF yangguoSet zhugeSet->(zhuge, baiqi}

![image-20210326162739994](C:\Users\19365\AppData\Roaming\Typora\typora-user-images\image-20210326162739994.png)

### 5、ZSet 有序集合结构

---

#### ZSet 常用命令

```
zadd key score member [[score member]…]	//往有序集合key中加入带分值元素
zrem key member [member …]		        //从有序集合key中删除元素
zscore key member 			            //返回有序集合key中元素member的分值
zincrby key increment member		    //为有序集合key中元素member的分值加上increment 
zcard key				                //返回有序集合key中元素个数
zrange key start stop [WITHSCORES]	    //正序获取有序集合key从start下标到stop下标的元素
zrevrange key start stop [WITHSCORES]	//倒序获取有序集合key从start下标到stop下标的元素
```

#### ZSet 集合常用操作

```
ZUNIONSTORE destkey numkeys key [key ...]  //并集计算
ZINTERSTORE destkey numkeys key [key …]    //交集计算
```

![image-20210326163156799](C:\Users\19365\AppData\Roaming\Typora\typora-user-images\image-20210326163156799.png)

#### ZSet 应用场景



1、点击新闻

>   ​		ZINCRBY hotNews:20190819 1 守护香港

2、展示当日排行前十

>   ​		ZREVRANGE hotNews:20190819 0 9 WITHSCORES 

3、七日搜索榜单计算

>   ​		ZUNIONSTORE hotNews:20190813-20190819 7 
>
>   ​		hotNews:20190813 hotNews:20190814... hotNews:20190819

4、展示七日排行前十

>   ​		ZREVRANGE hotNews:20190813-20190819 0 9 WITHSCORES

![image-20210326163230497](C:\Users\19365\AppData\Roaming\Typora\typora-user-images\image-20210326163230497.png)

## 3、Redis的单线程和高性能

### 1、Redis是单线程吗

Redis 的单线程主要是指 Redis 的网络 IO 和键值对读写是由一个线程来完成的、这也是 Redis 对外提供键值存储服务的主要流程，但 Redis 的其它功能、如持久化、异步删除、集群数据同步等、其实是由额外的线程执行的

### 2、Redis单线程为什么还能这么快

因为它所有的数据都在内存中、所有的运算都是内存级别的运算、而且单线程避免了多线程的切换性能损耗问题、**正因为Redis是单线程、所以小心使用 Redis 指令**，对于那些耗时的指令（比如 keys）、一定要谨慎使用、一不小心就可能会导致 Redis 卡顿

### 3、Redis 单线程如何处理那么多的并发客户端连接？

Redis的IO多路复用：redis利用epoll来实现IO多路复用、将连接信息和事件放到队列中、依次放到文件事件分派器、事件分派器将事件分派给事件处理器

![image-20210326165301711](C:\Users\19365\AppData\Roaming\Typora\typora-user-images\image-20210326165301711.png)

### 4、其它高级命令

#### **keys**

>   全量遍历键，用来列出所有满足特定正则字符串规则的key，当redis数据量比较大时， 性能比较差，要避免使用

#### **scan**

>   渐进式遍历键 SCAN cursor [MATCH pattern] [COUNT count] scan 参数提供了三个参数，第一个是 cursor 整数值(hash桶的索引值)，第二个是 key 的正则模式， 第三个是一次遍历的key的数量(参考值，底层遍历的数量不一定)，并不是符合条件的结果数量。第 一次遍历时，cursor 值为 0，然后将返回结果中第一个整数值作为下一次遍历的 cursor。一直遍历 到返回的 cursor 值为 0 时结束。 注意：但是scan并非完美无瑕， 如果在scan的过程中如果有键的变化（增加、 删除、 修改） ，那 么遍历效果可能会碰到如下问题： 新增的键可能没有遍历到， 遍历出了重复的键等情况， 也就是说 scan并不能保证完整的遍历出来所有的键， 这些是我们在开发时需要考虑的。

#### **Info**

>   查看redis服务运行信息，分为 9 大块，每个块都有非常多的参数，这 9 个块分别是:

```
Server 服务器运行的环境参数
Clients 客户端相关信息
Memory 服务器运行内存统计数据
Persistence 持久化信息
Stats 通用统计数据 
Replication 主从复制相关信息
CPU 使用情况
Cluster 集群信息
KeySpace 键值对统计数量信息
```

```
connected_clients:2            # 正在连接的客户端数量
instantaneous_ops_per_sec:789  # 每秒执行多少次指令
used_memory:929864             # Redis分配的内存总量(byte)，包含redis进程内部的开销和数据占用的内存
used_memory_human:908.07K      # Redis分配的内存总量(Kb，human会展示出单位)

used_memory_rss_human:2.28M    # 向操作系统申请的内存大小(Mb)（这个值一般是大于used_memory的，因为Redis的内存分配策略会产生内存碎片）

used_memory_peak:929864         # redis的内存消耗峰值(byte)
used_memory_peak_human:908.07K  # redis的内存消耗峰值(KB)
maxmemory:0                     # 配置中设置的最大可使用内存值(byte),默认0,不限制
maxmemory_human:0B              # 配置中设置的最大可使用内存值
maxmemory_policy:noeviction     # 当达到maxmemory时的淘汰策略
```







