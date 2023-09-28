# Docker安装

## 环境说明

我们使用的是 CentOS 7 (64-bit) 目前，CentOS 仅发行版本中的内核支持 Docker。 Docker 运行在 CentOS 7 上，要求系统为64位、系统内核版本为 3.10 以上。

**查看自己的内核**

`uname -r` 命令用于打印当前系统相关信息（内核版本号、硬件架构、主机名称和操作系统类型等）。

```bash
[root@hecs-233798 ~]# uname -r
3.10.0-1160.92.1.el7.x86_64
```

**查看版本信息**

`cat /etc/os-release`

```bash
[root@hecs-233798 ~]# cat /etc/os-release
NAME="CentOS Linux"
VERSION="7 (Core)"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="7"
PRETTY_NAME="CentOS Linux 7 (Core)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:centos:centos:7"
HOME_URL="https://www.centos.org/"
BUG_REPORT_URL="https://bugs.centos.org/"

CENTOS_MANTISBT_PROJECT="CentOS-7"
CENTOS_MANTISBT_PROJECT_VERSION="7"
REDHAT_SUPPORT_PRODUCT="centos"
REDHAT_SUPPORT_PRODUCT_VERSION="7"
```

## 安装步骤

1、官网安装参考手册：<https://docs.docker.com/engine/install/centos/> 2、确定你是CentOS7及以上版本，我们已经做过了 3、yum安装gcc相关环境（需要确保 虚拟机可以上外网 ）

```bash
yum -y install gcc
yum -y install gcc-c++
```

4、卸载旧版本

```bash
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

5、安装需要的软件包

```bash
yum install -y yum-utils
```

6、设置镜像仓库

```bash
yum-config-manager --add-repo
http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

7、更新yum软件包索引

```bash
yum makecache fast
```

8、安装 Docker Engine

```bash
yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

9、启动 Docker

```bash
systemctl start docker
```

10、测试命令

```bash
docker version
docker run hello-world
docker images
```

```bash
[root@hecs-233798 ~]# docker images
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
hello-world   latest    9c7a54a9a43c   4 months ago   13.3kB
```

11、卸载 Docker Engine

```bash
# 1.卸载 Docker Engine、CLI、containerd 和 Docker Compose 软件包：
yum remove docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras
# 2.主机上的映像、容器、卷或自定义配置文件不会自动删除。要删除所有映像、容器和卷：
rm -rf /var/lib/docker
rm -rf /var/lib/containerd
```

# Docker 常用命令

## 帮助命令

```bash
docker version			# 版本信息
docker info				# 系统信息，包括镜像和容器的数量
docker <命令> --help  	# 万能命令
```

命令帮助文档查询：<https://docs.docker.com/engine/reference/commandline/docker/>

## 镜像命令

`docker images`

```bash
# 列出本地主机上的镜像
[root@hecs-233798 ~]# docker images
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
hello-world   latest    9c7a54a9a43c   4 months ago   13.3kB
# 解释
REPOSITORY 	镜像的仓库源
TAG 		镜像的标签
IMAGE ID 	镜像的ID
CREATED 	镜像创建时间
SIZE 		镜像大小
# 同一个仓库源可以有多个 TAG，代表这个仓库源的不同版本，我们使用REPOSITORY：TAG 定义不同
的镜像，如果你不定义镜像的标签版本，docker将默认使用 lastest 镜像！
# 可选项
-a： 列出本地所有镜像
-q： 只显示镜像id
--digests： 显示镜像的摘要信息
```

`docker search`

```bash
# 搜索镜像
NAME                            DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
mysql                           MySQL is a widely used, open-source relation…   14446     [OK]       
mariadb                         MariaDB Server is a high performing open sou…   5512      [OK]       
percona                         Percona Server is a fork of the MySQL relati…   619       [OK] 
# docker search 某个镜像的名称 对应DockerHub仓库中的镜像
# 可选项
--filter=stars=3000 ： 列出收藏数不小于指定值的镜像。
[root@hecs-233798 ~]# docker search mysql --filter=STARS=3000
NAME      DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
mysql     MySQL is a widely used, open-source relation…   14446     [OK]       
mariadb   MariaDB Server is a high performing open sou…   5512      [OK]
```

`docker pull`

```bash
# 下载镜像 docker pull 镜像名[:tag]
[root@hecs-233798 ~]# docker pull mysql
Using default tag: latest				#如果不写tag，默认就是latest
latest: Pulling from library/mysql
b193354265ba: Pull complete 			#分层下载，docker image的核心 联合文件系统
14a15c0bb358: Pull complete 
02da291ad1e4: Pull complete 
9a89a1d664ee: Pull complete 
a24ae6513051: Pull complete 
b85424247193: Pull complete 
9a240a3b3d51: Pull complete 
8bf57120f71f: Pull complete 
c64090e82a0b: Pull complete 
af7c7515d542: Pull complete 
Digest: sha256:c0455ac041844b5e65cd08571387fa5b50ab2a6179557fd938298cab13acf0dd		#签名
Status: Downloaded newer image for mysql:latest
docker.io/library/mysql:latest			#真实地址
# 等价命令
docker pull mysql
docker pull docker.io/library/mysql:latest
# 指定版本下载
[root@kuangshen ~]# docker pull mysql:5.7
....
```

`docker rmi`

```bash
# 删除镜像
docker rmi -f 镜像id 				# 删除单个
docker rmi -f 镜像名:tag 镜像名:tag 	# 删除多个
docker rmi -f $(docker images -aq) 	# 删除全部
```

## 容器命令

**说明**：有镜像才能创建容器，我们这里使用 centos 的镜像来测试，就是虚拟一个 centos ！

`docker pull centos`

```bash
# 命令
docker run [可选参数] image [COMMAND][ARG...]
# 常用参数说明
--name="Name" 	# 给容器指定一个名字
-d 				# 后台方式运行容器，并返回容器的id！
-it 			# 以交互模式运行容器，进入容器查看内容
-P 				# 随机端口映射（大写）
-p 				# 指定端口映射（小写），一般可以有四种写法
	-p ip:[主机端口]:[容器端口]
	-p [容器端口]
	-p [主机端口]:[容器端口] (常用)
	[容器端口]
# 测试
[root@hecs-233798 ~]# docker run -it centos /bin/bash
[root@28a9d886b3a6 /]# ls			# 注意地址，已经切换到容器内部了,命令不完善！
bin  dev  etc  home  lib  lib64  lost+found  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var

[root@28a9d886b3a6 /]# exit		# 使用 exit 退出容器

exit
[root@hecs-233798 ~]# 

```

**列出所有运行的容器**

```bash
# docker ps 命令
		# 列出当前所有正在运行的容器
-a 		# 列出当前所有正在运行的容器 + 历史运行过的容器
-l 		# 显示最近创建的容器
-n=? 	# 显示最近n个创建的容器
-q 		# 静默模式，只显示容器编号。

[root@hecs-233798 /]# docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
[root@hecs-233798 /]# docker ps -a
CONTAINER ID   IMAGE          COMMAND       CREATED         STATUS                     PORTS     NAMES
28a9d886b3a6   centos         "/bin/bash"   8 minutes ago   Exited (0) 4 minutes ago             blissful_kirch
f9dcb98345e4   9c7a54a9a43c   "/hello"      22 hours ago    Exited (0) 22 hours ago              amazing_lamarr

```

**退出容器**

```bash
exit # 容器停止退出
ctrl + P + Q # 容器不停止退出
```

**删除容器**

```bash
docker rm 容器id 				# 删除指定容器
docker rm -f $(docker ps -a -q) # 删除所有容器
docker ps -a -q|xargs docker rm # 删除所有容器
```

**启动停止容器**

```bash
docker start (容器id or 容器名) 		# 启动容器
docker restart (容器id or 容器名) 	# 重启容器
docker stop (容器id or 容器名) 		# 停止容器
docker kill (容器id or 容器名) 		# 强制停止容器
```

## 常用其他命令

**后台启动容器**

```bash
# 命令
docker run -d 容器名
# 例子
docker run -d centos # 启动centos，使用后台方式启动
# 问题： 使用docker ps 查看，发现容器已经退出了！
# 解释：Docker容器后台运行，就必须有一个前台进程，容器运行的命令如果不是那些一直挂起的命令，就会自动退出。
# 比如，你运行了nginx服务，但是docker前台没有运行应用，这种情况下，容器启动后，会立即自杀，因为他觉得没有程序了，所以最好的情况是，将你的应用使用前台进程的方式运行启动。
```

**查看日志**

```bash
# 命令
docker logs -tf --tail [条数] [容器id]
# 例子：我们启动 centos，并编写一段脚本来测试玩玩！最后查看日志

[root@hecs-233798 /]# docker run -d centos /bin/bash -c "while true;do echo haha wocao;sleep 1;done"
f461d8de49c56373d4c18066eba1b06339a3d5728f5666a71e826744f8de05fb
[root@hecs-233798 /]# docker logs -tf --tail 10 f461d8de49c5
2023-09-14T08:08:11.392323059Z haha wocao
2023-09-14T08:08:12.393936117Z haha wocao
2023-09-14T08:08:13.395604600Z haha wocao
2023-09-14T08:08:14.397269803Z haha wocao
2023-09-14T08:08:15.398940267Z haha wocao
[root@hecs-233798 /]# docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS     NAMES
f461d8de49c5   centos    "/bin/bash -c 'while…"   50 seconds ago   Up 50 seconds             infallible_ritchie
```

**查看容器中运行的进程信息，支持 ps 命令参数。**

```bash
# 命令
docker top [容器id]
# 测试
[root@hecs-233798 /]# docker top f461d8de49c5
UID          PID          PPID          C          STIME        TTY       TIME          CMD
root         23737        23719         0          16:06        ?         00:00:00      /bin/bash -c while ....
root         25036        23737         0          16:26        ?         00:00:00      /usr/bin/coreutils ....
```

**查看容器/镜像的元数据**

```bash
# 命令
docker inspect [容器id]
# 测试
[root@hecs-233798 /]# docker inspect f461d8de49c5
[
    {
        "Id": "f461d8de49c56373d4c18066eba1b06339a3d5728f5666a71e826744f8de05fb",
        "Created": "2023-09-14T08:06:42.060322863Z",
        "Path": "/bin/bash",
        "Args": [
            "-c",
            "while true;do echo haha wocao;sleep 1;done"
        ],
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 23737,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2023-09-14T08:06:42.246113856Z",
            "FinishedAt": "0001-01-01T00:00:00Z"
        },
....
```

**进入正在运行的容器**

```bash
# 命令1
docker exec -it [容器id] /bin/bash
# 测试1
[root@hecs-233798 /]# 
[root@hecs-233798 /]# docker exec -it f461d8de49c5 /bin/bash
[root@f461d8de49c5 /]# ls
bin  dev  etc  home  lib  lib64  lost+found  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var

# 命令2
docker attach [容器id]
# 测试2
[root@hecs-233798 /]# docker attach f461d8de49c5
haha wocao
haha wocao

# 区别
# exec 是在容器中打开新的终端，并且可以启动新的进程,以 exit 不终止容器退出
# attach 直接进入容器启动命令的终端，不会启动新的进程，以 Ctrl + p + q 不终止容器退出
# 因此，如果想直接在终端中查看启动命令的输出，可使用 attach，否则使用 exec。
# 但实际生产中，看启动输出，一般我们是通过 docker logs -f [容器id] 命令
```

**从容器内拷贝文件到主机上**

```bash
# 命令
docker cp [容器id]:[容器内路径] [目的主机路径]
# 测试
# 容器内执行，创建一个文件测试
[root@hecs-233798 home]# docker run -it centos /bin/bash
[root@5402e32aaec0 /]# cd /home
[root@5402e32aaec0 home]# touch testcp 
[root@5402e32aaec0 home]# ls
testcp
[root@5402e32aaec0 home]# exit
exit
# 容器被关闭
[root@hecs-233798 home]# docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
# 在主机操作，将文件拷贝到主机
[root@hecs-233798 home]# docker cp 5402e32aaec0:/home/testcp /home
Successfully copied 1.54kB to /home
[root@hecs-233798 home]# ls
testcp

```

**总结常用命令**

```bash
attach Attach to a running container # 当前 shell 下attach 连接指定运行镜像
build Build an image from a Dockerfile # 通过 Dockerfile 定制镜像
commit Create a new image from a container changes # 提交当前容器为新的镜像
cp Copy files/folders from the containers filesystem to the host path #从容器中拷贝指定文件或者目录到宿主机中
create Create a new container # 创建一个新的容器，同run，但不启动容器
diff Inspect changes on a container's filesystem # 查看 docker 容器变化
events Get real time events from the server # 从 docker 服务获取容器实时事件
exec Run a command in an existing container # 在已存在的容器上运行命令
export Stream the contents of a container as a tar archive # 导出容器的内容流作为一个 tar 归档文件[对应 import ]
history Show the history of an image # 展示一个镜像形成历史
images List images # 列出系统当前镜像
import Create a new filesystem image from the contents of a tarball # 从tar包中的内容创建一个新的文件系统映像[对应export]
info Display system-wide information # 显示系统相关信息
inspect Return low-level information on a container # 查看容器详细信息
kill Kill a running container # kill 指定 docker 容器
load Load an image from a tar archive # 从一个 tar 包中加载一个镜像[对应 save]
login Register or Login to the docker registry server # 注册或者登陆一个docker 源服务器
logout Log out from a Docker registry server # 从当前 Dockerregistry 退出
logs Fetch the logs of a container # 输出当前容器日志信息
port Lookup the public-facing port which is NAT-ed to PRIVATE_PORT #查看映射端口对应的容器内部源端口
pause Pause all processes within a container # 暂停容器
ps List containers # 列出容器列表
pull Pull an image or a repository from the docker registry server #从docker镜像源服务器拉取指定镜像或者库镜像
push Push an image or a repository to the docker registry server #推送指定镜像或者库镜像至docker源服务器
restart Restart a running container # 重启运行的容器
rm Remove one or more containers # 移除一个或者多个容器
rmi Remove one or more images # 移除一个或多个镜像[无容器使用该镜像才可删除，否则需删除相关容器才可继续或 -f 强制删除]
run Run a command in a new container # 创建一个新的容器并运行一个命令
save Save an image to a tar archive # 保存一个镜像为一个tar 包[对应 load]
search Search for an image on the Docker Hub # 在 docker hub 中搜索镜像
start Start a stopped containers # 启动容器
stop Stop a running containers # 停止容器
tag Tag an image into a repository # 给源中镜像打标签
top Lookup the running processes of a container # 查看容器中运行的进程信息
unpause Unpause a paused container # 取消暂停容器
version Show the docker version information # 查看 docker 版本号
wait Block until a container stops, then print its exit code # 截取容器停止时的退出状态值
```

## 作业练习

> 使用 Docker 安装 Nginx

```bash
# 1、搜索镜像 search 建议去dockerhub搜索 可以看到帮助文档
[root@hecs-233798 ~]# docker search nginx
NAME              DESCRIPTION                STARS     OFFICIAL   AUTOMATED
nginx             Official build of Nginx.   19002     [OK]      

# 2、下载镜像 pull 
[root@hecs-233798 ~]# docker pull nginx:1.25.2
1.25.2: Pulling from library/nginx
360eba32fa65: Pull complete 
c5903f3678a7: Pull complete 
27e923fb52d3: Pull complete 
72de7d1ce3a4: Pull complete 
94f34d60e454: Pull complete 
e42dcfe1730b: Pull complete 
907d1bb4e931: Pull complete 
Digest: sha256:6926dd802f40e5e7257fded83e0d8030039642e4e10c4a98a6478e9c6fe06153
Status: Downloaded newer image for nginx:1.25.2
docker.io/library/nginx:1.25.2

# 3、运行测试
[root@hecs-233798 ~]# docker images
REPOSITORY   TAG       IMAGE ID       CREATED         SIZE
nginx        1.25.2    f5a6b296b8a2   7 days ago      187MB
centos       latest    5d0da3dc9764   24 months ago   231MB

# -d 后台运行
# --name [起个容器名]
# -p [宿主机port]:[容器port]
[root@hecs-233798 ~]# docker run -d --name nginx01 -p 3344:80 nginx:1.25.2
94ff9f21ec6884641947842e7d26f5ec5b7b2cf65ec20c7309665b2cce865f31
[root@hecs-233798 ~]# docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                                   NAMES
94ff9f21ec68   nginx:1.25.2   "/docker-entrypoint.…"   26 seconds ago   Up 25 seconds   0.0.0.0:3344->80/tcp, :::3344->80/tcp   nginx01
[root@hecs-233798 ~]# curl localhost:3344
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

# 浏览器访问Nginx
http://114.115.167.245:3344/
# 进入容器
[root@hecs-233798 ~]# docker exec -it nginx01 /bin/bash
root@94ff9f21ec68:/# whereis nginx
nginx: /usr/sbin/nginx /usr/lib/nginx /etc/nginx /usr/share/nginx
root@94ff9f21ec68:/# cd /etc/nginx
root@94ff9f21ec68:/etc/nginx# ls
conf.d	fastcgi_params	mime.types  modules  nginx.conf  scgi_params  uwsgi_params
root@94ff9f21ec68:/etc/nginx# 

# 关闭容器
[root@hecs-233798 ~]# docker stop nginx01
nginx01
# 浏览器再次访问Nginx
404

#总结 
1、启动Nginx镜像的时候，Docker使用宿主机的内核虚拟了一个微型debian的系统，在系统里运行了Nginx
2、外部浏览器访问的公网服务器 http://114.115.167.245:3344/
由于在114.115.167.245运行Nginx容器时 
docker run -d --name nginx01 -p 3344:80 nginx:1.25.2 
此命令将服务器localhost的3344端口映射到容器内（debian）的80（Nginx）端口，
所以浏览器可以访问Nginx
```

> 使用 Docker 安装 Tomcat

```bash
# 官方文档使用
docker run -it --rm tomcat:9.0
# 之前的案例都是后台启动，停止之后容器还存在。使用上述方式是测试用的，用完即删。
# 常规方式下载
docker pull tomcat:9.0
# 启动运行
docker run -d -p 3355:8080 --name tomcat01 tomcat:9.0
# 测试访问没有问题（虽然404，但是有tomcat版本号）
# 进入容器
docker exec -it tomcat01 /bin/bash
#发现问题 1、命令少了 2、没有webapps 镜像的原因，默认安装最小镜像，所有不必要的全都剔除了。
#保证最小可运行的环境！
```

思考：我们以后要部署项目，还需要进入容器中，是不是十分麻烦，要是有一种技术，可以将容器 内和我们Linux进行映射挂载就好了？我们后面会将数据卷技术来进行挂载操作，也是一个核心内容，这 里大家先听听名词就好，我们很快就会讲到！

> 使用docker 部署 es + kibana

```bash
# es 暴露的端口很多
# es 十分耗内存
# es 的数据一般要放置到安全目录！挂载
# --net somenetwork 网络配置
docker run -d --name elasticsearch -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" elasticsearch:7.6.2

# 启动后会很卡
# 使用 docker stats 命令查看cpu状态
# 测试一下是否成功
[root@hecs-233798 ~]# curl localhost:9200
{
  "name" : "c4608b36e954",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "laHZb2d4SvSEm2fnEC0YoA",
  "version" : {
    "number" : "7.6.2",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "ef48eb35cf30adf4db14086e8aabd07ef6fb113f",
    "build_date" : "2020-03-26T06:34:37.794943Z",
    "build_snapshot" : false,
    "lucene_version" : "8.4.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
# 赶紧停掉，增加内存限制，修改配置文件 -e 环境配置修改
docker run -d --name elasticsearch02 -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" -e ES_JAVA_OPTS="-Xms64m -Xmx512m" elasticsearch:7.6.2
# 再次测试 curl localhost:9200 效果一样！
```

思考：如果我们要使用 kibana , 如果配置连接上我们的es呢？网络该如何配置呢？

## 可视化

*   Portainer（先用这个）

```bash
docker run -d -p 8088:9000 \
--restart=always -v /var/run/docker.sock:/var/run/docker.sock --privileged=true portainer/portainer
```

*   Rancher（CI/CD再用这个） 什么是Portainer? Docker 图形化界面管理工具，提供后台面板供我们操作。 访问方式：<http://服务器IP:8088>

# Docker镜像讲解

***

原理图详见 狂神说Docker笔记

## 镜像Commit

***

**docker commit 从容器创建一个新的镜像。**

```bash
docker commit 提交容器副本使之成为一个新的镜像！
# 语法
docker commit -m="[提交的描述]" -a="[作者]" [容器id] [镜像名]:[TAG]
```

实战测试

```bash
# 1、启动一个默认的tomcat

# 2、这个tomcat没有webapps（默认没有）

# 3、自己拷贝一个app到webapps

# 4、使用commit提交成为自己方便使用的镜像！
[root@hecs-233798 ~]# docker commit -a "zhanmiao" -m "move.dist to webapps" 3d3d60eabbcc tomcat02:1.0
# 如果你想要保存你当前的状态，可以通过commit，来提交镜像，方便使用，类似于 VM 中的快照！
```

# 容器数据卷

## 什么是容器数据卷

Docker原理回顾 如果在容器中存放了一些数据。那么当没有commit并删除容器的时候，数据就会丢失！ **需求：数据持久化** **需求：MySql容器，想把数据同步到宿主机** 容器之间有一个数据共享的技术，将Docker容器中产生的数据同步到本地！ 使用卷技术！将容器中的目录挂载到本地宿主机！ **总结：容器的持久化和同步。容器间数据共享。**

## 使用数据卷

> 方式一：容器中直接使用命令来挂载

```bash
# 命令
docker run -it -v [宿主机路径]:[容器内路径] [镜像名]
# 测试
[root@hecs-233798 home]# docker run -it -v /home/ostest:/home/test centos /bin/bash
# 启动后可以通过inspect命令来检测是否正确挂载数据卷 docker inspect [容器id]
"Mounts": [
            {
                "Type": "bind",
                "Source": "/home/ostest",
                "Destination": "/home/test",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            }
        ],

# 正确挂载后，两个目录为双向绑定，效果等同于同一个目录
# 并且关闭容器后，修改宿主机目录，启动容器后也会自动更新到容器
```

> 使用 Docker 安装 MySql 思考：mysql 数据持久化的问题！

```bash
# 1、拉取镜像
docker pull mysql:5.7
# 2、启动
-d 后台启动
-p 端口映射
-v 数据卷挂载
-e 配置环境变量
--name 容器名
docker run -d -p 3306:3306 -v /home/mysql/conf:/etc/mysql/conf.d -v /home/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 --name mysql01 mysql:5.7
# 启动后使用本地的sqlyog测试连接
sqlyog--连接到服务器3306--连接容器3306映射，这时候连接就成功了
# 3、sqlyog新建一个test库 CREATE DATABASE test;
进入/home/mysql/data ls发现多出test文件夹
# 4、删除MySql容器
[root@hecs-233798 mysql]# docker rm -f mysql01
mysql01
[root@hecs-233798 mysql]# ls data
auto.cnf    ca.pem           client-key.pem  ibdata1      ib_logfile1  mysql       performance_schema  public_key.pem   server-key.pem  test
ca-key.pem  client-cert.pem  ib_buffer_pool  ib_logfile0  ibtmp1       mysql.sock  private_key.pem     server-cert.pem  sys
#可以看到我们刚才的创建的数据，实现了持久化。
```

## 匿名和具名挂载

```bash
# 匿名挂载
-v 容器内路径
docker run -P -d --name nginx02 -v /etc/nginx nginx:1.25.2
# 匿名挂载的缺点，就是不好维护，通常使用命令 docker volume维护
docker volume ls 
#查看所有volume情况
[root@hecs-233798 mysql]# docker volume ls
DRIVER    VOLUME NAME
local     ff12026405f177d2dde4e953c6593ea23ccfa1ba11c32fd785e802c81a3a5f0a
# 具名挂载
# 通过 -v [卷名]:[容器内路径] 挂载
[root@hecs-233798 mysql]# docker run -d -P --name nginx03 -v juming:/etc/nginx nginx:1.25.2
1bfe5a4e8fcf8e1feb55b4c79e4762be3f71611bfa1c537e1008936346b0ce5d
[root@hecs-233798 mysql]# docker volume ls
DRIVER    VOLUME NAME
local     ff12026405f177d2dde4e953c6593ea23ccfa1ba11c32fd785e802c81a3a5f0a
local     juming
# 查看一下这个卷的信息 
# docker volume inspect [卷名]
[root@hecs-233798 mysql]# docker volume inspect juming
[
    {
        "CreatedAt": "2023-09-19T16:46:37+08:00",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/juming/_data",	# 宿主机路径
        "Name": "juming",
        "Options": null,
        "Scope": "local"
    }
]
```

所有的Docker容器内的卷，没有指定本地路径的情况下都是在 `/var/lib/docker/volumes/[卷名]/_data`

通过具名挂载，我们可以方便的找到卷的路径，所以建议使用**具名挂载**

```bash
# 如何区分是具名挂载/匿名挂载？
-v [容器内路径]					#匿名挂载
-v [卷名]:[容器内路径]			#具名挂载 建议
-v [宿主机路径]:[容器内路径]		#指定路径挂载
# docker volume inspect xxx 可以查看匿名挂载/具名挂载

# 拓展
# 改变文件的读写权限
ro: readonly		#只读
rw: readwrite		#读写
# 指定容器对我们挂载出来的内容的读写权限
docker run -d -P --name nginx02 -v nginxconfig:/etc/nginx:ro nginx
docker run -d -P --name nginx02 -v nginxconfig:/etc/nginx:rw nginx
# ro 只要看到ro 说明这个路径只能通过宿主机操作，容器内无法操作！
```

## 初识DockerFile

DockerFile 是用来构建Docker镜像的构建文件，是由一些列命令和参数构成的脚本。&#x20;

我们在这里，先体验下，后面我们会详细讲解 DockerFile ！

```bash
# touch 一个dockerFile 文件，建议dockerFile
# 文件内容 [大写指令] [参数]
[root@hecs-233798 test_volume]# pwd
/home/test_volume
[root@hecs-233798 test_volume]# touch dockerFile1
[root@hecs-233798 test_volume]# cat dockerFile1 
FROM centos

VOLUME ["/volume01","/volume02"]	# 匿名挂载

CMD echo "----finish----"

CMD /bin/bash
# 这里每个命令都是即将创建的镜像的一层
# 构建自己的镜像
[root@hecs-233798 test_volume]# docker build -f /home/test_volume/dockerFile1 -t zhanmiao/centos:1.0 .
[+] Building 0.2s (5/5) FINISHED   
# 启动自己的容器 
[root@hecs-233798 test_volume]# docker run -it zhanmiao/centos:1.0 /bin/bash
[root@cbd6e5ca0adf /]# ls
bin  dev  etc  home  lib  lib64  lost+found  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var  volume01	volume02
```

发现刚才dockerFile1 中挂载的volume01 volume02！

这两个卷一定指向外部宿主机的两个路径。

查看卷挂载的路径&#x20;

```bash
[root@hecs-233798 /]# docker inspect f087cd98fc10
"Mounts": [
            {
                "Type": "volume",
                "Name": "25d48febea50b87e906f575a15d98bdffc7823ac0402d61c9cc9532cd149e46e",
                "Source": "/var/lib/docker/volumes/25d48febea50b87e906f575a15d98bdffc7823ac0402d61c9cc9532cd149e46e/_data",
                "Destination": "/volume01",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            },
            {
                "Type": "volume",
                "Name": "0fad2f575a8b9fefa32061a52bef330c1215a8f0f555a00906498a228958d2c3",
                "Source": "/var/lib/docker/volumes/0fad2f575a8b9fefa32061a52bef330c1215a8f0f555a00906498a228958d2c3/_data",
                "Destination": "/volume02",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }
        ],

```

在容器内 volume01 创建一个文件 container.txt

测试是否同步？成功。

```bash
[root@hecs-233798 /]# cd /var/lib/docker/volumes/25d48febea50b87e906f575a15d98bdffc7823ac0402d61c9cc9532cd149e46e/_data
[root@hecs-233798 _data]# cat container.txt 
[root@hecs-233798 _data]# ls
container.txt
```

未来我们会经常使用这种方式，因为我们通常构建自己的镜像。

加入构建镜像的时候没有挂载卷，要手动具名挂载 -v \[卷名]:\[容器内路径]

出于可移植和分享的考虑，我们之前使用的 -v 主机目录:容器目录 这种方式不能够直接在 DockerFile中实现。&#x20;

由于宿主机目录是依赖于特定宿主机的，并不能够保证在所有宿主机上都存在这样的特定目录.

## 数据卷容器

命名的容器挂载数据卷，其他容器通过挂载这个（父容器）实现数据共享，挂载数据卷的容器，称之为 数据卷容器。&#x20;

我们使用上一步的镜像：zhanmiao/centos:1.0 为模板，运行容器 volmaster，volslave，voldidi，他们都会具有容器卷&#x20;

`volume01 volume02`

我们来测试下，容器间传递共享

1、先启动一个父容器volmaster，然后在volume01文件夹下新增文件

```bash
[root@hecs-233798 test_volume]# docker run -it zhanmiao/centos:1.0 /bin/bash
[root@6dd56e3cdbae /]# ls
bin  ....  sys  tmp  usr  var  volume01	volume02
[root@6dd56e3cdbae /]# cd volume01
[root@6dd56e3cdbae volume01]# ls
[root@6dd56e3cdbae volume01]# touch immaster.txt
```

退出不停止：ctrl+P+Q

2、创建volslave，voldidi让他们继承 volmaster `--volumes-from`

```bash
# 创建容器 volslave 继承 volmaster 的数据卷
[root@hecs-233798 home]# docker run -it --name volslave --volumes-from volmaster zhanmiao/centos:1.0
[root@73a584d26624 /]# cd volume01
[root@73a584d26624 volume01]# ls
immaster.txt
[root@73a584d26624 volume01]# touch imslave.txt
[root@73a584d26624 volume01]# ls
immaster.txt  imslave.txt
# 创建容器 voldidi 继承 volmaster 的数据卷
[root@hecs-233798 home]# docker run -it --name voldidi --volumes-from volmaster zhanmiao/centos:1.0
[root@ef8a8b8264f6 /]# ls 
bin  ....  sys  tmp  usr  var  volume01	volume02
[root@ef8a8b8264f6 /]# cd volume01
[root@ef8a8b8264f6 volume01]# ls
immaster.txt  imslave.txt
[root@ef8a8b8264f6 volume01]# touch imdidi.txt
[root@ef8a8b8264f6 volume01]# ls
imdidi.txt  immaster.txt  imslave.txt
```

3、回到 volmaster 发现可以看到 volslave 和 voldidi 添加的共享文件

```bash
[root@hecs-233798 home]# docker attach volmaster 
[root@6dd56e3cdbae volume01]# ls
imdidi.txt  immaster.txt  imslave.txt
```

4、删除volmaster，volslave 修改后 voldidi 还能不能访问?可以！

5、结论

*   当 `volmaster` 以镜像DockerFile方式启动，会自动配置成 **带有数据卷的容器** 。

    作为父容器，供其他子容器 `volslave` `voldidi` 以 `--volume-from` 方式启动，会共享数据卷（默认RW）。
*   共享数据卷的效果是（默认RW），在任意一个容器的数据卷里修改文件，无论父/子容器的数据卷中的文件都会随之改变。

6、原理

通过 `docker inspect volmaster` `docker inspect volslave` `docker inspect voldidi` 三个命令会发现：

这三个共享数据卷的容器内 "/volume01"，"/volume02"路径全都指向宿主机的实际物理路径&#x20;

"/volume01"-->"69d4233decf877b6347741392d71f659fc2ef0c25fd708e724022dad631d4e7d",

"/volume02"-->"605c7c774b9b4aab044af57a6ef7d6b86dae744b56c98f8fbecd0d610bc5d3ae"&#x20;

*   本例使用 "RW": true 的挂载方式，修改其中任何一个容器的文件，就是修改 \*\*宿主机物理路径 \*\*下的文件。所以其他容器卷的数据也会有更新的效果。
*   无论挂载容器数据卷顺序是怎样，其实都是映射到 **宿主机物理路径** ，所以即使删除了父容器/子容器，其余容器的映射也会保持。且文件仍然保存在 **宿主机物理路径** 下不会丢失。

```bash
[root@hecs-233798 ~]# docker inspect volmaster
"Mounts": [
            {
                "Type": "volume",
                "Name": "69d4233decf877b6347741392d71f659fc2ef0c25fd708e724022dad631d4e7d",
                "Source": "/var/lib/docker/volumes/69d4233decf877b6347741392d71f659fc2ef0c25fd708e724022dad631d4e7d/_data",
                "Destination": "/volume01",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            },
            {
                "Type": "volume",
                "Name": "605c7c774b9b4aab044af57a6ef7d6b86dae744b56c98f8fbecd0d610bc5d3ae",
                "Source": "/var/lib/docker/volumes/605c7c774b9b4aab044af57a6ef7d6b86dae744b56c98f8fbecd0d610bc5d3ae/_data",
                "Destination": "/volume02",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }

# 删掉所有相关容器够，数据还在！
[root@hecs-233798 ~]# docker rm volmaster volslave voldidi
volmaster 
volslave 
voldidi
[root@hecs-233798 ~]# cd /var/lib/docker/volumes/605c7c774b9b4aab044af57a6ef7d6b86dae744b56c98f8fbecd0d610bc5d3ae/_data
[root@hecs-233798 69d4233decf877b6347741392d71f659fc2ef0c25fd708e724022dad631d4e7d]# cd _data/
[root@hecs-233798 _data]# ls
imdidi2.txt  imdidi.txt  immaster.txt  imslave.txt
```

# DockerFile

## DockerFile介绍

DockerFile是用来构建docker镜像的文件，就是个命令参数脚本。

构建步骤：

1、编写一个DockerFile文件

2、docker build 构建成为一个镜像

3、docker run 运行镜像

4、docker push 发布镜像（GitHub、阿里云镜像仓库）

官方很多镜像都是基础包，只保留一些基本功能，我们通常要搭建自己的镜像。

官方既然自己可以制作镜像，那我们也可以。

## DockerFile构建过程

**基础知识：**

1、每条保留字（指令）都必须为大写字母，后面要跟随至少一个参数

2、指令按照从上到下，顺序执行&#x20;

3、# 表示注释&#x20;

4、每条指令都会创建一个新的镜像层，并对镜像进行提交

DockerFile是面向开发的，以后我们发布项目，做成docker镜像发布，就需要编写DockerFile文件，这个文件十分简单。

Docker镜像逐渐成为企业交付的标准，必须要掌握！

**步骤：开发、部署、运维**

DockerFile ：构建文件，定义了镜像一切生成的步骤（开发）

images ： 通过DockerFile构建生成的镜像，最终作为产品发布（部署）

容器：容器就是软件运行起来的状态（运维）

DockerFile 面向开发，Docker镜像成为交付标准，Docker容器则涉及部署与运维，三者缺一不可！

## DockerFile指令

我们用DockerFile指令，可以构建自己的镜像。

```bash
FROM 			# 基础镜像，当前新镜像是基于哪个镜像的
MAINTAINER 		# 镜像维护者的姓名混合邮箱地址
RUN 			# 容器构建时需要运行的命令
EXPOSE 			# 当前容器对外暴露出的端口
WORKDIR 		# 指定在创建容器后，终端默认登录的进来工作目录，一个落脚点
ENV 			# 用来在构建镜像过程中设置环境变量
ADD 			# 将宿主机目录下的文件拷贝进镜像且ADD命令会自动处理URL和解压tar压缩包
COPY 			# 类似ADD，拷贝文件和目录到镜像中！
VOLUME 			# 容器数据卷，用于数据保存和持久化工作
CMD 			# 指定这个容器启动时要运行的命令，dockerFile中可以有多个CMD指令，会被最后一个覆盖！
ENTRYPOINT 		# 指定这个容器启动时要运行的命令！可以追加命令
ONBUILD 		# 当构建一个被继承的DockerFile时运行命令，父镜像在被子镜像继承后，父镜像的ONBUILD被触发
```

## 实战测试

Docker Hub 中99% 的镜像都是通过在base镜像（Scratch）中安装和配置需要的软件构建出来的

```bash
FROM scratch
ADD centos-7-x86_64-docker.tar.xz /

LABEL \
    org.label-schema.schema-version="1.0" \
    org.label-schema.name="CentOS Base Image" \
    org.label-schema.vendor="CentOS" \
    org.label-schema.license="GPLv2" \
    org.label-schema.build-date="20201113" \
    org.opencontainers.image.title="CentOS Base Image" \
    org.opencontainers.image.vendor="CentOS" \
    org.opencontainers.image.licenses="GPL-2.0-only" \
    org.opencontainers.image.created="2020-11-13 00:00:00+00:00"

CMD ["/bin/bash"]
```

> 构建自己的centos

```bash
# 1、编写DockerFile文件
[root@hecs-233798 dockerfile]# cat df_centos 
FROM centos:7
MAINTAINER zhanmiao<357833687@qq.com>

ENV MYPATH /usr/local
WORKDIR $MYPATH

RUN yum -y install vim
RUN yum -y install net-tools

EXPOSE 80

CMD echo $MYPASH
CMD echo "----finish----"
CMD /bin/bash
```

```bash
# 2、通过这个文件构建镜像
docker build -f df_centos -t zhanmiaos:0.1 .
# docker build -f [dockerFile] -t [镜像名]:[版本号] .
# 3、测试运行
[root@hecs-233798 dockerfile]# docker run -it zhanmiaos:0.1
[root@36dc922d963c local]# 
# vim net-tools 已经正确安装
# 原生centos镜像是不带有这些软件的
[root@36dc922d963c local]# vim test
[root@36dc922d963c local]# 
[root@36dc922d963c local]# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.3  netmask 255.255.0.0  broadcast 172.17.255.255
....

# 4、列出镜像地的变更历史
[root@hecs-233798 /]# docker history zhanmiaos:0.1
IMAGE          CREATED          CREATED BY                                      SIZE      COMMENT
fb0f5e8cf6b7   14 minutes ago   CMD ["/bin/sh" "-c" "/bin/bash"]                0B        buildkit.dockerfile.v0
<missing>      14 minutes ago   CMD ["/bin/sh" "-c" "echo \"----finish----\"…   0B        buildkit.dockerfile.v0
<missing>      14 minutes ago   CMD ["/bin/sh" "-c" "echo $MYPASH"]             0B        buildkit.dockerfile.v0
<missing>      14 minutes ago   EXPOSE map[80/tcp:{}]                           0B        buildkit.dockerfile.v0
<missing>      14 minutes ago   RUN /bin/sh -c yum -y install net-tools # bu…   194MB     buildkit.dockerfile.v0
<missing>      14 minutes ago   RUN /bin/sh -c yum -y install vim # buildkit    280MB     buildkit.dockerfile.v0
<missing>      14 minutes ago   WORKDIR /usr/local                              0B        buildkit.dockerfile.v0
<missing>      14 minutes ago   ENV MYPATH=/usr/local                           0B        buildkit.dockerfile.v0
<missing>      14 minutes ago   MAINTAINER zhanmiao<357833687@qq.com>           0B        buildkit.dockerfile.v0
<missing>      2 years ago      /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B        
<missing>      2 years ago      /bin/sh -c #(nop)  LABEL org.label-schema.sc…   0B        
<missing>      2 years ago      /bin/sh -c #(nop) ADD file:b3ebbe8bd304723d4…   204MB     
```

当我们拿到一个镜像，就可以研究它是怎么做的了

> CMD 和 ENTRYPOINT 的区别

```bash
CMD 			# 指定这个容器启动时要运行的命令，dockerFile中可以有多个CMD指令，会被最后一个覆盖！
ENTRYPOINT 		# 指定这个容器启动时要运行的命令！可以追加命令
```

测试CMD

```bash
# 编写 DockerFile 文件
[root@hecs-233798 dockerfile]# vim df_cmd_test
FROM centos:7

CMD ["ls","-a"]
# 构建镜像
[root@hecs-233798 dockerfile]# docker build -f df_cmd_test -t cmd_test .
# 运行，发现ls -a生效了
[root@hecs-233798 dockerfile]# docker run 06f2cc65ea4a
.
..
.dockerenv
anaconda-post.log
bin
dev
etc
home
lib
# 想追加一个命令 -l 使原DockerFile中的拼接成 ls -a -l
[root@hecs-233798 dockerfile]#  docker run 06f2cc65ea4a -l
docker: Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: exec: "-l": executable file not found in $PATH: unknown.
# 问题：我们可以看到可执行文件找不到的报错，executable file not found。
# 之前我们说过，跟在镜像名后面的是 command，运行时会替换 CMD 的默认值。
# 因此这里的 -l 替换了原来的 CMD，而不是添加在原来的 ls -a 后面。而 -l 根本不是命令，所
以自然找不到。
# 那么如果我们希望加入 -l 这参数，我们就必须重新完整的输入这个命令：
docker run 06f2cc65ea4a ls -al
```

测试ENTRYPOINT

```bash
# 1、编写 DockerFile
[root@hecs-233798 dockerfile]# vim df_entrypoint_test
FROM centos:7

ENTRYPOINT ["ls","-a"]
# 2、构建镜像
[root@hecs-233798 dockerfile]# docker build -f df_entrypoint_test -t entrypoint_test .
[+] Building 2.1s (5/5) FINISHED                                                                                                                                                           
# 3、运行测试-l参数，发现可以直接使用，这里就是一种追加，我们可以明显的知道 CMD 和ENTRYPOINT 的区别了
[root@hecs-233798 dockerfile]# docker run 5184c7d459a0 -l
total 64
drwxr-xr-x  1 root root  4096 Sep 24 02:53 .
drwxr-xr-x  1 root root  4096 Sep 24 02:53 ..
-rwxr-xr-x  1 root root     0 Sep 24 02:53 .dockerenv
-rw-r--r--  1 root root 12114 Nov 13  2020 anaconda-post.log
lrwxrwxrwx  1 root root     7 Nov 13  2020 bin -> usr/bin
drwxr-xr-x  5 root root   340 Sep 24 02:53 dev
drwxr-xr-x  1 root root  4096 Sep 24 02:53 etc
....
```

## 实战：Tomcat 镜像

tips：上传的两种方式

*   使用rz/sz命令`yum -y install lrzsz`&#x20;

```bash
[root@hecs-233798 tomcat]# rz -bey	#使用二进制传输避免乱码
[root@hecs-233798 tomcat]# ls
jdk-8u381-linux-x64.tar.gz
```

*   本地 CMD 使用 scp 命令上传

```bash
C:\Users\syx>scp C:\Users\syx\Downloads\tomcat-9.0.80.tar.gz root@114.115.167.245:/home/zhanmiao/build/tomcat
root@114.115.167.245's password:
tomcat-9.0.80.tar.gz                                                                  100% 6135KB   2.4MB/s   00:02
#下载是 sz -r tomcat-9.0.80.tar.gz 
```

1、准备镜像文件、JDK压缩包、Tomcat压缩包

坑：JDK和Tomcat去官网下载，Tomcat要bin二进制的

```bash
/home/zhanmiao/build/tomcat
[root@hecs-233798 tomcat]# ll
total 142148
-rw-r--r-- 1 root root 139273048 Sep 24 16:47 jdk-8u381-linux-x64.tar.gz
-rw-r--r-- 1 root root   6282089 Sep 24 16:47 tomcat-9.0.80.tar.gz
```

2、 编写Dockerfile文件，官方命名`Dockerfile`，build 会自动寻找这个文件，就不需要 -f 指定了。

```bash
FROM centos:7
MAINTAINER zhanmiao<357833687@qq.com>

COPY readme.txt /usr/local/readme.txt

ADD jdk-8u381-linux-x64.tar.gz /usr/local/
ADD apache-tomcat-9.0.80.tar.gz /usr/local/

RUN yum -y install vim

ENV MYPATH /usr/local
WORKDIR $MYPATH

ENV JAVA_HOME /usr/local/jdk1.8.0_381
ENV CLASS_PATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV CATALINA_HOME /usr/local/apache-tomcat-9.0.80
ENV CATALINA_BASE /usr/local/apache-tomcat-9.0.80
ENV PATH $PATH:$JAVA_HOME/bin:$CATALINA_HOME/lib:$CATALINA_HOME/bin

EXPOSE 8080

CMD /usr/local/apache-tomcat-9.0.80/bin/startup.sh && tail -F /usr/local/apache-tomcat-9.0.80/bin/logs/catalina.out
```

3、构建镜像

`docker build -t diytomcat .`

4、启动容器

```bash
[root@hecs-233798 tomcat]# docker run -d -p 9090:8080 --name zhanmiaocat -v /home/zhanmiao/build/tomcat/test:/usr/local/apache-tomcat-9.0.80/webapps/test -v /home/zhanmiao/build/tomcat/tomcatlogs:/usr/local/apache-tomcat-9.0.80/logs diytomcat
7f71d72ac17f7183cb96c45299c6865bfc4de39ee00aa26d01dcc10748cdd724
[root@hecs-233798 tomcat]# ls
Dockerfile  jdk1.8.0_381  jdk-8u381-linux-x64.tar.gz  readme.txt  test  tomcat-9.0.80  tomcat-9.0.80.tar.gz  tomcatlogs
# 进入容器 docker exec -it 4da66bcdbc91 /bin/bash
```

5、验证访问 成功

`curl localhost:9090`

6、结合前面学习的容器卷将测试的web服务test发布：创建web.xml index.jsp

/usr/local/apache-tomcat-9.0.80/webapps/test/WEB-INF/web.xml

```bash
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xmlns="http://java.sun.com/xml/ns/javaee"
xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
id="WebApp_ID" version="2.5">
<display-name>test</display-name>
</web-app>
```

/usr/local/apache-tomcat-9.0.80/webapps/test/index.jsp

```bash
<%@ page language="java" contentType="text/html; charset=UTF-8"
pageEncoding="UTF-8"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN"
"http://www.w3.org/TR/html4/loose.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
<title>hello，kuangshen</title>
</head>
<body>
-----------welcome------------
<%=" my docker tomcat,zhanmiao coming "%>
<br>
<br>
<% System.out.println("-------my docker tomcat-------");%>
</body>
</html>
```

7、测试 ，也可直接访问 <http://114.115.167.245:9090/test/>

```bash
[root@hecs-233798 test]# curl localhost:9090/test/index.jsp

<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN"
"http://www.w3.org/TR/html4/loose.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
<title>hello，zhanmiao</title>
</head>
<body>
-----------welcome------------
 my docker tomcat,zhanmiao coming 
<br>
<br>

</body>
</html>
```

8、查看日志

```bash

[root@hecs-233798 tomcat]# cat tomcatlogs/catalina.out
-------my docker tomcat-------
-------my docker tomcat-------
-------my docker tomcat-------
-------my docker tomcat-------
```

## 发布镜像

> 发布到 Docker Hub

```bash
# 1、查看登录命令
[root@hecs-233798 ~]# docker login --help

Usage:  docker login [OPTIONS] [SERVER]

Log in to a registry.
If no server is specified, the default is defined by the daemon.

Options:
  -p, --password string   Password
      --password-stdin    Take the password from stdin
  -u, --username string   Username

# 2、登录
[root@hecs-233798 ~]# docker login -u sunyuxiao123@gmail.com
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
# 3、将镜像发布出去
[root@hecs-233798 ~]# docker push zhanmiao/diytomcat:1.0
The push refers to repository [docker.io/zhanmiao/diytomcat]
An image does not exist locally with the tag: zhanmiao/diytomcat
[root@hecs-233798 ~]# docker push diytomcat
Using default tag: latest
The push refers to repository [docker.io/library/diytomcat]
5f70bf18a086: Preparing 
01414f094b4c: Preparing 
5488b0afd4eb: Preparing 
88223b0f7ec8: Preparing 
65273d8baf31: Preparing 
174f56854903: Waiting 
# 拒绝：请求的资源访问被拒绝
denied: requested access to the resource is denied

# 问题：本地镜像名无帐号信息，解决加 tag即可
[root@hecs-233798 ~]# docker tag b77e5c5fde71 141415134141/diytomcat:1.0
# 再次 push， ok
[root@hecs-233798 ~]# docker push 141415134141/diytomcat:1.0
The push refers to repository [docker.io/141415134141/diytomcat]
5f70bf18a086: Pushed 
01414f094b4c: Pushing [====>                                              ]  25.12MB/279.7MB
5488b0afd4eb: Pushing [==========================>                        ]  8.706MB/16.29MB
88223b0f7ec8: Pushing [==>                                                ]  19.69MB/343.3MB
65273d8baf31: Pushed 
174f56854903: Pushing [==>                                                ]  11.45MB/203.9MB
```

> 发布到阿里云镜像

*   阿里云账号创建命名空间和容器

```bash
1、登录阿里云
2、找到容器镜像服务
3、创建命名空间
4、创建镜像仓库
5、点击进入这个镜像仓库，可以看到所有的信息
```

*   测试推送发布

```bash
#退出之前的登录状态
[root@hecs-233798 ~]# docker logout
Removing login credentials for https://index.docker.io/v1/
#1、 登录阿里云
[root@hecs-233798 ~]# docker login -u wanzhu66@outlook.com -p syx63346183 registry-intl.cn-qingdao.aliyuncs.com
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
# 2、设置 tag
docker tag [ImageId] registry-intl.cn-qingdao.aliyuncs.com/zhanmiao/docker:[镜像版本号]

[root@hecs-233798 ~]# docker tag 39ac5829bade registry-intl.cn-qingdao.aliyuncs.com/zhanmiao/miaoredis:0.10
[root@hecs-233798 ~]# docker images
REPOSITORY                                                 TAG       IMAGE ID       CREATED         SIZE
redis                                                      latest    39ac5829bade   2 weeks ago     138MB
zhanmiao/miaoredis                                         0.10      39ac5829bade   2 weeks ago     138MB
registry-intl.cn-qingdao.aliyuncs.com/zhanmiao/miaoredis   0.10      39ac5829bade   2 weeks ago     138MB

# 3、推送命令
docker push registry-intl.cn-qingdao.aliyuncs.com/zhanmiao/docker:[镜像版本号]

[root@hecs-233798 ~]# docker push registry-intl.cn-qingdao.aliyuncs.com/zhanmiao/miaoredis:0.10
The push refers to repository [registry-intl.cn-qingdao.aliyuncs.com/zhanmiao/miaoredis]
9274ee1aa5b0: Pushed 
705044e3d94c: Pushed 
074c89cb285f: Pushing [====>                                              ]   5.85MB/58.79MB
0d062699d8d0: Pushing [==================================>                ]  2.857MB/4.116MB
22cd9a63017e: Pushed 
a2d7501dfb35: Pushing [=>                                                 ]   2.19MB/74.76MB
# 完成后去阿里云镜像仓库查看效果
```

# Docker网络

## &#x20;理解Docker0

准备工作：清空所有的容器，清空所有的镜像

```bash
docker rm -f $(docker ps -aq) # 删除所有容器
docker rmi -f $(docker images -aq) # 删除全部镜像
```

> 我们先来做个测试

查看本地ip `ip addr`

```bash
[root@hecs-233798 ~]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether fa:16:3e:32:8a:e0 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.86/24 brd 192.168.0.255 scope global noprefixroute dynamic eth0
       valid_lft 52573sec preferred_lft 52573sec
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:77:f6:f1:9a brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
```

可以看到三个网络

```bash
lo		 127.0.0.1 		# 本机回环地址
eth0	 192.168.0.86 	# 华为云的私有IP
docker0	 172.17.0.1 	# docker网桥
# 问题：Docker 是如何处理容器网络访问的？
```

查看tomcat01的ip地址，docker会给每个容器都分配一个ip

```bash
这里如果出现OCI runtime exec failed，是容器默认最小安装导致的容器内部没有网络工具
更新apt-get ，apt 安装 iproute2 、net-tools 即可
`apt-get update` 					#apt更新
`apt install -y iproute2` 			#ip addr
`apt install -y net-tools` 			#ifconfig
`apt install -y iputils-ping`		#ping

[root@hecs-233798 ~]# docker run -d -P --name tomcat01 tomcat
# 查看容器的内部ip地址 ip addr
[root@hecs-233798 dockerfile]# docker exec -it tomcat01 ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
94: eth0@if95: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever

# 思考，我们的linux服务器是否可以ping通容器内的tomcat ？
[root@hecs-233798 dockerfile]# ping 172.17.0.2
PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.043 ms
64 bytes from 172.17.0.2: icmp_seq=2 ttl=64 time=0.045 ms

```

> 原理

1、每一个安装了Docker的linux主机都有一个docker0的虚拟网卡。这是个桥接网卡，使用了veth-pair 技术！

```bash
# 我们再次查看主机的 ip addr
[root@hecs-233798 dockerfile]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether fa:16:3e:32:8a:e0 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.86/24 brd 192.168.0.255 scope global noprefixroute dynamic eth0
       valid_lft 49203sec preferred_lft 49203sec
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:77:f6:f1:9a brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
95: veth4bb01e8@if94: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
    link/ether ce:30:12:58:e7:6c brd ff:ff:ff:ff:ff:ff link-netnsid 0
# 发现：本来我们有三个网络，我们在启动了个tomcat容器之后，多了一个95的网络
```

2、每启动一个容器，linux主机就会多了一个虚拟网卡。

```bash
# 我们启动了一个tomcat01，主机的ip地址多了一个 95: veth4bb01e8@if94
# 然后我们在tomcat01容器中查看容器的ip是 94: eth0@if95
# 我们再启动一个tomcat02观察
[root@hecs-233798 dockerfile]# docker run -d -P --name tomcat02 tomcat
# 然后发现linux主机上又多了一个网卡 97: vetha2d2f69@if96: 
# 我们看下tomcat02的容器内ip地址是 96: eth0@if97: 
[root@hecs-233798 dockerfile]# docker exec -it tomcat02 ip addr
# 观察现象：
# tomcat --- linux主机 veth4bb01e8@if94 ---- 容器内 eth0@if95
# tomcat --- linux主机 vetha2d2f69@if96 ---- 容器内 eth0@if97
# 相信到了这里，大家应该能看出点小猫腻了吧！只要启动一个容器，就有一对网卡
# veth-pair 就是一对的虚拟设备接口，它都是成对出现的。一端连着协议栈，一端彼此相连着。
# 正因为有这个特性，它常常充当着一个桥梁，连接着各种虚拟网络设备!
# “Bridge、OVS 之间的连接”，“Docker 容器之间的连接” 等等，以此构建出非常复杂的虚拟网络结构，比如 OpenStack Neutron。
```

3、我们来测试下tomcat01和tomcat02容器间是否可以互相ping通

```bash
[root@hecs-233798 dockerfile]# docker exec -it tomcat01 ping 172.12.0.3
PING 172.12.0.3 (172.12.0.3) 56(84) bytes of data.
3 packets transmitted, 0 received, 100% packet loss, time 1999ms
# 结论：容器和容器之间是可以互相访问的。

```

4、我们来画一个网络模型图

```bash
结论：tomcat1和tomcat2共用一个路由器。是的，他们使用的一个，就是docker0。任何一个容器启动
默认都是docker0网络。
docker默认会给容器分配一个可用ip。
```

> 小结

Docker使用Linux桥接，在宿主机虚拟一个Docker容器网桥(docker0)，Docker启动一个容器时会根据 Docker网桥的网段分配给容器一个IP地址，称为Container-IP，同时Docker网桥是每个容器的默认网 关。因为在同一宿主机内的容器都接入同一个网桥，这样容器之间就能够通过容器的Container-IP直接 通信。

Docker容器网络就很好的利用了Linux虚拟网络技术，在本地主机和容器内分别创建一个虚拟接口，并 让他们彼此联通（这样一对接口叫veth pair）； Docker中的网络接口默认都是虚拟的接口。虚拟接口的优势就是转发效率极高（因为Linux是在内核中 进行数据的复制来实现虚拟接口之间的数据转发，无需通过外部的网络设备交换），对于本地系统和容 器系统来说，虚拟接口跟一个正常的以太网卡相比并没有区别，只是他的速度快很多。

## --Link(不推荐)

思考一个场景，我们编写一个微服务，数据库连接地址原来是使用ip的，如果ip变化就不行了，那我们\
能不能使用服务名访问呢？\
jdbc\:mysql://mysql:3306，这样的话哪怕mysql重启，我们也不需要修改配置了！docker提供了 --link\
的操作&#x20;

```bash
# 我们使用tomcat02，直接通过容器名ping tomcat01，不使用ip
[root@hecs-233798 ~]# docker exec -it tomcat02 ping tomcat01
ping: tomcat01: Name or service not known
# 发现ping不通
# 我们再启动一个tomcat03，但是启动的时候--link连接tomcat02
[root@hecs-233798 ~]# docker run -d -P --name tomcat03 --link tomcat02 tomcat
8f48efcc4f386af28236d408ee3c500687bb62d1cc12fc6caae2550c133ffa6f

# 这个时候，我们就可以使用tomcat03 ping通tomcat02 了
[root@hecs-233798 ~]# docker exec -it tomcat03 ping tomcat02
PING tomcat02 (172.17.0.3) 56(84) bytes of data.
64 bytes from tomcat02 (172.17.0.3): icmp_seq=1 ttl=64 time=0.085 ms
64 bytes from tomcat02 (172.17.0.3): icmp_seq=2 ttl=64 time=0.057 ms

# 再来测试，tomcat03 是否可以ping tomcat01 失败
[root@hecs-233798 ~]# docker exec -it tomcat03 ping tomcat01
ping: tomcat01: Name or service not known

# 再来测试，tomcat02 是否可以ping tomcat03 反向也ping不通
[root@hecs-233798 ~]# docker exec -it tomcat02 ping tomcat03
ping: tomcat03: Name or service not known
```

探究inspect

```bash
[root@hecs-233798 ~]# docker inspect tomcat03
"HostConfig": {
....
            "Links": [
                "/tomcat02:/tomcat03/tomcat02"
            ],
....
```

思考，这个原理是什么呢？我们进入tomcat03中查看下host配置文件&#x20;

```bash
[root@hecs-233798 ~]# docker exec -it tomcat03 cat /etc/hosts
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
172.17.0.3	tomcat02 62d961f6b63c
172.17.0.4	8f48efcc4f38
# 所以这里其实就是配置了一个 hosts 地址而已！
# 原因：--link的时候，直接把需要link的主机的域名和ip直接配置到了hosts文件中了。
```

\--link早都过时了，我们不推荐使用！我们可以使用自定义网络的方式&#x20;

## 自定义网络

> 查看所有的docker网络

`docker network ls`

**网络模式**

bridge：桥接模式，docker默认，自己create的时候也建议使用

none：不配置网络

host：和宿主机共享网络

container：容器网络连通，很少使用，局限大

> 自定义网卡

1、删除原来的所有容器

`docker rm -f $(docker ps -aq)`

2、接下来我们来创建容器，但是我们知道默认创建的容器都是docker0网卡的&#x20;

```bash
# 默认我们不配置网络，也就相当于默认值
# --net bridge 使用的就是docker0的默认网卡
docker run -d -P --name tomcat01 --net bridge tomcat 
等价于
docker run -d -P --name tomcat01 tomcat
# docker0网络的特点
1.它是默认的
2.域名访问不通
3.--link 域名通了，但是删了又不行，需要一个一个docker run指定配置
```

3、我们可以让容器创建的时候使用自定义网络&#x20;

查看帮助

`docker network create --help`

```bash
# docker0特点 ：默认域名不能相互访问，需要用--link依次打通链接

# 所以我们使用自定义的网络
--driver bridge					默认 可以不写
--subnet 192.168.0.0/16			子网 192.168.0.2 ~ 192.168.255.255(56634个)
--gateway 192.168.0.1 			网关 可以理解路由器地址

[root@hecs-233798 ~]# docker network create --driver bridge --subnet 192.168.0.0/16 --gateway 192.168.0.1 mynet
04a09832333348ebfea71175575049b456193ed3fa37130bce5837fb6ea96269

# 确认一下
[root@hecs-233798 ~]# docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
c85c2faafa3c   bridge    bridge    local
ca7c328f85d7   host      host      local
04a098323333   mynet     bridge    local
e3f4172bffb3   none      null      local

[root@hecs-233798 ~]# docker network inspect mynet
[
    {
        "Name": "mynet",
        "Id": "04a09832333348ebfea71175575049b456193ed3fa37130bce5837fb6ea96269",
        "Created": "2023-09-26T11:44:42.103570658+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.0.0/16",
                    "Gateway": "192.168.0.1"
                }
....


# 我们来启动两个容器测试，使用自己的 mynet
[root@hecs-233798 ~]# docker run -d -P --net mynet --name tomcat-net-01 tomcat
a10d5fd79830a9d832dd98f7dd75e8cdc11d860b3633acce79866c9514b50d9e
[root@hecs-233798 ~]# docker run -d -P --net mynet --name tomcat-net-02 tomcat
7c3b8a3804165824ce33d8dd782f982ed770880b2a7466a993a0c8e0f7005e5a

# 再来查看下
[root@hecs-233798 ~]# docker network inspect mynet
....
                {
                    "Subnet": "192.168.0.0/16",
                    "Gateway": "192.168.0.1"
                }
....
        },
        "ConfigOnly": false,
        "Containers": {
            "7c3b8a3804165824ce33d8dd782f982ed770880b2a7466a993a0c8e0f7005e5a": {
                "Name": "tomcat-net-02",
                "EndpointID": "a5affad15f7993f5522e52200925cb2e3c541f48ef59f99809f7ae88eb511050",
                "MacAddress": "02:42:c0:a8:00:03",
                "IPv4Address": "192.168.0.3/16",
                "IPv6Address": ""
            },
            "a10d5fd79830a9d832dd98f7dd75e8cdc11d860b3633acce79866c9514b50d9e": {
                "Name": "tomcat-net-01",
                "EndpointID": "2351880e36c08439b1d5f18762f7f2aa19b65a0ab4b6d22a74db1f477e4e7c03",
                "MacAddress": "02:42:c0:a8:00:02",
                "IPv4Address": "192.168.0.2/16",
                "IPv6Address": ""
....

# 我们来测试ping容器名和ip试试，都可以ping通
[root@hecs-233798 ~]# docker exec -it tomcat-net-02 ping tomcat-net-01
PING tomcat-net-01 (192.168.0.2) 56(84) bytes of data.
64 bytes from tomcat-net-01.mynet (192.168.0.2): icmp_seq=1 ttl=64 time=0.050 ms

```

## 网络连通

docker0和自定义网络肯定不通，我们使用自定义网络的好处就是网络隔离：\
大家公司项目部署的业务都非常多，假设我们有一个商城，我们会有订单业务（操作不同数据），会有\
订单业务购物车业务（操作不同缓存）。如果在一个网络下，有的程序猿的恶意代码就不能防止了，所\
以我们就在部署的时候网络隔离，创建两个桥接网卡，比如订单业务（里面的数据库，redis，mq，全\
部业务 都在order-net网络下）其他业务在其他网络。\
那关键的问题来了，如何让 tomcat-net-01 访问 tomcat1？&#x20;

```bash

# 启动默认的容器，在docker0网络下
[root@hecs-233798 ~]# docker run -d -P --name tomcat01 tomcat
07b9adcd5beb08b45af5881f80f5bc253079ab376946e948856b5c356720316f
[root@hecs-233798 ~]# docker run -d -P --name tomcat02 tomcat
f0934ce21afc4df855e7e9a4c98274673539b6632920657c2c11c2e0efb8ebbf

# 查看当前的容器
[root@hecs-233798 ~]# docker ps
CONTAINER ID   IMAGE          COMMAND             CREATED          STATUS          PORTS                                         NAMES
f0934ce21afc   tomcat         "catalina.sh run"   5 seconds ago    Up 4 seconds    0.0.0.0:32776->8080/tcp, :::32776->8080/tcp   tomcat02
07b9adcd5beb   tomcat         "catalina.sh run"   10 seconds ago   Up 9 seconds    0.0.0.0:32775->8080/tcp, :::32775->8080/tcp   tomcat01
2499c5a87770   mytomcat:1.0   "catalina.sh run"   32 minutes ago   Up 32 minutes   0.0.0.0:32774->8080/tcp, :::32774->8080/tcp   tomcat-net-02
2835c25ca7ae   mytomcat:1.0   "catalina.sh run"   32 minutes ago   Up 32 minutes   0.0.0.0:32773->8080/tcp, :::32773->8080/tcp   tomcat-net-01

# 我们来查看下network帮助，发现一个命令 connect
[root@hecs-233798 ~]# docker network --help
Commands:
  connect     Connect a container to a network
  create      Create a network
  disconnect  Disconnect a container from a network
  inspect     Display detailed information on one or more networks
  ls          List networks
  prune       Remove all unused networks
  rm          Remove one or more networks
# 查看使用细节
[root@hecs-233798 ~]# docker network connect -h
Usage:  docker network connect [OPTIONS] NETWORK CONTAINER

# 我们来测试一下！打通mynet-docker0

# 命令 docker network connect [OPTIONS] NETWORK CONTAINER
[root@hecs-233798 ~]# docker network connect mynet tomcat01
[root@hecs-233798 ~]# docker network inspect mynet
[
    {
        "Name": "mynet",
        "Id": "04a09832333348ebfea71175575049b456193ed3fa37130bce5837fb6ea96269",
        "Created": "2023-09-26T11:44:42.103570658+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.0.0/16",
                    "Gateway": "192.168.0.1"
                }
....
        "Containers": {
            "07b9adcd5beb08b45af5881f80f5bc253079ab376946e948856b5c356720316f": {
                "Name": "tomcat01",
                "EndpointID": "bb849e897387df906749abc45ab62da55dffe4e1aef04d653d605bbd26e9639b",
                "MacAddress": "02:42:c0:a8:00:04",
                "IPv4Address": "192.168.0.4/16",
                "IPv6Address": ""
            },
            "2499c5a8777085520494be89b7cd7d23c56bbd722d1a8d580b5237aef2ab35c1": {
                "Name": "tomcat-net-02",
                "EndpointID": "6deda97168aa8dca1a9e454576b7c47a78814fffde3a74de49807443f3d1dbfe",
                "MacAddress": "02:42:c0:a8:00:03",
                "IPv4Address": "192.168.0.3/16",
                "IPv6Address": ""
            },
            "2835c25ca7aed26658a345c2b1170d804f050c6e305893999f4d96a3a6579531": {
                "Name": "tomcat-net-01",
                "EndpointID": "c2465984377a622faee71c6688c1c5029e5dd6b12586b1089635675346d94091",
                "MacAddress": "02:42:c0:a8:00:02",
                "IPv4Address": "192.168.0.2/16",
                "IPv6Address": ""
            }
....

# tomcat01 可以ping通了 
[root@hecs-233798 ~]# docker exec -it tomcat01 ping tomcat-net-01
PING tomcat-net-01 (192.168.0.2) 56(84) bytes of data.
64 bytes from tomcat-net-01.mynet (192.168.0.2): icmp_seq=1 ttl=64 time=0.065 ms
64 bytes from tomcat-net-01.mynet (192.168.0.2): icmp_seq=2 ttl=64 time=0.055 ms

# tomcat02 仍然ping不通
[root@hecs-233798 ~]# docker exec -it tomcat02 ping tomcat-net-01
ping: tomcat-net-01: Name or service not known
```

重点：之前的 tomcat-net-01 tomcat-net-02 都不能访问百度，导致 apt-get 无法更新、下载

尝试将 tomcat-net-01 放在 bridge 下就可以访问公网了。

结论：

*   如果要跨网络操作别人，就需要使用 `docker network connect [OPTIONS] NETWORK CONTAINER` 连接&#x20;
*   docker network connect 实际上就是直接把容器放在指定网络下，使一个容器有多个IP



## Docker Compose容器编排

## Docker Swarm集群部署

## Redis集群部署实战

（先学完redis再补全）<https://www.bilibili.com/video/BV1og4y1q7M4?p=38>

## CI/CD之jenkins