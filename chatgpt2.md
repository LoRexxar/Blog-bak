---
title: 从0到1的ChatGPT - 入门篇（二） - 如何与ChatGPT对话？
date: 2023-04-26 19:04:11
tags:
- chatgpt
---

在上篇文章的结尾，我提到了ChatGPT其实更像是一把铲子，在拥有这把铲子之前，我们只知道可以把土堆成房子，但是不知道用什么把土堆起来，但在有了这把铲子之后，铲土只是铲子最直白的利用，如何用铲子堆一个又大又漂亮的房子可能我们还不知道，但至少我们现在已经开始尝试做这样的事情了。

其实从ChatGPT诞生至今，所有从事相关研究的朋友都在努力的在ChatGPT上探索各种各样的使用方式，甚至现在已经诞生了所谓的prompt工程师。

这篇文章就聊聊很多现在已有的关于ChatGPT使用的技巧。

<!--more-->

# 4A & 4W

首先ChatGPT在自然语言的理解上虽然有着领先时代的表现，但事实上ChatGPT并不是你的蛔虫，你试图通过简单的问题获得准确的回答是不可能的，也不现实。

这里我也用一下，在讲述这个问题时最常用的“如何减肥”的例子。如果你只是简单的问，那么chatgpt的回答就会模糊而概括。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304261915275.png)

随着大家的探索，逐渐诞生了两种常用的扮演法指令模式，也就是4A & 4W。

- 4A: Actor(角色) - Aim(目标) - Ask(提要求) - Addition(补充)

4A模型是Prompt中比较典型的例子，晚上大部分流行的提问方式都是这个结构，还是拿减肥举例子，这一次我提供了我的身高和体重，并且给他赋予了角色定位。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304261917114.png)

相比之前更简单的提问，ChatGPT给了更具体的回应以及更详细的范例，但实际上在这个范例中，虽然内容详细但事实上没有太具体的计划。

在这个基础上，又有人提出了4W模型。

- 4W: What(我的情况是) - Will(我想) - Who(你是谁) - Want(我要你)

我们把前面的问题换个问法

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304261917325.png)

这一次ChatGPT反馈的最大变化就是他会根据我的内容进行发散，进而进一步的反馈详细的内容和反馈。

事实上ChatGPT对于问题的回答并不是一定的，相比4A模型，4W模型的反馈质量更高反馈也比较直白，在GPT4版本之后发散度也更高，也是现在比较主流的扮演提问法。

但事实上，4W的基础提问法只是比较通用的问法，但在扮演法可以有更详细的提问方式。比如我们先问问有没有专业的健身教练。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304261917970.png)

根据他的反馈，我们直接找其中一个人，让ChatGPT扮演这个人来提供建议。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304261917172.png)

在这种情况下，ChatGPT有可能，注意是有可能，会生成带有强烈的个人风格的反馈内容。而这部分内容一般来说有效度会更高，因为他很有可能是基于已有的内容生成的。但这并不绝对，因为ChatGPT还没有真正意义上联网。不过使用这种更详细的扮演法在某些情况下会让你的结果更有效。

# Openai 的官方最佳实践

这里我们也一起看一看openai公开的prompt最佳实践，里面其实也是提到了一些我们熟知的，这里我提取几个比较关键的点。

1、把指令放在Prompt的开头，并且用###或者"""来分割指令和上下文。

```plain
我需要把下面这段代码压缩到一行
###
var cookieStr = 'ppmsglist_action_3907326541=card';
var cookieArr = cookieStr.split('; ');
for (var i = 0; i < cookieArr.length; i++) {
	var cookie = cookieArr[i];
	var arr = cookie.split('=');
	document.cookie = arr[0] + '=' + arr[1];
}
```

2、对希望得到的内容的背景、结果、长度、格式、风格尽可能的详细

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304261917854.png)

3、通过示例阐述所需的输出格式

其实也很好理解，你可以用一些范例来表达你想要的内容，来帮助chatgpt矫正结果。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304261918159.png)

![img](https://cdn.nlark.com/yuque/0/2023/png/26687441/1682335543484-60940a89-2cac-4d8d-9667-8c14a397e52d.png)

4、先不提供范例，再尝试给出范例，然后根据返回微调。

5、减少不精确的描述。

比起“我想要一段短小的内容”，最好直接指明内容的长度，比如“我想要一段100字左右的内容”

6、与其说什么不该做，不如说什么该做。

7、在想要生成代码的时候，使用引导词让模型向特定的模式发展。

在openai的范例中，他用import作为python的引导词，用select作为sql的引导词。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304261918631.png)

# Prompt Engineering

其实相比简单的提问式回答，现在的Prompt相关的内容已经相当成熟了，比如github上现在有很多类似的项目，整理了大量的经典Prompt场景

- https://github.com/PlexPt/awesome-chatgpt-prompts-zh

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304261918427.png)

甚至已经有相当成熟的网站分享相关的信息https://www.explainthis.io/zh-hant/chatgpt

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304261918478.png)

除了这种简单的指令分享，甚至还有更牛逼的直接把这个东西直接包装成产品，直接辅助你去写各种prompt。

# 额外参数

除了简单的对话技巧以及各种方案，ChatGPT还提供了不少的额外参数以影响返回的结果，其中我挑部分我觉得比较有意思的参数

## temperature

temperature这个参数官方给出的解释是，衡量模型输出不太准确信息的频率，temperature越高，输出越随机，并更具有创造性。但相比官方的解释，我们甚至可以把temperature理解为情感值或者温度值。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304261918453.png)

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304261918806.png)

temperature默认是0.8，最高为2。通常来说，在询问具有创造力的结果时，可以让temperature提高，来获得更有意思的结果。在询问某些事实或者准确的内容时，可以降低temperature来获得更准确的结果。你可以在调用api的时候设置这个参数来控制它。

这是temperature为0.2时返回的结果。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304261918267.png)

当temperature为2的时候，chatgpt就有点儿傻，返回的非准确内容中会大量的随机各种结果。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304261918349.png)

当temperature为1的时候，chatgpt相对比较平衡

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304261918640.png)

## presence_penalty

presence_penalty也是一个比较常见的参数，我们可以把这个参数认为是话题新鲜度，也可以认为是话题拓展的可能性。这个值越大，chatgpt在对话中也会越主动的发起新的分支。

这个值默认是0，可以从-2到2之间。

我自己尝试了一下感觉这个参数的表现其实比较弱，一般的问题回复其实是感受不到的。

## frequency_penalty

frequency_penalty整体上和presence_penalty类似，主要是控制总体使用频率较高的单词和短语概率，这个值越高，chatgpt中就会尽量减少重复。

这个值默认是0，可以从-2到2之间。

## max_tokens

标志返回的token长度的硬截止限制，这个token之前也说过其实标志是的是单词或者短语，这个计算方式相当宽泛，所以一般设置max_tokens就是为了保底，避免某些特殊的问题导致超长的回复浪费api的资源。

## stop

一个特殊标记，可以在文本生成过程中暂停文本的生成。

# 写在最后

掌握了ChatGPT简单问答式的用法，就相当于我们已经学会了用铲子铲土。

而在ChatGPT基础上做进一步的探索相当有趣，下篇文章就讲讲怎么用铲子盖房子。
