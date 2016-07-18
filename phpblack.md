title: is_numeric和trim导致的判断绕过
date: 2016-04-04 15:47:18
tags:
- Blogs
- php
categories:
- Blogs
---
前两天做了一道前段时间的三个白帽，遇到了一个有趣的php黑魔法...

<!--more-->

题目的writeup是从这里看到的
[http://drops.wooyun.org/tips/10564](http://drops.wooyun.org/tips/10564)

题目的源码首先是这样的
```
<?php
if(isset($_GET['source'])){
    highlight_file(__FILE__);
    exit;
}
include_once("flag.php");
 /*
    shougong check if the $number is a palindrome number(hui wen shu)
 */
function is_palindrome_number($number) {
    $number = strval($number);
    $i = 0;
    $j = strlen($number) - 1;
    while($i < $j) {
        if($number[$i] !== $number[$j]) {
            return false;
        }
        $i++;
        $j--;
    }
    return true;
}
ini_set("display_error", false);
error_reporting(0);
$info = "";
$req = [];
foreach([$_GET, $_POST] as $global_var) {
    foreach($global_var as $key => $value) {
        $value = trim($value);
        is_string($value) && is_numeric($value) && $req[$key] = addslashes($value);
    }
}    
 
$n1 = intval($req["number"]);
$n2 = intval(strrev($req["number"]));
if($n1 && $n2) {
    if ($req["number"] != intval($req["number"])) {
        $info = "number must be integer!";
    } elseif ($req["number"][0] == "+" || $req["number"][0] == "-") {
        $info = "no symbol";
    } elseif ($n1 != $n2) { //first check
        $info = "no, this is not a palindrome number!";
    } else { //second check
        if(is_palindrome_number($req["number"])) {
            $info = "nice! {$n1} is a palindrome number!";
        } else {
            if(strpos($req["number"], ".") === false && $n1 < 2147483646) {
                $info = "find another strange dongxi: " . FLAG2;
            } else {
                $info = "find a strange dongxi: " . FLAG;
            }
        }
    }
} else {
    $info = "no number input~";
}
?>
```
简单分析下逻辑，flag需要满足三个条件
1、number = intval(number)
2、intval(number) = intval(strrev(number))
3、not a palindorme number

还有一个很重要的判断是区别于falg1和2的
**if(strpos($req["number"], ".") === false && $n1 < 2147483646)**
这一句让我知道flag1的做法很清楚了就是上溢或者下溢，一种是2147483647,还有一种是number=2147483647.00000000001，这样在intval下，0.0000000000001会变成2147483647,满足条件。

有些系统下可能用了64位，那么溢出的数字要是9223372036854775807，这种情况下的payload是:number=09223372036854775807.


当然上面这种不是主要目的，问题在flag2，这里禁止使用.且数字必须小于2147483646，那么就不能使用溢出的方式了。

让我们来看看is_numeric的源码。
![](/img/php/1.jpg)
从画框的地方，我们可以看到，在is_numeric开始判断之前，首先要跳过所有的空白字符，也就是说即使前面我们传入一些空格什么的也是可以过判断的。

但是我们会发现前面不是有trim吗，这里我们看看trim的源码
![](/img/php/2.jpg)
我们发现过滤的空白字符少了一个\f，那么就很清楚了，我们可以用%0c过这里的判断了

`number=%0c121`


