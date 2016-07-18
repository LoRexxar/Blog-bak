title: 简单CSP&Bypass
date: 2016-03-17 16:22:31
tags:
- Blogs
- csp
categories:
- Blogs
---

最近杠了几道和csp有关的题目，所以就好好研究了下csp的问题，记录下我知道的简单的csp和bypass方法

<!--more-->

# 什么是CSP?

**Content Security Policy** (CSP) is an added layer of security that helps to detect and mitigate certain types of attacks, including Cross-Site Scripting (XSS) and data injection attacks. These attacks are used for everything from data theft to site defacement or distribution of malware.

简单来说，csp就是为了减少xss，csrf等攻击的，是通过控制可信来源的方式，类似于同源策略...

# 支持CSP的浏览器

Although Content Security Policy first shipped in Firefox 4, that implementation, using the X-Content-Security-Policy header, pre-dated the existence of a formal spec for CSP. Firefox 23 contains an updated implementation of CSP that uses the unprefixed Content-Security-Policy header and the directives as described in the W3C CSP 1.0 spec.


CSP主要有三个header，分别是：Content-Security-Policy，X-Content-Security-Policy，X-WebKit-CSP

- Content-Security-Policy
  chrome 25+，Firefox 23+，Opera 19+
- X-Content-Security-Policy
  Firefox 23+，IE10+
- X-WebKit-CSP
  Chrome 25+

平时见的比较多的都是第一个Content Security Policy

# CSP配置

这一部分的东西基本都是来自于[w3c的文档](https://www.w3.org/TR/CSP3/#source-expression)

## CSP的来源

我们经常见到的CSP都是类似于这样的：
```
header("Content-Security-Policy:default-src 'none'; connect-src 'self'; frame-src 'self'; script-src xxxx/hctfj6/js/ 'sha256-KcMxZjpVxhUhzZiwuZ82bc0vAhYbUJsxyCXODP5ulto=' 'sha256-u++5+hMvnsKeoBWohJxxO3U9yHQHZU+2damUA6wnikQ=' 'sha256-zArnh0kTjtEOVDnamfOrI8qSpoiZbXttc6LzqNno8MM=' 'sha256-3PB3EBmojhuJg8mStgxkyy3OEJYJ73ruOF7nRScYnxk=' 'sha256-bk9UfcsBy+DUFULLU6uX/sJa0q7O7B8Aal2VVl43aDs=';font-src xxxx/fonts/ fonts.gstatic.com; style-src xxxx/css/ fonts.googleapis.com; img-src 'self'");
```
里面包括了各种各样的写法：

1、Keywords such as 'none' and 'self' (which match nothing and the current URL’s origin, respectively)

2、Serialized URLs such as https://example.com/path/to/file.js (which matches a specific file) or https://example.com/ (which matches everything on that origin)

3、Schemes such as https: (which matches any resource having the specified scheme)

4、Hosts such as example.com (which matches any resource on the host, regardless of scheme) or *.example.com (which matches any resource on the host or any of its subdomains (and any of its subdomains' subdomains, and so on))

5、Nonces such as 'nonce-qwertyu12345' (which can match specific elements on a page)

6、Digests such as 'sha256-abcd...' (which can match specific elements on a page)

1、第一种就是类似于none和self这样的，一种会什么都不匹配，一种会匹配同源的来源。
2、第二种是类似于https://example.com/path/to/file.js这样的会匹配特殊的目录，或者https://example.com/这样会匹配所有来自这个来源的。
3、第三种是类似于https:，会匹配所有包含这个特殊的格式的来源。
4、也有可能是example.com这样的，会匹配所有这个host的来源，无论什么可是，或者回有*.example.com,会匹配这个host的所有子域。
5、第五种是类似于nonce-qwertyu12345会匹配一个特殊的节点。
6、当然还有加密过的类似于sha256-abcd...同样会匹配页面中一个特殊的节点。


在文档上能够找到一个详细的例子：
```
serialized-source-list = ( source-expression *( RWS source-expression ) ) / "'none'"
source-expression      = scheme-source / host-source / keyword-source
                         / nonce-source / hash-source

; Schemes:
scheme-source = scheme ":"
                ; scheme is defined in section 3.1 of RFC 3986.

; Hosts: "example.com" / "*.example.com" / "https://*.example.com:12/path/to/file.js"
host-source = [ scheme-part "://" ] host-part [ port-part ] [ path-part ]
scheme-part = scheme
host-part   = "*" / [ "*." ] 1*host-char *( "." 1*host-char )
host-char   = ALPHA / DIGIT / "-"
port-part   = ":" ( 1*DIGIT / "*" )
path-part   = path
              ; path is defined in section 3.3 of RFC 3986.
   
; Keywords:
keyword-source = "'self'" / "'unsafe-inline'" / "'unsafe-eval'" 

; Nonces: 'nonce-[nonce goes here]'
nonce-source  = "'nonce-" base64-value "'"
base64-value  = 1*( ALPHA / DIGIT / "+" / "/" / "-" / "_" )*2( "=" ) 

; Digests: 'sha256-[digest goes here]'
hash-source    = "'" hash-algorithm "-" base64-value "'"
hash-algorithm = "sha256" / "sha384" / "sha512"
```

有个小问题是关于使用ip的
**Note: Though IP address do match the grammar above, only 127.0.0.1 will actually match a URL when used in a source expression (see §6.1.11.2 Does url match source list? for details). The security properties of IP addresses are suspect, and authors ought to prefer hostnames whenever possible.**

使用ip尽管会匹配到语法，但是对ip的安全属性是怀疑的，作者如果可以最好还是用域名。

## CSP的属性

### child-src
The child-src directive governs the creation of nested browsing contexts (e.g. iframe and frame navigations) and Worker execution contexts.

child-src会匹配iframe和frame标签
```
Given a page with the following Content Security Policy:
Content-Security-Policy: child-src https://example.com/
Fetches for the following code will all return network errors, as the URLs provided do not match child-src's source list:

<iframe src="https://not-example.com"></iframe>
<script>
  var blockedWorker = new Worker("data:application/javascript,...");
</script>
```

### connect-src
The connect-src directive restricts the URLs which can be loaded using script interfaces.

connect-src会阻止a的ping属性，也控制着websocket的连接，有点难描述，举个例子。

```
<a ping="https://not-example.com">...
<script>
  var xhr = new XMLHttpRequest();
  xhr.open('GET', 'https://not-example.com/');
  xhr.send();

  var ws = new WebSocket("https://not-example.com/");
        
  var es = new EventSource("https://not-example.com/");

  navigator.sendBeacon("https://not-example.com/", { ... });
</script>

```

这样的属性都会返回网络错误。

### default-src
The default-src directive serves as a fallback for the other fetch directives.

这个属性代表着默认属性，一般来说default-src 'none'; script-src 'self'这样的情况就会是script-src遵循self，其他的都会使用none。

```
如果设置了
Content-Security-Policy: default-src 'self'; script-src https://example.com
就会出现
Content-Security-Policy: child-src 'self';
                         connect-src 'self';
                         font-src 'self';
                         img-src 'self';
                         media-src 'self';
                         object-src 'self';
                         script-src https://example.com;
                         style-src 'self'
这样的情况
```

### font-src
The font-src directive restricts the URLs from which font resources may be loaded.

这个属性管理的css中的字体来源。
```
Given a page with the following Content Security Policy:
Content-Security-Policy: font-src https://example.com/
Fetches for the following code will return a network errors, as the URL provided do not match font-src's source list:

<style>
  @font-face {
    font-family: "Example Font";
    src: url("https://not-example.com/font");
  }
  body {
    font-family: "Example Font";
  }
</style>
```

### img-src
The img-src directive restricts the URLs from which image resources may be loaded.

这个属性包括了所有包括image属性的request。
```
Given a page with the following Content Security Policy:
Content-Security-Policy: img-src https://example.com/
Fetches for the following code will return a network errors, as the URL provided do not match img-src's source list:

<img src="https://not-example.com/img">
```
### manifest-src
The manifest-src directive restricts the URLs from which application manifests may be loaded [APPMANIFEST].

这个属性不太熟，比较常见的就是link

```
Given a page with the following Content Security Policy:
Content-Security-Policy: manifest-src https://example.com/
Fetches for the following code will return a network errors, as the URL provided do not match manifest-src's source list:

<link rel="manifest" href="https://not-example.com/manifest">
```

### media-src
The media-src directive restricts the URLs from which video, audio, and associated text track resources may be loaded.

这个属性主要针对的是audio video以及连带的文本
```
Content-Security-Policy: media-src https://example.com/
Fetches for the following code will return a network errors, as the URL provided do not match media-src's source list:

<audio src="https://not-example.com/audio"></audio>
<video src="https://not-example.com/video">
    <track kind="subtitles" src="https://not-example.com/subtitles">
</video>
```

### object-src
he object-src directive restricts the URLs from which plugin content may be loaded.

不太熟的属性，好像是和flash相关的。
```
Given a page with the following Content Security Policy:
Content-Security-Policy: object-src https://example.com/
Fetches for the following code will return a network errors, as the URL provided do not match object-src's source list:

<embed src="https://not-example.com/flash"></embed>
<object data="https://not-example.com/flash"></object>
<applet archive="https://not-example.com/flash"></applet>
```
有个需要注意的地方
**Note: The object-src directive acts upon any request made on behalf of an object, embed, or applet element. This includes requests which would populate the nested browsing context generated by the former two (also including navigations). This is true even when the data is semantically equivalent to content which would otherwise be restricted by another directive, such as an object element with a text/html MIME type.**


###　script-src

The script-src directive restricts the locations from which scripts may be executed. This includes not only URLs loaded directly into script elements, but also things like inline script blocks and XSLT stylesheets [XSLT] which can trigger script execution.


这个属性包括所有的script节点，而不仅仅是url形式的,还有个很重要的参数叫'unsafe-inline'
,如果加上这个参数，就不会阻止内联函数，但这被认为是不安全的。

对于这个属性有个特殊的配置叫unsafe-eval，他会禁用下面几个函数
```
eval()

Function()

setTimeout() with an initial argument which is not callable.

setInterval() with an initial argument which is not callable.
```

### style-src

The style-src directive restricts the locations from which style may be applied to a Document.

style-src属性包括下面三种引用的css属性，style也有个‘unsafe-inline’这个参数，同理会允许所有的内联css。


1、Stylesheet requests originating from a link element.

2、Stylesheet requests originating from the @import rule.

3、Stylesheet requests originating from a Link HTTP response header field [RFC5988].

### 总的来说

总的来说根据request类型的不同，会执行下面不同的步骤：

**""**

1、If the request’s initiator is "fetch", return connect-src.

2、If the request’s initiator is "manifest", return manifest-src.

3、If the request’s destination is "subresource", return connect-src.

4、If the request’s destination is "unknown", return object-src.

5、If the request’s destination is "document" and the request’s target browsing context is a nested browsing context, return child-src.

**"audio"**

**"track"**

**"video"**

   Return media-src.

**"font"**

   Return font-src.

**"image"**

   Return image-src.

**"style"**

   Return style-src.

**"script"**

1、Switch on request’s destination, and execute the associated steps:

**"subresource"**

   Return script-src.

**"serviceworker"**

**"sharedworker"**

**"worker"**

   Return child-src.

2、Return null.



基本上来说根据上面的文档，csp的意思已经能够理解了，那么怎么bypass csp呢


# Bypass CSP

关于这部分知识是从朋友的博客看到的，有个比较叼的ppt
[http://dl.lightless.me/CSP-kuza55.pptx](http://dl.lightless.me/CSP-kuza55.pptx)


基本上来说，CSP上容易存在的xss漏洞不多，除非你坚持使用‘unsafe-inline’，否则来说，xss会被大幅度的减少，而bypass CSP更多来说是不容易被csp杀掉的csrf。

## xxxx-src *

上面的那个*符号出现，表示，允许除了内联函数以外所有的url式的请求，那么bypass的方式比较简单，类似于src引用的方式，很容易造成csrf漏洞。

## xxxx-src self

一般来说，self代表只接受符合同源策略的url，这样一来，大部分的xss和crsf都会失效，有个标签比较例外，虽然已经被加入的现在的csp草案中，但是的确还没有施行。
```
<link rel="prefetch" herf="xxxxxxx">
```
根据测试来看，即使default是none，这个标签的请求都会生效，甚至不仅仅是同源下。





