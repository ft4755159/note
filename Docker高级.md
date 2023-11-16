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

**（1） 架构图**

<img src="images\image-20231025162902927.png" alt="image-20231025162902927" style="zoom:80%;" />

**（2） swarm node**  

​    从物理上讲， 一个 Swarm 是由若干安装了 Docker Engine 的物理机或者虚拟机组成，这
些主机上的 Docker Engine 都采用 Swarm 模式运行。
​    从逻辑上讲，一个 Swarm 由若干节点 node 构成，每个 node 最终会落实在一个物理
Docker 主机上，但一个物理 Docker 主机并不一定就是一个 node。即 swarm node 与 Docker
主机并不是一对一的关系。
swarm node 共有两种类型： manager 与 worker。

**（3） Manager**

​    Manager 节点用于维护 swarm 集群状态、调试 servcie、处理 swarm 集群管理任务。为
了防止单点故障问题，一个 Swarm 集群一般都会包含多个 manager。这些 manager 间通过
Raft 算法维护着一致性。

**（4） Worker**

​    Worker 节点用于在其 Contiainer 中运行 task 任务，即对外提供 service 服务。默认情况
下， manager 节点同时也充当着 worker 角色，可以运行 task 任务。

**（5） 角色转换**

​    manager 节点与 worker 节点角色并不是一成不变的，它们之间是可以相互转换的。
- manager 转变为 worker 称为节点降级
- worker 转变为 manager 称为节点升级

**3.1.3.服务架构**

**（1） 架构图**

<img src="images\image-20231025163121338.png" alt="image-20231025163121338" style="zoom:80%;" />

<img src="images\image-20231025163217251.png" alt="image-20231025163217251" style="zoom:80%;" />

**（2） service**

​    搭建 docker swarm 集群的目的是为了能够在 swarm 集群中运行应用，为用户提供具备
更强抗压能力的服务。 docker swarm 中的服务 service 就是一个逻辑概念，表示 swarm 集群
对外提供的服务。

**（3） task**

​    一个 service 最终是通过任务 task 的形式出现在 swarm 的各个节点中，而每个节点中的
task 又都是通过具体的运行着应用进程的容器对外提供的服务。

**（4） 编排器**

​    在 swarm manager 中具有一个编排器，用于管理副本 task 任务的创建与停止。例如，
当在 swarm manager 中定义一个具有 3 个 task 副本任务的 service 时， 编排器首先会创建 3
个 task，为每个 task 分配一个 taskID，并通过分配器为每个 task 分配一个虚拟 IP，即 VIP。
然后再将该 task 注册到内置的 DNS 中。当 service 的某 task 不可用时， 编排器会在 DNS 中  
注销该 task。

**（5） 分发器**

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

**（1） 官方图**

​    service 以副本任务 task 的形式部署在 swarm 集群节点上。根据 task 数量与节点数量的
关系，常见的 service 部署模式有两种： replicated 模式与 global 模式。

<img src="images\image-20231025163423853.png" alt="image-20231025163423853" style="zoom:80%;" />

**（2） replicated 模式**

​    replicated 模式，即副本模式， service 的默认部署模式。需要指定 task 的数量。当需要
的副本任务 task 数量不等于 swarm 集群的节点数量时，就需要使用 replicated 模式。 manager
中的分发器会找到指定 task 个数的 available node 可用节点，然后为这些节点中的每个节点
分配一个或若干个 task。

**（3） global 模式**

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

**（1） 获取添加命令**

​    若要为 swarm 集群添加 manager 节点，需要首先在 manager 节点获取添加命令。

```bash
[root@docker1 ~]# docker swarm join-token manager
To add a manager to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-4271ztf28268up945rj2pw77z9ldfhotbxa9p1ye7bwflg4k3i-cl7o3cixbi8gtm9j0mu0p4dbo 192.168.110.101:2377
```

**（2） 添加节点**

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



**3.2.7. 第五步 查看 swarm 节点**

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

**（1） worker 退群**
    对于 worker 节点退群，直接运行 docker swarm leave 命令即可。
此时在 manager 节点中查看节点情况，可以看到 docker5 已经 Down 了。  

```bash
[root@docker5 ~]# docker swarm leave
Node left the swarm.

[root@docker1 ~]# docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
v4oae6qx5vy7fsfa614zlhnof *   docker1    Ready     Active         Leader           24.0.6
ic8lmc4sy6230twxkv3qzf0u2     docker2    Ready     Active         Reachable        24.0.6
e7jd5dw6rvieqatsru7qztm9i     docker3    Ready     Active         Reachable        24.0.6
y403qqf2v2pqd8jfva36nr0pa     docker4    Ready     Active                          24.0.6
yu71e8fkq9k0x0ih6brv73lvd     docker5    Down      Active                          24.0.6

```



**（2） worker 重新加入**
    首先在 manager 节点上运行 docker swarm join-token worker 命令，生成加入 worker 节点
的命令。

```bash
[root@docker1 ~]# docker swarm join-token worker
To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-4271ztf28268up945rj2pw77z9ldfhotbxa9p1ye7bwflg4k3i-07rau3ixwkb2r9c5w900l7ty8 192.168.110.101:2377
```

复制生成的命令，在 docker5 节点上运行，将此节点添加到 swarm 集群。

```bash
[root@docker5 ~]# docker swarm join --token SWMTKN-1-4271ztf28268up945rj2pw77z9ldfhotbxa9p1ye7bwflg4k3i-07rau3ixwkb2r9c5w900l7ty8 192.168.110.101:2377
This node joined a swarm as a worker.
```

**（3） 查看节点情况**
    此时在 manager 节点中查看节点情况，可以看到原来的 docker5 依然是 Down，但又新
增了一个新的 docker5 节点，其状态为 Ready。

```bash
[root@docker1 ~]# docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
v4oae6qx5vy7fsfa614zlhnof *   docker1    Ready     Active         Leader           24.0.6
ic8lmc4sy6230twxkv3qzf0u2     docker2    Ready     Active         Reachable        24.0.6
e7jd5dw6rvieqatsru7qztm9i     docker3    Ready     Active         Reachable        24.0.6
y403qqf2v2pqd8jfva36nr0pa     docker4    Ready     Active                          24.0.6
8qgrrxzh32gb8fsog0fgj2emf     docker5    Ready     Active                          24.0.6
yu71e8fkq9k0x0ih6brv73lvd     docker5    Down      Active                          24.0.6
```

​    此时在 manager 节点通过 docker info 命令可以查看到节点数量变为了 6 个，这增加的
一个就是两种状态的 docker5。  

**（4） 删除 Down 状态节点**
    对于Down状态的节点是完全可以将其删除的。通过在manager节点运行docker node rm
命令完成。

```bash
[root@docker1 ~]# docker node rm yu71e8fkq9k0x0ih6brv73lvd
yu71e8fkq9k0x0ih6brv73lvd
```

**（5） manager 退群**
    对于 manager 节点，原则上是不推荐直接退群的，这样会导致 swarm 集群的一致性受
到损坏。如果 manager 执意要退群，可在 docker swarm leave 命令后添加-f 或--force 选项进
行强制退群。  

```bash
# 1 重新初始化，改变集群状态
docker swarm init --force-new-cluster
# 2 强制退群
docker swarm leave -f
# 3 建议方式：将manager降级为worker，再退群
```



**3.3.2.swarm 自动锁定**

**（1） swarm 集群自动锁定原理**
    在 manager 集群中， swarm 通过 Raft 日志方式维护了 manager 集群中数据的一致性。
即在 manager 集群中每个节点通过 manager 间通信方式维护着自己的 Raft 日志。
    但在通信过程中存在有一种风险： Raft 日志攻击者会通过 Raft 日志数据的传递来访问、
篡改 manager 节点中的配置或数据。为了防止被攻击， swarm 开启了一种集群自动锁定功能，
为 manager 间的通信启用了 TLS 加密。用于加密和解密的公钥与私钥，全部都维护在各个节
点的 Docker 内存中。一旦节点的 Docker 重启，则密钥丢失。
    swarm 中通过 autolock 标志来设置集群的自动锁定功能：为 true 则开启自动锁定，为
false 则关闭自动锁定。

**（2） 设置自动锁定**
    在 manager 节点通过 docker swarm update –autolock=true 命令可以开启当前 swarm 集群
的自动锁定功能。

```bash
[root@docker1 ~]# docker swarm update --autolock=true
Swarm updated.
To unlock a swarm manager after it restarts, run the `docker swarm unlock`
command and provide the following key:

    SWMKEY-1-FrAs8rteczZynN5eXqY9NHfp4zG8OOtZQ4CjcvugaYw	# 解锁密钥

Please remember to store this key in a password manager, since without it you
will not be able to restart the manager.
```

​    此时查看 manager 的 docker info 可以看到， autolock 已经为 true 了。  

**（3） 查看解锁密钥**
    如果没有保存 docker swarm update --autolock=true 命令中生成的密钥，也可通过在
manager 中运行 docker swarm unlock-key 命令查看。  

```bash
[root@docker1 ~]# docker swarm unlock-key
To unlock a swarm manager after it restarts, run the `docker swarm unlock`
command and provide the following key:

    SWMKEY-1-FrAs8rteczZynN5eXqY9NHfp4zG8OOtZQ4CjcvugaYw

Please remember to store this key in a password manager, since without it you
will not be able to restart the manager.
```

**（4） 关闭一个 manager**
    直接关闭 docker3 的 docker 引擎，模拟一个 manager 宕机的情况。

```bash
[root@docker3 ~]# systemctl stop docker
Warning: Stopping docker.service, but it can still be activated by:
  docker.socket
```

**（5） 加入 manager**
    启动 docker3 的 docker 引擎。

```bash
[root@docker3 ~]# systemctl start docker
```

​    此时再查看该节点的 docker info，可以看到 Swarm 值为 locked，即当前节点看到的 Swarm
集群的状态为锁定状态，其若要加入，必须先解锁。  

```bash
[root@docker3 ~]# docker info
...
 Swarm: locked
...
```

​    在 docker3 中运行 docker swarm unlock 命令，解锁 swarm。

```bash
[root@docker3 ~]# docker swarm unlock
Please enter unlock key: 					# 粘贴输入（2）（3）生成的key
```

​    此时再查看节点信息，该 manager 已经加入。

## 3.4.swarm 节点维护

**3.4.1.角色转换**

​    Swarm 集群中节点的角色只有 manager 与 worker，所以其角色也只是在 manager 与
worker 间的转换。即 worker 升级为 manager，或 manager 降级为 worker。  

**（1） worker 升级为 manager**
通过 docker node promote 命令可以将 worker 升级为 manager。例如，下面的命令是将
docker4 与 docker5 两个节点升级为了 manager，即当前集群中全部为 manager。

```bash
[root@docker1 ~]# docker node promote docker4
Node docker4 promoted to a manager in the swarm.
```

**（2） manager 降级为 worker**
通过 docker node demote 命令可以将 manager 降级为 worker。例如，下面的命令是将
docker2 与 docker3 两个节点降级为了 worker。

```bash
[root@docker1 ~]# docker node demote docker2
Manager docker2 demoted in the swarm.
```

**（3） docker node update 变更角色**
除了通过 docker node demote|promote 可以变更节点角色外，通过 docker node update
--role [manager|worker] [node]也可变更指定节点的角色。  

```bash
[root@docker1 ~]# docker node update --role manager docker2
docker2
```

```bash
[root@docker1 ~]# docker node update --role worker docker4
docker4
```

​    以下命令将 docker2 与 docker3 两个节点又变为了 manager。
​    以下命令将 docker4 与 docker5 两个节点又变为了 worker。

**3.4.2.节点标签**

​    swarm 可以通过命令为其节点添加描述性标签，以方便管理员去了解该节点的更多信息。

**（1） 添加/修改节点标签**
    通过 docker node update --label-add 命令可以为指定 node 添加指定的 key=value 的标签。
若该标签的 key 已经存在，则会使用新的 value 替换掉该 key 的原 value。不过需要注意的是，
若要添加或修改多个标签，则需要通过多个--label-add 选项指定。 

```bash
[root@docker1 ~]# docker node update --label-add auth=zs --label-add email=zs@163.com docker2
docker2
[root@docker1 ~]# docker node update --label-add auth=ls --label-add email=ls.163.com docker2
docker2
```

​    通过 docker node inspect 在查看该节点详情时可看到添加的标签。

```bash
            "Labels": {
                "auth": "ls",
                "email": "ls.163.com"
            },
```

​    docker node inspect --pretty 可以 key:value 的形式显示信息。

**（2） 删除节点标签**
    通过 docker node update --label-rm 命令可以为指定的 node 删除指定 key 的标签。同样，
若要删除多个标签，则需要通过多个--label-rm 选项指定要删除 key 的标签。  
查看节点详情，发现这两个标签已经消失。

```bash
[root@docker1 ~]# docker node update --label-rm auth docker2
docker2
```



**3.4.3.节点删除**

​    manager 节点通过 docker node rm 命令可以删除一个 Down 状态的、指定的 worker 节点。
注意，该命令只能删除 worker 节点，不能删除 manager 节点。

**（1） 有问题的删除**
    对于 Ready 状态的 worker 节点是无法直接删除的。  

```bash
[root@docker1 ~]# docker node rm docker5
Error response from daemon: rpc error: code = FailedPrecondition desc = node 7j3z1pb6azhp4dj8iz80atnyf is not down and can't be removed
```

​    对于 manager 节点也是无法删除的。

```bash
[root@docker1 ~]# docker node rm docker3
Error response from daemon: rpc error: code = FailedPrecondition desc = node st541p0oc2oe18zryc7oskldf is a cluster manager and is a member of the raft cluster. It must be demoted to worker before removal
```

**（2） 正确的删除**
    若要删除一个 worker 节点，首先要将该节点的 Docker 关闭，使该节点变为 Down 状态，
然后再进行删除。
    关闭 docker2 节点的 Docker 引擎：
    删除节点：  

```bash
[root@docker1 ~]# docker node rm docker5
docker5
```

```bash
[root@docker5 ~]# docker info
 Swarm: pending
```

**（3） 强制删除**
    前面的删除方式有些麻烦，其实也可以通过添加-f 选项来实现强制删除。

```bash
[root@docker1 ~]# docker node rm -f docker4
docker4
```

```bash
[root@docker4 ~]# docker info
 Swarm: active
```

​    但对于 manager 节点，强制删除也不能删除。

```bash
[root@docker1 ~]# docker node rm -f docker3
Error response from daemon: rpc error: code = FailedPrecondition desc = node st541p0oc2oe18zryc7oskldf is a cluster manager and is a member of the raft cluster. It must be demoted to worker before removal
```

**实质上：**

​    docker node rm –f 命令会使一个节点强制退群，而 docker swarm leave 命令是使当前的
docker 主机关闭 swarm 模式。  

​    所以想要重新加入 swarm 必须 leave+rm ，两步顺序不重要

```bash
[root@docker4 ~]# docker swarm leave
Node left the swarm.

[root@docker5 ~]# docker swarm leave
Node left the swarm.

```

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

<img src="images\image-20231103160340371.png" alt="image-20231103160340371" style="zoom:80%;" />

​    当有节点加入 Swarm 时， 需要复制 manager 中相应的 docker swarm join 加入命令，并
在该节点中运行。而这个过程主要是通过随机密钥这种对称验证方式保障通信安全的。  

<img src="images\image-20231103160422221.png" alt="image-20231103160422221" style="zoom:80%;" />

<img src="images\image-20231103160500914.png" alt="image-20231103160500914" style="zoom:80%;" />

​    一旦节点加入了 Swarm 集群，那么它们间的通信全部都是通过 TLS 加密方式进行的。
首先是通过 CA 证书对通信对方的身份进行验证，在验证通过后再进行数据通信。而通信的
数据则是通过随机密钥加密过的。

**12.5.2.CA 数字证书轮换**

**（1） 轮换周期**
    Swarm 的 CA 数字证书也是有可能被攻击、篡改的。为了保证 swarm 的数字证书的安全  
性， Swarm 提供了 CA 数字证书轮换机制，定期更换 CA 数字证书。默认 swarm 的 CA 数字
证书 90 天轮换一次。

**（2） 指定证书**
    那么，用于轮换的新的 CA 数字证书来自于哪里呢？通过 docker swarm ca 命令可以指定
外部 CA 数字证书，或生成新的 CA 数字证书。无论哪种数字证书变更方式，都需要 CA 根证
书的加密/解密。而根证书也是会发生变化的，具体见“轮转过程”。

**（3） 轮转过程**
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

**3.6.1.热备容灾**

​    Swarm 的 manager 节点集群采用的是热备方式来提升集群的容灾能力。即在 manager 集群中只有一个处于 leader 状态，用于完成 swarm 节点的管理，其余 manager 处于热备状 态。当 manager leader 宕机，其余 manager 就会自动发起 leader 选举，重新选举产生一个新 的 manager leader。

**3.6.2.容灾能力**

​    manager 集群的 leader 选举采用的是 Raft 算法。Raft 算法是一种比较复杂的一致性算法， 具体见后面“Raft 算法”。其选举 leader 的简单思路是，所有可用的 manager 全部具有选举 权与被选举权。最终获得过半选票的 manager 当选新的 leader。为了保证一次性可以选举出 新的leader，官方推荐使用奇数个manager。但并不是说偶数个manager就无法选举出leader。

**3.6.3.容灾模拟**

​    目前是 docker1、docker2、docker3 三个 manager，其中 docker1 为 leader。

```bash
[root@docker1 ~]# docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
08hv8a5lzmwsv65vb4u5gdv6r *   docker1    Ready     Active         Leader           24.0.6
bmbk2y45qjvpkfb0fxvglxn6e     docker2    Ready     Active         Reachable        24.0.6
st541p0oc2oe18zryc7oskldf     docker3    Ready     Active         Reachable        24.0.6
uq7jq3dnpq2qni74shakxomwh     docker4    Ready     Active                          24.0.6
pvzynnh5r30ujr76w7fvt7fno     docker5    Ready     Active                          24.0.6
```

​    现在关闭 docker1 主机的 docker daemon，模拟其宕机。

```bash
[root@docker1 ~]# systemctl stop docker
Warning: Stopping docker.service, but it can still be activated by:
  docker.socket
```

然后在 docker2 或 docker3 主机上查看当前的节点情况，可以看到 docker2 或 docker3 已经成为了新的 leader。

```bash
[root@docker2 ~]# docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
08hv8a5lzmwsv65vb4u5gdv6r     docker1    Ready     Active         Reachable        24.0.6
bmbk2y45qjvpkfb0fxvglxn6e *   docker2    Ready     Active         Reachable        24.0.6
st541p0oc2oe18zryc7oskldf     docker3    Ready     Active         Leader           24.0.6
uq7jq3dnpq2qni74shakxomwh     docker4    Ready     Active                          24.0.6
pvzynnh5r30ujr76w7fvt7fno     docker5    Ready     Active                          24.0.6
```

此时如果再使某个 manager 宕机，例如使 docker2 的 docker daemon 关闭，那么整个 swarm 就会瘫痪。因为剩下的 manager 已经无法达成过半的选票，无法选举出新的 leader。

```bash
[root@docker2 ~]# systemctl stop docker
Warning: Stopping docker.service, but it can still be activated by:
  docker.socket


[root@docker3 ~]# docker node ls
Error response from daemon: rpc error: code = Unknown desc = The swarm does not have a leader. It's possible that too few managers are online. Make sure more than half of the managers are online.
```





## 3.7.service 创建 

​    注意，service 只能依附于 docker swarm 集群，所以 service 的创建前提是，swarm 集群 搭建完毕。 

**3.7.1.创建\删除 service** 

**创建**

​    docker service create 命令用于创建 service，需要在 manager 中运行。其与创建容器的命 令 docker run 非常类似，具有类似的选项。 

​    目前的节点状态如下： 

```bash
[root@docker1 ~]# docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
08hv8a5lzmwsv65vb4u5gdv6r *   docker1    Ready     Active         Leader           24.0.6
bmbk2y45qjvpkfb0fxvglxn6e     docker2    Ready     Active         Reachable        24.0.6
st541p0oc2oe18zryc7oskldf     docker3    Ready     Active         Reachable        24.0.6
uq7jq3dnpq2qni74shakxomwh     docker4    Ready     Active                          24.0.6
pvzynnh5r30ujr76w7fvt7fno     docker5    Ready     Active                          24.0.6
```

​    现在要在 swarm 中创建一个运行 tomcat:8.5.49 镜像的 service，服务名称为 toms，包含 3 个副本 task，对外映射端口号为 9000。

```bash
[root@docker1 ~]# docker service create --name toms --replicas 3 -p 9000:8080 tomcat:8.5.49
xepenxqp6iq4u9dsilvtrnj7z
overall progress: 3 out of 3 tasks 
1/3: running   [==================================================>] 
2/3: running   [==================================================>] 
3/3: running   [==================================================>] 
verify: Service converged 
```

​     命令下生成的一串码为 service 的 ID。 

**删除**

docker service rm 命令用于删除 service

```bash
[root@docker1 ~]# docker service rm mytoms
mytoms
```

**3.7.2.查看服务列表** 

​    docker service ls 命令用于查看当前 swarm 集群中正在运行的 service 列表信息。一个 swarm 中可以运行多个 service。

```bash
[root@docker1 ~]# docker service ls
ID             NAME      MODE         REPLICAS   IMAGE           PORTS
xepenxqp6iq4   toms      replicated   3/3        tomcat:8.5.49   *:9000->8080/tcp
```

**3.7.3.查看服务详情** 

​    通过 docker service inspect [service name|service ID]命令可以查看指定 service 的详情。 

```bash
[root@docker1 ~]# docker service inspect toms
```

**3.7.4.用户访问服务** 

​    当服务创建完毕后，该服务也就运行了起来。此时用户就可通过浏览器进行访问了。用 户可以访问 swarm 集群中任意主机。 

访问 manager： http://docker1:9000

访问 worker：http://docker4:9000

**3.7.5.查看 task 节点** 

docker service ps [service name|service ID]命令可以查看指定服务的各个 task 所分配的节 点信息。

```bash
[root@docker1 ~]# docker service ps toms
ID             NAME      IMAGE           NODE      DESIRED STATE   CURRENT STATE                ERROR     PORTS
jw7lv4wp48ja   toms.1    tomcat:8.5.49   docker1   Running         Running about a minute ago             
vxe9y8d9zv71   toms.2    tomcat:8.5.49   docker2   Running         Running about a minute ago             
7go2wi8b4p97   toms.3    tomcat:8.5.49   docker5   Running         Running about a minute ago 
```

​    可以看到，toms 服务的 3 个 task 被分配到了 docker2、docker、docker5 三个主机。其 中 ID 为 task ID，NAME 为 task 的 name。task name 是 service name 后添加从 1 开始的流水号形成的。 

**3.7.6.查看节点 task** 

​    通过 docker node ps [node]可以查看指定节点中运行的 task 的信息。默认查看的是当前 节点的 task 信息。

```bash
[root@docker1 ~]# docker node ps
ID             NAME       IMAGE           NODE      DESIRED STATE   CURRENT STATE               ERROR     PORTS
e8brrqm81k3y   mytoms.4   tomcat:8.5.49   docker1   Running         Running 35 minutes ago                
jw7lv4wp48ja   toms.1     tomcat:8.5.49   docker1   Running         Running about an hour ago             
```

```bash
[root@docker1 ~]# docker node ps docker2
ID             NAME       IMAGE           NODE      DESIRED STATE   CURRENT STATE               ERROR     PORTS
jfdnppu3mfj7   mytoms.3   tomcat:8.5.49   docker2   Running         Running 33 minutes ago                
vxe9y8d9zv71   toms.2     tomcat:8.5.49   docker2   Running         Running about an hour ago             
```

**3.7.7.查看服务日志** 

​    通过 docker service logs 命令可以查看指定 service 或 task 的日志。通过 docker service logs  –f 命令可动态监听指定 service 或 task 的日志。 

**（1） 查看 service 日志** 

​    通过 docker service logs [service name|service ID]命令可以查看指定 service 的日志。这些 日志实际是所有 task 在节点容器中的运行日志。 

```bash
[root@docker1 ~]# docker service logs toms
```

**（2） 查看 task 日志** 

​    通过 docker service logs [task ID]命令可以查看指定 task 的日志。注意，这里只能指定 taskID，不能指定 task name。这些日志实际是指定 task 在节点容器中的运行日志。

```bash
[root@docker1 ~]# docker service ps toms
ID             NAME      IMAGE           NODE      DESIRED STATE   CURRENT STATE               ERROR     PORTS
jw7lv4wp48ja   toms.1    tomcat:8.5.49   docker1   Running         Running about an hour ago             
vxe9y8d9zv71   toms.2    tomcat:8.5.49   docker2   Running         Running about an hour ago             
7go2wi8b4p97   toms.3    tomcat:8.5.49   docker5   Running         Running about an hour ago             
[root@docker1 ~]# 
[root@docker1 ~]# 
[root@docker1 ~]# docker service logs jw7lv4wp48ja

```

**3.7.8.查看节点容器** 

​    在 docker1、docker2、docker5 三个主机中查看正在运行的容器列表，可以看到相应的 tomcat 容器。 

```bash
# 分别进入相应主机
[root@docker1 ~]# docker ps
CONTAINER ID   IMAGE           COMMAND             CREATED          STATUS          PORTS      NAMES
3cf982abfc65   tomcat:8.5.49   "catalina.sh run"   43 minutes ago   Up 43 minutes   8080/tcp   mytoms.4.e8brrqm81k3y2ixfos1tbe416
58e66ebb8a0e   tomcat:8.5.49   "catalina.sh run"   2 hours ago      Up 2 hours      8080/tcp   toms.1.jw7lv4wp48jajwh5p3xwflzzi
```

​    容器的 NAME 是由 task name 后添加 task ID 形成的。 

​    不过，在 docker3、docker4 主机中是没有该服务的 task 容器的。 



**3.7.9.负载均衡** 

​    当一个 service 包含多个 task 时，用户对 service 的访问最终会通过负载均衡方式转发给 各个 task 处理。这个负载均衡为轮询策略，且无法通过修改 service 的属性方式进行变更。

但由于该负载均衡为三层负载均衡，所以其可以通过第三方实现负载均衡策略的变更，例如 通过 Nginx、HAProxy 等。 

**（1） 创建 service** 

​    为了能够展示出 service 对访问请求负载均衡的处理方式，这里使用一个镜像 containous/whoami。该镜像中应用端口号为 80，通过浏览器访问，返回结果中包含很多信 息，其中最重要的是处理该请求的容器 ID。

​    下面的命令用于创建该镜像的一个 service，包含 5 个副本 task。 

```bash
[root@docker1 ~]# docker service create --name web --replicas 5 -p 8080:80 containous/whoami
qf37zmhp2pmr6ek840d563h6o
overall progress: 5 out of 5 tasks 
1/5: running   [==================================================>] 
2/5: running   [==================================================>] 
3/5: running   [==================================================>] 
4/5: running   [==================================================>] 
5/5: running   [==================================================>] 
verify: Service converged 
```

**（2） 记录容器 ID** 

​    可以看到，每个节点上都分配了一个 task，即每个节点上都运行了一个该 task 的容器。

```bash
[root@docker1 ~]# docker service ps web
ID             NAME      IMAGE                      NODE      DESIRED STATE   CURRENT STATE                ERROR     PORTS
qv829hjrv4bw   web.1     containous/whoami:latest   docker3   Running         Running 53 seconds ago                 
uvxyql7qko1q   web.2     containous/whoami:latest   docker5   Running         Running 53 seconds ago                 
p51qzku3jwb5   web.3     containous/whoami:latest   docker2   Running         Running 53 seconds ago                 
yfqcrw0l5g0z   web.4     containous/whoami:latest   docker1   Running         Running about a minute ago             
byqbu37h1cbz   web.5     containous/whoami:latest   docker4   Running         Running 53 seconds ago        
```

​     为了体现负载均衡的效果，这里需要将各个节点主机中该 service 的 task 容器的 ID 查询 并记录下来。

```bash
[root@docker1 ~]# docker ps
CONTAINER ID   IMAGE                      COMMAND             CREATED         STATUS         PORTS      NAMES
dd6b9c854d16   containous/whoami:latest   "/whoami"           8 minutes ago   Up 8 minutes   80/tcp     web.4.yfqcrw0l5g0zgdlrkr04m259h
```

```bash
[root@docker2 ~]# docker ps
CONTAINER ID   IMAGE                      COMMAND             CREATED         STATUS         PORTS      NAMES
4146a1c48186   containous/whoami:latest   "/whoami"           8 minutes ago   Up 8 minutes   80/tcp     web.3.p51qzku3jwb5h8g2nv7jzx2n1
```

```bash
[root@docker3 ~]# docker ps
CONTAINER ID   IMAGE                      COMMAND     CREATED         STATUS         PORTS     NAMES
a03c4debec79   containous/whoami:latest   "/whoami"   8 minutes ago   Up 8 minutes   80/tcp    web.1.qv829hjrv4bw85cpyv26kbrm3
```

```bash
[root@docker4 ~]# docker ps
CONTAINER ID   IMAGE                      COMMAND     CREATED         STATUS         PORTS     NAMES
8f56bcceef2b   containous/whoami:latest   "/whoami"   8 minutes ago   Up 8 minutes   80/tcp    web.5.byqbu37h1cbz6brpjnl3qlhp6
```

```bash
[root@docker5 ~]# docker ps
CONTAINER ID   IMAGE                      COMMAND             CREATED         STATUS         PORTS      NAMES
bbe55a978d6c   containous/whoami:latest   "/whoami"           8 minutes ago   Up 8 minutes   80/tcp     web.2.uvxyql7qko1q9zonlxq5bgea3
```

**（3） 访问** 

​    在任意主机上使用curl命令访问swarm集群中的任意节点，无论是manager还是worker， 快速访问后，在返回结果中的 Hostname 值就是处理该请求的容器的 ID，第 2 个 IP 为该节点 在 Swarm 集群局域网中的 IP。

​    从结果可以看出，这些请求被轮询分配给了各个 task 容器进行的处理，实现了 service 对访问请求的负载均衡。

```bash
[root@docker1 ~]# curl 192.168.110.101:8080
Hostname: 4146a1c48186
IP: 127.0.0.1
IP: 10.0.0.24
IP: 172.18.0.4
RemoteAddr: 10.0.0.2:47248
GET / HTTP/1.1
Host: 192.168.110.101:8080
User-Agent: curl/7.29.0
Accept: */*

[root@docker1 ~]# curl 192.168.110.101:8080
Hostname: a03c4debec79
IP: 127.0.0.1
IP: 10.0.0.22
IP: 172.18.0.3
RemoteAddr: 10.0.0.2:47254
GET / HTTP/1.1
Host: 192.168.110.101:8080
User-Agent: curl/7.29.0
Accept: */*

[root@docker1 ~]# curl 192.168.110.101:8080
Hostname: dd6b9c854d16
IP: 127.0.0.1
IP: 10.0.0.20
IP: 172.18.0.4
RemoteAddr: 10.0.0.2:47256
GET / HTTP/1.1
Host: 192.168.110.101:8080
User-Agent: curl/7.29.0
Accept: */*
```

## 3.8. service 操作 

**3.8.1.task 伸缩** 

​    根据访问量的变化，需要在不停止服务的前提下对服务的 task 进行扩容/缩容，即对服 务进行伸缩变化。有两种实现方式： 

**（1） docker service update 方式** 

​    通过 docker service update --replicas 命令可以实现对指定服务的 task 数量进行变更。 

```bash
[root@docker1 ~]# docker service update --replicas 4 toms
toms
overall progress: 4 out of 4 tasks 
1/4: running   [==================================================>] 
2/4: running   [==================================================>] 
3/4: running   [==================================================>] 
4/4: running   [==================================================>] 
```

​    此时可以看到新增了一个 task 节点。 

```bash
[root@docker1 ~]# docker service ps toms
ID             NAME      IMAGE           NODE      DESIRED STATE   CURRENT STATE            
jw7lv4wp48ja   toms.1    tomcat:8.5.49   docker1   Running         Running 5 hours ago       
vxe9y8d9zv71   toms.2    tomcat:8.5.49   docker2   Running         Running 5 hours ago       
7go2wi8b4p97   toms.3    tomcat:8.5.49   docker5   Running         Running 5 hours ago       
qukk49pesmzx   toms.4    tomcat:8.5.49   docker3   Running         Running 17 seconds ago   
```

**（2） docker service scale 方式** 

​    通过 docker service scale 命令可以为指定的服务变更 task 数量。

```bash
[root@docker1 ~]# docker service scale toms=7
toms scaled to 7
overall progress: 7 out of 7 tasks 
1/7: running   [==================================================>] 
2/7: running   [==================================================>] 
3/7: running   [==================================================>] 
4/7: running   [==================================================>] 
5/7: running   [==================================================>] 
6/7: running   [==================================================>] 
7/7: running   [==================================================>] 
verify: Service converged
```

​    此时可以看到新增了 3 个 task 节点。由于共有 5 台主机，现有 7 个 task，所以就出现了 一个主机上有多个 task 的情况。例如本例中，docker4 与 docker5 中分别有 2 个 task。

```bash
[root@docker1 ~]# docker service ps toms
ID             NAME      IMAGE           NODE      DESIRED STATE   CURRENT STATE             
...         
6fuba3hkwwwy   toms.5    tomcat:8.5.49   docker4   Running      Running about a minute ago   
wueehst6yv0q   toms.6    tomcat:8.5.49   docker4   Running      Running about a minute ago   
s588bxcmw68n   toms.7    tomcat:8.5.49   docker5   Running      Running about a minute ago
```

​    当然，也可以使 task 数量减小。例如，下面的命令使 task 又变回了 3 个。

```bash
docker service scale toms=3
```

​    这三个 task 分别在 docker2、docker 与 docker5 主机。

```bash
[root@docker1 ~]# docker service ps toms
ID             NAME      IMAGE           NODE      DESIRED STATE   CURRENT STATE         
jw7lv4wp48ja   toms.1    tomcat:8.5.49   docker1   Running         Running 5 hours ago       
vxe9y8d9zv71   toms.2    tomcat:8.5.49   docker2   Running         Running 5 hours ago       
7go2wi8b4p97   toms.3    tomcat:8.5.49   docker5   Running         Running 5 hours ago 
```

**（3） 暂停节点的 task 分配** 

​    生产环境下，可能由于某主机性能不高，在进行 task 扩容时，<font color=red>不想再为该主机再分配 更多的 task</font>，此时可通过 pause 暂停该主机节点的可用性来达到此目的。 

​    例如，当前 docker1、docker2 与 docker5 三个主机上的 toms 服务的 task 情况如下。 

```bash
[root@docker1 ~]# docker service ps toms
ID             NAME      IMAGE           NODE      DESIRED STATE   CURRENT STATE         
jw7lv4wp48ja   toms.1    tomcat:8.5.49   docker1   Running         Running 5 hours ago       
vxe9y8d9zv71   toms.2    tomcat:8.5.49   docker2   Running         Running 5 hours ago       
7go2wi8b4p97   toms.3    tomcat:8.5.49   docker5   Running         Running 5 hours ago 
```

​    现准备将 toms 服务的 task 扩容为 10，但保持 docker2 节点中的 task 数量仍为 1 不变， 此时就可通过 docker node update --availability pause 命令修改 docker2 节点的可用性。

```bash
[root@docker1 ~]# docker node update --availability pause docker2
docker2
[root@docker1 ~]# docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
08hv8a5lzmwsv65vb4u5gdv6r *   docker1    Ready     Active         Leader           24.0.6
bmbk2y45qjvpkfb0fxvglxn6e     docker2    Ready     Pause          Reachable        24.0.6
st541p0oc2oe18zryc7oskldf     docker3    Ready     Active         Reachable        24.0.6
uq7jq3dnpq2qni74shakxomwh     docker4    Ready     Active                          24.0.6
pvzynnh5r30ujr76w7fvt7fno     docker5    Ready     Active                          24.0.6
```

 将 toms 服务的 task 扩容为 10。

```bash
[root@docker1 ~]# docker service scale toms=10
toms scaled to 10
overall progress: 10 out of 10 tasks 
1/10: running   [==================================================>] 
2/10: running   [==================================================>] 
3/10: running   [==================================================>] 
4/10: running   [==================================================>] 
5/10: running   [==================================================>] 
6/10: running   [==================================================>] 
7/10: running   [==================================================>] 
8/10: running   [==================================================>] 
9/10: running   [==================================================>] 
10/10: running   [==================================================>] 
verify: Service converged
```

​    查看各节点分配的 task 情况会发现，原本应该平均分配到每个节点 2 个 task，但 docker2 的 task 数量<font color=red>并未增加</font>，所以其它节点主机（docker3）的就多于 2 个了。 

​    注意：task缩容的时候，docker node update --availability pause [host]命令pause的节点不会受影响（只减不增），这是<font color=red>因为pause的本意在于降低指定节点的负载</font>，所以缩容时会被减去负载。

```bash
[root@docker1 ~]# docker service ps toms
ID             NAME      IMAGE           NODE      DESIRED STATE   CURRENT STATE           
jw7lv4wp48ja   toms.1    tomcat:8.5.49   docker1   Running         Running 5 hours ago       
vxe9y8d9zv71   toms.2    tomcat:8.5.49   docker2   Running         Running 5 hours ago       
7go2wi8b4p97   toms.3    tomcat:8.5.49   docker5   Running         Running 5 hours ago       
wyj2bb3jv15o   toms.4    tomcat:8.5.49   docker3   Running         Running 13 seconds ago   
h4tek4ptugyh   toms.5    tomcat:8.5.49   docker4   Running         Running 14 seconds ago   
lzpiamdubfqg   toms.6    tomcat:8.5.49   docker3   Running         Running 13 seconds ago   
ivcii7mexahw   toms.7    tomcat:8.5.49   docker3   Running         Running 14 seconds ago   
rhl02niagarb   toms.8    tomcat:8.5.49   docker5   Running         Running 15 seconds ago   
mwjtq2up4054   toms.9    tomcat:8.5.49   docker1   Running         Running 15 seconds ago   
r1pbqsbsmjby   toms.10   tomcat:8.5.49   docker4   Running         Running 15 seconds ago
```

**（4） 清空 task** 

​    默认情况下，manager 节点同时也具备 worker 节点的功能，可以由分发器为其分配 task。 但 manager 节点使用 raft 算法来达成 manager 间数据的一致性，对资源较敏感。因此，阻 止 manager 节点接收 task 是比较好的选择。

​    或者，由于某节点出现了性能问题，需要停止服务进行维修，此时最好是将该节点上的 task 清空，以不影响 service 的整体性能。

​    通过 docker node update –availability drain 命令可以清空指定节点中的所有 task。

​    例如，目前各个节点的对于 toms 服务的 task 分配情况如下：

```bash
[root@docker1 ~]# docker service ps toms
ID             NAME      IMAGE           NODE      DESIRED STATE   CURRENT STATE           
jw7lv4wp48ja   toms.1    tomcat:8.5.49   docker1   Running         Running 5 hours ago       
vxe9y8d9zv71   toms.2    tomcat:8.5.49   docker2   Running         Running 5 hours ago       
7go2wi8b4p97   toms.3    tomcat:8.5.49   docker5   Running         Running 5 hours ago       
wyj2bb3jv15o   toms.4    tomcat:8.5.49   docker3   Running         Running 24 minutes ago   
h4tek4ptugyh   toms.5    tomcat:8.5.49   docker4   Running         Running 24 minutes ago
```

​    现对 docker2 与 docker5 两个节点进行 task 清空操作。

```bash
[root@docker1 ~]# docker node update --availability drain docker2
docker2
[root@docker1 ~]# docker node update --availability drain docker5
docker5
```

​    此时可以看到，toms 服务的 task 总量并没有减少，只是 docker2 与 docker5 两个节点上 是没有 task 的，而全部都分配到了 docker1、docker3 与 docker4 三个节点上了。这个结果就 是由编排器与分发器共同维护的。 

```bash
[root@docker1 ~]# docker service ls
ID             NAME      MODE         REPLICAS   IMAGE           PORTS
xepenxqp6iq4   toms      replicated   5/5        tomcat:8.5.49   *:9000->8080/tcp
[root@docker1 ~]# docker service ps toms
ID             NAME         IMAGE           NODE      DESIRED STATE   CURRENT STATE         
jw7lv4wp48ja   toms.1       tomcat:8.5.49   docker1   Running    Running 6 hours ago   
zzqn8c1gebf0   toms.2       tomcat:8.5.49   docker1   Running    Running about a minute ago 
vxe9y8d9zv71    \_ toms.2   tomcat:8.5.49   docker2   Shutdown   Shutdown about a minute ago 
r940308qvlax   toms.3       tomcat:8.5.49   docker3   Running    Running about a minute ago 
7go2wi8b4p97    \_ toms.3   tomcat:8.5.49   docker5   Shutdown   Shutdown about a minute ago 
wyj2bb3jv15o   toms.4       tomcat:8.5.49   docker3   Running    Running 31 minutes ago 
h4tek4ptugyh   toms.5       tomcat:8.5.49   docker4   Running    Running 31 minutes ago
```

**3.8.2.task 容错** 

​    当某个 task 所在的主机或容器出现了问题时，manager 的编排器会自动再创建出新的 task，然后分发器会再选择出一台 available node 可用节点，并将该节点分配给新的 task。

​    <font color=red>原则是设定好的task总数保持不变</font>，即使手动在某个主机上停掉容器，集群也会重新启动一个新的task维持总数。如果start重启手动关闭的容器，该容器会启动但不会重新回到集群task中。

**（1） 停掉容器** 

​    现在通过停掉 某个主机容器的方式来模拟故障情况。例 如停掉 docker4 的容器。 

```bash
[root@docker4 ~]# docker stop 9ddb9e7d0603
9ddb9e7d0603
```

**（2） 查看 task 节点** 

​    此时再查看服务的 task 节点信息可以看到，原来 docker4 上的 task 已经是 Shutdown 状 态了，而新增了一个新的 toms.1 的 task，其分配的是 docker4 主机。

```bash
[root@docker1 ~]# docker service ps toms
ID             NAME         IMAGE           NODE      DESIRED STATE   CURRENT STATE            ERROR                         PORTS
jw7lv4wp48ja   toms.1       tomcat:8.5.49   docker1   Running         Running 6 hours ago   
zzqn8c1gebf0   toms.2       tomcat:8.5.49   docker1   Running         Running 4 minutes ago 
vxe9y8d9zv71    \_ toms.2   tomcat:8.5.49   docker2   Shutdown        Shutdown 4 minutes ago 
r940308qvlax   toms.3       tomcat:8.5.49   docker3   Running         Running 4 minutes ago 
7go2wi8b4p97    \_ toms.3   tomcat:8.5.49   docker5   Shutdown        Shutdown 4 minutes ago 
wyj2bb3jv15o   toms.4       tomcat:8.5.49   docker3   Running         Running 34 minutes ago 
vlwp5d21ix1i   toms.5       tomcat:8.5.49   docker4   Running         Running 4 seconds ago 
h4tek4ptugyh    \_ toms.5   tomcat:8.5.49   docker4   Shutdown        Failed 11 seconds ago    "task: non-zero exit (143)"
```

 **3.8.3.服务删除** 

​    通过 docker service rm [service name|service ID]可以删除指定的一个或多个 service。 

```bash
docker service rm toms
```

​    删除后，该 service 消失，当然，该 service 的所有 task 也全部删除，task 相关的节点容 器全部消失。

```bash
# docker service ls
# docker service ps toms
```



**3.8.4.滚动更新** 

​    当一个 service 的 task 较多时，为了不影响对外提供的服务，在对 service 进行更新时可 采用滚动更新方式。 

**（1） 需求** 

​    这里要实现的更新时，将原本镜像为 tomcat:8.5.39 的 service 的镜像滚动更新为 tomcat:8.5.49。 

**（2） 创建 service** 

​    创建一个包含 10 个副本 task 的服务，该服务使用的镜像为 tomcat:8.5.39。

```bash
docker service create \
--name toms \
--replicas 10 \
--update-parallelism 2 \
--update-delay 3s \
--update-max-failure-ratio 0.2 \
--update-failure-action rollback \
--rollback-parallelism 2 \
--rollback-delay 3s \
--rollback-max-failure-ratio 0.2 \
--rollback-failure-action continue \
-p 9000:8080 \
tomcat:8.5.39
```

​    这 10 个 task 被非常平均的分配到了 5 个 swarm 节点上了。

```bash
[root@docker1 ~]# docker service ps toms
ID             NAME      IMAGE           NODE      DESIRED STATE  CURRENT STATE             
w7f6u8ixhmti   toms.1    tomcat:8.5.39   docker4   Running        Running about a minute ago 
ut4i40chkhvl   toms.2    tomcat:8.5.39   docker4   Running        Running about a minute ago 
vaztptbfv2nl   toms.3    tomcat:8.5.39   docker1   Running        Running about a minute ago 
38tmmikrl4sv   toms.4    tomcat:8.5.39   docker2   Running        Running 44 seconds ago     
iunrao3kzn5n   toms.5    tomcat:8.5.39   docker3   Running        Running about a minute ago 
mcn2xjycqm2n   toms.6    tomcat:8.5.39   docker5   Running        Running 39 seconds ago     
njkaglk5zfh5   toms.7    tomcat:8.5.39   docker5   Running        Running 39 seconds ago     
tnsjgk17u0lx   toms.8    tomcat:8.5.39   docker1   Running        Running about a minute ago 
i0q7qn4cnajz   toms.9    tomcat:8.5.39   docker3   Running        Running about a minute ago 
mpxnj4a7x5d8   toms.10   tomcat:8.5.39   docker2   Running        Running 44 seconds ago 
```

**（3） 更新 service** 

​    现要将 service 使用的镜像由 tomcat:8.5.39 更新为 tomcat:8.5.49。 

```bash
[root@docker1 ~]# docker service update --image tomcat:8.5.49 toms
toms
```

​    会发现这个更新的过程就是前面在创建服务时指定的那样，每次更新 2 个 task，更新间 隔为 3 秒。

​     更新完毕后再查看当前的 task 情况发现，已经将所有任务的镜像更新为了 8.5.49 版本。



**3.8.5.更新回滚** 

​    在更新过程中如果更新失败，则会按照设置的回滚策略进行回滚，回滚到更新前的状态。 但用户也可通过命令方式手工回滚。

​    下面的命令会按照前面设置的每次回滚 2 个 task，每次回滚间隔 3 秒进行回滚。下面的 是回滚过程中的某个回滚瞬间。 

```bash
[root@docker1 ~]# docker service update --rollback toms
toms
```

​    回滚完毕后再查看当前的 task 情况发现，已经将所有任务的镜像恢复为了 8.5.49 版本。 但需要注意，task name 保持未变，但 task ID 与原来的 task ID 也是不同的，并不是恢复到了 更新之前的 task ID。即编排器新创建了 task，并由分发器重新为其分配了 node。

```bash
[root@docker1 ~]# docker service ps toms
ID            NAME       IMAGE          NODE      DESIRED STATE  CURRENT STATE         
yp61rlvcyij7  toms.1     tomcat:8.5.39  docker4   Running        Running about a minute ago 
57bb2hlm0ns7   \_ toms.1 tomcat:8.5.49  docker4   Shutdown       Shutdown about a minute ago 
w7f6u8ixhmti   \_ toms.1 tomcat:8.5.39  docker4   Shutdown       Shutdown 6 minutes ago     
sc8r40b5gpyh  toms.2     tomcat:8.5.39  docker4   Running        Running about a minute ago 
zv7vnyxz1ihk   \_ toms.2 tomcat:8.5.49  docker4   Shutdown       Shutdown about a minute ago 
ut4i40chkhvl   \_ toms.2 tomcat:8.5.39  docker4   Shutdown       Shutdown 6 minutes ago     
ue2j7je5f0a2  toms.3     tomcat:8.5.39  docker3   Running        Running 2 minutes ago       
c7mgc2vxzq61   \_ toms.3 tomcat:8.5.49  docker1   Shutdown       Shutdown 2 minutes ago     
vaztptbfv2nl   \_ toms.3 tomcat:8.5.39  docker1   Shutdown       Shutdown 6 minutes ago
...
```

## 3.9.service 全局部署模式 

​    根据 task 数量与节点数量的关系，常见的 service 部署模式有两种：replicated 模式与 global 模式。前面创建的 service 是 replicated 模式的，下面来创建 global 模式的 service。 

```bash
[root@docker1 ~]# docker service ls
ID             NAME      MODE         REPLICAS   IMAGE           PORTS
xepenxqp6iq4   toms      replicated   5/5        tomcat:8.5.49   *:9000->8080/tcp
```

**3.9.1.环境变更** 

​    为了后面的演示效果，让 swarm 集群的节点变为 4 个。这里先使 docker5 退群。

```bash
[root@docker5 ~]# docker swarm leave
Node left the swarm.
```

​    此时 docker5 的节点状态变为了 Down。 

```bash
[root@docker1 ~]# docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
08hv8a5lzmwsv65vb4u5gdv6r *   docker1    Ready     Active         Leader           24.0.6
bmbk2y45qjvpkfb0fxvglxn6e     docker2    Ready     Active         Reachable        24.0.6
st541p0oc2oe18zryc7oskldf     docker3    Ready     Active         Reachable        24.0.6
uq7jq3dnpq2qni74shakxomwh     docker4    Ready     Active                          24.0.6
pvzynnh5r30ujr76w7fvt7fno     docker5    Down      Active                          24.0.6
```

​    将此节点再从 swarm 集群中删除。 

```bash
[root@docker1 ~]# docker node rm docker5
docker5
```

​    现在 docker5 节点才彻底被删除。

```bash
[root@docker1 ~]# docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
08hv8a5lzmwsv65vb4u5gdv6r *   docker1    Ready     Active         Leader           24.0.6
bmbk2y45qjvpkfb0fxvglxn6e     docker2    Ready     Active         Reachable        24.0.6
st541p0oc2oe18zryc7oskldf     docker3    Ready     Active         Reachable        24.0.6
uq7jq3dnpq2qni74shakxomwh     docker4    Ready     Active                          24.0.6
```

**3.9.2.创建 service** 

​    在 docker service create 命令中通过--mode 选项可以指定要使用的 service 部署模式，默认为 replicated 模式。

```bash
[root@docker1 ~]# docker service create --name toms --mode global -p 9000:8080 tomcat:8.5.49
gc0o2lo6rapfuf3nlgxsdvwgv
```

​    该模式会在每个节点上分配一个 task。 

```bash
[root@docker1 ~]# docker service ps toms
ID             NAME           IMAGE           NODE     DESIRED STATE   CURRENT STATE
vctic1deuorj   toms.08hv...   tomcat:8.5.49   docker1  Running         Running 2 minutes ago 
j1rn10q1oxv5   toms.bmbk...   tomcat:8.5.49   docker2  Running         Running 2 minutes ago 
71lk606ibnv7   toms.st54...   tomcat:8.5.49   docker3  Running         Running 2 minutes ago 
sblc0g25ru23   toms.uq7j...   tomcat:8.5.49   docker4  Running         Running 2 minutes ago
```

**3.9.3.task 伸缩** 

​    对于 global 模式来说，若要实现对 service 的 task 数量的变更，必须通过改变该 servicve 所依附的 swarm 集群的节点数量来改变。节点增加，则 task 会自动增加；节点减少，则 task 会自动减少。

​    下面要在这个 4 节点的 swarm 集群中增加一个节点，以使 toms 服务的 task 也增一。

​    首先在 manager 节点获取新增一个节点的 token。

```bash
[root@docker1 ~]# docker swarm join-token worker
To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-2hfmwci7bixe8e29hrt4l0ka8syp22jrns6izogm7bfjjpazmy-55a6ejvrz45y9jjv46p0u64mt 192.168.110.101:2377
```

​    在 docker5 上运行加入命令，完成 swarm 的入群。 

```bash
[root@docker5 ~]# docker swarm join --token SWMTKN-1-2hfmwci7bixe8e29hrt4l0ka8syp22jrns6izogm7bfjjpazmy-55a6ejvrz45y9jjv46p0u64mt 192.168.110.101:2377
```

​    此时查看 toms 服务的 task 详情，发现已经自动增加了一个 task。 

```bash
[root@docker1 ~]# docker service ps toms
ID             NAME           IMAGE           NODE     DESIRED STATE  CURRENT STATE
vctic1deuorj   toms.08hv...   tomcat:8.5.49   docker1  Running        Running 8 minutes ago
j1rn10q1oxv5   toms.bmbk...   tomcat:8.5.49   docker2  Running        Running 8 minutes ago
ramt4d4ve3tl   toms.lwu3...   tomcat:8.5.49   docker5  Running        Running 51 seconds ago
71lk606ibnv7   toms.st54...   tomcat:8.5.49   docker3  Running        Running 8 minutes ago
sblc0g25ru23   toms.uq7j...   tomcat:8.5.49   docker4  Running        Running 8 minutes ago
```

## 3.10.overlay 网络 

**3.10.1.测试环境 1 搭建** 

**（1） 暂停分配 task** 

```bash
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
08hv8a5lzmwsv65vb4u5gdv6r *   docker1    Ready     Active         Leader           24.0.6
bmbk2y45qjvpkfb0fxvglxn6e     docker2    Ready     Active         Reachable        24.0.6
st541p0oc2oe18zryc7oskldf     docker3    Ready     Active         Reachable        24.0.6
uq7jq3dnpq2qni74shakxomwh     docker4    Ready     Active                          24.0.6
lwu3siwpr0ueuhmnmhhwohxv2     docker5    Ready     Active                          24.0.6
```

​    现让 docker2 主机暂停分配 task。

```bash
[root@docker1 ~]# docker node update --availability pause docker2
docker2
[root@docker1 ~]# docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
08hv8a5lzmwsv65vb4u5gdv6r *   docker1    Ready     Active         Leader           24.0.6
bmbk2y45qjvpkfb0fxvglxn6e     docker2    Ready     Pause          Reachable        24.0.6
st541p0oc2oe18zryc7oskldf     docker3    Ready     Active         Reachable        24.0.6
uq7jq3dnpq2qni74shakxomwh     docker4    Ready     Active                          24.0.6
lwu3siwpr0ueuhmnmhhwohxv2     docker5    Ready     Active                          24.0.6
```

**（2） 创建 service** 

​    现启动一个 service，包含 10 个 task。 

```bash
[root@docker1 ~]# docker service create --name toms --replicas 10 -p 9000:8080 tomcat:8.5.49
```

​    当前 swarm 集群共有 5 个节点，10 个 task 被分配到了 4 个可用节点上，其中除了被暂 停的 docker2 节点上是没有分配 task 外，其余节点都分配了多个 task。

```bash
[root@docker1 ~]# docker service ps toms
ID             NAME      IMAGE           NODE      DESIRED STATE   CURRENT STATE            
vd2ev81tpen1   toms.1    tomcat:8.5.49   docker3   Running         Running 30 seconds ago   
njinzq07rnvf   toms.2    tomcat:8.5.49   docker5   Running         Running 34 seconds ago   
rxpbd35jqwi2   toms.3    tomcat:8.5.49   docker4   Running         Running 36 seconds ago   
vcbeetuy4d2x   toms.4    tomcat:8.5.49   docker5   Running         Running 33 seconds ago   
0wpad65dcokg   toms.5    tomcat:8.5.49   docker3   Running         Running 33 seconds ago   
gs0o79i083t1   toms.6    tomcat:8.5.49   docker4   Running         Running 34 seconds ago   
sprk0h0575p1   toms.7    tomcat:8.5.49   docker1   Running         Running 34 seconds ago   
u8699tx8fmie   toms.8    tomcat:8.5.49   docker1   Running         Running 33 seconds ago   
sftffs0k32u8   toms.9    tomcat:8.5.49   docker5   Running         Running 35 seconds ago   
p78xwb5554u2   toms.10   tomcat:8.5.49   docker3   Running         Running 31 seconds ago   
```

**3.10.2.overlay 网络概述** 

**（1） overlay 网络简介** 

​    overlay 网络，也称为重叠网络或覆盖网络，是一种构建于 underlay 网络之上的逻辑虚 拟网络。即在物理网络的基础上，通过节点间的单播隧道机制将主机两两相连形成的一种虚 拟的、独立的网络。

​    Docker Swarm 集群中的 overlay 网络主要是通过 iptables、ipvs、vxlan 等技术实现的、基 于其本身通信需求的网络模型。 

**（2） overlay 网络模型** 

​    这里要说的 overlay 网络模型，确切地说，是 Docker Swarm 集群的 overlay 网络模型。

<img src="images\image-20231103161250697.png" alt="image-20231103161250697" style="zoom:80%;" />

​    Docker Swarm 集群的 overlay 网络模型在创建时，会创建出两个网络：docker_gwbidge 网络与 ingress 网络。这就是典型的 overlay 网络——在宿主机的物理网络之上又创建出新的 网络。同时还创建出了 docker_gwbidge 网关与 br0 网关，及 ingress-sbox 容器。

​    当请求到达后会首先经由 docker_gwbidge 网关跳转到 ingress-sbox 容器，在其中具有当 前整个service的所有容器IP，在其中通过轮询负载均衡方式选择一个容器IP作为目标地址， 然后再跳转到 br0 网关。在 br0 网关中会根据目标地址所在主机进行判断。若目标地址为本 地容器 IP，则直接将请求转发给该容器处理即可。若目标地址非本地容器 IP，则会将请求经 由 vxlan 接口，通过 vxlan 隧道技术将请求转发给目标地址容器。 

**3.10.3.docker_gwbridg 网络基础信息** 

​    在详细分析 overlay 网络模型的通信原理之前，首先来了解一下 docker swarm 的 overlay 网络的基础信息。 

**（1） 查看 docker_gwbridge 网络详情** 

​    docker swarm 集群的 overlay 网络模型在创建时，会自动创建两个网络：docker_gwbridge 网络与 ingress 网络。

```bash
[root@docker1 ~]# docker network ls
NETWORK ID     NAME              DRIVER    SCOPE
10a7a17daf35   bridge            bridge    local
f28f9854508e   docker_gwbridge   bridge    local
88ab902df959   host              host      local
tlwuvdglqyaf   ingress           overlay   swarm
```

​    查看 docker_gwbridge 网络详情可以看到，docker_gwbridge 网络包含的子网为 172.18.0.0/16，其网关为 172.18.0.1。那么，这个网关是谁呢？ 

```bash
[root@docker1 ~]# docker network inspect docker_gwbridge
[
    {
        "Name": "docker_gwbridge",
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
...
```

​    同时还看到，该网络中包含了 3 个容器。其中 2 个为 service 的 task 容器，另一个的容 器 ID 为 ingress-sbox。

```bash
"Containers": {
    "0ec759c8482b58673f819bb848010df68beaf4af1f83286aa2ac72996566299d": {
        "Name": "gateway_eb8e6473f176",
        "EndpointID": "32f370408ec6413a6cd5eb1c795d7a6130784fdf388e6c1b5ac870a0548d68c6",
        "MacAddress": "02:42:ac:12:00:03",
        "IPv4Address": "172.18.0.3/16",
        "IPv6Address": ""
    },
    "2baae470e2a236d8049ea486aab516549bc0fe5c5d277a0bb690865b8635a8e0": {
        "Name": "gateway_ca3b5c946dd9",
        "EndpointID": "c0159e01ea02f7ffe262bef1d436080ad8d2afe3f1241f76bb970b9f977660a2",
        "MacAddress": "02:42:ac:12:00:04",
        "IPv4Address": "172.18.0.4/16",
        "IPv6Address": ""
    },
    "ingress-sbox": {
        "Name": "gateway_ingress-sbox",
        "EndpointID": "665d7bcc721367ff15bc918544c231e12f9dbd1bf1a5e55367b3656f916dc892",
        "MacAddress": "02:42:ac:12:00:02",
        "IPv4Address": "172.18.0.2/16",
        "IPv6Address": ""
    }
```

**（2） ingress-sbox 容器** 

```bash
[root@docker1 ~]# docker ps -a
CONTAINER ID   IMAGE           COMMAND             CREATED         STATUS         PORTS     
2baae470e2a2   tomcat:8.5.49   "catalina.sh run"   7 minutes ago   Up 7 minutes   8080/tcp   
0ec759c8482b   tomcat:8.5.49   "catalina.sh run"   7 minutes ago   Up 7 minutes   8080/tcp   
```

​    通过 docker ps –a 命令查看当前主机中的所有容器，发现并没有 ingress-sbox 容器。为 什么？因为 docker ps 命令的本质是 docker process status，查看的是当前主机中真实存在的 容器进程的状态。而 ingress-sbox 容器是由 overlay 网络虚拟出的，并不是真实存在的进程， 所以通过 docker ps 命令是查看不到的。

​    从 docker_gwbridge 的网络详情中可以看到，其中 2 个为 service 的 task 容器，其 ID 由 64 位 16 进制数构成，而 ingress-sbox 容器的 ID 就是 ingress-sbox，与其它 2 个容器的 ID 构 成方式完全不同。 

**（3） docker_gwbridge 网关** 

​    docker_gwbridge 的网络详情中的网关 172.18.0.1 是谁呢？ 

[root@docker1 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:6f:0c:ce brd ff:ff:ff:ff:ff:ff
    inet 192.168.110.101/24 brd 192.168.110.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::7ff6:9e0f:483c:8c10/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
5: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:35:7e:05:84 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
<font color=red>6: docker_gwbridge</font>: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:d2:c3:60:45 brd ff:ff:ff:ff:ff:ff
    <font color=red>inet 172.18.0.1/16</font> brd 172.18.255.255 scope global docker_gwbridge
       valid_lft forever preferred_lft forever
    inet6 fe80::42:d2ff:fec3:6045/64 scope link 
       valid_lft forever preferred_lft forever
<font color=GreenYellow>11: veth4746a74@if10:</font> <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master <font color=red>docker_gwbridge</font> state UP group default 
    link/ether 46:fb:b5:a3:5c:2c brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::44fb:b5ff:fea3:5c2c/64 scope link 
       valid_lft forever preferred_lft forever
<font color=cyan>93: veth37c0d94@if92</font>: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master <font color=red>docker_gwbridge</font> state UP group default 
    link/ether 66:11:3c:52:47:b8 brd ff:ff:ff:ff:ff:ff link-netnsid 3
    inet6 fe80::6411:3cff:fe52:47b8/64 scope link 
       valid_lft forever preferred_lft forever
<font color=cyan>95: vethcb06524@if94:</font> <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master <font color=red>docker_gwbridge</font> state UP group default 
    link/ether a6:cf:8b:3a:77:c7 brd ff:ff:ff:ff:ff:ff link-netnsid 2
    inet6 fe80::a4cf:8bff:fe3a:77c7/64 scope link 
       valid_lft forever preferred_lft forever

​    在宿主机中通过 ip a 命令查看宿主机的网络接口，可以看到 docker_gwbridge 接口的 IP为 172.18.0.1。即 docker_gwbridge 网络中具有一个与网络名称同名的网关。同时还看到，下 面的 3 个接口全部都是连接在 docker_gwbridge 上的。 

**（4） 查看 task 容器的接口** 

查看 docker_gwbridge 网络的 task 容器的接口情况，可以看到这些容器中正好有接口与 docker_gwbridge 网关中的相应接口构成 veth paire。

[root@docker1 ~]# docker ps 
CONTAINER ID   IMAGE           COMMAND             CREATED          STATUS          PORTS      NAMES
<font color=red>2baae470e2a2</font>   tomcat:8.5.49   "catalina.sh run"   16 minutes ago   Up 16 minutes   8080/tcp   toms.7.sprk0h0575p1ni0id41fybte4
0ec759c8482b   tomcat:8.5.49   "catalina.sh run"   16 minutes ago   Up 16 minutes   8080/tcp   toms.8.u8699tx8fmientcmv3jcl7vi8

[root@docker1 ~]# docker exec -it  <font color=red>2baae470e2a2</font> ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
90: eth0@if91: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default 
    link/ether 02:42:0a:00:00:70 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.112/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
 <font color=cyan>94: eth1@if95:</font> <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:12:00:04 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet  <font color=red>172.18.0.4/16</font> brd 172.18.255.255 scope global eth1
       valid_lft forever preferred_lft forever

----

[root@docker1 ~]# docker ps
CONTAINER ID   IMAGE           COMMAND             CREATED          STATUS          PORTS      NAMES
<font color=red>2baae470e2a2</font>   tomcat:8.5.49   "catalina.sh run"   21 minutes ago   Up 20 minutes   8080/tcp   toms.7.sprk0h0575p1ni0id41fybte4
0ec759c8482b   tomcat:8.5.49   "catalina.sh run"   21 minutes ago   Up 20 minutes   8080/tcp   toms.8.u8699tx8fmientcmv3jcl7vi8

[root@docker1 ~]# docker exec -it <font color=red>0ec759c8482b</font> ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
88: eth0@if89: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default 
    link/ether 02:42:0a:00:00:69 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.105/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
<font color=cyan>92: eth1@if93:</font> <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:12:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet <font color=red>172.18.0.3/16</font> brd 172.18.255.255 scope global eth1
       valid_lft forever preferred_lft forever

----

**（5） 查看 ingress-sbox 容器的接口** 

​    如何查看docker_gwbridge网络的ingress-sbox容器的接口情况呢？每个容器都具有一个 独立的网络命名空间，而每个 docker 主机中的网络命名空间，都是以文件的形式保存在目 录/var/run/docker/netns 中。

```bash
[root@docker1 ~]# ll /var/run/docker/netns/
总用量 0
-r--r--r-- 1 root root 0 11月  1 10:04 1-tlwuvdglqy
-r--r--r-- 1 root root 0 11月  2 11:54 ca3b5c946dd9
-r--r--r-- 1 root root 0 11月  2 11:54 eb8e6473f176
-r--r--r-- 1 root root 0 11月  1 10:04 ingress_sbox
```

​    其中 ingress_sbox 就是容器 ingress-sbox 的网络命名空间。通过 nsenter 命令可进入该命 名空间并查看其接口情况。可以看到该命名空间中正好也存在接口与 docker_gwbridge 网关 中的相应接口构成 veth paire。

----

[root@docker1 ~]# <font color=red>nsenter --net=/var/run/docker/netns/ingress_sbox ip a</font>
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
8: eth0@if9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default 
    link/ether 02:42:0a:00:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.2/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet 10.0.0.104/32 scope global eth0
       valid_lft forever preferred_lft forever
<font color=GreenYellow>10: eth1@if11:</font> <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet 172.18.0.2/16 brd 172.18.255.255 scope global eth1
       valid_lft forever preferred_lft forever

**3.10.4.ingress 网络基础信息** 

**（1） 查看 ingress 网络详情** 

​    overlay 网络除了创建了 docker_gwbridge 网络外，还创建了一个 ingress 网络。 

```bash
[root@docker1 ~]# docker network ls
NETWORK ID     NAME              DRIVER    SCOPE
10a7a17daf35   bridge            bridge    local
f28f9854508e   docker_gwbridge   bridge    local
88ab902df959   host              host      local
tlwuvdglqyaf   ingress           overlay   swarm
aaf81b81f43e   none              null      local
```

​    查看 ingress 网络详情可以看到，ingress 网络包含的子网为 10.0.0.0/24，其网关为 10.0.0.1。 那么，这个网关是谁呢？

```bash
[root@docker1 ~]# docker network inspect ingress
...
                {
                    "Subnet": "10.0.0.0/24",
                    "Gateway": "10.0.0.1"
                }
```

​    同时还看到，该网络中也包含了 3 个容器，这 3 个容器与 docker_gwbridge 网络中的 3 个容器是相同的容器，虽然 Name 不同，IP 不同，但容器 ID 相同。说明这 3 个容器都同时 连接在 2 个网络中。

```bash
"Containers": {
        "0ec759c8482b58673f819bb848010df68beaf4af1f83286aa2ac72996566299d": {
                "Name": "toms.8.u8699tx8fmientcmv3jcl7vi8",
                "EndpointID": "3c97ece50d08a7e04e5913cea1a781c50120575767d2a8d8431a95bc79e72412",
                "MacAddress": "02:42:0a:00:00:69",
                "IPv4Address": "10.0.0.105/24",
                "IPv6Address": ""
            },
        "2baae470e2a236d8049ea486aab516549bc0fe5c5d277a0bb690865b8635a8e0": {
                "Name": "toms.7.sprk0h0575p1ni0id41fybte4",
                "EndpointID": "ea183637b07c9c159ea23834f9257a440fe149d9aa859e04866b935665107a52",
                "MacAddress": "02:42:0a:00:00:70",
                "IPv4Address": "10.0.0.112/24",
                "IPv6Address": ""
            },
        "ingress-sbox": {
                "Name": "ingress-endpoint",
                "EndpointID": "91422c5064570c656fc45977338390f601b0afef75160f130aeb71f625a3a403",
                "MacAddress": "02:42:0a:00:00:02",
                "IPv4Address": "10.0.0.2/24",
                "IPv6Address": ""
            }
```

**（2） br0 网关** 

​    10.0.0.1 网关是谁呢？ 

​    每个容器都具有一个独立的网络空间，而每个 docker 主机中的网络命名空间，都是以 文件的形式保存在/var/run/docker/netns 目录中。查看当前主机的网络空间： 

```bash
[root@docker1 ~]# ll /var/run/docker/netns/
总用量 0
-r--r--r-- 1 root root 0 11月  1 10:04 1-tlwuvdglqy
-r--r--r-- 1 root root 0 11月  2 11:54 ca3b5c946dd9
-r--r--r-- 1 root root 0 11月  2 11:54 eb8e6473f176
-r--r--r-- 1 root root 0 11月  1 10:04 ingress_sbox
```

​    查看/var/run/docker/netns 目录中的命名空间发现，其包含的 4 个命名空间中，有 2 个 命名空间是 2 个 task 容器的，它们的名称由 12 位长度的 16 进制数构成；ingress_sbox 是 ingress-sbox 容器的命名空间。那么，1-tlwuvdglqy 命名空间是谁呢？进入该命名空间，查看 其接口信息。

[root@docker1 ~]# <font color=red>nsenter --net=/var/run/docker/netns/1-tlwuvdglqy ip a</font>
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: <font color=red>br0:</font> <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default 
    link/ether 26:db:bd:b1:cf:91 brd ff:ff:ff:ff:ff:ff
    inet <font color=red>10.0.0.1/24</font> brd 10.0.0.255 scope global br0
       valid_lft forever preferred_lft forever
7: vxlan0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master <font color=red>br0</font> state UNKNOWN group default 
    link/ether 6e:54:2a:dc:a2:01 brd ff:ff:ff:ff:ff:ff link-netnsid 0
<font color=GreenYellow>9: veth0@if8:</font> <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master <font color=red>br0</font> state UP group default 
    link/ether 26:db:bd:b1:cf:91 brd ff:ff:ff:ff:ff:ff link-netnsid 1
<font color=cyan>89: veth20@if88:</font> <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master <font color=red>br0</font> state UP group default 
    link/ether 86:0d:fc:ae:42:64 brd ff:ff:ff:ff:ff:ff link-netnsid 3
<font color=cyan>91: veth21@if90: </font><BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master <font color=red>br0</font> state UP group default 
    link/ether f2:1d:8f:9b:b9:27 brd ff:ff:ff:ff:ff:ff link-netnsid 2

----

​    可以看到 2 号接口 br0 的 IP 为 10.0.0.1，即 ingress 网络的网关为 1-tlwuvdglqy 命名空间 中的 br0。同时还看到，br0 上还连接着 4 个接口，说明 br0 就是一个网关。那么，都是谁 连接在这 4 个接口上呢？ 

**（3） 查看 task 容器的接口** 

查看 ingress 网络的 task 容器的接口情况，可以看到这些容器中正好有接口与 br0 网关 中的相应接口构成 veth paire。

[root@docker1 ~]# docker ps
CONTAINER ID   IMAGE           COMMAND             CREATED          STATUS          PORTS      NAMES
<font color=red>2baae470e2a2</font>   tomcat:8.5.49   "catalina.sh run"   54 minutes ago   Up 54 minutes   8080/tcp   toms.7.sprk0h0575p1ni0id41fybte4
0ec759c8482b   tomcat:8.5.49   "catalina.sh run"   54 minutes ago   Up 54 minutes   8080/tcp   toms.8.u8699tx8fmientcmv3jcl7vi8

[root@docker1 ~]# docker exec -it <font color=red>2baae470e2a2</font> ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
<font color=cyan>90: eth0@if91:</font> <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default 
    link/ether 02:42:0a:00:00:70 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.112/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
94: eth1@if95: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:12:00:04 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet 172.18.0.4/16 brd 172.18.255.255 scope global eth1
       valid_lft forever preferred_lft forever

----

[root@docker1 ~]# docker ps
CONTAINER ID   IMAGE           COMMAND             CREATED          STATUS          PORTS      NAMES
2baae470e2a2   tomcat:8.5.49   "catalina.sh run"   57 minutes ago   Up 57 minutes   8080/tcp   toms.7.sprk0h0575p1ni0id41fybte4
<font color=red>0ec759c8482b</font>   tomcat:8.5.49   "catalina.sh run"   57 minutes ago   Up 57 minutes   8080/tcp   toms.8.u8699tx8fmientcmv3jcl7vi8

[root@docker1 ~]# docker exec -it <font color=red>0ec759c8482b</font> ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
<font color=cyan>88: eth0@if89:</font> <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default 
    link/ether 02:42:0a:00:00:69 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.105/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
92: eth1@if93: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:12:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet 172.18.0.3/16 brd 172.18.255.255 scope global eth1
       valid_lft forever preferred_lft forever

----

**（4） 查看 ingress-sbox 容器的接口** 

查看 ingress-sbox 容器的命名空间 ingress_sbox 的接口情况，可以看到该命名空间中正 好也存在接口与 br0 网关中的相应接口构成 veth paire。

[root@docker1 ~]# <font color=red>nsenter --net=/var/run/docker/netns/ingress_sbox ip a</font>
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
<font color=GreenYellow>8: eth0@if9:</font> <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default 
    link/ether 02:42:0a:00:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.2/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet 10.0.0.104/32 scope global eth0
       valid_lft forever preferred_lft forever
10: eth1@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet 172.18.0.2/16 brd 172.18.255.255 scope global eth1
       valid_lft forever preferred_lft forever

**3.10.5.宿主机的 NAT 过程** 

**（1） 查看宿主机路由** 

​    用户提交的192.168.110.101:9000请求会首先被192.168.110.101主机的哪个接口接收并 处理呢？通过命令 ip route 可以查看当前网络命名空间中的静态路由信息。 

----

[root@docker1 ~]# ip route
default via 192.168.110.2 dev ens33 proto static metric 100 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 
172.18.0.0/16 dev docker_gwbridge proto kernel scope link src 172.18.0.1 
<font color=orange>192.168.110.0/24 dev ens33</font> proto kernel scope link src 192.168.110.101 metric 100 

----

​    可以看出，所有对 192.168.110.0/24 网络的请求，都需要经过 ens33 接口，而该接口连 接的 IP 为 192.168.110.101。即 ens33 接口会处理该请求。当然，查看该主机的接口情况也 可以看到，ens33 接口地址为 192.168.110.101。 

----

[root@docker1 ~]# ip a
...
2: <font color=orange>ens33: </font><BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:6f:0c:ce brd ff:ff:ff:ff:ff:ff
    inet <font color=orange>192.168.110.101/24</font> brd 192.168.110.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
6: <font color=cyan>docker_gwbridge: </font><BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:d2:c3:60:45 brd ff:ff:ff:ff:ff:ff
    inet <font color=cyan>172.18.0.1/16</font> brd 172.18.255.255 scope global docker_gwbridge
       valid_lft forever preferred_lft forever

----

​    那么 ens33 接口又会将请求转发到哪里呢？这就需要查看宿主机的路由转发表 nat 中的 路由规则了。 

**（2） 查看 ip 转换规则** 

​    首先通过 iptables –nvL –t nat 命令来查看宿主机中网络地址转发表 nat 中的转发规则。 

​    nat 表的主要功能是根据规则进行地址映射、端口映射，以完成地址转换。

```bash
[root@docker1 ~]# iptables -nvL -t nat
...
Chain DOCKER-INGRESS (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0     tcp dpt:9000 to:172.18.0.2:9000
11056  664K RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0           
```

​    DOCKER-INGRESS 路由链路中的 DNAT 映射规则中指出，对于任何源 IP，只要其访问端 口号为 9000，就会将其转换为 172.18.0.2:9000 的请求，即将请求转发到 172.18.0.2。那么请 求是如何到达 172.18.0.2 的呢？ 

**（3） 查看宿主机路由** 

​    通过 ip route 命令查看当前宿主机的静态路由信息。 

----

[root@docker1 ~]# ip route
default via 192.168.110.2 dev ens33 proto static metric 100 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 
<font color=cyan>172.18.0.0/16 dev docker_gwbridge</font> proto kernel scope link src <font color=cyan>172.18.0.1</font> 
192.168.110.0/24 dev ens33 proto kernel scope link src 192.168.110.101 metric 100

----

​    可以看出，所有对 172.18.0.0/16 网络的请求，都需要经过 docker_gwbridge 接口，而该 接口连接的 IP 为 172.18.0.1。即 docker_gwbridge 接口会处理该请求。由一个网络去访问另 一个网络必须要经过该目标网络的网关。经前面的学习知道，docker_gwbridge 正好就是 172.18.0.0/16 网络的网关。

​    也就是说，客户端提交的 192.168.192.101:9000 的请求，经 docker_gwbridge 网关，被 路由到了 IP 为 172.18.0.2 的接口。那么谁的 IP 是 172.18.0.2 呢？经过前面网络基础信息查 看可知，docker_gwbridge 网络中包含 IP 为 172.18.0.2 的 ingress-sbox 容器。

**3.10.6.ingress_sbox 的负载均衡** 

​    客户端请求经宿主机的 NAT 已经成功通过 docker_gwbridge 网关转发到了 172.18.0.2， 即转发到了 ingress-sbox 容器，或者更确切地说，是转发到了 ingress_sbox 命名空间。那么， ingress_sbox 命名空间又会将请求转发到哪里呢？这就需要查看 ingress_sbox 命名空间的 iptables 的 mangle 表与 IPVS 功能了。 

**（1） 查看 ingress_sbox 的 mangle 表** 

​    mangle 表的主要功能是根据规则修改数据包的一些标志位，以便其他规则或程序可以 利用这种标志对数据包进行过滤或路由。

```bash
[root@docker1 ~]# nsenter --net=/var/run/docker/netns/ingress_sbox iptables -nvL -t mangle
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MARK       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:9000 MARK set 0x108
```

​    该路由链中为任意源地址端口为 9000 的请求打了一个 MARK 标记 0x108(十进制264)，该 MARK 标 记将被 IPVS 用于负载均衡。 

**（2） 安装 ipvsadm 命令** 

​    后面我们需要使用该命令查看 IPVS 实现的负载均衡规则，但由于 CentOS 系统中默认没 有安装 ipvsadm 命令，所以需要先 yum 安装。

```bash
[root@docker1 ~]# yum install -y ipvsadm
```

**（3） 查看 ingress_sbox 负载均衡规则** 

```bash
[root@docker1 ~]# nsenter --net=/var/run/docker/netns/ingress_sbox ipvsadm
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
FWM  264 rr
  -> 10.0.0.105:0                 Masq    1      0          0         
  -> 10.0.0.106:0                 Masq    1      0          0         
  -> 10.0.0.107:0                 Masq    1      0          0         
  -> 10.0.0.108:0                 Masq    1      0          0         
  -> 10.0.0.109:0                 Masq    1      0          0         
  -> 10.0.0.110:0                 Masq    1      0          0         
  -> 10.0.0.111:0                 Masq    1      0          0         
  -> 10.0.0.112:0                 Masq    1      0          0         
  -> 10.0.0.113:0                 Masq    1      0          0         
  -> 10.0.0.114:0                 Masq    1      0          0  
```

​    端口为 9000 的请求被打上了一个数值为 264 的 MARK 标记，该标记通过 LVS 的 IPVS 的 负载均衡，将该请求转发到了以上的 10 个 IP 接口，且这 10 个接口的权重 weight 是相同的， 都是 1。这 10 个 IP 接口具有一个共同点，全部来自于 10.0.0.0/24 网络。那么，如何能到达 10.0.0.0/24 网络呢？ 

**（4） 查看命名空间路由** 

​    通过前面的学习可知，若要由一个网络转发到另一个网络，则必须要先到目标网络的网 关。由于目前尚在 172.18.0.0/16 网络，预转发到 10.0.0.0/24 网络，所以必须要先到 10.0.0.0/24 网络的网关 10.0.0.1，即 br0。通过查看 br0 所在命名空间 1-tlwuvdglqy 的静态路由也可看出：

```bash
[root@docker1 ~]# nsenter --net=/var/run/docker/netns/1-tlwuvdglqy ip route
10.0.0.0/24 dev br0 proto kernel scope link src 10.0.0.1 
```

​    但存在的问题是，请求目前尚在 ingress_sbox 命名空间中，怎样才能从 ingress_sbox 命 名空间中出去，然后跳转到 br0 呢？查看 ingress_sbox 命名空间中的静态 IP 路由：

----

[root@docker1 ~]# nsenter --net=/var/run/docker/netns/ingress_sbox ip route
default via 172.18.0.1 dev eth1 
10.0.0.0/24 dev eth0 proto kernel scope link src <font color=GreenYellow>10.0.0.2 </font>
172.18.0.0/16 dev eth1 proto kernel scope link src 172.18.0.2

----

​    可以看出，所有对 10.0.0.0/24 网络的请求，都需要经过 eth0 接口，而该接口连接的 IP 为 10.0.0.2。在 ingress_sbox 命名空间中 eth0 接口就是 8 号接口，其 veth pair 接口就是 br0 中的 9 号接口。所以，ingress_sbox 命名空间中请求经由 8 号接口跳转到了 br0 网关。 

----

[root@docker1 ~]# nsenter --net=/var/run/docker/netns/ingress_sbox ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
<font color=GreenYellow>8: eth0@if9:</font> <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default 
    link/ether 02:42:0a:00:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet <font color=GreenYellow>10.0.0.2/24</font> brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet 10.0.0.104/32 scope global eth0
       valid_lft forever preferred_lft forever
10: eth1@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet 172.18.0.2/16 brd 172.18.255.255 scope global eth1
       valid_lft forever preferred_lft forever

----


**（5） br0 网关的处理** 

​    到达 br0 后，再将请求从 br0 的哪个接口转发出去，是由目标地址决定的，而目标地址 就是 IPVS 负载均衡选择出的 IP。请求到达 br0 后，首先会将目标地址与本地的 task 容器地 址进行比较，若恰好就是当前宿主机中的 task 容器的 IP，那么直接将请求通过相应的接口 将其转发；若不是当前宿主机中的 IP，则会将请求转发到 vxlan0 接口。经过 vxlan0 接口， 可经由 VXLAN 技术将请求通过“网络隧道”发送到目标地址。 

**3.10.7.VXLAN** 

**（1） VXLAN 简介** 

​    VXLAN 是一种隧道技术，可以将不同协议的数据包重新封装后发送。新的包头提供了路由信息，从而使被封装的数据包在隧道的两个端点间通过公共互联网络进行路由。被封装的 数据包在公共互联网络上传递时所经过的逻辑路径称为隧道。一旦到达网络终点，数据将被 解包并转发到最终目的地。 

**（2） 测试环境 2 搭建** 

​    为了能够看清楚请求在不同主机的容器间所进行了通信，及通信过程中所使用的 VXLAN 技术，这里将原来的服务先删除，然后再创建一个新的服务。不过，该服务仅有一个副本。 

首先删除原来的 service。

```bash
[root@docker1 ~]# docker service rm toms
toms
```

 然后在任意主机中创建一个新的 servivce，其仅包含一个副本。这里在 docker3 主机创 建了服务。可以看到，这唯一的副本被分配到了 docker5 主机。 

```bash
[root@docker1 ~]# docker service create --name toms --replicas 1 -p 9000:8080 tomcat:8.5.49
xiwapia845l7867fts2fj0n27
overall progress: 1 out of 1 tasks 
1/1: running   [==================================================>] 
verify: Service converged 
```



**（3） 安装 tcpdump 命令** 

​    这里准备使用 tcpdump 命令对 VXLAN 数据进行监听，但在 centOS7 系统中默认是没有 安装 tcpdump 命令的，所以需要使用 yum 命令先在 docker5 主机安装。

```bash
[root@docker5 ~]# yum -y install tcpdump
```

**（4） docker5 先监听** 

​    无论对哪个主机的该服务进行访问，请求最终都会通过 docker5 主机的 ens33 接口进入， 然后再找到该 task 容器。所以这里要先监听 docker5 的 ens33 接口。 

```bash
[root@docker5 ~]# tcpdump -i ens33 port 4789
```

**（5） docker3 访问** 

在浏览器可以对任意主机提交访问请求。这里是向 docker3 主机发出的访问请求。 

http://docker3:9000

**（6） docker5 查看抓包数据** 

当向 docker3 主机发送了访问请求后，docker5 上就会看到抓取的 VXLAN 数据包。

```bash
[root@docker5 ~]# tcpdump -i ens33 port 4789
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ens33, link-type EN10MB (Ethernet), capture size 262144 bytes
18:17:01.831707 IP 192.168.110.103.35187 > docker5.4789: VXLAN, flags [I] (0x08), vni 4096
IP 10.0.0.6.9770 > 10.0.0.7.cslistener: Flags [S], seq 3827067898, win 64240, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
18:17:01.831746 IP 192.168.110.103.39333 > docker5.4789: VXLAN, flags [I] (0x08), vni 4096
IP 10.0.0.6.9771 > 10.0.0.7.cslistener: Flags [S], seq 4149147090, win 64240, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
18:17:01.831840 IP docker5.56354 > 192.168.110.103.4789: VXLAN, flags [I] (0x08), vni 4096
IP 10.0.0.7.cslistener > 10.0.0.6.9770: Flags [S.], seq 2823547562, ack 3827067899, win 28200, options [mss 1410,nop,nop,sackOK,nop,wscale 7], length 0
18:17:01.831943 IP docker5.45899 > 192.168.110.103.4789: VXLAN, flags [I] (0x08), vni 4096
IP 10.0.0.7.cslistener > 10.0.0.6.9771: Flags [S.], seq 35315279, ack 4149147091, win 28200, options [mss 1410,nop,nop,sackOK,nop,wscale 7], length 0
18:17:01.832558 IP 192.168.110.103.35187 > docker5.4789: VXLAN, flags [I] (0x08), vni 4096
IP 10.0.0.6.9770 > 10.0.0.7.cslistener: Flags [.], ack 1, win 512, length 0
```

# 4.CI/CD之jenkins

## 4.1.平台登录页面 

**4.1.1.GitLab-9999-root** 

<img src="images\image-20231108163427465.png" alt="image-20231108163427465" style="zoom:80%;" />

**4.1.2.Jenkins-8080-zhangsan**

<img src="images\image-20231108163526275.png" alt="image-20231108163526275" style="zoom:80%;" />

**4.1.3.SonarQube-9000-admin** 

<img src="images\image-20231108163602092.png" alt="image-20231108163602092" style="zoom:80%;" />

**4.1.4.harbor-80-admin** 

<img src="images\image-20231108163633825.png" alt="image-20231108163633825" style="zoom:80%;" />

## 4.2.CI/CD 与 DevOps 

**4.2.1.CI/CD 简介** 

<img src="images\image-20231108163737778.png" alt="image-20231108163737778" style="zoom:80%;" />

  CI，Continuous Integration，持续集成。即将持续不断更新的代码经构建、测试后也持 续不断的集成到项目主干分支。

  CD，包含两层含义：Continuous Delivery，持续交付，和 Continuous Deployment，持续 部署。 

- 持续交付：是持续集成的后续步骤，持续频繁地将软件的新版本交付到类生产环境预发， 即交付给测试、产品部门进行集成测试、API 测试等验收，确保交付的产物可直接部署 

- 持续部署：是持续交付的后续步骤，将持续交付的产物部署到生产环境 

**4.2.2.DevOps 简介** 

<img src="images\image-20231108163807816.png" alt="image-20231108163807816" style="zoom:80%;" />

​    百度百科中是这样介绍 DevOps 的： 

​    DevOps（Development 和 Operations 的组合词）是一组过程、方法与系统的统称，用于 促进开发（应用程序/软件工程）、技术运营和质量保障（QA）部门之间的沟通、协作与整合。 

​    它是一种重视“软件开发人员（Dev）”和“IT 运维技术人员（Ops）”之间沟通合作的 文化、运动或惯例。透过自动化“软件交付”和“架构变更”的流程，来使得构建、测试、 发布软件能够更加地快捷、频繁和可靠。 

​    DevOps 是一种思想，是一种管理模式，是一种执行规范与标准。 

**4.2.3.CI/CD 与 DevOps 关系** 

​    CI/CD 是目标，DevOps 为 CI/CD 目标的实现提供了前提与保障。 

## 4.3.系统架构图 

​    最终要搭建出如下图所示架构的系统。 

<img src="images\image-20231108163846670.png" alt="image-20231108163846670" style="zoom:99%;" />

## 4.4.Idea 中 Git 配置 

**4.4.1.Git 简介** 

​    百度百科中是这样介绍 Git 的： 

​    Git（读音为/gɪt/）是一个开源的分布式版本控制系统，可以有效、高速地处理从很小到 非常大的项目版本管理。也是 Linus Torvalds 为了帮助管理 Linux 内核开发而开发的一个开放 源码的版本控制软件。

**4.4.2.Git 的工作流程** 

![image-20231108163950929](images\image-20231108163950929.png)

**4.4.3.Git 的下载与安装** 

从 Git 的官网下载 Git。其官网为：https://git-scm.com 。根据安装向导“下一步”式 安装即可。

<img src="images\image-20231108164026242.png" alt="image-20231108164026242" style="zoom:80%;" />

<img src="images\image-20231108164112427.png" alt="image-20231108164112427" style="zoom:80%;" />

**4.4.4.Idea 中配置 Git**

**git版本：最新版**

<img src="images\image-20231108164144336.png" alt="image-20231108164144336" style="zoom:80%;" />

<img src="images\image-20231108164208088.png" alt="image-20231108164208088" style="zoom:80%;" />

<img src="images\image-20231108164258460.png" alt="image-20231108164258460" style="zoom:80%;" />

## 4.5.GitLab 安装与配置 *

**4.5.1.简介** 192.168.110.121:9999 ccgitlab (4GB ram) root

​    GitLab 是一个源码托管开源工具，其使用 Git 作为代码管理工具，并在此基础上搭建起 来的 Web 服务。GitLab 由乌克兰程序员使用 Ruby 语言开发，后来一部分使用 Go 语言重写。 生产中通常使用 GitLab 搭建私有源码托管平台。 

**4.5.2.GitLab 的安装** 

**（1） 主机要求** 

​    这里要使用 docker 方式来安装 GitLab，所以需要一台安装有 docker 及 docker-compose 的主机，且该主机内存至少 4G。 

**（2） 拉取镜像** 

​    这里使用由 gitlab 官网发布的社区版镜像 **gitlab/gitlab-ce:latest（16.5.1）**。该镜像最好是先拉取到 本地后再使用，因为该镜像比较大。 

```bash
1、配置镜像源
[root@ccgitlab docker]# pwd
/etc/docker
[root@ccgitlab docker]# vim daemon.json
{
  "registry-mirrors": ["https://mirror.baidubce.com"]
}

2、pull
[root@ccgitlab docker]# docker pull gitlab/gitlab-ce
Using default tag: latest
latest: Pulling from gitlab/gitlab-ce
```

**（3） 定义 compose.yml** 

​    由于启动 GitLab 容器时需要设置的内容较多，为了方便，这里使用 docker compose 方 式启动。 在任意目录 mkdir 一个子目录，例如在/usr/local 下新建一个 gitlab 目录。在该目录中新 建 compose.yml 文件。文件内容如下： 

```bash
[root@ccgitlab gitlab]# pwd
/usr/local/gitlab
[root@ccgitlab gitlab]# vim compose.yml

services:
  gitlab:
    image: gitlab/gitlab-ce
    container_name: gitlab
    restart: always
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://192.168.110.121:9999'
        gitlab_rails['gitlab_shell_ssh_port'] = 2222
    ports:
      - 9999:9999
      - 2222:2222
    volumes:
      - ./config:/etc/gitlab
      - ./logs:/var/log/gitlab
      - ./data:/var/opt/gitlab

```

**（4） 启动 gitLab** 

​    使用 docker-compose up -d 命令启动容器服务。不过，其启动过程时间较长些。

```bash
[root@ccgitlab gitlab]# docker compose up -d
[+] Running 2/2
 ✔ Network gitlab_default  Created                                                                            0.0s
 ✔ Container gitlab        Started                                                                            0.1s
```

​    在等待过程中，可以查看其启动日志。 

```bash
[root@ccgitlab gitlab]# docker compose logs -f
```

​    可以看到如下的大量日志。 

```bash
gitlab  |       - create symlink at /opt/gitlab/init/prometheus to /opt/gitlab/embedded/bin/sv
gitlab  |     * file[/opt/gitlab/sv/prometheus/down] action nothing (skipped due to action :nothing)
gitlab  | [2023-11-06T09:00:26+00:00] INFO: template[/opt/gitlab/sv/prometheus/run] sending run action to ruby_block[restart_service] (delayed)
gitlab  |     * ruby_block[restart_service] action run (skipped due to only_if)
```

**4.5.3.GitLab 的密码配置** 

**（1） 浏览器访问** 

​    在浏览器中直接键入 http://192.168.110.121:9999 即可打开登录页面。不过，这个过程 一般需要的时间较长。这里需要登录的用户名与密码。默认的用户名为 root，而默认密码需 要进入容器中查看。

**（2） 查看登录密码** 

​    gitLab 平台的登录用户名默认为 root，初始密码在容器中/etc/gitlab/initial_root_password 文件中。所以需要首先进入容器，然后查看该文件内容。然后再将 root 用户名与复制来的 密码填写到登录页面的相应位置即可登录成功。 

```bash
[root@ccgitlab docker]# docker exec -it gitlab bash
root@75bacd0beb30:/# cat /etc/gitlab/initial_root_password
# WARNING: This value is valid only in the following conditions
#          1. If provided manually (either via `GITLAB_ROOT_PASSWORD` environment variable or via `gitlab_rails['initial_root_password']` setting in `gitlab.rb`, it was provided before database was seeded for the first time (usually, the first reconfigure run).
#          2. Password hasn't been changed manually, either via UI or via command line.
#
#          If the password shown here doesn't work, you must reset the admin password following https://docs.gitlab.com/ee/security/reset_user_password.html#reset-your-root-password.

Password: 4pUj5sGJFyi8tC1589NxGyJvaLLrpJMC/L0Y3ATKCnM=

# NOTE: This file will be automatically deleted in the first reconfigure run after 24 hours.

```

**（3） 修改密码** 

​    登录后，首先要修改初始密码。新密码要求长度不能少于 8 位。为了方便，这里将新密 码设置为 qwe.1234。

 http://192.168.110.121:9999 

<img src="images\image-20231108164355206.png" alt="image-20231108164355206" style="zoom:80%;" />

<img src="images\image-20231108164520712.png" alt="image-20231108164520712" style="zoom:80%;" />

​     将复制来的初始密码粘贴到 Current password 中，要求新密码长度不能低于 8 位。这里 新密码设置为 qwe.1234。

## 4.6.SonarQube 安装与配置 *

**4.6.1简介** 192.168.110.123:9000 ccsonarqube(2GB ram) admin

​    SonarQube 是一个开源的代码扫描与分析平台，用来持续扫描、分析和评测项目源代码 的质量与安全。 通过 SonarQube 可以检测项目中代码量、安全隐患、编写规范隐患、重复 度、复杂度、代码增量、测试覆盖率等多个维度，并通过 SonarQube web UI 展示出来。

​    SonarQube 支持 30+种编程语言代码的扫描与分析，并能够方便的与代码 IDE、CI/CD 平 台完美集成。 

​    SonarQube 的官网地址：https://www.sonarsource.com/ 

**4.6.2.主机要求** 

​    这里要使用docker方式来安装，所以需要一台安装有docker及docker-compose的主机。 

**4.6.3.安装与配置** 

**（1） 下载两个镜像** 

​    由于 SonarQube 需要 Postgres 数据库的支持，所以安装 SonarQube 之前需要先安装 Postgres 数据库。所以需要下载 Postgres 与 SonarQube 两个镜像。 

**sonarqube:9.9-community**

```bash
[root@ccsonarqube sonarqube]# docker images
REPOSITORY   TAG             IMAGE ID       CREATED        SIZE
sonarqube    9.9-community   190725cfc485   7 days ago     617MB
postgres     13.10           c562f2f06bc5   6 months ago   374MB
```

**（2） 定义 compose.yml** 

​    由于需要启动两个容器，所以这里使用 docker-compose 方式。 

​    在/usr/local 下 mkdir 一个 sonar 目录，在其中定义 compose.yml 文件。

```yml
services:
  postgres:
    image: postgres:16
    container_name: pgdb
    restart: always
    ports:
      - 5342:5432
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar

  sonarqube:
    image: sonarqube:9.9-community
    container_name: sonarqb
    restart: always
    depends_on:
      - postgres
    ports:
      - 9000:9000
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://pgdb:5432/sonar
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: sonar
```



**（3） 修改虚拟内存大小** 

​    在/etc/sysctl.conf 文件中指定 vm.max_map_count 虚拟内存大小。 

```bash
# sysctl settings are defined through files in
# /usr/lib/sysctl.d/, /run/sysctl.d/, and /etc/sysctl.d/.
#
# Vendors settings live in /usr/lib/sysctl.d/.
# To override a whole file, create a new file with the same in
# /etc/sysctl.d/ and put new settings there. To override
# only specific settings, add a file with a lexically later
# name in /etc/sysctl.d/ and put new settings there.
#
# For more information, see sysctl.conf(5) and sysctl.d(5).

vm.max_map_count=262144
```

​    修改保存后再运行 sysctl –p 命令使 Linux 内核加载文件中的配置。 

```bash
[root@ccsonarqube sonarqube]# sysctl -p
vm.max_map_count = 262144
```

**（4） 启动 SonarQube** 

​    通过 docker-compose up –d 命令启动容器，并 docker ps 查看是否启动成功。

```bash
[root@ccsonarqube sonarqube]# docker compose up -d
```

**（5） 登录 SonarQube** 

​    在浏览器键入 SonarQube 服务器的 IP 与端口号 9000，即可打开登录页面。默认用户名 与密码都是 admin。

​    Log in 后即可跳转到更新密码页面。这里更新密码为 qwe.1234。 

​    Update 后即可看到首页。

**（6） 安装汉化插件** 

<img src="images\image-20231108164715058.png" alt="image-20231108164715058" style="zoom:80%;" />

<img src="images\image-20231108164817451.png" alt="image-20231108164817451" style="zoom:80%;" />

​    在 Maketplace 中键入关键字 Chinese 后即可找到要安装的汉化插件，点击 I understand  the risk（我了解风险）后即可看到 Install 按钮，点击安装。 

<img src="images\image-20231108164857945.png" alt="image-20231108164857945" style="zoom:80%;" />

​    安装成功后，在页面上部就可看到 Restart Server 的提示，让重启 SonarQube。

<img src="images\image-20231108164925362.png" alt="image-20231108164925362" style="zoom:80%;" />

​    重启后，页面会自动跳转到具有中文的登录页面。登录进入后，页面已经变为了中文。

## 4.7.harbor 安装与配置 *

**4.7.1.Harbor 安装系统要求** 192.168.110.124:80 ccharbor(4GB ram) admin

​    Harbor 要安装的主机需要满足硬件与软件上的要求。 

**（1） 硬件要求** 

|硬件资源|最小要求|推荐要求|
| ---- | ---- | ---- |
|CPU |2CPU|4CPU|
|内存 |4GB|8GB|
|硬盘 |40GB|160GB|

**（2） 软件要求** 

| 软件资源       | 版本要求           | 作用                                                 |
| -------------- | ------------------ | ---------------------------------------------------- |
| Docker CE 引擎 | 17.06.0 或更高版本 | Harbor 是以容器形式在运行，需要 Docker 引擎          |
| Docker Compose | 1.18.0 或更高版本  | Harbor 是 10 个容器在运行，通过 Docker  Compose 编排 |
| OpenSSL        | 最新版             | 生成数字证书，以支持 HTTPS                           |

**4.7.2.安装 Harbor** 

**（1） 下载安装包** 

在官网复制 Latest 最新版的离线安装包的下载链接地址，在 Linux 系统中通过 wget 命令 下载，将其下载到某目录中。 

**harbor-offline-installer-v2.7.3.tgz**

```bash
[root@ccharbor local]# wget https://github.com/goharbor/harbor/releases/download/v2.7.3/harbor-offline-installer-v2.7.3.tgz
```

```bash
[root@ccharbor local]# ls
bin  etc  games  harbor-offline-installer-v2.7.3.tgz  include  lib  lib64  libexec  sbin  share  src
[root@ccharbor local]# pwd
/usr/local
```

**（2） 解压安装包** 

将下载好的包解压到某目录中。解压后其就是一个独立的目录 harbor。  

```bash
[root@ccharbor local]# tar -zxvf harbor-offline-installer-v2.7.3.tgz
```

```bash
[root@ccharbor local]# ls
bin  etc  games  harbor  harbor-offline-installer-v2.7.3.tgz  include  lib  lib64  libexec  sbin  share  src
```

**（3） 修改 harbor.yml** 

复制一份 harbor 解压包中的 harbor.yml.tmpl，并重命名为 harbor.yml。 

```bash
[root@ccharbor harbor]# cp harbor.yml.tmpl harbor.yml
[root@ccharbor harbor]# ls
common.sh  harbor.v2.7.3.tar.gz  harbor.yml  harbor.yml.tmpl  install.sh  LICENSE  prepare
[root@ccharbor harbor]# vim harbor.yml

# Configuration file of Harbor

# The IP address or hostname to access admin UI and registry service.
# DO NOT use localhost or 127.0.0.1, because Harbor needs to be accessed by external clients.
hostname: 192.168.110.124

# http related config
http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  port: 80

# https related config
# https:
  # https port for harbor, default is 443
  # port: 443
  # The path of cert and key files for nginx
  # certificate: /your/certificate/path
  # private_key: /your/private/key/path

# # Uncomment following will enable tls communication between all harbor components
# internal_tls:
#   # set enabled to true means internal tls is enabled
#   enabled: true
#   # put your cert and key files on dir
#   dir: /etc/harbor/tls/internal

# Uncomment external_url if you want to enable external proxy
# And when it enabled the hostname will no longer used
# external_url: https://reg.mydomain.com:8433

# The initial password of Harbor admin
# It only works in first time to install harbor
# Remember Change the admin password from UI after launching Harbor.
harbor_admin_password: qwe.1234
```

**（4） 运行 prepare** 

运行 harbor 解压目录中的 prepare 命令。该命令会先拉取 prepare 镜像，然后再生成很 多的其后期要用到的配置文件。

```bash
[root@ccharbor harbor]# ./prepare
prepare base dir is set to /usr/local/harbor
Unable to find image 'goharbor/prepare:v2.7.3' locally
v2.7.3: Pulling from goharbor/prepare
```

**（5） 运行 install.sh** 

运行 harbor 解压目录中的 install.sh 命令，其会自动完成五步的安装过程，并在最终启 动很多的容器。这些容器本质上就是通过 docker-compose 进行编排管理的。

```bash
[root@ccharbor harbor]# ./install.sh
```

```bash
[+] Running 10/10
 ✔ Network harbor_harbor        Created                                                                        0.0s
 ✔ Container harbor-log         Started                                                                        0.0s
 ✔ Container harbor-db          Started                                                                        0.0s
 ✔ Container harbor-portal      Started                                                                        0.0s
 ✔ Container registryctl        Started                                                                        0.0s
 ✔ Container registry           Started                                                                        0.0s
 ✔ Container redis              Started                                                                        0.0s
 ✔ Container harbor-core        Started                                                                        0.0s
 ✔ Container nginx              Started                                                                        0.0s
 ✔ Container harbor-jobservice  Started                                                                        0.0s
✔ ----Harbor has been installed and started successfully.----
```

 **（6） 新建仓库** 

​    在浏览器地址栏中输入 http://192.168.110.124 即可看到登录页面，在其中输入用户名 admin，密码为默认的 qwe.1234，即可登录。

​    登录后点击“新建项目”，新建一个镜像仓库。 

## 4.8.目标服务器安装与配置 *

**4.8.1.docker 引擎** 192.168.110.125 cctarget(1GB ram)

由于目标服务器需要从镜像中心 Harbor 中 docker pull 镜像，然后使用 docker run 来运 行容器，所以目标服务器中需要安装 Docker 引擎。 

**4.8.2.docker-compose** 

由于目标服务器需要通过 docker-compose 运行 compose.yml 文件来启动容器，所以目 标服务器中需要安装 docker-compose。 

**4.8.3.接收目录** 

​    Jenkins 通过 SSH 将命令发送到目标服务器，以使目标服务器可以从 Harbor 拉取镜像、 运行容器等。所以在目标服务器中需要具有一个用户接收 Jenkins 发送数据的目录。本例将 该接收目录创建在/usr/local/jenkins 中。

```bash
[root@cctarget local]# mkdir jenkins
```



## 4.9.Jenkins 安装与配置 * 

**4.9.1.Jenkins 简介** 192.168.110.122:8080 ccjenkins(1GB ram) zhangsan

（1） 百度百科 

​    以下是百度百科中关于 Jenkins 的词条： Jenkins 是一个开源软件项目，是基于 Java 开发的一种持续集成工具，用于监控持续重 复的工作，旨在提供一个开放易用的软件平台，使软件项目可以进行持续集成。 

（2） 主机要求 

​    这里要使用docker方式来安装，所以需要一台安装有docker及docker-compose的主机。 

**4.9.2.安装 JDK** 

​    由于 Jenkins 通过调用 Maven 来实现对项目的构建，所以需要在 Jenkins 主机中安装 Maven。由于 maven 的运行需要 JDK 的环境，所以需要首安装 JDK。 

​    对于 JDK 的安装非常简单，只需要从官网下载相应版本的 JDK 到 Linux 系统后，直接解 压即可。无需配置。这里下载的是 **jdk-8u211-linux-x64.tar.gz**，将其解压到了/opt/apps 目录下， 并重命名为了 jdk。

```bash
[root@ccjenkins ~]# rz -b  # 二进制上传
[root@ccjenkins ~]# ll
总用量 199576
-rw-r--r--  1 root root   9359994 10月  3 02:06 apache-maven-3.9.5-bin.tar.gz
-rw-r--r--  1 root root 194990602 11月  8 11:37 jdk-8u211-linux-x64.tar.gz

[root@ccjenkins ~]# tar -zxvf jdk-8u211-linux-x64.tar.gz -C /opt/apps/
```

```bash
[root@ccjenkins apps]# mv jdk1.8.0_211/ jdk
[root@ccjenkins apps]# ll
总用量 0
drwxr-xr-x 7 10 143 245 4月   2 2019 jdk
```

**4.9.3.安装 maven** 

**（1） 下载解压 maven** 

​    首先需要从官网下载最新版本的 Maven 到 Linux 系统后，直接解压。这里下载的是 **apache-maven-3.9.5-bin.tar.gz**，将其解压到/opt/apps 目录下，并重命名为 maven。

```bash
[root@ccjenkins ~]# wget https://dlcdn.apache.org/maven/maven-3/3.9.5/binaries/apache-maven-3.9.5-bin.tar.gz

[root@ccjenkins ~]# tar -zxvf apache-maven-3.9.5-bin.tar.gz -C /opt/apps/
```

```bash
[root@ccjenkins apps]# mv apache-maven-3.9.5/ maven
[root@ccjenkins apps]# ll
总用量 0
drwxr-xr-x 7   10  143 245 4月   2 2019 jdk
drwxr-xr-x 6 root root  99 11月  8 12:05 maven
```

**（2） 配置 maven 镜像仓库** 

​    maven解压后需要修改解压目录中conf/settings.xml文件中的两处配置。这里配置maven 的镜像源为 aliyun。 

```bash
```

**（3） 配置 maven 编译器版本** 

​    maven 默认的编译器版本为 JDK1.4，这里需要指定为 JDK1.8。配置了该<profile>后，在文件最后的中再激活一下即可。

```bash
```

**4.9.4.安装启动 Jenkins** 

**（1） 下载镜像** 

​    这里要使用 docker 方式来安装 Jenkins，所以需要先下载 Jenkins 的镜像。 

**jenkins/jenkins:2.387.1-lts**

```bash
[root@ccjenkins apps]# docker images
REPOSITORY        TAG           IMAGE ID       CREATED        SIZE
jenkins/jenkins   2.387.1-lts   d5ed2ceef0ec   8 months ago   471MB
```

**（2） 启动 jenkins** 

​    使用 docker run 命令启动 Jenkins。 

```bash
[root@ccjenkins apps]# docker run --name jenkins \
--restart always \
-p 8080:8080 \
-p 50000:50000 \
-v /var/jenkins_home:/var/jenkins_home \
-d jenkins/jenkins:2.387.1-lts
```

**（3） 修改数据卷权限**

​    当 Jenkins 启动后，通过 docker logs jenkins 命令查看 jenkins 的日志可以看到出错了。

```bash
[root@ccjenkins apps]# docker logs jenkins -f
touch: cannot touch '/var/jenkins_home/copy_reference_file.log': Permission denied
Can not write to /var/jenkins_home/copy_reference_file.log. Wrong volume permissions?
touch: cannot touch '/var/jenkins_home/copy_reference_file.log': Permission denied
Can not write to /var/jenkins_home/copy_reference_file.log. Wrong volume permissions?
```

​    原因是，jenkins 需向数据卷挂载点的文件/var/jenkins_home/copy_reference_file.log 中写 入日志时，由于写入操作的用户不是 root 用户，而非 root 用户对数据卷没有写操作权限。

```bash
drwxr-xr-x   2 root root    6 11月  8 12:23 jenkins_home
```

​    此时需要修改数据卷操作权限，为非 root 用户添加写操作权限。

```bash 
[root@ccjenkins var]# chmod -R 777 /var/jenkins_home
```

​    此时再查看，发现已经具备了写操作权限。 

```bash
drwxrwxrwx   2 root root    6 11月  8 12:23 jenkins_home
```

 **（4） 重启 jenkins** 

​    重启 jenkins 容器。 

```bash
[root@ccjenkins apps]# docker restart jenkins
```

**（5） 修改插件下载源** 

​    由于 jenkins 在后期运行时需要下载很多的插件，而这些插件默认都是从国外的 Jenkins 官方服务器上下载的，下载速度很慢，且下载失败的比例很高。所以，一般会先将这些插件 的下载源更新为国内的服务器。 

​    该更新文件是数据卷目录中的 hudson.model.UpdateCenter.xml。

```bash
[root@ccjenkins jenkins_home]# ll
总用量 24
-rw-r--r--  1 admin admin 1663 11月  8 15:53 config.xml
-rw-r--r--  1 admin admin   50 11月  8 15:52 copy_reference_file.log
-rw-r--r--  1 admin admin  156 11月  8 15:52 hudson.model.UpdateCenter.xml
-rw-r--r--  1 admin admin  171 11月  8 15:52 jenkins.telemetry.Correlator.xml
drwxr-xr-x  2 admin admin    6 11月  8 15:52 jobs
-rw-r--r--  1 admin admin  907 11月  8 15:52 nodeMonitors.xml
drwxr-xr-x  2 admin admin    6 11月  8 15:52 nodes
drwxr-xr-x  2 admin admin    6 11月  8 15:52 plugins
-rw-r--r--  1 admin admin   64 11月  8 15:52 secret.key
-rw-r--r--  1 admin admin    0 11月  8 15:52 secret.key.not-so-secret
drwx------  2 admin admin   91 11月  8 15:52 secrets
drwxr-xr-x  2 admin admin   67 11月  8 15:53 updates
drwxr-xr-x  2 admin admin   24 11月  8 15:52 userContent
drwxr-xr-x  3 admin admin   56 11月  8 15:52 users
drwxr-xr-x 11 admin admin  264 11月  8 15:52 war
```

​    查看该文件内容： 

```bash
<?xml version='1.1' encoding='UTF-8'?>
<sites>
  <site>
    <id>default</id>
    <url>https://updates.jenkins.io/update-center.json</url>
  </site>
</sites>
```

​    将该默认的更换为清华大学的下载源地址。 https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/current/update-center.json 

```bash
<?xml version='1.1' encoding='UTF-8'?>
<sites>
  <site>
    <id>default</id>
    <url>https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/current/update-center.json</url>
  </site>
</sites>
```

**（6） 查看 admin 默认密码** 

​    通过 docker logs jenkins 命令查看日志，可以看到已经正常了。并且在最后还可以看到 Jenkins 的 admin 用户及其初始化密码。 

```bash
Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

07fa90e7439b4980a14c58655626aa10

This may also be found at: /var/jenkins_home/secrets/initialAdminPassword

*************************************************************
*************************************************************
*************************************************************
```

**（7） 插件下载** 

​    在浏览器中键入 jenkins 的地址后进行访问，可看到 Jenkins 解锁页面。在管理员密码中 输入前面 docker logs jenkins 中看到的初始化密码后继续。

<img src="images\image-20231108165235046.png" alt="image-20231108165235046" style="zoom:80%;" />

<img src="images\image-20231108165305393.png" alt="image-20231108165305393" style="zoom:80%;" />

​    选择插件来安装。

<img src="images\image-20231108165341278.png" alt="image-20231108165341278" style="zoom:80%;" />

​    这里保持默认的选择即可。

<img src="images\image-20231108165420965.png" alt="image-20231108165420965" style="zoom:80%;" />

​    该页面需要的时间可能会较长。 

**（8） 创建管理员用户** 

<img src="images\image-20231108165456750.png" alt="image-20231108165456750" style="zoom:80%;" />

​    当插件下载完毕后，会自动跳转到该页面。填写上第一个管理员信息后保存并完成。这里填写的用户名为 zhangsan，密码为 qwe.1234。 

<img src="images\image-20231108165605268.png" alt="image-20231108165605268" style="zoom:80%;" />

<img src="images\image-20231108165643435.png" alt="image-20231108165643435" style="zoom:80%;" />

<img src="images\image-20231108165706167.png" alt="image-20231108165706167" style="zoom:80%;" />

**4.9.5.配置 Jenkins** 

**（1） 安装两个插件** 

​    点击 Manage Jenkins 中的 Manage Plugins 页面，在 Available plugins 选项卡页面的搜索 栏中分别键入 Git Parameter 与 Publish Over SSH，选中它们后，Install without restart。

<img src="images\image-20231108165733662.png" alt="image-20231108165733662" style="zoom:80%;" />

<img src="images\image-20231108165753170.png" alt="image-20231108165753170" style="zoom:80%;" />

​    然后就可看到下载过程显示“等待”，直到看到下面的“完成”“Success”后，即可返 回首页了。

<img src="images\image-20231108165826090.png" alt="image-20231108165826090" style="zoom:80%;" />

**（2） 移动 JDK 与 Maven** 

​    首先要将 Jenkins 主机中的 JDK 与 Maven 解压目录移动到数据卷/var/Jenkins_home 中。 

```bash
[root@ccjenkins jenkins_home]# mv /opt/apps/maven/ ./
[root@ccjenkins jenkins_home]# mv /opt/apps/jdk/ ./
```

**（3） 配置 JDK 与 Maven** 

​    在 Manage Jenkins 的 Global Tool Configuration 页面中配置 Maven 与 JDK。 

<img src="images\image-20231108165854354.png" alt="image-20231108165854354" style="zoom:80%;" />

<img src="images\image-20231108165916572.png" alt="image-20231108165916572" style="zoom:80%;" />

<img src="images\image-20231108165937650.png" alt="image-20231108165937650" style="zoom:80%;" />

​    这里填写的是容器中挂载点目录中的路径。 

<img src="images\image-20231108170007866.png" alt="image-20231108170007866" style="zoom:80%;" />

<img src="images\image-20231108170035362.png" alt="image-20231108170035362" style="zoom:80%;" />

​    这里填写的也是容器中挂载点目录中的路径。最后再应用并保存。



## 4.10.Jenkins 集成 SonarQube *

**4.10.1.Jenkins 中安装 SonarScanner** 

**（1） SonarScanner 简介** 

​    SonarScanner 是一种代码扫描工具，专门用来扫描和分析项目代码质量。扫描和分析完 成之后，会将结果写入到 SonarQube 服务器的数据库中，并在 SonarQube 平台显示这些数 据。 

**（2） 下载** 

​    在 SonarQube 官网的帮助文档中可以下载 SonarScanner。这里下载一个 Linux 系统下使用的版本。 

**sonar-scanner-cli-4.8.0.2856-linux.zip**

<img src="images\image-20231113174127657.png" alt="image-20231113174127657" style="zoom:80%;" />

<img src="images\image-20231113174210763.png" alt="image-20231113174210763" style="zoom:80%;" />

**（3） 安装 unzip** 

​    由于下载的 SonarScannner 是 zip 压缩类型的，所以需要在 Linux 系统中安装 unzip 命令， 以解压该压缩包。 

```bash
[root@ccjenkins ~]# yum -y install unzip
```

**（4） 解压/移动** 

​    解压 zip 压缩包。

```bash
[root@ccjenkins ~]# unzip sonar-scanner-cli-4.8.0.2856-linux.zip
```

​    由于后期要在 Jenkins 中集成 SonarScanner，需要 SonarScanner 存在于 Jenkins 服务器中的数据卷目录中。所以将解压后的目录移动到数据卷jenkins_home下并更名为sonar_scanner。 

```bash
[root@ccjenkins ~]# mv sonar-scanner-4.8.0.2856-linux/ /var/jenkins_home/sonar-scanner
```

```bash
...
drwx------  2 admin admin  162 11月  8 16:10 secrets
drwxr-xr-x  6 root  root    51 12月 22 2022 sonar-scanner
drwxr-xr-x  2 admin admin  182 11月  8 16:11 updates
drwxr-xr-x  2 admin admin   24 11月  8 15:52 userContent
drwxr-xr-x  3 admin admin   59 11月  8 16:07 users
drwxr-xr-x 11 admin admin  264 11月  8 15:52 war
[root@ccjenkins jenkins_home]# 
```

**（5） 修改配置文件** 

​    在 sonar-scanner 目录的 conf 目录下有其配置文件 sonar-scanner.properties。修改该配置 文件。 

​    ./代表sonar-scanner 目录下的 workspace文件夹

```bash
/var/jenkins_home/sonar-scanner/conf
[root@ccjenkins conf]# vim sonar-scanner.properties

#Configure here general information about the environment, such as SonarQube server connection details for example
#No information about specific project should appear here

#----- Default SonarQube server
sonar.host.url=http://192.168.110.123:9000

#----- Default source code encoding
sonar.sourceEncoding=UTF-8

sonar.sources=./
sonar.java.binaries=./target
```



**4.10.2.Jenkins 配置 SonarQube** 

**（1） 安装插件** 

​    在 Jenkins 页面的系统管理 - 插件管理 - Available plugins 中搜索 sonarqube scanner， 安装该插件。该插件用于连接 SonarScanner。 

<img src="images\image-20231113174313462.png" alt="image-20231113174313462" style="zoom:80%;" />

<img src="images\image-20231113174348655.png" alt="image-20231113174348655" style="zoom:80%;" />

​    安装完毕后，点选“安装完成后重启 Jenkins”，进行重启。

**（2） 添加 Sonarqube** 

​    打开 Jenkins 的 Manage Jenkins - Configure System 页面，找到 SonarQube servers，添 加 SonarQube 服务器。 

<img src="images\image-20231113174427806.png" alt="image-20231113174427806" style="zoom:80%;" />

​    点击 Add SonarQube 按钮后即可看到以下配置框。

http://192.168.110.123:9000

<img src="images\image-20231113174511400.png" alt="image-20231113174511400" style="zoom:80%;" />

<img src="images\image-20231113174538179.png" alt="image-20231113174538179" style="zoom:80%;" />

**（3） 添加 SonarScanner** 

​    这里要将前面安装在 Jenkins 数据卷中的 SonarScanner 配置到 Jenkins 中。 

​    在 Jenkins 页面的 Manage Jenkins - 全局工具配置 中找到 SonarQube Scanner。

<img src="images\image-20231113174845900.png" alt="image-20231113174845900" style="zoom:80%;" />

<img src="images\image-20231113174923254.png" alt="image-20231113174923254" style="zoom:80%;" />

## 4.11.Jenkins 集成目标服务器 *

​    这里要配置连接到目标服务器的连接方式。打开 Manage Jenkins 中的 Configure System 页面。 

<img src="images\image-20231113175027234.png" alt="image-20231113175027234" style="zoom:80%;" />

​    将页面拉到最下面，可以看到 Publish over SSH。这里可以设置非对称加密的身份验证方 式，也可设置对称加密的身份验证方式。这里采用对称加密身份验证方式。点击新增按钮。

<img src="images\image-20231113175058699.png" alt="image-20231113175058699" style="zoom:80%;" />

192.168.110.125

<img src="images\image-20231113175239411.png" alt="image-20231113175239411" style="zoom:80%;" />

<img src="images\image-20231113175406545.png" alt="image-20231113175406545" style="zoom:80%;" />

​    填写完毕后，页面拉到最下面，点击 Test Configuration 进行测试。如果可以看到 Success， 说明连接成功。然后再应用并保存。 



## 4.12.自由风格的 CI 操作(中间架构) **

**4.12.1中间架构图** 

<img src="images\image-20231113175510802.png" alt="image-20231113175510802" style="zoom:80%;" />

**4.12.2.创建 web 项目** 

​    创建一个 web 项目，就使用最简单的 spring boot 工程，例如工程名为 hellojks。仅需导 入 spring web 依赖即可。

**（1） 创建工程** 

​    创建前基本环境配置

<img src="images\001.png" alt="001" style="zoom:80%;" />

<img src="images\002.png" alt="002" style="zoom:80%;" />

创建一个 Spring Boot 工程，其仅包含一个 spring web 依赖。 

<img src="images\003.png" alt="003" style="zoom:80%;" />

<img src="images\004.png" alt="004" style="zoom:80%;" />

<img src="images\005.png" alt="005" style="zoom:80%;" />

<img src="images\006.png" alt="006" style="zoom:80%;" />

检查项目是否有问题

<img src="images\007.png" alt="007" style="zoom:80%;" />

<img src="images\008.png" alt="008" style="zoom:80%;" />

<img src="images\009.png" alt="009" style="zoom:80%;" />



**（2） 定义 Controller** 

​    只需定义一个简单的 Controller 即可。 

<img src="images\010.png" alt="010" style="zoom:80%;" />

<img src="images\011.png" alt="011" style="zoom:80%;" />

运行

<img src="images\012.png" alt="012" style="zoom:80%;" />

**（3） 访问效果** 

<img src="images\013.png" alt="013" style="zoom:80%;" />

**4.12.3.Idea 提交项目到远程仓库** 

**（1） 在 GitLab 中创建远程仓库** 

​    首先在 GitLab 中创建一个远程仓库，用于管理前面 Idea 中创建的工程。

<img src="images\image-20231113175804612.png" alt="image-20231113175804612" style="zoom:80%;" />

<img src="images\image-20231113175827682.png" alt="image-20231113175827682" style="zoom:80%;" />

<img src="images\image-20231113175854549.png" alt="image-20231113175854549" style="zoom:80%;" />

​    点击Create project后就可进入下个页面，可以看到当前仓库的信息及相关的操作命令。 客户端通过这些命令可完成对该仓库的操作。

<img src="images\image-20231113180231449.png" alt="image-20231113180231449" style="zoom:80%;" />

 **（2） 创建用户** 

​    仿照远程仓库页面中的 Git global stetup 中的命令，在项目的 Terminal 窗口中创建一个 全局用户。

<img src="images\image-20231113180308392.png" alt="image-20231113180308392" style="zoom:80%;" />

**（3） 初始化本地仓库** 

​    将当前的项目目录 hellojks 初始化为本地仓库。

<img src="images\image-20231113180346449.png" alt="image-20231113180346449" style="zoom:80%;" />

<img src="images\image-20231113180429253.png" alt="image-20231113180429253" style="zoom:80%;" />

**（4） 提交代码到本地库** 

​    在项目上右击，选择 Git - Commit Directory。

<img src="images\image-20231113180509321.png" alt="image-20231113180509321" style="zoom:80%;" />

​    此时会弹出一个 Commit to master 的窗口。在其中选择要提交的文件，并在文本区填写 提交日志。然后 Commit。

<img src="images\image-20231113180538282.png" alt="image-20231113180538282" style="zoom:80%;" />

​    然后会看到警告，不影响提交，直接再 Commit Anyway 即可。

<img src="images\image-20231113180610565.png" alt="image-20231113180610565" style="zoom:80%;" />

​    提交后变为以下形式。 

<img src="images\image-20231113180711165.png" alt="image-20231113180711165" style="zoom:80%;" />

**（5） 提交到远程库** 

​    首先要从远程仓库中获取仓库地址。选择复制 Clone with HTTP 的地址。

<img src="images\image-20231113180649990.png" alt="image-20231113180649990" style="zoom:80%;" />

​    然后在项目上右键，选择 Git - Push。 

<img src="images\image-20231113180745310.png" alt="image-20231113180745310" style="zoom:80%;" />

​    在新窗口中点击 Define remote，在弹出的窗口中粘贴进复制来的远程仓库地址。

192.168.110.121

<img src="images\image-20231113180815229.png" alt="image-20231113180815229" style="zoom:80%;" />

<img src="images\image-20231113180926533.png" alt="image-20231113180926533" style="zoom:80%;" />

​    Push 后会弹出访问 GitLab 的登录窗口，输入用户名 root，密码为前面修改过的 qwe.1234 。 

<img src="images\image-20231113181001830.png" alt="image-20231113181001830" style="zoom:80%;" />

​    推送成功后，在 idea 右下角即可看到成功提示。

<img src="images\image-20231113181055577.png" alt="image-20231113181055577" style="zoom:80%;" />

​    此时刷新 GitLab 页面，即可看到推送来的项目。 

<img src="images\image-20231113181124723.png" alt="image-20231113181124723" style="zoom: 80%;" />

**4.12.4从 GitLab 拉取代码** 

**（1） 新建任务**  

<img src="images\image-20231113181155735.png" alt="image-20231113181155735" style="zoom:80%;" />

<img src="images\image-20231113181241464.png" alt="image-20231113181241464" style="zoom:80%;" />

**（2） Jenkins 集成 GitLab** 

​    在点击确定后即可立即配置 Jenkins 中 GitLab 服务器的信息。 192.168.110.121

<img src="images\image-20231113181306917.png" alt="image-20231113181306917" style="zoom:80%;" />

​    对于 public 的 GitLab 仓库，直接指定仓库地址，应用保存即可。但对于 private 仓库，则需要指定访问 GitLab 的用户名与密码。点击添加按钮，即可打开下面的窗口。

<img src="images\image-20231113181420660.png" alt="image-20231113181420660" style="zoom:80%;" />

​    在其中填写用户名与密码后“添加”即可返回之前的页面，此时在 Credentials 下拉框中 即可找到新添加的用户信息，选择即可。 

![image-20231113181457167](images\image-20231113181457167.png)

**（3） 立即构建**

​    任务创建成功后即可看到如下页面。在该页面中点击“立即构建”，Jenkins 即可开始从 GitLab 上拉取项目。此时右下角就会发生变化。 

<img src="images\image-20231113181517195.png" alt="image-20231113181517195" style="zoom:80%;" />

​    点击右下角的日期时间，选择控制台输出，可看到这个拉取过程的日志。

<img src="images\image-20231113181544943.png" alt="image-20231113181544943" style="zoom: 80%;" />

​    从以上日志的 git init /var/jenkins_home/workspace/my_hellojks 命令可以看出，Jenkins 将其容器内的/var/jenkins_home/workspace/my_hellojks 目录作为项目的本地仓库。也就是将 数据卷目录。进入 jenkins 数据卷可以看到该项目已经存在了。

```bash
[root@ccjenkins my_hellojks]# ll
总用量 4
-rw-r--r-- 1 admin admin 1415 11月 13 11:33 pom.xml
drwxr-xr-x 4 admin admin   30 11月 13 11:33 src
```

 **4.12.5.将项目打为 jar 包** 

​    在 Jenkins 能够通过配置，调用本地的 maven 的 mvn 命令，将拉取来的项目打为 Jar 包。 

**（1） Jenkins 配置 mvn 命令** 

<img src="images\image-20231113181631802.png" alt="image-20231113181631802" style="zoom: 80%;" />

​    点击配置后，打开配置页面。然后点击 Build Steps，跳转到以下位置。

<img src="images\image-20231113181651727.png" alt="image-20231113181651727" style="zoom: 80%;" />

<img src="images\image-20231113181711860.png" alt="image-20231113181711860" style="zoom:80%;" />

​    选择调用顶层 Maven 目标，即可使用前面配置的 Maven 来完成打包任务。 

<img src="images\image-20231113181734258.png" alt="image-20231113181734258" style="zoom:80%;" />

​    在 Maven 版本下拉框中选择前面配置好的 maven，目标中写入需要 maven 去执行的 maven 命令，应用保存后，自动跳转回任务首页。 

**（2） 重新构建** 

​    在配置好 maven 的构建命令后，再次执行“立即构建”。 

<img src="images\image-20231113181807166.png" alt="image-20231113181807166" style="zoom:80%;" />

​    查看构建成功的日志，可看到熟悉的 BUILD SUCCESS。 

<img src="images\image-20231113181831931.png" alt="image-20231113181831931" style="zoom:80%;" />

​    构建成功后进入 jenkins 数据卷目录/var/jenkins_home/workspace/my_hellojks 中可以看到新增了 target 目录。打开 target 目录，可以看到打出的 jar 包。

```bash
[root@ccjenkins my_hellojks]# ll
总用量 4
-rw-r--r-- 1 admin admin 1415 11月 13 11:33 pom.xml
drwxr-xr-x 4 admin admin   30 11月 13 11:33 src
drwxr-xr-x 8 admin admin  217 11月 13 11:43 target
[root@ccjenkins my_hellojks]# ll target/
总用量 17348
drwxr-xr-x 3 admin admin       47 11月 13 11:43 classes
drwxr-xr-x 3 admin admin       25 11月 13 11:42 generated-sources
drwxr-xr-x 3 admin admin       30 11月 13 11:43 generated-test-sources
-rw-r--r-- 1 admin admin 17759236 11月 13 11:43 hellojks-0.0.1-SNAPSHOT.jar
-rw-r--r-- 1 admin admin     3080 11月 13 11:43 hellojks-0.0.1-SNAPSHOT.jar.original
drwxr-xr-x 2 admin admin       28 11月 13 11:43 maven-archiver
drwxr-xr-x 3 admin admin       35 11月 13 11:42 maven-status
drwxr-xr-x 3 admin admin       17 11月 13 11:43 test-classes
[root@ccjenkins my_hellojks]# pwd
/var/jenkins_home/workspace/my_hellojks
```

**4.12.6.代码质量检测** 

**（1） sonar-scanner 手动检测** 

​    在 Jenkins 中数据卷目录/var/Jenkins_home 中已经安装过了 sonar-scanner。该目录下的 bin 目录中有 sonar-scanner 命令。Jenkins 就是通过调用该命令完成代码质量检测的。 

```bash
[root@ccjenkins bin]# ll
总用量 8
-rwxr-xr-x 1 root root 1822 12月 22 2022 sonar-scanner
-rwxr-xr-x 1 root root  662 12月 22 2022 sonar-scanner-debug
[root@ccjenkins bin]# pwd
/var/jenkins_home/sonar-scanner/bin
```

​    这里先通过手工执行命令方式来体验一下 sonar-scanner 命令完成代码检测的过程。 由于配置文件中使用的相对路径都是相对路径，所以若要运行 sonar-scanner 命令对项 目进行手工质量检测，就需要在 workspace 的需要检测的项目目录中执行。所以，要么配置 全局变量 path，要么直接使用 sonar-scanner 命令的绝对路径。为了简单，采用命令绝对路 径方式。 

```bash
# 注意，一定要在项目目录下执行，在哪执行，sonar-scanner就扫哪里。
# 因为/var/jenkins_home/sonar-scanner/conf 下的sonar-scanner.properties 
sonar.sources=./ 
sonar.java.binaries=./target

[root@ccjenkins my_hellojks]# ll
-rw-r--r-- 1 admin admin 1415 11月 13 11:33 pom.xml
drwxr-xr-x 4 admin admin   30 11月 13 11:33 src
drwxr-xr-x 8 admin admin  217 11月 13 11:43 target 
# target里面有maven打包的jar包

# 本例执行目录 /var/jenkins_home/workspace/my_hellojks
/var/jenkins_home/sonar-scanner/bin/sonar-scanner \
-Dsonar.login=admin \
-Dsonar.password=qwe.1234 \
-Dsonar.projectKey=my_hello_jks
```

​    看到以下日志，说明检测成功。 

```bash
INFO: Analysis report compressed in 20ms, zip size=21.7 kB
INFO: Analysis report uploaded in 162ms
INFO: ANALYSIS SUCCESSFUL, you can find the results at: http://192.168.110.123:9000/dashboard?id=my_hello_jks
INFO: Note that you will be able to access the updated dashboard once the server has processed the submitted analysis report
INFO: More about the report processing at http://192.168.110.123:9000/api/ce/task?id=AYvG_ZDbf6nS70rHfdrd
INFO: Analysis total time: 8.201 s
INFO: ------------------------------------------------------------------------
INFO: EXECUTION SUCCESS
INFO: ------------------------------------------------------------------------
INFO: Total time: 9.469s
INFO: Final Memory: 23M/71M
INFO: ------------------------------------------------------------------------
```

​    此时，在 sonarqube 首页就可看到多出了一个检测项目。 

<img src="images\image-20231113181928229.png" alt="image-20231113181928229" style="zoom: 80%;" />

**（2） 任务中配置 SonarScanner** 

​    现在要在 Jenkins 的 my_hellojks 项目中应用 SonarScanner 对其代码进行质量检测。所以 需要在该项目中配置 SonarScanner。

<img src="images\image-20231113181953935.png" alt="image-20231113181953935" style="zoom:80%;" />

<img src="images\image-20231113182018879.png" alt="image-20231113182018879" style="zoom: 80%;" />

<img src="images\image-20231113182045836.png" alt="image-20231113182045836" style="zoom:80%;" />

**（3） 重新构建** 

​    <font color=red>注意！要先删除上次手工检测的遗留文件，否则报错！</font>

```bash
[root@ccjenkins my_hellojks]# ll -a
总用量 8
drwxr-xr-x 6 admin admin   96 11月 13 16:57 .
drwxr-xr-x 3 admin admin   25 11月 13 11:26 ..
drwxr-xr-x 8 admin admin  315 11月 13 16:58 .git
-rw-r--r-- 1 admin admin  395 11月 13 11:33 .gitignore
-rw-r--r-- 1 admin admin 1415 11月 13 11:33 pom.xml
drwxr-xr-x 2 root  root    48 11月 13 16:58 .scannerwork
drwxr-xr-x 4 admin admin   30 11月 13 11:33 src
drwxr-xr-x 8 admin admin  217 11月 13 16:57 target
[root@ccjenkins my_hellojks]# rm -rf .scannerwork/
```

​    再次执行“立即构建”，构建成功后，刷新 SonarQube 页面，便可看到新增了一个项目。

<img src="images\image-20231113182120236.png" alt="image-20231113182120236" style="zoom:80%;" />

 **4.12.7.将 jar 包推送到目标服务器** 

**（1） 配置 SSH** 

​    Jenkins 通过 SSH 方式连接上目标服务器，并将 jar 包推送到目标服务器。 

<img src="images\image-20231113182157734.png" alt="image-20231113182157734" style="zoom:80%;" />

​    点击配置后，打开配置页面。将页面拉到最下面，找到“增加构建后操作步骤”。 

<img src="images\image-20231113182232079.png" alt="image-20231113182232079" style="zoom:80%;" />

<img src="images\image-20231113182300694.png" alt="image-20231113182300694" style="zoom:80%;" />

**（2） 重新构建** 

​    在返回的任务首页中，再次执行立即构建。查看日志可以看到连接目标服务器，推送 1 个文件的日志。 

<img src="images\image-20231113182323605.png" alt="image-20231113182323605" style="zoom:80%;" />

​    查看目标服务器的目标目录/usr/local/jenkins，可以看到 jar 包已经推送了过来。

```bash
[root@cctarget jenkins]# ll
总用量 0
drwxr-xr-x 2 root root 41 11月 13 17:26 target
[root@cctarget jenkins]# ll target/
总用量 17344
-rw-r--r-- 1 root root 17759236 11月 13 17:26 hellojks-0.0.1-SNAPSHOT.jar
```

**4.12.8.构建镜像启动容器** 

​    通过在 Jenkins 中配置在目标服务器中将要执行的相关命令，使得 Jenkins 将 jar 包推送 到目标服务器后，立即自动执行配置的命令，将 jar 包构建为一个镜像，并启动其相应的容器，使项目启动运行。 

先用 idea 在 terminal 中输入 `mvn clean package` 将项目打包成 hellojks-0.0.1-SNAPSHOT.jar，位置在 target 文件夹下

如果mvn命令不可用，定义系统变量和环境变量下的 MAVEN_HOME 和 Path

**（1） 定义 Dockerfile** 

​    若要构建镜像，就需要定义其 Dockerfile。现在 Idea 的工程中新建一个 Directory，例如 docker，然后在其中新建一个 file。 

<img src="images\image-20231115185507886.png" alt="image-20231115185507886" style="zoom:80%;" />

```bash
FROM openjdk:8u102
LABEL auth="zhangsan" email="zhangsan@163.com"
COPY hellojks-0.0.1-SNAPSHOT.jar /usr/local/jenkins/hellojks.jar
WORKDIR /usr/local/jenkins/
ENTRYPOINT ["java","-jar","hellojks.jar"]
```

**（2） 定义 compose.yml** 

​    在 idea 的新建目录中再新建一个 compose.yml，用于构建镜像和启动容器。 

<img src="images\image-20231115185543572.png" alt="image-20231115185543572" style="zoom:80%;" />

```bash
services:
  hellojks:
    build: ./
    image: hellojks
    container_name: myhellojks
    ports:
      - 8080:8080
```



**（3） 推送到 GitLab** 

​    将定义的这两个文件推送到 GitLab。在项目上右击，选择 Git - Commit Directory…。 www.bjpowernode.com 503 / 566 Copyright© 动力节点 

<img src="images\image-20231115185616065.png" alt="image-20231115185616065" style="zoom:80%;" />

<img src="images\image-20231115185636074.png" alt="image-20231115185636074" style="zoom:80%;" />

**（4） 再配置构建后操作** 

​    重新返回到任务首页，再次对“构建后操作”进行配置。

<img src="images\image-20231115185715630.png" alt="image-20231115185715630" style="zoom:80%;" />

<img src="images\image-20231115185736263.png" alt="image-20231115185736263" style="zoom:80%;" />

`target/*.jar docker/*`

```bash
cd /usr/local/jenkins/docker
mv ../target/*.jar ./
docker compose down
docker compose up -d --build
docker image prune -f
```



**（5） 重新构建** 

​    Jenkins 中在返回的任务首页中，再次执行立即构建。构建成功后，查看目标服务器中 的 /usr/local/jenkins 目录，发现 docker 目录及其下的两个 docker 文件已经存在了，且 jar 包 也复制了进来。 

```bash
[root@cctarget jenkins]# ll
总用量 0
drwxr-xr-x 2 root root 78 11月 14 12:42 docker
drwxr-xr-x 2 root root  6 11月 14 12:42 target
[root@cctarget jenkins]# ll docker
总用量 17352
-rw-r--r-- 1 root root      115 11月 14 12:42 compose.yml
-rw-r--r-- 1 root root      200 11月 14 12:42 Dockerfile
-rw-r--r-- 1 root root 17759249 11月 14 12:42 hellojks-0.0.1-SNAPSHOT.jar
```

​    在目标服务器中 docker images，可以看到 hellojks 镜像已经生成。 

```bash
[root@cctarget jenkins]# docker images
REPOSITORY   TAG       IMAGE ID       CREATED         SIZE
hellojks     latest    4a6b61ecfe16   8 minutes ago   659MB
openjdk      8u102     ca5dd051db43   7 years ago     641MB
```

​    在目标服务器中 docker ps，可以看到容器已经启动了。 

```bash
[root@cctarget jenkins]# docker ps
CONTAINER ID   IMAGE      COMMAND                   CREATED         STATUS         PORTS                                       NAMES
f02b2f59324a   hellojks   "java -jar hellojks.…"   8 minutes ago   Up 8 minutes   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp   myhellojks
```

​    在浏览器中访问目标服务器中的应用，已经可以访问了。

http://192.168.110.125:8080/some

<img src="images\image-20231115185823314.png" alt="image-20231115185823314" style="zoom:80%;" />



## 4.13.自由风格的 CI 操作(最终架构) **

​    前面的架构存在的问题是，若有多个目标服务器都需要使用该镜像，那么每个目标服务器都需要在本地构建镜像，形成系统资源浪费。若能够在 Jenkins 中将镜像构建好并推 送到 Harbor 镜像中心，那么无论有多少目标服务器需要该镜像，都只需要从 Harbor 拉取即可。

**4.13.1.Jenkins 容器化实现** 

**（1） Jenkins 容器化实现方案**

​    有两种方案： 

- DioD：要容器内部安装 Docker 引擎 

- DooD：与宿主机共享 Docker 引擎 （推荐）

删除 idea 中的 comopse.yml 文件

**（2） 修改 docker.sock 权限** 

​    /var/run/docker.sock 文件是 docker client 和 docker daemon 在本地进行通信的 socket 文件。默认的组为 docker，且 other 用户不具有读写权限，这样 Jenkins 是无法来操作该文件的。

```bash
drwx------  8 root           root            180 11月 14 12:05 docker
-rw-r--r--  1 root           root              4 11月 14 12:05 docker.pid
srw-rw----  1 root           docker            0 11月 14 12:05 docker.sock
-rw-------  1 root           root              0 11月 14 12:03 ebtables.lock
drwxr-xr-x  2 root           root             40 11月 14 12:03 faillock
```

​    所以这里需要将其组调整为 root，且为其分配读写权限。 

```bash
[root@ccjenkins run]# chown root:root docker.sock
[root@ccjenkins run]# chmod o+rw docker.sock
```

​    此时再查看便可看到组与权限已经发生了变化。

​    注意，docker.sock 每次关机 都会恢复原状

```bash
drwx------  8 root           root            180 11月 14 12:05 docker
-rw-r--r--  1 root           root              4 11月 14 12:05 docker.pid
srw-rw-rw-  1 root           root              0 11月 14 12:05 docker.sock
-rw-------  1 root           root              0 11月 14 12:03 ebtables.lock
drwxr-xr-x  2 root           root             40 11月 14 12:03 faillock
```

**（3） 修改 Jenkins 启动命令后重启** 

​    首先强制删除正在运行的 Jenkins 容器。 

```bash
[root@ccjenkins var]# docker rm -f jenkins
```

​    然后在 Jenkins 启动命令中新增/var/run/docker.sock，docker 命令文件/usr/bin/docker， 及/etc/docker/daemon.json 文件为数据卷。重启 Jenkins 容器。 

```bash
docker run --name jenkins \
--restart always \
-p 8080:8080 \
-p 50000:50000 \
-v /var/jenkins_home:/var/jenkins_home \
-v /var/run/docker.sock:/var/run/docker.sock \
-v /usr/bin/docker:/usr/bin/docker \
-v /etc/docker/daemon.json:/etc/docker/daemon.json \
-d jenkins/jenkins:2.387.1-lts
```

**4.13.2.构建并推送镜像到 Harbor** 

​    这里要实现 Jenkins 将来自 GitLab 的 jar 包构建为镜像，并推送到 Harbor。

（1） 修改 daemon.json 文件 

​    Jenkins 是 Harbor 的客户端，需要修改/etc/docker/daemon.json 文件。修改后重启 Docker。

```bash 
{
"insecure-registries": ["192.168.110.124"]
}
```

（2） Jenkins 删除构建后操作 

​    原来的 Jenkins 中配置的“构建后操作”完成的是将代码推送到目标服务器后，让目标 服务器通过 docker compose 完成镜像的构建与启动。但现在不需要了，因为镜像构建任务 要由 Jenkins 自己完成了。在 Jenkins 当前任务下的“配置”中删除。

<img src="images\image-20231115190122227.png" alt="image-20231115190122227" style="zoom:80%;" />

<img src="images\image-20231115190147042.png" alt="image-20231115190147042" style="zoom:80%;" />

（3） Jenkins 添加 shell 命令 

​    在 sonarqube 对代码质量检测完毕后，再添加一个“构建步骤”。这个构建步骤通过 shell 命令方式完成。

<img src="images\image-20231115190218287.png" alt="image-20231115190218287" style="zoom:80%;" />

<img src="images\image-20231115190242956.png" alt="image-20231115190242956" style="zoom:80%;" />

```bash
mv target/*.jar docker/
docker build -t hellojks docker/
docker login -u admin -p qwe.1234 192.168.110.124
docker tag hellojks 192.168.110.124/jks/hellojks
docker image prune -f
docker push 192.168.110.124/jks/hellojks
```



（4） 重新构建 

​    Jenkins 中在返回的任务首页中，再次执行立即构建。构建成功后，在 Jenkins 主机中可 以查看到构建好的镜像与重 tag 过的镜像。 

```bash 
[root@ccjenkins my_hellojks]# docker images
REPOSITORY                     TAG           IMAGE ID       CREATED         SIZE
192.168.110.124/jks/hellojks   latest        da8c4987a161   3 minutes ago   659MB
hellojks                       latest        da8c4987a161   3 minutes ago   659MB
jenkins/jenkins                2.387.1-lts   d5ed2ceef0ec   8 months ago    471MB
openjdk                        8u102         ca5dd051db43   7 years ago     641MB
```

​    在 harbor 的仓库中也可以看到推送来的镜像。

<img src="images\image-20231115190319313.png" alt="image-20231115190319313" style="zoom:80%;" />

<img src="images\image-20231115190342271.png" alt="image-20231115190342271" style="zoom:80%;" />

<img src="images\image-20231115190406189.png" alt="image-20231115190406189" style="zoom:80%;" />

**4.13.3.通知目标服务器** 

**（1） 修改 daemon.json 文件** 

​    目标服务器是 Harbor 的客户端，需要修改/etc/docker/daemon.json 文件。修改后重启 Docker。 

```bash
{
"insecure-registries": ["192.168.110.124"]
}
```

**（2） 定义脚本文件** 

​    在目标服务器 PATH 路径下的任意目录中定义一个脚本文件 deploy.sh。例如，定义在 /usr/local/bin 目录下。然后再为其赋予可执行权限。这样该 deploy 命令就可以在任意目录 下运行了。 

```bash
[root@cctarget bin]# chmod a+x deploy.sh
[root@cctarget bin]#
[root@cctarget bin]# ll
总用量 0
-rwxr-xr-x 1 root root 0 11月 15 10:22 deploy.sh
```

文件内容如下：

```bash
#传参顺序
harbor_addr=$1
harbor_proj=$2
image_repo=$3
image_tag=$4
app_port=$5
export_port=$6

#先删除已经存在的同名容器
exist_container_id=`docker ps -a|grep $image_repo|awk '{print $1}'`

if [ "$exist_container_id" != "" ]; then
        docker stop $exist_container_id
        docker rm $exist_container_id
fi

#先删除已经存在的同名镜像
exist_image_tag=`docker images|grep $harbor_addr/$harbor_proj/$image_repo|awk '{print $2}'`

image=$harbor_addr/$harbor_proj/$image_repo:$image_tag

if [[ "$exist_image_tag" =~ "$image_tag" ]]; then
        docker rmi -f $image
fi


docker login -u admin -p qwe.1234 $harbor_addr
docker pull $image
docker run --name $image_repo -d -p $export_port:$app_port $image

echo "SUCCESS"
```

**（3） Jenkins 添加端口号参数** 

​    为了使用户可以随时指定容器对外暴露的参数，这里在 Jenkins 当前任务下的“配置” 中“参数化构建过程”中添加一个字符参数。

<img src="images\image-20231115190438959.png" alt="image-20231115190438959" style="zoom:80%;" />

<img src="images\image-20231115190458341.png" alt="image-20231115190458341" style="zoom:80%;" />

<img src="images\image-20231115190523915.png" alt="image-20231115190523915" style="zoom:80%;" />

**（4） Jenkins 添加构建后操作** 

​    还是在 Jenkins 当前任务下的“配置”中，为任务添加构建后操作。

<img src="images\image-20231115190555831.png" alt="image-20231115190555831" style="zoom:80%;" />

<img src="images\image-20231115190613420.png" alt="image-20231115190613420" style="zoom:80%;" />

<img src="images\image-20231115190644056.png" alt="image-20231115190644056" style="zoom:80%;" />

**（5） 重新构建工程** 

​    这次重新构建，可以看到出现了 export_port 的文本框。在这里可以修改容器对外暴露的端口号8083。

<img src="images\image-20231115190710406.png" alt="image-20231115190710406" style="zoom:80%;" />

​    构建成功后可以看到，目标服务器中增加了新的镜像，该镜像是从 harbor 拉取的。 

```bash
[root@cctarget bin]# docker images
REPOSITORY                     TAG       IMAGE ID       CREATED              SIZE
192.168.110.124/jks/hellojks   latest    d308c1ef340a   About a minute ago   659MB
hellojks                       latest    4a6b61ecfe16   24 hours ago         659MB
openjdk                        8u102     ca5dd051db43   7 years ago          641MB
```

​    还可以看到，该镜像的容器也已经启动。 

```bash
[root@cctarget bin]# docker ps
CONTAINER ID   IMAGE                                 COMMAND                   CREATED          STATUS          PORTS                                       NAMES
4da4f2a5fa84   192.168.110.124/jks/hellojks:latest   "java -jar hellojks.…"   18 seconds ago   Up 16 seconds   0.0.0.0:8083->8080/tcp, :::8083->8080/tcp   hellojks
```

​    通过浏览器访问目标服务器的应用，是没有问题的。

http://192.168.110.125:8083/some

<img src="images\image-20231115190738372.png" alt="image-20231115190738372" style="zoom:80%;" />

## 4.14.自由风格的 CD 操作  **

​    现在要为 GitLab 中当前的项目主干分支 origin/master 上的代码打上一个 Tag，例如 v1.0.0。 然后修改代码后仍提交到 GitLab 的主干分支 origin/master 上，此时再给项目打上一个 Tag， 例如 v2.0.0。这样， hellojenkins 项目的主干分支 origin/master 上就打上了两个 Tag。 

​    而 Jenkins 可以根据主干分支 origin/master 上代码的不同 Tag 对项目进行分别构建。实现项目的持续交付与持续部署。 

**4.14.1.发布 V1.0.0 版本** 

**（1） 修改代码并推送** 

​    简单修改一个 Controller 中方法的返回值。修改代码后，将其推送到 GitLab。

<img src="images\image-20231115190803513.png" alt="image-20231115190803513" style="zoom:80%;" />

**（2） GitLab 中项目打 Tag**

<img src="images\image-20231115190831200.png" alt="image-20231115190831200" style="zoom:80%;" />

<img src="images\image-20231115190920151.png" alt="image-20231115190920151" style="zoom:80%;" />

<img src="images\image-20231115190941872.png" alt="image-20231115190941872" style="zoom:80%;" />

<img src="images\image-20231115191000964.png" alt="image-20231115191000964" style="zoom:80%;" />

**4.14.2.发布 V2.0.0 版本** 

**（1） 修改代码** 

​    简单修改一个 Controller 中方法的返回值。将修改后的项目源码提交到 GitLab。 

<img src="images\image-20231115191026080.png" alt="image-20231115191026080" style="zoom:80%;" />

**（2） GitLab 中再打 Tag** 

​    在 GitLab 中再次为刚提交到主干分支 origin/master 上的代码再打上一个新的 Tag。

<img src="images\image-20231115191108935.png" alt="image-20231115191108935" style="zoom:80%;" />

<img src="images\image-20231115191131438.png" alt="image-20231115191131438" style="zoom:80%;" />

<img src="images\image-20231115191153708.png" alt="image-20231115191153708" style="zoom:80%;" />

<img src="images\image-20231115191214229.png" alt="image-20231115191214229" style="zoom:80%;" />

​    此时可以看到，当前项目具有了两个 Tag。 

<img src="images\image-20231115191233900.png" alt="image-20231115191233900" style="zoom:80%;" />

**4.14.3.Jenkins 配置 tag 参数** 

​    由于 GitLab 中的项目具有 tag 标签，那么 Jenkins 在进行项目构建时就需要让用户选择 准备构建哪个 tag 的项目。所以，需要在 Jenkins 中配置一个 Git 参数 tag 作为用户选项。 

**（1） 添加 Git 参数** 

<img src="images\image-20231115191256468.png" alt="image-20231115191256468" style="zoom:80%;" />

<img src="images\image-20231115191316800.png" alt="image-20231115191316800" style="zoom:80%;" />

​    这里选择的 Git 参数，即为前面下载的 Git Parameter 插件。

htag

<img src="images\image-20231115191348144.png" alt="image-20231115191348144" style="zoom:80%;" />

**（2） 添加 checkout 命令** 

​    然后当前页面继续下拉，找到 Build Steps。

<img src="images\image-20231115191507173.png" alt="image-20231115191507173" style="zoom:80%;" />

`git checkout $htag`

<img src="images\image-20231115191602982.png" alt="image-20231115191602982" style="zoom:80%;" />

**（3） 修改构建命令配置** 

​    然后当前页面继续下拉，找到 Build Steps 中原来添加的构建命令。在所有涉及镜像的命 令中添加上$htag 变量引用。然后应用保存。 

<img src="images\image-20231115191638797.png" alt="image-20231115191638797" style="zoom:80%;" />

```bash
mv target/*.jar docker/
docker build -t hellojks:$htag docker/
docker login -u admin -p qwe.1234 192.168.110.124
docker tag hellojks:$htag 192.168.110.124/jks/hellojks:$htag
docker image prune -f
docker push 192.168.110.124/jks/hellojks:$htag
```



**（4） 修改 SSH 配置** 

​    然后当前页面继续下拉，找到“构建后操作”中的 Send build artifacts over SSH 中的 Exec  command，将原来写死的版本 latest 修改为$htag。

`deploy.sh 192.168.110.124 jks hellojks $htag 8080 $export_port`

<img src="images\image-20231115191738922.png" alt="image-20231115191738922" style="zoom:80%;" />

**4.14.4.部署 v1.0.0** 

**（1） 重新构建工程** 

​    任务首页中再次点击 Build with Parameters 构建项目，发现增加了 hjtag 选项。这里选择 v1.0.0 进行构建。

<img src="images\image-20231115191809792.png" alt="image-20231115191809792" style="zoom:80%;" />

 **（2） 构建结果** 

​    构建成功后，在 Jenkins 中可以看到增加了新的镜像。 

```bash
[root@ccjenkins my_hellojks]# docker images
REPOSITORY                     TAG           IMAGE ID       CREATED              SIZE
192.168.110.124/jks/hellojks   v1.0.0        20d4feb2ffa2   About a minute ago   659MB
hellojks                       v1.0.0        20d4feb2ffa2   About a minute ago   659MB
hellojks                       latest        8c365585d136   5 hours ago          659MB
192.168.110.124/jks/hellojks   latest        8c365585d136   5 hours ago          659MB
jenkins/jenkins                2.387.1-lts   d5ed2ceef0ec   8 months ago         471MB
openjdk                        8u102         ca5dd051db43   7 years ago          641MB
```

​    Harbor 中新增了 v1.0.0 的镜像。 

<img src="images\image-20231115191832427.png" alt="image-20231115191832427" style="zoom:80%;" />

​    在目标服务器上新增了 v1.0.0 的镜像，且该容器也运行了起来。

```bash
[root@cctarget bin]# docker images
REPOSITORY                     TAG       IMAGE ID       CREATED          SIZE
192.168.110.124/jks/hellojks   v1.0.0    20d4feb2ffa2   25 seconds ago   659MB
192.168.110.124/jks/hellojks   latest    8c365585d136   5 hours ago      659MB
hellojks                       latest    4a6b61ecfe16   29 hours ago     659MB
openjdk                        8u102     ca5dd051db43   7 years ago      641MB
```

```bash
[root@cctarget bin]# docker ps
CONTAINER ID   IMAGE                                 COMMAND                   CREATED         STATUS         PORTS                                       NAMES
8028cef63bec   192.168.110.124/jks/hellojks:v1.0.0   "java -jar hellojks.…"   2 minutes ago   Up 2 minutes   0.0.0.0:8081->8080/tcp, :::8081->8080/tcp   hellojks
```

​    在浏览器上访问到的页面内容也是 v1.0.0 的内容了。 

192.168.110.125:8081/some

<img src="images\image-20231115191918099.png" alt="image-20231115191918099" style="zoom:80%;" />

**4.14.5.部署 v2.0.0** 

​    此时再选择 v2.0.0 进行构建。 

<img src="images\image-20231115192003544.png" alt="image-20231115192003544" style="zoom:80%;" />

​    构建成功后，在浏览器刷新即可看到 v2.0.0 版本的内容了。

192.168.110.125:8081/some

<img src="images\image-20231115192026187.png" alt="image-20231115192026187" style="zoom:80%;" />



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

