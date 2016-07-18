---
title: shell上一些有趣的命令
date: 2016-07-18 11:44:12
tags:
- cmd
- windows
- shell
- linux
categories:
- Blogs
---

有时候能碰到一些有趣的命令，那就记录下来，以后会更新在这里

<!--more-->

# windows #

## cmd ##

### 看已连接过的wifi密码 ###

windows下cmd只要是root权限跑起来都可以执行

```
for /f "skip=9 tokens=1,2 delims=:" %i in ('netsh wlan show profiles') do  @echo %j | findstr -i -v echo | netsh wlan show profiles %j key=clear
```

# linux #