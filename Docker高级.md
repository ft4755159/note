根据尚硅谷Docker教程补全 https://www.bilibili.com/video/BV1gr4y1U7CY

# 1.Docker 主从复制（Mysql 1主1从）

见Docker2022笔记 1.Docker复杂安装详说

# 2.Docker Swarm 集群部署（尚硅谷Redis 3主3从）

## host模式

1、关闭防火墙+启动docker后台服务

`systemctl start docker`

2、新建6个docker容器redis实例

```bash
docker run -d --name redis-node-1 --net host --privileged=true -v /data/redis/share/redis-node-1:/data redis:6.2.1 --cluster-enabled yes --appendonly yes --port 6381
 
docker run -d --name redis-node-2 --net host --privileged=true -v /data/redis/share/redis-node-2:/data redis:6.2.1 --cluster-enabled yes --appendonly yes --port 6382
 
docker run -d --name redis-node-3 --net host --privileged=true -v /data/redis/share/redis-node-3:/data redis:6.2.1 --cluster-enabled yes --appendonly yes --port 6383
 
docker run -d --name redis-node-4 --net host --privileged=true -v /data/redis/share/redis-node-4:/data redis:6.2.1 --cluster-enabled yes --appendonly yes --port 6384
 
docker run -d --name redis-node-5 --net host --privileged=true -v /data/redis/share/redis-node-5:/data redis:6.2.1 --cluster-enabled yes --appendonly yes --port 6385
 
docker run -d --name redis-node-6 --net host --privileged=true -v /data/redis/share/redis-node-6:/data redis:6.2.1 --cluster-enabled yes --appendonly yes --port 6386
```

3、进入容器redis-node-1并为6台机器构建集群关系

```bash
# 进入容器
docker exec -it redis-node-1 /bin/bash

# 构建主从关系
# 注意，进入docker容器后才能执行一下命令，且注意自己的真实IP地址
# --net host 所以对应网卡 eth0:下的ip 192.168.0.86
redis-cli --cluster create 192.168.0.86:6381 192.168.0.86:6382 192.168.0.86:6383 192.168.0.86:6384 192.168.0.86:6385 192.168.0.86:6386 --cluster-replicas 1
# --cluster-replicas 1 表示为每个master创建一个slave节点
一切OK的话，3主3从搞定
```

4、链接进入6381作为切入点，查看集群状态

```bash
cluster nodes
cluster info
```

## bridge模式

bridge模式 ：

https://www.jianshu.com/p/2dec44f9bc21

https://www.cnblogs.com/cyit/p/16888373.html

**前期规划：**

- redis版本 redis:6.2.1

- 网络配置，我们会创建名称为 redisnet 的网络，子网掩码为 172.18.0.0/16的网络

我们会启动六个redis。对应的容器名分别为：

redis-1，redis-2，redis-3，redis-4，redis-5，redis-6；对应的IP为：172.18.0.11，172.18.0.12，172.18.0.13，172.18.0.14，172.18.0.15，172.18.0.16；映射到宿主机的端口分别为：6381，6382，6383，6384，6385，6386；配置文件和数据文件分别挂载在宿主机：/data/redis/node-1，/data/redis/node-2，/data/redis/node-3，/data/redis/node-4，/data/redis/node-5，/data/redis/node-6

1、创建网络

```bash
# 创建一个名为redis，子网掩码为172.38.0.0/16的网络
[root@hecs-233798 ~]# docker network create redisnet --subnet 172.18.0.0/16
# 查看网络是否创建成功，看到NAME为redis的则表示创建成功
[root@hecs-233798 ~]# docker network ls
NETWORK ID     NAME       DRIVER    SCOPE
98f6d4edab36   bridge     bridge    local
ca7c328f85d7   host       host      local
e3f4172bffb3   none       null      local
e479845fe163   redisnet   bridge    local

# docker network inspect redis可以查看redis的详细信息
[root@hecs-233798 ~]# docker network inspect redisnet
[
    {
        "Name": "redisnet",
        "Id": "e479845fe163eaaf45154b60e605740989e7eb0717d4ccdf42be75878b16a772",
        "Created": "2023-10-16T15:17:41.17871473+08:00",
        "Scope": "local",
        "Driver": "bridge",
...
            "Config": [
                {
                    "Subnet": "172.18.0.0/16"
                }
            ]
...

```

2、创建6个redis节点的配置文件

我们会在宿主机上的 /data/conf 文件夹下，分别创建 redis1.conf  redis2.conf  redis3.conf  redis4.conf  redis5.conf  redis6.conf 文件的内容。

注意cluster-announce-ip对应的ip（原本是对应宿主机ip，本例为bridge模式，所以对应容器ip）。

```bash
# 创建6个redis对应的配置，下面的命令可以直接复制到xshell里面执行。会生成对应的文件和写入相应的配置
for index in $(seq 1 6);\
do \
mkdir -p /data/conf
touch /data/conf/redis${index}.conf
cat << EOF >> /data/conf/redis${index}.conf
port 638${index}
bind 0.0.0.0
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
cluster-announce-ip 172.18.0.1${index}
cluster-announce-port 638${index}
cluster-announce-bus-port 1638${index}
appendonly yes
EOF
done
```

3、启动6个实例

```bash
# 查看相应的文件是否成功，配置文件是否正常写入
[root@hecs-233798 redis]# pwd
/data/redis
[root@hecs-233798 redis]# ls
node-1  node-2  node-3  node-4  node-5  node-6
[root@hecs-233798 redis]# cd /data/conf/
[root@hecs-233798 conf]# ls
redis1.conf  redis2.conf  redis3.conf  redis4.conf  redis5.conf  redis6.conf

# 启动6个redis。注意ip的配置。注意docker数据文件和配置文件挂载到宿主机。下面的内容可以直接复制到xshell里面执行
for index in $(seq 1 6);\
do \
docker run -p 638${index}:6379 -p 1638${index}:16379 --name redis-${index} \
-v /data/redis/node-${index}:/data \
-v /data/conf/redis${index}.conf:/etc/redis/redis.conf \
-d --net redisnet --ip 172.18.0.1${index} redis:6.2.1 redis-server /etc/redis/redis.conf
done

# 4. 查看redis是否启动成功，没啥大问题的话，docker ps 我们可以看到我们启动了6个redis
[root@localhost redis]# docker ps 
```

3、进入任意一个容器中

```bash
# 1. 创建集群，这里我们有6个redis,我们随便进入一个redis里面配置，比如这里我们进入redis-1里面配置  docker exec -it redis-1 /bin/bash 进入容器
[root@hecs-233798 conf]# docker exec -it redis-1 /bin/bash
```

4、创建集群

```bash
# 进入容器之后，输入命令 redis-cli --cluster create 172.18.0.11:6381 172.18.0.12:6382 172.18.0.13:6383 172.18.0.14:6384 172.18.0.15:6385 172.18.0.16:6386 --cluster-replicas 1
# 执行过程中，提示输入的时候，输入yes
root@49bb3e6110a3:/data# redis-cli --cluster create 172.18.0.11:6381 172.18.0.12:6382 172.18.0.13:6383 172.18.0.14:6384 172.18.0.15:6385 172.18.0.16:6386 --cluster-replicas 1
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 172.18.0.15:6385 to 172.18.0.11:6381
Adding replica 172.18.0.16:6386 to 172.18.0.12:6382
Adding replica 172.18.0.14:6384 to 172.18.0.13:6383
M: 988334d0c6471a2f187ba096e9a34f28640e6015 172.18.0.11:6381
   slots:[0-5460] (5461 slots) master
M: c4e2b4266517d640394714cb4b296f2a073fce26 172.18.0.12:6382
   slots:[5461-10922] (5462 slots) master
M: a17fdf9ea0c1f8ac73dd69a53c6134af992ea8a9 172.18.0.13:6383
   slots:[10923-16383] (5461 slots) master
S: 8a4103c86a57bddf18e2a4b85cebf86a37f5ee44 172.18.0.14:6384
   replicates a17fdf9ea0c1f8ac73dd69a53c6134af992ea8a9
S: b6a3cdb36005c9c0c0413860ae501dc71ad6bc69 172.18.0.15:6385
   replicates 988334d0c6471a2f187ba096e9a34f28640e6015
S: 65a5218c2ed49152b055ee061241ef52003b52a9 172.18.0.16:6386
   replicates c4e2b4266517d640394714cb4b296f2a073fce26
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
...
>>> Performing Cluster Check (using node 172.18.0.11:6381)
M: 988334d0c6471a2f187ba096e9a34f28640e6015 172.18.0.11:6381
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 8a4103c86a57bddf18e2a4b85cebf86a37f5ee44 172.18.0.14:6384
   slots: (0 slots) slave
   replicates a17fdf9ea0c1f8ac73dd69a53c6134af992ea8a9
M: a17fdf9ea0c1f8ac73dd69a53c6134af992ea8a9 172.18.0.13:6383
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: b6a3cdb36005c9c0c0413860ae501dc71ad6bc69 172.18.0.15:6385
   slots: (0 slots) slave
   replicates 988334d0c6471a2f187ba096e9a34f28640e6015
M: c4e2b4266517d640394714cb4b296f2a073fce26 172.18.0.12:6382
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 65a5218c2ed49152b055ee061241ef52003b52a9 172.18.0.16:6386
   slots: (0 slots) slave
   replicates c4e2b4266517d640394714cb4b296f2a073fce26
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.


# 2. 查看集群是否创建成功，进入容器，连接redis,cluster nodes查看集群信息
[root@hecs-233798 conf]# docker exec -it redis-1 /bin/bash
root@49bb3e6110a3:/data# redis-cli -c -h 172.18.0.11 -p 6381
172.18.0.11:6381> cluster nodes
8a4103c86a57bddf18e2a4b85cebf86a37f5ee44 172.18.0.14:6384@16384 slave a17fdf9ea0c1f8ac73dd69a53c6134af992ea8a9 0 1697445217133 3 connected
a17fdf9ea0c1f8ac73dd69a53c6134af992ea8a9 172.18.0.13:6383@16383 master - 0 1697445218536 3 connected 10923-16383
988334d0c6471a2f187ba096e9a34f28640e6015 172.18.0.11:6381@16381 myself,master - 0 1697445218000 1 connected 0-5460
b6a3cdb36005c9c0c0413860ae501dc71ad6bc69 172.18.0.15:6385@16385 slave 988334d0c6471a2f187ba096e9a34f28640e6015 0 1697445219137 1 connected
c4e2b4266517d640394714cb4b296f2a073fce26 172.18.0.12:6382@16382 master - 0 1697445218000 2 connected 5461-10922
65a5218c2ed49152b055ee061241ef52003b52a9 172.18.0.16:6386@16386 slave c4e2b4266517d640394714cb4b296f2a073fce26 0 1697445218135 2 connected

```



## 2.1.主从扩容

说明：6387为主机 6388为从机

1、新建6387、6388两个节点+新建后启动+查看是否8节点

```bash
docker run -d --name redis-node-7 --net host --privileged=true -v /data/redis/share/redis-node-7:/data redis:6.2.1 --cluster-enabled yes --appendonly yes --port 6387
docker run -d --name redis-node-8 --net host --privileged=true -v /data/redis/share/redis-node-8:/data redis:6.2.1 --cluster-enabled yes --appendonly yes --port 6388
docker ps
```



2、进入6387容器实例内部

`docker exec -it redis-node-7 /bin/bash`

3、将新增的6387节点(空槽号)作为master节点加入原集群

```bash
# 将新增的6387作为master节点加入集群
# redis-cli --cluster add-node 自己实际IP地址:6387 自己实际IP地址:6381
redis-cli --cluster add-node 192.168.0.86:6387 192.168.0.86:6381

6387 就是将要作为master新增节点
6381 就是原来集群节点里面的领路人，相当于6387拜拜6381的码头从而找到组织加入集群
```



4、检查集群情况第1次

```bash
# redis-cli --cluster check 真实ip地址:6381
root@hecs-233798:/data# redis-cli --cluster check 192.168.0.86:6387
```



5、重新分派槽号

```bash
# 重新分派槽号
# 命令:redis-cli --cluster reshard IP地址:端口号
root@hecs-233798:/data# redis-cli --cluster reshard 192.168.0.86:6381
>>> Performing Cluster Check (using node 192.168.0.86:6381)
M: 040ed250e7f4f6ae281bf94039d3d344a02e1496 192.168.0.86:6381
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: fc6440425aebd86655037ae9582eecba7afcba07 192.168.0.86:6386
   slots: (0 slots) slave
   replicates 361aa9fd7b71c7090b9a4507d76953b66f6cf4d4
M: 361aa9fd7b71c7090b9a4507d76953b66f6cf4d4 192.168.0.86:6382
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: f9f35df09fb3eddcb76dcfa296b91e7c1225ed2e 192.168.0.86:6384
   slots: (0 slots) slave
   replicates 27fb2426c1bfc94a1ea5959cb33331602b8bdafb
S: 94bec02092a848ca2bfaa360e345bf603f6dc00b 192.168.0.86:6385
   slots: (0 slots) slave
   replicates 040ed250e7f4f6ae281bf94039d3d344a02e1496
M: 27fb2426c1bfc94a1ea5959cb33331602b8bdafb 192.168.0.86:6383
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
M: 2a5ebdf6a3768fa0e01b7e1a8675172ae1861ce5 192.168.0.86:6387
   slots: (0 slots) master
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
How many slots do you want to move (from 1 to 16384)? 4096 # 16384/master个数
What is the receiving node ID? 2a5ebdf6a3768fa0e01b7e1a8675172ae1861ce5 # 6387的id
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: all # 全部分配 all

...
    Moving slot 12286 from 27fb2426c1bfc94a1ea5959cb33331602b8bdafb
    Moving slot 12287 from 27fb2426c1bfc94a1ea5959cb33331602b8bdafb
Do you want to proceed with the proposed reshard plan (yes/no)? yes
...
```



6、检查集群情况第2次

```bash
# redis-cli --cluster check 真实ip地址:6381
root@hecs-233798:/data# redis-cli --cluster check 192.168.0.86:6387
...
>>> Performing Cluster Check (using node 192.168.0.86:6387)
M: 2a5ebdf6a3768fa0e01b7e1a8675172ae1861ce5 192.168.0.86:6387
   slots:[0-1364],[5461-6826],[10923-12287] (4096 slots) master
M: 27fb2426c1bfc94a1ea5959cb33331602b8bdafb 192.168.0.86:6383
   slots:[12288-16383] (4096 slots) master
   1 additional replica(s)
M: 361aa9fd7b71c7090b9a4507d76953b66f6cf4d4 192.168.0.86:6382
   slots:[6827-10922] (4096 slots) master
   1 additional replica(s)
M: 040ed250e7f4f6ae281bf94039d3d344a02e1496 192.168.0.86:6381
   slots:[1365-5460] (4096 slots) master
   1 additional replica(s)
...
[OK] All nodes agree about slots configuration.

# 槽号分派说明
为什么6387是3个新的区间，以前的还是连续？
重新分配成本太高，所以前3家各自匀出来一部分，从6381/6382/6383三个旧节点分别匀出1364个坑位给新节点6387
```



7、为主节点6387分配从节点6388

```bash
# 命令：redis-cli --cluster add-node ip:新slave端口 ip:新master端口 --cluster-slave --cluster-master-id 新主机节点ID
 
root@hecs-233798:/data# redis-cli --cluster add-node 192.168.0.86:6388 192.168.0.86:6387 --cluster-slave --cluster-master-id 2a5ebdf6a3768fa0e01b7e1a8675172ae1861ce5
2a5ebdf6a3768fa0e01b7e1a8675172ae1861ce5----这个是6387的编号，按照自己实际情况

```



8、检查集群情况第3次

```bash
# redis-cli --cluster check 真实ip地址:6381
root@hecs-233798:/data# redis-cli --cluster check 192.168.0.86:6387
```



## 2.2.主从缩容

1、目的：6387和6388下线

2、检查集群情况1获得6388的节点ID

```bash
root@hecs-233798:/data# redis-cli --cluster check 192.168.0.86:6388
```



3、将6388删除
从集群中将4号从节点6388删除

```bash
命令：redis-cli --cluster del-node ip:从机端口 从机6388节点ID
 
root@hecs-233798:/data# redis-cli --cluster del-node 192.168.0.86:6388 b6770698bb3d5c0e13f9262e500b533033cd46a4

root@hecs-233798:/data# redis-cli --cluster check 192.168.0.86:6387
 检查一下发现，6388被删除了，只剩下7台机器了。
```



4、将6387的槽号清空，重新分配，本例将还原分配成初始状态--需要操作3次

```bash
# 第一次
root@hecs-233798:/data# redis-cli --cluster reshard 192.168.0.86:6381
>>> Performing Cluster Check (using node 192.168.0.86:6381)
M: 040ed250e7f4f6ae281bf94039d3d344a02e1496 192.168.0.86:6381
   slots:[1365-5460] (4096 slots) master
   1 additional replica(s)
M: 2a5ebdf6a3768fa0e01b7e1a8675172ae1861ce5 192.168.0.86:6387
   slots:[0-1364],[5461-6826],[10923-12287] (4096 slots) master
...
How many slots do you want to move (from 1 to 16384)? 1365				#本次分配多少个节点？
What is the receiving node ID? 040ed250e7f4f6ae281bf94039d3d344a02e1496 # 谁来接收？6381节点ID
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: 2a5ebdf6a3768fa0e01b7e1a8675172ae1861ce5 			# 谁被释放？6387节点ID
Source node #2: done
...
Moving slot 1364 from 192.168.0.86:6387 to 192.168.0.86:6381: 

# 第二次
root@hecs-233798:/data# redis-cli --cluster reshard 192.168.0.86:6381
...
M: 361aa9fd7b71c7090b9a4507d76953b66f6cf4d4 192.168.0.86:6382
   slots:[6827-10922] (4096 slots) master
   1 additional replica(s)
M: 2a5ebdf6a3768fa0e01b7e1a8675172ae1861ce5 192.168.0.86:6387
   slots:[0-1364],[5461-6826],[10923-12287] (4096 slots) master
...
How many slots do you want to move (from 1 to 16384)? 1366				#本次分配多少个节点？
What is the receiving node ID? 361aa9fd7b71c7090b9a4507d76953b66f6cf4d4 # 谁来接收？6382节点ID
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: 2a5ebdf6a3768fa0e01b7e1a8675172ae1861ce5 			# 谁被释放？6387节点ID
Source node #2: done
...
Moving slot 6826 from 192.168.0.86:6387 to 192.168.0.86:6382:

# 第三次
root@hecs-233798:/data# redis-cli --cluster reshard 192.168.0.86:6381
>>> Performing Cluster Check (using node 192.168.0.86:6381)
M: 27fb2426c1bfc94a1ea5959cb33331602b8bdafb 192.168.0.86:6383
   slots:[12288-16383] (4096 slots) master
   1 additional replica(s)
M: 2a5ebdf6a3768fa0e01b7e1a8675172ae1861ce5 192.168.0.86:6387
   slots:[10923-12287] (1365 slots) master
...
[OK] All 16384 slots covered.
How many slots do you want to move (from 1 to 16384)? 1365				#本次分配多少个节点？
What is the receiving node ID? 27fb2426c1bfc94a1ea5959cb33331602b8bdafb # 谁来接收？6382节点ID
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: 2a5ebdf6a3768fa0e01b7e1a8675172ae1861ce5
Source node #2: done
...
Moving slot 12287 from 192.168.0.86:6387 to 192.168.0.86:6383: 			# 谁被释放？6387节点ID

```



5、检查集群情况第二次

```bash
root@hecs-233798:/data# redis-cli --cluster check 192.168.0.86:6381
192.168.0.86:6381 (040ed250...) -> 1 keys | 5461 slots | 1 slaves.
192.168.0.86:6382 (361aa9fd...) -> 0 keys | 5462 slots | 1 slaves.
192.168.0.86:6383 (27fb2426...) -> 1 keys | 5461 slots | 1 slaves.
192.168.0.86:6387 (2a5ebdf6...) -> 0 keys | 0 slots | 0 slaves.

4096个槽位已被全部分配给 6381 6382 6383
```



6、将6387删除

```bash
命令：redis-cli --cluster del-node ip:端口 6387节点ID
 
root@hecs-233798:/data# redis-cli --cluster del-node 192.168.0.86:6387 2a5ebdf6a3768fa0e01b7e1a8675172ae1861ce5
>>> Removing node 2a5ebdf6a3768fa0e01b7e1a8675172ae1861ce5 from cluster 192.168.0.86:6387
```



7、检查集群情况第三次

```bash
root@hecs-233798:/data# redis-cli --cluster check 192.168.0.86:6381
192.168.0.86:6381 (040ed250...) -> 1 keys | 5461 slots | 1 slaves.
192.168.0.86:6382 (361aa9fd...) -> 0 keys | 5462 slots | 1 slaves.
192.168.0.86:6383 (27fb2426...) -> 1 keys | 5461 slots | 1 slaves.
[OK] 2 keys in 3 masters.
```

# 3.Docker Swarm 集群部署（动力节点）

# 78.Docker Compose 容器编排

## 78.1.Docker Compose 简介

**能干嘛**

docker建议我们每一个容器中只运行一个服务,因为docker容器本身占用资源极少,所以最好是将每个服务单独的分割开来但是这样我们又面临了一个问题？

如果我需要同时部署好多个服务,难道要每个服务单独写Dockerfile然后在构建镜像,构建容器,这样累都累死了,所以docker官方给我们提供了docker-compose多服务部署的工具

例如要实现一个Web微服务项目，除了Web服务容器本身，往往还需要再加上后端的数据库mysql服务容器，redis服务器，注册中心eureka，甚至还包括负载均衡容器等等。。。。。。

Compose允许用户通过一个单独的 <font color=cyan >docker-compose.yml</font> 模板文件（YAML 格式）来定义一组<font color= red>相关联的应用容器为一个项目（project）</font>。

可以很容易地用一个配置文件定义一个多容器的应用，然后使用一条指令安装这个应用的所有依赖，完成构建。Docker-Compose 解决了容器与容器之间如何管理编排的问题。

**安装方式**

官网 https://docs.docker.com/compose/compose-file/compose-file-v3/

下载地址 https://docs.docker.com/compose/install/

**安装步骤**

```bash
# curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
# chmod +x /usr/local/bin/docker-compose
# docker-compose --versioncurl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
# chmod +x /usr/local/bin/docker-compose
# docker-compose --version

新版本Docker已经内置了Compose
[root@hecs-233798 ~]# docker compose version
Docker Compose version v2.21.0
```

**卸载**

如果使用curl以下方式安装，则卸载Docker Compose：

`rm /usr/local/bin/docker-compose`

## 78.2.Compose 核心概念

**一文件**

 <font color=cyan >docker-compose.yml</font>

**两要素**

- 服务（service）：

  一个个应用容器实例，比如订单微服务、库存微服务、mysql容器、nginx容器或者redis容器

- 工程（project）

  由一组关联的应用容器组成的一个 <font color=red>完整业务单元</font>，在 docker-compose.yml 文件中定义。

## 78.3.Compose使用的三个步骤

1、编写Dockerfile定义各个微服务应用并构建出对应的镜像文件

2、使用 docker-compose.yml 定义一个完整业务单元，安排好整体应用中的各个容器服务

3、最后，执行docker-compose up命令 来启动并运行整个应用程序，完成一键部署上线

## 78.4.Compose常用命令
docker-compose -h                           # 查看帮助
docker-compose up                           # 启动所有docker-compose服务
<font color=red>docker-compose up -d                        # 启动所有docker-compose服务并后台运行</font>
<font color=red>docker-compose down                         # 停止并删除容器、网络、卷、镜像。</font>
docker-compose exec  yml里面的服务id                 # 进入容器实例内部  

docker-compose exec <font color=red>docker-compose.yml文件中写的服务id</font> /bin/bash
docker-compose ps                      # 展示当前docker-compose编排过的运行的所有容器
docker-compose top                     # 展示当前docker-compose编排过的容器进程

docker-compose logs  yml里面的服务id     # 查看容器输出日志
<font color=red>docker-compose config     # 检查配置</font>
<font color=red>docker-compose config -q  # 检查配置，有问题才有输出</font>
docker-compose restart   # 重启服务
docker-compose start     # 启动服务
docker-compose stop      # 停止服务

## 78.5.不使用Compose编排微服务

前提条件：准备jar包，打包成镜像

```bash
cd /root/finance
mv ssrm-0.0.1-SNAPSHOT.jar ssrm.jar
# 创建 Dockerfile 文件
vim Dockerfile

FROM openjdk:8u102
MAINTAINER zs@163.com
LABEL version="1.0" description="this is a spring boot application"
COPY ssrm.jar ssrm.jar
ENTRYPOINT ["java", "-jar", "ssrm.jar"]
EXPOSE 8080
# 此处 EXPOSE 8080 实际不起作用，作用是提示内部端口号。因为下一步启动时要加 -p 9000 参数
# 在当前目录开始构建
docker build -t finance:1.0 .

```

1、启动mysql容器，并检查配置，准备数据

启动前添加网络 `docker network create bjnet --subnet 172.19.0.0/16`

```bash
# 启动容器
docker run -d -p 3306:3306 --net bjnet \
-v /root/mysql/log:/var/log/mysql \
-v /root/mysql/data:/var/lib/mysql \
-v /root/mysql/conf:/etc/mysql/conf.d \
-e MYSQL_ROOT_PASSWORD=111 \
--name bjpnmysql mysql:5.7

# 查看字符集发现有 latin1
SHOW VARIABLES LIKE 'character%';
# 进入容器内修改 /etc/my.cnf
docker exec -it bjpnmysql bash
# 建议直接cat my.cnf 复制出来，插入两条
[mysql]
default-character-set=utf8
[mysqld]
character-set-server=utf8

# 将修改好的内容覆盖 my.cnf
rm my.cnf
 cat << EOF >> /etc/my.cnf
 [内容]
 EOF

# 建表语句
CREATE DATABASE IF NOT EXISTS `test`;
USE `test`;
DROP TABLE IF EXISTS `product`;
CREATE TABLE `product` (
`id` int(11) NOT NULL AUTO_INCREMENT,
`name` varchar(20) DEFAULT NULL,
`rate` double DEFAULT NULL,
`amount` double DEFAULT NULL,
`raised` double DEFAULT NULL,
`cycle` int(11) DEFAULT NULL,
`endTime` char(10) DEFAULT '0',
PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=9 DEFAULT CHARSET=utf8;

INSERT INTO `product` VALUES
(1,'TXTY',2.76,50000,20000,30,'2022-07-10'),
(2,'GTTY',2.86,30000,30000,60,'2022-07-12'),
(3,'GTGX',2.55,60000,50000,90,'2022-07-09'),
(4,'GFMA',2.96,30000,20000,7,'2022-05-10'),
(5,'TYXD',2.65,80000,60000,20,'2022-07-05'),
(6,'HNSY',3.05,30000,20000,10,'2022-06-10'),
(7,'HNSX',2.76,50000,30000,30,'2022-07-02'),
(8,'LXSY',2.86,30000,20000,20,'2022-07-11');
```

2、启动redis

```bash
docker run -d -p 6379:6379 --net bjnet \
-v /root/redis/redis.conf:/etc/redis/redis.conf \
-v /root/redis/data:/data \
--name bjpnredis redis:7.0 \
redis-server /etc/redis/redis.conf
```

3、启动应用容器  

```bash
docker run --name myapp --net bjnet \
-v /root/finance/log:/var/applogs \
-dp 9000:8080 \
finance:1.0
```

4、测试

- 浏览器访问 http://114.115.167.245:9000/ 可以查询所有数据

- 安装Chrome插件Restlet Client ，以POST方式发送给http://114.115.167.245:9000/register以下json,可以插入一条数据，同时redis会清空避免脏数据

```bash
{
"name":"TXSF",
"rate":4.0,
"amount":1000,
"raised":200,
"cycle":15,
"endTime":"2023-3-15"
}
# 进入mysql查询结果
mysql -uroot -p111
mysql> use test;
mysql> select * from product;
+----+------+------+--------+--------+-------+------------+
| id | name | rate | amount | raised | cycle | endTime    |
+----+------+------+--------+--------+-------+------------+
|  1 | TXTY | 2.76 |  50000 |  20000 |    30 | 2022-07-10 |
|  2 | GTTY | 2.86 |  30000 |  30000 |    60 | 2022-07-12 |
|  3 | GTGX | 2.55 |  60000 |  50000 |    90 | 2022-07-09 |
|  4 | GFMA | 2.96 |  30000 |  20000 |     7 | 2022-05-10 |
|  5 | TYXD | 2.65 |  80000 |  60000 |    20 | 2022-07-05 |
|  6 | HNSY | 3.05 |  30000 |  20000 |    10 | 2022-06-10 |
|  7 | HNSX | 2.76 |  50000 |  30000 |    30 | 2022-07-02 |
|  8 | LXSY | 2.86 |  30000 |  20000 |    20 | 2022-07-11 |
|  9 | TXSF |    4 |   1000 |    200 |    15 | 2023-3-15  |
+----+------+------+--------+--------+-------+------------+
9 rows in set (0.00 sec)

```

## 78.6.使用Compose编排微服务

1、定义 compose.yml  

在 finance 目录中新建一个文件 compose.yml  

```yml
# ./代表当前目录 /root/finance
version: "3"

services:
  bjpnapp:
    build: ./
    container_name: myapp
    ports:
      - 9000:8080
    volumes:
      - ./logs:/var/applogs
    networks:
      - bjnet
    depends_on:
      - bjpnmysql
      - bjpnredis

  bjpnmysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: 111
    ports:
      - 3306:3306
    volumes:
      - /root/mysql/log:/var/log/mysql
      - /root/mysql/data:/var/lib/mysql
      - /root/mysql/conf:/etc/mysql/conf.d
    networks:
      - bjnet

  bjpnredis:
    image: redis:7.0
    ports:
      - 6379:6379
    volumes:
      - /root/redis/redis.conf:/etc/redis/redis.conf
      - /root/redis/data:/data
    networks:
      - bjnet
    command: redis-server /etc/redis/redis.conf

networks:
  bjnet:
```

2、

（1）修改应用的 application.yml 的几处配置，注意账号密码要与 compose.yml 一致

jdbc:mysql://bjpnmysql:3306/

host: bjpnredis

```yml
spring:
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://bjpnmysql:3306/test?useUnicode=true&characterEncoding=utf-8
    username: root
    password: 111

  redis:
    host: bjpnredis
    port: 6379
  cache:
    type: redis
    cache-names: pc
```

（2）确认无误重新打包项目，上传jar包

由于应用程序的配置文件发生了变化，所以需要对应用程序重新进行 package 打包，并
将新的 jar 包上传到 Linux 系统中的/root/finance 目录中。  

3、启动所有容器

```bash
[root@hecs-233798 finance]# docker compose up -d
[+] Building 2.3s (7/7) FINISHED                                                                                                                                                            docker:default
 => [bjpnapp internal] load build definition from Dockerfile                                     0.0s
 => => transferring dockerfile: 223B                                                             0.0s
 => [bjpnapp internal] load .dockerignore                                                        0.0s
 => => transferring context: 2B                                                                  0.0s
 => [bjpnapp internal] load metadata for docker.io/library/openjdk:8u102                         2.2s
 => [bjpnapp internal] load build context                                                        0.0s
 => => transferring context: 32B                                                                 0.0s
 => [bjpnapp 1/2] FROM docker.io/library/openjdk:8u102@sha256:0c2d5b5369724d7d7935ca2f8d0e2813608822cb297c587fe2942776a7ef7461                                                                                               0.0s
 => CACHED [bjpnapp 2/2] COPY ssrm.jar ssrm.jar                                                  0.0s
 => [bjpnapp] exporting to image                                                                 0.0s
 => => exporting layers                                                                          0.0s
 => => writing image sha256:22f1b98f295d8fcbffda2a3dacb1990ae5fbd1640a215f0cb3c2bf027b78a037     0.0s
 => => naming to docker.io/library/finance-bjpnapp                                               0.0s
[+] Running 4/4
 ✔ Network finance_bjnet          Created                                                        0.1s 
 ✔ Container finance-bjpnredis-1  Started                                                        0.1s 
 ✔ Container finance-bjpnmysql-1  Started                                                        0.1s 
 ✔ Container myapp                Started                                                        0.0s 

```



## 78.7.上面的例子无法使用ifconfig ping，修改Dockerfile文件重新做Compose

（仅供测试研究网络连接原理，真实生产环境一般还用上面78.6的例子，比较简单）

前置条件： 

/root/finance2 路径下准备几个文件compose.yml  Dockerfile  jdk-8u102-linux-x64.tar.gz  ssrm.jar

```bash
[root@hecs-233798 finance2]# ls
compose.yml  Dockerfile  jdk-8u102-linux-x64.tar.gz  ssrm.jar
[root@hecs-233798 finance2]# pwd
/root/finance2
```

1、Dockerfile 构建镜像测试

```bash
# 创建Dockerfile文件
# vim Dockerfile
FROM centos:7
MAINTAINER zhanmiao<357833687@qq.com>
LABEL version="1.1" description="this is a spring boot application with jdk and yum"

ADD jdk-8u102-linux-x64.tar.gz /usr/local/
RUN yum -y install vim
RUN yum -y install yum
RUN yum -y install iputils
RUN yum -y install net-tools

ENV MYPATH /usr/local
WORKDIR $MYPATH

ENV JAVA_HOME /usr/local/jdk1.8.0_102
ENV CLASS_PATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV JRE_HOME /usr/local/jdk1.8.0_102/jre
ENV PATH $JAVA_HOME/bin:$PATH
COPY ssrm.jar ssrm.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "/usr/local/ssrm.jar"]
-------------------------------------------------------------------
# build测试----成功
[root@hecs-233798 finance2]# docker build -t zhanmiaos:1.1 .
[+] Building 2.0s (13/13) FINISHED    

# run测试----成功
[root@hecs-233798 finance2]# docker run -d zhanmiaos:1.1
8cabc0c040d2155256432437266d002b8985bee848131d3e229c7360f015dde7
[root@hecs-233798 finance2]# docker ps
CONTAINER ID   IMAGE           COMMAND                  CREATED          STATUS          PORTS      NAMES
8cabc0c040d2   zhanmiaos:1.1   "java -jar /usr/loca…"   40 seconds ago   Up 39 seconds   8080/tcp   quizzical_mirzakhani

# 进入容器，测试命令是否正确安装----成功
[root@8cabc0c040d2 local]# ls
bin  etc  games  include  jdk1.8.0_102  lib  lib64  libexec  sbin  share  src  ssrm.jar
[root@8cabc0c040d2 local]# vim
[root@8cabc0c040d2 local]# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.2  netmask 255.255.0.0  broadcast 172.17.255.255

#最后删除镜像和容器，以便测试Compose
```

2、容器编排测试，直接使用78.6中的 compose.yml 文件

```yml
[root@hecs-233798 finance2]# vim compose.yml
version: "3"

services:
  bjpnapp:
    build: ./
    container_name: myapp
    ports:
      - 9000:8080
    volumes:
      - ./logs:/var/applogs
    networks:
      - bjnet
    depends_on:
      - bjpnmysql
      - bjpnredis

  bjpnmysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: 111
    ports:
      - 3306:3306
    volumes:
      - /root/mysql/log:/var/log/mysql
      - /root/mysql/data:/var/lib/mysql
      - /root/mysql/conf:/etc/mysql/conf.d
    networks:
      - bjnet

  bjpnredis:
    image: redis:7.0
    ports:
      - 6379:6379
    volumes:
      - /root/redis/redis.conf:/etc/redis/redis.conf
      - /root/redis/data:/data
    networks:
      - bjnet
    command: redis-server /etc/redis/redis.conf

networks:
  bjnet:
```

Compose启动，并测试网络

```bash
# Compose启动
[root@hecs-233798 finance2]# docker compose up -d
[+] Running 4/4
 ✔ Network finance2_bjnet          Created                                                        0.1s 
 ✔ Container finance2-bjpnredis-1  Started                                                        0.0s 
 ✔ Container finance2-bjpnmysql-1  Started                                                        0.0s 
 ✔ Container myapp                 Started                                                        0.0s
 
# 进入容器
[root@hecs-233798 finance2]# docker exec -it myapp /bin/bash
[root@c7f656499578 local]# 

# 直接ping redis和 mysql的服务名----成功
# 说明compose.yml定义的服务名可以直接被解析
[root@c7f656499578 local]# ping bjpnredis 
PING bjpnredis (172.27.0.3) 56(84) bytes of data.
64 bytes from finance2-bjpnredis-1.finance2_bjnet (172.27.0.3): icmp_seq=1 ttl=64 time=0.066 ms
...
[root@c7f656499578 local]# ping bjpnmysql
PING bjpnmysql (172.27.0.2) 56(84) bytes of data.
64 bytes from finance2-bjpnmysql-1.finance2_bjnet (172.27.0.2): icmp_seq=1 ttl=64 time=0.048 ms
...

# 对照一下ssrm.jar的application.yml
# 可以看出项目打包的时候是通过 指定服务名，来进行容器间的链接
spring:
## mysql://<服务名或ip>:3306/ 
...
    url: jdbc:mysql://bjpnmysql:3306/test?useUnicode=true&characterEncoding=utf-8 
    username: root
    password: 111
## host: <服务名或ip>
  redis:
    host: bjpnredis
    port: 6379
...
```

compose2 测试通过，也可加上container_name、image参数规范容器名，镜像名

```bash
version: "3"

services:
  bjpnapp:
    build: ./
    image: finance:2.0
    container_name: dlgapp
    ports:
      - 9000:8080
    volumes:
      - ./logs:/var/applogs
    networks:
      - bjnet
    depends_on:
      - bjpnmysql
      - bjpnredis

  bjpnmysql:
    image: mysql:5.7
    container_name: dlgmysql
    environment:
      MYSQL_ROOT_PASSWORD: 111
    ports:
      - 3306:3306
    volumes:
      - /root/mysql/log:/var/log/mysql
      - /root/mysql/data:/var/lib/mysql
      - /root/mysql/conf:/etc/mysql/conf.d
    networks:
      - bjnet

  bjpnredis:
    image: redis:7.0
    container_name: dlgredis
    ports:
      - 6379:6379
    volumes:
      - /root/redis/redis.conf:/etc/redis/redis.conf
      - /root/redis/data:/data
    networks:
      - bjnet
    command: redis-server /etc/redis/redis.conf

networks:
  bjnet:
```









# 126.CIG 重量级管理工具

CIG，即 CAdvisor、 InfluxDB 与 Grafana，被称为 Docker 监控三剑客。其中 CAdvisor 用于
监控数据的收集， InfluxDB 用于数据存储， Grafana 用于数据展示。
（1） CAdvisor
cAdvisor 是谷歌公司用来分析运行中的 Docker 容器的资源占用以及性能特性的工具，
包括对容器的内存、 CPU、网络、磁盘 IO 等监控，同时提供了 WEB 页面用于展示监控数据。
cAdvisor 使用一个运行中的守护进程来收集、聚合、处理和导出运行容器相关的信息， 为每
个容器保存独立的参数、历史资源使用情况和完整的资源使用数据。
默认情况下， CAdvisor 可以针对单个主机存储 2 分钟的监控数据。不过其提供了很多的
数据集成接口用于存储监控数据，支持 InfluxDB， Redis， Kafka， Elasticsearch 等，官方推荐
InfluxDB。
（2） InfluxDB
InfluxDB 是一个由 InfluxData 用 GO 语言开发的、 开源的、高性能的、 时序型数据库，
专注于海量时序数据的高性能读、高性能写、高效存储与实时分析等，无需外部依赖。
（3） Grafana
Grafana 是一款采用 GO 语言编写的、开源的、 数据监控分析可视化平台，主要用于大
规模指标数据的可视化展示，是网络架构和应用分析中最流行的时序数据展示工具，目前已
经支持绝大部分常用的时序数据库。拥有丰富的插件及模板功能支持图表权限控制和报警。  

## 126.1.启动配置

1、创建文件 /root/cig/compose.yml 写入配置

```yaml
# cadv 类似库名
# restart: always 每次随docker启动
# links: - "influxdb:influxsrv" 指定influxdb可以通过host://influxsrv:8086访问
# command: -storage_driver=influxdb ... 通过查看dockerhub了解在构建的时候使用了ENTRYPOINT参数
# 所以这里command代表的含义是追加命令
services:
  influxdb:
    image: tutum/influxdb:0.9
    container_name: mydb
    restart: always
    environment:
      - PRE_CREATE_DB=cadv
    ports:
      - 8083:8083
      - 8086:8086
    volumes:
      - /root/cig/data/influxdb:/data

  cadvisor:
    image: google/cadvisor
    container_name: mycollector
    links:
      - "influxdb:influxsrv"
    command: -storage_driver=influxdb -storage_driver_db=cadv -storage_driver_host=influxsrv:8086
    restart: always
    ports:
      - 8080:8080
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro

  grafana:
    user: "104"
    image: grafana/grafana
    container_name: myui
    restart: always
    links:
      - "influxdb:influxsrv"
    ports:
      - 3000:3000
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - HTTP_USER=admin
      - HTTP_PASS=admin
      - INFLUXDB_HOST=influxsrv
      - INFLUXDB_PORT=8086
      - INFLUXDB_NAME=cadv
      - INFLUXDB_USER=root
      - INFLUXDB_PASS=root

volumes:
  grafana_data: {}
```

2、检查 compose.yml，顺便清空所有容器

```bash
docker compose config -q
```

3、后台启动

```bash
docker compose up -d
```

4、查看容器链接信息 

`docker inspect mycollector`

```bash
"Links": [
    "mydb:influxsrv",
    "mydb:influxdb-1",
    "mydb:cig-influxdb-1"
],
"Aliases": [
    "mycollector",
    "cadvisor",
    "0abdffcab425"
],
```

`docker inspect myui`

```bash
"Links": [
    "mydb:influxsrv",
    "mydb:influxdb-1",
    "mydb:cig-influxdb-1"
],
"Aliases": [
    "myui",
    "grafana",
    "e9931d15e5f0"
],
```

`docker inspect mydb`

```bash
"Links": null,
"Aliases": [
    "mydb",
    "influxdb",
    "1fda4fb18f4f"
],
```

可以看到容器的几个别名和连接的别名

另外，网络也自动配置了

`docker network inspect cig_default`

```bash
"Containers": {
    "0abdffcab425754256b5f3650c5a6a3e9dd60d8b29847617a136516b28922bf8": {
        "Name": "mycollector",
        "IPv4Address": "192.168.16.3/20",
    },
    "1fda4fb18f4f475403d504bfaf0a409788bcade9b692d16303f850120dbab396": {
        "Name": "mydb",
        "IPv4Address": "192.168.16.2/20",
    },
    "e9931d15e5f0d255be13cea71cae10960d824258324824f4c0594269b04e83a7": {
        "Name": "myui",
        "IPv4Address": "192.168.16.4/20",
    }
},
```

## 126.2.查看工具界面

访问前记得打开 compose.yml 的所有端口（8080 8083 8086 3000）

1、cAdvisor 收集

http://114.115.167.245:8080/

2、influxdb 存储

http://114.115.167.245:8083/

3、grafana 展示

http://114.115.167.245:3000/

登录账号密码： admin admin（见compose.yml）然后skip

配置数据源

<img src="images\image-20231024190734900.png" alt="image-20231024190734900" style="zoom:80%;" />

账号密码： root root（见compose.yml）

<img src="images\image-20231024191104184.png" alt="image-20231024191104184" style="zoom:80%;" />

save & test

具体使用待有需要时再搞清楚






# CI/CD之jenkins