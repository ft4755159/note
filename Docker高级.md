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

## 3.1.swarm 理论基础

**3.1.1.简介**

​    Docker Swarm 是由 Docker 公司推出的 Docker 的原生集群管理系统， 它将一个 Docker
主机池变成了一个单独的虚拟主机，用户只需通过简单的 API 即可实现与 Docker 集群的通
信。 Docker Swarm 使用 GO 语言开发。从 Docker 1.12.0 版本开始， Docker Swarm 已经内置于

​    Docker 引擎中，无需再专门的进行安装配置。
​    Docker Swarm 在 Docker 官网的地址为： https://docs.docker.com/engine/swarm/

**3.1.2.节点架构**

（1） 架构图

<img src="images\image-20231025162902927.png" alt="image-20231025162902927" style="zoom:80%;" />

（2） swarm node  

​    从物理上讲， 一个 Swarm 是由若干安装了 Docker Engine 的物理机或者虚拟机组成，这
些主机上的 Docker Engine 都采用 Swarm 模式运行。
​    从逻辑上讲，一个 Swarm 由若干节点 node 构成，每个 node 最终会落实在一个物理
Docker 主机上，但一个物理 Docker 主机并不一定就是一个 node。即 swarm node 与 Docker
主机并不是一对一的关系。
swarm node 共有两种类型： manager 与 worker。

（3） Manager

​    Manager 节点用于维护 swarm 集群状态、调试 servcie、处理 swarm 集群管理任务。为
了防止单点故障问题，一个 Swarm 集群一般都会包含多个 manager。这些 manager 间通过
Raft 算法维护着一致性。

（4） Worker

​    Worker 节点用于在其 Contiainer 中运行 task 任务，即对外提供 service 服务。默认情况
下， manager 节点同时也充当着 worker 角色，可以运行 task 任务。

（5） 角色转换

​    manager 节点与 worker 节点角色并不是一成不变的，它们之间是可以相互转换的。
- manager 转变为 worker 称为节点降级
- worker 转变为 manager 称为节点升级

**3.1.3.服务架构**

（1） 架构图

<img src="images\image-20231025163121338.png" alt="image-20231025163121338" style="zoom:80%;" />

<img src="images\image-20231025163217251.png" alt="image-20231025163217251" style="zoom:80%;" />

（2） service

​    搭建 docker swarm 集群的目的是为了能够在 swarm 集群中运行应用，为用户提供具备
更强抗压能力的服务。 docker swarm 中的服务 service 就是一个逻辑概念，表示 swarm 集群
对外提供的服务。

（3） task

​    一个 service 最终是通过任务 task 的形式出现在 swarm 的各个节点中，而每个节点中的
task 又都是通过具体的运行着应用进程的容器对外提供的服务。

（4） 编排器

​    在 swarm manager 中具有一个编排器，用于管理副本 task 任务的创建与停止。例如，
当在 swarm manager 中定义一个具有 3 个 task 副本任务的 service 时， 编排器首先会创建 3
个 task，为每个 task 分配一个 taskID，并通过分配器为每个 task 分配一个虚拟 IP，即 VIP。
然后再将该 task 注册到内置的 DNS 中。当 service 的某 task 不可用时， 编排器会在 DNS 中  
注销该 task。

（5） 分发器

​    在 swarm manager 中具有一个分发器，用于完成对副本 task 任务的监听、调度等操作。
在前面的例子中，当编排器创建了 3 个 task 副本任务后，会调用分发器为每个 task 分配节
点。 分发器首先会在 swarm 集群的所有节点中找到 3 个 available node 可用节点，每个节点
上分配一个 task。而每个 task 就像是一个“插槽”， 分发器会在每个“插槽”中放入一个应
用容器。每个应用容器其实就是一个具体的 task 实例。一旦应用容器运行起来， 分发器就
可以监测到其运行状态，即 task 的运行状态。
​    如果容器不可用或被终止， task 也将被终止。此时编排器会立即在内置 DNS 中注销该
task，然后编排器会再生成一个新的 task，并在 DNS 中进行注册，然后再调用分发器为之分
配一个新的 available node，然后再该节点上再运行应用容器。 编排器始终维护着 3 个 task
副本任务。
​    分发器除了为 task 分配节点外，还实现了对访问请求的负载均衡。当有客户端来访问
swarm 提供的 service 服务时，该请求会被 manager 处理：根据其内置 DNS，实现访问的负
载均衡。

**3.1.4.服务部署模式**

（1） 官方图

​    service 以副本任务 task 的形式部署在 swarm 集群节点上。根据 task 数量与节点数量的
关系，常见的 service 部署模式有两种： replicated 模式与 global 模式。

<img src="images\image-20231025163423853.png" alt="image-20231025163423853" style="zoom:80%;" />

（2） replicated 模式

​    replicated 模式，即副本模式， service 的默认部署模式。需要指定 task 的数量。当需要
的副本任务 task 数量不等于 swarm 集群的节点数量时，就需要使用 replicated 模式。 manager
中的分发器会找到指定 task 个数的 available node 可用节点，然后为这些节点中的每个节点
分配一个或若干个 task。

（3） global 模式

​    global 模式，即全局模式。 分发器会为每个 swarm 集群节点分配一个 task，不能指定 task
的数量。 swarm 集群每增加一个节点， 编排器就会创建一个 task，并通过分发器分配到新的
节点上。

## 3.2.swarm 集群搭建*

**3.2.1.需求（准备阶段）**

现要搭建一个 docker swarm 集群，包含 5 个 swarm 节点。这 5 个 swarm 节点的 IP 与暂
时的角色分配如下（注意，是暂时的）：  

| hostname | ip              | role    |
| -------- | --------------- | ------- |
| docker1  | 192.168.110.101 | manager |
| docker2  | 192.168.110.102 | manager |
| docker3  | 192.168.110.103 | manager |
| docker4  | 192.168.110.104 | worker  |
| docker5  | 192.168.110.105 | worker  |

**3.2.2.克隆主机**

​    克隆 docker 主机，这5台主机名分别为docker1、 docker2、 docker3、 docker4 与 docker5。
克隆完毕后修改如下配置文件（以 docker1 为例）：

- 修改主机名： vim /etc/hostname

```bash
docker1
```

- 修改网络配置： vim /etc/sysconfig/network-scripts/ifcfg-ens33

```bash
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="static"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="ens33"
DEVICE="ens33"
ONBOOT="yes"
IPADDR=192.168.110.101
NETMASK=255.255.255.0
GATEWAY=192.168.110.2
DNS1=8.8.8.8
DNS2=8.8.4.4
```



**3.2.3.第一步 查看 swarm 激活状态**

​    在任意 docker 主机上通过 docker info 命令可以查看到当前 docker 引擎 Server 端对于
swarm 的激活状态。由于尚未初始化 swarm 集群，所以这些 docker 主机间没有任何关系，  
且 swarm 均未被激活。

`docker info`

```bash
[root@docker1 ~]# docker info
Client: Docker Engine - Community
 Version:    24.0.6
...
Server:
...
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive			# 这里说明swarm集群没有活动
 Runtimes: io.containerd.runc.v2 runc
 Default Runtime: runc
 Init Binary: docker-init
```

​    在任意 docker 主机上通过 docker info 命令可以查看到当前 docker 引擎 Server 端对于
swarm 的激活状态。由于尚未初始化 swarm 集群，所以这些 docker 主机间没有任何关系，  
且 swarm 均未被激活。

**3.2.4.第二步 swarm 初始化**

​    在主机名为“docker” 的主机上运行 docker swarm init 命令，创建并初始化一个 swarm。

```bash
[root@docker1 ~]# docker swarm init
Swarm initialized: current node (v4oae6qx5vy7fsfa614zlhnof) is now a manager.

To add a worker to this swarm, run the following command:
# 如果想添加一个worker，复制这条命令到worker服务器执行
    docker swarm join --token SWMTKN-1-4271ztf28268up945rj2pw77z9ldfhotbxa9p1ye7bwflg4k3i-07rau3ixwkb2r9c5w900l7ty8 192.168.110.101:2377
# 如果想添加一个manager，复制这条命令到本服务器执行，会给出一个命令，再复制到manager服务器执行
To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

**3.2.5.第三步 添加 worker 节点**

​    复制 docker swarm init 命令的响应结果中添加 wroker 节点的命令在 docker4 与 docker5
节点上运行，将这两个节点添加为 worker 节点。

```bash
[root@docker4 ~]# docker swarm join --token SWMTKN-1-4271ztf28268up945rj2pw77z9ldfhotbxa9p1ye7bwflg4k3i-07rau3ixwkb2r9c5w900l7ty8 192.168.110.101:2377
This node joined a swarm as a worker.
```

```bash
[root@docker5 ~]# docker swarm join --token SWMTKN-1-4271ztf28268up945rj2pw77z9ldfhotbxa9p1ye7bwflg4k3i-07rau3ixwkb2r9c5w900l7ty8 192.168.110.101:2377
This node joined a swarm as a worker.
```



**3.2.6.第四步 添加 manager 节点**

（1） 获取添加命令

​    若要为 swarm 集群添加 manager 节点，需要首先在 manager 节点获取添加命令。

```bash
[root@docker1 ~]# docker swarm join-token manager
To add a manager to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-4271ztf28268up945rj2pw77z9ldfhotbxa9p1ye7bwflg4k3i-cl7o3cixbi8gtm9j0mu0p4dbo 192.168.110.101:2377
```

（2） 添加节点

​    复制 docker swarm join-token 命令生成的 manager 添加命令，然后在 docker2 与 docker3
节点上运行，将这两个节点添加为 manager 节点。

```bash
[root@docker2 ~]# docker swarm join --token SWMTKN-1-4271ztf28268up945rj2pw77z9ldfhotbxa9p1ye7bwflg4k3i-cl7o3cixbi8gtm9j0mu0p4dbo 192.168.110.101:2377
This node joined a swarm as a manager.
```

```bash
[root@docker3 ~]# docker swarm join --token SWMTKN-1-4271ztf28268up945rj2pw77z9ldfhotbxa9p1ye7bwflg4k3i-cl7o3cixbi8gtm9j0mu0p4dbo 192.168.110.101:2377
This node joined a swarm as a manager.
```



**3.2.7.查看 swarm 节点**

​    在 manager 节点 docker1、 docker2、 docker3 上通过 docker node ls 命令可以查看到当前  
swarm 集群所包含的节点状态数据。
但在 worker 节点上是不能运行 docker node ls 命令的。* 代表当前所在的主机。

```bash
[root@docker1 ~]# docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
v4oae6qx5vy7fsfa614zlhnof *   docker1    Ready     Active         Leader           24.0.6
ic8lmc4sy6230twxkv3qzf0u2     docker2    Ready     Active         Reachable        24.0.6
e7jd5dw6rvieqatsru7qztm9i     docker3    Ready     Active         Reachable        24.0.6
y403qqf2v2pqd8jfva36nr0pa     docker4    Ready     Active                          24.0.6
yu71e8fkq9k0x0ih6brv73lvd     docker5    Ready     Active                          24.0.6
```



## 3.3.swarm 集群维护

**3.3.1.退出 swarm 集群**

​    当一个节点想从 swarm 集群中退出时，可以通过 docker swarm leave 命令。不过 worker
节点与 manager 节点的退群方式是不同的。

（1） worker 退群
    对于 worker 节点退群，直接运行 docker swarm leave 命令即可。
此时在 manager 节点中查看节点情况，可以看到 docker5 已经 Down 了。  

（2） worker 重新加入
    首先在 manager 节点上运行 docker swarm join-token worker 命令，生成加入 worker 节点
的命令。
复制生成的命令，在 docker5 节点上运行，将此节点添加到 swarm 集群。

（3） 查看节点情况
    此时在 manager 节点中查看节点情况，可以看到原来的 docker5 依然是 Down，但又新
增了一个新的 docker5 节点，其状态为 Ready。



​    此时在 manager 节点通过 docker info 命令可以查看到节点数量变为了 6 个，这增加的
一个就是两种状态的 docker5。  

（4） 删除 Down 状态节点
    对于Down状态的节点是完全可以将其删除的。通过在manager节点运行docker node rm
命令完成。

（5） manager 退群
    对于 manager 节点，原则上是不推荐直接退群的，这样会导致 swarm 集群的一致性受
到损坏。如果 manager 执意要退群，可在 docker swarm leave 命令后添加-f 或--force 选项进
行强制退群。  

**3.3.2.swarm 自动锁定**

（1） swarm 集群自动锁定原理
    在 manager 集群中， swarm 通过 Raft 日志方式维护了 manager 集群中数据的一致性。
即在 manager 集群中每个节点通过 manager 间通信方式维护着自己的 Raft 日志。
    但在通信过程中存在有一种风险： Raft 日志攻击者会通过 Raft 日志数据的传递来访问、
篡改 manager 节点中的配置或数据。为了防止被攻击， swarm 开启了一种集群自动锁定功能，
为 manager 间的通信启用了 TLS 加密。用于加密和解密的公钥与私钥，全部都维护在各个节
点的 Docker 内存中。一旦节点的 Docker 重启，则密钥丢失。
    swarm 中通过 autolock 标志来设置集群的自动锁定功能：为 true 则开启自动锁定，为
false 则关闭自动锁定。

（2） 设置自动锁定
在 manager 节点通过 docker swarm update –autolock=true 命令可以开启当前 swarm 集群
的自动锁定功能。
此时查看 manager 的 docker info 可以看到， autolock 已经为 true 了。  

（3） 查看解锁密钥
如果没有保存 docker swarm update --autolock=true 命令中生成的密钥，也可通过在
manager 中运行 docker swarm unlock-key 命令查看。  

（4） 关闭一个 manager
直接关闭 docker3 的 docker 引擎，模拟一个 manager 宕机的情况。

（5） 加入 manager
    启动 docker3 的 docker 引擎。

​    此时再查看该节点的 docker info，可以看到 Swarm 值为 locked，即当前节点看到的 Swarm
集群的状态为锁定状态，其若要加入，必须先解锁。  

​    在 docker3 中运行 docker swarm unlock 命令，解锁 swarm。

​    此时再查看节点信息，该 manager 已经加入。

## 3.4.swarm 节点维护

**3.4.1.角色转换**

​    Swarm 集群中节点的角色只有 manager 与 worker，所以其角色也只是在 manager 与
worker 间的转换。即 worker 升级为 manager，或 manager 降级为 worker。  

（1） worker 升级为 manager
通过 docker node promote 命令可以将 worker 升级为 manager。例如，下面的命令是将
docker4 与 docker5 两个节点升级为了 manager，即当前集群中全部为 manager。

（2） manager 降级为 worker
通过 docker node demote 命令可以将 manager 降级为 worker。例如，下面的命令是将
docker2 与 docker3 两个节点降级为了 worker。

（3） docker node update 变更角色
除了通过 docker node demote|promote 可以变更节点角色外，通过 docker node update
--role [manager|worker] [node]也可变更指定节点的角色。  

​    以下命令将 docker2 与 docker3 两个节点又变为了 manager。
​    以下命令将 docker4 与 docker5 两个节点又变为了 worker。

**3.4.2.节点标签**

​    swarm 可以通过命令为其节点添加描述性标签，以方便管理员去了解该节点的更多信息。

（1） 添加/修改节点标签
通过 docker node update --label-add 命令可以为指定 node 添加指定的 key=value 的标签。
若该标签的 key 已经存在，则会使用新的 value 替换掉该 key 的原 value。不过需要注意的是，
若要添加或修改多个标签，则需要通过多个--label-add 选项指定。 
通过 docker node inspect 在查看该节点详情时可看到添加的标签。
docker node inspect --pretty 可以 key:value 的形式显示信息。

（2） 删除节点标签
通过 docker node update --label-rm 命令可以为指定的 node 删除指定 key 的标签。同样，
若要删除多个标签，则需要通过多个--label-rm 选项指定要删除 key 的标签。  
查看节点详情，发现这两个标签已经消失。

**3.4.3.节点删除**

​    manager 节点通过 docker node rm 命令可以删除一个 Down 状态的、指定的 worker 节点。
注意，该命令只能删除 worker 节点，不能删除 manager 节点。

（1） 有问题的删除
    对于 Ready 状态的 worker 节点是无法直接删除的。  



​    对于 manager 节点也是无法删除的。



（2） 正确的删除
    若要删除一个 worker 节点，首先要将该节点的 Docker 关闭，使该节点变为 Down 状态，
然后再进行删除。
    关闭 docker2 节点的 Docker 引擎：
    删除节点：  

（3） 强制删除
    前面的删除方式有些麻烦，其实也可以通过添加-f 选项来实现强制删除。

​    但对于 manager 节点，强制删除也不能删除。

​    docker node rm –f 命令会使一个节点强制退群，而 docker swarm leave 命令是使当前的
docker 主机关闭 swarm 模式。  

## 3.5.swarm 安全(PKI)

​    Docker 内置了 PKI（public key infrastructure，公钥基础设施），使得保障发布容器化的业
务流程系统的安全性变得很简单。

**3.5.1.TLS 安全保障**

​    Swarm 节点之间采用 TLS 来鉴权、授权和加密通信。
​    具体来说是，当运行 docker swarm init 命令时， Docker 指定当前节点为一个 manager
节点。默认情况下， manager 节点会生成一个新的 swarm 的 CA 根证书以及一对密钥。 同时，
manager 节点还会生成两个 token，一个用于添加 worker 节点，一个用于添加 manager 节点。
每个 token 包含上了 CA 根证书的 digest 和一个随机密钥。 CA 根证书、一对密钥和随机密钥
都将会被用在节点之间的通信上。



当有节点加入 Swarm 时， 需要复制 manager 中相应的 docker swarm join 加入命令，并
在该节点中运行。而这个过程主要是通过随机密钥这种对称验证方式保障通信安全的。  



一旦节点加入了 Swarm 集群，那么它们间的通信全部都是通过 TLS 加密方式进行的。
首先是通过 CA 证书对通信对方的身份进行验证，在验证通过后再进行数据通信。而通信的
数据则是通过随机密钥加密过的。

**12.5.2.CA 数字证书轮换**

（1） 轮换周期
    Swarm 的 CA 数字证书也是有可能被攻击、篡改的。为了保证 swarm 的数字证书的安全  
性， Swarm 提供了 CA 数字证书轮换机制，定期更换 CA 数字证书。默认 swarm 的 CA 数字
证书 90 天轮换一次。

（2） 指定证书
    那么，用于轮换的新的 CA 数字证书来自于哪里呢？通过 docker swarm ca 命令可以指定
外部 CA 数字证书，或生成新的 CA 数字证书。无论哪种数字证书变更方式，都需要 CA 根证
书的加密/解密。而根证书也是会发生变化的，具体见“轮转过程”。

（3） 轮转过程
    当 manager 运行了 docker swarm ca --rotate 命令后，会按顺序发生下面的事情：

- Docker 会生成一个交叉签名（cross-signed） 根证书， 即新根证书是由旧的根证书签署
生成的， 这个交叉签名根证书将作为一个过渡性的根证书。这是为了确保节点仍然能够
信任旧的根证书，也能使用新的根证书验证签名。
- 在 Docker 17.06 或者更高版本中， Docker 会通知所有节点立即更新根证书。根据 swarm
中节点数量多少，这个过程可能会花费几分钟时间。
- 在所有的节点都更新了新 CA 根证书后， manager 会通知所有节点仅信任新的根证书，
不再信任旧根证书及交叉签名根证书。
- 所有节点使用新根证书签发自己的数字证书。
如果直接使用外部的 CA 根证书，那么就不存在交叉签名根证书的生成过程，直接由运
行docker swarm ca命令的节点通知所有节点立即更新根证书。后续过程与前面的就相同了。  

## 3.6.manager 集群容灾  










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

# 135.几个基础知识

## 135.1.HTTP/HTTPS 协议

（1） 协议
HTTP 与 HTTPS 协议都是客户端浏览器和服务器间的一种约定，约定如何将服务器中的
信息下载到本地，并通过浏览器显示出来。
不同的是， HTTP 协议是一种明文传输协议，其对传输的数据不提供任何加密措施。而
HTTPS 协议则是通过 SSL/TLS 为数据加密，以保障数据的安全性。  

<img src="images\image-20231024212744614.png" alt="image-20231024212744614" style="zoom:80%;" />

通常情况下， HTTP 会直接与运输层的 TCP 进行通信，默认使用 80 端口号。 但在使用
SSL/TLS 协议的 HTTPS 后，就演变成了直接与运输层的 SSL/TLS 进行通信，再由 SSL/TLS 与 TCP
进行通信。即 HTTPS 是间接与 TCP 进行通信的。 HTTPS 默认使用 443 端口号。

（2） SSL/TLS
SSL， Secure Sockets Layer，安全套接字协议
TLS， Transport Layer Security，传输层安全协议
它们主要用于保障在 Internet 上数据传输的安全性与完整性。

（3） 加密验证方式HTTP 协议通过明文传输存在三大风险：
- 被窃听的风险
- 被篡改的风险
- 被冒充的风险 

HTTPS 协议通过<font color= red>用户身份验证</font>与<font color= red>传输加密</font>，大大降低了 HTTP 中的这些风险。
HTTPS 协议中的身份验证采用的是<font color= red>非对称加密验证</font>方式，而传输加密采用的则是<font color= red>对称加密验证</font>方式。

- 对称加密验证：加密与解密使用的密钥相同。例如，登录使用的账号/密码就属于对称加密验证（用户提交的账号/密码与保存在服务器中的账号/密码必须相同，验证才能成功）。
- 非对称加密验证：加密与解密使用的密钥不同。其本质上就是数据算法。主要有三类算法：因子分解算法、离散对数算法、椭圆曲线算法。其中，因了分解算法使用最为广泛。

（4） 公钥与私钥
非对称加密验证中需要一对密钥。其中一个用于加密，一个用于解密，即所谓的公钥与
私钥。
- 公钥：可以公开的密钥，是发放给其他人的密钥

- 私钥：非公开的密钥，是只有加密者自己保存的密钥

  公钥与私钥都可用于加密/解密。

- 公钥加密，私钥解密：称为信息加密与信息解密

- 私钥加密，公钥解密：称为数字签名与签名验证



## 135.2.以故事方式开始 HTTPS 工作原理  

以“特工张三与总部李四的通信故事”来说明 HTTPS 的工作原理。

**（1） 明文通信过程**  

<img src="images\image-20231024213728736.png" alt="image-20231024213728736" style="zoom:80%;" />

存在的问题是，信息可能被劫持（数据被窃取、被篡改），数据非常不安全。

**（2） 使用数字签名加密**

整个通信过程包含两个阶段：通信关系建立阶段与通信阶段。  

<img src="images\image-20231024213833536.png" alt="image-20231024213833536" style="zoom:80%;" />

<img src="images\image-20231024213938451.png" alt="image-20231024213938451" style="zoom:80%;" />

**（3） 钓鱼问题**  

<img src="images\image-20231024214033403.png" alt="image-20231024214033403" style="zoom:80%;" />

**（4） 使用数字证书**

整个通信过程包含三个阶段：通信基础构建阶段、通信关系建立阶段与通信阶段。  

<img src="images\image-20231024214128396.png" alt="image-20231024214128396" style="zoom:80%;" />

<img src="images\image-20231024214217437.png" alt="image-20231024214217437" style="zoom:80%;" />

<img src="images\image-20231024214253892.png" alt="image-20231024214253892" style="zoom:80%;" />

**（5） 对称加密通信**

整个通信过程包含三个阶段：通信基础构建阶段、通信关系建立阶段与通信阶段。通信阶段中的身份验证采用非对称加密验证方式，通信过程采用对称加密验证方式。  

<img src="images\image-20231024214527704.png" alt="image-20231024214527704" style="zoom:80%;" />

<img src="images\image-20231024214553440.png" alt="image-20231024214553440" style="zoom:80%;" />

<img src="images\image-20231024214621571.png" alt="image-20231024214621571" style="zoom:80%;" />

**（6） HTTPS 通信原理**  

<img src="images\image-20231024214702513.png" alt="image-20231024214702513" style="zoom:80%;" />

<img src="images\image-20231024214734494.png" alt="image-20231024214734494" style="zoom:80%;" />

<img src="images\image-20231024214818311.png" alt="image-20231024214818311" style="zoom:80%;" />





## 135.3.HTTPS 重要概念

**（1） 数字证书**
    数字证书，也称为 SSL/TLS 证书， 是互联网通讯中标志通讯各方身份信息的一串数字。
提供了一种在 Internet 上验证通信实体身份的方式。它是由 CA（Certificate Authority，证书
权威认证机构，证书中心） 颁发的一种身份证明。它里面包含了该通讯方的公钥、 证书有效  
时间、 域名及 CA 的数字签名等。 数字证书的一个非常重要的作用就是“防钓鱼”。  
    全球的 CA（权威证书中心）一共也没有几个，即全球可以颁发数字证书的机构并不多。
而像我国阿里、腾讯等也都属于这些大的权威证书中心的代理机构。我们可以通过他们来申
办证书，而他们本身并不具有生成证书的权限。  

**（2） 根证书**
    数字证书是由 CA（Certificate Authority，证书权威认证机构，证书中心） 颁发的一种身
份证明，是通过 CA 私钥加密过的。所以，客户端必须具有 CA 公钥才能解密要访问平台服
务器的数字证书。而这个 CA 公钥就被称为 CA 根证书，也称为根证书。
    当然，数字证书除了权威证书中心可申请到外，也可自己生成。但自己生成的证书并没
有在客户端系统中，这时就需要用户在客户端先安装数字证书，并将其添加到相应的“信任”
状态才可。这就是我们平时如果要在本地电脑中打开网银平台对自己的电子银行进行操作之
前，会先提示安装根证书的原因。这个安装根证书的提示，就包含网银的数字证书与 CA 根
证书。
**（3） 数字摘要**
    数字摘要是将任意长度的消息变成固定长度的短消息。数字摘要就是利用了 Hash 函数
的<font color=red> 单向性</font>， 将需要加密的明文“摘要” 成一串 128 位长度数字串。这个数字串又称为数字指
纹。其单向性体现在： 不同明文“摘要的结果” 一定是不同的， 相同明文“摘要的结果” 必
定是一致。 但摘要结果无法计算出其原始明文。
**（4） 数字签名**
    数字签名，是只有信息的发送者才能产生的别人无法伪造的一段数字串。它是一种类似
写在纸上的手写物理签名，用于鉴别数字信息是否被篡改的方法。
数字签名是非对称密钥技术与数字摘要技术的应用。 使用私钥对明文的数字摘要加密，
形成数字签名；使用公钥对数字签名解密，称为签名验证。  

## 135.4.htpasswd 命令 

registry 私有镜像中心中默认是没有用户认证功能的，可通过 htpasswd 来实现用户认证。

**（1） 简介**

htpasswd 是开源 Web 服务器 Apache HTTP Server 的内置工具，用于创建、更新 HTTP
基本认证的密码文件。  

需要时再补充。。。

## 135.5.容器的退出状态码

Docker Daemon 通过退出状态码向用户反馈<font color=red>容器中应用</font>的退出方式。

**（1） 状态码分类**

容器退出状态码是[0, 255]范围内的整数，分为三类： 0、 [1,128]与[129,255]。
- 状态码0表示容器中应用是正常退出，例如通过docker stop命令退出/bin/bash，或docker
引擎关闭后引发的容器退出/bin/bash 等。
- [1,128]范围内的状态码，非正常退出状态，表示容器内部运行错误引发的容器无法启动，
或应用运行出错。
- [129,255]范围内的状态码，非正常退出状态，表示容器接收到终止信号而退出。

常见的状态码如下：

**（2） 状态码 1**

常见有两种情况可能会引发容器的退出状态码为 1。

- 应用程序内部错误。例如，分母为 0，内存溢出，数组下标越界等。

- Dockerfile 中的无效引用。即 Dockerfile 中引用了不存在的文件，导致容器无法启动。

**（3） 状态码 125**

  容器启动后要执行指定的[command]，但[command]没有运行成功。没有成功的原因通
  常是[command] 引用了未定义的变量，或执行了没有权限的命令等。

**（4） 状态码 126**

  容器启动后要执行指定的[command]，但[command]没有运行成功。没有成功的原因通
  常是该[command]的执行缺少依赖。
  例如， Dockerfile 中要运行 CMD 为[“java”,”-jar”,”demo.jar”]，而该容器中没有 JDK。

**（5） 状态码 127**

  容器启动后要执行指定的[command]，但[command]没有运行成功。没有成功的原因通
  常是该[command]中引用了不存在的文件或目录。
  例如， Dockerfile 中要运行 CMD 为[“java”,”-jar”,”demo.jar”]，而该容器中没有 demo.jar。  

**（6） 状态码 128**

由自己开发的容器内的代码触发了退出命令并给出了退出状态码，但状态码不在 0-255
范围内。此时返回给用户的退出状态码为 128。

**（7） 状态码 130**

当容器中的应用接收到来自操作系统的终止信号时，应用会立即退出，并返回给用户
130 状态码。

**（8） 状态码 137**

当容器中的应用接收到来自 dockerd 的强制终止信号时会立即退出，并返回给用户 137
状态码。

**（9） 状态码 143**

当容器接收到来自 dockerd 的优雅终止信号时，如果当前容器并没有用户访问，那么容
器会立即退出，并返回给用户 143 状态码。如果容器正好有用户访问，那么 dockerd 会等待
10s。 10s 后会向容器发送 kill 信号，此时返回给用户 137 状态码。  

**（10） 退出状态码查看方式**

查看退出状态码的方式有两种： docker ps –a 与 docker inspcet。  



## 134.6.容器的重启策略 

docker 容器启动后并不会永远处于运行状态，各种意外都可能会导致容器退出。在生产
环境下容器退出后采用手动重启方式肯定是非常低效或不可行的。 而 Docker 引擎提供了容
器的重启策略，通过在容器创建时指定--restart 选项不同的值，来达到不同的重启效果。 其
取值有以下几种：
**（1） no**
默认策略，在容器退出时不重启容器。
**（2） on-failure[:n]**
在容器非正常退出时，即退出状态码非 0 的情况下才会重启容器。 其后还可以跟一个整
型数，表示重启的次数。
**（3） always**
只要容器退出就会重启容器。  

**（4） unless-stopped**
只要容器退出就会重启容器，除非通过 docker stop 或 docker kill 命令停止容器。


# CI/CD之jenkins