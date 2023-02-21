---
title: 赛博偶像速成指南
date: 2023-02-21 19:24:14
tags:
---

随着ChatGPT的爆火，最近和人工智能有关的各个部分也有一次爆火起来，由ai制成的美少女也是最近的一个爆火的话题，花了一点儿时间了解了一下，感觉还挺有意思的，现有的工具已经是非常成熟可用的东西了，接下来简单介绍一下怎么玩

<!--more-->

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202302211925401.jpeg)

# Stable Diffusion WebUI

这次我是用的是Stable Diffusion WebUI来生成ai图，这是一款现在非常流行的用ai来绘图的开源工具，给出一组描述词，ai就可以根据描述词画出你想要的图片。现在使用最多的是AUTOMATIC1111改进的图形化版本，支持Linux/Windows/MacOS系統，以及Nvidia/AMD/Apple的GPU，几乎没有门槛，装好即用。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202302211925924.png)

需要注意的是，这个玩意及其吃GPU以及显存，我的工作机跑这个一下就卡死了，很吃力。

# 安装指南

## 基础环境

- N卡或者A卡对应的驱动程序
- python3.10+，最好新一点儿
- Git环境

## Stable Diffusion模型

Stable Diffusion生成图需要基础模型，主要有两部分

- 脸部修复模型GFPGAN
- 绘图使用的相关模型

GFPGAN可以去https://github.com/TencentARC/GFPGAN/releases/tag/v1.3.4直接下载

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202302211925786.png)

还有一部分是绘图相关的模型，这部分模型有很多，可以在很多不同的网站上搜索下载，比如https://huggingface.co/models或者https://civitai.com/都是比较有名的模型下载网站

| **名称**         | **说明**                                | **下载**                                                     |
| ---------------- | --------------------------------------- | ------------------------------------------------------------ |
| Stable Diffusion | CompVis发布的基础模型，适合真人和动物。 | [HuggingFace](https://huggingface.co/stabilityai/stable-diffusion-2) |
| Chilloutmix      | 写实风格的模型，融合真人和动漫风格      | [HuggingFace](https://huggingface.co/TASUKU2023/Chilloutmix) |
| Anything         | 适合漫画                                | [HuggingFace](https://huggingface.co/andite/anything-v4.0)   |
| Waifu Diffusion  | 使用Danbooru图库训练而成，适合漫画      | [HuggingFace](https://huggingface.co/hakurei/waifu-diffusion-v1-4) |

你也可以选择在civitai上直接找自己喜欢的模型下载，ckpt后缀和safetensors后缀都快可以。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202302211925087.png)

在筛选中勾选Checkpoint对应的就是基础绘画模型。

## 安装Stable Diffusion

1、首先从github下载源码

```bash
git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui.git
```

2、把前面下载的GFPGANv1.4.pth放在对应的文件夹下。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202302211925017.png)

3、下载相应的各种依赖库，windows执行bat，linux执行sh。

```bash
cd stable-diffusion-webui
./webui-user.bat
```

每次运行的时候都会通过git同步最新版本的各种库，第一次运行的时候会下载各种依赖库，有点儿大而且涉及github，快的话大概30分钟左右，慢的话可能会很慢。

如果显示web的链接，那么说明所有的依赖已经下载成功了。要注意的是，依赖当中有关torch有可能会遇到很多报错，可以考虑手动下载安装

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202302211925809.png)

4、安装对应的绘画模型

前面提到的https://huggingface.co/models或者https://civitai.com/中下载的模型需要放在models/Stable-diffusion/中

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202302211925420.png)

5、如果GPU的显存小于4G，那么你可以在在启动脚本当中加入--medvram，windows编辑webui-user.bat。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202302211925672.png)

6、当你打开对应的链接可以访问使用时，说明已经安装成功了。

# 简单的使用指南

## 关键字

使用Stable Diffusion生成图时，最重要的就是关键字，分别是**正向关键字**和**负向关键字**。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202302211926663.png)

无论是那个模式都是通过关键字来影响图片生成，这个关键字主要有几个部分。

- 必须是英文输入，你可以使用**关键字词语**，比如girl,long hair等关键字，也可以使用**一段句子来描述**，比如a girl with long hair，或者使用**某个艺术家**的名字来，[Voldy](https://rentry.org/artists_sd-v1-4)可以查到相关的艺术家画风，又或者直接使用某个**特定的动漫人物的名字**，输出相关的作品和角色。ai会根据你的描述来生成图。
- 你可以使用括号增加标签的权重，**括号越多权重越高**。

这里也分享一个Hentai Diffusion分享的万用的负向关键字，可以防止出现断手断脚

```bash
(((deformed))), blurry, bad anatomy, disfigured, poorly drawn face, mutation, mutated, (extra_limb), (ugly), (poorly drawn hands), fused fingers, messy drawing, broken legs censor, censored, censor_bar, multiple breasts, (mutated hands and fingers:1.5), (long body :1.3), (mutation, poorly drawn :1.2), black-white, bad anatomy, liquid body, liquidtongue, disfigured, malformed, mutated, anatomical nonsense, text font ui, error, malformed hands, long neck, blurred, lowers, low res, bad anatomy, bad proportions, bad shadow, uncoordinated body, unnatural body, fused breasts, bad breasts, huge breasts, poorly drawn breasts, extra breasts, liquid breasts, heavy breasts, missingbreasts, huge haunch, huge thighs, huge calf, bad hands, fused hand, missing hand, disappearing arms, disappearing thigh, disappearing calf, disappearing legs, fusedears, bad ears, poorly drawn ears, extra ears, liquid ears, heavy ears, missing ears, fused animal ears, bad animal ears, poorly drawn animal ears, extra animal ears, liquidanimal ears, heavy animal ears, missing animal ears, text, ui, error, missing fingers, missing limb, fused fingers, one hand with more than 5 fingers, one hand with less than5 fingers, one hand with more than 5 digit, one hand with less than 5 digit, extra digit, fewer digits, fused digit, missing digit, bad digit, liquid digit, colorful tongue, blacktongue, cropped, watermark, username, blurry, JPEG artifacts, signature, 3D, 3D game, 3D game scene, 3D character, malformed feet, extra feet, bad feet, poorly drawnfeet, fused feet, missing feet, extra shoes, bad shoes, fused shoes, more than two shoes, poorly drawn shoes, bad gloves, poorly drawn gloves, fused gloves, bad cum, poorly drawn cum, fused cum, bad hairs, poorly drawn hairs, fused hairs, big muscles, ugly, bad face, fused face, poorly drawn face, cloned face, big face, long face, badeyes, fused eyes poorly drawn eyes, extra eyes, malformed limbs, more than 2 nipples, missing nipples, different nipples, fused nipples, bad nipples, poorly drawnnipples, black nipples, colorful nipples, gross proportions. short arm, (((missing arms))), missing thighs, missing calf, missing legs, mutation, duplicate, morbid, mutilated, poorly drawn hands, more than 1 left hand, more than 1 right hand, deformed, (blurry), disfigured, missing legs, extra arms, extra thighs, more than 2 thighs, extra calf,fused calf, extra legs, bad knee, extra knee, more than 2 legs, bad tails, bad mouth, fused mouth, poorly drawn mouth, bad tongue, tongue within mouth, too longtongue, black tongue, big mouth, cracked mouth, bad mouth, dirty face, dirty teeth, dirty pantie, fused pantie, poorly drawn pantie, fused cloth, poorly drawn cloth, badpantie, yellow teeth, thick lips, bad camel toe, colorful camel toe, bad asshole, poorly drawn asshole, fused asshole, missing asshole, bad anus, bad pussy, bad crotch, badcrotch seam, fused anus, fused pussy, fused anus, fused crotch, poorly drawn crotch, fused seam, poorly drawn anus, poorly drawn pussy, poorly drawn crotch, poorlydrawn crotch seam, bad thigh gap, missing thigh gap, fused thigh gap, liquid thigh gap, poorly drawn thigh gap, poorly drawn anus, bad collarbone, fused collarbone, missing collarbone, liquid collarbone, strong girl, obesity, worst quality, low quality, normal quality, liquid tentacles, bad tentacles, poorly drawn tentacles, split tentacles, fused tentacles, missing clit, bad clit, fused clit, colorful clit, black clit, liquid clit, QR code, bar code, censored, safety panties, safety knickers, beard, furry, pony, pubic hair, mosaic, futa, testis, (((deformed))), blurry, bad anatomy, disfigured, poorly drawn face, mutation, mutated, (extra_limb), (ugly), (poorly drawn hands), fused fingers, messy drawing, broken legs censor, censored, censor_bar, multiple breasts, (mutated hands and fingers:1.5), (long body :1.3), (mutation, poorly drawn :1.2), black-white, bad anatomy, liquid body, liquidtongue, disfigured, malformed, mutated, anatomical nonsense, text font ui, error, malformed hands, long neck, blurred, lowers, low res, bad anatomy, bad proportions, bad shadow, uncoordinated body, unnatural body, fused breasts, bad breasts, huge breasts, poorly drawn breasts, extra breasts, liquid breasts, heavy breasts, missingbreasts, huge haunch, huge thighs, huge calf, bad hands, fused hand, missing hand, disappearing arms, disappearing thigh, disappearing calf, disappearing legs, fusedears, bad ears, poorly drawn ears, extra ears, liquid ears, heavy ears, missing ears, fused animal ears, bad animal ears, poorly drawn animal ears, extra animal ears, liquidanimal ears, heavy animal ears, missing animal ears, text, ui, error, missing fingers, missing limb, fused fingers, one hand with more than 5 fingers, one hand with less than5 fingers, one hand with more than 5 digit, one hand with less than 5 digit, extra digit, fewer digits, fused digit, missing digit, bad digit, liquid digit, colorful tongue, blacktongue, cropped, watermark, username, blurry, JPEG artifacts, signature, 3D, 3D game, 3D game scene, 3D character, malformed feet, extra feet, bad feet, poorly drawnfeet, fused feet, missing feet, extra shoes, bad shoes, fused shoes, more than two shoes, poorly drawn shoes, bad gloves, poorly drawn gloves, fused gloves, bad cum, poorly drawn cum, fused cum, bad hairs, poorly drawn hairs, fused hairs, big muscles, ugly, bad face, fused face, poorly drawn face, cloned face, big face, long face, badeyes, fused eyes poorly drawn eyes, extra eyes, malformed limbs, more than 2 nipples, missing nipples, different nipples, fused nipples, bad nipples, poorly drawnnipples, black nipples, colorful nipples, gross proportions. short arm, (((missing arms))), missing thighs, missing calf, missing legs, mutation, duplicate, morbid, mutilated, poorly drawn hands, more than 1 left hand, more than 1 right hand, deformed, (blurry), disfigured, missing legs, extra arms, extra thighs, more than 2 thighs, extra calf,fused calf, extra legs, bad knee, extra knee, more than 2 legs, bad tails, bad mouth, fused mouth, poorly drawn mouth, bad tongue, tongue within mouth, too longtongue, black tongue, big mouth, cracked mouth, bad mouth, dirty face, dirty teeth, dirty pantie, fused pantie, poorly drawn pantie, fused cloth, poorly drawn cloth, badpantie, yellow teeth, thick lips, bad camel toe, colorful camel toe, bad asshole, poorly drawn asshole, fused asshole, missing asshole, bad anus, bad pussy, bad crotch, badcrotch seam, fused anus, fused pussy, fused anus, fused crotch, poorly drawn crotch, fused seam, poorly drawn anus, poorly drawn pussy, poorly drawn crotch, poorlydrawn crotch seam, bad thigh gap, missing thigh gap, fused thigh gap, liquid thigh gap, poorly drawn thigh gap, poorly drawn anus, bad collarbone, fused collarbone, missing collarbone, liquid collarbone, strong girl, obesity, worst quality, low quality, normal quality, liquid tentacles, bad tentacles, poorly drawn tentacles, split tentacles, fused tentacles, missing clit, bad clit, fused clit, colorful clit, black clit, liquid clit, QR code, bar code, censored, safety panties, safety knickers, beard, furry, pony, pubic hair, mosaic, futa, testis
```

## 合并多个绘图模型

在进入web平台之后，左上角可以选择你使用的绘画模型。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202302211926509.png)

我使用的过程中发现，这个绘画模型要比关键字对绘图来说更关键，如果有一个很好的绘画模型会很容易生成好看的图，而stable diffusion提供了合并模型的功能，而现在很多不错的模型都是混搭出来的，比如最近非常流行的Korean Doll Likeness就是由[Uber Realistic Porn Merge (URPM)](https://civitai.com/models/2661/uber-realistic-porn-merge-urpm)**-0.7** **and** [ChilloutMix](https://civitai.com/models/6424/chilloutmix)**-0.3**混搭而来。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202302211926752.png)

你也可以选择用这个功能混搭一个自己喜欢的基础模型。

## 文生图或者图生图

前面提到stable diffusion生成图片是通过关键字实现的，除了文字以外，还提供了提供图片生成的图生图功能。你可以使用上传图片+关键字组合的方式生成图片。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202302211926447.png)

其中的很多配置我也没搞明白，就说几个比较关键的配置。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202302211926005.png)

- sampling steps代表图片演化的步骤，可以调高一点儿30、40左右。
- width和Height是生成的图片大小，建议不要调这个，这个调大了很卡。
- batch count表示生成图片的数量，建议不要太高，占用太高可能会闪退。

## 使用分享的关键字以及参数

在了解的前面的东西之后，你应该可以生成一些各种类型的图片了，但是我说实话，生成的很多图效果都很差，这个时候可以去**civitai**上去逛一逛，找一点儿别人分享的图和参数来看看。

随便选一个喜欢的分享，点进去往下翻，可以看到很多使用该模型生成的图

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202302211926832.png)

随便选一个喜欢的图，右下角点击叹号，可以看到生成该图使用的各种参数。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202302211926562.png)

你可以把所有的配置直接偷下来，使用该种子就可以生成你想要的图，也可以不指定种子就可以用这个模板来生成图。

除了这种比较简单的关键字参数以外，你还可以在stable diffusion引用lora模型，在civitai上筛选LORA

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202302211926355.png)

选择一个你喜欢的LORA模型，在右上角下载。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202302211926803.png)

然后放在models/Lora里面

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202302211926995.png)

然后在右边generate的下面点击粉色图标，点到Lora中就可以看到刚才导入的Lora模型

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202302211926556.png)

点选就可以在关键字的基础上再引入额外的绘画模型。可以画出更有风格的图。

## Inpaint 辅助绘制

除了简单的生成图片以外，stable diffusion还有很神奇的功能是辅助绘制，在图生图里，有个分栏叫Inpaint，你可以选择涂黑图片的一部分，ai就只会修改涂黑的部分

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202302211926902.png)

比如这里我涂黑了裙子部分，ai就自动生成了一个腰带，通过这个功能可以智能的修改某个东西，还是很有趣的。

# 写在最后

其实这个玩意出来已经很多年了，之前看过一些但是当时还比较简单，大部分只是基于图包训练，很多结果说实话都很一般，最近又火起来之后就又研究了一下，没想到这个玩意已经成熟到这个地步了，感觉还蛮有意思的，很多图成熟度已经非常高了，估计以后会有大量的赛博偶像了:>
