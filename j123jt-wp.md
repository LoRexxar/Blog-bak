title: j123jt的聊天板大型wp
date: 2016-04-11 13:25:39
tags:
- Blogs
- ctf
categories:
- Blogs
---
前段时间正好有时间，所以花了一些时间写了一个j123jt的聊天板，里面包含了很多种洞，算是比较像是渗透的渗透题目了，后面由于为了造洞把代码改的有些乱，所以没能按照预期写下去，还是比较可惜的，不过还是写出完整的wp，有兴趣的人可以去做做看，不过后面的xss可能懒得打开了，自己可以叉自己试试...

<!--more-->
# 目录
j123jt的聊天板
1：
login bypass
2、
弱口令
3、
业务逻辑，注册覆盖

4、忘记密码修改j123jt密码

5、xss

6、开启csp后

7、开启完整的csp，留csrf


# j123jt的聊天板alpha版	    POINT: 100
本题题解详情 
题目ID： 79
题目描述： j123jt在LoRexxar的催促下，赶工出了一个全是洞的alpha版聊天板，你能找到吗 XD！！http://115.28.78.16/hctfj1/welcome.php
Hint: 1、由于语句的问题，所以开始很多人做出来都不知道怎么回事，给个hint
2、sqli bypass login

第一个版本的洞很简单很简单了，就是所谓的万能密码登陆
主要问题的源码是这样的

```
		$user=trim($_POST['user']);
        $pass=md5(trim($_POST['pass']));
        $userid=$_COOKIE['userid']；

        $query="select password from users where username = '".$user."' and password = '".$password."'";
```
如果你看得懂php的话这里就很清晰了，username完全没有任何过滤，这里$user可以构造成`'+or+'1'='1`，合并到语句中就是
`select password from users where username = '' or '1'='1' and password = '`
这里就是构造了永真条件，所以就能登录到j123jt的账号了o(*￣▽￣*)ブ。

对了这里最短的payload应该是5位`'||1#`

# j123jt的聊天版beta1.0版	    POINT: 100
本题题解详情 
题目ID： 80
题目描述： j123jt在LoRexxar友好的（mdzz）帮助下，修改了初版的洞，但是...http://115.28.78.16/hctfj2/welcome.php
Hint: 1、如果让你写一个站，你会使用什么测试账号呢？ 
2、j123jt一般使用id作为username和password的一部分
3、为了实现洞，前面几题的环境是不同的，别指望从第一版本可以找的别的洞哦！


hint已经写成这样了，我觉得很好理解，这题就是弱口令，所谓弱口令就是一个用户的使用习惯，仔细想想其实很多人都和我有相同的弱口令习惯，所以这里的密码其实不难猜。

**username=j123jt&password=j123jt123**

# j123jt的聊天板beta1.1版	    POINT: 100
本题题解详情 
题目ID： 81
题目描述： 被批斗了的j123jt删除了测试账号后，但是beta版难道就没有洞了吗？http://115.28.78.16/hctfj3/welcome.php
Hint: 1、业务逻辑漏洞 
2、你有没有发现你可以注册很多个相同账号名字的账号呢？ 

所谓业务逻辑漏洞，就是说在书写代码的时候产生的非代码层的漏洞，而是逻辑层的判断，这里的hint也很清楚了，问题所在就是没有在注册的时候做用户唯一性检查。

主要部分源码是这样的
```
$user=$_POST['user'];
$pass=md5(trim($_POST['pass']));
$userid=$_COOKIE['userid'];

if(!get_magic_quotes_gpc()) {
        $user = addslashes($user);
        $pass = addslashes($pass);
        $userid = addslashes($userid);

    }

$query = "insert into users (username,password) values ('".$user."' , '".$pass."')";
        $result = $db->query($query);

```
我们看到并没有做什么判断，所以注册一个j123jt的账号，就可以登陆了。

# j123jt的聊天板beta1.2版	    POINT: 150
本题题解详情 
题目ID： 87
题目描述： 在LoRexxar善意的帮助下，聊天板终于可以打开了，在LoRexxar的强烈抗议下，j123jt添加了修改密码的功能，结果....http://115.28.78.16/hctfj4/welcome.php
Hint: 业务逻辑

这里就是一个挺有趣的业务逻辑漏洞了，也是以前遇到过的一种洞，仔细观察可以知道，在要求用户输入旧密码的判定成功后直接跳到了修改页面，但修改页面并没有做任何用户验证，导致可以修改任意用户的密码，这里为了降低难度，参数通过get方式传输了。

后台的主要源码是这样的
changep.php
```
$userid=$_COOKIE['userid'];
$pass=$_POST['password'];

$pass=md5($pass);
if(!get_magic_quotes_gpc()) {
        $userid = addslashes($userid);
        $pass = addslashes($pass);
}
$query="select password from users where lastcookie = '".$userid."'";
$result=$db->query($query);

$name=$result->fetch_assoc();

if($name['password']==$pass)
{
        $query="select username from users where lastcookie = '".$userid."'";
        $result=$db->query($query);
        $username=$result->fetch_assoc();
        $user=base64_encode(base64_encode(base64_encode($username['username'])));
        echo "<script>window.location.href='./verify.php?name=".$user."'</script>";
}

else
{
        echo "<script>alert('Wrong password!')</script>";
        echo "<script>window.location.href='./change.php'</script>";
}
```
这里只是做了一个简单的判断并跳转
而verify.php
```
$user=trim($_GET['name']);
if(!empty($_POST['password']))
{
        $pass=md5(trim($_POST['password']));
}
if(!get_magic_quotes_gpc()) {
        $user = addslashes($user);
}

if(!empty($pass))
{
        $userid=base64_decode(base64_decode(base64_decode($user)));
        $db = new mysqli($config['hostname'],$config['username'],$config['password'],$config['database']);
        $query = "update users set password = '".$pass."' where username = '".$userid."'";
        $db->query($query);
        echo "<script>alert('succeed!')</script>";
        echo "<script>window.location.href='./welcome.php'</script>";
}

```
可以看到从get参数中获取user，然后进行修改密码，所以跳转到这页请求j123jt的修改密码，即可修改密码成功，由于很多人会随手修改一个密码，所以在后台有一个每20秒修改一次密码的脚本，不知道有没有人因为这个原因无法登陆，如果有可以自己再尝试下。

# j123jt的聊天板beta2.0版	    POINT: 200
本题题解详情 
题目ID： 88
题目描述： 在LoRexxar的不懈努力下（mdzz），聊天版终于可以使用了，可是j123jt是个不懂xss的人....http://115.28.78.16/hctfj5/welcome.php
Hint: 1、前段时间做了一些xss弹窗的题目，可是xss仅仅是弹窗吗？
2、管理员没课才会打开看聊天板，别急哟！

到这里，前台的洞已经被修复完了，但是聊天板还没有任何过滤，所以所有的xss都可以执行，你可以使用站内互发的方式，也可以用请求自己vps的方式，两种方式都可以get 管理员cookie。

先看看源码吧。
submit.php

```
if(empty($_POST['to'])||empty($_POST['message'])){
        header('location:./user.php');
}

$user=trim($_POST['to']);
$message=trim($_POST['message']);

$user = filter($user);
$message = filter($message);

$query = "insert into m (user,message) values ('".$user."' , '".$message."')";
$result=$db->query($query);

```
里面有个比较重要的是filter，在class.php
```
function filter($string)
{
                $escape = array('\'','\\\\');
                $escape = '/' . implode('|', $escape) . '/';
                $string = preg_replace($escape, '_', $string);

                $safe = array('select', 'insert', 'update', 'delete', 'where');
                $safe = '/' . implode('|', $safe) . '/i';
                $string = preg_replace($safe, 'hacker', $string);

                $xsssafe = array('img','script','on','svg');
                $xsssafe = '/' . implode('|', $xsssafe) . '/i';
                return preg_replace($xsssafe, '', $string);


}
```
仔细看就知道标签只过滤了一次，而且没有转移单引号，xsspayload基本都是可以的，简单尝试下就知道了。

# j123jt的聊天板beta2.1版	    POINT: 200
本题题解详情 
题目ID： 92
题目描述： j123jt听说开启CSP可以有效防止xss，就从网上搜了一下，然而...http://115.28.78.16/hctfj6/welcome.php
Hint: 1、xss 
2、由于出题时候遇到一些问题，现在改为如果成功执行js，payload发到LoRexxar处获取flag

题目到这里就有一点儿难了，出题的时候正好在研究csp，所以正好出了最后两题，我们先来看看这题的csp。
有兴趣学csp可以看我的微博
[http://lorexxar.cn/2016/03/17/ccsp/](http://lorexxar.cn/2016/03/17/ccsp/)

```
Content-Security-Policy:default-src 'none'; connect-src 'self'; frame-src *; script-src http://115.28.78.16/hctfj6/js/ 'sha256-T32nlLrKkuMnyNpkJKR7kozfPzdcJi+Ql4gfcfl6PSM=';font-src http://115.28.78.16/hctfj6/fonts/ fonts.gstatic.com; style-src 'self' 'unsafe-inline'; img-src 'self'
```
仔细看可以发现里面有一条frame-src *,说明iframe没有做任何的处理，如果使用iframe，我们可以调用任意位置的js执行，只可惜我在出题的时候没考虑到iframe标签的不同源问题，导致这里其实无法盗取cookie，所以后台获取flag方式改为只要执行js即可...


# j123jt的聊天板beta2.2版	    POINT: 250
本题题解详情 
题目ID： 95
题目描述： j123jt终于好好研究了下csp并修改了到找不到bug，为了能更好的管理聊天板，j123jt给自己的账号加了添加管理员的功能，而管理员可以查看所有的message，结果...http://115.28.78.16/hctfj7/welcome.php
Hint: csrf+bypass csp


题目也是用了一个黑科技，我曾写过一篇博客
[http://lorexxar.cn/2015/11/19/xss-link/](http://lorexxar.cn/2015/11/19/xss-link/)
在关于csp的文章我也提到了这个
[http://lorexxar.cn/2016/03/17/ccsp/#more](http://lorexxar.cn/2016/03/17/ccsp/#more)

先打开看一眼csp
```
Content-Security-Policy:default-src 'none'; connect-src 'self'; frame-src 'self'; script-src http://115.28.78.16/hctfj7/js/ 'sha256-T32nlLrKkuMnyNpkJKR7kozfPzdcJi+Ql4gfcfl6PSM=';font-src http://115.28.78.16/hctfj7/fonts/ fonts.gstatic.com; style-src 'self' 'unsafe-inline'; img-src 'self'
```
好像没有什么问题，那构造一个`<link>`发现请求是有效的
源码里发现提示
```
<!--only for j123jt
	<form method="get" action="submit.php">
	<input type="text" class="form-control" name="addadmin">
	<input type="submit" value="send">
	</form>

-->
```
比较清楚了，构造
`<link rel="prefetch" herf="http://115.28.78.16/submit.php?addadmin=xxx">`
管理员打开后就会发出请求，这样就会产生csrf漏洞了



所有题目的的wp就到这里，如果有什么不懂得，可以直接私聊我（LoRexxar），这套题目本来要出很多题目，但是由于前面写的太乱，我改成mvc模式还是太乱了，所以sqli的题目没办法出下去了，我把全部源码传到coding上，有兴趣可以去自己搭着玩，源码可能有些太乱了，不必深究....

# 源码地址
[https://coding.net/u/LoRexxar/p/j123jt_ltb/git](https://coding.net/u/LoRexxar/p/j123jt_ltb/git)