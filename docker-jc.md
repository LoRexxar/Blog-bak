title: docker 基础操作
date: 2016-05-04 10:47:53
tags:
- docker
- Blogs
categories:
- Blogs
---

稍微测试了下windows，感觉好奇怪，放弃，还是用linux吧，linux大法好 (・ω・)ノ

<!--more-->

# 安装docker

```
sudo apt-get update 
sudo apt-get install -y docker.io 
sudo ln -sf /usr/bin/docker.io /usr/local/bin/docker
```
这里测试sudo docker info 有效就好了

# docker 基础操作

## 启动docker
docker run 提供了docker容器创建到启动的功能
```
sudo docker run -i -t ubuntu /bin/bash
```
这里会自动pull下来一个ubuntu镜像，然后-i表示容器中STDIN是开启的，-t表示要为容器分配一个伪tty,这样就有了一个交互式shell了

我们可以通过hostname查看主机名。可以直接apt-get安各种东西

## 查看系统中的容器列表

sudo docker ps -a查看当前系统中的容器列表
如果想在创建的时候制定一个名称，而不是随机生成一个（因为你必须通过id或者name操作使用哪个容器）
```
sudo docker run --name 容器的名字  -i -t ubuntu /bin/bash
```
## 重启启动已经停止的容器（除非你启动的是一个守护式的容器，否则在离开的时候都会停止容器）

```
sudo docker start ID或Name
```

容器重新启动后我们需要重新附着到容器的回话中
```
sudo docker attach NAME或者ID`
```

## 创建守护式容器

除了交互式运行的容器意外，我们更多需要创建长期运行的容器，非常适合运行一个守护式进程
```
sudo docker run --name 给容器起个名字 -d ubuntu /bin/sh -c "while true; do echo hello world; sleep 1;done"
```
这样就跑起来一个正在循环输出hello world的进程
### 我们可以看看容器内在干吗
```
sudo docker logs 容器的名字
```
还可以动态的看，就好像tail -f一样
```
sudo docker logs -f 容器的名字
```
还可以加上时间戳

```
sudo docker logs -fs 容器的名字
```

## 怎么看容器的进程呢？

```
sudo docker top 容器的名字
```

如果还想运行别的进程呢
```
sudo docker exec -d 容器的名字 命令
```
example: `sudo docker exec -d xxxx touch /etc/xxxx`

而且你还可以打开一个交互式的shell操作
```
sudo docker exec -t -i 容器的名字 /bin/bash
```
## 停止守护式容器
```
sudo docker stop 容器的名字或ID
```
如果由于某种错误导致容器停止运行，那么我们可以通过--restart来自动重新启动这个容器
```
sudo docker run --restart=always --name 容器的名字 -d ubuntu /bin/sh -c “命令”
```

这里的always是指无论退出代码是什么都自动重启，但是我们可以设置为on-failure，这样是指当容器代码为非0的时候，才会自动重启，另外这个on-failure还可以接受一个参数是可选的重启次数`--restart=on-failure:5`

## 深入容器
```
sudo docker inspect 容器的名字
```

这样可以获得更多的信息，我们还可以通过-f 或者--format这样的获取容器的运行状态
```
sudo docker inspect --format='{{ .State.Running }}' 容器的名字
```
这里实际是支持完整的GO语言的

## 删除容器
```
sudo docker rm ID
```
ps：正在运行中的容器是无法删除的

## 使用docker镜像和仓库
```
sudo docker images 列出docker主机上可用的镜像
sudo docker pull xxxx
```
从镜像仓库拉去，仓库在registry，默认是从docker hub拉去
可以通过在仓库名后面加上一个冒号和标签名来指定该仓库的某一镜像
```
sudo docker run -t -i --name 容器的名字 ubuntu:14.04 /bin/bash
(如果不指定，默认是lasest版本)
```

## 查找镜像
```
sudo docker search xxxx
```

## 构建镜像

有两种方法docker commit 和 Dockerfile

但是现在不推荐使用第一种方式，而是多使用dockerfile
```
sudo docker commit 容器id 镜像仓库/镜像名
sudo docker commit 4aafdsse3 jamtur01/apache2
```
构建好了可以提交到仓库去
```
sudo docker commit -m="A new custom image" --author="James" 4adsfads3 jamtru01/apache2:webserver
```
如果想用刚才生成的新镜像运行一个容器
```
docker run -t -i jamtur01/apache2:webserver /bin/bash
```

Dockerfile是docker主要使用的构建镜像方式，所以分来专门一个文章