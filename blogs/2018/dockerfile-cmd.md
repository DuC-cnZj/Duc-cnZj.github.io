---
title: DockerFile 命令总结
date: '2021-05-12 12:54:14'
sidebar: true
categories:
 - DockerFile
tags:
 - DockerFile
publish: true
---


> 转自 [k8s中文社区](https://mp.weixin.qq.com/s?__biz=MzI5ODQ2MzI3NQ==&mid=2247485972&idx=1&sn=6e839c0b05464878f2b2fcc7e7e9a276&chksm=eca43350dbd3ba461b5c0d3a7368badac3af123c7cae2a624aabde44f3972247554e1f9b6026&scene=38#wechat_redirect)

> 只描述非`windows`系统。

-  `FROM <image>[:<tag>] [AS <name>]`: 设置基础镜像

```dockerfile
FROM alpine:latest
```

-  `RUN <command> \ ["executable", "param1", "param2"]`: 执行`shell`脚本。进来少使用`RUN`，因为没执行一次 `docker`就会增加一层只读层。

```dockerfile
RUN /bin/bash -c 'source $HOME/.bashrc; \
echo $HOME'
等同于
RUN /bin/bash -c 'source $HOME/.bashrc; echo $HOME'
等同于
RUN ["/bin/bash", "-c", "source $HOME/.bashrc; echo $HOME"]
```

-  `CMD ["executable","param1","param2"] \ ["param1","param2"] \ command param1 param2`: `DockerFile`中只有一个`CMD`，多于一个将执行最后一个。它的意思差不多就是启动容器后执行的默认命令。

```dockerfile
FROM *:*
CMD ["catalina.sh", "run"]
```

-  `LABEL <key>=<value> <key>=<value> ...` : 镜像标签

```dockerfile
LABEL "com.example.vendor"="ACME Incorporated"
LABEL com.example.label-with-value="foo"
LABEL version="1.0"
LABEL description="This text illustrates \
that label-values can span multiple lines."
```

-  `EXPOSE <port> [<port>/<protocol>...]`: 暴露容器的端口

```dockerfile
EXPOSE 80/tcp
EXPOSE 80/udp
```

-  `ENV <key> <value> \ <key>=<value>`: 设置容器环境变量。可以使用`docker run --env <key>=<value>`修改环境变量

```dockerfile
ENV myName John Doe
ENV myDog Rex The Dog
ENV myCat fluffy
```

-  `ADD [--chown=<user>:<group>] <src>... <dest> \ [--chown=<user>:<group>] ["<src>",... "<dest>"]`: 拷贝一个新文件，或者文件夹或者远程文件的 URLS，把他们添加到，镜像的文件系统中。`<dest>` 为绝对路径或者由`WORKDIR`定义的相对路径。

```dockerfile
ADD hom* /mydir/         # 添加所有以 "hom" 开头的文件
ADD hom?.txt /mydir/     # ? 替换任何单个的字符, e.g., "home.txt"

ADD test relativeDir/    # 添加 "test" 到 `WORKDIR`/relativeDir/
ADD test /absoluteDir/   # 添加 "test" 到 /absoluteDir/

ENV cpath /home/zb
ARG zbpath=/home/lala
WORKDIR $cpath
ADD **.jpg $cpath
ADD **.jpg $zbpath

# 添加含有特殊字符的文件或者文件夹时如“[]”,需要遵循 golang 的规则将它们进行转义，以防它们为匹配模式
ADD arr[[]0].txt /mydir/ # 复制一个文件名为 "arr[0].txt" 到 /mydir/

# 通过 --chown 指定添加文件或者文件夹的用户名和组名
ADD --chown=55:mygroup files* /somedir/
ADD --chown=bin files* /somedir/
ADD --chown=1 files* /somedir/
ADD --chown=10:11 files* /somedir/
```

- `COPY [--chown=<user>:<group>] <src>... <dest> / [--chown=<user>:<group>] ["<src>",... "<dest>"]`:  与 `ADD`命令差不多。
- `ENTRYPOINT ["executable", "param1", "param2"] / command param1 param2`:  容器启动后执行该命令。如果定义`ENTRYPOINT` 那么`CMD`形式就变为`CMD ["param1","param2"]` `json`数组中变为`ENTRYPOINT`的第一个参数和第二个参数

```dockerfile
ARG VERSION=latest
FROM alpine:$VERSION
ENTRYPOINT ["ls", "-la"]
```

`ENTRYPOINT`与`CMD`的比较

1. 当有多个`ENTRYPOINT CMD`它们都只执行最后一个
2. 当容器为一个可执行文件时应该定义`ENTRYPOINT` 
3. 当同时定义`ENTRYPOINT 和 CMD`时，`CMD`为`ENTRYPOINT`的默认参数
4. 当`docker`执行`run`命令时，在里面指定`CMD`时，`CMD`将会被重写。

-  `VOLUME ["/data"]`: 在制作镜像时挂载卷。会在宿主机上自动生成一个对应的共享目录。

```dockerfile
RUN mkdir /data1
RUN touch /data1/2a.txt
VOLUME /data1

# 通过命令 docker inspect hasVvolume
"Mounts": [
            {
                "Type": "volume",
                "Name": "0d63fcdf621ee728bb85dfcc2b30f3acddf6489a0e93b81ced17f05860497321",
                "Source": "/var/lib/docker/volumes/0d63fcdf621ee728bb85dfcc2b30f3acddf6489a0e93b81ced17f05860497321/_data",
                "Destination": "/data1",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }
        ]
====================================================================================
# 也可以通过 docker run -v 来挂载共享目录，这时 Source 指出 -v 时定义的目录
"Mounts": [
        {
                "Type": "bind",
                "Source": "/Users/zhangbo/Desktop/data1",
                "Destination": "/data1",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            }
        ]
```

容器中共享目录

```bash
# 使用 --volumes-from，达到容器中文件夹共享
docker run -itd --name noVvolume-v-1 --volumes-from noVvolume-v 48cd9e43b6a9
```

-  `USER <user>[:<group>] / <UID>[:<GID>]`: 给镜像添加一个用户
-  `WORKDIR /path/to/workdir or WORKDIR to_workdir /path/to/workdir`: 为 `RUN, CMD, ENTRYPOINT, COPY, ADD` 创建工作目录

```dockerfile
ENV DIRPATH /path
WORKDIR $DIRPATH/$DIRNAME
RUN pwd
```

-  `ARG <name>[=<default value>]` : `docker file`中的变量

```
FROM busybox
ARG user1="zhang bo"
ARG buildno
RUN echo $user1
RUN echo $buildno

Step */* : RUN echo $user1
 ---> Running in a56a602a8f87
zhang bo
Removing intermediate container a56a602a8f87
 ---> 3e9c6ec19129
Step */* : RUN echo $buildno
 ---> Running in 6598768a1080
```

预制变量

```dockerfile
FROM ubuntu
ARG CONT_IMG_VER
ENV CONT_IMG_VER ${CONT_IMG_VER:-v1.0.0}
RUN echo $CONT_IMG_VER

# 可以通过 --build-arg 标签进行给定的预制的变量。--build-arg <varname>=<value>
docker run --build-arg CONT_IMG_VER=******* .
```

## docker 常用命令如下
管理命令：

```txt
  container   管理容器
  image       管理镜像
  network     管理网络
  node        管理Swarm节点
  plugin      管理插件
  secret      管理Docker secrets
  service     管理服务
  stack       管理Docker stacks
  swarm       管理Swarm集群
  system      查看系统信息
  volume      管理卷
  
  如：docker container ls 显示所有容器
```
普通命令：
```txt
  attach     进入一个运行的容器
  build      从一个DockerFile构建镜像
  commit     从容器创建一个镜像
  cp          从容器和主机文件系统之间拷贝文件 
  create      创建一个容器
  diff        检查容器文件系统上的更改
  events      从服务器获取实时事件
  exec        在正在运行的容器中运行命令
  export      将容器的文件系统导出为tar存档
  history     显示镜像的历史记录
  images      查看镜像列表
  import      从归档文件中创建镜像
  info        显示系统范围的信息
  inspect     返回Docker对象的低级信息
  kill        kill运行中的容器
  load        从存档或者STDIN加载镜像
  login       登陆docker镜像仓库
  logout      退出docker镜像仓库
  logs        获取一个容器的日志
  pause       暂停一个或多个容器中的所有进程
  port        查看端口映射或容器的特定映射列表
  ps          查看容器列表
  pull        从镜像仓库拉取镜像
  push        将本地的镜像上传到镜像仓库,要先登陆到镜像仓库
  rename      重命名容器
  restart     重启容器
  rm          删除容器
  rmi         删除镜像
  run         创建一个新的容器并运行一个命令
  save        将指定镜像保存成 tar 归档文件
  search      从Docker Hub搜索镜像
  start       启动容器
  stats       实时显示容器资源使用情况的统计信息
  stop       停止容器
  tag         标记本地镜像，将其归入某一仓库
  top         展示一个容器中运行的进程
  unpause     恢复容器中所有的进程
  update      更新容器配置
  version    显示Docker的版本信息
  wait        阻塞直到容器停止，然后打印退出代码
```