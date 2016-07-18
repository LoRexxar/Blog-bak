title: hctfgame_ll的流量分析题
date: 2016-04-06 18:35:24
tags:
- Blogs
- ctf
categories:
- Blogs
---

前两天lightless出了一道有趣的流量分析，好多学弟都难以理解，大黑客分享了他的脚本，我就分享下我的半手工脚本吧，俗话说，不管白猫黑猫，能get flag就是好payload...

<!--more-->


lightless&aklis的渗透教室 之 入侵分析	    POINT: 300 DONE 
本题题解详情
题目ID： 104
题目描述： 有一台服务器被大黑客给日了，这是在服务器上捕获的流量，你能分析出什么东西么？
附件地址：http://7ktoky.com1.z0.glb.clouddn.com/hahaha.pcapng.gz
Hint: 啧啧啧，怕你们清明节没事干。。
1. 这个数据包是在服务器上捕获的攻击流量，请仔细分析数据包中的流量，找到攻击方法。
2. 跟踪这次攻击事件，找到本次入侵的损失和危害。
3. 大黑客在攻击过程中拿到了flag，分析攻击的流量就能找到flag。

打开看了一眼是个sqlmap跑盲注的全部流量，这里老司机一看就明白了，通过分析盲注返回，可以知道sqlmap跑了什么，因为给的不是log日志，所以这里分析不是很方便，大黑客用了py的dkpt数据包分析模块，我尝试了一下发现我跑不起来，没办法，那么就粗暴点吧，首先用linux命令strings把数据包的大部分数据转为字符，出来的大概是这样的。
```
Linux 4.2.0-16-generic
Dumpcap 1.12.7 (Git Rev Unknown from unknown)
eno16777736
Linux 4.2.0-16-generic
4 ;@
( <@
GET /user.php?id=1 HTTP/1.1
Host: 192.168.197.129
Connection: keep-alive
Cache-Control: max-age=0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/49.0.2623.87 Safari/537.36
DNT: 1
Accept-Encoding: gzip, deflate, sdch
Accept-Language: en-US,en;q=0.8,zh-CN;q=0.6,zh;q=0.4,zh-TW;q=0.2
HTTP/1.1 200 OK
Server: nginx
Date: Fri, 01 Apr 2016 08:07:53 GMT
Content-Type: text/html; charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
Vary: Accept-Encoding
X-Powered-By: PHP/5.6.14
Content-Encoding: gzip
+-N-
+H,..
R042615
( >@
in-addr
arpa
in-addr
arpa
in-addr
arpa
prisoner
iana
hostmaster
root-servers
in-addr
arpa
prisoner
iana
hostmaster
root-servers
in-addr
arpa
in-addr
arpa
in-addr
arpa
arpa
in-addr
arpa
prisoner
iana
hostmaster
root-servers
arpa
ip6-servers
hostmaster
icann
in-addr
arpa
in-addr
arpa
icann
in-addr
arpa
in-addr
arpa
in-addr
arpa
in-addr
arpa
in-addr
arpa
in-addr
arpa
in-addr
arpa
in-addr
arpa
4 B@
( C@
```
可以看到返回基本都炸了，但是没关系，发现请求都是完整的，那么先把请求提出来吧。
```
import urllib

log = open('hahaha.txt','r')
log2 = open('flag.log','w')

for eachLine in log:
		if(eachLine.find('GET')!=-1):
			log2.write(erllib.unquote(eachLine))

log.close()
log2.close()

```
那么我们得到了flag.log，大概是这样的

```
GET /user.php?id=1 HTTP/1.1
GET /user.php?id=1 HTTP/1.1
HTTP/1.1 200 OK
GET /user.php?id=1 HTTP/1.1
GET /user.php?id=1 HTTP/1.1
HTTP/1.1 200 OK
GET /user.php?id=1&dKgX=6521 AND 1=1 UNION ALL SELECT 1,2,3,table_name FROM information_schema.tables WHERE 2>1-- ../../../etc/passwd HTTP/1.1
GET /user.php?id=1&dKgX=6521 AND 1=1 UNION ALL SELECT 1,2,3,table_name FROM information_schema.tables WHERE 2>1-- ../../../etc/passwd HTTP/1.1
HTTP/1.1 200 OK
GET /user.php?id=1 HTTP/1.1
GET /user.php?id=1 HTTP/1.1
HTTP/1.1 200 OK
GET /user.php?id=1484 HTTP/1.1
GET /user.php?id=1484 HTTP/1.1
HTTP/1.1 200 OK
GET /user.php?id=8492 HTTP/1.1
GET /user.php?id=8492 HTTP/1.1
HTTP/1.1 200 OK
GET /user.php?id=1,.'"(()')' HTTP/1.1
GET /user.php?id=1,.'"(()')' HTTP/1.1
HTTP/1.1 200 OK
GET /user.php?id=8788-8787 HTTP/1.1
GET /user.php?id=8788-8787 HTTP/1.1
```
这里发现前面有很多测试的payload，先把他们去掉吧，翻了翻数据包，发现开始注入的时候表名和字段名都得到了，是ctf.flag
把这部分提取出来吧...

```
log = open('flag','r')
log2 = open('flag2.log','w')

for eachLine in log:
		if(eachLine.find('ctf.flag')!=-1):
			log2.write(erllib.unquote(eachLine))

log.close()
log2.close()
```
请求都得到了，大概是这样的
```
GET /user.php?id=1 AND ORD(MID((SELECT IFNULL(CAST(flag AS CHAR),0x20) FROM ctf.flag ORDER BY flag LIMIT 0,1),1,1))>64 HTTP/1.1
GET /user.php?id=1 AND ORD(MID((SELECT IFNULL(CAST(flag AS CHAR),0x20) FROM ctf.flag ORDER BY flag LIMIT 0,1),1,1))>64 HTTP/1.1
GET /user.php?id=1 AND ORD(MID((SELECT IFNULL(CAST(flag AS CHAR),0x20) FROM ctf.flag ORDER BY flag LIMIT 0,1),1,1))>96 HTTP/1.1
GET /user.php?id=1 AND ORD(MID((SELECT IFNULL(CAST(flag AS CHAR),0x20) FROM ctf.flag ORDER BY flag LIMIT 0,1),1,1))>96 HTTP/1.1
GET /user.php?id=1 AND ORD(MID((SELECT IFNULL(CAST(flag AS CHAR),0x20) FROM ctf.flag ORDER BY flag LIMIT 0,1),1,1))>112 HTTP/1.1
GET /user.php?id=1 AND ORD(MID((SELECT IFNULL(CAST(flag AS CHAR),0x20) FROM ctf.flag ORDER BY flag LIMIT 0,1),1,1))>112 HTTP/1.1
GET /user.php?id=1 AND ORD(MID((SELECT IFNULL(CAST(flag AS CHAR),0x20) FROM ctf.flag ORDER BY flag LIMIT 0,1),1,1))>104 HTTP/1.1
GET /user.php?id=1 AND ORD(MID((SELECT IFNULL(CAST(flag AS CHAR),0x20) FROM ctf.flag ORDER BY flag LIMIT 0,1),1,1))>104 HTTP/1.1
GET /user.php?id=1 AND ORD(MID((SELECT IFNULL(CAST(flag AS CHAR),0x20) FROM ctf.flag ORDER BY flag LIMIT 0,1),1,1))>100 HTTP/1.1
GET /user.php?id=1 AND ORD(MID((SELECT IFNULL(CAST(flag AS CHAR),0x20) FROM ctf.flag ORDER BY flag LIMIT 0,1),1,1))>100 HTTP/1.1
GET /user.php?id=1 AND ORD(MID((SELECT IFNULL(CAST(flag AS CHAR),0x20) FROM ctf.flag ORDER BY flag LIMIT 0,1),1,1))>102 HTTP/1.1
GET /user.php?id=1 AND ORD(MID((SELECT IFNULL(CAST(flag AS CHAR),0x20) FROM ctf.flag ORDER BY flag LIMIT 0,1),1,1))>102 HTTP/1.1
GET /user.php?id=1 AND ORD(MID((SELECT IFNULL(CAST(flag AS CHAR),0x20) FROM ctf.flag ORDER BY flag LIMIT 0,1),1,1))>103 HTTP/1.1
GET /user.php?id=1 AND ORD(MID((SELECT IFNULL(CAST(flag AS CHAR),0x20) FROM ctf.flag ORDER BY flag LIMIT 0,1),1,1))>103 HTTP/1.1
```
请求都有了那么久写脚本吧，limit 0,1)后面的数字代表位数，后面的>xx是代表二分法确定字符是什么，这里有个问题就是二分法最后的测试payload有可能是flag的本身或者小一位，没办法，这种方式无法判断返回的数据，那么就手工吧，最后的脚本
```
import urllib

log = open('flag2.log','r')
#log2 = open('flag2.log','w')
i=1
flag = ''
f = 0

for eachLine in log:
        pos = eachLine.find('LIMIT 0,1)')
        result = urllib.unquote(eachLine[pos+11:])
        poss = result.find(',')
        index = result[:poss]
        #if(eachLine.find('ctf.flag')!=-1):
        	#print urllib.unquote(eachLine)
        if(index != i):
        	i = index
        	#print 'index:'+i
        	#print f
        	if(i=='1'):
        		flag+=''
        	elif(i=='2'):
        		flag+=chr(int(f)+1)
        	elif(i=='4'):
        		flag+=chr(int(f)+1)
        	elif(i=='5'):
        		flag+=chr(int(f)+1)
        	elif(i=='7'):
        		flag+=chr(int(f)+1)
        	elif(i=='8'):
        		flag+=chr(int(f)+1)
        	elif(i=='12'):
        		flag+=chr(int(f)+1)
        	elif(i=='23'):
        		flag+=chr(int(f)+1)
        	elif(i=='16'):
        		flag+=chr(int(f)+1)	
        	elif(i=='19'):
        		flag+=chr(int(f)+1)	
        	elif(i=='20'):
        		flag+=chr(int(f)+1)
        	elif(i=='26'):
        		flag+=chr(int(f)+1)				
        	else:
        		flag+=chr(int(f))
        

        posss = result.find('>')
        possss = result.find(' HTTP')
        index2 = result[posss+1:possss]

        f = index2
        


print flag
log.close()
#log2.close()
```

虽然中间用了半手工的模式，但是还是得到flag，能拿到flag的脚本就是好脚本(●'◡'●)