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

**概述**

- String 是 Redis 最基本的类型，你可以理解成与 Memcached 一模一样的类型，一个 key 对应一个 value。
- String 类型是**二进制安全**的。意味着 Redis 的 string 可以包含任何数据。比如 jpg 图片或者序列化的对象。
- String 类型是 Redis 最基本的数据类型，一个 Redis 中字符串 value 最多可以是 **512M**。

**命令**

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

**数据结构**
String 的数据结构为简单动态字符串 (Simple Dynamic String, 缩写 SDS)，是可以修改的字符串，内部结构实现上类似于 Java 的 ArrayList，采用预分配冗余空间的方式来减少内存的频繁分配.

![image-20230928163223630](C:\Users\syx\AppData\Roaming\Typora\typora-user-images\image-20230928163223630.png)

如图中所示，内部为当前字符串实际分配的空间 capacity 一般要高于实际字符串长度 len。当字符串长度小于 1M 时，扩容都是加倍现有的空间，如果超过 1M，扩容时一次只会多扩 1M 的空间。需要注意的是字符串最大长度为 512M。




## 2.2.Redis 列表（List)

## 2.3.Redis 集合（Set）

## 2.4.Redis 哈希（Hash）

## 2.5.Redis 有序集合 Zest（Sorted Set）



















