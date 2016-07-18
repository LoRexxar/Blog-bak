title: Windows Hexo博客安装配置优化（小白篇）
date: 2015-05-22 01:23:54
tags:
- Hexo
- Blogs
categories:
- Blogs
---
折腾了好多天的Hexo博客，过程各种日狗，现在分享我折腾这么久的经验，包括完整的windows环境下的安装配置和优化...
<!--more-->
**我们为什么要使用hexo搭建个人博客？**
在使用hexo搭建个人博客，我们首先应该知道，为什么我们要用hexo搭建个人博客？
Hexo出自台湾大学生 tommy351 之手，是一个基于 Node.js 的静态博客程序，其编译上百篇文字只需要几秒。hexo生成的静态网页可以直接放到GitHub Pages，BAE，SAE等平台上。正是因为这样，全静态的博客不存在后台，静态网页放到GitHub Pages等平台上，简单粗暴...
在写博客之前，推荐一篇别人的博文，也是一篇很好的文章[如何搭建一个独立博客——简明Github Pages与Hexo教程](http://cnfeat.com/2014/05/10/2014-05-11-how-to-build-a-blog/)
下面我要详细讲一下用hexo搭建博客的整个过程...

# Hexo的安装 #
## 安装git ##
[git](http://git-scm.com/downloads)
选择适合自己系统的git下载安装，我选择的windows。
## 安装node.js ##
[node.js](https://nodejs.org/download/)
同样选择适合自己系统的下载
## 安装编辑器（可选）##
在后面配置文件的时候，系统自带的记事本支持很差，为了能够更方便的编辑，可以选择下载Sublime Text3或者Notepad++，方便编辑...
## 安装Hexo ##
打开刚才安装的git bash,输入下面的指令。
```
npm install -g hexo
```
## 更新Hexo ##
更新Hexo到最新版
```
npm update hexo -g
```
## 部署Hexo ##
在你想要放置hexo博客文件的位置建立一个名为hexo的文件夹，然后在这个文件夹中右键打开Git Bash。
```
$ hexo init
```
然后，hexo会自动在这个文件夹里生成你建立网站的所有文件。
到这里我们已经搭建起本地的博客了，执行以下命令，同样在hexo文件夹下右键Git Bash输入以下命令：
```
$ hexo g
$ hexo s
```
上面的hexo g就是hexo generate，hexo d就是hexo deploy，到这里本地的博客已经搭起来了，你可以执行以下命令开启本地服务：
```
hexo server
```
打开浏览器输入 http://localhost:4000可以看到效果，如果没记错，初始的模板是挺丑的，所以不要急，下面我们还要优化，下面我们要把hexo博客部署到网络上面，这样就可以让别人浏览到...

**hexo博客只支持ie8以上的浏览器，否则，后果你懂的。**
# 配置gitcafe-Pages or github-Pages #
我们搭建的是全静态的前台博客，不同于wordpress，我们必须把源码放在网络上，这样我们才能在网络上看到它。这里我们选择的是国内的Gitcafe-Pages,不同于大多数人选择的github-pages，gitcafe的设置更加简单，而且不用担心被墙，下面我们来配置gitcafe。
## 注册gitcafe ##
[gitcafe官网](https://gitcafe.com)
## 添加项目 ##
在个人主页中可以新建一个空的项目，写好名称，记得名称要和自己的id一致，比如博主的LoRexxar
## 配置ssh密钥 ##
ps:如果看不懂我的教程，你也可以看[gitcafe官方教程](https://gitcafe.com/GitCafe/Help/wiki/如何安装和设置-Git#wiki)
1.打开桌面的git bash，检查本机的密钥。
```
 $ cd ~/. ssh
```
如果提示：No such file or directory 说明你是第一次使用git。
2.如果不是第一次使用，执行下面的操作：
```
 $ mkdir key_backup
 $ cp id_rsa* key_backup    
 $ rm id_rsa*
```
3.生成自己的密钥，记得把“”中间的邮箱改成自己的。
```
ssh-keygen -t rsa -C "YOUR_EMAIL@YOUREMAIL.COM"
```
生成过程中会出现以下信息，按屏幕提示操作，并记得输入 passphrase 口令（这里直接回车过去既可，以后出现这个都是空密码，直接回车即可）
```
$ ssh-keygen -t rsa -C "YOUR_EMAIL@YOUREMAIL.COM"
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /c/Users/USERNAME/.ssh/id_rsa.
Your public key has been saved in /c/Users/USERNAME/.ssh/id_rsa.pub.
The key fingerprint is:
15:81:d2:7a:c6:6c:0f:ec:b0:b6:d4:18:b8:d1:41:48 YOUR_EMAIL@YOUREMAIL.COM
```
4.SSH 秘钥生成结束后，你可以在用户目录 (~/.ssh/) 下看到私钥 id_rsa 和公钥 id_rsa.pub 这两个文件，记住千万不要把私钥文件 id_rsa 透露给任何人。不然别人就可以以管理员的身份登陆，修改你项目中的文件（也就是你的博客）。
5.添加ssh到gitcafe
使用任意文本工具打开公钥文件  ~/.ssh/id_rsa.pub,复制里面的所有内容。

进入 GitCafe -->账户设置-->SSH 公钥管理设置项，点击添加新公钥 按钮，在 Title 文本框中输入你的id，在 Key 文本框粘贴刚才复制的公钥字符串，按保存按钮完成操作。

6.测试是否可以链接服务器，同样打开Git Bash输入：
```
ssh -T git@gitcafe.com
```
如果是第一次连接，会出现以下警告：
```
The authenticity of host 'gitcafe.com (50.116.2.223)' can't be established.
#RSA key fingerprint is 84:9e:c9:8e:7f:36:28:08:7e:13:bf:43:12:74:11:4e.
#Are you sure you want to continue connecting (yes/no)?
```
上面出现的其实是所谓的指纹，用来识别你接受到的是否和服务器发送的一致，来识别是否收到中间人攻击，这里直接yes既可。之后会提示出入passphrase口令，如果上面设置没问题，直接回车即可。

最后出现下面的字符，就说明成功了。
```
Hi USERNAME! You've successfully authenticated, but GitCafe does not provide shell access.
```
## 设置git的账户信息 ##
打开桌面的git bash ，输入：
```
$ git config --global user.name "yourname"                  //gitcafe上的用户名
$ git config --global user.email "yourmaill@yourmaili.com"  //填写自己的邮箱
```

## 添加Pages目录 ##
如果直接上传，上传上去的只是所谓的博客源码，如果我们要让我们博客显示出来，就要建立pages目录，同样先放上[gitacfe的官方文档](https://gitcafe.com/GitCafe/Help/wiki/Pages-相关帮助#wiki)地址
1.在本地创建一个Git目录
2.添加一个试验性的html文档
3.提交第一个版本
4.添加项目remote
5.创建一个gitcafe-pages的分支，并切换到该分支
6.提交该分支至GitCafe

打开hexo目录，右键git bash输入下面的代码，期中很多都会报错，但是没关系，每一个都输入了就会建立pages分支。
```
echo 'Hello, world' > index.html
git init
git add .
git commit -a -m 'Hello, world!'
git remote add origin git@gitcafe.com:LoRexxar/LoRexxar.git     //把这里的LoRexxar替换成自己的id
git checkout -b gitcafe-pages
git push origin gitcafe-pages
```
完成上面的所有操作，在gitcafe的项目页面中便会出现gitcafe-pages的分支，进入项目设置，把默认分支改为gitcafe-pages，到这里我们已经基本完成了gitcafe的配置...
## 删除默认的master分支（可选）##
由于我们要建立个人博客，所以只需要使用到gitcafe-pages的分支，而且使用pages的时候有可能会产生冲突或者仅仅是因为洁癖，我们想要删除默认的master分支。
这里放上[官方文档](https://gitcafe.com/GitCafe/Help/wiki/如何删除-Master-分支)，由于不是必须，这里就不赘述了。

## 配置hexo配置文件_config.yml ##
配置好全部的hexo和gitcafe后，我们必须把两者联系起来，这里打开hexo根目录下的_config.yml,这里就需要用到所谓的编辑器了，如果仅仅使用记事本的话，这里打开可能导致看到的是一篇无格式的乱码，这里我们使用Sublime Text3，打开之后，把最后面的deploy改为图中的样子。
![](/img/hexo/1.png)
github的配置信息和gitcafe不同是下面这样的（其中的github必须改为git，hexo3.0后的改动）
![](/img/hexo/2.png)
**注意：这里使用的是nodo.js语法，这种语法对格式的要求相当高，：后必须跟上空格，不然会报错，请严格按照格式来写。**

## 绑定自定义域名 ##
域名的申请和购买就不多赘述，可以自行百度想要的域名，然后绑定到自己的博客上。
1.进入你的Page Repo的项目管理设置 
2.你将会看到左侧的导航栏中有自定义域名设置，点击进入，输入你想要设置的域名：
3.需要注意的是，请正确填写你期望自定义的域名，填写时请不要填写http://这样的协议，当然如果你填写了，我们会智能的擦除这些字符。
4.最后在你的域名管理界面添加一个CNAME记录，将它指向GitCafe Pages服务器的domain：gitcafe.io。如果你的 DNS 管理商不提供 CNAME 记录，请添加 A 记录到 207.226.141.162。
**ps:由于官方随时可能更换自己的服务器地址，所以请随时关注官网**
## 完成配置 ##
到这里，基本的配置已经完成，在hexo的目录中打开git bash，输入：
```
hexo g
hexo d
```
**ps:每次修改自己的任何配置文件，或者新建了博文，都必须输入这两个指令上传！**

如果没有爆出错误，说明配置成功，可以打开xxx.gitcafe.io(xxx为自己的id)查看自己的博客主页。如果报错，请检查前面的配置有没有错误，如果找不到错误，请参见后面的**hexo常见报错**。


# Hexo的优化 #
完成安装后，我们就能看到自己的博客页面，但是很明显页面中还是很简陋，那么我们怎么把页面改成自己想要的样子呢？
## Hexo中的主题 ##
Hexo的默认安装中提供了一个主题，如果不喜欢的话，我们可以下载自己喜欢的主题进行定制。由于更新了hexo3.0，所以很多主题可能都处于不支持阶段，识别的方式非常简单，如果在正常的环境下hexo g&d能够成功，但更换主题会爆出未知错误的话，这里就有很大的可能是因为主题不支持。
[官网主题](https://github.com/hexojs/hexo/wiki/Themes)可以找到很多的主题,这里也贴出来我使用的主题[地址](https://github.com/iissnan/hexo-theme-next)
挑选好主题后，在hexo目录下打开git bash，输入：
```
git clone https://github.com/iissnan/hexo-theme-next themes/next

#在./_config.yml，修改主题为writing
theme: next

hexo g       #hexo generate简写
hexo s       #hexo server简写
```
这里每一个主题和主题的名字都有不同，但大部分主题，都会提供详细的书写方式，仔细查看既可。

## hexo站点_config.yml的详细配置 ##
这里比较复杂，甚至因不同的主题也会有不同，这里贴上我的全部配置信息作详解：
```
# Hexo Configuration
## Docs: http://hexo.io/docs/configuration.html
## Source: https://github.com/hexojs/hexo/

# Site
title: LoRexxar's Blog                         #博客的名字
subtitle: 见证菜鸟小白的成长之路...				#网站的副标题					
description:									#描述
author: LoRexxar								#网站作者
email: guoyingqi001@126.com						#邮箱

language: zh-Hans								#已启用的语言，hans是简体中文
# language: fr-FR								#若要更换，把前面的#删掉修改，记得严格遵守格式
# language: zh-hk
# language: zh-tw
# language: ru
timezone:

# URL
## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
url: http://yoursite.com
root: /
permalink: :year/:month/:day/:title/
permalink_defaults:

# Directory
source_dir: source
public_dir: public
tag_dir: tags
archive_dir: archives
category_dir: categories
code_dir: downloads/code
i18n_dir: :lang
skip_render:

# Writing
new_post_name: :title.md # File name of new posts
default_layout: post
titlecase: false # Transform title into titlecase
external_link: true # Open external links in new tab
filename_case: 0
render_drafts: false
post_asset_folder: false
relative_link: false
future: true
highlight:
  enable: true
  line_number: true
  auto_detect: true
  tab_replace:

# Category & Tag
default_category: uncategorized
category_map:
tag_map:

# Date / Time format
## Hexo uses Moment.js to parse and display date
## You can customize the date format as defined in
## http://momentjs.com/docs/#/displaying/format/
date_format: YYYY-MM-DD
time_format: HH:mm:ss

# Pagination
## Set per_page to 0 to disable pagination
per_page: 10                                          #每页显示的博文个数
pagination_dir: page

# Extensions
## Plugins: http://hexo.io/plugins/
## Themes: http://hexo.io/themes/
theme: next											#主题设置

baidu_analytics: LoRexxar                             #百度分析开启

# Deployment
## Docs: http://hexo.io/docs/deployment.html

deploy:
  type: git
  repository: git@gitcafe.com:LoRexxar/LoRexxar.git
  branch: gitcafe-pages
```

## 主题的优化 ##
这部分由于不同的主题配置可能完全不一样，我只能以我是用的next主题作为例子，[next的官方文档地址](https://github.com/iissnan/hexo-theme-next)

### 加入多说/disqus评论系统 ###
1.首先我们注册[多说](http://dev.duoshuo.com/)，并登录账号，获取通用代码。
2.这里就会有一些不同，默认的多说插件代码在themes\landscape\layout\_partial\article.ejs文件中，
这里是[官方文档地址](http://dev.duoshuo.com/threads/541d3b2b40b5abcd2e4df0e9)，只要照着文档做，大部分都会搞定。
3.但是有一部分主题并不存在article文件，就比如博主使用的主题，在\themes\next\layout\_scripts\comments下能发现一个明显的duoshuo.swig，打开覆盖刚才复制的代码，然后保存并部署hexo，我们会发现多说插件已经出现，但是多说插件出现在页面的外面，并且长度颜色都不符合我们的要求，所以我们还要修改。
4.修改多说评论框的css文件，这里是[官方文档](http://dev.duoshuo.com/docs/4ff1cfd0397309552c000017),登录自己的多说，点开设置，设置中的基本设置，向下滑找到自定义css，修改添加参数，既可修改成想要的样子。下面是博主的css：
```
#ds-thread {width: 800px; text-align:center; margin-left:auto; margin-right:auto}
```
5.光是这样我们发现远远不够，首先就是评论框不会随着文章而单独，甚至会爆出data-thread-key错误，我们打开刚才的duoshuo.swig,把其中的前两行代码改为（不包括start和end那两行）
```
<div class="ds-thread" data-thread-key="{{page.path}}"	data-title="{{page.title}}" data-url="{{page.permalink}}"></div>
```
关于评论框调用参数，[官方文档的地址](http://dev.duoshuo.com/docs/5003ecd94cab3e7250000008/)
6.最后一步，这时候我们发现大部分地方的评论框已经正常了，但是位置仍然不符合我的要求，于是打开/themes/next/layout下的各个文件，寻找这样一条代码:
```
{% include '_scripts/comments/duoshuo.swig' %}
```
这条语句就是多说评论框的引用，博主的选择是加在post.swig文件中的这个位置：

![](/img/hexo/3.png)
7.这下发现大部分问题都解决了，如果你自认为nodo.js水平高，就可以自己寻找代码的位置，然后加到想要的地方

### 增加百度统计/google ###
1.首先同样是注册，注册百度统计站长版并登录，获得代码。
2.这里又一次遇到一样的问题，就是不同主题这里的设置会不一样，原版是在hexo\themes\light\layout\_partial\after_footer.ejs里增加如下代码(这里是我的代码，注意修改)：
```
<script type="text/javascript">
var _bdhmProtocol = (("https:" == document.location.protocol) ? " https://" : " http://");
document.write(unescape("%3Cscript src='" + _bdhmProtocol + "hm.baidu.com/h.js%3Fcfedc723a9dc30bd7db67ad8e53a97fa' type='text/javascript'%3E%3C/script%3E"));
</script>
```
百度统计异步代码是以异步加载形式加载了网站分析代码，使用该代码能够大幅提升您网站的打开速度(目前使用百度统计异步代码会导致百度统计图标和代码检查功能的失效).使用这种方式需要将代码添加至网站全部页面的标签前, 因此只需要在 hexo\themes\light\layout\_partial\head.ejs 里添加如下代码[这里添加的是我的代码, 请适当修改 cfedc723a9dc30bd7db67ad8e53a97fa 这部分成你的ID]
```
<script>
var _hmt = _hmt || [];
(function() {
  var hm = document.createElement("script");
  hm.src = "//hm.baidu.com/hm.js?cfedc723a9dc30bd7db67ad8e53a97fa";
  var s = document.getElementsByTagName("script")[0]; 
  s.parentNode.insertBefore(hm, s);
})();
</script>
```
如果记得没错，这里就是刚才复制的代码。
3.如果是博主所使用的next主题，百度分析的添加方式便是不同的，在\themes\next\layout\_scripts\analytics中可以找到百度分析的文件，打开并对应覆盖复制的代码，保存打开站点的 _config.yml ，新增字段 google_analytics 或者 baidu_analytics（取决于使用的统计系统）
```
google_analytics: your-analytics-id

baidu_analytics: your-analytics-id
```
4.这里配置便完成了，然后在百度统计的地方，可以输入域名检查是否安装成功。
### 其他 ###
1.如果想要修改边框，头像，边栏超链接地址，这个一般因不同的主题而异，所以不好详述，详细要参阅对应的官方文档。

2.其实hexo还能添加很多不同的插件，博主修为尚浅，所以就不写出不清楚的插件用法，这里贴上一些别人的博客。
[Hexo 优化与定制](http://lukang.me/2014/optimization-of-hexo.html)
[hexo](http://zipperary.com/categories/hexo/)


# Hexo的使用 #
上面密密麻麻的写了一大堆，我们终于建好了自己的博客，可是我们发现我们能看到博客，能浏览博文，却没有发写博客，这里介绍怎么使用
## 常用命令 ##
```
hexo new "postName"                     #新建文章
hexo new page "pageName"                #新建页面
hexo generate                           #生成静态页面至public目录
hexo server                             #开启预览访问端口（默认端口4000，'ctrl + c'关闭server）
hexo deploy                             #将.deploy目录部署到GitHub
```
1.首先是第一个命令，也就是最常用的命令，postname可以替换为我们想要的文章名，文章名方便记忆就好，只是在书写的时候我们能够区分就可以了，别人是看不到的。
2.第二个命令比较复杂，但是如果仅仅是写博文的话，第二个是用不到的。
3.35命令就不用多解释了，每次写完博文，都需要这两个命令上传。
## Markdown语法 ##
1.当我们输入命令，新建了一个文章，我们发现在hexo文件夹下source/_posts下生成了一个md文件，md文件是什么呢？我们首先得了解Markdown语法。
2.这里有一篇文章[简书](http://www.jianshu.com/p/q81RER)，足够你得到你想要的左右信息。
3.下面我们需要下载一个编辑器，方便我们写博客，这里推荐的是Markdowmpad 2，左右分栏，你可以实时看到页面编辑出来的样子（有时候会有bug，不要在意）
4.文章的头：
```
title: postName               #文章页面上的显示名称，可以任意修改，不会出现在URL中
date: 2013-12-02 15:30:16     #文章生成时间，一般不改，当然也可以任意修改
categories: 
- example                     #分类,多个分类依次
tags:    
- tag1                        #文章标签，可空，多标签请用格式，注意:后面有个空格
description:                  #附加一段文章摘要，字数最好在140字以内。
---
```
要注意这里格式如果错误，上传时候会报错！
5.Markdown语法备忘
由于页面会编译，所以还是贴上[链接](http://www.jb51.net/article/56296.htm)
6.语法的神奇和强大远远超过你的想象，慢慢感受！
## 关于本地图片和所属路径的问题 ##
当我们想使用我们自己的本地图片的时候，我们会纠结一件事，就是我们该把文件放在哪？路径又该怎么引用，我来解释下这个问题：
1.当我们编辑好我们的博文的时候，我们输入指令hexo g ，会自动生成所对应的html页面添加到hexo文件夹下的public，然后上传public，所以public便是我们的主目录。
2.但是我们发现public内的东西是修改不了的，所以本地图片不能存放在public中，所以我们要把图片放在source中，在source中新建img文件夹，然后建对应博文的文件夹，放置所属的图片，这样当我们需要引用本地图片的时候，则输入这样的路径(/img/Hexo_Blogs/1.png),然后我们就发现，图片没有问题了。


# Hexo 常见问题以及解决方案 #
在搭配hexo的时候我们总会遇到各种各样的问题，下面汇总了所有的解决方案，在查阅之前，先检查是不是因为nodo.js的语法问题报错，这个错误占80%的错误！！
1.找不到git部署

ERROR Deployer not found: git
解决方法:
```
npm install hexo-deployer-git --save
```
2.由于引用不便，这里把解决方案的链接放上来，以供参阅

[Hexo常见问题解决方案](http://xuanwo.org/2014/08/14/hexo-usual-problem/#%E6%9C%AC%E5%9C%B0%E6%B5%8F%E8%A7%88%E6%B2%A1%E9%97%AE%E9%A2%98%EF%BC%8CDeploy%E6%8A%A5%E9%94%99)









总体来说，hexo博客的功能远远超过想象，这里把详细的搭建方式放上来，并且很多地方都是细化讲解，希望小白也能通过这篇博文搭建自己的博客...