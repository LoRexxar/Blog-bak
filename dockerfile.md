title: dockerfile  (・ω・)ノ
date: 2016-05-05 19:47:53
tags:
- docker
- Blogs
categories:
- Blogs
---

构建镜像有docker commit和docker file两种，而docker file是现在使用最主要的方式，这里记录下docker file的基础指令

<!--more-->

首先看一个普通的dockerfile
```
# Version: 0.0.1

FROM ubuntu:14.04
MAINTAINER James Turnbull "james@example.com"

RUN apt-get update
RUN apt-get install -y nginx
RUN echo 'Hi!' >/usr/share/nginx/html/index.html

EXPOSE 80
```

dockerfile由一系列指令和参数组成，每条指令都必须为大写字母，后面必须跟一个参数

# dockerfile 指令
## FROM
每个dockerfile的第一条指令都应该是from，from指令指定一个已经存在的镜像，后续指令都将基于这个镜像进行
`FROM <image>`或`FROM <image>:<tag>`

## MAINTAINER

MAINTAINER指定了docker镜像的作者
`MAINTAINER <name>`

## RUN

RUN会在镜像中运行指定的命令，默认情况下/bin/sh -c运行，当然如果在一个不支持shell的平台上运行也可以通过使用exec格式的run指令

`RUN COMMAND`或`RUN ["executable", "param1", "param2"]`
前者为/bin/sh -c，后者使用exec执行，例如["/bin/bash", "-c", "echo hello"]

## CMD
指定启动容器时的命令，三种格式

`CMD ["executable", "param1", "param2"]`，推荐方式

`CMD COMMAND param1 param2`，在/bin/sh执行

`CMD ["param1", "param2"]`，提供给ENTRYPOINT的默认参数

## EXPOSE

暴露container的端口给主机。
`EXPOSE <port> [<port>...]`

## ENV
指定一个环境变量，格式：`ENV <key> <value>`

## ADD
复制指定的<src>到容器中的<dest>，<src>可以是Dockerfile所在目录的一个相对路径，也可以是URL，也可以是一个tar文件。
`ADD <src> <dest>`

## COPY
`COPY <src> <dest>`，复制本地的src到容器中dest。

## ENTRYPOINT

配置容器启动后执行的命令，不可被docker run提供的参数覆盖。每个dockerfile只能有一个ENTRYPOINT，如果指定多个，只有最后一个生效。
`ENTRYPOINT ["executable", "param1", "param2"]`
`ENTRYPOINT command param1, param2`

## VOLUME
`VOLUME ["/data"]`

## USER

`USER daemon`
指定运行容器的用户名或UID，后续的RUN使用指定的用户。

## WORKDIR

`WORKDIR /path/to/workdir`
为后续的RUN，CMD，ENTRYPOINT指令配置工作目录。
可以使用多个WORKDIR指令，如果后续的是相对路径，则会基于之前命令指定的路径。

## ONBUILD
`ONBUILD [INSTRUCTION]`

配置当所创建的镜像作为其他新创建镜像的基础镜像时，所执行的操作指令。

# 构建镜像

打开到Dockerfile的目录下输入下面的命令就会跑起来了，值得注意的是，后面有个点
```
sudo docker build -t jamtur01/static_web:v1 .
```
如果成功，最后会返回新镜像的id
如果失败，就比如，刚才的Dockerfile就会因为源的问题而报错，但每一步都会生成一个新的id的镜像，我们可以
根据每一步ID生成镜像，然后手动查找问题所在

而dcoker会将之前构建时创建的镜像当做缓存并作为新的开始点，如果重新构建的话，事实上会从上一次报错的地方直接开始，所以我们需要一个参数
```
sudo dcoker build --no-cache -t jamtur01/static_web .
```

# 用镜像来启动一个新容器
```
sudo docker run -d -p 80 --name static_web jamtur01/static_web nginx -g "daemon off;"
```
-d表示以分离的方式在后台运行，也就是用于类似nginx需要长时间运行的程序
-p是指公开那些网络端口给外部宿主机。

关于端口：
1、docker可以在49153~65535的大端口映射到80端口上
2、可以在docker宿主机中指定一个具体的端口号来映射

我们可以用`docker ps`查看端口分配情况
还可以用`docker port 容器ID 容器中需要分配的端口 `这样我们可以看到它绑定到了宿主机的那个端口
我们甚至可以在run的时候'直接指定
```
sudo docker run -d -p 8080:80 --name static_web jamtur01/static_web nginx -g "daemon off;"
```
这条命令指把容器中的80端口绑定到宿主机的8080上，我们还可以指定ip
```
sudo docker run -d -p 127.0.0.1:80:80 --name static_web jamtur01/static_web nginx -g "daemon off;"
```
这条命令是指80端口帮顶到了宿主机上的127.0.0.1 80端口
docker还提供了一个-P参数，可以用来对外公开dockerfile中expose指令中设置的所有端口。