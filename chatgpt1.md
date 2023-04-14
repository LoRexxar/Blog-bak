---
title: 从0到1的ChatGPT - 入门篇
date: 2023-04-14 18:40:31
tags:
- chatgpt
---

在2023年年初，ChatGPT像一颗流星一样突然出现在大家的面前，围绕ChatGPT的探索也以各种各样的方式出现在大家的面前。

相比基于ChatGPT的探索，openai的平台和国内的对抗反倒在潜移默化的升级，我没有了解过openai到底有什么样的背景导致一直执着于国内使用者的封禁，这篇文章就先讲讲我在这个过程的所有探索以及相应的解决方案吧。

# IP限制

这个东西是在使用ChatGPT过程中遇到的最大的问题，而且其中的相应策略极其复杂，这里只列举我撞到的策略和绕过方案。

## 通过代理解决

首先来源IP这块就不用多说了，你正常直接去访问openai.com都会撞到GFW的拦截，当然作为技术从业者有自己的科学手段自然不用多说了，但如果说GFW是你入门的门槛的话，那chatgpt使用的方案可以说是釜底抽薪。国内百分之90的科学手段大抵上都是利用国外的服务器来实现的，而且除开使用机场的朋友以外，大部分都是使用比较有名的各种云服务器，chatgpt搞得第一个门槛就是，**封禁了大部分的云服务器ip以及网段**。

- vultr，godaddy
- aws，oricle，linode
- 阿里云、腾讯云

这就直接导致了一个问题，就是在ip层面你就需要想办法绕过。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304141841776.png)

根据我的了解，其实大部分人都使用了比较冷门的云服务商或者比较冷门的机场来解决，这样比较一劳永逸，而我选择了用另一个方案就是v2ray+cf wrap，这里我就不详细解释具体是怎么回事了，大概是用了CF推出的一个相对比较真实的ip来做代理，很多朋友会用这个方案来绕过Netflix的限制。具体可以参考这个链接来实现。

https://github.com/willoong9559/XrayWarp

### 配置warp

1、参考CF的文档来安装warp

https://developers.cloudflare.com/warp-client/get-started/linux/

2、注册客户端

```plain
warp-cli register
```

3、设置WARP代理模式

```plain
warp-cli set-mode proxy
```

4、连接WARP

```plain
warp-cli connect
```

此时WARP会使用socks5本机代理127.0.0.1：40000

 5、打开warp always-on

```plain
warp-cli enable-always-on
```

6、测试socks代理，理检查ip是否改变

```plain
export ALL_PROXY=socks5://127.0.0.1:40000
curl ifconfig.me
```

7、修改V2ray的配置

- 为inbounds启动sniffing

```plain
"sniffing": {
    "enabled": true,
    "destOverride": ["http", "tls"]
}
```

- 为outbounds中加入socks_out相关的配置

```plain
"outbounds": [
        {
            "tag": "default",
            "protocol": "freedom"
        },
        {
            "tag":"socks_out",
            "protocol": "socks",
            "settings": {
                "servers": [
                     {
                        "address": "127.0.0.1",
                        "port": 40000
                    }
                ]
            }
        }
    ],
    "routing": {
        "rules": [
            {
                "type": "field",
                "outboundTag": "socks_out",
                "domain": ["geosite:netflix"]
            },
            {
                "type": "field",
                "outboundTag": "default",
                "network": "udp,tcp"
            }
        ]
    }
```

- 下面的routing配置当中，我们可以把需要过warp的域名配置进去，**因为warp相对慢很多，所以其他域最好不要过warp。**

```plain
            {
                "type": "field",
                "outboundTag": "socks_out",
                "domain": [
									"geosite:netflix",
                	"openai.com",
									]
            },
```

- 重新启动v2ray/xray

```plain
systemctl restart v2ray
systemctl status v2ray
```

如果要应用geosite的域名列表，则需要下载geosite和geoip包放到/usr/local/bin中。

如果想要测试有没有效果，可以通过添加ipip的域名并访问ipip查看是不是服务器对应的ip。

### 其他方案以及问题

当配置完warp其实正常的服务器就可以正常使用了，如果你刚好有没有被封的账号，那你现在已经可以正常使用了。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304141841112.png)

但如果你的账号以某种方式被封了，到这里你还是没办法使用。（有趣的是，这种封禁并不是永久的，他会以某种方式被解封）

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304141841253.png)

而更糟糕的问题是，如果你试图注册一个新的账号，那么很大概率会提示，相同的ip下注册请求过多。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304141841279.png)

关于这个问题其实我没有找到特别完美的方案来解决，网上在这一步使用的方案大多是使用一个冷门的服务器上直接在windows服务器上远程操作实现又或者是让别人来注册。

## 通过API来解决

其实在openai的设定中有个很有意思的设定，就是chatgpt平台和API平台是分开的。

首先大家比较常说的chatgpt其实是在线的一个平台，也就是我们见的最多的对话平台。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304141841821.png)

这个平台的限制最严格，账号最容易被封，但优势是在这个网页我们可以获得一手的使用体验，如果升级plus还可以优先使用chatgpt后续的最新更新。

但事实上大部分朋友其实是不需要这些东西的，我们可以通过chatgpt的api配合一些第三方开发的平台来实现。

而ChatGPT的API我们可以在openai的platform上看到

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304141841079.png)

这个平台其实对国内用户的封禁是没那么严格的，而且刚注册的账号是可以在前3个月使用免费的18刀额度，所以很多人其实是选择用这个接口来使用的

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304141841473.png)

如果你把免费的额度使用完之后，你可能会遇到无法付费的问题，具体可以看后面。

在openai的platform后台可以新建一个API keys

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304141841250.png)

配合一些现在做的很不错的二次开发平台，能体验到比原版chatgpt更实用的感受。有一个比较好用的ChatGPT Next Web

https://github.com/Yidadaa/ChatGPT-Next-Web

ChatGPT Next Web提供了一个基于vercel实现的方案，可以允许做一个私有化的平台，并利用web pages功能+绑定自定义域名部署在网上，这个方案相当实用，同样也不依赖科学上网。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304141842076.png)

# 账号邮箱封禁

在初版的ChatGPT注册的时候其实是没有这个限制的，所以大部分朋友估计都没有遇到过这个问题，我最早的账号直接就是Gmail注册的也没遇到类似的问题，但是在**最近chatgpt封禁了大部分我们能用到的邮箱。**

1. QQ邮箱,foxmail邮箱
2. 163邮箱，网易邮箱，126邮箱，新浪邮箱
3. Outlook、hotmail邮箱
4. edu.cn邮箱
5. Gmail，只能快捷登录

正常来说的话，其实我们国内能用得到的大部分邮箱里只有gmail可以正常使用了，但是gmail的注册最近也有一些问题这里就先不聊。

如果你的账号是因为邮箱被封禁，会在注册的时候出现类似的提示。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304141842720.png)

其实比较实用的方案还是用自己的域名，绑定自定义域名应该可以绕过这个问题。但是要搞一个企业邮箱，这里先不提了。

# 手机号封禁

在解决了邮箱的问题之后，你遇到的第二个问题就是手机号封禁，在注册openai的账号的同时你需要一个国外的手机号接受短信，一般来说SMS Active是比较实用的一个网站，国内可以直接用visa卡来支付，收费也不贵以后也用得到.

https://sms-activate.org/cn/

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304141842107.png)

这个网站租用的虚拟手机号是专门用来注册各种各样的网站的，其中就有openai，我们可以直接选openai

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304141842513.png)

然后选择对应的国家点击购买就会获得一个随机的手机号，复制手机号到openai对应激活接受短信即可。由于这个平台大家使用的还是比较多的，所以可以用稍微冷门一点儿的国家手机号有效度比较高，如果激活失败可以多尝试几个手机。

# 使用相关的问题

当你搞定所有的问题之后，你可能会遇到一些使用上的问题，这里我写两个最常见的问题。

首先大家比较常说的chatgpt其实是在线的一个平台

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304141842292.png)

在这个平台里可以选择不同的model比如GPT4，每个session都是独立的，会保存一定程度的上下文，但同样有很多的限制，其中最常见的就是GPT-4，3小时只能发送25条消息。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304141842124.png)

而ChatGPT plus也是针对这个页面的收费服务，ChatGPT本身是免费的，但如果你订阅ChatGPT plus会有很多额外的权益，其中最实用的就是更快的响应速度和新功能优先使用。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304141842079.png)

但其中容易被忽略的问题是，ChatGPT plus看上去并不便宜，需要20刀每个月，但其实并不提升API，**plus只是单纯针对Chat网页的优化**，而更关键的是，由于ChatGPT在后续的更新中加入了CF做防护，现在基本上已经没办法通过通过第三方来模拟ChatGPT了，大部分都是使用官方提供的API接口。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304141842203.png)

而API的收费方式是根据tokens计算的，你可以简单的把tokens认为是单词片段。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304141842684.png)

而ChatGPT的API我们可以在openai的platform上看到

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304141842833.png)

这个平台其实对国内用户的封禁是没那么严格的，而且刚注册的账号是可以在前3个月使用免费的18刀额度，所以很多人其实是选择用这个接口来使用的

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304141842183.png)

如果你解决不了chat界面的封禁，可以尝试直接使用api来使用，也是个不错的方案。

当你把免费额度使用完之后，你会遇到一个新的问题，就是如何支付？

## 如何支付？

其实除开服务器的ip限制，最麻烦的问题就是这个，其实大部分的国外网站都是可以使用visa卡直接支付的，很多银行都有全球卡，但麻烦的是，openai似乎对这方面做了一定的限制，你无法使用国内银行卡来支付，你会遇到一个类似这样的提示。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304141842043.png)

我在研究了很多方案以后，得到了一个最简单的方案，就是depay，这个东西其实最近也挺流行的，depay使用usdt作为标准代币可以申请虚拟信用卡，而且depay的信用卡相当实用，申请的虚拟卡甚至可以绑定微信和支付宝，可以用来把国外的赏金转回来。但比较麻烦的是，depay必须使用udst充值，这方面我就不科普了，可以用各大平台APP来c2c交易。感兴趣的话可以用我的邀请链接注册。

https://depay.depay.one/web-app/register-h5?invitCode=689747&lang=zh-cn

搞定虚拟信用卡之后可以直接在后台绑定卡，要注意depay必须提前充值余额进去才能使用。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304141842439.png)

要注意下面的账单地址最好找个国外的地址，否则可能会触发一些限制，比较简单的方案是直接在google map上搜一个地址贴上去。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304141842885.png)

基本上找个差不多的地址都ok。正常添加完就可以使用了，注意可以加入一点儿使用限制，避免超额

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202304141842908.png)

## 如何使用？

其实在你接触到ChatGPT之前，你可能会接受到无限的关于ChatGPT的吹捧。但当你真正使用ChatGPT的时候，你可能玩的很开心，但确想不到这个东西究竟有啥用，因为他又不能用来查询资料，也不能凭空做工作。当你冷静下来可能会觉得ChatGPT更像是个玩具。

但ChatGPT其实更像是一把铲子，在拥有这把铲子之前，我们只知道可以把土堆成房子，但是不知道用什么把土堆起来，但在有了这把铲子之后，铲土只是铲子最直白的利用，如何用铲子堆一个又大又漂亮的房子可能我们还不知道，但至少我们现在已经开始尝试做这样的事情了。

关于具体的使用方案，在这篇入门篇先不提，下篇再见。
