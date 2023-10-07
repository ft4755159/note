# 1.Redis概述

## 1.1.Redis 介绍

- Redis 是一个开源的 key-value 存储系统。
- 和 Memcached 类似，它支持存储的 value 类型相对更多，包括 string (字符串)、list (链表)、set (集合)、zset (sorted set –有序集合) 和 hash（哈希类型）。
- 这些数据类型都支持 push/pop、add/remove 及取交集并集和差集及更丰富的操作，而且这些操作都是原子性的。
  在此基础上，Redis 支持各种不同方式的排序。
- 与 memcached 一样，为了保证效率，数据都是缓存在内存中。
- 区别的是 Redis 会周期性的把更新的数据写入磁盘或者把修改操作写入追加的记录文件。
- 并且在此基础上实现了 master-slave (主从) 同步。

## 1.2.应用场景

**配合关系型数据库做高速缓存**

- 高频次，热门访问的数据，降低数据库 IO。

- 分布式架构，做 session 共享。

![image-20230928110940805](C:\Users\syx\AppData\Roaming\Typora\typora-user-images\image-20230928110940805.png)



**多样的数据结构存储持久化数据**

![image-20230928111036732](C:\Users\syx\AppData\Roaming\Typora\typora-user-images\image-20230928111036732.png)

## 1.3.相关技术

**Redis 使用的是单线程 + 多路 IO 复用技术：**

多路复用是指使用一个线程来检查多个文件描述符（Socket）的就绪状态，比如调用 select 和 poll 函数，传入多个文件描述符，如果有一个文件描述符就绪，则返回，否则阻塞直到超时。得到就绪状态后进行真正的操作可以在同一个线程里执行，也可以启动线程执行（比如使用线程池）。

**串行 vs 多线程 + 锁（memcached） vs 单线程 + 多路 IO 复用 (Redis)**（与 Memcache 三点不同：支持多数据类型，支持持久化，单线程 + 多路 IO 复用） 。

![image-20230928144814738](C:\Users\syx\AppData\Roaming\Typora\typora-user-images\image-20230928144814738.png)

![image-20230928144905732](C:\Users\syx\AppData\Roaming\Typora\typora-user-images\image-20230928144905732.png)

## 1.4.安装

环境：Centos7

1 查看gcc版本，没有就安装

`gcc --version`

`yun install -y gcc`

2 下载 redis-6.2.1.tar.gz 放在/opt 下

3 解压：`tar -zxvf redis-6.2.1.tar.gz`

4 解压完成进入目录：`cd edis-6.2.1`

5 在 redis-6.2.1 目录执行 make 命令（只是编译好）

6 如果没有C语言环境，make 会报错

7 解决方案：运行 make distclean

8 在 redis-6.2.1 目录再次执行 make 命令（只是编译好）

9 跳过 make test 继续执行：make install

安装目录：/usr/local/bin

![image-20230928113822471](C:\Users\syx\AppData\Roaming\Typora\typora-user-images\image-20230928113822471.png)

## 1.5.操作

前台启动，命令行关闭，redis就会停止（不建议）

`redis-server`

**后台启动（推荐）**

1 备份 redis.conf 到其他目录

`cp /opt/redis-6.2.1/redis.conf /etc`

2 后台启动设置 daemonize no 改为 yes

`vim /etc/redis.conf` 查找 /daem  

3 Redis 启动

```bash
# 需要进入安装目录启动
[root@hecs-233798 redis-6.2.1]# cd /usr/local/bin/
[root@hecs-233798 bin]# ls
cloud-id    cloud-init-per  easy_install      jsondiff   jsonpointer  normalizer       redis-check-aof  redis-cli       redis-server
cloud-init  dump.rdb        easy_install-3.6  jsonpatch  jsonschema   redis-benchmark  redis-check-rdb  redis-sentinel
[root@hecs-233798 bin]# redis-server /etc/redis.conf 
[root@hecs-233798 bin]# 

# 进入客户端
[root@hecs-233798 bin]# redis-cli
127.0.0.1:6379> 
127.0.0.1:6379> ping
PONG
```

4 Redis 关闭

单实例关闭：`redis-cli shutdown`

也可进入终端输入 shutdown

多实例关闭：kill -9 [pid] 或者 redis-cli -p 6379 shutdown

## 1.6.配置文件详解

1 注释掉bind（支持远程访问）

![image-20231007211558173](C:\Users\syx\AppData\Roaming\Typora\typora-user-images\image-20231007211558173.png)

2 protected-mode yes 改为 no （关闭保护模式）

3 daemonize no 改为 yes （后台启动）

其他

logfile "" 输出日志路径



# 2.Redis 数据类型

## Redis 键（key）

keys * # 查看当前库所有 key

exists <key> # 判断某个 key 是否存在

type <key> # 查看 key 的类型

del <key> # 删除指定 key 数据

**unlink <key> ** # 根据 value 选择非阻塞删除：仅将 keys 从 keyspace 元数据中删除，真正删除会在后续异步执行

expire <key> <sec> # 为给定的 key 设置过期时间10秒

ttl <key> # 查看还有多少秒过期，-1永不过期，-2已过期

select <0-15># 切换数据库

dbsize # 查看当前数据库的 key 数量

flushdb # 清空当前库

flushall # 通杀全部库

## 2.1.Redis 字符串（String）

### 概述

- String 是 Redis 最基本的类型，你可以理解成与 Memcached 一模一样的类型，一个 key 对应一个 value。
- String 类型是**二进制安全**的。意味着 Redis 的 string 可以包含任何数据。比如 jpg 图片或者序列化的对象。
- String 类型是 Redis 最基本的数据类型，一个 Redis 中字符串 value 最多可以是 **512M**。

### 命令

set <key> <value>	# 添加键值对

get <key> # 查询对应键的值

append <key> <value> # 将给定的 value 追加到原 value 的末尾

strlen <key> # 获取 value 的长度

setnx <key> <value> # 在 key 不存在时，设置 key value

incr <key> # 将 key 的数字值增加1 ，只能对数字值操作，如果为空那么新值为1

decr <key> # 将 key 的数字值增加1 ，只能对数字值操作，如果为空那么新值为-1

incrby/decrby <key> <step> # 增减 key 的数字值，step 设置步长

mset <k1> <v1> <k2> <v2> ... # 同时设置一个或多个key-value 对

mget <k1> <k2> <k3>... # 同时获取一个或多个value

msetnx <k1> <v1> <k2> <v2> ... # 同时设置一个或多个 key-value 对，当且仅当所有给定 key 都不存在。
**原子性，有一个失败则都失败**

getrange <key> <起始位置> <结束位置> # 获得值的范围，类似java 中的 substring，前包，后包。

```bash
127.0.0.1:6379> set name lucyhahaha
OK
127.0.0.1:6379> getrange name 0 3
"lucy"
```

setrange <key> <起始位置> <value> # 用<value>覆写<key>所储存的字符串值，从<起始位置>开始(**索引从0开始**)。

```bash
127.0.0.1:6379> setrange name 3 abc
(integer) 10
127.0.0.1:6379> get name
"lucabchaha"
```



**setex <key> <过期时间> <value>** # 设置键值的同时，设置过期时间，单位秒。
getset <key> <value> # 以新换旧，设置了新值同时获得旧值。

```bash
127.0.0.1:6379> getset name jack
"lucabchaha"
127.0.0.1:6379> get name
"jack"
```

### 数据结构
String 的数据结构为简单动态字符串 (Simple Dynamic String, 缩写 SDS)，是可以修改的字符串，内部结构实现上类似于 Java 的 ArrayList，采用预分配冗余空间的方式来减少内存的频繁分配.

![image-20230928163223630](C:\Users\syx\AppData\Roaming\Typora\typora-user-images\image-20230928163223630.png)

如图中所示，内部为当前字符串实际分配的空间 capacity 一般要高于实际字符串长度 len。当字符串长度小于 1M 时，扩容都是加倍现有的空间，如果超过 1M，扩容时一次只会多扩 1M 的空间。需要注意的是字符串最大长度为 512M。



## 2.2.Redis 列表（List)

### 概述

单键多值：**Redis** 列表是简单的字符串列表，按照插入顺序排序。你可以添加一个元素到列表的头部（左边）或者尾部（右边）。

它的底层实际是个**双向链表**，对两端的操作性能很高，通过索引下标的操作中间的节点性能会较差。


![image-20231007093440428](C:\Users\syx\AppData\Roaming\Typora\typora-user-images\image-20231007093440428.png)

### 命令
lpush/rpush<key><value1><value2><value3>....从左边/右边插入一个或多个值。

```bash
127.0.0.1:6379> lpush k1 v1 v2 v3
(integer) 3
127.0.0.1:6379> rpush k2 v1 v2 v3
(integer) 3
127.0.0.1:6379> lrange k1 0 -1
1) "v3"
2) "v2"
3) "v1"
127.0.0.1:6379> lrange k2 0 -1
1) "v1"
2) "v2"
3) "v3"
```

lpop/rpop<key>从左边/右边吐出一个值。**值在键在，值光键亡**。

```bash
127.0.0.1:6379> lpop k1
"v3"
127.0.0.1:6379> rpop k2
"v3"
```

rpoplpush<key1><key2>从<key1>列表右边吐出一个值，插到<key2>列表左边。

```bash
127.0.0.1:6379> lpush k1 v1 v2 v3
(integer) 3
127.0.0.1:6379> rpush k2 v11 v12 v13
(integer) 3
127.0.0.1:6379> rpoplpush k1 k2
"v1"
127.0.0.1:6379> lrange k2 0 -1
1) "v1"
2) "v11"
3) "v12"
4) "v13"
```

<img src="C:\Users\syx\AppData\Roaming\Typora\typora-user-images\image-20231007101919983.png" alt="image-20231007101919983" style="zoom:50%;" />



lrange <key><start><stop>       取值，按照索引下标获得元素（从左到右）

lrange mylist 0 -1      0左边第一个，**-1右边第一个，(0 -1表示获取所有)**

lindex<key><index>     按照索引下标获得元素（从左到右，从0开始）

```bash
127.0.0.1:6379> lrange k2 0 -1
1) "v1"
2) "v11"
3) "v12"
4) "v13"
127.0.0.1:6379> lindex k2 0
"v1"
127.0.0.1:6379> lindex k2 2
"v12"
```

llen<key>     获得列表长度

```bash
127.0.0.1:6379> llen k2
(integer) 4
```

linsert<key>before<value><newvalue>        在<value>的后面插入<newvalue>。

```bash
127.0.0.1:6379> linsert k2 before "v11" "left"
(integer) 5
127.0.0.1:6379> linsert k2 after "v11" "right"
(integer) 6
127.0.0.1:6379> lrange k2 0 -1
1) "v1"
2) "left"
3) "v11"
4) "right"
5) "v12"
6) "v13"
```

lrem<key><n><value>    从左边删除n个value（从左到右）

```bash
127.0.0.1:6379> lrange k2 0 -1
1) "v1"
2) "new"
3) "v11"
4) "new"
5) "v12"
6) "new"
7) "v13"
127.0.0.1:6379> lrem k2 2 'new'
(integer) 2
127.0.0.1:6379> lrange k2 0 -1
1) "v1"
2) "v11"
3) "v12"
4) "new"
5) "v13"
```

lset<key><index><value>将列表key下标为index的值替换成value

```bash
127.0.0.1:6379> lset k2 1 hello
OK
127.0.0.1:6379> lrange k2 0 -1
1) "v1"
2) "hello"
3) "v12"
4) "new"
5) "v13"
```

### 数据结构
List 的数据结构为快速链表 quickList。

首先在列表元素较少的情况下会使用一块连续的内存存储，这个结构是 ziplist，也即是压缩列表。它将所有的元素紧挨着一起存储，分配的是一块连续的内存。

<img src="C:\Users\syx\AppData\Roaming\Typora\typora-user-images\image-20231007105320976.png" alt="image-20231007105320976" style="zoom: 67%;" />

当数据量比较多的时候才会改成 quicklist。因为普通的链表需要的附加指针空间太大，会比较浪费空间。比如这个列表里存的只是 int 类型的数据，结构上还需要两个额外的指针 prev 和 next。

![image-20231007105140965](C:\Users\syx\AppData\Roaming\Typora\typora-user-images\image-20231007105140965.png)

Redis 将链表和 ziplist 结合起来组成了 quicklist。也就是将多个 ziplist 使用双向指针串起来使用。这样既满足了快速的插入删除性能，又不会出现太大的空间冗余。



## 2.3.Redis 集合（Set）

### 概述
Redis set 对外提供的功能与 list 类似，是一个列表的功能，特殊之处在于 set 是可以**自动排重**的，当你需要存储一个列表数据，又不希望出现重复数据时，set 是一个很好的选择，并且 set 提供了判断某个成员是否在一个 set 集合内的重要接口，这个也是 list 所不能提供的。

Redis 的 Set 是 string 类型的**无序集合**。**它底层其实是一个 value 为 null 的 hash 表**，所以添加，删除，查找的 **复杂度都是 O (1)**。

一个算法，随着数据的增加，执行时间的长短，如果是 O (1)，数据增加，查找数据的时间不变。

### 命令

sadd <key><value1><value2>....
将一个或多个member元素加入到集合key中，已经存在的member元素将被忽略

```bash
127.0.0.1:6379> sadd k1 v1 v2 v3
(integer) 3
```

smembers<key>				取出该集合的所有值。

```bash
127.0.0.1:6379> smembers k1
1) "v3"
2) "v1"
3) "v2"
```

sismember<key><value> 			判断集合<key>是否为含有该<value>值，有1，没有0

```bash
127.0.0.1:6379> sismember k1 v1
(integer) 1
127.0.0.1:6379> sismember k1 v11
(integer) 0
```

scard<key>						返回该集合的元素个数。

```bash
127.0.0.1:6379> scard k1
(integer) 3
```

srem<key><value1><value2>….				删除集合中的某个元素。

```bash
127.0.0.1:6379> srem k1 v1 v2
(integer) 2
127.0.0.1:6379> smembers k1
1) "v3"
```

spop<key>					随机从该集合中吐出一个值。

```bash
127.0.0.1:6379> sadd k2 v1 v2 v3 v4
(integer) 4
127.0.0.1:6379> spop k2
"v2"
127.0.0.1:6379> spop k2
"v4"
```

srandmember<key><n>				随机从该集合中取出n个值。不会从集合中删除。

```bash
127.0.0.1:6379> srandmember k2 2
1) "v3"
2) "v1"
```

smove<source><destination>value把集合中一个值从一个集合移动到另一个集合。

```bash
127.0.0.1:6379> sadd k1 v1 v2 v3
(integer) 3
127.0.0.1:6379> sadd k2 v3 v4 v5
(integer) 3
127.0.0.1:6379> smove k1 k2 v3
(integer) 1
127.0.0.1:6379> smembers k1
1) "v1"
2) "v2"
127.0.0.1:6379> smembers k2
1) "v5"
2) "v4"
3) "v3"
```

sinter<key1><key2>返回两个集合的交集元素。

```bash
127.0.0.1:6379> sadd k1 v1 v2 v3
(integer) 3
127.0.0.1:6379> sadd k2 v3 v4 v5
(integer) 3
127.0.0.1:6379> sinter k1 k2
1) "v3"
```

sunion<key1><key2>返回两个集合的并集元素。

```bash
127.0.0.1:6379> sunion k1 k2
1) "v2"
2) "v3"
3) "v1"
4) "v5"
5) "v4"
```

sdif<key1><key2>返回两个集合的差集元素（key1-key2）

```bash
127.0.0.1:6379> sdiff k1 k2
1) "v1"
2) "v2"
```

### 数据结构
Set 数据结构是 dict 字典，字典是用哈希表实现的。
Java 中 HashSet 的内部实现使用的是 HashMap，只不过所有的 value 都指向同一个对象。Redis 的 set 结构也是一样，它的内部也使用 hash 结构，所有的 value 都指向同一个内部值。

## 2.4.Redis 哈希（Hash）

### 概述
Redis hash 是一个键值对集合。

Redis hash 是一个 string 类型的 **field** 和 **value** 的映射表，hash 特别适合用于存储对象。

类似 Java 里面的 Map<String,Object>。

![image-20231007154143368](C:\Users\syx\AppData\Roaming\Typora\typora-user-images\image-20231007154143368.png)

用户 ID 为查找的 key，存储的 value 用户对象包含姓名，年龄，生日等信息，如果用普通的 key/value 结构来存储，主要有以下 2 种存储方式：

<img src="C:\Users\syx\AppData\Roaming\Typora\typora-user-images\image-20231007154426019.png" alt="image-20231007154426019" style="zoom: 67%;" />

方法一：每次修改用户的某个属性需要，先反序列化改好后再序列化回去。开销较大。

方法二：用户 ID 数据冗余。

<img src="C:\Users\syx\AppData\Roaming\Typora\typora-user-images\image-20231007154506816.png" alt="image-20231007154506816" style="zoom:67%;" />

通过 key (用户 ID) + field (属性标签) 就可以操作对应属性数据了，既不需要重复存储数据，也不会带来序列化和并发修改控制的问题。

### 命令

hset<key><field><value>		给<key>集合中的<field>键赋值<value>

```bash
127.0.0.1:6379> hset user:1001 id 1
(integer) 1
127.0.0.1:6379> hset user:1001 name zhangsan
(integer) 1
```

hget<key><field>					从<key1>集合<field>取出value

```bash
127.0.0.1:6379> hget user:1001 id
"1"
127.0.0.1:6379> hget user:1001 name
"zhangsan"
```

hmset<key><field1><value1><field2><value2>				....批量设置hash的值

```bash
127.0.0.1:6379> hmset user:1002 id 2 name lisi
OK
```

hexists<key><field>			查看哈希表key中，给定域feld是否存在。

```bash
127.0.0.1:6379> hexists user:1002 name
(integer) 1
127.0.0.1:6379> hexists user:1002 age
(integer) 0
```

hkeys<key>				列出该hash集合的所有field

```bash
127.0.0.1:6379> hkeys user:1001
1) "id"
2) "name"
```

hvals<key>				列出该hash集合的所有value

```bash
127.0.0.1:6379> hvals user:1001
1) "1"
2) "zhangsan"
```

hincrby<key><field><increment>			为哈希表key中的域field的值加上增量1 -1

```bash
127.0.0.1:6379> hincrby user:1002 age 2
(integer) 32
```

hsetnx<key><field><value>			将哈希表key中的域field的值设置为value，当且仅当域field不存在。

```bash
127.0.0.1:6379> hsetnx user:1002 age 40
(integer) 0
127.0.0.1:6379> hsetnx user:1002 gender 1
(integer) 1
```

### 数据结构
Hash 类型对应的数据结构是两种：ziplist（压缩列表），hashtable（哈希表）。当 field-value 长度较短且个数较少时，使用 ziplist，否则使用 hashtable。




## 2.5.Redis 有序集合 Zest（Sorted Set）

### 概述
Redis 有序集合 zset 与普通集合 set 非常相似，是一个**没有重复元素**的字符串集合。

不同之处是有序集合的每个成员都关联了一个**评分（score）**，这个评分（score）被用来按照从最低分到最高分的方式排序集合中的成员。**集合的成员是唯一的，但是评分可以是重复了** 。

因为元素是有序的，所以你也可以很快的根据评分（score）或者次序（position）来获取一个范围的元素。

访问有序集合的中间元素也是非常快的，因此你能够使用有序集合作为一个没有重复成员的智能列表。

### 命令

zadd <key><score1><value1><score2><value2>...
将一个或多个member元素及其score值加入到有序集key当中。

```bash
127.0.0.1:6379> zadd topn 200 java 300 c++ 400 mysql 500 php
(integer) 4
```

zrange <key><start><stop> [WITHSCORES]
返回有序集key中，下标在<start><stop>之间的元素
带WITHSCORES，可以让分数一起和值返回到结果集。

```bash
127.0.0.1:6379> zrange topn 0 -1
1) "java"
2) "c++"
3) "mysql"
4) "php"
127.0.0.1:6379> zrange topn 0 -1 withscores
1) "java"
2) "200"
3) "c++"
4) "300"
5) "mysql"
6) "400"
7) "php"
8) "500"
```

zrangebyscore key minmax [withscores] [limit offset count]
返回有序集key中，所有score值介于min和max之间(包括等于min或max)的成员。
有序集成员按score值递增（从小到大）次序排列。

```bash
127.0.0.1:6379> zrangebyscore topn 300 400
1) "c++"
2) "mysql"
127.0.0.1:6379> zrangebyscore topn 300 400 withscores
1) "c++"
2) "300"
3) "mysql"
4) "400"
```

zrevrangebyscore key maxmin [withscores] [limit offset count]
同上，改为从大到小排列。

```bash
127.0.0.1:6379> zrevrangebyscore topn 500 200
1) "php"
2) "mysql"
3) "c++"
4) "java"
127.0.0.1:6379> zrevrangebyscore topn 500 200 withscores
1) "php"
2) "500"
3) "mysql"
4) "400"
5) "c++"
6) "300"
7) "java"
8) "200"
```

zincrby <key><increment><value>为元素的score加上增量

```bash
127.0.0.1:6379> zincrby topn 50 java
"250"
```

zrem<key><value>删除该集合下，指定值的元素

```bash
127.0.0.1:6379> zrem topn java
(integer) 1
```

zcount<key><min><max>统计该集合，分数区间内的元素个数

```bash
127.0.0.1:6379> zcount topn 200 300
(integer) 2
```

zrank<key><value>返回该值在集合中的排名，从0开始。

```bash
127.0.0.1:6379> zrank topn mysql
(integer) 2
```

案例：如何利用zst实现一个文章访问量的排行榜？





### 数据结构

SortedSet (zset) 是 Redis 提供的一个非常特别的数据结构，一方面它等价于 Java 的数据结构 Map<String, Double>，可以给每一个元素 value 赋予一个权重 score，另一方面它又类似于 TreeSet，内部的元素会按照权重 score 进行排序，可以得到每个元素的名次，还可以通过 score 的范围来获取元素的列表。

zset 底层使用了两个数据结构：

- hash，hash 的作用就是关联元素 value 和权重 score，保障元素 value 的唯一性，可以通过元素 value 找到相应的 score 值。

<img src="C:\Users\syx\AppData\Roaming\Typora\typora-user-images\image-20231007163925545.png" alt="image-20231007163925545" style="zoom:67%;" />

- 跳跃表，跳跃表的目的在于给元素 value 排序，根据 score 的范围获取元素列表。

<img src="C:\Users\syx\AppData\Roaming\Typora\typora-user-images\image-20231007164007922.png" alt="image-20231007164007922" style="zoom:67%;" />



### 跳跃表
**简介**

有序集合在生活中比较常见，例如根据成绩对学生排名，根据得分对玩家排名等。对于有序集合的底层实现，可以用数组、平衡树、链表等。数组不便元素的插入、删除；平衡树或红黑树虽然效率高但结构复杂；链表查询需要遍历所有效率低。Redis 采用的是跳跃表，跳跃表效率堪比红黑树，实现远比红黑树简单。

**实例**

对比有序链表和跳跃表，从链表中查询出 51：

1 有序链表

![image-20231007164305722](C:\Users\syx\AppData\Roaming\Typora\typora-user-images\image-20231007164305722.png)

要查找值为 51 的元素，需要从第一个元素开始依次查找、比较才能找到。共需要 6 次比较。

2 跳跃表

![image-20231007164344142](C:\Users\syx\AppData\Roaming\Typora\typora-user-images\image-20231007164344142.png)

- 从第 2 层开始，1 节点比 51 节点小，向后比较；

- 21 节点比 51 节点小，继续向后比较，后面就是 NULL 了，所以从 21 节点向下到第 1 层；

- 在第 1 层，41 节点比 51 节点小，继续向后，61 节点比 51 节点大，所以从 41 向下；

- 在第 0 层，51 节点为要查找的节点，节点被找到，共查找 4 次。

从此可以看出跳跃表比有序链表效率要高。











# 3.Redis 的发布和订阅

## 3.1什么是发布和订阅

- Redis发布订阅(pub/sub)是一种消息通信模式：发送者(pub)发送消息，订阅者(sub)接收消息。
- Redis客户端可以订阅任意数量的频道。

## 3.2Redis 的发布和订阅

客户端可以订阅频道如下图：

![image-20231007213952214](C:\Users\syx\AppData\Roaming\Typora\typora-user-images\image-20231007213952214.png)

当给这个频道发布消息后，消息就会发送给订阅的客户端：

![image-20231007214022568](C:\Users\syx\AppData\Roaming\Typora\typora-user-images\image-20231007214022568.png)

## 3.3发布订阅命令行实现

打开一个客户端订阅 channel1：

```bash
127.0.0.1:6379> SUBSCRIBE channel1
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "channel1"
3) (integer) 1
```

打开另一个客户端，给 channel1 发布消息 hello：

```bash
127.0.0.1:6379> publish channel1 hello!
(integer) 1
```

打开第一个客户端可以看到发送的消息：

```bash
127.0.0.1:6379> SUBSCRIBE channel1
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "channel1"
3) (integer) 1
1) "message"
2) "channel1"
3) "hello!"
```







