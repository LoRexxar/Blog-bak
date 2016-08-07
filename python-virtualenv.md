---
title: python virtualenv沙盒命令
date: 2016-08-07 23:46:43
tags:
- python
- virtualenv
categories:
- Blogs
---
最近一直在写python的项目，突然用起了一个平时没接触过的virtualenv，是一个python环境的沙盒，在windows下有很多不同于linux的地方，稍微记录下

<!--more-->

# 首先有几篇不错的blog #

[http://www.cnblogs.com/tk091/p/3700013.html](http://www.cnblogs.com/tk091/p/3700013.html)

[http://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/001432712108300322c61f256c74803b43bfd65c6f8d0d0000](http://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/001432712108300322c61f256c74803b43bfd65c6f8d0d0000)

# 使用 #

首先是安装
```
pip install virtualenv
```

然后是创建并打开目录，当然windows直接建就好了
```
$ mkdir myproject
$ cd myproject/
```

创建一个python沙箱环境，假设我们命名为venv，一般来说是项目名+venv
```
$ virtualenv --no-site-packages venv
Using base prefix '/usr/local/.../Python.framework/Versions/3.4'
New python executable in venv/bin/python3.4
Also creating executable in venv/bin/python
Installing setuptools, pip, wheel...done.
```

进入该环境
1、linux下是这样的
```
$ source venv/bin/activate
```
如果我们看到有venv前缀，说明已经进入了

在windows下不同，windows下没有source命令，所以需要直接打开进

```
$ cd venv/Scripts
$ ./activate
```

然后我们就进入了一个独立的python环境中，在这个环境下，我们还需要重新配置整个python的库

如果想要退出，使用
```
$ deactivate 
```
