---
title: sqlmap 源码分析（一）开始、参数解析
date: 2016-08-09 18:24:22
tags:
- python
- sqlmap
categories:
- Blogs
---

sqlmap是web狗永远也绕不过去的神器，为了能自由的使用sqlmap，阅读源码还是有必要的...

<!--more-->

# 基本配置 #

sqlmap在启动前，首先是基本的配置

```sqlmap.py(64~66)
paths.SQLMAP_ROOT_PATH = modulePath()
setPaths()
banner()
```
设置了和路径有关的配置参数，banner则是输出了sqlmap的信息
```
sqlmap/0.9 - automatic SQL injection and database takeover tool
http://sqlmap.sourceforge.net
```

# 参数解析 #

紧接着是对参数的解析
```sqlmap.py(69)
cmdLineOptions.update(cmdLineParser().__dict__)
```
跟着cmdLineParser我们进入了
```
from lib.parse.cmdline import cmdLineParser
```
/lib/parse/cmdline

## 必选参数 ##
```
# Target options
	target = OptionGroup(parser, "Target", "At least one of these "
	                     "options has to be specified to set the source "
	                     "to get target urls from.")
	
	target.add_option("-d", dest="direct", help="Direct "
	                  "connection to the database")
	
	target.add_option("-u", "--url", dest="url", help="Target url")
	
	target.add_option("-l", dest="list", help="Parse targets from Burp "
	                  "or WebScarab proxy logs")
	
	target.add_option("-r", dest="requestFile",
	                  help="Load HTTP request from a file")
	
	target.add_option("-g", dest="googleDork",
	                  help="Process Google dork results as target urls")
	
	target.add_option("-c", dest="configFile",
	                          help="Load options from a configuration INI file")

```
从--help上看到是这样的
```
Target:
	At least one of these options has to be specified to set the source to
	get target urls from.
	
	-d DIRECT           Direct connection to the database
	-u URL, --url=URL   Target url
	-l LIST             Parse targets from Burp or WebScarab proxy logs
	-r REQUESTFILE      Load HTTP request from a file
	-g GOOGLEDORK       Process Google dork results as target urls
	-c CONFIGFILE       Load options from a configuration INI file
```

- -d direct 直连数据库的方式
- -u url 目标url
- -l list 从Burp or WebScarab获取的代理log
- -r 加载从文件获取的http request
- -g 目标url在Process Google dork的结果
- -c 加载ini文件设置

## request参数 ##
```
# Request options
	request = OptionGroup(parser, "Request", "These options can be used "
	                      "to specify how to connect to the target url.")
	
	request.add_option("--data", dest="data",
	                   help="Data string to be sent through POST")
	
	request.add_option("--cookie", dest="cookie",
	                   help="HTTP Cookie header")
	
	request.add_option("--cookie-urlencode", dest="cookieUrlencode",
	                     action="store_true", default=False,
	                     help="URL Encode generated cookie injections")
	
	request.add_option("--drop-set-cookie", dest="dropSetCookie",
	                   action="store_true", default=False,
	                   help="Ignore Set-Cookie header from response")
	
	request.add_option("--user-agent", dest="agent",
	                   help="HTTP User-Agent header")
	
	request.add_option("--random-agent", dest="randomAgent",
	                   action="store_true", default=False,
	                   help="Use randomly selected HTTP User-Agent header")
	
	request.add_option("--referer", dest="referer",
	                   help="HTTP Referer header")
	
	request.add_option("--headers", dest="headers",
	                   help="Extra HTTP headers newline separated")
	
	request.add_option("--auth-type", dest="aType",
	                   help="HTTP authentication type "
	                        "(Basic, Digest or NTLM)")
	
	request.add_option("--auth-cred", dest="aCred",
	                   help="HTTP authentication credentials "
	                        "(name:password)")
	
	request.add_option("--auth-cert", dest="aCert",
	                   help="HTTP authentication certificate ("
	                        "key_file,cert_file)")
	
	request.add_option("--proxy", dest="proxy",
	                   help="Use a HTTP proxy to connect to the target url")
	
	request.add_option("--proxy-cred", dest="pCred",
	                   help="HTTP proxy authentication credentials "
	                        "(name:password)")
	
	request.add_option("--ignore-proxy", dest="ignoreProxy", action="store_true",
	                   default=False, help="Ignore system default HTTP proxy")
	
	request.add_option("--delay", dest="delay", type="float", default=0,
	                   help="Delay in seconds between each HTTP request")
	
	request.add_option("--timeout", dest="timeout", type="float", default=30,
	                   help="Seconds to wait before timeout connection "
	                        "(default 30)")
	
	request.add_option("--retries", dest="retries", type="int", default=3,
	                   help="Retries when the connection timeouts "
	                        "(default 3)")
	
	request.add_option("--scope", dest="scope", 
	                   help="Regexp to filter targets from provided proxy log")
	
	request.add_option("--safe-url", dest="safUrl", 
	                   help="Url address to visit frequently during testing")
	
	request.add_option("--safe-freq", dest="saFreq", type="int", default=0,
	                   help="Test requests between two visits to a given safe url")
```
从--help中看是这样的
```
Request:
    These options can be used to specify how to connect to the target url.

    --data=DATA         Data string to be sent through POST
    --cookie=COOKIE     HTTP Cookie header
    --cookie-urlencode  URL Encode generated cookie injections
    --drop-set-cookie   Ignore Set-Cookie header from response
    --user-agent=AGENT  HTTP User-Agent header
    --random-agent      Use randomly selected HTTP User-Agent header
    --referer=REFERER   HTTP Referer header
    --headers=HEADERS   Extra HTTP headers newline separated
    --auth-type=ATYPE   HTTP authentication type (Basic, Digest or NTLM)
    --auth-cred=ACRED   HTTP authentication credentials (name:password)
    --auth-cert=ACERT   HTTP authentication certificate (key_file,cert_file)
    --proxy=PROXY       Use a HTTP proxy to connect to the target url
    --proxy-cred=PCRED  HTTP proxy authentication credentials (name:password)
    --ignore-proxy      Ignore system default HTTP proxy
    --delay=DELAY       Delay in seconds between each HTTP request
    --timeout=TIMEOUT   Seconds to wait before timeout connection (default 30)
    --retries=RETRIES   Retries when the connection timeouts (default 3)
    --scope=SCOPE       Regexp to filter targets from provided proxy log
    --safe-url=SAFURL   Url address to visit frequently during testing
    --safe-freq=SAFREQ  Test requests between two visits to a given safe url
```

- --data 很好理解，就是post的时候的数据
- --cookie 请求的时候所带的cookie
- --cookie-urlencode url编码的cookie注入
- --drop-set-cookie 忽略返回头中的set-cookie
- --user-agent=AGENT  http头中，关于ua的设置
- --random-agent 更简单的方式，随机使用ua
- --referer=REFERER http头中，关于referer的设置
- --headers=HEADERS 额外的http请求头，要求换行符分割
- --auth-type=ATYPE http authentication的类型(Basic, Digest 和 NTLM)
- --auth-cred=ACRED 如果是账号密码的形式（name:password）
- --auth-cert=ACERT 如果是公私钥的方式，则要求(key_file,cert_file)
- --proxy=PROXY 使用http的代理来连接目标url
- --proxy-cred=PCRED 如果是待用户验证的代理，你需要加上账号密码（name:password）
- --ignore-proxy 忽略系统默认的http代理
- --delay=DELAY 每次请求的延迟，单位为秒
- --timeout=TIMEOUT 请求超时的判定，默认为30
- --retries=RETRIES 请求超时的重试次数，默认为3次
- --scope=SCOPE 从提供的代理日志过滤目标
- --safe-url=SAFURL 经常访问用来测试的url地址
- --safe-freq=SAFREQ 2个访问者的安全链接

## 性能参数 ##
```
 # Optimization options
    optimization = OptionGroup(parser, "Optimization", "These "
                           "options can be used to optimize the "
                           "performance of sqlmap.")

    optimization.add_option("-o", dest="optimize",
                             action="store_true", default=False,
                             help="Turn on all optimization switches")

    optimization.add_option("--predict-output", dest="predictOutput", action="store_true",
                      default=False, help="Predict common queries output")

    optimization.add_option("--keep-alive", dest="keepAlive", action="store_true",
                       default=False, help="Use persistent HTTP(s) connections")

    optimization.add_option("--null-connection", dest="nullConnection", action="store_true",
                      default=False, help="Retrieve page length without actual HTTP response body")

    optimization.add_option("--threads", dest="threads", type="int", default=1,
                       help="Max number of concurrent HTTP(s) "
                            "requests (default 1)")
```
从--help中看是这样的
```
Optimization:
	These options can be used to optimize the performance of sqlmap.
	
	-o                  Turn on all optimization switches
	--predict-output    Predict common queries output
	--keep-alive        Use persistent HTTP(s) connections
	--null-connection   Retrieve page length without actual HTTP response body
	--threads=THREADS   Max number of concurrent HTTP(s) requests (default 1)
```
- -o 打开所有优化开关
- --predict-output 预测常用的查询输出
- --keep-alive 使用一个持续的http连接
- --null-connection 检索页面长度没有实际的HTTP响应页面
- --threads=THREADS 最大的http连接线程数


## 注入参数 ##

```
# Injection options
    injection = OptionGroup(parser, "Injection", "These options can be "
                            "used to specify which parameters to test "
                            "for, provide custom injection payloads and "
                            "optional tampering scripts.")

    injection.add_option("-p", dest="testParameter",
                         help="Testable parameter(s)")

    injection.add_option("--dbms", dest="dbms",
                         help="Force back-end DBMS to this value")

    injection.add_option("--os", dest="os",
                         help="Force back-end DBMS operating system "
                              "to this value")

    injection.add_option("--prefix", dest="prefix",
                         help="Injection payload prefix string")

    injection.add_option("--suffix", dest="suffix",
                         help="Injection payload suffix string")

    injection.add_option("--tamper", dest="tamper",
                         help="Use given script(s) for tampering injection data")
```
从--help中看是这样的
```
 Injection:
    These options can be used to specify which parameters to test for,
    provide custom injection payloads and optional tampering scripts.

    -p TESTPARAMETER    Testable parameter(s)
    --dbms=DBMS         Force back-end DBMS to this value
    --os=OS             Force back-end DBMS operating system to this value
    --prefix=PREFIX     Injection payload prefix string
    --suffix=SUFFIX     Injection payload suffix string
    --tamper=TAMPER     Use given script(s) for tampering injection data
```
这个参数是用来配置自定义的注入payload和可以篡改的脚本

- -p TESTPARAMETER 可测试的参数
- --dbms=DBMS 后端数据库的的值（？）
- --os=OS 后端数据库的操作系统
- --prefix=PREFIX 注入payload的前缀字符串
- --suffix=SUFFIX 注入payload的后缀字符串
- --tamper=TAMPER 使用给定的脚本篡改注入数据

## 盲注参数 ##
```
# Detection options
    detection = OptionGroup(parser, "Detection", "These options can be "
                            "used to specify how to parse "
                            "and compare page content from "
                            "HTTP responses when using blind SQL "
                            "injection technique.")

    detection.add_option("--level", dest="level", default=1, type="int",
                         help="Level of tests to perform (1-5, "
                              "default 1)")

    detection.add_option("--risk", dest="risk", default=1, type="int",
                         help="Risk of tests to perform (0-3, "
                              "default 1)")

    detection.add_option("--string", dest="string",
                         help="String to match in page when the "
                              "query is valid")

    detection.add_option("--regexp", dest="regexp",
                         help="Regexp to match in page when the "
                              "query is valid")

    detection.add_option("--text-only", dest="textOnly",
                         action="store_true", default=False,
                         help="Compare pages based only on the textual content")
```

从--help中看，是这样的
```
 Detection:
    These options can be used to specify how to parse and compare page
    content from HTTP responses when using blind SQL injection technique.

    --level=LEVEL       Level of tests to perform (1-5, default 1)
    --risk=RISK         Risk of tests to perform (0-3, default 1)
    --string=STRING     String to match in page when the query is valid
    --regexp=REGEXP     Regexp to match in page when the query is valid
    --text-only         Compare pages based only on the textual content
```

这部分的参数是在盲注的方式下，指定如何解析页面和比较页面的

- --level 要进行测试的等级，默认为1
- --risk 要进行测试的风险级别，默认为1
- --string 字符串匹配时查询是有效的
- --regexp 正则匹配时查询是有效的
- --text-only 页面只基于文本进行比较

## 技术参数 ##
```
# Techniques options
    techniques = OptionGroup(parser, "Techniques", "These options can be "
                             "used to tweak testing of specific SQL "
                             "injection techniques.")

    techniques.add_option("--technique", dest="tech", default="BEUST",
                          help="SQL injection techniques to test for "
                               "(default BEUST)")

    techniques.add_option("--time-sec", dest="timeSec",
                          type="int", default=TIME_DEFAULT_DELAY,
                          help="Seconds to delay the DBMS response "
                               "(default 5)")

    techniques.add_option("--union-cols", dest="uCols",
                          help="Range of columns to test for UNION query SQL injection")

    techniques.add_option("--union-char", dest="uChar",
                          help="Character to use for bruteforcing number of columns")
```

在--help中
```
 Techniques:
    These options can be used to tweak testing of specific SQL injection
    techniques.

    --technique=TECH    SQL injection techniques to test for (default BEUST)
    --time-sec=TIMESEC  Seconds to delay the DBMS response (default 5)
    --union-cols=UCOLS  Range of columns to test for UNION query SQL injection
    --union-char=UCHAR  Character to use for bruteforcing number of columns
```

这部分参数负责调整注入测试时的技术

- --technique=TECH sql注入技术，默认为beust
- --time-sec=TIMESEC 数据库返回延迟，默认为5（应该是时间盲注时候的配置）
- --union-cols=UCOLS 联合查询是的列数
- --union-char=UCHAR 字符用于爆破的列数（union注入的列数测试？）


## 指纹选项 ##

```
# Fingerprint options
    fingerprint = OptionGroup(parser, "Fingerprint")

    fingerprint.add_option("-f", "--fingerprint", dest="extensiveFp",
                           action="store_true", default=False,
                           help="Perform an extensive DBMS version fingerprint")
```

```
  Fingerprint:
    -f, --fingerprint   Perform an extensive DBMS version fingerprint
```

这个参数只有一个选项，是指明数据库版本的指纹（还是bool型...没懂）

## 枚举选项 ##

应该说也叫目标选项，是一些关于注入的选项，比较核心
```
# Enumeration options
    enumeration = OptionGroup(parser, "Enumeration", "These options can "
                              "be used to enumerate the back-end database "
                              "management system information, structure "
                              "and data contained in the tables. Moreover "
                              "you can run your own SQL statements.")

    enumeration.add_option("-b", "--banner", dest="getBanner",
                           action="store_true", default=False, help="Retrieve DBMS banner")

    enumeration.add_option("--current-user", dest="getCurrentUser",
                           action="store_true", default=False,
                           help="Retrieve DBMS current user")

    enumeration.add_option("--current-db", dest="getCurrentDb",
                           action="store_true", default=False,
                           help="Retrieve DBMS current database")

    enumeration.add_option("--is-dba", dest="isDba",
                           action="store_true", default=False,
                           help="Detect if the DBMS current user is DBA")

    enumeration.add_option("--users", dest="getUsers", action="store_true",
                           default=False, help="Enumerate DBMS users")

    enumeration.add_option("--passwords", dest="getPasswordHashes",
                           action="store_true", default=False,
                           help="Enumerate DBMS users password hashes")

    enumeration.add_option("--privileges", dest="getPrivileges",
                           action="store_true", default=False,
                           help="Enumerate DBMS users privileges")

    enumeration.add_option("--roles", dest="getRoles",
                           action="store_true", default=False,
                           help="Enumerate DBMS users roles")

    enumeration.add_option("--dbs", dest="getDbs", action="store_true",
                           default=False, help="Enumerate DBMS databases")

    enumeration.add_option("--tables", dest="getTables", action="store_true",
                           default=False, help="Enumerate DBMS database tables")

    enumeration.add_option("--columns", dest="getColumns", action="store_true",
                           default=False, help="Enumerate DBMS database table columns")

    enumeration.add_option("--dump", dest="dumpTable", action="store_true",
                           default=False, help="Dump DBMS database table entries")

    enumeration.add_option("--dump-all", dest="dumpAll", action="store_true",
                           default=False, help="Dump all DBMS databases tables entries")

    enumeration.add_option("--search", dest="search", action="store_true",
                           default=False, help="Search column(s), table(s) and/or database name(s)")

    enumeration.add_option("-D", dest="db",
                           help="DBMS database to enumerate")

    enumeration.add_option("-T", dest="tbl",
                           help="DBMS database table to enumerate")

    enumeration.add_option("-C", dest="col",
                           help="DBMS database table column to enumerate")

    enumeration.add_option("-U", dest="user",
                           help="DBMS user to enumerate")

    enumeration.add_option("--exclude-sysdbs", dest="excludeSysDbs",
                           action="store_true", default=False,
                           help="Exclude DBMS system databases when "
                                "enumerating tables")

    enumeration.add_option("--start", dest="limitStart", type="int",
                           help="First query output entry to retrieve")

    enumeration.add_option("--stop", dest="limitStop", type="int",
                           help="Last query output entry to retrieve")

    enumeration.add_option("--first", dest="firstChar", type="int",
                           help="First query output word character to retrieve")

    enumeration.add_option("--last", dest="lastChar", type="int",
                           help="Last query output word character to retrieve")

    enumeration.add_option("--sql-query", dest="query",
                           help="SQL statement to be executed")

    enumeration.add_option("--sql-shell", dest="sqlShell",
                           action="store_true", default=False,
                           help="Prompt for an interactive SQL shell")
```

在--help中是
```
Enumeration:
    These options can be used to enumerate the back-end database
    management system information, structure and data contained in the
    tables. Moreover you can run your own SQL statements.

    -b, --banner        Retrieve DBMS banner
    --current-user      Retrieve DBMS current user
    --current-db        Retrieve DBMS current database
    --is-dba            Detect if the DBMS current user is DBA
    --users             Enumerate DBMS users
    --passwords         Enumerate DBMS users password hashes
    --privileges        Enumerate DBMS users privileges
    --roles             Enumerate DBMS users roles
    --dbs               Enumerate DBMS databases
    --tables            Enumerate DBMS database tables
    --columns           Enumerate DBMS database table columns
    --dump              Dump DBMS database table entries
    --dump-all          Dump all DBMS databases tables entries
    --search            Search column(s), table(s) and/or database name(s)
    -D DB               DBMS database to enumerate
    -T TBL              DBMS database table to enumerate
    -C COL              DBMS database table column to enumerate
    -U USER             DBMS user to enumerate
    --exclude-sysdbs    Exclude DBMS system databases when enumerating tables
    --start=LIMITSTART  First query output entry to retrieve
    --stop=LIMITSTOP    Last query output entry to retrieve
    --first=FIRSTCHAR   First query output word character to retrieve
    --last=LASTCHAR     Last query output word character to retrieve
    --sql-query=QUERY   SQL statement to be executed
    --sql-shell         Prompt for an interactive SQL shell
```

这些选项涉及到sqlmap攻击的数据库目标，还可以执行自己的语句，可以说是sqlmap中最重要的一些参数了。

- -b, --banner 获取数据库的版本信息
- --current-user 获取数据库的用户名
- --current-db 获取数据库名字（当前）
- --is-dba 查看是否是数据库管理员
- --users 列出数据库所有用户
- --passwords 列出数据库所有用户hash
- --privileges 列出所有用户的权限
- --roles 列出所有用户的角色
- --dbs 列出所有数据库
- --tables  列出所有表
- --columns 列出所有列
- –dump -T “” -D “” -C “” #列出指定数据库的表的字段的数据(–dump -T users -D master -C surname)  
- --dump-all 列出所有表的数据
- --search 查询列、表、库名
- -D 指定数据库名
- -T 指定表名
- -C 指定列名
- -U 指定用户名
- --exclude-sysdbs 列举表是排除的数据库
- --start=LIMITSTART 第一次查询输出的条目
- --stop=LIMITSTOP 最后一次查询输出的条目
- --first=FIRSTCHAR 第一个查询输出的字符
- --last=LASTCHAR 最后一个查询输出的字符
- --sql-query=QUERY 执行指定的sql语句
- --sql-shell 执行指定的sql命令


## 爆破参数 ##

```
# User-defined function options
    brute = OptionGroup(parser, "Brute force", "These "
                      "options can be used to run brute force "
                      "checks.")

    brute.add_option("--common-tables", dest="commonTables", action="store_true",
                           default=False, help="Check existence of common tables")

    brute.add_option("--common-columns", dest="commonColumns", action="store_true",
                           default=False, help="Check existence of common columns")
```
在--help中是这样的
```
  Brute force:
    These options can be used to run brute force checks.

    --common-tables     Check existence of common tables
    --common-columns    Check existence of common columns
```

这部分主要是选择是否使用爆破

没啥好说的，分别针对表和列

## 自定义函数参数 ##
```
# User-defined function options
    udf = OptionGroup(parser, "User-defined function injection", "These "
                      "options can be used to create custom user-defined "
                      "functions.")

    udf.add_option("--udf-inject", dest="udfInject", action="store_true",
                   default=False, help="Inject custom user-defined functions")

    udf.add_option("--shared-lib", dest="shLib",
                   help="Local path of the shared library")
```

--help
```
 User-defined function injection:
    These options can be used to create custom user-defined functions.

    --udf-inject        Inject custom user-defined functions
    --shared-lib=SHLIB  Local path of the shared library
```

这部分主要是负责自定义函数，你可以通过编译mysql注入你想要的函数，也可以PostgreSQL在windows上共享库、dll，或者在linux上共享对象

- --udf-inject 注入传统的用户自定义函数
- --shared-lib=SHLIB 本地共享的库


## 文件系统配置 ##

```
 # File system options
    filesystem = OptionGroup(parser, "File system access", "These options "
                             "can be used to access the back-end database "
                             "management system underlying file system.")

    filesystem.add_option("--file-read", dest="rFile",
                          help="Read a file from the back-end DBMS "
                               "file system")

    filesystem.add_option("--file-write", dest="wFile",
                          help="Write a local file on the back-end "
                               "DBMS file system")

    filesystem.add_option("--file-dest", dest="dFile",
                          help="Back-end DBMS absolute filepath to "
                               "write to")

```

--help
```
 File system access:
    These options can be used to access the back-end database management
    system underlying file system.

    --file-read=RFILE   Read a file from the back-end DBMS file system
    --file-write=WFILE  Write a local file on the back-end DBMS file system
    --file-dest=DFILE   Back-end DBMS absolute filepath to write to
```

这些参数用来配置后端数据库管理的文件系统

- --file-read 读文件
- --file-write 写文件
- --file-dest 绝对路径写文件

## 操作系统参数 ##

```
# Takeover options
    takeover = OptionGroup(parser, "Operating system access", "These "
                           "options can be used to access the back-end "
                           "database management system underlying "
                           "operating system.")

    takeover.add_option("--os-cmd", dest="osCmd",
                        help="Execute an operating system command")

    takeover.add_option("--os-shell", dest="osShell",
                        action="store_true", default=False,
                        help="Prompt for an interactive operating "
                             "system shell")

    takeover.add_option("--os-pwn", dest="osPwn",
                        action="store_true", default=False,
                        help="Prompt for an out-of-band shell, "
                             "meterpreter or VNC")

    takeover.add_option("--os-smbrelay", dest="osSmb",
                        action="store_true", default=False,
                        help="One click prompt for an OOB shell, "
                             "meterpreter or VNC")

    takeover.add_option("--os-bof", dest="osBof",
                        action="store_true", default=False,
                        help="Stored procedure buffer overflow "
                             "exploitation")

    takeover.add_option("--priv-esc", dest="privEsc",
                        action="store_true", default=False,
                        help="Database process' user privilege escalation")

    takeover.add_option("--msf-path", dest="msfPath",
                        help="Local path where Metasploit Framework 3 "
                             "is installed")

    takeover.add_option("--tmp-path", dest="tmpPath",
                        help="Remote absolute path of temporary files "
                             "directory")

```
--help

```
 Operating system access:
    These options can be used to access the back-end database management
    system underlying operating system.

    --os-cmd=OSCMD      Execute an operating system command
    --os-shell          Prompt for an interactive operating system shell
    --os-pwn            Prompt for an out-of-band shell, meterpreter or VNC
    --os-smbrelay       One click prompt for an OOB shell, meterpreter or VNC
    --os-bof            Stored procedure buffer overflow exploitation
    --priv-esc          Database process' user privilege escalation
    --msf-path=MSFPATH  Local path where Metasploit Framework 3 is installed
    --tmp-path=TMPPATH  Remote absolute path of temporary files directory
```

这部分的参数主要是用于对数据库后端的操作系统的惭怍

- --os-cmd=OSCMD 执行一个操作系统的shell
- --os-shell 提供一个交互式的shell
- --os-pwn 提供一个shell, meterpreter or VNC
- --os-smbrelay 提供一个an OOB shell, meterpreter or VNC
- --os-bof 使用一个缓冲区溢出
- --priv-esc 数据库进程提权
- --msf-path=MSFPATH 本地Metasploit的安装路径
- --tmp-path=TMPPATH 远程临时文件目录的绝对路径


## windows注册表参数 ##

```
# Windows registry options
    windows = OptionGroup(parser, "Windows registry access", "These "
                           "options can be used to access the back-end "
                           "database management system Windows "
                           "registry.")

    windows.add_option("--reg-read", dest="regRead",
                        action="store_true", default=False,
                        help="Read a Windows registry key value")

    windows.add_option("--reg-add", dest="regAdd",
                        action="store_true", default=False,
                        help="Write a Windows registry key value data")

    windows.add_option("--reg-del", dest="regDel",
                        action="store_true", default=False,
                        help="Delete a Windows registry key value")

    windows.add_option("--reg-key", dest="regKey",
                        help="Windows registry key")

    windows.add_option("--reg-value", dest="regVal",
                        help="Windows registry key value")

    windows.add_option("--reg-data", dest="regData",
                        help="Windows registry key value data")

    windows.add_option("--reg-type", dest="regType",
                        help="Windows registry key value type")
```

--help
```
 Windows registry access:
    These options can be used to access the back-end database management
    system Windows registry.

    --reg-read          Read a Windows registry key value
    --reg-add           Write a Windows registry key value data
    --reg-del           Delete a Windows registry key value
    --reg-key=REGKEY    Windows registry key
    --reg-value=REGVAL  Windows registry key value
    --reg-data=REGDATA  Windows registry key value data
    --reg-type=REGTYPE  Windows registry key value type
```

这个参数主要是用来针对后端数据库操作系统的windows注册表

- --reg-read 读注册表的键值
- --reg-add 添加注册表的键值
- --reg-del 删除注册表的键值
- --reg-key=REGKEY windows注册表的键
- --reg-value=REGVAL windows注册表的键值
- --reg-data=REGDATA windows注册表的键值数据
- --reg-type=REGTYPE windows注册表的键值类型


## 一般工作参数 ##

```
# General options
    general = OptionGroup(parser, "General", "These options can be used "
                         "to set some general working parameters. " )

    #general.add_option("-x", dest="xmlFile",
    #                    help="Dump the data into an XML file")

    general.add_option("-t", dest="trafficFile",
                        help="Log all HTTP traffic into a "
                        "textual file")

    general.add_option("-s", dest="sessionFile",
                        help="Save and resume all data retrieved "
                        "on a session file")

    general.add_option("--flush-session", dest="flushSession",
                        action="store_true", default=False,
                        help="Flush session file for current target")

    general.add_option("--fresh-queries", dest="freshQueries",
                        action="store_true", default=False,
                        help="Ignores query results stored in session file")

    general.add_option("--eta", dest="eta",
                        action="store_true", default=False,
                        help="Display for each output the "
                                  "estimated time of arrival")

    general.add_option("--update", dest="updateAll",
                        action="store_true", default=False,
                        help="Update sqlmap")

    general.add_option("--save", dest="saveCmdline",
                        action="store_true", default=False,
                        help="Save options on a configuration INI file")

    general.add_option("--batch", dest="batch",
                        action="store_true", default=False,
                        help="Never ask for user input, use the default behaviour")

```

--help
```
 General:
    These options can be used to set some general working parameters.

    -t TRAFFICFILE      Log all HTTP traffic into a textual file
    -s SESSIONFILE      Save and resume all data retrieved on a session file
    --flush-session     Flush session file for current target
    --fresh-queries     Ignores query results stored in session file
    --eta               Display for each output the estimated time of arrival
    --update            Update sqlmap
    --save              Save options on a configuration INI file
    --batch             Never ask for user input, use the default behaviour
```

这部分选项是用来设置某些一般的工作参数。

- -t TRAFFICFILE 记录所有的http流量到一个文本文件
- -s SESSIONFILE 保存和恢复所有的数据到一个session文件
- --flush-session 对于当前的目标刷新已有的session
- --fresh-queries 忽略查询结果中已有的结果
- --eta 预计每个输出的显示时间
- --update 更新sqlmap
- --save 保存配置文件的ini文件
- --batch 使用默认的行为

## 杂项配置 ##
```
# Miscellaneous options
    miscellaneous = OptionGroup(parser, "Miscellaneous")

    miscellaneous.add_option("--beep", dest="beep",
                              action="store_true", default=False,
                              help="Alert when sql injection found")

    miscellaneous.add_option("--check-payload", dest="checkPayload",
                              action="store_true", default=False,
                              help="IDS detection testing of injection payloads")

    miscellaneous.add_option("--cleanup", dest="cleanup",
                              action="store_true", default=False,
                              help="Clean up the DBMS by sqlmap specific "
                              "UDF and tables")

    miscellaneous.add_option("--forms", dest="forms",
                              action="store_true", default=False,
                              help="Parse and test forms on target url")

    miscellaneous.add_option("--gpage", dest="googlePage", type="int",
                              help="Use Google dork results from specified page number")

    miscellaneous.add_option("--page-rank", dest="pageRank",
                              action="store_true", default=False,
                              help="Display page rank (PR) for Google dork results")

    miscellaneous.add_option("--parse-errors", dest="parseErrors",
                              action="store_true", default=False,
                              help="Parse DBMS error messages from response pages")

    miscellaneous.add_option("--replicate", dest="replicate",
                              action="store_true", default=False,
                              help="Replicate dumped data into a sqlite3 database")

    miscellaneous.add_option("--tor", dest="tor", 
                              action="store_true", default=False,
                              help="Use default Tor (Vidalia/Privoxy/Polipo) proxy address")

    miscellaneous.add_option("--wizard", dest="wizard",
                              action="store_true", default=False,
                              help="Simple wizard interface for beginner users")
```
--help
```
 Miscellaneous:
    --beep              Alert when sql injection found
    --check-payload     IDS detection testing of injection payloads
    --cleanup           Clean up the DBMS by sqlmap specific UDF and tables
    --forms             Parse and test forms on target url
    --gpage=GOOGLEPAGE  Use Google dork results from specified page number
    --page-rank         Display page rank (PR) for Google dork results
    --parse-errors      Parse DBMS error messages from response pages
    --replicate         Replicate dumped data into a sqlite3 database
    --tor               Use default Tor (Vidalia/Privoxy/Polipo) proxy address
    --wizard            Simple wizard interface for beginner users
```

这里剩下的是一些杂项配置
- --beep 当发现注入时弹窗出来
- --check-payload IDS检测注入payload
- --cleanup 清理sqlmap特定的UDF和表
- --forms 解析和测试你的目标url
- --gpage=GOOGLEPAGE 使用Google dork指定结果页数
- --page-rank 显示google dork的排名结果
- --parse-errors 从页面解析数据库的错误信息
- --replicate 复制数据到sqlite3
- --tor 使用默认的代理地址
- --wizard 为新手配置简单的页面


# 初始化 #
```sqlmap.py(73-82)
 try:
    init(cmdLineOptions)
    if conf.profile:
        profile()
    elif conf.smokeTest:
        smokeTest()
    elif conf.liveTest:
        liveTest()
    else:
        start()
```

从这里开始就是正式开始解析sqlmap的参数，执行相应的函数了