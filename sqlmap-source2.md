---
title: sqlmap 源码分析（二）初始化
date: 2016-08-11 15:19:19
tags:
- python
- sqlmap
categories:
- Blogs
---


sqlmap是web狗永远也绕不过去的神器，为了能自由的使用sqlmap，阅读源码还是有必要的...

<!--more-->

# 初始化 #

参数解析完后，开始初始化
```
init(cmdLineOptions)
```

这一部分主要是根据之前的参数，设置属性和很多基于命令行和配置文件的选项

## 初始化一些必要的配置属性 ##

```/lib/core/option.py(1197-1226)
debugMsg = "initializing the configuration"
    logger.debug(debugMsg)

    conf.boundaries       = []
    conf.cj               = None
    conf.dbmsConnector    = None
    conf.dbmsHandler      = None
    conf.dumpPath         = None
    conf.httpHeaders      = []
    conf.hostname         = None
    conf.loggedToOut      = None
    conf.multipleTargets  = False
    conf.outputPath       = None
    conf.paramDict        = {}
    conf.parameters       = {}
    conf.path             = None
    conf.port             = None
    conf.redirectHandled  = False
    conf.scheme           = None
    conf.sessionFP        = None
    conf.start            = True
    conf.tests            = []
    conf.trafficFP        = None
    conf.wFileType        = None
```
后面也相同
`__setKnowledgeBaseAttributes()`

初始化了一些有关知识库配置的参数

然后是配置文件的初始化
`__mergeOptions(inputOptions, overrideOptions)`

# 新手引导 #

在分析参数的时候有一个叫做wizard的参数，是关于新手引导的，如果开启就会进入引导页面

```
def __useWizardInterface():
    """
    Presents simple wizard interface for beginner users
    """

    if not conf.wizard:
        return

    logger.info("starting wizard interface")

    while not conf.url:
        message = "Please enter full target URL (-u): "
        conf.url = readInput(message, default=None)

    message = "POST data (--data) [Enter for None]: "
    conf.data = readInput(message, default=None)

    choice = None

    while choice is None or choice not in ("", "1", "2", "3"):
        message = "Injection difficulty (--level/--risk). Please choose:\n"
        message += "[1] Normal (default)\n[2] Medium\n[3] Hard"
        choice = readInput(message, default='1')

        if choice == '2':
            conf.risk = 2
            conf.level = 3
        elif choice == '3':
            conf.risk = 3
            conf.level = 5
        else:
            conf.risk = 1
            conf.level = 1

    choice = None

    while choice is None or choice not in ("", "1", "2", "3"):
        message = "Enumeration (--banner/--current-user/etc). Please choose:\n"
        message += "[1] Basic (default)\n[2] Smart\n[3] All"
        choice = readInput(message, default='1')

        if choice == '2':
            map(lambda x: conf.__setitem__(x, True), ['getBanner', 'getCurrentUser', 'getCurrentDb', 'isDba', 'getUsers', 'getDbs', 'getTables', 'excludeSysDbs'])
        elif choice == '3':
            map(lambda x: conf.__setitem__(x, True), ['getBanner', 'getCurrentUser', 'getCurrentDb', 'isDba', 'getUsers', 'getPasswordHashes', 'getPrivileges', 'getRoles', 'dumpAll'])
        else:
            map(lambda x: conf.__setitem__(x, True), ['getBanner', 'getCurrentUser', 'getCurrentDb', 'isDba'])

    conf.batch = True
    conf.threads = 4

    logger.debug("muting sqlmap.. it will do the magic for you")
    conf.verbose = 0

    dataToStdout("\nsqlmap is running, please wait..\n\n")
```
从输入目标url
-->post数据
-->选择注入级别
-->注入目标的选择
-->配置完毕

# 设置输出log级别 #

`__setVerbosity():`

这里主要是设置了一些输出的log级别，在logger的基本设置上，sqlmap还拓展到了8级
```
def __setVerbosity():
    """
    This function set the verbosity of sqlmap output messages.
    """

    if conf.verbose is None:
        conf.verbose = 1

    conf.verbose = int(conf.verbose)

    if conf.verbose == 0:
        logger.setLevel(logging.ERROR)
    elif conf.verbose == 1:
        logger.setLevel(logging.INFO)
    elif conf.verbose > 2 and conf.eta:
        conf.verbose = 2
        logger.setLevel(logging.DEBUG)
    elif conf.verbose == 2:
        logger.setLevel(logging.DEBUG)
    elif conf.verbose == 3:
        logger.setLevel(9)
    elif conf.verbose == 4:
        logger.setLevel(8)
    elif conf.verbose >= 5:
        logger.setLevel(7)
```

# 保存命令行的配置到ini文件 #

当然，和前面的新手引导类似，这是需要额外参数才会进行的配置

`__saveCmdline`

```
def __saveCmdline():
    """
    Saves the command line options on a sqlmap configuration INI file
    Format.
    """

    if not conf.saveCmdline:
        return

    debugMsg = "saving command line options on a sqlmap configuration INI file"
    logger.debug(debugMsg)

    config = UnicodeRawConfigParser()
    userOpts = {}

    for family in optDict.keys():
        userOpts[family] = []

    for option, value in conf.items():
        for family, optionData in optDict.items():
            if option in optionData:
                userOpts[family].append((option, value, optionData[option]))

    for family, optionData in userOpts.items():
        config.add_section(family)

        optionData.sort()

        for option, value, datatype in optionData:
            if isinstance(datatype, (list, tuple, set)):
                datatype = datatype[0]

            if value is None:
                if datatype == "boolean":
                    value = "False"
                elif datatype in ( "integer", "float" ):
                    if option in ( "threads", "verbose" ):
                        value = "1"
                    elif option == "timeout":
                        value = "10"
                    else:
                        value = "0"
                elif datatype == "string":
                    value = ""

            if isinstance(value, basestring):
                value = value.replace("\n", "\n ")

            config.set(family, option, value)

    confFP = openFile(paths.SQLMAP_CONFIG, "wb")
    config.write(confFP)

    infoMsg = "saved command line options on '%s' configuration file" % paths.SQLMAP_CONFIG
    logger.info(infoMsg)
```

这里设置了默认的线程数，log日志级别、延时等等...并将其储存到了设定好的sqlmap_config目录中。

# 处理从文件获取的http请求 #
`__setRequestFromFile():`

```
def __setRequestFromFile():
    """
    This function checks if the way to make a HTTP request is through supplied
    textual file, parses it and saves the information into the knowledge base.
    """

    if not conf.requestFile:
        return

    addedTargetUrls = set()

    conf.requestFile = os.path.expanduser(conf.requestFile)

    infoMsg = "parsing HTTP request from '%s'" % conf.requestFile
    logger.info(infoMsg)

    if not os.path.isfile(conf.requestFile):
        errMsg  = "the specified HTTP request file "
        errMsg += "does not exist"
        raise sqlmapFilePathException, errMsg

    __feedTargetsDict(conf.requestFile, addedTargetUrls)

```

这里读取了conf.requestFile的内容初始化完成，然后开始处理文件内容

`__feedTargetsDict(conf.requestFile, addedTargetUrls)`

读取文件
```
fp = openFile(reqFile, "rb")

    content = fp.read()
    content = content.replace("\r", "")

    if conf.scope:
        logger.info("using regular expression '%s' for filtering targets" % conf.scope)
```
开始解包
```
__parseBurpLog(content)
__parseWebScarabLog(content)
```

先是一些基本的处理和判断

```
url    = extractRegexResult(r"URL: (?P<result>.+?)\n", request, re.I)
method = extractRegexResult(r"METHOD: (?P<result>.+?)\n", request, re.I)
cookie = extractRegexResult(r"COOKIE: (?P<result>.+?)\n", request, re.I)
getPostReq = True

if not method or not url:
    logger.debug("Invalid log data")
    continue
```
然后指明分析log文件是不支持post请求的
```
 if method.upper() == "POST":
    warnMsg = "POST requests from WebScarab logs aren't supported "
    warnMsg += "as their body content is stored in separate files. "
    warnMsg += "Nevertheless you can use -r to load them individually."
    logger.warning(warnMsg)
    continue
```

开始解包
` __parseWebScarabLog(content)`

分割出数据传输方式以及端口号
```
 if scheme is None:
    schemePort = re.search("\d\d[\:|\.]\d\d[\:|\.]\d\d\s+(http[\w]*)\:\/\/.*?\:([\d]+)", request, re.I)

    if schemePort:
        scheme = schemePort.group(1)
        port   = schemePort.group(2)
```

跳过一些无用的行，re.search如果搜索不到就会返回None
```
if not re.search ("^[\n]*(GET|POST).*?\sHTTP\/", request, re.I):
    continue

if re.search("^[\n]*(GET|POST).*?\.(gif|jpg|png)\sHTTP\/", request, re.I):
    continue
```
根据请求方式的不同，用多重方式获取
```
if len(line) == 0 or line == "\n":
    if method == HTTPMETHOD.POST and data is None:
        data = ""
        params = True

elif (line.startswith("GET ") or line.startswith("POST ")) and " HTTP/" in line:
    if line.startswith("GET "):
        index = 4
    else:
        index = 5

    url = line[index:line.index(" HTTP/")]
    method = line[:index-1]

    if "?" in line and "=" in line:
        params = True

    getPostReq = True
 # POST parameters
elif data is not None and params:
    data += line

# GET parameters
elif "?" in line and "=" in line and ": " not in line:
    params = True

```
然后处理请求头

```
 # Headers
elif ": " in line:
    key, value = line.split(": ", 1)

    # Cookie and Host headers
    if key.lower() == "cookie":
        cookie = value
    elif key.lower() == "host":
        if '://' in value:
            scheme, value = value.split('://')[:2]
        splitValue = value.split(":")
        host = splitValue[0]

        if len(splitValue) > 1:
            port = filterStringValue(splitValue[1], '[0-9]')

            if not scheme and port == "443":
                scheme = "https"

    # Avoid to add a static content length header to
    # conf.httpHeaders and consider the following lines as
    # POSTed data
    if key == "Content-Length":
        params = True

    # Avoid proxy and connection type related headers
    elif key not in ( "Proxy-Connection", "Connection" ):
        conf.httpHeaders.append((str(key), str(value)))
```

# 一些基本的参数错误处理 #

`__basicOptionValidation()`

```
if conf.limitStart is not None and not (isinstance(conf.limitStart, int) and conf.limitStart > 0):
        errMsg = "value for --start (limitStart) option must be an integer value greater than zero (>0)"
        raise sqlmapSyntaxException, errMsg

    if conf.limitStop is not None and not (isinstance(conf.limitStop, int) and conf.limitStop > 0):
        errMsg = "value for --stop (limitStop) option must be an integer value greater than zero (>0)"
        raise sqlmapSyntaxException, errMsg

    if conf.limitStart is not None and isinstance(conf.limitStart, int) and conf.limitStart > 0 and \
       conf.limitStop is not None and isinstance(conf.limitStop, int) and conf.limitStop <= conf.limitStart:
        errMsg = "value for --start (limitStart) option must be smaller than value for --stop (limitStop) option"
        raise sqlmapSyntaxException, errMsg

    if conf.firstChar is not None and isinstance(conf.firstChar, int) and conf.firstChar > 0 and \
       conf.lastChar is not None and isinstance(conf.lastChar, int) and conf.lastChar < conf.firstChar:
        errMsg = "value for --first (firstChar) option must be smaller than or equal to value for --last (lastChar) option"
        raise sqlmapSyntaxException, errMsg

    if conf.cpuThrottle is not None and isinstance(conf.cpuThrottle, int) and (conf.cpuThrottle > 100 or conf.cpuThrottle < 0):
        errMsg = "value for --cpu-throttle (cpuThrottle) option must be in range [0,100]"
        raise sqlmapSyntaxException, errMsg

    if conf.textOnly and conf.nullConnection:
        errMsg = "switch --text-only is incompatible with switch --null-connection"
        raise sqlmapSyntaxException, errMsg

    if conf.data and conf.nullConnection:
        errMsg = "switch --data is incompatible with switch --null-connection"
        raise sqlmapSyntaxException, errMsg

    if conf.predictOutput and conf.threads > 1:
        errMsg = "switch --predict-output is incompatible with switch --threads"
        raise sqlmapSyntaxException, errMsg

    if conf.threads > MAX_NUMBER_OF_THREADS:
        errMsg = "maximum number of used threads is %d avoiding possible connection issues" % MAX_NUMBER_OF_THREADS
        raise sqlmapSyntaxException, errMsg

    if conf.forms and not conf.url:
        errMsg = "switch --forms requires usage of -u (--url) switch"
        raise sqlmapSyntaxException, errMsg

    if conf.proxy and conf.ignoreProxy:
        errMsg = "switch --proxy is incompatible with switch --ignore-proxy"
        raise sqlmapSyntaxException, errMsg

    if conf.forms and (conf.list or conf.direct or conf.requestFile or conf.googleDork):
        errMsg = "switch --forms is compatible only with -u (--url) target switch"
        raise sqlmapSyntaxException, errMsg

    if conf.timeSec < 1:
        errMsg = "value for --time-sec option must be an integer greater than 0"
        raise sqlmapSyntaxException, errMsg

    if isinstance(conf.uCols, basestring) and ("-" not in conf.uCols or len(conf.uCols.split("-")) != 2):
        errMsg = "value for --union-cols must be a range with hyphon (e.g. 1-10)"
        raise sqlmapSyntaxException, errMsg
```

一些格式错误，内容错误都包含进去了

# 加载tamper的自定义函数 #

`__setTamperingFunctions()`

看了看，没什么特别的，就是简单的加载并做了一些错误处理

# 解析目标url&设置一些配置 #

`parseTargetUrl()`
首先对传入的目标url解析，分别把目标、端口、路径、域名都解析出来

```
if not conf.url:
        return

    if not re.search("^http[s]*://", conf.url):
        if ":443/" in conf.url:
            conf.url = "https://" + conf.url
        else:
            conf.url = "http://" + conf.url

    if URI_INJECTION_MARK_CHAR in conf.url:
        conf.url = conf.url.replace('?', URI_QUESTION_MARKER)

    __urlSplit = urlparse.urlsplit(conf.url)
    __hostnamePort = __urlSplit[1].split(":")

    conf.scheme = __urlSplit[0].strip()
    conf.path = __urlSplit[2].strip()
    conf.hostname = __hostnamePort[0].strip()
```

对于不自带端口的url，专门分析并设置端口号

```
if len(__hostnamePort) == 2:
        try:
            conf.port = int(__hostnamePort[1])
        except:
            errMsg = "invalid target url"
            raise sqlmapSyntaxException, errMsg
    elif conf.scheme == "https":
        conf.port = 443
    else:
        conf.port = 80
```

连接成conf.url
```
conf.url = "%s://%s:%d%s" % (conf.scheme, conf.hostname, conf.port, conf.path)
conf.url = conf.url.replace(URI_QUESTION_MARKER, '?')
```

# 解析目标数据库 #

这里解析目标数据库并设置了一些配置
`parseTargetDirect()`

这里处理的是直连数据库的解析函数，也就是前面提到的`-d`

```
if not conf.direct:
        return
```

配置相应的参数

```
for dbms in SUPPORTED_DBMS:
    details = re.search("^(?P<dbms>%s)://(?P<credentials>(?P<user>.+?)\:(?P<pass>.*?)\@)?(?P<remote>(?P<hostname>.+?)\:(?P<port>[\d]+)\/)?(?P<db>[\w\d\ \:\.\_\-\/\\\\]+?)$" % dbms, conf.direct, re.I)

    if details:
        conf.dbms = details.group('dbms')

        if details.group('credentials'):
            conf.dbmsUser = details.group('user')
            conf.dbmsPass = details.group('pass')
        else:
            conf.dbmsUser = unicode()
            conf.dbmsPass = unicode()

        if not conf.dbmsPass:
            conf.dbmsPass = None

        if details.group('remote'):
            remote = True
            conf.hostname = details.group('hostname')
            conf.port = int(details.group('port'))
        else:
            conf.hostname = "localhost"
            conf.port = 0

        conf.dbmsDb = details.group('db')

        conf.parameters[None] = "direct connection"

        break
```

处理并加载相应的模块

```
if dbmsName in (DBMS.MSSQL, DBMS.SYBASE):
    import _mssql
    import pymssql

    if not hasattr(pymssql, "__version__") or pymssql.__version__ < "1.0.2":
        errMsg = "pymssql library on your system must be "
        errMsg += "version 1.0.2 to work, get it from "
        errMsg += "http://sourceforge.net/projects/pymssql/files/pymssql/1.0.2/"
        raise sqlmapMissingDependence, errMsg

elif dbmsName == DBMS.MYSQL:
    import MySQLdb
elif dbmsName == DBMS.PGSQL:
    import psycopg2
elif dbmsName == DBMS.ORACLE:
    import cx_Oracle
elif dbmsName == DBMS.SQLITE:
    import sqlite3
elif dbmsName == DBMS.ACCESS:
    import pyodbc
elif dbmsName == DBMS.FIREBIRD:
    import kinterbasdb
```

# 开始一些注入有关的配置 #

## 设置延时 ##
`_setHTTPTimeout()`

除了默认设置30.0以外还判断不能小于3.0
```
if conf.timeout:
    debugMsg = "setting the HTTP timeout"
    logger.debug(debugMsg)

    conf.timeout = float(conf.timeout)

    if conf.timeout < 3.0:
        warnMsg = "the minimum HTTP timeout is 3 seconds, sqlmap "
        warnMsg += "will going to reset it"
        logger.warn(warnMsg)

        conf.timeout = 3.0
else:
    conf.timeout = 30.0

socket.setdefaulttimeout(conf.timeout)
```

## 设置请求头header ##
`_setHTTPExtraHeaders()`

如果设置了头，那么就把header中的内容以列表的形式分割并赋值给conf.httpHeaders,
如果没设置，那就默认头输入到conf.httpHeaders

```
if conf.headers:
    debugMsg = "setting extra HTTP headers"
    logger.debug(debugMsg)

    conf.headers = conf.headers.split("\n") if "\n" in conf.headers else conf.headers.split("\\n")

    for headerValue in conf.headers:
        if not headerValue.strip():
            continue

        if headerValue.count(':') >= 1:
            header, value = (_.lstrip() for _ in headerValue.split(":", 1))

            if header and value:
                conf.httpHeaders.append((header, value))
        else:
            errMsg = "invalid header value: %s. Valid header format is 'name:value'" % repr(headerValue).lstrip('u')
            raise SqlmapSyntaxException(errMsg)

elif not conf.requestFile and len(conf.httpHeaders or []) < 2:
    conf.httpHeaders.append((HTTP_HEADER.ACCEPT_LANGUAGE, "en-us,en;q=0.5"))
    if not conf.charset:
        conf.httpHeaders.append((HTTP_HEADER.ACCEPT_CHARSET, "ISO-8859-15,utf-8;q=0.7,*;q=0.7"))
    else:
        conf.httpHeaders.append((HTTP_HEADER.ACCEPT_CHARSET, "%s;q=0.7,*;q=0.1" % conf.charset))

    # Invalidating any caching mechanism in between
    # Reference: http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html
    conf.httpHeaders.append((HTTP_HEADER.CACHE_CONTROL, "no-cache,no-store"))
    conf.httpHeaders.append((HTTP_HEADER.PRAGMA, "no-cache"))
```

## 设置cookie ##

`_setHTTPCookies()`

把cookie加入到conf.httpHeaders列表中
```
if conf.cookie:
    debugMsg = "setting the HTTP Cookie header"
    logger.debug(debugMsg)

    conf.httpHeaders.append((HTTP_HEADER.COOKIE, conf.cookie))
```

## 设置referer ##

和cookie相同，这里是吧referer加入到conf.httpHeaders列表中
```
if conf.referer:
    debugMsg = "setting the HTTP Referer header"
    logger.debug(debugMsg)

    conf.httpHeaders.append((HTTP_HEADER.REFERER, conf.referer))
```

还有后面的一系列，也都是设置关于请求头的host和ua
```
_setHTTPHost()
_setHTTPUserAgent()
```

## 设置http用户验证 ##

`_setHTTPAuthentication()`

检查http用户验证，判断属于(Basic, Digest, NTLM or PKI)中的哪一种方法，其中前三种需要用户密码，后一种需要私钥文件

涉及的东西比较多，就不贴代码了

## 设置http代理 ##

`_setHTTPHandlers()`

简单看一下就是简单的判断，然后使用http/socks的代理。

基本的代理列表处理
```
if conf.proxyList is not None:
    if not conf.proxyList:
        errMsg = "list of usable proxies is exhausted"
        raise SqlmapNoneDataException(errMsg)

    conf.proxy = conf.proxyList[0]
    conf.proxyList = conf.proxyList[1:]

    infoMsg = "loading proxy '%s' from a supplied proxy list file" % conf.proxy
    logger.info(infoMsg)

elif not conf.proxy:
    if conf.hostname in ("localhost", "127.0.0.1") or conf.ignoreProxy:
        proxyHandler.proxies = {}
```

简单的配置并设置代理...

## 设置DNS缓存 ##

使用socket._getaddrinfo来设置请求的dns缓存

```
def _getaddrinfo(*args, **kwargs):
    if args in kb.cache:
        return kb.cache[args]

    else:
        kb.cache[args] = socket._getaddrinfo(*args, **kwargs)
        return kb.cache[args]

if not hasattr(socket, "_getaddrinfo"):
    socket._getaddrinfo = socket.getaddrinfo
    socket.getaddrinfo = _getaddrinfo
```

## 设置socket链接 ##

`_setSocketPreConnect()`

设置socket链接

简单的配置已经开始socket连接
```
if not hasattr(socket.socket, "_connect"):
    socket._ready = {}
    socket.socket._connect = socket.socket.connect
    socket.socket.connect = connect

    thread = threading.Thread(target=_)
    setDaemon(thread)
    thread.start()
```

## 设置安全连接 ##

`_setSafeVisit()`

简单的解包，然后设置安全连接
```
raw = readCachedFileContent(conf.safeReqFile)
match = re.search(r"\A([A-Z]+) ([^ ]+) HTTP/[0-9.]+\Z", raw[:raw.find('\n')])
```

设置安全链接
```
if match:
    kb.safeReq.method = match.group(1)
    kb.safeReq.url = match.group(2)
    kb.safeReq.headers = {}

    for line in raw[raw.find('\n') + 1:].split('\n'):
        line = line.strip()
        if line and ':' in line:
            key, value = line.split(':', 1)
            value = value.strip()
            kb.safeReq.headers[key] = value
            if key == HTTP_HEADER.HOST:
                if not value.startswith("http"):
                    scheme = "http"
                    if value.endswith(":443"):
                        scheme = "https"
                    value = "%s://%s" % (scheme, value)
                kb.safeReq.url = urlparse.urljoin(value, kb.safeReq.url)
        else:
            break

    post = None

    if '\r\n\r\n' in raw:
        post = raw[raw.find('\r\n\r\n') + 4:]
    elif '\n\n' in raw:
        post = raw[raw.find('\n\n') + 2:]

    if post and post.strip():
        kb.safeReq.post = post
    else:
        kb.safeReq.post = None
```

## 一些乱七八糟的东西 ##

```
_doSearch()
_setBulkMultipleTargets()
_setSitemapTargets()
_setCrawler()
_findPageForms()
```
根据参数的一些设置，还有tor connection的判断并设置
`_checkTor()`

没有深究的必要，接着看后面的

## 关于设置数据库的check ##

`_setDBMS()`

简单的处理以及判定

```
conf.dbms = conf.dbms.lower()
regex = re.search("%s ([\d\.]+)" % ("(%s)" % "|".join([alias for alias in SUPPORTED_DBMS])), conf.dbms, re.I)

if regex:
    conf.dbms = regex.group(1)
    Backend.setVersion(regex.group(2))

if conf.dbms not in SUPPORTED_DBMS:
    errMsg = "you provided an unsupported back-end database management "
    errMsg += "system. Supported DBMSes are as follows: %s. " % ', '.join(sorted(_ for _ in DBMS_DICT))
    errMsg += "If you do not know the back-end DBMS, do not provide "
    errMsg += "it and sqlmap will fingerprint it for you."
    raise SqlmapUnsupportedDBMSException(errMsg)

for dbms, aliases in DBMS_ALIASES:
    if conf.dbms in aliases:
        conf.dbms = dbms

        break
```

## 关于设置注入类型的check ##

和前面的数据库类型check相同，用户的设置在这里经过判断

`_setTechnique()`

同样是简单的处理以及判定

```
validTechniques = sorted(getPublicTypeMembers(PAYLOAD.TECHNIQUE), key=lambda x: x[1])
validLetters = [_[0][0].upper() for _ in validTechniques]

if conf.tech and isinstance(conf.tech, basestring):
    _ = []

    for letter in conf.tech.upper():
        if letter not in validLetters:
            errMsg = "value for --technique must be a string composed "
            errMsg += "by the letters %s. Refer to the " % ", ".join(validLetters)
            errMsg += "user's manual for details"
            raise SqlmapSyntaxException(errMsg)

        for validTech, validInt in validTechniques:
            if letter == validTech[0]:
                _.append(validInt)
                break

    conf.tech = _
```

## 设置线程数 ##

```
if not isinstance(conf.threads, int) or conf.threads <= 0:
        conf.threads = 1
```

## 检查后端数据库系统 ##

`_setOS()`
简单的处理和判定

```
if not conf.os:
        return

    if conf.os.lower() not in SUPPORTED_OS:
        errMsg = "you provided an unsupported back-end DBMS operating "
        errMsg += "system. The supported DBMS operating systems for OS "
        errMsg += "and file system access are %s. " % ', '.join([o.capitalize() for o in SUPPORTED_OS])
        errMsg += "If you do not know the back-end DBMS underlying OS, "
        errMsg += "do not provide it and sqlmap will fingerprint it for "
        errMsg += "you."
        raise SqlmapUnsupportedDBMSException(errMsg)

    debugMsg = "forcing back-end DBMS operating system to user defined "
    debugMsg += "value '%s'" % conf.os
    logger.debug(debugMsg)

    Backend.setOs(conf.os)
```
## 检查写文件目录 ##

`_setWriteFile()`

没什么特别的，就是基本的判定
```
if not os.path.exists(conf.wFile):
    errMsg = "the provided local file '%s' does not exist" % conf.wFile
    raise SqlmapFilePathException(errMsg)

if not conf.dFile:
    errMsg = "you did not provide the back-end DBMS absolute path "
    errMsg += "where you want to write the local file '%s'" % conf.wFile
    raise SqlmapMissingMandatoryOptionException(errMsg)

conf.wFileType = getFileType(conf.wFile)
```

## 检查Metasploit的设置 ##

本地Metasploit的一些配置，就不贴代码了

## 检查设置的数据库身份验证语句 ##

没什么可说的

```
if not conf.dbmsCred:
    return

debugMsg = "setting the DBMS authentication credentials"
logger.debug(debugMsg)

match = re.search("^(.+?):(.*?)$", conf.dbmsCred)

if not match:
    errMsg = "DBMS authentication credentials value must be in format "
    errMsg += "username:password"
    raise SqlmapSyntaxException(errMsg)

conf.dbmsUsername = match.group(1)
conf.dbmsPassword = match.group(2)
```

## 加载测试语句 ##

加载测试语句并解析，这里的paths.BOUNDARIES_XML为E:\sqlmap\xml\boundaries.xml
```
def loadBoundaries():
	try:
	    doc = et.parse(paths.BOUNDARIES_XML)
	except Exception, ex:
	    errMsg = "something appears to be wrong with "
	    errMsg += "the file '%s' ('%s'). Please make " % (paths.BOUNDARIES_XML, getSafeExString(ex))
	    errMsg += "sure that you haven't made any changes to it"
	    raise SqlmapInstallationException, errMsg
	
	root = doc.getroot()
	parseXmlNode(root)
```

## 加载payload格式 ##

然后是加载payload格式
```
def loadPayloads():
    for payloadFile in PAYLOAD_XML_FILES:
        payloadFilePath = os.path.join(paths.SQLMAP_XML_PAYLOADS_PATH, payloadFile)

        try:
            doc = et.parse(payloadFilePath)
        except Exception, ex:
            errMsg = "something appears to be wrong with "
            errMsg += "the file '%s' ('%s'). Please make " % (payloadFilePath, getSafeExString(ex))
            errMsg += "sure that you haven't made any changes to it"
            raise SqlmapInstallationException, errMsg

        root = doc.getroot()
        parseXmlNode(root)
```
其中payloadFilePath为：
```
E:\sqlmap\xml\payloads\boolean_blind.xml
E:\sqlmap\xml\payloads\error_based.xml
E:\sqlmap\xml\payloads\inline_query.xml
E:\sqlmap\xml\payloads\stacked_queries.xml
E:\sqlmap\xml\payloads\time_blind.xml
E:\sqlmap\xml\payloads\union_query.xml
```

## 更新版本 ##

如果检测到--update，那么进入update模式`update()`

挺有趣的，可以看看
```
success = False

    if not os.path.exists(os.path.join(paths.SQLMAP_ROOT_PATH, ".git")):
        errMsg = "not a git repository. Please checkout the 'sqlmapproject/sqlmap' repository "
        errMsg += "from GitHub (e.g. 'git clone https://github.com/sqlmapproject/sqlmap.git sqlmap')"
        logger.error(errMsg)
    else:
        infoMsg = "updating sqlmap to the latest development version from the "
        infoMsg += "GitHub repository"
        logger.info(infoMsg)

        debugMsg = "sqlmap will try to update itself using 'git' command"
        logger.debug(debugMsg)

        dataToStdout("\r[%s] [INFO] update in progress " % time.strftime("%X"))

        try:
            process = execute("git checkout . && git pull %s HEAD" % GIT_REPOSITORY, shell=True, stdout=PIPE, stderr=PIPE, cwd=paths.SQLMAP_ROOT_PATH.encode(locale.getpreferredencoding()))  # Reference: http://blog.stastnarodina.com/honza-en/spot/python-unicodeencodeerror/
            pollProcess(process, True)
            stdout, stderr = process.communicate()
            success = not process.returncode
        except (IOError, OSError), ex:
            success = False
            stderr = getSafeExString(ex)

        if success:
            import lib.core.settings
            _ = lib.core.settings.REVISION = getRevisionNumber()
            logger.info("%s the latest revision '%s'" % ("already at" if "Already" in stdout else "updated to", _))
        else:
            if "Not a git repository" in stderr:
                errMsg = "not a valid git repository. Please checkout the 'sqlmapproject/sqlmap' repository "
                errMsg += "from GitHub (e.g. 'git clone https://github.com/sqlmapproject/sqlmap.git sqlmap')"
                logger.error(errMsg)
            else:
                logger.error("update could not be completed ('%s')" % re.sub(r"\W+", " ", stderr).strip())

    if not success:
        if IS_WIN:
            infoMsg = "for Windows platform it's recommended "
            infoMsg += "to use a GitHub for Windows client for updating "
            infoMsg += "purposes (http://windows.github.com/) or just "
            infoMsg += "download the latest snapshot from "
            infoMsg += "https://github.com/sqlmapproject/sqlmap/downloads"
        else:
            infoMsg = "for Linux platform it's required "
            infoMsg += "to install a standard 'git' package (e.g.: 'sudo apt-get install git')"

        logger.info(infoMsg)

```

## 加载常用的查询语句 ##

最后也是初始化的核心，加载查询语句，针对不同数据的基础数据查询
```
def _loadQueries():
    """
    Loads queries from 'xml/queries.xml' file.
    """

    def iterate(node, retVal=None):
        class DictObject(object):
            def __init__(self):
                self.__dict__ = {}

            def __contains__(self, name):
                return name in self.__dict__

        if retVal is None:
            retVal = DictObject()

        for child in node.findall("*"):
            instance = DictObject()
            retVal.__dict__[child.tag] = instance
            if child.attrib:
                instance.__dict__.update(child.attrib)
            else:
                iterate(child, instance)

        return retVal

    tree = ElementTree()
    try:
        tree.parse(paths.QUERIES_XML)
    except Exception, ex:
        errMsg = "something appears to be wrong with "
        errMsg += "the file '%s' ('%s'). Please make " % (paths.QUERIES_XML, getSafeExString(ex))
        errMsg += "sure that you haven't made any changes to it"
        raise SqlmapInstallationException, errMsg

    for node in tree.findall("*"):
        queries[node.attrib['value']] = iterate(node)
```

xml/queries.xml就是各个语句的字典

# 开始 #

基本初始化完成后，就正式进入了注入测试中
`start()`