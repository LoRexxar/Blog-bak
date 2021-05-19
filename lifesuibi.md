---
title: 创宇四年
date: 2021-05-19 16:06:49
tags:
- 随笔
---

---

创建下这篇文章的时候，距离我倒计时离开公司还有2天，事情都是一些琐事，也很难提得起兴趣再做点儿什么，翻了翻还没写的代码，始终是没有动力开始写一个新功能，心中总是百感交集，往事一件件的过，不得不承认我是一个很内向但却是个表达欲望很强烈的人，即便心里深知在离职后谈论过去是很不合适的行为，但仍旧没忍住创建这篇文字。

也许是因为年纪正处于承上启下的关键，稍微停下来不知道该干什么的时候，有些矫情劲总是过不去，忍不住的回忆起过去的一件件过往，有点儿青春期感时伤怀的那味儿。

这些年北漂虽然说大体上过得还是比较符合心意，但仍旧经历了不少，回头看的仍旧不免感叹，既然起笔了，就从大三的暑假开始吧。



流水账警告！

<!--more-->

# 实习

那是刚开始北漂的时候，大致是大三下半学期的时候，协会的学长@Hcameal当时在知道创宇的404实验室实习，有次问我们有没有人想去实习。当时的我还不太明白想要找一份什么样的工作，只是尝试性的投递过阿里，但是被拒绝了（坐标是大二大三的暑假）。时至大三，当时是我变化比较快的一段时期，大量的CTF经历催着我不断成长，必须自豪的说，当时在国内同年龄里做Web这块我还是比较有信心的。当时心里想着毕业之后一定要回北方（学校在杭州），再加上我在入行前期曾受到过当时知道创宇404实验室@Evi1m0、@余弦、@heige很多影响，所以当时觉得知道创宇是不错的公司。

比较神奇的是，因为各种巧合，我是被免面招进创宇的（实习），甚至都没有打电话，就加了个微信，就把我算作通过了。阴差阳错的，我也就没太纠结要不要投一下其他家，也就顺水推舟的进了公司实习。可惜的是，当时@余弦离开公司去做区块链安全，@Evi1m0离开公司去了B站做安全负责人，老404几乎没剩下什么人了，新404基本上都是我这一届和上一届的人组成。

2017年6月30号，我排除万难跑到北京，和当时的一个同事一起在顺义（离公司真的好远）租了房子，也接受了我的第一份工作，初来公司还是畏惧感比较多吧，不知道能做什么，也不知道该干什么。当时主要的工作是做漏洞应急，也就是刷每天的漏洞咨询，如果看到有人放出来的漏洞，就会去研究一下内容，看看有没有意思。当时我的主要工作就是做漏洞应急，然后复现漏洞，如果漏洞还不错，就会写一篇分析，熟悉我的朋友也会知道，2017年的时候我写了大量的漏洞分析文章（在17年之前，我95%的文章都是CTF的Writeup）。

有趣的是，在长期的漏洞复现+漏洞分析过程中，我慢慢开始学习其他人挖掘漏洞的方式，通过分析同类漏洞，我也水了不少CVE（那个年代CVE基本上申请就可以拿到），于是我的工作逐渐也变成了漏洞应急、漏洞分析、漏洞挖掘。

在一个心血来潮的下午，当时我工作正处于长期漏洞挖掘中，在挖掘前期我总是会用一下当时比较有名的Seay扫描一下，但是不得不说基于正则的静态扫描方案实在是太垃圾了，于是在协会的群里就讨论起这个来，看看有没有什么解决方案。协会的学长@lightless当时在蘑菇街正好是做Cobra的，于是我的生命中悄悄的出现了一个新的分岔路口，就像我博客长期挂着的一句话一样。

**“当你老了，回顾一生，就会发觉。什么时候出国读书，什么时候决定做第一份职业，何时选定了对象而恋爱，什么时候结婚，都是命运的巨变。只是当时站在三岔路口，眼见风云千樯，你做出选择的那一日，在日记上，相当沉闷和平凡，当时还以为是生命中普通的一天。” 摘自陶杰《杀鹌鹑的少女》**

也不知是不是神无意中挥动的一根手指。在初次使用Cobra获得了很不好的体验之后，我气急败坏的开始了魔改Cobra的旅途，彼时正值Cobra刚刚开源，本来就有很多功能不好用，而且我的魔改方向与蘑菇街朋友的方向不一致。所以从开始的发pr，到后来我直接独立开始完善Cobra-W。此处也埋下一个大伏笔。

想起来非常好笑的是，当时我python水平仅限于写一个大爬虫，我却要魔改人家一个人团队几千行的代码。但反过来想，也许不是因为这种勇气，我的python水平也不会快速增长，为了研究明白基于AST的静态分析原理，我还去重读了了一遍编译原理的前半部分~

时间走到这里基本上已经是17年的年底吧，我的工作也变成了有漏洞就做漏洞分析，然后汲取相应的养分。没事的话，我就啃那本编译原理顺便研究怎么优化Cobra。作为生命中的小插曲也因为各种事情变成了主旋律，在临别北京前2个月，没忍住表白了在公司认识的很有好感的女孩，当时正值是否普信的自我质疑中。幸运的是，老天爷总会给你一些安慰，于是和我女朋友确认了关系也顺利的延续至今。这里也是一个很大的伏笔。

在创宇的实习期算是非常不好过吧，不得不说对于当时的我来说实习工资真的不高，只刚好够支付我所有的生活费用，为了支撑我的消费，当时我还跑过几个培训和比赛为了挣钱，也经历了熬夜4天做ppt，周五晚上飞外地，周日晚上飞回来的辛苦，想想还是比较有趣的。（那个年代还没有HVV外快赚）

在创宇实习期最大的收获可能就是收获了很多面对真实世界漏洞的经历，也快速提高了一波自己的开发知识和一些静态分析的经历，第一个有趣的点就是，研究过静态分析的这个经历是我在后面找正式工作时最大的亮点，在面试的过程中，面试官也更愿意从这个角度和我深入聊下去。

# 大四下以及找工作

时间静悄悄的往前走，我在创宇的实习一不小心就走过2018年的新年，我也开始准备返校。在年前后，我也开始琢磨自己的正式工作该何去何从。印象比较深的是，当时非常希望加入某巨大厂的某实验室，在我心目中应该是当时我最向往的No.1目标，于是在11月举办HCTF2017决赛的第二天，我还当晚飞回北京，就是为了参加这个厂的最后一面，可惜了，成也萧何败也萧何，在11月这次面试之后，我为了等这个回复，鸽了某DD的Offer，甚至错过了秋招黄金期。（当时还是坚持想去自己最想去的公司，所以最开始只投了一家，不想做取舍。）可惜的是，在年后一个月，我才得到明确的消息，我没能得到这份工作。

我相信懂哥们都明白，我只剩下春招的机会了，于是我找朋友投递了某60的某实验室，某巨大厂的某蓝军，某DD的安全工程师。因为各种各样的原因，有的是因为没有HC，有的是因为岗位不合适，有的是因为女朋友所以不想离开北京，所以我纷纷拒绝了，后来还有一些机会，但是我觉得都不如继续在创宇待下去更好。于是我还是选择了继续待在创宇。

当时考虑别的工作机会其实也是因为在创宇其实没有大哥带我，相对于其他的公司，在创宇至少我有非常高的自由度和自主度，对于刚走上社会的安全从业者，没有什么比能自由研究更好的了，后来也就坦然接受了命运的安排。

现在回想起当时的事情，很大程度上做决定也依赖一些所谓的年轻人的骄傲吧。就是那种“反正已经解决不了，大不了就是重头再来呗”的感觉。当时的执拗是不愿意听家里父母的劝告去国企或者其他大厂普通的岗位。身为年轻人还是有很多不愿将就的执拗吧~

3月回到学校之后，开始了毕业前最长的一段混子时期，当时借住在朋友家，那会儿正是吃鸡火的时候，我依稀记得当时最高打上过2200分（前0.8%），那会儿每天的日子就是，下午写代码，晚上吃鸡，玩到晚上3点睡，现在想想那可真是令人羡慕的咸鱼生活。

当时是CTF盛行的时候，每个周末都有2、3个大型的CTF比赛，所以每周的固定生活就是，吃饭、写代码、吃鸡，打比赛，顺带着，还会偷偷摸摸跑到北京找女朋友的经历。比较有意思的是，当时正好是我们Vidar团队力量比较成熟的时候，我“职业”CTF生涯中比较重的几个比赛成绩都是那个时候拿到，包括强网杯，XCTF-Final，拟态，还有我那的第一个全国冠军TCTF-RSCTF的第一名。

时间兜兜转转，有点儿转瞬即逝那味儿，喝了几顿酒，也算是告别了我的大学生涯。

# 2018

2018年7月初，由于各种原因延迟毕业的我，还是提前回到了北京，开始了科学上班的“复建”生活。从之前的欧洲时差顺利的开始向上班时差更正，生活也正式从孤僻的独居生活，开始了和女朋友同居磨合的生活开始转变，就好像生活一下子就完全变了样子，现在回首看去，有些风云聚变的味道。

2018年正赶上ETH的智能合约爆火的时候，相应的智能合约安全也成了那一年最火的东西。在我回来之前，同届的其他几个同事就早早回到学校，开始了各种相关的研究，我也不例外，智能合约成了那一年最重要的工作。

由于我们在很早的时期投入研究，很长一段时间，我们都算是引领着智能合约安全的研究方向，那段时间能和我们相比的基本上只有余弦的慢雾、成都链安这种公司，后期还有一些360和长亭的小团队。其中前两者现在已经成长为区块链安全的龙头了，而我们早已放弃了这块的研究，也算物是人非吧。

那一年我至今影响比较深的是Fomo3D事件，有趣的是，前两天我还在抖音的@报告网管里看到相关的视频，我还私聊他解释了下这个事件。说真的能把智能合约游戏的优势和特性发挥的淋漓尽致的游戏还真就数Fomo3D最厉害，无论是庞氏骗局玩弄人心的手段，还是当年攻击者惊为天人的阻塞攻击，真心让人叹为观止。有兴趣的朋友可以去看看我[当年的文章](https://lorexxar.cn/2018/08/24/eth-gamedead/)。

不知道是不是我的错觉，从这件事情之后，智能合约就开始走下坡路了，毕竟从那之后，智能合约最引以为傲的去中心化和权威性受到了打破。币圈也逐渐回到了过去以经济价值为主的体系当中。很长一段时间，基于区块链的各种东西都没有成型的东西出现。

不过，对于做安全的我们来说，这些可能都显得没什么意义。在研究智能合约的各种安全问题的时候，我们也逐渐发现，solidity作为新兴语言来说，在市面上出现的时间略短，所以能明显的感觉到随便在链上找一些代码都能发现solidity函数大量复用的样例，也正是建立在这个基础上，我们最早发起了对Solidity的静态扫描，甚至用不上分析AST/CFG这类基础，仅靠正则表达式限制条件就能很好的筛选出问题，当然这甚至不能算作漏洞扫描，那个时期更多把它叫做形式化验证。

建立在静态扫描的基础上，我们当时提出了两个方案做安全，一个是另外几个同事做的基于字节码的扫描方案，但比较麻烦的是，对版本的适配度很低，后来又提出了另一种基于大数据的字节码扫描方案。大体上就是在智能合约中，同样的函数会生成同样的字节码，通过对比上万个开源的智能合约，我们可以对比出大量的字节码，这个方案非常有效，也是当时我们输出的互联网扫描方案。

后来，我又在HaoTian（集成了智能合约的扫描等各种功能）的基础上，提出了监控的概念，通过监控链上的交易，配置部分监控规则，以达到第一时间发现智能合约问题的解决方案。我花了3个月开发了这套平台，当时可以每天交易链上的各种交易，有时候能发现一些有趣的东西。可惜的是，因为各种原因，这套平台最后被搁置了，要是可以再做一做把他开源或者开放出来就好了，在那个时期，这绝对是非常好用的平台。

从技术的角度去审视的话，其实那一年真的算是有不少成长，虽然有点儿像是技能树点歪的味道，但不得不说因为那一年的经历，我对区块链行业多了很多属于我自己的见解，这也影响了我后面的很多经历。

除此之外呢，HaoTian也是我第一做大型开发项目，除了核心安全代码，还接触前后端分离、rabbitMQ、分布式架构等各种概念，这也是我的第一个万行项目，对我后来的很多开发经历有很深的影响。

在2018年的年末，正赶上智能合约弊端越来越明显，也没有特别好的项目出现。从安全的角度来说，也没有太多值得投入研究精力的事情了，于是在完成了《智能合约安全白皮书》（至今仍然感觉合约漏洞离不开这里的规范）之后，我们的工作重心也逐渐迁移回去。不再投入经历在这一块。

说起来那一年年底评定绩效的时候我还拿了唯一一次A（创宇的评定体系里一般没有A，一般比较好的是B+），不过并没有带来太多直接的改变，也算是一次不太成功的成功吧。

# 2019

时间走到2019年，这一年是比较特殊的一年，对我来说算是承上启下的一年，比起作为来说，可能转变更多一些。

首先是新年的伊始，当时还处于区块链安全研究的过度阶段，前几个月我一直在独立调试研究HaoTian-Monitor，也就是前面做的那个监控平台。可惜最后这个平台也没能公开出来，现在想来还总是觉得很可惜，但要重新捡起来做又觉得成本很高，很可惜。

时间走到中期，当时做的一个研究，Mysql任意文件读取攻击链不断有新的收获。这个漏洞最早被提出来已经是2013年了，但是一直到2018年，这个漏洞一般都被别人当作Mysql蜜罐，很少有人会去深究这个漏洞的其他利用。我最早接触也是在2018年的TCTF上，有人用这个漏洞非预期了一个题。

因为一个意外，当时我的同事@dawu用这个漏洞配合读文件挖了一个phpmyadmin的漏洞，于是当时又把这个漏洞捡起来玩，最开始还停留在找一些可能会有链接Mysql的位置，因为一次无意间的想法，我们发现Mysql任意文件读取可以触发Phar反序列化漏洞。在这个发现的基础上，我挖掘了大多数主流CMS的框架。比较可惜的是其中最好用的漏洞是Joomla的后台Getshell，而且也由于利用链太多限制条件，最终被无意中修复了。

这段经历算是我的第一个长期的安全经历了，相关的各种研究也整理成ppt在CSS2019-Tsec大会上发表了议题，后续公开了出来。

- [CSS-T | Mysql Client 任意文件读取攻击链拓展](https://lorexxar.cn/2020/01/14/css-mysql-chain/)

这也是我第一次对外的议题演讲，算是一次很不错的经历吧，有个小插曲是最开始这个议题是投稿给defcon china的，但是那年好像取消（推迟？）了，于是后来投稿了T-sec。

那一年还有一个非常有印象的漏洞是PHP-fpm远程代码执行漏洞(CVE-2019-11043)，这是我第一次遇到这种级别的Web相关通杀漏洞，如果没记错的话，应该是一个外国队伍在打Realworld CTF的时候发现的，后来公开了相关的详情和EXP。

当时为了复现这个漏洞，大概研究PHP源码3天，最后还是问了好多朋友再把它复现出来，也算是我做漏洞分析里比较典型的一个经历吧。能和这个漏洞相比的基本上只有Wordpress5.0 RCE、CVE-2019-11229 等有限几个漏洞吧，不得不说，复现一个很牛逼的漏洞的时候，其特别的成就感也不亚于挖一个相应的漏洞。

在2019年的年中，我的工作也主要集中在安全研究部分，基本上只有遇到很有趣的漏洞时，才会花费几个小时时间完成一篇漏洞分析。普通的应急工作基本上已经是顺带着做了。

在2019年的年末，因为部门里的另外一个师傅离职再加上整体的一些变动，我接手了Seebug等一些对外的工作。但我必须承认的是，作为后乌云时代第一时间接手的通用漏洞平台No.1，在时代的变迁中逐渐落幕了。

在我运营的期间，尽管我以我的维度多次更新了全新的Seebug维护规则，都改变不了安全圈越来越成熟的现状。人们越来越不需要一个能沟通通用漏洞的平台了。如果手里有好用的0day，他们会选择bugcloud、补天五星，甚至私下交易掉。如果手里本身时不好用的0day，Seebug也给不出合适的价格，还不如索性直接公开出来。

即便是后来404这边推出了女娲计划来承载这部分高端通用漏洞的收取，也无法避免，Seebug在收取漏洞这部分上，越来越成了一个可有可无的功能向。于是在2019年我接手的初期，我更愿意强化他的漏洞收录属性，这也是一直到2021年的Seebug的主要属性。（这是一个伏笔）

在2019年的年末，由于我的老大给我提议看看Chrome插件，正赶上当时有人公开了一个evernote的插件漏洞，于是我开始了新一轮的研究，陆续写了几篇相关的文章之后总算是有了一些成果，当时感觉还挺有意思的，虽然是老生常谈的攻击向量，但是真要是说相关的安全研究，反而搜来搜去只有几篇论文。当时收获还是不小的，也准备了相关的一些漏洞准备后续做一个议题。（此处疫情伏笔）

在研究Chrome插件的过程中，其实探索了不少挖掘漏洞的方案，为了能自动化挖掘chrome插件漏洞，我一时兴起，花了几个月直接给Cobra-W加了JS的扫描引擎，大体上也是基于AST的回溯分析方案，只是相比PHP来说，JS的很多语法糖给我带来了很多困扰。这也是让我第一次意识到我对静态扫描的研究多匮乏。

也正是在这次契机下，在和朋友聊天的过程，我了解到了业界主流的源伞，以及当时很火的CodeQL等静态白盒工具，也第一次了解到过程间分析、基于IR的一些分析方案。我第一次感受到我触到了前进的困局，这是我第一真正萌生离开创宇的想法。我开始感受到我开始走到圈地自蒙的境地。

# 2020

正当我还在犹豫我要不要离开创宇的时候，一场席卷全世界的大灾变出现了。我必须承认的是，即便身在中国，都很难在这样的事情中漠然。说个题外话，当时正赶上中美关系极度恶化，以xg作为战场的两方各怀鬼胎，那段时间各种可能在电视剧里都看不到的魔幻剧情层出不断。这场疫情就像一场神迹一般，打破了所有的固有局面。那段时间我总是在想，会不会在一百年后的历史书上，会把疫情定义成第二次世纪冷战的转折点。

从题外话说回来，因为疫情席卷而来，萌生离开想法的我也暂时放下心思来，困在家里的一个半月时间里，完全没有做漏洞分析、挖掘之类的心思。在一边看南京大学的软件分析网课的同时，我决定开始做之前一直想做但是没做的事情。

想做一个针对全网扫描的漏洞扫描工具，那时候正赶上Xray火的时候，小范围使用Xray也能感觉到其优势，所以最开始想的是先做一个符合我要求的爬虫（能够自动无差别扩散扫描），在基于我一些前期知识体系的基础上，开始做了一个基于Chrome headless模拟点击/事件触发的基础架构，在后来的不断完善过程中，将架构扩充成了配合rabbitMQ的分布式扫描方案。这件事情持续时间长达3、4个月，很长一段时间都处在每天查看前一天的扫描结果，然后修复bug的生活中。

在经过这样很长一段时期之后，我开始对自动化漏扫有了新的体验。尤其是当时我的同事@w7ay也就是hacking8的作者，他使用了和我相反的漏扫方案，我选择了自建的爬虫+xray，他选择了clawlergo+自建扫描器，而结果是，我的方案能扫描到更多的无用漏洞（是指没有SRC或SRC不收取的），而他的方案能扫描到更有效的漏洞。这件事情让我开始重新审视了漏洞扫描这件事情本身，也开始逐渐明白，开源项目独特的意义所在。

在无限优化LSpider这个项目的经历让我扫描器开始明白，爬虫和扫描器各司其职才应该是应有的解决方案。就好像商店买的螺丝和螺母不可能完美的重合一样，我也不应该为了能让螺母搭配螺丝，就强行改变螺母的大小。于是我开始放弃螺丝（Xray），转而慢慢去自己造一个合适的螺丝。

除了LSpider和扫描器这条支线以外，我也开始重新审视我熟悉的白盒审计工具。在写[这篇文章](https://lorexxar.cn/2020/09/21/whiteboxaudit/)的时期，我认为CodeQL是白盒发展的未来，除了QL化的基础以外，我始终认为通过预置条件来构建谓词，再构造更复杂的扫描方案是科学的发展路径。在这个基础上，我尝试性的写了一个把PHP转为Code DB的方案，并在这个DB的基础上完成了一个[扫描PHP反序列化链的工具](https://lorexxar.cn/2021/02/05/kunlun-m-phpser/)。

在写完这个工具之后我开始有一点儿触碰到我的瓶颈了，我真的发现我实际上没有特别好的办法去验证理论的可行性，如果用来对开源代码扫描，很容易陷入了针对优化却又没有通用优化的发展边际。我既没办法确定我的想法是否正确，又没办法找到我继续下去的目标。于是在这时候我萌生了第二次想要离开创宇的想法。

在这之后，我一直没有找到特别合适的前进方向，于是我已经很长一段时间没有继续为KunLun-M（Cobra-W改名）完善新的功能了。

时间走到2020年年底，那时候正赶上我为无法魔改Xray而受到困扰的时期，LSpider走到困局，又赶上我始终为KunLun-M无人问津而可惜。所以我萌生了将内部代码开源化的想法，这也是星链计划的起源。

我相信有不少朋友是因为星链计划认识我的，这个计划最早还是比较简单的，初衷不过是因为感觉很多开源工具写的太烂了，甚至那种很简单的小脚本写不好。在这个基础上，一个是希望找一个契机公开一些我们内部的小玩意，另外一个也是为了通过开源我们的代码来一些我们对开源工具的项目。希望通过我们的身体力行来证明，一个开源项目最重要的可能不是成熟好用，而是持之以恒的维护。于是[星链计划](https://github.com/knownsec/404StarLink-Project)诞生了。

在星链计划运营的过程中，我开始逐渐意识到我们的局限和不足，抛开以前就有的积累不谈，一个全新的、成熟、好用的工具诞生周期太长了，只靠404实验室的力量，又不能在短期产出工具，也没办法覆盖到好多方向。在很多时候，我都必须承认，我对渗透、甲方的理解是不足以支撑我去开发一个相关的工具的。于是一个新的想法萌生了，既然我们做不了，我们是不是可以做好平台，去推荐其他人的工具呢？

在这个基础上，我们重新提出了[星链计划2.0](https://github.com/knownsec/404StarLink2.0-Galaxy)，最初版本的星链计划2.0还是决定以收录做核心，也就是收录任何好用的工具，然后做成那种大家寻找安全类开源工具导航入口，这样的项目其实在github上还有不少。

但经过一系列探讨，最终星链计划2.0被定义成了国内安全类开源工具社区。一方面推荐优秀的开源工具给大家，另一方面呢，给所有的开源工具作者提供交流的入口和工具曝光的入口。我想至少对于我来说，有人用自己写的工具才是给自己最大的动力了。

比较欣慰的是，星链计划2.0的运营至今仍维持了初衷，至少我希望这个计划能作为安全开源工具的权威，成为最直接的安全开源工具推荐平台。希望以后星链计划也能成为大家心中的另一个乌云。也请期待以后的发展。

与此同时，在经历了1年多Seebug的运营经历之后，我逐渐开始清晰我对Seebug的未来猜想，经过一系列沟通，Seebug的改版也在推进过程了，很可惜没能见到自己的想法践行的那一刻就离开了。

# 2021

时间匆匆，一不留神就走到2021了，偷摸看了一下写到这里已经写了8800字了，从开始的强烈的倾诉欲写到这里已经没什么写下去的想法了，也许是正处于局中茫然感剧烈的原因吧，想到事情就感觉无从下笔，所以关于2021的事情就暂时略过吧。

既然新工作已经确定了，那这部分就主要聊聊找新工作的经历吧。在找工作之前，我朋友@lightless问了我一个问题一下子打到了我的命门，“你想找一份什么样的工作呢？”，对啊，我想找份什么样的工作呢。如果是3年前，我会说我想找一份大佬多，有时间做安全研究的工作。换到现在呢？

在想了很久之后，我决定把新工作的目标定为偏开发的安全，或者可以说是安全开发工程师，无论是白盒还是安全自动化。我想比起单纯做漏洞挖掘来说，能做些什么对我来说更重要。

也正是基于这个原因，我一共投了字节、长亭、知乎、小米四家，也稍微的谈谈一些面试过程中的感觉。

知乎是第一个结束的，其经历比较魔幻，一面是一个非常资深的开发，其开发能力碾压5个我可能都不过分，面试的过程中，我必须承认我的开发基础并不过关，但面试官还是会比较强势的追问我不会的方向，一面就给我留下了很不好的印象。（从这次经历我也意识到，如果面试别人的时候能感觉别人不会的话，应该及时转移，也避免面试者的逆反心理），在一面之后说实话我已经做好被刷的准备了，但是HR坚持要二面，于是二面的是一个很明显的安全，在聊天过程中非常愉快，我也及时调整了心态。但可惜的是，<del>有可能是因为知乎上市，也有可能</del>是和我对接的HR离职了，知乎的面试到这里就告一段落了。当然这里和知乎安全团队无关，仅仅是HR的工作失误。（后来问了知乎的朋友）

长亭是第二个结束的，因为咨询了朋友，所以知道在你亭安全研究岗和开发是两个事业部，为了避免转岗到安全研究那边，于是专门找了产品组的HR投递了简历。但可惜的是，在面试过程中历经3次面试，最终明确长亭更需要懂安全的开发，而不是懂开发的安全，在第三次面试过程中，达成了双向拒绝的结果。

小米是第三个结束的，顺利拿到了offer，在沟通的过程中比较愉快。说起来投递小米其实是一个意外，因为最开始投递offer懒得找朋友，于是挂在boss上看到哪家比较顺眼就投一下，其他3家都是这样投递的。而小米是在求职期间在安全客无意中看到的招聘广告，甚至还辗转了不少人才投递成功。有个比较尴尬的是，小米面试的几位面试官都看过不少我的博客，而且白盒相关的内容（关联伏笔1）也是聊天的主要内容，虽然比较顺利，但总有种才不配位的心态在，所以感触还是比较深。

字节是第四个结束的，不但是最慢，也是波折最多的。这个岗位最开始是和猎头沟通的，当时比较心仪是最开始看到的一个安全开发的JD，但字节的流程实在是太乱了，一面之后明显感觉到面试官不是做安全开发的，遂第一次转岗，结果也不知道为啥转到了安全运营，面试官是一个不太懂安全的妹子，遂第三次转岗（说实话，我也不知道转到哪了）。第三次转岗的一面面试官我印象非常差，面试过程中一直问各种基础问题（我不知道大家会不会这样，就是工作很久了，问很基础的问题完全想不起来，平时遇到都是百度。总之就是他也很尴尬我也很尴尬），从第三次转岗之后的二面才算是步入正轨吧，有趣的是，聊天的主体还是围绕白盒。

![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20210519141907.png)

甚至最终岗位也不知道怎么的变成白盒了，总是有点儿那种阴差阳错的味道。总的来说字节的面试过程非常不愉快，而且后续和HR沟通的过程也非常浪费时间，前前后后字节的面试周期都要超过2个月了，还拖累了别家的进度，感觉还是比较差吧。

不过我必须承认，我开始有点儿能感受到视频面试的优势了。18年面试的时候基本上都是电话面试，我自己面试别人的也基本上都是电话面试。但这次投的4家都要求视频面试，开始还不是很理解，因为视频面试我必须回到家里，这样一天只有比较晚的时间可以安排面试，导致我的进度很慢。但面试结束之后，我在通过电话面试别人的过程中就明显感觉到我对面试者的情绪感受很模糊，而视频面试就弥补了这个缺点，无论作为面试者还是作为面试官，都能感受到对方的情绪。这是一份比较新的体验。

不管怎么说，最终还是选择了小米的offer，和我熟悉的朋友肯定知道我是老米卫兵了，虽然找工作的过程更多是源于一个以外，不过谁知道呢，妄言揣测未来只不过是管中窥豹而已，希望我的下一个阶段也能顺顺利利的吧~

# 写在最后

其实写到这里的时候，我已经是正式离职的第二天了，这篇文章断断续续的已经写了3天多了，也确确实实的写了万字长文。如果有朋友能一点一点的看完整篇文章，那真是非常感谢有这样的耐心看我写了万字的流水账，我是那种很有写作欲，但却没什么写作天赋的人，我本身就喜欢写博客，维护了6年还多的blog写了整整180篇文章，除了blog以外，我还在知乎写游戏评测，在B站做游戏视频。虽然总的来说看的人并不多，但对我来说，有人看过便对我是莫大的鼓励了。

有时候不得不承认，我是那种很没有天赋，全凭一身普信来支撑的人。从小在内蒙古长大，在当地还算凑合的高中也都只能排到30%，凭借和一本线擦边50分的极限发挥，总算是逃离家乡，完成了少年时期外出闯荡远游的梦想。

在杭电大一还算是认真学习的我，保持40%的成绩都很吃力。随后算是彻底在学习方面总算是认清了自己。凭借着那个年龄少有的认清自己的心态，把所有的精力都花费在玩游戏和当“黑客”上，有时候仍怀念那个时候的纯粹，换到现在，我都不能保证我每天花10个小时写代码/复现题目。犹记得那个时期最大的收获，就是终于明白什么是“学的越多不会的越多”这句话，贪婪成长带来的快感远比想象的要快乐，第一次做出题目，第一次写脚本，第一次拿到一血，第一次挣到钱，第一次拿到全国冠军，第一次靠自己带飞全场。必须承认，如果没有这些第一次，仅靠普通天赋的我，万万没有可能走到今天。

在写这几年的的回顾的过程中，也有时候不免感概，那个时候的我怎么那么菜啊？只是每每有这样的想法，便会马上意识到，身处困局当中的我，又怎么能看得清自己呢？就如现在的我一般，总归是要靠着那份当年的年轻人的骄傲吧，最坏又能坏到哪里去呢？对吧。

祝每个看完我文章的朋友，都能找到真正的自己，工作顺利~