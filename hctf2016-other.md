---
title: hctf2016 简单部分WEB && misc writeup
date: 2016-12-01 16:48:07
tags:
- Blogs
- ctf
- hctf
- misc
- php opcache
categories:
- Blogs
---

因为队伍里没有专门的misc选手，所以其实这次比赛的的misc都是我们凑的，除了最开始的杂项签到，只有你所知道的隐写就仅此而已嘛不是我出的，这里稍微整理一下，整体misc都非常简单，唯一一个比较难的misc题目还因为我的出题失误导致基本上只有一个师傅做题方式接近我的正解，下面稍微研究一下简单的web部分和misc题目

<!--more-->

# level1 #

## 2099年的flag ##

题目描述：only ios99 can get flag(Maybe you can easily get the flag in 2099
最终分数：10pt
完成人数：226

进去后发现这么一句话，相信有经验的人第一时间就能反应过来是要改ua。

感觉可能很多人不知道，chrome的f12本身就有功能可以模拟手机，复制请求头中的ua，然后修改为ios99即可（ps：ios是ipone的系统）

# level2 #

## RESTFUL ##

题目描述：博丽神社赛钱箱
最终分数：34pt
完成人数：152

其实题目是很简单的，只不过可能是很多人不熟悉的领域，百度RESTFUL研究一下就知道怎么回事了，在RESTFUL下，传参方式改成了路由传参，也就是`\money\12345`就是传递了money=12345。

所以根据题意，锁定payload，贴心的aklis大佬还自带了脚本
![](/img/hctf2016/10.png)

## pic again ##

出于平衡难度梯度的原则，所以出了道图片隐写，因为最近ichunxxx一条龙比赛越来越多，脑洞越来越大，索性就出了个标准的lsb隐写，为了不让题目太简单，我选择塞了一个压缩包进去

如果你要问lsb怎么办，我只能放代码了，如果这个看不懂的话，你可能需要审视一下自己到底是被ctf玩还是玩ctf了

```
# coding: utf-8
from PIL import Image


fflag = open("justastart.zip","rb")
flag = []

while True:
	byte = fflag.read(1)
	if byte == "":
		break
	else:
		hexstr = "%s" % byte.encode("hex")
		decnum = int(hexstr, 16)
		binnum = bin(int(hexstr, 16))[2:].zfill(8)

		for i in xrange(8):
			flag.append(binnum[i:i+1])

flag.reverse()

im = Image.open('misc1.jpg')


width = im.size[0]
height = im.size[1]

pic = Image.new("RGB",im.size)

for y in xrange(height):
	for x in xrange(width):

		pixel = list(im.getpixel((x, y)))

		for i in xrange(3):
			count = pixel[i]%2

			if len(flag) == 0:
				break

			if count == int(flag.pop()):
				continue

			if count == 0:
				pixel[i]+=1

			elif count == 1:
				pixel[i]-=1

		pic.putpixel([x, y],tuple(pixel))

pic.save("flag.png")
```

顺手附上解密代码

```
# coding: utf-8
from PIL import Image

im = Image.open('flag.png')


width = im.size[0]
height = im.size[1]

a = ""
aa = ""

for y in xrange(height):
	for x in xrange(width):

		pixel = im.getpixel((x, y))

		for i in xrange(3):
			aa += str(pixel[i]%2)

for i in xrange(len(aa)):
	try:
		a += chr(int(aa[i*8:i*8+8],2))
	except:
		break

fflag = open("test.zip","w")

fflag.write(a)
fflag.close()
```

如果你还不清楚怎么发现lsb特征的话，你可能需要`Stegsolve`这个工具

## gogogo ##

题目描述：上上下下左左右右baba

打游戏而已，就简单的打游戏好了，金手指无敌过关就可以get flag。

有人问有没有逆向解法，事实上其实没有，游戏是通过`Tile Layer Pro`直接对图层的修改，比较特别，有兴趣的人可以去玩玩看

# level4 #

## web选手的自我修养 ##

题目描述：新搭的wp居然爆了漏洞，真气，漏洞修复了却被安了后门，你能找到后门在哪吗？？？提供压缩包为docker镜像

这题目其实可以出的特别好，但是由于我个人要管的题目太多了，所以搞着搞着就乱了，导致这道题目被非预期所沦陷了，这里我就重新说一遍正解吧。

首先在正常的题目中，现在在home目录下的脚本是不该存在的，也不会存在所谓的viminfo以及access.log日志。

加载docker之后，首先发现docker中只存在lnmp环境，其余没有任何服务，我们找到`/home/wwwroot/default/`下是web目录，里面放了一个最新版的wp，并且还能找到压缩包

既然题目中是站中安了后门，首先肯定是解压源码，然后diff文件，很快就能发现，源码其实并没有被改变。

那么写一个phpinfo.php页面

![](/img/hctf2016/11.png)

我们发现php版本是7.0.7，接着向下看

![](/img/hctf2016/12.png)

我们发现不一样的东西。php opcache开启，而且检验一致性的配置已经关闭了（7.0.7增加了这个配置，为了漏洞存在，我关闭了这个）

那么猜测后门被直接写入了bin中，那么大家肯定会想到这个工具。
[http://lorexxar.cn/2016/05/27/opcache-jcfx/](http://lorexxar.cn/2016/05/27/opcache-jcfx/)

也就是我在测试中无意留在镜像中的脚本，但是很多队伍都发现了，这个脚本是存在问题的。

因为在镜像中有个库版本过高，所以整个脚本完全跑不起来，所以我想了另一种分析方式。

第一种，时间戳排序法

回想漏洞存在的条件，如果真的到了通过这种方式拿后门的情况，写后门一定会写入不容易被改变的位置，因为如果文件被改变，bin就会重新编译。

其次，既然我需要写入后门进bin中，肯定需要修改bin，那么文件的时间戳一定会被改变。

那么在一群没怎么被改变的文件处，最新的部分时间戳中一定有问题。

那么锁定文件，flag一定是可显字符串，可以没必要分析字节码，直接strings即可