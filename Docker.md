# Docker

## 一、Docker容器使用

### 1、运行交互式容器

```shell
#获取镜像
docker pull ubuntu 

#查看所有容器
docker ps -a

#启动一个已停止的容器
docker start containerid

#后台运行
docker run -itd --name ubutu-test ubuntu /bin/bash

#停止容器
docker stop containerid

#进入容器
docker attach 
docker exec :此命令退出同期终端，不会导致容器停止。

#导出和导入容器
docker export containerid > ubuntu.tar
cat docker/ubuntu.tar | docker import - test/ubuntu:v1
docker import http://example.com/exampleimage.tgz example/imagerepo #通过指定URL或者某个目录导入

#删除容器
docker rm -f containerid
docker container prine #清理掉所有停止状态的容器

#查看端口快捷方式
docker port containerid

#查看容器内部的标准输出
docker logs -f containerid

#查看容器内部运行的进程
docker top containerid

#查看Docker的底层信息，返回配置信息和状态信息
docker inspect containerid

#查看最后一次创建的容器
docker ps -l
```

```shell
docker run -d -P training/webapp python app.py

#0.0.0.0:32769->5000/tcp		Docker开放了5000端口映射到主机端口32769上
```

-P：将容器内部使用的网络端口随机映射到我们使用的主机上。

```shell
docker run -it ubuntu:15.10 /bin/bash
```

-t : 在新的容器内指定一个伪终端或终端。

-i：允许你对容器内的标准输入（STDIN）进行交互。

### 2、启动容器（后台模式）

```shell
docker run -d ubuntu:15.10 /bin/bash -c "while true; do echo hello world; sleep 1 done"
```

-d：后台运行

-c：command，启动时运行的命令

```shell
#在宿主机内查看container的log
docker logs containerid

#停止容器
docker stop containerid

```

### 3、镜像使用

```shell
#查找镜像
docker search httpd

#获取一个镜像
docker pull ubuntu:13.10

#删除镜像
docker rmi images_name

#更新镜像
docker commit -m="has update" -a="作者" containerid runoob/ubuntu:v2
#-m :提交描述信息
#-a：指定镜像作者

```

构建镜像：使用Dockerfile

```shell
runoob@runoob:~$ cat Dockerfile 
FROM    centos:6.7
MAINTAINER      Fisher "fisher@sudops.com"

RUN     /bin/echo 'root:123456' |chpasswd
RUN     useradd runoob
RUN     /bin/echo 'runoob:123456' |chpasswd
RUN     /bin/echo -e "LANG=\"en_US.UTF-8\"" >/etc/default/local
EXPOSE  22
EXPOSE  80
CMD     /usr/sbin/sshd -D
```

```shell
docker build -t runoob/centos:6.7 .
#-t：镜像名
#.：Dockerfile文件所在目录，可以指定Dockerfile的绝对路径。

#设置镜像标签
docker tag imageid runoob/centos:dev
```



### 4、Docker容器连接

1.网络端口映射

-P：是容器内部端口随机映射到主机的端口。

-p：是容器内部端口绑定到指定的主机端口。

```shell
docker run -d -p 5000:5000 training/webapp python app.py

#指定网络绑定地址（使用127.0.0.1:5001访问容器的5000端口）
docker run -d -p 127.0.0.1:5001:5000 training/webapp python app.py
```

2、Docker容器互联

docker有一个连接系统允许将多个容器连接在一起，共享连接信息。

docker连接会创建一个父子关系，其中父容器可以看到子容器的信息。

- 容器命名

```shell
docker run -d -P --name runoob training/webapp python app.py
```

- 新建网络

```shell
docker network create -d bridge test-net
```

-d ：Docker网络类型，有bridge、overlay两种。

安装ping

```shell
apt-get update
apt install iputils-ping
```

- 配置全局DNS

在宿主机/etc/docker/daemon.json文件中增加配置项

```json
{
  "dns" : [
    "114.114.114.114",
    "8.8.8.8"
  ]
}
```

配置完后需要重启docker才能生效，查看DNS是否生效

```shell
docker restart
docker run -it --rm ubuntu cat etc/resolv.conf
```

- 手动指定的容器的配置DNS

```shell
docker run -it --rm -h host_ubuntu  --dns=114.114.114.114 --dns-search=test.com ubuntu
```

--rm：容器退出时自动清理容器内部的文件系统。

-h：hostname或者 --hostname=HOSTNAME

--dns：添加 DNS 服务器到容器的 /etc/resolv.conf 中，让容器用这个服务器来解析所有不在 /etc/hosts 中的主机名。

--dns-search：设定容器的搜索域，当设定搜索域为 .example.com 时，在搜索一个名为 host 的主机时，DNS 不仅搜索 host，还会搜索 host.example.com。

如果在容器启动时没有指定 **--dns** 和 **--dns-search**，Docker 会默认用宿主主机上的 /etc/resolv.conf 来配置容器的 DNS。

### 5、Docker仓库管理

```shell
#在https://hub.docker.com注册账号

#登录
docker login  
#登出
docker logout

#查找
docker search ubuntu

#拉取
docker pull ubuntu

#推送
docker push username/ubuntu:v2
```

### 6、Dockerfile

Dockerfile是用来构建镜像文本文件，文本内容包含了一条条构建镜像所需的指令和说明。

```shell
FROM nginx
RUN echo '这是一个本地构建的nginx镜像' > /usr/share/nginx/html/index.html

#FROM：定制的镜像都是基于FROM的镜像，这里是nginx就是定制需要的基础镜像，后续的操作都是基于nginx。
#RUN：用于执行后面跟着的命令行命令
```

注意：Dockerfile 的指令每执行一次都会在 docker 上新建一层。所以过多无意义的层，会造成镜像膨胀过大。

以下执行创建3层镜像。

```shell
FROM centos
RUN yum -y install wget
RUN wget -O redis.tar.gz "http://download.redis.io/releases/redis-5.0.3.tar.gz"
RUN tar -xvf redis.tar.gz
```

```shell
FROM centos
RUN yum -y install wget \
    && wget -O redis.tar.gz "http://download.redis.io/releases/redis-5.0.3.tar.gz" \
    && tar -xvf redis.tar.gz
```

- 开始构建镜像，在Dockerfile文件存放的目录下，执行构建动作

```shell
docker build -t nginx:v3 .
```

Dockerfile指令

- FROM：指定基础镜像，用于后续的指令构建。
- MAINTAINER：指定Dockerfile的作者/维护者。（已弃用，推荐使用LABEL指令）
- LABEL：添加镜像的元数据，使用键值对的形式。
- RUN：在构建过程中在镜像中执行命令。
- CMD：指定容器创建时的默认命令。（可以被覆盖）
- ENTRYPOINT：设置容器创建时的主要命令。（不可被覆盖）
- EXPOSE: 声明容器运行时监听的特定网络端口。
- ENV:在容器内部设置环境变量。
- ADD: 将文件、目录或远程URL复制到镜像中。
- COPY:将文件或目录复制到镜像中。
- VOLUME：为容器创建挂载点或声明卷。
- WORKDIR：设置后续指令的工作目录。
- USER：指定后续指令的用户上下文。
- ARG：定义在构建过程中传递给构建器的变量，可使用 "docker build" 命令设置。
- ONBUILD：当该镜像被用作另一个构建过程的基础时，添加触发器。
- STOPSIGNAL：设置发送给容器以退出的系统调用信号。
- HEALTHCHECK：定义周期性检查容器健康状态的命令。
- SHELL：定义周期性检查容器健康状态的命令。

### 7、Docker Compose

1. 简介

   Compose 是用于定义和运行多容器 Docker 应用程序的工具。通过 Compose，您可以使用 YML 文件来配置应用程序需要的所有服务。然后，使用一个命令，就可以从 YML 文件配置中创建并启动所有服务。

2. yaml文件

   ```yaml
   # yaml 配置实例
   version: '3'
   services:
     web:
       build: .
       ports:
       - "5000:5000"
       volumes:
       - .:/code
       - logvolume01:/var/log
       links:
       - redis
     redis:
       image: redis
   volumes:
     logvolume01: {}
   ```

   - version: 指定yml已从compose哪个版本指定的。

   - build：指定为构建镜像上下文路径，Dockerfile路径。

     - context：上下文路径。
     - dockerfile：dockerfile文件名。
     - args：添加构建参数，只能在构建过程中访问的环境变量。
     - labels：设置构建镜像的标签。
     - target：多层构建，可以指定构建哪一层。

   - cap_add,cap_drop：添加或删除容器拥有的宿主机的内核功能。

   - cgroup_parent：为容器指定父 cgroup 组，意味着将继承该组的资源限制。

   - command：覆盖容器启动的默认命令。

   - deploy：指定与服务的部署和运行有关的配置。只在 swarm 模式下才会有用。

     ```yaml
     version: "3.7"
     services:
       redis:
         image: redis:alpine
         deploy:
           mode：replicated
           replicas: 6
           endpoint_mode: dnsrr
           labels: 
             description: "This redis service label"
           resources:
             limits:
               cpus: '0.50'
               memory: 50M
             reservations:
               cpus: '0.25'
               memory: 20M
           restart_policy:
             condition: on-failure
             delay: 5s
             max_attempts: 3
             window: 120s
     ```

     

3. Dockerfile文件

   ```shell
   FROM python:3.7-alpine
   WORKDIR /code
   ENV FLASK_APP app.py
   ENV FLASK_RUN_HOST 0.0.0.0
   RUN apk add --no-cache gcc musl-dev linux-headers
   COPY requirements.txt requirements.txt
   RUN pip install -r requirements.txt
   COPY . .
   CMD ["flask", "run"]
   ```

4. app.py

   ```python
   import time
   
   import redis
   from flask import Flask
   
   app = Flask(__name__)
   cache = redis.Redis(host='redis', port=6379)
   
   
   def get_hit_count():
       retries = 5
       while True:
           try:
               return cache.incr('hits')
           except redis.exceptions.ConnectionError as exc:
               if retries == 0:
                   raise exc
               retries -= 1
               time.sleep(0.5)
   
   
   @app.route('/')
   def hello():
       count = get_hit_count()
       return 'Hello World! I have been seen {} times.\n'.format(count)
   ```



### 8、Docker Swarm集群管理

