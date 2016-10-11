---
title: hitcon2016 misc writeup
date: 2016-10-11 12:21:14
tags:
- Blogs
- ctf
categories:
- Blogs
---

hitcon遇到了很多有趣的misc，学了很多东西，所以就专门整理一个wp吧。

<!--more-->

# Beelzemon #

这是一道ppc题目

```
Beelzemon gives you two integers 1 <= k <= n <= 20.
It wants to know if you can split a set {a | -(2**n) <= a <= (2**n) - 1} into two sets A, B s.t. |A| = |B| and sum({a**k | a in A}) = sum({b**k | b in B}).
Give Beelzemon either A or B to save your life. (separate the numbers by space)
```

简单描述下题意

大概有几点：

1、k\n在1到20之间，并且k<=n
2、a是一个从`-(2**n)`到`(2**n)`的整数集合
3、A\B中元素数量相等，并且和相等


这时候我们我们需要一些理论支持了，当天在做题的时候，我找到了这样一篇文章

[https://zhuanlan.zhihu.com/p/20559045](https://zhuanlan.zhihu.com/p/20559045)

这里有一个理论
![](/img/hitcon2016/ppc.png)

所以

![](/img/hitcon2016/ppc2.png)

但是我们又遇到了一个问题，题目中需要对包括负数的集合做处理（当时也没想明白），后看来看了wp才明白这里

[https://github.com/JulesDT/ctfWriteUps/tree/master/Hitcon%20Quals%202016/Beelzemon%20-%20PPC%20-%20150%20pts](https://github.com/JulesDT/ctfWriteUps/tree/master/Hitcon%20Quals%202016/Beelzemon%20-%20PPC%20-%20150%20pts)

贴上解题脚本
```
import socket
import re
import operator
import time

def find_partition(int_list,n,k):
    len_A=0; len_B=0; sum_A=0; sum_B=0
    Aret = ""; Bret = ""

    for i in range(0,len(int_list)):
        int_list[i] += 2**n
    int_list=int_list[::-1]
    for nb in int_list:
        if nb == 0:
            if len_A < len_B:
                len_A+=1
                Aret+= str(-2**n)+ " "
            else:
                len_B+=1
                Bret+= str(-2**n)+ " "
        else:
            if sum_A < sum_B:
               sum_A+=(nb**k)
               len_A+=1
               Aret+=str(nb-(2**n))+ " "
            else:
               sum_B+=(nb**k)
               len_B+=1
               Bret+=str(nb-(2**n))+ " "
    return (Aret)

def main():
    begin = time.time()
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect(('52.198.217.117', 6666))
    while True:
        data = s.recv(2048)
        print "Received:", data
        if len(repr(data)) <=2 :
            break;
        mgex = re.search('([0-9]+) ([0-9]+)', repr(data))
        if mgex != None:
            n = long(mgex.group(1));
            k = long(mgex.group(2));
            mySet = range(-2**n,2**n);
            partition = find_partition(mySet,n,k)
            s.send(partition+'\n');

    print "Connection closed."
    s.close()
    print "Process duration :", time.time() - begin

main()
```

这里我们看到列表通过了处理
```
int_list[i] += 2**n
```
通过这样的处理，所有本来的负数就被处理成了正数，然后再插入结果列表的时候在去掉，然后再利用刚才的理论，就可以得到结果了


# hackpad #

刚开始看到上来get 3次，然后就post暴力跑什么东西，错误的返回500，正确的返回200，那么看上去像是在跑cbc的iv了。

所以这里是**padding oracle attack**

web小白上也有提到这种攻击方式

[http://blog.zhaojie.me/2010/10/padding-oracle-attack-in-detail.html](http://blog.zhaojie.me/2010/10/padding-oracle-attack-in-detail.html)

简单来说就是这个逻辑
```
接受到正确的密文之后（填充正确且包含合法的值），应用程序正常返回（200 - OK）。
接受到非法的密文之后（解密后发现填充不正确），应用程序抛出一个解密异常（500 - Internal Server Error）。
接受到合法的密文（填充正确）但解密后得到一个非法的值，应用程序显示自定义错误消息（200 - OK）。
```

那么我们需要把每一次跑到的xor 0x01,0x02,0x03，然后异或对应密文。

```
def xor_c(s, key):
	r = ''
	for c in s:
    		r += chr(key ^ ord(c))

	return r

def xor_ss(s1, s2):
	r = ''
	for i in range(len(s1)):
    		r += chr(ord(s1[i]) ^ ord(s2[i]))

	return r

def main():
	bs = open('hackpad.pcap', 'rb').read()
    
	i = 0x2634 -1
	cs = []
	j = 0

	flag = ''
	while i!=-1:
    		i = bs.find('POST / HTTP/1.1', i+1)
    		msg = bs.find('msg=', i)
    		res = bs.find('HTTP/1.1', msg)
   	 
    		if bs[res+9] == '2':
        			iv = bs[msg+4:msg+36]
        			c = bs[msg+36:msg+68]
        			if iv[0] != '0':
            			#print iv,
            			iv = xor_c(iv.decode('hex'), 0x10)
            			cs.append(c.decode('hex'))
            			if j != 0:
                				flag += xor_ss(cs[j-1], iv)

            		j += 1
            		#print iv.encode('hex'), c
   	 
		print flag

if __name__ == "__main__":
	main()
```

ps：脚本是看wp的时候拖来的，并不是自己写的Orz


# RegExpert #

考验正则的题目，做题目的时候由于对正则实在太不熟悉了，导致第一步都没有过，所以今天仔细研究下。

## select ##

```
================= [SQL] =================
Please match string that contains "select" as a case insensitive subsequence.
```

上来第一步是select，条件是必须正则匹配到所有包含select的子字符串，在select中的任意位置都可以插入任意字符。

于是当时我的初版正则是长这样的

```
[Ss][A-Za-z]?[eE][A-Za-z]?[lL][A-Za-z]?[eE][A-Za-z]?[cC][A-Za-z]?[tT]
```
当然是有长度限制的

```
(?i)s.*e.*l.*e.*c.*t
```

## 递归正则？ ##

```
=============== [a^nb^n] ================
Yes, we know it is a classical example of context free grammer.
```

实话实说，没有特别搞明白这个题的意思，大概是说递归语法?

这里应该需要用到ruby的语法`\g<1>?`

payload:
```
^(a\g<1>?b)$
```


## 素数 ##

```
================= [x^p] =================
A prime is a natural number greater than 1 that has no positive divisors other than 1 and itself.
```

这里需要强制所有元素为x，为了避免空的正则，所以我们需要`^xx+$ `结尾

```
(?!(xx+)\1+$)^xx+$
```

## 回文？ ##

```
Both "QQ" and "TAT" are palindromes, but "PPAP" is not.
```

看上去应该同样是类似于递归的判断方式，取回文？我们需要匹配axa的模式，a为任意字符串模式

```
^(\w?|(\w)\g<1>\k<2>)$

或者

((.)(\g<1>)\2|.?)
```

## 上下文敏感语法 ##

```
============== [a^nb^nc^n] ==============
Is CFG too easy for you? How about some context SENSITIVE grammer?
```

网上能找到相应的语法
```
\A(?<AB>a\g<AB>b|){0}(?=\g<AB>c)a*(?<BC>b\g<BC>c|){1}\Z
```

可以简略到
```
^(?=(a\g<1>?b)c)a+(b\g<2>?c)$
```

