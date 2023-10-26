---
title: 深入浅出Joern（四）不常用语法大全
date: 2023-10-20 15:05:47
tags:
- joern
- sast
---

在深入了解Joern的源码以及设计的时候发现Joern其实实现了很多不常用语法，很多文档中没提到的东西，其实都有比较简洁实用的方式，但也从源码的设计中发现，其实Joern的设计理念也有很多问题，这个我们以后再写到。

本篇内容可以结合https://cpg.joern.io/食用

<!--more-->

# 基础常见节点结构

## Annotation 注解

Annotation是Java中的注解节点

- astOut：通过ast边指向的节点列表
- astIn：通过ast边指向Annotation的节点列表
- argumentOut：Annotation的参数节点列表
- _annotationParameterAssignViaAstOut：Annotation的参数(ANNOTATION_PARAMETER_ASSIGN)节点列表
- __memberViaAstIn：通过ast边指向的member节点列表
- __methodViaAstIn：通过ast边指向的method节点列表
- __methodParameterInViaAstIn：通过ast边指向的method参数节点列表
- _typeDeclViaAstIn：Annotation对应的类定义节点列表

## argument 参数

- _expressionViaArgumentOut：通过ARGUMENT边指向的argument节点列表
- _astNodeViaArgumentOut：通过ARGUMENT边指向的ast节点列表
- astIn：通过AST边指向Argument的节点列表

## if/else/for/do/continue/break/throw/switch/try/while条件分支

包含已知的所有分支跳转循环节点，包括if/else/for/do/continue/break/throw/switch/try/while等...

- argumentOut：条件节点指向的argument节点列表
- astOut：通过ast边指向的节点列表
- _blockViaAstOut：通过ast边指向的block节点列表
- _callViaAstOut：通过ast边指向的call节点列表
- _controlStructureViaAstOut：通过ast边指向的条件分支节点列表
- _identifierViaAstOut：通过ast边指向的变量列表
- _returnViaAstOut：通过ast边指向的return节点列表
- cdgOut：通过cdg边指向的节点列表
- _blockViaCdgOut：通过cdg边指向的block节点列表
- _callViaCdgOut：通过cdg边指向的call节点列表
- _controlStructureViaCdgOut：通过cdg边指向的条件分支节点列表
- _returnViaCdgOut：通过cdg边指向的return节点列表
- cfgOut：通过cfg边指向的节点列表
- conditionOut：条件分支(if, else if)对应的条件节点列表
- _blockViaConditionOut：条件分支(if, else if)对应的block类型条件节点列表
- _callViaConditionOut：条件分支(if, else if)对应的call类型条件节点列表
- _controlStructureViaConditionOut: 条件分支(if, else if)对应的同样为条件分支(else if)的条件节点列表
- astIn: 通过ast边指向当前节点的节点列表
- _blockViaAstIn：通过ast边指向当前节点的block节点列表
- _callViaAstIn：通过ast边指向当前节点的call节点列表
- containsIn：包含当前节点的Method节点列表

## call 调用

- argumentOut：指向调用节点的参数节点列表
- _blockViaArgumentOut：指向调用节点的block类型参数节点列表
- _callViaArgumentOut：指向调用节点的call类型参数节点列表
- _controlStructureViaArgumentOut：指向调用节点的条件分支类型参数节点列表
- astOut：通过ast边指向的节点列表
- _blockViaAstOut：通过ast边指向的block节点列表
- _callViaAstOut：通过ast边指向的call节点列表
- _controlStructureViaAstOut：通过ast边指向的条件分支节点列表
- _identifierViaAstOut：通过ast边指向的变量列表
- _returnViaAstOut：通过ast边指向的return节点列表
- _methodViaCallOut：call调用的对应method节点
- cdgOut：通过cdg边指向的节点列表
- _blockViaCdgOut：通过cdg边指向的block节点列表
- _callViaCdgOut：通过cdg边指向的call节点列表
- _controlStructureViaCdgOut：通过cdg边指向的条件分支节点列表
- _returnViaCdgOut：通过cdg边指向的return节点列表
- cfgOut：通过cfg边指向的节点列表
- conditionOut：条件分支(if, else if)对应的条件节点列表
- _blockViaConditionOut：条件分支(if, else if)对应的block类型条件节点列表
- _callViaConditionOut：条件分支(if, else if)对应的call类型条件节点列表
- _controlStructureViaConditionOut: 条件分支(if, else if)对应的同样为条件分支(else if)的条件节点列表
- astIn: 通过ast边指向当前节点的节点列表
- _blockViaAstIn：通过ast边指向当前节点的block节点列表
- _callViaAstIn：通过ast边指向当前节点的call节点列表
- containsIn：包含当前节点的Method节点列表

## Comment 注释

- file：注释所在的file节点
- astIn：通过ast边指向当前节点的节点列表
- _fileViaAstIn：指向当前注释节点的文件节点
- sourceFileIn：通过SOURCE_FILE指向当前节点的节点列表

## File 文件

- astOut：file节点通过ast边指向的节点列表
- comment：file节点对应的comment节点
- _importViaAstOut：file节点对应的import节点
- _namespaceBlockViaAstOut：file节点对应namespace节点
- containsOut：file节点包含的节点列表
- _methodViaContainsOut：file节点包含的method节点列表
- _typeDeclViaContainsOut：file节点包含的类定义节点列表
- sourceFileIn：通过SOURCE_FILE边连接到当前节点的节点列表
- method：通过SOURCE_FILE边连接到当前节点的method节点列表
- namespaceBlock：通过SOURCE_FILE边连接到当前节点的namespaceBlock节点列表
- typeDecl：通过SOURCE_FILE边连接到当前节点的typeDecl节点列表

## method 方法

- astOut：通过ast边指向的节点列表
- _annotationViaAstOut：通过ast边指向的annotation节点列表
- _methodViaCallOut：call调用的对应method节点
- block：method节点下对应的block节点
- parameter：method节点对应参数节点列表
- _methodParameterOutViaAstOut：method对应的METHOD_PARAMETER_OUT节点
- methodReturn：method节点对应的return节点列表
- _modifierViaAstOut：method节点对应的modifier修饰词
- _typeDeclViaAstOut：method节点对应的class节点
- cfgOut：通过cfg边指向的节点列表
- cfgFirst：通过cfg边指向的第一个节点
- containsOut：method通过contains边指向的节点列表
- _blockViaContainsOut：method通过contains边指向的block节点列表
- _callViaContainsOut：method通过contains边指向的call节点列表
- _controlStructureViaContainsOut：method通过contains边指向的条件节点列表
- astIn: 通过ast边指向当前节点的节点列表
- _blockViaAstIn：通过ast边指向当前节点的block节点列表
- _namespaceBlockViaAstIn：通过ast边指向当前节点的namespaceBlock节点列表
- _typeDeclViaAstIn：通过ast边指向当前节点的类定义节点列表
- callIn：当前method节点的对应call调用节点
- cfgIn：通过cfg边指向当前节点的节点列表
- containsIn：包含当前节点的节点列表，通过contains边指向当前节点
- _fileViaContainsIn：包含当前节点的file节点
- _typeDeclViaContainsIn：包含当前节点的类定义节点

## namespace 命名空间

joern中命名空间分为namespace和namespaceBlock两个节点，file->namespaceBlock->namespace，所以这两个节点可以混在一起讲

- Namespace._namespaceBlockViaRefIn：指向namespace的namespaceBlock节点
- NamespaceBlock._namespaceViaRefOut：bock对应的namespace节点
- NamespaceBlock.astOut：namespaceBlock节点通过ast边指向的节点
- NamespaceBlock._methodViaAstOut：通过ast边指向的method节点
- NamespaceBlock._typeDeclViaAstOut：通过ast边指向的class类定义节点
- NamespaceBlock.sourceFileOut：namespace对应的file节点
- NamespaceBlock._fileViaSourceFileOut：namespace对应的file节点
- NamespaceBlock.astIn：通过ast边指向当前节点的节点列表
- NamespaceBlock._fileViaAstIn：通过ast边指向当前节点的file节点列表

## parameter 参数

parameter在joern被定义为，函数定义的参数，而不是函数调用的参数，所以这个节点几乎只和method节点链接

- astOut：通过ast边指向的节点列表
- typ：参数节点对应的类型
- parameterLinkOut：参数节点对应的返回节点列表
- method：参数节点对应的method节点

## ret 返回值

return对应method节点的返回

- argumentOut：通过argument边指向的节点列表
- astOut：通过ast边指向的节点列表
- _blockViaAstOut：通过ast边指向的block节点列表
- _callViaAstOut：通过ast边指向的call节点列表
- _controlStructureViaAstOut：通过ast边指向的条件分支节点列表
- _identifierViaAstOut：通过ast边指向的变量列表
- _returnViaAstOut：通过ast边指向的return节点列表
- _methodReturnViaCfgOut：return节点对应的method节点
- astIn: 通过ast边指向当前节点的节点列表
- _blockViaAstIn：通过ast边指向当前节点的block节点列表
- _callViaAstIn：通过ast边指向当前节点的call节点列表
- _controlStructureViaAstIn：通过ast边指向当前节点的条件分支节点列表
- cfgIn：通过cfg边指向当前节点的节点列表
- containsIn：包含当前节点的节点列表，通过contains边指向当前节点
- _methodViaContainsIn：包含当前节点的method节点
- conditionIn：和当前节点有直接关系的条件节点
- _controlStructureViaConditionIn：和当前节点有直接关系的条件节点

## TypeDecl 类定义

TypeDecl等于常规意义上class的概念

- astOut：通过ast边指向的节点列表
- _annotationViaAstOut：通过ast边指向的annotation节点列表
- _memberViaAstOut：通过ast边指向的member变量列表
- _methodViaAstOut：通过ast边指向的method节点列表
- _typeDeclViaAstOut：通过ast边指向的class类定义节点
- _methodViaContainsOut：类节点下包含的方法节点
- inheritsFromOut：继承自的class类定义节点
- sourceFileOut：当前节点所属的file节点
- _fileViaSourceFileOut：当前节点所属的file节点
- astIn: 通过ast边指向当前节点的节点列表
- _methodViaAstIn：通过ast边指向当前节点的method节点列表
- namespaceBlock：通过ast边指向当前节点的namespaceBlock节点列表
- _typeDeclViaAstIn：通过ast边指向当前节点的class类定义节点

# 特殊语法关系

## method方法特殊语法

```java
@GetMapping("/codeinject")
public String codeInject(String filepath) throws IOException {

    String[] cmdList = new String[]{"sh", "-c", "ls -la " + filepath};
    ProcessBuilder builder = new ProcessBuilder(cmdList);
    builder.redirectErrorStream(true);
    Process process = builder.start();
    return WebUtils.convertStreamToString(process.getInputStream());
}
```

- caller

当前节点的调用位置方法，比如`cpg.method("start").caller`就会返回codeInject方法

- callIn

当前方法/函数节点的调用位置，比如`cpg.method("start").callIn`就会返回调用start方法的call节点

- cpg.method("start")

名为start的方法的定义节点

## 读取向外的所有边

- cpg.method.head.outE.map(n=>n.label).l

获取当前节点向外所有的边，并展示边类型

![image-20231023182018337](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202310231821493.png)

- cpg.method.head.outE.map(n=>n.inNode).l

获取当前节点向外所有的边，并展示连接到的节点

![image-20231023182055115](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202310231821093.png)
