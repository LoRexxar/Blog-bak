---
title: 从0到1的ChatGPT - 进阶篇（三）- ChatGPT+？
date: 2023-04-28 18:56:52
tags:
- chatgpt
---

在我们对ChatGPT的基础能力有了一定的了解之后，我们就要开始在ChatGPT的基础上探索更多的可能性。

而ChatGPT本身的问题也很多，ChatGPT在使用上最大也最明显的革命，**其实是对自然语言的处理能力**，抛开太多专业性的术语，你在使用的过程中也能明显感觉到，ChatGPT甚至在某些方面有着比正常人更厉害的解读能力，**它可以把一段模糊的要求和文字解读成需求，最牛逼的是它还支持中文**，毕竟理论上中文的自然语言处理难度是几个量级。

<!--more-->

拿下面这个图举例子，我感觉我都没说明白，但却获得了答案。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304281858932.png)

除了强大的自然语言处理能力，以及大模型背景下庞大的数据以外。ChatGPT还有很多明显的缺点，其中**最直白的问题就是数据的过时以及不联网问题。**目前ChatGPT的训练数据集截止到2021年，而没有准确数据集的数据，ChatGPT就没有置信数据可以参考，**而这类问题ChatGPT就会通过某种方式自我学习产生，而他的结果就会产生各种各样的错误。**

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304281858889.png)

当然，为了避免大数据污染等等问题，ChatGPT目前公开对外使用的接口，最多只会参考部分上下文以及限定单个对话session中做学习优化，但不会对用户的输入做学习。

为了解决这个问题，ChatGPT选择了**用第三方插件作为媒介**，让AI在比较安全的环境学习外界的数据，最早的合作公司由Expedia、FiscalNote、Instacart、KAYAK、Klarna、Milo、OpenTable、Shopify、Slack、Speak、Wolfram 和 Zapier 创建。

- https://openai.com/blog/chatgpt-plugins

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304281858263.jpeg)

其中的各种插件可以覆盖大部分的场景，包括：

- 检索实时信息：例如体育比分、股票价格、最新消息等；
- 检索知识库信息：例如公司文件、个人笔记等；
- 代表用户执行操作：例如，订机票、订餐等。

除此之外呢，ChatGPT官方还提供了两个插件，一个是网络浏览器，另一个是代码解释器，并开源了一个知识库检索插件的代码。现在，**任何开发人员都可以自行构建插件**，用来增强 ChatGPT 的信息库了。从这里开始ChatGPT+的概念算是诞生。

虽然目前这部分的插件还只是**开放给了候补名单中的用户和开发人员**，但计划中即将开放给部分ChatGPT plus的用户了。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304281858517.png)

# 一些有趣的ChatGPT周边

除了ChatGPT官方的插件以外，还有很多以各种各样的方案实现的ChatGPT衍生产品，其中有很多意思的东西，这里我就推荐几个比较有意思的

## WebChatGPT

https://chrome.google.com/webstore/detail/webchatgpt-chatgpt-with-i/lpfemeioodjbpieminkklglpmhlngfcn

WebChatGPT是一款Chrome的插件，它可以用一个特殊的方案来实现ChatGPT的联网，来让ChatGPT的返回数据更准确更新。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304281858086.png)

他的实现方案特别有意思，**简单来说，他会先把你的问题拿去搜索引擎上搜，然后把结果喂给ChatGPT，然后让ChatGPT以搜索结果作为上下文学习，之后再回答你的问题。**

插件安装成功之后，你的对话框上多了很多的参数

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304281859186.png)

Web access（是否要开启联网功能）、X results（想要它列出几条来源）、Time（多久之前的资料）、Region（哪个地区的资料），以及最右边的Prompt（默认指令）

**通过配置参数并开启，你可以获取到非常有时效性的内容**，比如说询问天气。因为正常来讲你询问chatgpt天气会返回这样的内容。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304281859491.png)

但如果你使用这个插件，你就可以获取这样的结果。



但要注意的是，**由于插件的实现方式（通过输入搜索内容关联上下文），使用WebChatGPT会大幅度削减原本对话中的上下文关联度，所以一般来说只有特定的场景下才使用这个插件。**

## Monica

[https://chrome.google.com/webstore/detail/monica-%E2%80%94-your-chatgpt-cop/ofpnmcalabcbjgholdjcjblkibolbppb](https://chrome.google.com/webstore/detail/monica-—-your-chatgpt-cop/ofpnmcalabcbjgholdjcjblkibolbppb)

这个插件是Google for ChatGPT的进阶，**它集合了你在上网过程中会遇到的各种问题和场景，并通过chatgpt来辅助使用。**

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304281859512.png)

当然，这个玩意现在越来越成熟了，所以它也开始收费了，大家可以自己感受一下

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304281859993.png)

这个插件最常见的功能就是搜索辅助，会**直接对你google的搜索结果做优化，直接返回你i想要查询的结果**。可以大幅度节省找到答案的时间。最牛的是，**它还支持百度**。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304281859494.png)

除此之外，还有一些比较有意思的东西，比如说划词右键解释

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304281859218.png)

使用ctrl+m可以打开侧边，这里有两个功能，一个是聊天和写作辅助

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304281859984.png)

它还可以提供阅读功能，可以直接给它一篇文章，然后让他阅读，他会直接给你返回文章的摘要。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304281859722.png)

你可以通过这个对话聊天功能来快速的阅读一篇文章。

## AutoGPT

- https://github.com/Significant-Gravitas/Auto-GPT

AutoGPT诞生没多久，比起这个东西本身的效果，AutoGPT本身的思路和理念很有意思，**AutoGPT依托于GPT4强大的算力和思考能力，对你的需求进行解构深入**。比较麻烦得是，**AutoGPT对算力的依赖比较强，GPT3.5能用但是很难用。而且由于AutoGPT的理念**

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304281859513.png)

**AutoGPT就像一个不知疲倦的实习生，他会对你下的指令进行多重解构，并对当前的问题持续发散探索更多话题**。

这里有个小例子，假设我想要一个web扫描器但是没有指定任何要求（你可以通过增加要求来优化回答的导向

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304281859044.png)

auto-gpt在分析了我的需求之后，提出了一个计划，是先去找一个网上的扫描器然后再根据需求改进，还提醒我小心代码的安全问题，不要盲目的clone代码。

如果我们继续让他分析，他会继续把分析的内容细化并深入，你也可以指定连续多步去分析。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304281859109.png)

当然除了让他继续分析，你也可以给与一些人工的干预，比如我说我不想要别人的代码，我想要自己写，他指出我们需要好好分析需求。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304281859302.png)

其实现在阶段的AutoGPT更多还是一个概念产品，使用上的体验更接近一个demo。**比起实际意义，AutoGPT通过反馈较正的方式给我们呈现了一种机器思考的感觉，很有趣。**

## 写在最后

这篇文章里我们讨论的大多都是现在已经成型的一些基于chatgpt的拓展，作为使用者我们能做的很多只有适应时代，下篇文章我们就讲讲，在chatgpt的基础上我们作为开发者能做什么？
