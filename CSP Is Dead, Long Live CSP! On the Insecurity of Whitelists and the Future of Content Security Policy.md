---
title: CSP Is Dead, Long Live CSP!-翻译
date: 2016-10-31 19:19:57
tags:
- Blogs
- csp
categories:
- Blogs
---



CSP Is Dead, Long Live CSP! On the Insecurity of Whitelists and the Future of Content Security Policy



无意间阅读到了去年google团队写的关于CSP现状和展望的论文，深有感触，所以翻译了大部分的文章内容，分享给大家。

[https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/45542.pdf](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/45542.pdf)

<!--more-->

# 摘要

Content Security Policy是一种web平台的安全机制，是为了减少xss漏洞而出现，这应该是现代web应用中顶级的安全漏洞。在本文中，我们仔细观察和考虑CSP的实际好处，也确定下现实世界中出现的可以导致94.72%的所有不同策略重大缺陷。

我们对互联网超过10亿个主机名大约1000亿的搜索引擎库，1680867台主机的CSP部署方式，26011种独特的CSP策略做了研究，这里我们介绍CSP规范的安全相关，并提供对其威胁模型的深入分析，重点关注对于XSS的保护，这里我们确定三种常见的CSP绕过方式，并解释它们是如何颠覆策略的安全性的。

之后呢，我们转而对部署在互联网上的策略分析，来了解他们的安全程度，我们发现，加载脚本最常列入白名单的有15个域，其中有14个不安全的站点，因此有75.81%的策略因为使用了了脚本白名单，允许了攻击者绕过了CSP。总而言之，我们发现尝试限制脚本执行的策略中有94.68%是无效的，并且99.34%具有CSP的主机制定的CSP策略对xss防御没有任何帮助。

最后，我们提出了“strict-dynamic”的关键字，这是规范的一个附加，它有助于基于加密随机数创建策略，而不依赖域白名单。我们讨论我们在复杂应用成语中部署一个这样基于nonce策略的经验，来为程序要提供指导来改进策略。

# 1、介绍 #

XSS-跨站脚本攻击，将攻击者控制的脚本注入到Web代码的上下文来控制脚本，可以说是最臭名昭著的Web漏洞之一。自从2000年第一正式提出了XSS以来，经过很多代人的研究、预防、环节，XSS仍然是网络最流行的安全问题之一，随着网络的发展，新的变化也被不断的被发现。

至今为止，CSP的仍然是最有希望的针对XSS对策。CSP是一个声明性的策略机制，允许Web开发者定义哪些客户端资源可以由浏览器加载和执行。通过禁止内敛脚本并仅允许可信域作为外部脚本的源，通过这种方式限制站点执行恶意客户端的代码。因此，即使攻击这发现了xss漏洞，CSP也可以保护应用程序的安全，攻击者不能在不控制可信主机的情况下加载恶意代码。

在这篇文章中，我们介绍第一次深入分析CSP在整个网络部署的结果，为了研究这一点，我们首先需要调查CSP的保护能力，通过分析威胁模型，分析可能的漏洞和可以使攻击这绕过保护的攻击方式。

我们遵循从google搜索引擎提取的真实世界CSP策略的大规模研究。我们发现目前至少有168000个网络主机部署了CSP策略。在对数据集进行归一化和重复数据处理后，我们确定了26011个唯一的CSP策略，其中94.72%都是可以被忽略的。攻击这可以使用各种方式破坏CSP保护的站点，尽管部署CSP方面花费了相当大的努力。现在的策略中90.63%可以包含通过允许执行内敛脚本或从任意外部主机加载脚本。而只有9.37%的策略具有更严格的配置，甚至防范着潜在的XSS。但是，我们发现，只有51%的这类策略仍然可以绕过，因为在script-src中存在微妙的策略配置错误或不安全的站点。

基于我们的研究结果，我们认为，为复杂的web端应用维护安全白名单在实践中是不可行的；因此，我们建议改变CSP的使用方式。我们建议通过指定脚本可以执行的url白名单来指定信任的应该被替换为基于nonce和散列值的方法，这已经被CSP规范定义并且在主流的浏览器实现中可用。

在基于nonce的策略中，web应用程序定义了在CSP中传递单一、不可猜测的token（nonce），以及一些合理的html属性，而不是把主机和域白名单用于脚本执行。用户代理仅允许执行那些nonce与策略中指定值匹配的脚本。可以将html注入到页面内的攻击者不知道临时值，因此无法执行恶意脚本。为了简化这种基于nonce的方式，我们提出了一个新的CSP源表达式关于script，我们称它为strict-dynamic。使用这个，动态生成的脚本隐式地从穿件他们的可信脚本集成值。这样，已经执行的合法脚本可以轻松地将新脚本添加到DOM，而不需要大量的代码修改。但是如果攻击者找到了xss漏洞，但是不知道正确的nonce，他们的脚本将会被阻止。

为了证明这种方法的可行性，我们提出了一个真实世界的案例研究。在一个比较主流的cms中使用基于nonce的策略。

我们的总结如下：

- 我们提出了CSP安全模型的第一深入分析结果，分析对标准中对于web漏洞的保护。我们识别常见的策略配置错误，并提供3类CSP的绕过方式，可以禁用策略的保护能力

- 我们通过google搜索引擎中提取策略，对现实世界的CSP部署的好处进行了大规模的研究。基于一个大约1060亿页的资料库，其中39亿是收到了CSP保护的，我们分析除了26011个独特的策略。我们发现，由于策略配置的错误和白名单的不安全，这些策略至少有94.71%对xss缓解没有帮助。

- 基于我们的研究结果，我们建议改变CSP在实际中的部署方式，不适用白名单，我们主张基于nonce的方式。为了进一步实现这种方法，我们提出了strict-dynamic，这是当前在chrome浏览器中实现的CSP3特性之一。我们讨论这种方法的好处，并提出了在主流cms中部署基于nonce和strict-dynamic策略的案例研究


在本文中，第二节中我们深入分析CSP，包括2.1中的技术基础，2.2和2.3讲了CSP设计策略时的常见漏洞。随后第3节中我们介绍了我们实际研究的结果，为了能够描述清楚问题，3.1中首先概述问题，3.2介绍我们的数据集，3.3中解释我们的研究方法，最后在3.4中给出结果和分析。基于这次研究，我们提出了改进CSP的方法，最后我们在第5节中介绍相关的东西，在第6节总结。

# 2.CSP #

## 2.1 概述 ##

CSP是一种声明机制的策略，它允许Web作者在站内指定许多安全限制，由浏览器执行。

CSP使开发人员在开发时可以以各种方式锁定他的站，减少XSS的风险和危害，降低站内各类脚本的执行权限。

CSP从建立开始一直快速发展，现在正在进行规范的是CSP3，但该标准在不同浏览器都是不同程度的试行，比如Chromium中具有完整的CSP2支持，并实现了CSP3中的大部分草案，但仍然落后于实验的标准，而在Firefox和基于Webkit的浏览器刚刚完成了完整的CSP2支持。这里我们不关注关于各个标准之间的修订，只是提供一个关于版本的概述。

CSP策略一般通过HTTP响应头中或在`<meta>`元素中传递。CSP的功能可以分为3类。

**限制资源加载**

CSP的最广为人知和常用的的方面是限制浏览器只加载开发人员允许的子资源，称之为源列表。常用的执行包括script-src, style-src, catcall, default-src, 作为特殊情况，script-src和style-src指令还有额外的配置选项，这些允许script和css有更精细的控制方式。


**基于url的控制**

某些类型的攻击不能通过仅管理子资源来防止，但仍然需要一个可以与之交互的可信来源。一个比较常见的例子是frame-ancestors指令，它定义了框架资源的起源，以防止点击劫持。类似的，base-url和form-action定义了`<base#href>`和`<form#action>`元素的目标，以防一部分xss攻击


**其他限制**

这里包括blocks-all-mixed-content和upgrade-insecurerequests关键字，可以防止混合内容错误和改进了对于https的支持，plugin-types限制了允许的插件格式，sandbox反应了一些HTML5沙箱框架的一些安全功能


为了能让整个Web站和CSP兼容，对XSS的防御有帮助，Web的开发者通常需要重构正常逻辑生成的html标记，框架以及模板化整个站。特别是涉及到内联脚本、eval和等效的结构、内联时间处理和javascript: URL的使用必须避免和重构为对CSP友好的结构。


除了默认的强制执行策略限制方式之外，还可以在Reprot-Only模式配置CSP，其中记录不合理的配置，但不会强制执行。在这两种情况下，report-uri指令都可以用作发送不合理配置报告，已通知站开发者站不合理的标记。

### 2.1.1 源列表 ###

CSP源列表（也可以叫做白名单）是CSP的核心部分，并且是指定信任关系的和新方式。例如：

```
Content-Security-Policy: script-src ’self’; style-src
cdn.example.org third-party.org; child-src https:
```

应用程序仅允许信任的托管域可以加载脚本，也允许来自cdn.example.org\third-party.org的字体或者图像，并且需要通过https加载，同时有不对其他资源施加任何限制。

对于任何指令，白名单可以由主机名（example.org, example.com）组成，可能包括\*通配符以拓展对所有子域（\*.example.com）的信任、schema(https:/http:)、和特殊的关键字'self'（表示当前文档的来源）还有'none'（强制执行空源，也就是说精致加载任何资源）

从CSP2开始，开发者还可以选择在白名单中制定路径（example.org/resources/js/）.有趣的是，不能依赖基于路径的限制来限制可以加载资源的位置，这个问题稍后在讨论。


### 2.1.2脚本执行的限制 ###

由于脚本在现在Web应用程序中的重要性，script-src提供了几个关键字，以允许对脚本的执行提供更精细的控制：

1、unsafe-inline 

允许执行内联的`<script>`块和javascript时间处理程序。

2、unsafe-eval

允许使用javascript api来执行字符串作为代码，包括eval、settimeout、setinterval和function。否则，这些api会被script-src指令策略阻止

3、CSP随机数

允许策略指定允许的脚本授权令牌(`script-src 'nonce-random-value'`)，将允许页面上具有正确token属性的任何脚本执行。

4、CSP散列值

允许开发人员列出页面中预期的脚本加密散列（`scriptsrc'sha256-nGA ...`），将允许任何内容的摘要和策略中提供相匹配的脚本执行。

nonce和hash同样可以在style-src中使用，来允许通过随机数加载CSS。


## 2.2 CSP的威胁模型 ##

CSP提供的的优势在于，即使这些漏洞会对访问站的用户产生恶意行为，CSP也会拦截。在目前的形式中，CSP提供了3种类型的保护：

- XSS: 在站内注入和执行不受信任的脚本的能力（受到script-src和object-src指令保护）

- Clickjacking(点击劫持): 欺骗用户点击受攻击这控制的页面上覆盖的隐藏内容（受到frame-ancestors）

- mixed content: 通过https传递的不安全协议加载资源（受到upgrade-insecure-requests和blockall-mixed-content保护，通过限制脚本和加载敏感资源）

因此，只有一小部分CSP指令对XSS保护游泳。此外，在站内上下文中执行恶意脚本的能力破坏了其他指令提供的保护，后面2.2.2会提到这个。


### 2.2.1 采用CSP的好处 ###

由于一些流行的用户代理还不支持CSP或者只提供部分支持，所以CSP应该只被应用于深度防御，用于主安全机制失败后的攻击尝试。因此，使用CSP的站仍然需要采用传统的保护机制。例如，采用具有严格上下文转移的框架来生成标签，使用`x-frame-options`的头来防止点击劫持，并确保页面上的资源通过https获取。

设置CSP的实际好处就在于此，在主安全机制失效的情况下，CSP可以帮助保护用户，当开发人员出现了编程错误，CSP可以有效的减少XSS、点击劫持、混合内容的攻击危害。

然而，在实际生活中，`x-frame-options`的点击劫持很少有办法破坏，而且在现代的用户代理中，默认情况下已经组织了混合内容的攻击（通过http从https网页加载脚本和内容），因此，CSP的主要价值其实就是在于防止XSS的利用，因为他是唯一一类可以通过CSP减轻的漏洞，而且通常被开发人员无意中完成。

### 2.2.2 防御XSS ###

CSP的安全优势绝大多数集中在两个防止脚本执行的指令中: script-src和object-src（flash可以在嵌入式页面中执行js）或者default-src的缺漏。

能够注入和执行脚本的攻击者一般能绕过所有其他指令的限制。因此，使用不够安全的script-src和object-src源列表的策略的站从CSP中获取的利益非常有限。比起提供有意义的安全优势，站点必须首先使用能成功防止脚本执行的安全策略。一般来说，攻击者可以作post-xss或者无脚本攻击（类似于csrf或者是网络钓鱼），所以，CSP只在对XSS保护有效的时候才能提高安全性。（也就是说并不是对所有的xss攻击都有效）

为了组织不必要的脚本执行，策略必须满足三个要求:

- 策略必须同时定义script-src和object-src（或default-src）

```
bypass CSP 当script-src和object-src未定义时

<script src="//evil.com"></script>

<object data="//evil.com/evil.swf">
<param name="allowscriptaccess" value="always">
</object>
```

- script-src源列表不应该包含`unsafe-inline`(除非伴随有nonce)或者允许`data: URLs`

```
当定义了unsafe-inline或者data: URLs

<img src="x" onerror="evil()">

<script src="data:text/javascript,evil()"></script>
```

- scrip-src和object-src源列表不能包含攻击可以控制响应的不安全的站点

```
当攻击者可以控制返回的bypass方式

<script src="/api/jsonp?callback=evil"></script>

<script src="angular.js"></script> <div ng-app>
{{ executeEvilCodeInUnsafeSandbox() }} </div>

```

如果不满足上面的任何一个条件，则策略在防止脚本执行上都不是有效的，不能对xss有防护作用。

下面我们专注于研究对于端点类型的分析，也就是将CDN等加入了白名单导致攻击者可以绕过CSP保护执行脚本的方式。

## 绕过CSP执行脚本 ##

在使用CSP的基本安全原则之一就是，在策略中列入白名单的域仅提供安全内容。因此，攻击者不应该能在这些白名单来源的响应中注入有效的js代码。在下面的小节中，我们讨论现代web应用违反这个原则的几个例子。

### 2.3.1 带有用户控制回调的js ###

尽管许多的js资源都是静态的，但是在某些情况下，开发人员可能希望通过允许请求参数来动态生成脚本，用来设置加载脚本时必要的函数。例如：在回调函数中包装js对象的jsonp接口通常允许通过第三方域的脚本作为源数据来加载api数据。

```
<script
src="/path/jsonp?callback=alert(document.domain)//">
</script>

/* API response */
alert(document.domain);//{"var": "data", ...});

```

如同范例上一样，如果策略中列入白名单的域包含jsonp接口，攻击者可以通过控制回调函数的加载作为`<script>`来执行任何js函数。如果攻击者可以控制jsonp响应，那么就可以无限制的执行脚本，它们可以使用类似[some](http://www.benhayak.com/2015/06/same-origin-method-execution-some.html)的技术，这些技术几乎等于没有约束的xss。


### 2.3.2 Reflection or symbolic execution ###

CSP对脚本执行的限制有可能会被白名单中的协作脚本绕过。例如，脚本可以使用反射来查找和调用全局域下的函数。

```
// Can be used to invoke window.* functions with
// arbitrary arguments via markup such as:
// <input id="cmd" value="alert,safe string">

var array = document.getElementById(’cmd’).value.split(’,’);
window[array[0]].apply(this, array.slice(1));

```

这种js工具通常不会危害安全，因为参数受到网页加载脚本开发者的控制。但当这样的脚本通过检查dom获取数据时出现问题，如果站内有可控的标签属性，攻击者可以通过绕过CSP来执行任意函数。

有一个比较有名的例子就是引入angularJS库，它允许强大的模板语法来创建web页面。

```
<script src="whitelisted.com/angular.js"></script>

<div ng-app>{{ 1000 - 1 }}</div>
```

angularJS在页面指定部分解析模板并执行他们。如果我们可以控制能被angularJS解析执行就可以认为，我们可以执行任意js。默认情况下，angular使用eval函数来评估沙箱的执行，这是不会被unsafe-eval拦截的。然而angularJS还提供了CSP兼容模式（ng-csp），其中通过符号执行来执行代码，使得在CSP模式下仍然可以调用任意js代码。

因此，可以从CSP白名单源列表中加载angular库，以绕过CSP。即便被攻击的站本身不使用angular，但只要白名单域内存在angular库的托管，因此，可信域中只要存在任何angular库就能破坏CSP提供的保护。

### 2.3.3 Unexpected JavaScript-parseable responses ###

由于兼容性的原因，Web浏览器通常很容易检查响应的MIME类型是否与使用响应的页面上下文匹配。任何可以解析为js而没有出现语法错误，并且在第一个运行时错误出现的攻击者控制的数据响应可以导致脚本执行。因此，CSP可以被下面的方式绕过：

- 部分可以被攻击者控制的内容的逗号分隔（csv）数据：
```
Name,Value
alert(1),234
```

- 回显请求参数的错误消息
```
Error: alert(1)// not found.
```

因此，如果白名单的源列表中具有此类属性的任何页面，攻击者可以伪造脚本响应并执行js。类似的问题也适用于object-src白名单：如果攻击者可以将一个被解释为flash对象的资源上传到针对object-src的白名单域中，则可以执行脚本。

重要的是要注意，上述的几种情况都不会造成直接的安全风险，因此开发人员通常不会直接改变他们。然而，当站内采用csp时，这样的端点变为安全问题，因为他们可以绕过CSP。

更大的问题是，这样的问题不仅仅存在于部分白名单来源，还影响了将其列入script-src白名单中的所有其他域。这些域通常包含不太了解CSP的第三方或者CDN，他们没理由修复这样的CSP绕过问题。

### 2.3.4 Path restrictions as a security mechanism ###

为了解决CSP对于域限制不足的问题，CSP2引入了可以将白名单限制到域的特定路径的能力（类似于example.org/foo/bar），开发者可以选择在可信域上指定特定的目录来加载脚本和其他资源。

不幸的是，由于处理跨源重定向的问题，这种限制被放宽了。如果白名单源列表目录中包含重定向器，则这个重定向器可以在不被允许的目录加载资源。

```
Content-Security-Policy: script-src example.org
partially-trusted.org/foo/bar.js
// Allows loading of untrusted resources via:

<script src="//example.org?
redirect=partially-trusted.org/evil/script.js">
```

由于这种行为和复杂Web应用的重定向很流行，所以限制路径作为CSP中的安全机制是不安全的。

上面我们展示了很多看上去很安全的编程方式是如何允许攻击者绕过CSP来xss的了，反过来，我们转向分析这种绕过对于现实世界的影响。


# CSP的实证研究 #

由于原文属于论文性质的文章，为了使翻译不过于拖沓，所以略过前面关于准备工作的描述，直接跳到关键部分。

### 评估策略的安全性 ###

为了评估是否可以绕过CSP以执行攻击者控制的脚本，我们需要进行以下检查：

1、'unsafe-inline'的使用，使用unsafe-inline关键字的策略如果不指定脚本随机数，那本质上是不安全的，这样的策略会被标记为可绕过。

2、缺少object-src: 指定了script-src但缺少object-src(并且不设置default-src)的策略允许通过插入插件资源来执行脚本。

3、在白名单中包含通配符： 如果CSP的白名单列表中包含通配符或url schema2,允许包含任意主机的内容，则策略仍然是不安全的。

4、白名单中的不安全策略：当把可以绕过CSP的端点域列入白名单时，CSP的保护能力将变为无效，如2、3节所述，为了评估具有这种特性的列表。如果给定策略的白名单中出现在列表中，我们将认为这个策略是可以被bypass的。

接下来的部分，原文对会造成csp bypass的域做了基本分析，可以发现会造成这样情况的域中基本有包含jsonp和angularjs两种，而在jsonp下，即便对接口做了字符限制，仍然可以使用same手段执行任意js函数。所以可以认为包含jsonp接口的源就是不安全的。而angularjs更多问题出在低版本上，所以危害性又有不同。

经过数据量庞大的分析，有了下面的结果

![image_1bbt7blfa15tg1l1p1ai51oigppa9.png-85.9kB][1]

后面篇幅着重介绍了基于nonce的方法和“strict-dynamic”关键字这两个新提出来的csp实现方式，但这种实现方式和现在web端js实现方式本身有冲突，所以其作用还有待考证。


  [1]: http://static.zybuluo.com/LoRexxar/a56u1xigxrgg1wyph9h4ng15/image_1bbt7blfa15tg1l1p1ai51oigppa9.png