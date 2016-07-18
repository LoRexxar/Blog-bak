title: 简单的git教程
date: 2016-02-18 16:25:35
tags:
- Blogs
- git
categories:
- Blogs
---
为了能让学弟学妹尽快上手github，而不是浪费大部分时间在上传东西上，简单的写一个git的教程

<!--more-->

## git是什么？
Git是一款免费、开源的分布式版本控制系统，用于敏捷高效地处理任何或小或大的项目。[1]  Git的读音为/gɪt/。
Git是一个开源的分布式版本控制系统，用以有效、高速的处理从很小到非常大的项目版本管理。[2]  Git 是 Linus Torvalds 为了帮助管理 Linux 内核开发而开发的一个开放源码的版本控制软件。

## git的安装
[http://git-scm.com/downloads](http://git-scm.com/downloads)
选择适合自己系统的git下载安装，我选择的windows。
linux一般只要一句命令就好了`sudo apt-get install git`

## 配置github

### 首先你得注册一个github的账号
[github官网](github.com)

### 添加ssh公钥
如果看不懂我的教程，这一步可以看官方教程。
[github的官方教程](https://help.github.com/articles/generating-a-new-ssh-key/)
[gitcafe的官方教程](https://gitcafe.com/GitCafe/Help/wiki/%E5%A6%82%E4%BD%95%E5%AE%89%E8%A3%85%E5%92%8C%E8%AE%BE%E7%BD%AE-Git#wiki)

1.打开桌面的git bash，检查本机的密钥。
`$ cd ~/. ssh`
如果提示：No such file or directory 说明你是第一次使用git。
2.如果不是第一次使用，执行下面的操作：
```
 mkdir key_backup
 cp id_rsa* key_backup    
 rm id_rsa*
```
3.生成自己的密钥，记得把“”中间的邮箱改成自己的。
`ssh-keygen -t rsa -C "YOUR_EMAIL@YOUREMAIL.COM"`
生成过程中会出现以下信息，按屏幕提示操作，并记得输入 passphrase 口令（这里直接回车过去既可，以后出现这个都是空密码，直接回车即可）
```
$ ssh-keygen -t rsa -C "YOUR_EMAIL@YOUREMAIL.COM"
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /c/Users/USERNAME/.ssh/id_rsa.
Your public key has been saved in /c/Users/USERNAME/.ssh/id_rsa.pub.
The key fingerprint is:
15:81:d2:7a:c6:6c:0f:ec:b0:b6:d4:18:b8:d1:41:48 YOUR_EMAIL@YOUREMAIL.COM
```
4.SSH 秘钥生成结束后，你可以在用户目录 (~/.ssh/) 下看到私钥 id_rsa 和公钥 id_rsa.pub 这两个文件，记住千万不要把私钥文件 id_rsa 透露给任何人。不然别人就可以以管理员的身份登陆，修改你项目中的文件（也就是你的博客）。
5.添加ssh到github
使用任意文本工具打开公钥文件 ~/.ssh/id_rsa.pub,复制里面的所有内容。

进入 GitCafe –>setting–>SSH key，点击添加新公钥 按钮，在 Title 文本框中输入你的id，在 Key 文本框粘贴刚才复制的公钥字符串，按保存按钮完成操作。

6.测试是否可以链接服务器，同样打开Git Bash输入：
`ssh -T git@github.com`
如果是第一次连接，会出现以下警告：
```
The authenticity of host 'gitcafe.com (50.116.2.223)' can't be established.
#RSA key fingerprint is 84:9e:c9:8e:7f:36:28:08:7e:13:bf:43:12:74:11:4e.
#Are you sure you want to continue connecting (yes/no)?
```
上面出现的其实是所谓的指纹，用来识别你接受到的是否和服务器发送的一致，来识别是否收到中间人攻击，这里直接yes既可。之后会提示出入passphrase口令，如果上面设置没问题，直接回车即可。

最后出现下面的字符，就说明成功了。
```
Hi USERNAME! You've successfully authenticated, but GitCafe does not provide shell access.
```

### 设置账户信息
```
$ git config --global user.name "yourname"                  //github上的用户名
$ git config --global user.email "yourmaill@yourmaili.com"  //填写自己的邮箱
```

## 创建代码库
在github的首页，可以看到比较明显的新建项目，输入对应的项目名，比如hctf_writeup,然后我们就能看到项目已经建好了...
然后让我们来仔细讲解每一步
```
echo "# hctfgame_writeup" >> README.md
git init
git add README.md
git commit -m "first commit"
git remote add origin git@github.com:LoRexxar/hctfgame_writeup.git
git push -u origin master
```
在本地你所要提交的文件夹内右键打开git bash
`git init`
初始化一个空的本地代码库。
`echo "# hctfgame_writeup" >> README.md`
将‘hctfgame_writeup’写到README.md中
`git add README.md`
将readme.md添加到缓存区(每次提交文件，都必须先添加到缓存区，一般是git add .)
`git commit -m 'xxx'`
xxx是指这次提交的更新信息，这个信息尽量不要无意义，起码要知道这次更新是改变了什么，例如“updata week0_writeup”
`git remote add origin git@github.com:LoRexxar/hctfgame_writeup.git`
这句话的大概意思是说绑定到远程代码库，后面的这串.git是github每个项目对应的ssh链接。打开项目可以看到（只要第一次使用就好）
`git push -u origin master`
提交到默认的master分支。

打开看我们的项目，可以看到已经提交上去了。

### 切换分支
之所以有分支的存在是由于，对于多人合作的远程代码库来说，常常是分多人合作，但是你不能保证你手里的代码库是最新版，你总不可能永远直到别人是不是同时在工作，这个时候分支的重要就体现出来了。
或者也有一种可能，对于你的代码来说，你可能在试图实现一个功能的时候遇到了困难，你不知道哪一种修改方式更为合理，这时候，多个分支就成了不二的选择。
`$ git branch <branchname>`
branchname就是你的新分支名字
`$ git checkout <branch>`
切换到新分支branch

### 从远程拉取代码库
往往你会遇到你的本地代码库版本弱于远程代码库，有可能是因为有别人修改了你的项目，也有可能是你换了一台电脑，这时候，你就需要先从远程拉去代码库。
`git pull git@github.com:LoRexxar/hctfgame_writeup.git`
ps：首先你得在本地初始化一个代码库（git init）

简单的git教程就到这里吧，以后有时间在完善。

