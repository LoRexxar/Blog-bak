title: hexo转移到coding配置
date: 2016-03-05 18:39:21
tags:
- Hexo
- Blogs
---

前两天突然国内的gitcafe被coding收购了，好像到5月底就会停止供应，那没办法，就需要移植下hexo博客了，稍微研究了下配置，更改还是比较简单的...

<!--more-->

# 注册coding

[https://coding.net](https://coding.net)

# 添加公钥

注册过github gitcafe的应该不陌生，打开~/.ssh/id_rsa.pub复制粘贴。
等一会儿就可以了。

# 修改配置

修改根目录的_config.yml
```
deploy:
  type: git
  repository: git@git.coding.net:LoRexxar/LoRexxar.git
  branch: coding-pages

```

重新添加以下就好了。

```
hexo g
hexo d
```