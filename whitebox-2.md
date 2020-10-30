---
title: 构造一个CodeDB来探索全新的白盒静态扫描方案
date: 2020-10-30 11:33:22
tags:
- 白盒
- 静态分析
---

---

前段时间开源新版本KunLun-M的时候，写了一篇《从0开始聊聊自动化静态代码审计工具》的文章，里面分享了许多在这些年白盒静态扫描演变过程中出现的扫描思路、技术等等。在文章中我用了一个简单的例子描述了一下基于.QL的扫描思路，但实际在这个领域我可能只见过一个活的SemmleQL（也就是CodeQL的原型）。这篇文章中我也聊一聊这相关的东西，也分享一些我尝试探索的一些全新的静态扫描方案。

本文提到的小demo phpunserializechain作为星链计划的一员开源，希望能给相关的安全从业者带来帮助。

本文会提及大量的名词，其中如有解释错误或使用不当欢迎指正。
<!--more-->

# 什么是.QL?

QL全称Query Language，是一种用于从数据库查询数据的语言。我们常见的SQL就是QL的一种，这是一个很常见的概念。

而.QL是什么呢？Wiki上的解释是，一种面向对象的查询语言，用于从关系数据库中检索数据。

而.QL又和静态分析有什么关系呢？我们需要理解一个概念叫做SCID.

SCID: Source Code in Database 是指一种将代码语法解析并储存进代码中的操作方法.而这种数据库我们可以简单的称之为Code DB.

当我们通过一种方案生成了Code DB之后，我们就需要构造一种QL语言来处理它。当然Code QL正是一种实现了Code DB并设计好了相应的QL语言的平台。而Semmle QL设计的查询语言就是一种.QL，它同时符合了几种特点其中包括SQL、Datalog、Eindhoven Quantifier Notation、Classes are Predicates其中涵盖了针对代码的不同逻辑而使用的多种解决方案。当然，本文并不是要讨论CodeQL的原理，所以这里我们并不深入解释Semmle QL中的解决方案。

.QL的概念最早在2007年被提出，详情可以参考：

- [https://help.semmle.com/home/Resources/pdfs/scam07.pdf](https://help.semmle.com/home/Resources/pdfs/scam07.pdf)

# 为什么使用.QL呢？

在《从0开始聊聊自动化静态代码审计工具》中我曾经把基于.QL的认为是未来白盒发展的主要趋势，其主要原因在于现代普遍使用的白盒核心技术存在许多的无解问题，在上一篇文章中，我主要用一些基于技术原理的角度解释了几种现代的扫描方案，今天我就从技术本身聊聊这其中的区别。

其实我在前文中提到的两种分析方式，无论是基于AST的分析、还是基于IR/CFG的分析方式，他们的区别只是技术基础不同，但分析的理论差异不大，我们可以粗略的将它们统一叫做Data-flow analysis，也就是数据流分析（污点分析可以算作是数据流分析的变种）。

数据流分析有很多种种类，其本质是流敏感的，但是通常来说是路径不敏感的，当然，我们可以按照敏感类型将其分类：

- 流敏感分析：flow-sensitive，考虑语句的执行先后顺序，这种分析通常依赖CFG控制流图
- 路径敏感分析：path-sensitive，不仅考虑语句的执行顺序，还要分析路径的执行条件（比如if条件等），以确定是否存在可实际运行的执行路径。
- 上下文敏感分析：context-sensitive，属于一种过程间分析，在分析函数调用目标时会考虑调用上下文。主要面向的的场景为同一个函数/方法在不同次调用/不同位置调用时上下文不同的情况。

当然，需要注意的是，这里仅指的是数据流分析的分类方式，与基于的技术原理无关，如果你愿意，你当然也可以基于AST来完成流敏感的分析工具。

在基于数据流的扫描方案中，如果能够完整的支持各种语法充足的分析逻辑，我们就可以针对每一种漏洞寻找相应的数据流挖掘漏洞。可惜事实是，问题比想象的还要多。这里我举几个可能被解决、也可能被暂时解决、也可能没人能解决的问题作为例子。

1、如何判断全局过滤方案？
2、如何处理专用的过滤函数未完全过滤的情况？
3、如何审计深度重构的框架？
4、如何扫描储存型xss？
5、如何扫描二次注入？
6、如何扫描eval中出现的伪代码逻辑？

当在现代扫描方案不断进步的同时，或许许多问题都得到一定程度的解决，但可惜的是，这就像是扫描方案与开发人员的博弈一样，我们永远致力于降低误报率、漏报率却不能真正的解决，这样一来好像问题就变得又无解了起来...

当然，.QL的概念的扫描方案并不是为了解决这些问题而诞生的，可幸运的是， 从我的视角来看，基于.QL概念的扫描方案将静态扫描走到了新的路中，让我们不再拘泥于探讨如何处理流敏感、等等。上次我简单解释了基于.QL扫描方式的原理。

其核心的原理就在于通过把每一个操作具象化模板化，并储存到数据库中。比如

```
a($b);
```

这个语句被具象为
```
Function-a  FunctionCall ($b)
```

然后这样的三元组我们可以作为数据库中的一条数据。

而当我们想要在代码中寻找执行a函数的语句时，我们就可以直接通过
```
select * from code_db from where type = 'FunctionCall' and node_name = 'Function-a';
```

这样的一条语句可以寻找到代码中所有的执行a函数的节点。

当然，静态分析不可能仅靠这样的简单语句就找到漏洞，但事实就是，当我们针对Code DB做分析的时候，我们既保证了强代码执行顺序，又可以跨越多重壁垒直接从sink点出发做分析，当相应的QL支持越来越多的高级查询又或者是自定义高级规则之后，或许可以直接实现。
```
select * where {
    Source : $_GET,
    Sink : echo,
    is_filterxss : False,
}
```

也正是因为如此，CodeQL的出现，被许多人认为是跨时代的出现，静态分析从底层的代码分析，需要深入到编译过程中的方式，变成了在平台上巧妙构思的规则语句，或许从现在来说，CodeQL这种先铺好底层的方式并不能直接的看到效果，可幸运的是，作为技术本身而言，我们又有了新的前进方向。

下面的文章，我们就跟着我前段时间的一些短期研究成果，探索一下到底如何实现一个合理的CodeDB呢

# 如何实现一个合理的CodeDB呢?

在最早只有Semmle QL的时候我就翻看过一些paper，到后来的LGTM，再到后来的CodeQL我都有一些了解，后来CodeQL出来的时候，翻看过一些人写的规则都距离CodeQL想要达到的目标相去甚远，之后就一直想要自己试着写一个类似的玩具试试看。这次在更新KunLun-M的过程中我又多次受制于基于AST的数据流分析的种种困难，于是有了这次的计划诞生。

为了践行我的想法，这次我花了几个星期的事件设计了一个简易版本的Code DB，并基于Code DB写了一个简单的寻找php反序列化链的工具，工具源码详见:

- [https://github.com/LoRexxar/Kunlun-M/tree/master/core/plugins/phpunserializechain](https://github.com/LoRexxar/Kunlun-M/tree/master/core/plugins/phpunserializechain)

在聊具体的实现方案之前，我们需要想明白CodeDB到底需要记录什么？

首先，每一行代码的执行顺序、所在文件是基本信息。其次当前代码所在的域环境、代码类型、代码相关的信息也是必要的条件。

在这个基础上，我尝试使用域定位、执行顺序、源节点、节点类型、节点信息这5个维度作为五元组储存数据。举一个简单的例子：

```
test.php

<?php
$a = $_GET['a'];

if (1>0){
    echo $b;
}
    
```

上面的代码转化的结果为

```
test_php 1 Variable-$a Assignment ArrayOffset-$_GET@a
test_php 2 if If ['1', '>', '0']
test_php.if 0 1 BinaryOp-> 0
test_php.if 1 echo FunctionCall ('$a',)
```

由于这里我主要是一个尝试，所以我直接依赖SQL来做查询并将分析逻辑直接从代码实现，这里我们直接用sql语句做查询。

```
select * from code_db where node_type='FunctionCall' and node_name='echo'
```

用上述语句查询出echo语句，然后分析节点信息得到参数为`$a`。

然后通过
```
select * from code_db where node_locate = 'test_php.if' and node_sort=0 
```
来获取if的条件信息，并判断为真。

紧接着我们可以通过SQL语句为

```
select * from code_db where node_name='$a' and node_type='Assignment' and node_locate like 'test_php%' and node_id >= 4 
```

得到赋值语句，经过判断就可以得到变量来源于`$_GET`。

当然，逻辑处理远比想像的要复杂，这里我们举了一个简单的例子做实例，通过sort为0记录参数信息和条件信息，如果出现同一个语句中的多条指令，可能会出现sort相同的多个节点，还需要sort和id共同处理...

这里我尝试性的构造了基于五元组的CodeDB生成方案，并通过一些SQL语句配合代码逻辑分析，我们得到了想要扫描结果。事实上，虽然这种基于五元组的CodeDB仍不成熟，但我们的确通过这种方式构造了一种全新的扫描思路，如果CodeDB构造成熟，然后封装一些基础的查询逻辑，我们就可以大幅度解决我在KunLun-M中遇到的许多困境。

# 写在最后

这篇文章用了大量的篇幅解释了什么是基于.QL的扫描方案，聊了聊许多现代代码审计遇到的问题、困境。在这个基础上，我也做了一些尝试，这里讲的这种基于五元组的Code DB生成方案属于我最近探索的比较有趣的生成方案，在这个基础上，我也探索了一个简单的查询php反序列化的小插件，后续可能花费比较大的代价去做优化并定制一些基础的查询函数，希望这篇文章能给阅读的你带来一些收获。

如果对相应的代码感兴趣，可以持续关注KunLun-M的更新

- [https://github.com/LoRexxar/Kunlun-M](https://github.com/LoRexxar/Kunlun-M)

  

# ref

- [https://help.semmle.com/publications.html](https://help.semmle.com/publications.html)
- [https://help.semmle.com/home/Resources/pdfs/scam07.pdf](https://help.semmle.com/home/Resources/pdfs/scam07.pdf)
- [https://en.wikipedia.org/wiki/Data-flow_analysis](https://en.wikipedia.org/wiki/Data-flow_analysis)
- [https://en.wikipedia.org/wiki/Control-flow_graph](https://en.wikipedia.org/wiki/Control-flow_graph)
- [https://en.wikipedia.org/wiki/Source_Code_in_Database](https://en.wikipedia.org/wiki/Source_Code_in_Database)
- [https://en.wikipedia.org/wiki/.QL](https://en.wikipedia.org/wiki/.QL)
- [https://firmianay.gitbooks.io/ctf-all-in-one/content/doc/5.4_dataflow_analysis.html](https://firmianay.gitbooks.io/ctf-all-in-one/content/doc/5.4_dataflow_analysis.html)
- [https://blog.csdn.net/nklofy/article/details/83963125](https://blog.csdn.net/nklofy/article/details/83963125)
- [https://blog.csdn.net/nklofy/article/details/84206428](https://blog.csdn.net/nklofy/article/details/84206428)
- [https://lorexxar.cn/2020/09/21/whiteboxaudit/](https://lorexxar.cn/2020/09/21/whiteboxaudit/)

