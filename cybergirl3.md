---
title: 赛博偶像速成指南（三）- Midjourney
date: 2023-06-21 17:27:49
tags:
- chatgpt
- midjourney
---

之前的几篇关于AI生成图片的文章讲的都是**stable diffusion**，虽然SD**出现的更早而且开源免费**，但其实在设计圈使用更广泛的是**Midjourney**，**Midjourney最大的优点就是使用的便利性，任何一个不懂技术的设计都可以通Midjourney来快速完成设计，而且Midjourney的底层基础模型成熟度相当高，生成的图质量都很高。**

<!--more-->

# 快速入门

注册Discord并授权使用Midjourney的bot

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202306211729980.png)

现在的Midjourney已经正式转为**收费版本**了，价格还是比较便宜的，最便宜的版本一个月60多块也还算能接受。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202306211729280.png)

订阅完成之后就**可以加入Midjourney的频道，在左边可以选择newbie或者general的频道**进去

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202306211729966.png)

你可以直接在这个频道里使用，当然，这个频道里消息非常多，刷的很快，也有一个办法是你可以**在自己的私人频道使用这个bot**。首先你需要有一个你**拥有管理权限的频道**，然后在Midjourney中找到这个bot并把它添加到自己的服务器中。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202306211729624.png)

然后就可以**正常的使用命令**了。

**最常用的命令就是/imagine**，输入命令会直接引导格式让你输入对应的prompt，但很可惜的是Midjourney本体**对中文的支持并不是很好**，生成结束之后就会弹出结果。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202306211729655.png)

下面的几个选项对应的是**上面图的进一步发展方向**

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202306211729743.png)

其中**U是选定并放大其中的一张图，V是选定一张图在这个图的结构上进一步生成**，如果都不满意还可以重新生成一次。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202306211729119.png)

假设我们选择U1之后，他会获得一个**放大版本的更多细节的图。**

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202306211729840.png)

之后如果你还想进一步生成，还可以点选**Make Variations来继续生成之前的4格图**继续风格的演变和生成。

# 进阶配置

Midjourney的基础用法就是**用imagine配合基础的文字描述**来生成图片

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202306211729744.png)

但事实上，Midjourney还有**很多进阶选项**

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202306211729676.png)

首先你可以通过**传入图片链接来使用图生图功能**。除此之外，还有一些参数可以提供。

- **--aspect，--ar：**修改生成图片的纵横比. 

ps: --aspect 2:3

- **--chaos <number 0–100>：**改变结果的多样性，值越高变量越大
- **--iw <0–2>：**设置图片prompt和文字prompt的权重比例，默认值是1
- **--no Negative prompting：**否定提示词，比如说--no plants可以去除图里的植物

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202306211729328.png)

- **--quality <.25, .5, or 1>：**花费多少渲染质量时间，默认值是1
- **--repeat <1–40>，--r <1–40>：**重复多少次
- **--seed <integer between 0–4294967295>：**随机数种子
- **--stop <integer between 10–100>：**在中途停止生成
- **--style <raw>：**切换Midjourney的版本
- **--stylize <number>，--s <number>：**Midjourney美术风格的强度
- **--tile：**重复图块以生成更大的图

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202306211729601.png)

除此之外你也可以指定使用Midjourney不同版本的基础模型

- -**-niji：**适用于动漫风格的基础模型
- **--version <1, 2, 3, 4, or 5>：**不同版本的Midjourney算法

# 继续拓展

在使用Midjourney的过程中，能最明显感觉到的是，Midjourney其实是通过**大幅度简化来实现产品使用体验的大幅度提升**，使用Midjourney你并**不能像SD一样在很多地方影响图片的生成逻辑**，拿生成赛博偶像的系列图举例子，你**很难生成同样外形的多个系列图片**。

而Midjourney这个工具其实更聚焦于设计师群体，重要的是你想要画一个什么东西出来，**一个符合预期的prompt提示词，配合多轮的迭代式绘画改进，你基本上可以用Midjourney画出一个满意的东西。**

而我觉得Midjourney除了**超棒的交互式体验**，最牛的就是**底层关键词的解读逻辑有效度非常之高，而且基础模型的审美非常不错**，你大概写一个描述词基本上都能得到一个质量非常高的图，这点是其他绘画ai都达不到的高度。

举个例子，**a hacker before computer** 

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202306211729209.png)

如果是在**文心一言**，你就会得到这样一个图

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202306211729605.png)

这里你能很明显的感受到他们的区别，就是**Midjourney生成结果质量非常之高**，你还可以让他进一步生成，如果你觉得直接选择V变化太小了，你可以选择**用大图来重新图生图。**

比如这里我拿第二个图进行重新生成，**加入赛博朋克风格**

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202306211729504.png)

当然你可能没有什么想法，但你只是喜欢这个画风，那你**可以通过chaos这个参数来增加多样性**，当然这个值建议不要设置太大，否则基本和原图就关系不大了。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202306211730156.png)

**你可以在midjourney上轻而易举的获得你想要的任何改动，不用担心你的需求无法理解，Midjourney会不知疲倦的持续产出各种东西来。每次用这个东西的时候都能感觉到设计师的悲惨。**

之前也曾经听过一个事情，设计师届曾**大量的抱团反对ai绘画工具的诞生，用设计师设计的东西训练ai，最后反倒毁掉了设计师的工作**，真是令人唏嘘。

# 在线部署Midjourney

前面说了很多关于Midjourney的使用问题，但对于国内来说Midjourney最大的问题就是discord是需要科学的。所以接下来介绍一个可以在国内部署的方案

- https://github.com/novicezk/midjourney-proxy

你可以使用这个项目来部署一个Midjourney的代理，我觉得比较好用的是部署在Railway的版本，不需要用自己的服务器，这个网站提供免费的5美刀和每个月免费的500小时使用。

- https://github.com/LoRexxar/midjourney-proxy/blob/main/docs/railway-start.md

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202306211730156.png)

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202306211730645.png)

部署完成之后可以获得一个mid的链接

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202306211730344.png)

这个时候还只有api，你需要配合一个前端来完成，我选用了魔改版的ChatGPT-Midjourney

- https://github.com/Licoy/ChatGPT-Midjourney

这个东西可以直接部署Vercel版本的，非常好弄，直接点击deploy就可以开始配置了

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202306211730562.png)

部署好了绑定域名就可以用了，很方便。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202306211730247.png)
