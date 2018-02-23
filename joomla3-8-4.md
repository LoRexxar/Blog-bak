---
title: 从补丁到漏洞分析 --记一次joomla漏洞应急
date: 2018-02-07 11:12:04
tags:
- sqli
- xss
- 漏洞分析
---


2018年1月30日，joomla更新了3.8.4版本，这次更新修复了4个安全漏洞，以及上百个bug修复。

[https://www.joomla.org/announcements/release-news/5723-joomla-3-8-4-release.html](https://www.joomla.org/announcements/release-news/5723-joomla-3-8-4-release.html)

为了漏洞应急这几个漏洞，我花费了大量的时间分析漏洞成因、寻找漏洞触发位置、回溯逻辑，下面的文章比起漏洞分析来说，更接近我思考的思路，希望能给大家带来不一样的东西。

# 背景 #

其中的4个安全漏洞包括
```
- Low Priority - Core - XSS vulnerability in module chromes (affecting Joomla 3.0.0 through 3.8.3) 
- Low Priority - Core - XSS vulnerability in com_fields (affecting Joomla 3.7.0 through 3.8.3) 
- Low Priority - Core - XSS vulnerability in Uri class (affecting Joomla 1.5.0 through 3.8.3) 
- Low Priority - Core - SQLi vulnerability in Hathor postinstall message (affecting Joomla 3.7.0 through 3.8.3)
```

根据更新，我们去到github上的joomla项目，从中寻找相应的修复补丁，可以发现，4个安全漏洞的是和3.8.4的release版同时更新的。

[https://github.com/joomla/joomla-cms/commit/0ec372fdc6ad5ad63082636a0942b3ea39acc7b7](https://github.com/joomla/joomla-cms/commit/0ec372fdc6ad5ad63082636a0942b3ea39acc7b7)

通过补丁配合漏洞详情中的简单描述我们可以确定漏洞的一部信息，紧接着通过这部分信息来回溯漏洞成因。

# SQLi vulnerability in Hathor postinstall message

[https://developer.joomla.org/security-centre/722-20180104-core-sqli-vulnerability.html](https://developer.joomla.org/security-centre/722-20180104-core-sqli-vulnerability.html)

```
Description
The lack of type casting of a variable in SQL statement leads to a SQL injection vulnerability in the Hathor postinstall message.

Affected Installs
Joomla! CMS versions 3.7.0 through 3.8.3
```

## 补丁分析

第一个漏洞说的比较明白，是说在Hathor的postinstall信息处，由于错误的类型转换导致了注入漏洞。

我们来看看相应的补丁

![image.png-107.3kB][1]

符合漏洞描述的点就是这里，原来的取第一位改为了对取出信息做强制类型转换，然后拼接入sql语句。

这里假设我们可以控制`$adminstyle`，如果我们通过传入数组的方式设置该变量为数组格式，并且第1个字符串可控，那么这里就是一个可以成立的漏洞点。

现在我们需要找到这个功能的位置，并且回溯变量判断是否可控。

## 找到漏洞位置

hathor是joomla自带的两个后台模板之一，由于hathor更新迭代没有isis快，部分功能会缺失，所以在安装完成之后，joomla的模板为isis，我们需要手动设置该部分。

```
Templates->styles->adminnistrator->hathor
```
![image.png-51.3kB][2]

修改完成后回到首页，右边就是postinstallation message

![image.png-81.9kB][3]

## 回溯漏洞

回到代码中，我们需要找到`$adminstyle`这个变量进入的地方。

```
$adminstyle = $user->getParam('admin_style', ''); 
```

这里user为`JFactory::getUser()`，跟入getParam方法

```
/libraries/src/User/User.php line 318

public function getParam($key, $default = null)
{
	return $this->_params->get($key, $default);
}
```

这里`$this->_params`来自`$this->_params = new Registry;`

跟入Registry的get方法

```
libraries/vendor/joomla/registry/src/Registry.php line 201

public function get($path, $default = null)
	{
		// Return default value if path is empty
		if (empty($path))
		{
			return $default;
		}

		if (!strpos($path, $this->separator))
		{
			return (isset($this->data->$path) && $this->data->$path !== null && $this->data->$path !== '') ? $this->data->$path : $default;
		}

		// Explode the registry path into an array
		$nodes = explode($this->separator, trim($path));

		// Initialize the current node to be the registry root.
		$node = $this->data;
		$found = false;

		// Traverse the registry to find the correct node for the result.
		foreach ($nodes as $n)
		{
			if (is_array($node) && isset($node[$n]))
			{
				$node = $node[$n];
				$found = true;

				continue;
			}

			if (!isset($node->$n))
			{
				return $default;
			}

			$node = $node->$n;
			$found = true;
		}

		if (!$found || $node === null || $node === '')
		{
			return $default;
		}

		return $node;
	}
```

根据这里的调用方式来看，这里会通过这里的的判断获取是否存在adminstyle，如果没有则会返回default(这里为空)

![image.png-127kB][4]

接着回溯`$this->data`,data来自`$this->data = new \stdClass;`


回溯到这里可以发现`$admin_style`的地方是从全局变量中中读取的。

默认设置为空`/administrator/components/com_users/models/forms/user.xml`

![image.png-86.2kB][5]

但我们是可以设置这个的

后台`users->users->super user`设置，右边我们可以设置当前账户使用的后台模板，将右边修改为使用hathor型模板。

通过抓包我们可以发现，这里显式的设置了当前账户的`admin_type`，这样如果我们通过传入数组，就可以设置`admin_type`为任意值

![image.png-121.5kB][6]

然后进入代码中的数据库操作
`/administrator/templates/hathor/postinstall/hathormessage.php function hathormessage_postinstall_condition`

![image.png-131.4kB][7]

访问post_install页面触发

![image.png-20kB][8]


# XSS vulnerability in com_fields

[https://developer.joomla.org/security-centre/720-20180102-core-xss-vulnerability.html](https://developer.joomla.org/security-centre/720-20180102-core-xss-vulnerability.html)

```
Description
Inadequate input filtering in com_fields leads to a XSS vulnerability in multiple field types, i.e. list, radio and checkbox.

Affected Installs
Joomla! CMS versions 3.7.0 through 3.8.3
```

## 补丁分析

![image.png-40.5kB][9]

漏洞详情写了很多，反而是补丁比较模糊，我们可以大胆猜测下，当插入的字段类型为list、radio、checkbox多出的部分变量没有经过转义

首先我们需要先找到触发点

后台`content->fields->new`，然后设置type为`radio`，在键名处加入相应的payload

![image.png-40.4kB][10]

然后保存新建文章

![EH@9IYLOQ8LNR_XN{P`A8DP.png-73.3kB][11]

成功触发

## 漏洞分析

由于补丁修复的方式比较特殊，可以猜测是在某些部分调用时使用了textContent而不是nodeValue，在分析变量时以此为重点。

漏洞的出发点`/administrator/components/com_fields/libraries/fieldslistplugin.php line 31`

![image.png-220.2kB][12]

由于找不到该方法的调用点，所以我们从触发漏洞的点分析流程。

编辑文章的上边栏是通过`administrator/components/com_content/views/article/tmp/edit.php line 99`载入的

![image.png-127.2kB][13]

这里`JLayoutHelper:render`会进入`/layouts/joomla/edit/params.php`

然后在129行进入`JLayoutHelper::render('joomla.edit.fieldset', $displayData);`

![image.png-111.5kB][14]

跟入`/layouts/joomla/edit/fieldset.php line 16`，代码在这里通过执行`form`的`getFieldset`获取了提交的自定义字段信息。

![image.png-60.2kB][15]

跟入`/libraries/src/Form/Form.php line 329 function getFieldset`

![image.png-118.8kB][16]

跟如1683行 `findFieldsByFieldset`函数。

![image.png-242.5kB][17]

这里调用xml来获取数据，从全局的xml变量中匹配。

这里的全局变量中的xml中的option字段就来自于设置时的`$option->textContent`，而只有` list, radio and checkbox.`这三种是通过这里的函数做处理，其中list比较特殊，在后面的处理过程中，list类型的自定义字段会在`/libraries/cms/html/select.php line 742 function  options`被二次处理，但radio不会，所以漏洞存在。

![image.png-82.4kB][18]

整个xss漏洞从插入到触发限制都比较大，实战价值较低。


# XSS vulnerability in Uri class

[https://developer.joomla.org/security-centre/721-20180103-core-xss-vulnerability.html](https://developer.joomla.org/security-centre/721-20180103-core-xss-vulnerability.html)

```
Description
Inadequate input filtering in the Uri class (formerly JUri) leads to a XSS vulnerability.

Affected Installs
Joomla! CMS versions 1.5.0 through 3.8.3
```

## 补丁分析

比起其他几个来说，这里的漏洞就属于特别清晰的，就是在获取系统变量时，没做相应的过滤。

![image.png-33.1kB][19]

前台触发方式特别简单，因为这里的`script_name`是获取基础url路径的，会拼接进所有页面的和链接有关系的地方，包括js或者css的引入。

## 漏洞利用

让我们来看看完整的代码

```
if (strpos(php_sapi_name(), 'cgi') !== false && !ini_get('cgi.fix_pathinfo') && !empty($_SERVER['REQUEST_URI']))
{
	// PHP-CGI on Apache with "cgi.fix_pathinfo = 0"

	// We shouldn't have user-supplied PATH_INFO in PHP_SELF in this case
	// because PHP will not work with PATH_INFO at all.
	$script_name = $_SERVER['PHP_SELF'];
}
else
{
	// Others
	$script_name = $_SERVER['SCRIPT_NAME'];
}

static::$base['path'] = rtrim(dirname($script_name), '/\\');
```

很明显只有当`$script_name = $_SERVER['PHP_SELF']`的时候，漏洞才有可能成立

只有当**php是fastcgi运行，而且cgi.fix_pathinfo = 0**时才能进入这个判断，然后利用漏洞还有一个条件，就是服务端对路径的解析存在问题才行。

```
http://127.0.0.1/index.php/{evil_code}/321321

--->

http://127.0.0.1/index.php
```

当该路径能被正常解析时，`http://127.0.0.1/index.php/{evil_code}`就会被错误的设置为基础URL拼接入页面中。

![image.png-56kB][20]

一个无限制的xss就成立了

# XSS vulnerability in module chromes

[https://developer.joomla.org/security-centre/718-20180101-core-xss-vulnerability.html](https://developer.joomla.org/security-centre/718-20180101-core-xss-vulnerability.html)

```
Description
Lack of escaping in the module chromes leads to XSS vulnerabilities in the module system.

Affected Installs
Joomla! CMS versions 3.0.0 through 3.8.3
```

## 补丁分析

![image.png-57kB][21]

漏洞存在的点比较清楚，修复中将`$moduleTag`进行了一次转义，同样的地方有三处，但都是同一个变量导致的。

这个触发也比较简单，当我们把前台模板设置为protostar（默认）时，访问前台就会触发这里的`modChrome_well`函数。

## 漏洞利用

让我们看看完整的代码

![image.png-280.4kB][22]

很明显后面`module_tag`没有经过更多处理，就输出了，假设我们可控`module_tag`，那么漏洞就成立。

问题在于怎么控制，这里的函数找不到调用的地方，能触发的地方都返回了传入的第二个值，猜测和上面的`get_param`一样，如果没有设置该变量，则返回`default`值。

经过一番研究，并没有找到可控的设置的点，这里只能暂时放弃。


# ref

- Joomla 3.8.4
[https://www.joomla.org/announcements/release-news/5723-joomla-3-8-4-release.html](https://www.joomla.org/announcements/release-news/5723-joomla-3-8-4-release.html)
- Joomla security patches
[https://developer.joomla.org/security-centre/](https://developer.joomla.org/security-centre/)



  [1]: http://static.zybuluo.com/LoRexxar/eo8ih0686tme2c9i1vnn7so9/image.png
  [2]: http://static.zybuluo.com/LoRexxar/q7f7zy4z7nunlxzvev35oqvs/image.png
  [3]: http://static.zybuluo.com/LoRexxar/7nmou7dh3uivdg645hattsgi/image.png
  [4]: http://static.zybuluo.com/LoRexxar/40o6vrf2b4v1me2sch4b2m90/image.png
  [5]: http://static.zybuluo.com/LoRexxar/8yq9o0c5v997525ysu1mblb8/image.png
  [6]: http://static.zybuluo.com/LoRexxar/2vaampsksds8za4utzxoxkw3/image.png
  [7]: http://static.zybuluo.com/LoRexxar/b05913bekg7574m69o3y9jdt/image.png
  [8]: http://static.zybuluo.com/LoRexxar/vclviz1fgwz2p6cchmfo9rih/image.png
  [9]: http://static.zybuluo.com/LoRexxar/dkvxqxctwwb0stwo1n2790h5/image.png
  [10]: http://static.zybuluo.com/LoRexxar/dmi9fce5zw3solpqth24t3u1/image.png
  [11]: http://static.zybuluo.com/LoRexxar/l5ucf3zr5hyjx0c1ezfthzec/EH@9IYLOQ8LNR_XN%7BP%60A8DP.png
  [12]: http://static.zybuluo.com/LoRexxar/43nr2wr5rxi2q1iq4yfug9lx/image.png
  [13]: http://static.zybuluo.com/LoRexxar/fufwrgbek6l4ptax45axxd6s/image.png
  [14]: http://static.zybuluo.com/LoRexxar/ujdc5omxwakv59xvkkjuvyqm/image.png
  [15]: http://static.zybuluo.com/LoRexxar/j8aq2nlhdkzhc744kud40jur/image.png
  [16]: http://static.zybuluo.com/LoRexxar/zx2amhrhlkmoh5lj82prkj6l/image.png
  [17]: http://static.zybuluo.com/LoRexxar/7x3h6n5xr060jry19sk22i20/image.png
  [18]: http://static.zybuluo.com/LoRexxar/h7skxub2nvrmtinmpvmrx9aa/image.png
  [19]: http://static.zybuluo.com/LoRexxar/gcq2hg7mbqlvibo39afvu2o5/image.png
  [20]: http://static.zybuluo.com/LoRexxar/mnmsg7fhb6hwjm56r65ed5jq/image.png
  [21]: http://static.zybuluo.com/LoRexxar/zw2ohjczh3fec8sru3c745a9/image.png
  [22]: http://static.zybuluo.com/LoRexxar/vzrrwbjibfek4hccomqoyq8n/image.png
