根据尚硅谷Docker教程补全 https://www.bilibili.com/video/BV1gr4y1U7CY

# 41.Docker 主从复制（Mysql 1主1从）

见Docker2022笔记 1.Docker复杂安装详说

# 45.Docker Swarm 集群部署（Redis 3主3从）

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



# 52.主从扩容

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

# 54.主从缩容

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



# Docker Compose 容器编排










# CI/CD之jenkins