---
title: 智能合约游戏之殇-类Fomo3D攻击分析
date: 2018-08-24 10:15:22
tags:
- eth
- Fomo3D
- 智能合约
---

**当你还在寻求棋局中的出路时，却发现棋盘已经不存在了。**

<!--more-->

2018年8月22日，以太坊上异常火爆的Fomo3D游戏第一轮正式结束，钱包开始为0xa169的用户最终拿走了这笔约**10,469 eth**的奖金，换算成人民币约2200万。

看上去只是一个好运的人买到了那张最大奖的“彩票”，可事实却是，攻击者凭借着对智能合约原理的熟悉，进行了一场精致的“攻击”！

这次攻击的结果，也直接影响了类Fomo3D的所有游戏，而且**无法修复，无法避免**，那么为什么会这样呢？

# 类Fomo3D #

在分析整个事件之前，我们需要对类Fomo3D游戏的规则有一个基本的认识。

Fomo3D游戏最最核心的规则就是**最后一个购买的玩家获得最大的利益**

其中主要规则有这么几条：
- 游戏开始有24小时倒计时
- 每位玩家购买，时间就会延长30s
- 越早购买的玩家，能获得更多的分红
- **最后一个购买的玩家获得奖池中48%的eth**

其中还有一些细致的规则：
- 每位玩家购买的是分红权，买的越多，分红权就会越多
- 每次玩家购买花费的eth会充入奖金池，而之前买过的玩家会获得分红
- 随着奖池的变化，key的价格会更高

换而言之，就是越早买的玩家优势越大。

最终，资金池里的 ETH 48%分配给获胜者， 2%分配给社区基金会，剩余的 50%按照四种团队模式进行分配。

游戏规则清楚之后，就很容易明白这个游戏吸引人的地方在哪，只要参与的人数够多，有人存在侥幸心理，就会有源源不断的人投入到游戏中。游戏的核心就在于，庄家要保证游戏规则的权威性，而区块链的可信以及不可篡改性，正是完美的匹配了这种模式。

简单来说，这是一个**基于区块链可信原则**而诞生的游戏，也同样是一场巨大的社会实验。

可，问题是怎么发生的呢？让我们一起来回顾一下事件。

# 事件回顾 #

2018年8月22日，以太坊上异常火爆的Fomo3D游戏第一轮正式结束，钱包开始为0xa169的用户最终拿走了这笔约**10,469 eth**的奖金。

![image.png-80.9kB][1]

看上去好像没什么问题，但事实真的是这样吗

在Fomo3D的规则基础上，用户a169在购买到最后一次key之后，游戏的剩余时间延长到了3分钟，在接下来的3分钟内，没有任何交易诞生。这3分钟时间，总共有12个区块被打包。但没有任何一个Fomo3D交易被打包成功。

![image.png-82.4kB][2]

除此之外，这部分区块数量也极少，而且伴随着数个合约交易失败的例子
![image.png-93kB][3]

这里涉及到最多的就是合约`0x18e1B664C6a2E88b93C1b71F61Cbf76a726B7801`,该合约在开奖的那段时间连续的失败交易，花费了巨量的手续费。

而且最重要的是，该合约就是上面最后拿到Fomo3D大奖的用户所创建的
![image.png-107.8kB][4]

在这期间的每个区块中，都有这个合约发起的巨额eth手续费的请求。

攻击用户通过这种方式，阻塞了其他游戏者购买的交易，最后成功拿到了大奖。

那么为什么呢？

# 事件原理 #

在解释事件发生原理之前，我们需要先了解一下关于区块链底层的知识。

以太坊约14s左右会被挖出一个区块，一个区块中会打包交易，只有被打包的交易才会在链上永不可篡改。

所以**为了奖励挖出区块的矿工，区块链上的每一笔交易都会消耗gas，这部分钱用于奖励矿工**，而矿工会**优先挑选gas消耗比较大的交易进行打包**以便获得更大的利益，目前，一个区块的gas上限一般为8000000。

而对于每一笔交易来说，交易发起者也可以定义gas limit，如果交易消耗的gas总值超过gas limit，该交易就会失败，而大部分交易，会在**交易失败时回滚**。

为了让交易不回滚，攻击者还使用了一个特殊的指令`assert()`，这是一个类似于require的函数，他和require唯一的区别就是，`当条件不满足时，assret会耗光所有的gas`。原理是因为在EVM底层的执行过程中，assret对应一个未定义过的操作符`0xfe`，EVM返回invalid opcode error，并报错结束。

而攻击者这里所做的事情呢，就是在确定自己是最后一个key的持有者时，发起超大gasprice的交易，如图所示：

![image.png-80.6kB][5]

当攻击者不断的发起高手续费的交易时，矿工会优先挑选这些高花费的交易打包，这段时间内，其他交易（包括所有以太坊链上发起的交易、Fomo3D的交易）都很难被矿工打包进入。这样一来，**攻击者就有很高的概率成为最后一个持有key的赢家**

整个攻击流程如下：
1）Fomo3D倒计时剩下3分钟左右
2）攻击者购买了最后一个key
3）攻击者通过提前准备的合约发起大量消耗巨量gas的垃圾交易
4) 3分钟内不断判断自己是不是最后一个key持有者
5）无人购买，成功获得大奖


在支付了大量以太币作为手续费之后，攻击者赢得了价值2200万人民币的最终大奖。

# 总结 #

自智能合约游戏中以类Fomo3D诞生之后，这类游戏就不断成为人们眼中的焦点，精巧的规则设计和社会原理再加上区块链特性，组成了这个看上去前景无限的游戏。Fomo3D自诞生以来就不断成为人们眼中的焦点，类Fomo3D游戏不断丛生。

随之而来的是，有无数黑客也在盯着这块大蛋糕，除了Fomo3D被盗事件以外， Last Winner等类Fomo3D也被黑产盯上...短短时间内，攻击者从中获利无数。

而我们仔细回顾事件发生的原因，我们却不难发现，类Fomo3D游戏核心所依赖的**可信、不可篡改原则**和区块链本身的特性`矿工利益最优原则`冲突，也就是说，**只要矿工优先打包高手续费的交易，那么交易的顺序就是可控的！**，那么规则本身就是不可信赖的。

**当你还在寻求棋局中的出路时，却发现棋盘已经不存在了。**

**当Fomo3D游戏失去了自己的安全、公平之后，对于试图从中投机的你，还会相信自己会是最后的赢家吗？**


# REF #

\[1\] Fomo3D
[https://exitscam.me/play](https://exitscam.me/play)

\[2\] 获利交易
[https://etherscan.io/tx/0xe08a519c03cb0aed0e04b33104112d65fa1d3a48cd3aeab65f047b2abce9d508](https://etherscan.io/tx/0xe08a519c03cb0aed0e04b33104112d65fa1d3a48cd3aeab65f047b2abce9d508)

\[3\] 攻击合约
[https://etherscan.io/address/0x18e1b664c6a2e88b93c1b71f61cbf76a726b7801](https://etherscan.io/address/0x18e1b664c6a2e88b93c1b71f61cbf76a726b7801)

\[4\] Fomo3D 千万大奖获得者“特殊攻击技巧”最全揭露
[https://mp.weixin.qq.com/s/MCuGJepXr_f18xrXZsImBQ](https://mp.weixin.qq.com/s/MCuGJepXr_f18xrXZsImBQ)

\[5\] 「首次深度揭秘」Fomo3D，被黑客拿走的2200万
[https://mp.weixin.qq.com/s/s_RCF_EDlptQpm3d7mzApA](https://mp.weixin.qq.com/s/s_RCF_EDlptQpm3d7mzApA)


  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/ny8ikv87ex1y5ak6k7ctgek8/image.png
  [2]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/wd1m1sn4sccybe4owzhabnt7/image.png
  [3]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/wbtd3tmc5pykdo254ulule93/image.png
  [4]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/l3b4vd61s8wu1wt638xai3r4/image.png
  [5]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/j776fccqmuyk574l6gp27fpp/image.png
