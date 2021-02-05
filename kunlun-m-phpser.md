---
title: 如何自动化挖掘php反序列化链 - phpunserializechain诞生记
date: 2021-02-05 13:37:27
tags:
- 白盒
---

---

反序列化漏洞是PHP漏洞中重要的一个印象面，而反序列化漏洞的危害则需要反序列化链来界定，如何挖掘一条反序列化链，往往成为了漏洞挖掘中最浪费时间的部分。

而和挖掘漏洞一样，建立在流敏感分析基础上的自动化白盒漏洞扫描技术，依赖数据流中大量的语法节点数据，通过合理的分析手段，我们就可以回溯分析挖掘漏洞，而挖掘php反序列化链也一样，只要有合理的分析思路，那么我们就可以通过分析数据流来获得我们想要的结果。

今天我们就来一起聊聊，如何把人工审计转化成自动化挖掘方案吧~

<!--more-->

# 如何挖掘一个PHP反序列化链

反序列化漏洞的原理这里就不再赘述了，而PoP链的核心，就是魔术方法。而php的魔术方法中涉及到反序列化的大致有以下几种：
```
__destruct: 析构函数，会在到某个对象的所有引用都被删除或者当对象被显式销毁时执行。一般来说，也是Pop链的入口。
__toString: 类对象遇到字符串操作时触发。
__wakeup:   类实例反序列化时触发。
__call：    当调用了类对象中不存在或者不可访问的方法时触发。
__callStatic： 当调用了类对象中不可访问的静态方法时触发。
__get：     当获取了类对象中不可访问的属性时触发。    
__set：     当试图向类对象中不可访问的属性赋值时触发。
__invoke:   当对象调用为函数时触发
```

通俗来讲，我们可以把`__destruct`当作挖掘反序列化链的入口，因为`__wakeup`一般内容为反序列化的限制。

从`__destruct`开始，我们探讨，在不同的情况下我们分别会如何寻找调用链？

## `__call`与`__callstatic`

当代码出现
```
function __destruct(){
    $this->a();
}
```

我们可以直接跟下去看a方法的内容，如果当前类不存在a方法，则会优先查找父类的a方法。如果父类也不存在a方法（或是不可访问），那么就会触发当前类的__call魔术方法。

```
$this->a() ==> 当前类a方法 ==> 父类a方法 ==> 当前类__call方法 ==> 父类__call方法
```

值得注意的是，如果触发`__call`方法，那么a会作为`__call`的方法的第一个参数，而参数列表会作为`__call`的方法第二个参数。

而当代码出现

```
function __destruct(){
    $this->a->b();
}
```
此时，我们便不用纠结于`$this->a`是代表什么类了，我们可以调用任意类的b方法。换言之，我们也可以调用任意没有b方法的类对象的`__call`方法。

```
 $this->a->b() ==> 任意类的b方法 ==> 任意类的__call方法
```

而`__callstatic`和call大同小异，唯一的区别就是当调用静态方法时触发，例如：
```
function __destruct(){
    $this::a();
}
```

但可惜的是`__callstatic`一般来说都不会有太有价值的代码。

## `__get`与`__set`

当代码出现
```
function __destruct(){
    echo $this->a;
}
```

这时echo会优先访问当前类的a变量，然后寻找父类的a变量，如果不存在该类变量或者不可访问时，则会调用对应的__get方法

![image-20210204163257467](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/image-20210204163257467.png)

```
$this->a ==> 当前类a变量 ==> 父类a变量 ==> 当前类__get方法 ==> 父类__get方法
```

同样，如果调用`$this->a->b`，我们就有可能触发任意类的`__get`方法。

而当代码出现
```
function __destruct(){
    $this->a = 1;
}
```

如果当前类不存在a变量时，则会触发`__set`方法

![image-20210204165309768](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/image-20210204165309768.png)

## `__toString`

`__toSring`是一个很特别的魔术方法，当类对象遇到字符串操作时触发。

他一般常见于这种代码中
```
function __destruct(){
    echo $this->a;
}
```

当我们控制`$this->a`时，我们就可以触发任意类的`__toSring`方法。

![image-20210204170419508](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/image-20210204170419508.png)

```
echo $this->a ==> 任意类的__toSring
```

## 其他方法

其他方法主要包括`__wakeup`和`__invoke`，这两种方法比较特殊，在反序列化链中出现的概率比较小。

由于`__wakeup`是在反序列化时执行，所以一般来说，开发者会倾向于在wakeup函数中加入过滤部分，以减少反序列化漏洞的危害。

而`__invoke`触发条件是当尝试以调用函数的方式调用一个对象时。但可惜的是，如果你试图调用一个方法时，会优先执行`__call`逻辑，而不是invoke。

![image-20210204172009929](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/image-20210204172009929.png)

换言之，只有经过二次赋值的代码才有可能触发这个函数

![image-20210204172252298](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/image-20210204172252298.png)

在实际环境中，很难见到这样的代码出现了。

在了解了挖掘反序列化链的基础知识后，我们就把前面的思路整理整理，一起来看看怎么写一个自动化挖掘php反序列化链的小工具吧。

# 完成一个自动化挖掘php反序列化链的小工具

不知道为什么写到这里感觉有点儿像 :>

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/fcfaaf51f3deb48f9bd0e57afa1f3a292cf578dc)

到这里为止，你需要我之前的文章[构造一个 CodeDB 来探索全新的白盒静态扫描方案](https://paper.seebug.org/1387/)的一些前置知识。

现在我们手里有了一张通过格式化AST数据流生成的CodeDB表，我们的目标是完成一个能自动化挖掘php反序列化的工具。

现在我们至少拥有了这样一个思路。首先，我们需要寻找所有的destruct函数

![image-20210204182124042](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/image-20210204182124042.png)

首先，我们加入对call方法的分析

```
$this->a() ==> 当前类a方法 ==> 父类a方法 ==> 当前类__call方法 ==> 父类__call方法
```
那么流程图就变成了
![image-20210204183416313](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/image-20210204183416313.png)

紧接着我们要加入`__get`和`__set`两部分的处理。

```
$this->a ==> 当前类a变量 ==> 父类a变量 ==> 当前类__get方法 ==> 父类__get方法
```

![image-20210205105145831](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/image-20210205105145831.png)

在这个基础上，我们再加上`__toString`部分，如果出现字符串操作，那么进到tostring函数的追溯。

```
echo $this->a ==> 任意类的__toSring
```

![image-20210205105639061](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/image-20210205105639061.png)

到目前为止，整个工具的大体架构就确定下来了。为了更好的确认每种会触发魔术方法的方式。我们直接将所有的语法结构分类。

比如MethodCall（方法调用）就有可能触发`__call`，而`Assignment`(赋值操作)的左值可能触发`__set`，右值可能触发`__set`。建立在这个基础上，我们圈定了每种分类可能触发的魔术方法顺序以及范围，落成代码就成了已有的工具框架。

最后一个需要确定的问题是，如何界定是否存在危害？

这里我用了，可控+参数数量一致+敏感函数3个限制来圈定范围
```
    def check_danger_sink(self, node):
        """
        检查当前节点是否调用了危险函数并可控
        :param node:
        :return:
        """
        self.danger_function = {'call_user_func': [0],
                                'call_user_func_array': [0, 1],
                                'eval': [0],
                                'system': [0],
                                'file_put_contents': [0, 1],
                                'create_function': [0, 1],
                                }

        self.indirect_danger_function = {
                                'array_map': [0],
                                'call_user_func_array': [0],
                                }

        if node.node_type == 'FunctionCall' and node.source_node in self.danger_function:
            sink_node = eval(node.sink_node)

            if len(sink_node) >= len(self.danger_function[node.source_node]):
                # 必须有更多参数
                for i in self.danger_function[node.source_node]:
                    if self.check_param_controllable(sink_node[i], node):
                        continue

                    return False
                return True

```

只有出现对应敏感函数，且存在相应的必选参数，且相应的必选参数可控才会被认定为有危害。

到这里为止，我们已经完成了一个看上去还不错的工具雏形，接下来一起看看效果吧。

# Joomla 3.9.2反序列化利用链

这里我们拿Joomla 3.9.2做范例，目前版本的工具可以扫描到几个利用链。其中主要的一条利用链如下：

![image-20210205124653395](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/image-20210205124653395.png)

可以跟进去看看代码

![image-20210205124734611](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/image-20210205124734611.png)

有兴趣的朋友可以深入去看下这里的利用链，这是一个可以构造下去的利用链。

# 写在最后

在研究基于.QL的白盒扫描方案过程中，我遇到了很多很难解决的困难，所以就生出了写一个小工具探索一下试试看的想法。于是phpunserializechain就诞生了。在写插件的过程中也切实体会到了许多有趣的问题，也完善了一个更完整的CodeDB生成方案。践行了不少想法，具体代码可以看

[https://github.com/LoRexxar/Kunlun-M/tree/master/core/plugins/phpunserializechain](https://github.com/LoRexxar/Kunlun-M/tree/master/core/plugins/phpunserializechain)

如果感兴趣的朋友可以通过星链计划联系我，一起交流思路，最后提前祝新年快乐啦~