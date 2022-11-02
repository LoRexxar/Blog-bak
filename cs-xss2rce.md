---
title: CS Xss2Rce CVE-2022-39197分析与复现
date: 2022-11-02 21:14:51
tags:
- cs
- CVE-2022-39197
- java
---

前段时间这个漏洞被挖掘出来之后的时候还是引发了很多关注的，但是最初一直都没有什么像样的分析文章出来，最早看@漂亮鼠的文章之后才大体上对这个漏洞有了一个基本的认识。

https://mp.weixin.qq.com/s/l5e2p_WtYSCYYhYE0lzRdQ

但是不知道是我的java水平真的不够，又或者说这篇文章中隐去的部分太多了，我顺着文章研究了一段时间但是几个点都串不起来。后来又接二连三的看了几篇文章，直到看完@pang0lin才算是把逻辑串联起来许多

https://mp.weixin.qq.com/s?__biz=MzkzNjMxNDM0Mg==&mid=2247485450&idx=1&sn=5662a9f2c081fc8521eee651b357323f&chksm=c2a1dc83f5d655957cf2a1c88adf45cd0d028316f536e0a8a18c9e7cb0440869ed4077822ce8&mpshare=1&scene=1&srcid=1017PFOrquxivFM2ck4Prl9m&sharer_sharetime=1665993698480&sharer_shareid=8c0858e06fee9d607c68521b3949c3e9#rd

这篇文章不知道什么时候发出来，主要是记录一下整个复现分析的心路历程，以便以后需要的时候没处看。



<!--more-->

# 漏洞原文

https://www.cobaltstrike.com/blog/out-of-band-update-cobalt-strike-4-7-1/

CVE-2022-39197

An independent researcher identified as “Beichendream” reached out to inform us about an XSS vulnerability that they discovered in the teamserver. This would allow an attacker to set a malformed username in the Beacon configuration, allowing them to remotely execute code. We created a CVE for this issue which has been fixed.



这个漏洞在cs的 4.7.1版本中被修复，他允许攻击者可以伪造用户名来构造xss，通过xss利用可以造成RCE。

修复的手段也很简单，就是把这个xss给修了，正所谓只要入口点没了，你后面也利用不起来。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022225152.png)

# Cobalt Strike与swing

在理解这个漏洞之前，我们首先需要理解cs和swing这个框架的关系。

众所周知，Cobalt Strike是世界上最流行的网站管理工具，cs提供了允许多个目标通过非对称的加密方式链接teamserver，在这个过程中，只要传输的数据可以被teamserver解开，那么这个连接就会上线，这本质上就是一个简单的服务通信逻辑，这没什么问题。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022225479.png)

而这个公钥呢可以从Beacon stage的bin文件中获取，相关的脚本其实也可以多，比如

https://github.com/LiAoRJ/CS_fakesubmit

本地测试的话，可以用更简单这个解密脚本获得这个公私钥。

```java
import java.io.File;
import java.util.Base64;
import common.CommonUtils;
import java.security.KeyPair;

class DumpKeys
{   
    public static void main(String[] args)
    {
        try {
            File file = new File(".cobaltstrike.beacon_keys");
            if (file.exists()) {
                KeyPair keyPair = (KeyPair)CommonUtils.readObject(file, null);
                System.out.printf("Private Key: %s\n\n", new String(Base64.getEncoder().encode(keyPair.getPrivate().getEncoded())));
                System.out.printf("Public Key: %s\n\n", new String(Base64.getEncoder().encode(keyPair.getPublic().getEncoded())));
            }
            else {
                System.out.println("Could not find .cobaltstrike.beacon_keys file");
            }
        }
        catch (Exception exception) {
           System.out.println("Could not read asymmetric keys");
        }
    }
}
```

只要获得了公钥，那么我们就可以向teamserver传入一个虚假的会话上线，通过客户端链接服务端之后就可以看到这条链接。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022225231.png)

在这个页面当中虽然说看上去信息很多，实际上可控的部分并不多，这一点在漂亮鼠的文章中有详细的解释，主要是数据结构中的可控点问题

https://mp.weixin.qq.com/s/l5e2p_WtYSCYYhYE0lzRdQ

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022225371.png)

这里我们先不深究cs的传输协议，我们只需要关注，在这个页面中可以控制的其实就是中间的会话信息部分，其中包括总共117字节的user、computer、process一共3个部分

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022225753.png)

而且理论上来说，如果伪造脚本足够合理，那么由这个界面衍生出来的各种界面都是可以控制的，最简单的就比如原文提到的查看进程列表，但其实其他的比如查看文件管理等功能其实都差不多。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022225060.png)

而前面说了这么多，和swing有什么关系呢？**前面看到的所有画面都是用swing这个库画的，所谓的XSS也来源于这里。**

# Swing库与Xss

https://docs.oracle.com/javase/tutorial/uiswing/components/html.html

从相关的官方文档可以知道，Swing本身是**支持HTML的标签**的。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022225327.png)

只要在文本的开始加入html标签就会解析html标签，而比较有趣的事情是，这套html的解析引擎是他自己实现的，并不是引了一个第三方引擎。从swing的代码中可以看到这一点。

path: swing.text.html

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022225448.png)![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022225253.png)



而更有趣的是，类似script这种标签虽然解析，但是并没有用，而是套了一个娃直接关了

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022225834.png)

而这个漏洞的入口点XSS也正是由于这个原因，网上的很多分析文章也都是到这里为止，算是进了个门口

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022226978.png)

# 从Swing XSS 到 RCE

## 从XSS到实例化任意类？

逻辑走到这里接下来最大的问题就是**如何从XSS走到RCE**，这也是这个漏洞最有意思的地方。

总所周知啊，一般来说我们常规意义上的XSS利用主要是围绕JS来做文章，即便是那种客户端的xss2rce，大多数也都是建立在Electron的基础上，说白了是在Node的环境下执行JS，由Node才从客户端走到服务端，才构成RCE。

而Swing不一样，它本质上是一个Java的组件，在Java环境上想要靠Xss来执行命令显然是天方夜谈，更关键的是，我们甚至没办法执行JS代码。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022226363.png)

而有意思的点就在这里，因为这是一套Swing自己实现的解析引擎，所以它选择不解析script那我们又不能凭空变出来执行js的方法。而恰恰是因为自己实现了这套引擎，Swing自己也整了一些花活。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022226632.png)

除了object其实也有一些奇奇怪怪的标签，但是大多数标签都被直接引到HiddenTagView了，只有少数的一些标签有专门的处理逻辑，object这个标签在这个逻辑里面显得有点儿特立独行，所以顺着这个逻辑直接往里走。

Path: swing.text.html.ObjectView.java

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022226970.png)

从这段代码看到，获取到classid的类会直接实例化并且相应传参，或许看代码可能还没看明白，这段代码上面还有一段范例。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022226639.png)

直接快捷复制一下这段代码也可以看到结果

```java
package com.company;

import javax.swing.*;

public class SwingDemo {
    private static void createAndShowGUI() {
        JFrame.setDefaultLookAndFeelDecorated(true);

        JFrame frame = new JFrame("cve-2022-39197");
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);

        JLabel label2 = new JLabel("<html><object classid='javax.swing.JLabel'><param name='text' value='sample text'></object>");
        frame.getContentPane().add(label2);

        frame.pack();
        frame.setVisible(true);
    }

    public static void main(String[] args) {
        javax.swing.SwingUtilities.invokeLater(new Runnable() {
            public void run() {
                createAndShowGUI();
            }
        });
    }
}
```

实际执行效果就是这个

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022226091.png)

好，现在我们获得了一个可以实例化任何类的入口，接下来的问题在于，我们需要找到一个满足所有条件的入口

## 自动化寻找符合条件的类

首先我们需要看看这个想要调用的任意类有什么要求



1、这个类必须有无参的构造方法

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022226243.png)

2、这个类必须继承于Component

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022226465.png)

3、传入的参数必须是String类型

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022226719.png)

4、这个传入的参数必须是可写方法，换句话说就是必须有setxxxx方法

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022226072.png)

5、这个setxxxx方法必须只接受1个参数

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022226198.png)



把前面的所有条件聚合一下就得出了我们要找的这个类的要求

1、一个继承于Component的类，其中必须包含无参的构造方法。

2、这个类中需要有一个setXXX方法，这个方法必须只接受一个参数，而且可以接受String类型。



现在我们有了一个新的问题，就是**如何找一个这样的类？找一个这样的方法？**

到这里我们遇到了一个大坎，我研究了好一段时间，网上常用的一些工具我也尝试过一些，有各种各样的bug，最后用来用去还是选了用代码来扫描判断。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022226156.png)

首先在IDEA导出继承自Component的所有类

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022226615.png)

然后直接写代码把所有类名导出来

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022226159.png)直接copy了pang0lin的代码(0v0)，扫描所有类满足条件的方法

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022226562.png)

最终得到满足条件的所有结果

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022226850.png)



## 寻找RCE利用链

逻辑走到这一部分其实基本路就走的差不多了，关键就是如何筛选出有意义的方法。归根结底，其实是这个特征比较特别，只接受一个string类型参数的函数，还是set方法，这种类型的方法大概率都是对于某个参数的预处理，很难涉及到RCE相关的代码。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022226967.png)

但这个先放在一边，首先可以去掉一些一看就没什么卵用的方法，比如setName、setLabel、setTitle、setAsText、setToolTipText。

然后还有很多那种多重继承一串的没有什么乱用的方法，就比如

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022226464.png)

说实话搞到这里，我脑子里想着都是，如果我当时写了java的代码分析工具就好了，我就可以直接扫了，到这块理论上来说只要有一个关键函数入口的判断就能省很多事了，只能快捷一个人工的审。

其实大概翻一翻swing库本身的几个方法，不难发现，swing的库几乎都是围绕窗口的，不是设置content type就是设置title，这类方法不用看都知道没啥用。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022226851.png)

所以回到思路本身上来看，问题就在于什么类型的这种方法有可能构成RCE，显然在通用库或者swing上我们找不到这种问题的答案，于是我们将视角放回到cs当中，在更具体的场景下才更有可能找到直接关联到底层的方法。

我们把cs导入到Libraries中，然后重复这样的扫描。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022226287.png)

其实到这里稍微筛选一下就很清楚了，主要是像前面一样，set函数基本上没什么功能，稍微花时间翻一下，我们得到最终的目标函数

```java
org.apache.batik.swing.JSVGCanvas-->setURI
```

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022227239.png)

说实话到这里，如果看其他人发的几篇分析中后续的思路还是都挺神奇的，因为作为web🐕的我，看到load svg肯定脑子里转不过来在svg里加载java代码的想法，我说实话也不知道beichen最早的漏洞逻辑是不是这么走过来的，哪怕是后续复现漏洞的朋友，这个思路也是很妖。

按照web的逻辑去想，一个web层面的东西怎么会联系到java呢，这就要往代码里进去跟进去看了。

首先随便搞一个简单的svg然后引用一下试试看

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022227474.png)

到这里最关键的是找到具体svg的解析逻辑，但是这块代码不是很好跟，一般来说到xml这里第一反应肯定是XXE，然后我就顺手去搜了一下，结果还找到这个库之前报过的一个XXE漏洞，但是这个漏洞在1.9版本就已经修复了，现在的batik版本已经是1.14。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022227528.png)

原文中展示了一个思路挺有意思的，其实走到svg第一反应肯定还是可以执行js，那么我们就尝试执行一个试试看

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022227071.png)

理所当然的报错了，这里我们顺着报错的执行顺序直接跟到具体的解析代码中

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022227324.png)

从这里可以找到一个关键方法org.apache.batik.bridge.BaseScriptingEnvironment.loadScripts

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022227361.png)

在这里的分支逻辑可以明显发现，如果var6不等于application/java-archive，我们就走到了刚才这个走不下去的分支，那么假设我们让var6满足条件，那么我们会走到什么分支呢？

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022227529.png)

从这里的代码可以明显发现，这里是从var26加载一个jar包，然后加载var13这个类，如果var26可控，那么我们就可以通过这种方式来构造RCE。

原文当中走到这里其实后续就是研究逻辑如何构造满足条件的svg文件逻辑了，但是到这里我想到，既然代码当中留了这样的一个功能，那么理论上来说就应该有类似的官方文档吧，于是开始顺着这个思路去找，首先发现的是，在cs漏洞曝光一段时间之后，batik也修复了这个漏洞。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022227278.png)

甚至还有相应的补丁可以找到

https://github.com/apache/xmlgraphics-batik/commit/905f368b50c2567cf2c4869a0ab596a7b1b5125c

顺着这个思路尝试去找这个功能相关的文档，没想到文档没找到，但是却找到了不应该找到的东西

https://www.agarri.fr/blog/archives/2012/05/11/svg_files_and_java_code_execution/index.html

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022227472.png)

直接进行一个抄

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022227765.png)

本地测试也顺利执行了

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022227574.png)

# 回到CS

在swing这边走完之后，我们先回到CS上，其实CS这边的问题也很简单，其实说白了就是长度问题，因为前面关于cs的交互包里提到过，我们可控的其实只有中间的会话信息部分，其中包括总共58的user、computer、process一共3个部分。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022227601.png)

而前面我们构造出来的payload大概是120个字符，也就是说，在这个数据包的构造下我们是肯定没办法利用这个的，其实这部分逻辑在漂亮鼠的文章中已经讲的很详细了，这个绕过的方式也很简单，要不就是想办法找到满足要求的payload，要不就是想办法绕过这个长度限制。

## 绕过长度限制

首先我们必须要明白一个问题，就是这个所谓的长度限制来源于哪里，在漂亮鼠的分析文章里提到了，这个所谓的117个字节的长度限制，是来自于rsa的加密字符串长度，这一点我们可以在cs_fakesubmit这个脚本当中获得验证。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022227354.png)

如果抛开太理论的部分，直接对上线包的做解密也能发现这个问题，可控部分位58个字符

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022227188.png)

换句话说，就是在数据包层面，这个长度是没办法绕过的，如果说上线包cs做了简单的长度限制和截断，但cs不可能所有的交互都有这样的限制，就比如最简单的获取进程列表，我们也可以从wireshark中确认这点。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022227676.png)

![img](https://cdn.nlark.com/yuque/0/2022/png/26687441/1667209803602-6d2deed2-0e00-4c2f-a307-d2a36cf28a70.png)

而这些大段的数据是通过aes来加密的，同样的也没有长度限制，比较可惜的是，我研究了一下没有找到获取AES密钥的办法，那可以从相对简单的逻辑去解决这个问题，最简单的方式就是想办法控制一个进程名，你可以通过hook底层函数的方式，也可以通过别的任何方式解决，比较可惜的是，windows不允许文件名中包含<>，否则直接新建一个文件都可以实现目标。

## 符合条件的payload

现在我们将视角转换一下，现在想办法把payload压缩到58个字符以内，如果说object这个表现天生就自带长度无法满足要求，我们就需要寻找一个办法引入一个外界的东西去解决这个问题。我们再将视角转回到swing上面。

按照我们正常的思路的话，无非就两个办法，一个是想办法引入外界的链接或者脚本，比如iframe或者link这类。要不就是想办法拼接多个字符串。

前面的代码分析中也详细提到过，swing自己实现了一套标签解析逻辑，其中大部分的标签都没有实际的功能，而frame标签刚好不同。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022227155.png)

这个代码逻辑有点儿怪，如果你直接输入

```plain
<html><frame src=x>
```

那么他会走到frameset逻辑里，然后就会有frameset的格式限制，但如果你在frame标签前面加入一点儿字符就能走进frame逻辑，说实话我翻了一下代码也没找到相关的逻辑，不过这个不重要，重要的是后续的问题。

```plain
<html>a<frame src=x>
```

如果我们尝试这样的payload，那么会报这样一个错

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022227964.png)

报这个错的原因是在逻辑中有强制类型转化错误

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022227712.png)

这里非常怪，因为在我看来这里的报错没有道理，如果说的代码本身没有bug，那么一定是哪里出了问题，我也在网上查了查相关的问题，也有的帖子说这是一个jdk8的问题，但我本地测试是不止影响jdk8的，关键在于这个被强制类型转化的类是从哪来的。然后追溯了一下这个frame的官方调用逻辑发现这个区别在于父组件的类型，如果测试代码是这样写的就可以触发逻辑。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022227483.png)

那么顺着这个思路我去cs的代码里找找有没有类似的逻辑，结果果然找到了类似的东西而且的确可以触发逻辑链

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022227683.png)

在cs里面出现这种代码的位置有几个，主要包括

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022228732.png)

其中dialog.DialogUtils.java这段代码更像可控点

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202211022228111.png)

比较可惜的是，大概翻了一下相关的代码，没有找到那种明显可控的位置，更像是cs的二开或者插件会调用到的函数，而且这个东西也没法调试，算是比较麻烦的一点。这里就先不深入研究这里了，毕竟到这里基本上等于控制非上线包的功能了，其实逻辑基本上和前面模拟交互包类似。

由这里反推前面也能发现，其实frame这个标签本身使用方式没问题，而是场景问题，就像其他几篇文章里面提到的，在部分特定条件下，这个强制类型转化也是可以成功的，而在JEditorPane的组件场景下，如果可控那么payload也是同一个。

# 写在最后

其实这篇文章短短续续的写了很长时间了，由于不熟java，所以中间文章一度断了很久直至最近才陆陆续续把文章写完，其实思路大体上还是顺着几篇别人的文章来顺着，不得不说还是很佩服挖漏洞以及复现漏洞的几位朋友，中间好几个点我个人感觉思路都很神奇，算是学到了不少东西。
