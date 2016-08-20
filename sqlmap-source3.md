---
title: sqlmap 源码分析（三）在注入之前
date: 2016-08-16 15:19:19
tags:
- python
- sqlmap
categories:
- Blogs
---


sqlmap是web狗永远也绕不过去的神器，为了能自由的使用sqlmap，阅读源码还是有必要的...

<!--more-->

# 开始 #

在初始化完成后，就进入了正式的测试环节
`start()`

# 直连数据库方式 #

## 初始化目标环境 ##

没什么特殊的，有一部分是设置了urlencode过的post参数

```
if conf.data:
    class _(unicode):
        pass

    kb.postUrlEncode = True

    for key, value in conf.httpHeaders:
        if key.upper() == HTTP_HEADER.CONTENT_TYPE.upper():
            kb.postUrlEncode = "urlencoded" in value
            break

    if kb.postUrlEncode:
        original = conf.data
        conf.data = _(urldecode(conf.data))
        setattr(conf.data, UNENCODED_ORIGINAL_VALUE, original)
        kb.postSpaceToPlus = '+' in original
```

# 创建输出的储存文件 #

```
def setupTargetEnv():
	_createTargetDirs()
	_setRequestParams()
	_setHashDB()
	_resumeHashDBValues()
	_setResultsFile()
	_setAuthCred()

```

## 创建文件 ##

`_createTargetDirs()`

检查并创建文件作为输出目录，如果出现系统错误，爆出相应的警告
```
try:
    if not os.path.isdir(paths.SQLMAP_OUTPUT_PATH):
        os.makedirs(paths.SQLMAP_OUTPUT_PATH, 0755)

    _ = os.path.join(paths.SQLMAP_OUTPUT_PATH, randomStr())
    open(_, "w+b").close()
    os.remove(_)

    if conf.outputDir:
        warnMsg = "using '%s' as the output directory" % paths.SQLMAP_OUTPUT_PATH
        logger.warn(warnMsg)
except (OSError, IOError), ex:
        try:
            tempDir = tempfile.mkdtemp(prefix="sqlmapoutput")
        except Exception, _:
            errMsg = "unable to write to the temporary directory ('%s'). " % _
            errMsg += "Please make sure that your disk is not full and "
            errMsg += "that you have sufficient write permissions to "
            errMsg += "create temporary files and/or directories"
            raise SqlmapSystemException(errMsg)
```

## 写入基本信息 ##

```
try:
    with codecs.open(os.path.join(conf.outputPath, "target.txt"), "w+", UNICODE_ENCODING) as f:
        f.write(kb.originalUrls.get(conf.url) or conf.url or conf.hostname)
        f.write(" (%s)" % (HTTPMETHOD.POST if conf.data else HTTPMETHOD.GET))
        if conf.data:
            f.write("\n\n%s" % getUnicode(conf.data))
except IOError, ex:
    if "denied" in getUnicode(ex):
        errMsg = "you don't have enough permissions "
    else:
        errMsg = "something went wrong while trying "
    errMsg += "to write to the output directory '%s' (%s)" % (paths.SQLMAP_OUTPUT_PATH, getSafeExString(ex))

    raise SqlmapMissingPrivileges(errMsg)

```

## 创建dump文件夹 ##

```
def _createDumpDir():
    """
    Create the dump directory.
    """

    if not conf.dumpTable and not conf.dumpAll and not conf.search:
        return

    conf.dumpPath = paths.SQLMAP_DUMP_PATH % conf.hostname

    if not os.path.isdir(conf.dumpPath):
        try:
            os.makedirs(conf.dumpPath, 0755)
        except OSError, ex:
            tempDir = tempfile.mkdtemp(prefix="sqlmapdump")
            warnMsg = "unable to create dump directory "
            warnMsg += "'%s' (%s). " % (conf.dumpPath, getUnicode(ex))
            warnMsg += "Using temporary directory '%s' instead" % tempDir
            logger.warn(warnMsg)

            conf.dumpPath = tempDir
```

## 创建文件文件夹 ##
```
def _createFilesDir():
    """
    Create the file directory.
    """

    if not conf.rFile:
        return

    conf.filePath = paths.SQLMAP_FILES_PATH % conf.hostname

    if not os.path.isdir(conf.filePath):
        try:
            os.makedirs(conf.filePath, 0755)
        except OSError, ex:
            tempDir = tempfile.mkdtemp(prefix="sqlmapfiles")
            warnMsg = "unable to create files directory "
            warnMsg += "'%s' (%s). " % (conf.filePath, getUnicode(ex))
            warnMsg += "Using temporary directory '%s' instead" % tempDir
            logger.warn(warnMsg)

            conf.filePath = tempDir
```

# 检查post data中的参数 #

`_setRequestParams()`

## 检查参数 ##
这里检查post中的所有参数

执行对参数的检查,其中parameters就是get的参数，conf.data则是post的参数
```
# Perform checks on GET parameters
if conf.parameters.get(PLACE.GET):
    parameters = conf.parameters[PLACE.GET]
    paramDict = paramToDict(PLACE.GET, parameters)

    if paramDict:
        conf.paramDict[PLACE.GET] = paramDict
        testableParameters = True

# Perform checks on POST parameters
if conf.method == HTTPMETHOD.POST and conf.data is None:
    logger.warn("detected empty POST body")
    conf.data = ""
```

## 判断是不是存在注入标志位 ##

在sqlmap中*号是为手动标志的注入位置，这里的CUSTOM_INJECTION_MARK_CHAR就是星号

```
def process(match, repl):
    retVal = match.group(0)

    if not (conf.testParameter and match.group("name") not in conf.testParameter):
        retVal = repl
        while True:
            _ = re.search(r"\\g<([^>]+)>", retVal)
            if _:
                retVal = retVal.replace(_.group(0), match.group(int(_.group(1)) if _.group(1).isdigit() else _.group(1)))
            else:
                break
        if CUSTOM_INJECTION_MARK_CHAR in retVal:
            hintNames.append((retVal.split(CUSTOM_INJECTION_MARK_CHAR)[0], match.group("name")))
    return retVal

if kb.processUserMarks is None and CUSTOM_INJECTION_MARK_CHAR in conf.data:
    message = "custom injection marking character ('%s') found in option " % CUSTOM_INJECTION_MARK_CHAR
    message += "'--data'. Do you want to process it? [Y/n/q] "
    test = readInput(message, default="Y")
    if test and test[0] in ("q", "Q"):
        raise SqlmapUserQuitException
    else:
        kb.processUserMarks = not test or test[0] not in ("n", "N")

        if kb.processUserMarks:
            kb.testOnlyCustom = True
```

## JSON数据处理 ##

```
if not (kb.processUserMarks and CUSTOM_INJECTION_MARK_CHAR in conf.data):
	if re.search(JSON_RECOGNITION_REGEX, conf.data):
	    message = "JSON data found in %s data. " % conf.method
	    message += "Do you want to process it? [Y/n/q] "
	    test = readInput(message, default="Y")
	    if test and test[0] in ("q", "Q"):
	        raise SqlmapUserQuitException
	    elif test[0] not in ("n", "N"):
	        conf.data = getattr(conf.data, UNENCODED_ORIGINAL_VALUE, conf.data)
	        conf.data = conf.data.replace(CUSTOM_INJECTION_MARK_CHAR, ASTERISK_MARKER)
	        conf.data = re.sub(r'("(?P<name>[^"]+)"\s*:\s*"[^"]+)"', functools.partial(process, repl=r'\g<1>%s"' % CUSTOM_INJECTION_MARK_CHAR), conf.data)
	        conf.data = re.sub(r'("(?P<name>[^"]+)"\s*:\s*)(-?\d[\d\.]*\b)', functools.partial(process, repl=r'\g<0>%s' % CUSTOM_INJECTION_MARK_CHAR), conf.data)
	        match = re.search(r'(?P<name>[^"]+)"\s*:\s*\[([^\]]+)\]', conf.data)
	        if match and not (conf.testParameter and match.group("name") not in conf.testParameter):
	            _ = match.group(2)
	            _ = re.sub(r'("[^"]+)"', '\g<1>%s"' % CUSTOM_INJECTION_MARK_CHAR, _)
	            _ = re.sub(r'(\A|,|\s+)(-?\d[\d\.]*\b)', '\g<0>%s' % CUSTOM_INJECTION_MARK_CHAR, _)
	            conf.data = conf.data.replace(match.group(0), match.group(0).replace(match.group(2), _))
	        kb.postHint = POST_HINT.JSON
	
	elif re.search(JSON_LIKE_RECOGNITION_REGEX, conf.data):
	    message = "JSON-like data found in %s data. " % conf.method
	    message += "Do you want to process it? [Y/n/q] "
	    test = readInput(message, default="Y")
	    if test and test[0] in ("q", "Q"):
	        raise SqlmapUserQuitException
	    elif test[0] not in ("n", "N"):
	        conf.data = getattr(conf.data, UNENCODED_ORIGINAL_VALUE, conf.data)
	        conf.data = conf.data.replace(CUSTOM_INJECTION_MARK_CHAR, ASTERISK_MARKER)
	        conf.data = re.sub(r"('(?P<name>[^']+)'\s*:\s*'[^']+)'", functools.partial(process, repl=r"\g<1>%s'" % CUSTOM_INJECTION_MARK_CHAR), conf.data)
	        conf.data = re.sub(r"('(?P<name>[^']+)'\s*:\s*)(-?\d[\d\.]*\b)", functools.partial(process, repl=r"\g<0>%s" % CUSTOM_INJECTION_MARK_CHAR), conf.data)
	        kb.postHint = POST_HINT.JSON_LIKE

```

## 数组数据处理 ##


```
elif re.search(ARRAY_LIKE_RECOGNITION_REGEX, conf.data):
    message = "Array-like data found in %s data. " % conf.method
    message += "Do you want to process it? [Y/n/q] "
    test = readInput(message, default="Y")
    if test and test[0] in ("q", "Q"):
        raise SqlmapUserQuitException
    elif test[0] not in ("n", "N"):
        conf.data = conf.data.replace(CUSTOM_INJECTION_MARK_CHAR, ASTERISK_MARKER)
        conf.data = re.sub(r"(=[^%s]+)" % DEFAULT_GET_POST_DELIMITER, r"\g<1>%s" % CUSTOM_INJECTION_MARK_CHAR, conf.data)
        kb.postHint = POST_HINT.ARRAY_LIKE
```

## SOAP/XML数据处理 ##

```
elif re.search(XML_RECOGNITION_REGEX, conf.data):
    message = "SOAP/XML data found in %s data. " % conf.method
    message += "Do you want to process it? [Y/n/q] "
    test = readInput(message, default="Y")
    if test and test[0] in ("q", "Q"):
        raise SqlmapUserQuitException
    elif test[0] not in ("n", "N"):
        conf.data = getattr(conf.data, UNENCODED_ORIGINAL_VALUE, conf.data)
        conf.data = conf.data.replace(CUSTOM_INJECTION_MARK_CHAR, ASTERISK_MARKER)
        conf.data = re.sub(r"(<(?P<name>[^>]+)( [^<]*)?>)([^<]+)(</\2)", functools.partial(process, repl=r"\g<1>\g<4>%s\g<5>" % CUSTOM_INJECTION_MARK_CHAR), conf.data)
        kb.postHint = POST_HINT.SOAP if "soap" in conf.data.lower() else POST_HINT.XML
```

## Multipart-like数据处理 ##

```
elif re.search(MULTIPART_RECOGNITION_REGEX, conf.data):
    message = "Multipart-like data found in %s data. " % conf.method
    message += "Do you want to process it? [Y/n/q] "
    test = readInput(message, default="Y")
    if test and test[0] in ("q", "Q"):
        raise SqlmapUserQuitException
    elif test[0] not in ("n", "N"):
        conf.data = getattr(conf.data, UNENCODED_ORIGINAL_VALUE, conf.data)
        conf.data = conf.data.replace(CUSTOM_INJECTION_MARK_CHAR, ASTERISK_MARKER)
        conf.data = re.sub(r"(?si)((Content-Disposition[^\n]+?name\s*=\s*[\"'](?P<name>[^\n]+?)[\"']).+?)(((\r)?\n)+--)", functools.partial(process, repl=r"\g<1>%s\g<4>" % CUSTOM_INJECTION_MARK_CHAR), conf.data)
        kb.postHint = POST_HINT.MULTIPART
```

后面还有各种处理，包括cookie，请求头的处理

```
# Perform checks on Cookie parameters
if conf.cookie:
conf.parameters[PLACE.COOKIE] = conf.cookie
paramDict = paramToDict(PLACE.COOKIE, conf.cookie)

if paramDict:
    conf.paramDict[PLACE.COOKIE] = paramDict
    testableParameters = True

# Perform checks on header values
if conf.httpHeaders:
for httpHeader, headerValue in list(conf.httpHeaders):
    # Url encoding of the header values should be avoided
    # Reference: http://stackoverflow.com/questions/5085904/is-ok-to-urlencode-the-value-in-headerlocation-value

    if httpHeader.title() == HTTP_HEADER.USER_AGENT:
        conf.parameters[PLACE.USER_AGENT] = urldecode(headerValue)

        condition = any((not conf.testParameter, intersect(conf.testParameter, USER_AGENT_ALIASES, True)))

        if condition:
            conf.paramDict[PLACE.USER_AGENT] = {PLACE.USER_AGENT: headerValue}
            testableParameters = True

    elif httpHeader.title() == HTTP_HEADER.REFERER:
        conf.parameters[PLACE.REFERER] = urldecode(headerValue)

        condition = any((not conf.testParameter, intersect(conf.testParameter, REFERER_ALIASES, True)))

        if condition:
            conf.paramDict[PLACE.REFERER] = {PLACE.REFERER: headerValue}
            testableParameters = True

    elif httpHeader.title() == HTTP_HEADER.HOST:
        conf.parameters[PLACE.HOST] = urldecode(headerValue)

        condition = any((not conf.testParameter, intersect(conf.testParameter, HOST_ALIASES, True)))

        if condition:
            conf.paramDict[PLACE.HOST] = {PLACE.HOST: headerValue}
            testableParameters = True

    else:
        condition = intersect(conf.testParameter, [httpHeader], True)

        if condition:
            conf.parameters[PLACE.CUSTOM_HEADER] = str(conf.httpHeaders)
            conf.paramDict[PLACE.CUSTOM_HEADER] = {httpHeader: "%s,%s%s" % (httpHeader, headerValue, CUSTOM_INJECTION_MARK_CHAR)}
            conf.httpHeaders = [(header, value.replace(CUSTOM_INJECTION_MARK_CHAR, "")) for header, value in conf.httpHeaders]
            testableParameters = True
```

还有很多错误处理，就没必要贴了

## crsf token的处理 ##

```
if conf.csrfToken:
    if not any(conf.csrfToken in _ for _ in (conf.paramDict.get(PLACE.GET, {}), conf.paramDict.get(PLACE.POST, {}))) and not re.search(r"\b%s\b" % re.escape(conf.csrfToken), conf.data or "") and not conf.csrfToken in set(_[0].lower() for _ in conf.httpHeaders) and not conf.csrfToken in conf.paramDict.get(PLACE.COOKIE, {}):
        errMsg = "anti-CSRF token parameter '%s' not " % conf.csrfToken
        errMsg += "found in provided GET, POST, Cookie or header values"
        raise SqlmapGenericException(errMsg)
else:
    for place in (PLACE.GET, PLACE.POST, PLACE.COOKIE):
        for parameter in conf.paramDict.get(place, {}):
            if any(parameter.lower().count(_) for _ in CSRF_TOKEN_PARAMETER_INFIXES):
                message = "%s parameter '%s' appears to hold anti-CSRF token. " % (place, parameter)
                message += "Do you want sqlmap to automatically update it in further requests? [y/N] "
                test = readInput(message, default="N")
                if test and test[0] in ("y", "Y"):
                    conf.csrfToken = parameter
                break
```

# 设置储存的数据库 #

sqlmap默认使用的是sqlite
```
def _setHashDB():
    """
    Check and set the HashDB SQLite file for query resume functionality.
    """

    if not conf.hashDBFile:
        conf.hashDBFile = conf.sessionFile or os.path.join(conf.outputPath, "session.sqlite")

    if os.path.exists(conf.hashDBFile):
        if conf.flushSession:
            try:
                os.remove(conf.hashDBFile)
                logger.info("flushing session file")
            except OSError, msg:
                errMsg = "unable to flush the session file (%s)" % msg
                raise SqlmapFilePathException(errMsg)

    conf.hashDB = HashDB(conf.hashDBFile)
```

## 储存基本信息到数据库 ##

```
def _resumeHashDBValues():
    """
    Resume stored data values from HashDB
    """

    kb.absFilePaths = hashDBRetrieve(HASHDB_KEYS.KB_ABS_FILE_PATHS, True) or kb.absFilePaths
    kb.brute.tables = hashDBRetrieve(HASHDB_KEYS.KB_BRUTE_TABLES, True) or kb.brute.tables
    kb.brute.columns = hashDBRetrieve(HASHDB_KEYS.KB_BRUTE_COLUMNS, True) or kb.brute.columns
    kb.chars = hashDBRetrieve(HASHDB_KEYS.KB_CHARS, True) or kb.chars
    kb.dynamicMarkings = hashDBRetrieve(HASHDB_KEYS.KB_DYNAMIC_MARKINGS, True) or kb.dynamicMarkings
    kb.xpCmdshellAvailable = hashDBRetrieve(HASHDB_KEYS.KB_XP_CMDSHELL_AVAILABLE) or kb.xpCmdshellAvailable

    kb.errorChunkLength = hashDBRetrieve(HASHDB_KEYS.KB_ERROR_CHUNK_LENGTH)
    if kb.errorChunkLength and kb.errorChunkLength.isdigit():
        kb.errorChunkLength = int(kb.errorChunkLength)
    else:
        kb.errorChunkLength = None

    conf.tmpPath = conf.tmpPath or hashDBRetrieve(HASHDB_KEYS.CONF_TMP_PATH)

    for injection in hashDBRetrieve(HASHDB_KEYS.KB_INJECTIONS, True) or []:
        if isinstance(injection, InjectionDict) and injection.place in conf.paramDict and \
            injection.parameter in conf.paramDict[injection.place]:

            if not conf.tech or intersect(conf.tech, injection.data.keys()):
                if intersect(conf.tech, injection.data.keys()):
                    injection.data = dict(filter(lambda (key, item): key in conf.tech, injection.data.items()))

                if injection not in kb.injections:
                    kb.injections.append(injection)

    _resumeDBMS()
    _resumeOS()
```

## 储存数据库信息到数据库 ##

```
def _resumeDBMS():
    """
    Resume stored DBMS information from HashDB
    """

    value = hashDBRetrieve(HASHDB_KEYS.DBMS)

    if not value:
        return

    dbms = value.lower()
    dbmsVersion = [UNKNOWN_DBMS_VERSION]
    _ = "(%s)" % ("|".join([alias for alias in SUPPORTED_DBMS]))
    _ = re.search(r"\A%s (.*)" % _, dbms, re.I)

    if _:
        dbms = _.group(1).lower()
        dbmsVersion = [_.group(2)]

    if conf.dbms:
        check = True
        for aliases, _, _, _ in DBMS_DICT.values():
            if conf.dbms.lower() in aliases and dbms not in aliases:
                check = False
                break

        if not check:
            message = "you provided '%s' as a back-end DBMS, " % conf.dbms
            message += "but from a past scan information on the target URL "
            message += "sqlmap assumes the back-end DBMS is '%s'. " % dbms
            message += "Do you really want to force the back-end "
            message += "DBMS value? [y/N] "
            test = readInput(message, default="N")

            if not test or test[0] in ("n", "N"):
                conf.dbms = None
                Backend.setDbms(dbms)
                Backend.setVersionList(dbmsVersion)
    else:
        infoMsg = "resuming back-end DBMS '%s' " % dbms
        logger.info(infoMsg)

        Backend.setDbms(dbms)
        Backend.setVersionList(dbmsVersion)
```

## 储存操作系统信息到数据库 ##

```
def _resumeOS():
    """
    Resume stored OS information from HashDB
    """

    value = hashDBRetrieve(HASHDB_KEYS.OS)

    if not value:
        return

    os = value

    if os and os != 'None':
        infoMsg = "resuming back-end DBMS operating system '%s' " % os
        logger.info(infoMsg)

        if conf.os and conf.os.lower() != os.lower():
            message = "you provided '%s' as back-end DBMS operating " % conf.os
            message += "system, but from a past scan information on the "
            message += "target URL sqlmap assumes the back-end DBMS "
            message += "operating system is %s. " % os
            message += "Do you really want to force the back-end DBMS "
            message += "OS value? [y/N] "
            test = readInput(message, default="N")

            if not test or test[0] in ("n", "N"):
                conf.os = os
        else:
            conf.os = os

        Backend.setOs(conf.os)
```

## 添加身份验证 ##

如果有的话，添加身份验证。
```
def _setAuthCred():
    """
    Adds authentication credentials (if any) for current target to the password manager
    (used by connection handler)
    """

    if kb.passwordMgr and all(_ is not None for _ in (conf.scheme, conf.hostname, conf.port, conf.authUsername, conf.authPassword)):
        kb.passwordMgr.add_password(None, "%s://%s:%d" % (conf.scheme, conf.hostname, conf.port), conf.authUsername, conf.authPassword)

```

# 直连数据库方式开始注入 #


开始注入，注入过程和普通相同，所以稍后在研究。


# sss #

## 处理目标参数 ##

在开始之前，处理
```
if conf.url and not any((conf.forms, conf.crawlDepth)):
    kb.targets.add((conf.url, conf.method, conf.data, conf.cookie, None))
```

## config文件 ##

```
if conf.configFile and not kb.targets:
	errMsg = "you did not edit the configuration file properly, set "
	errMsg += "the target URL, list of targets or google dork"
	logger.error(errMsg)
	return False
```

## 多目标处理 ##

```
if kb.targets and len(kb.targets) > 1:
    infoMsg = "sqlmap got a total of %d targets" % len(kb.targets)
    logger.info(infoMsg)
```

# 循环解包开始注入 #

```
for targetUrl, targetMethod, targetData, targetCookie, targetHeaders in kb.targets:
    try:
        conf.url = targetUrl
        conf.method = targetMethod.upper() if targetMethod else targetMethod
        conf.data = targetData
        conf.cookie = targetCookie
        conf.httpHeaders = list(initialHeaders)
        conf.httpHeaders.extend(targetHeaders or [])

```

# 处理参数 #

这里分GET和POST两种处理方式，
```
if PLACE.GET in conf.parameters and not any([conf.data, conf.testParameter]):
    for parameter in re.findall(r"([^=]+)=([^%s]+%s?|\Z)" % (re.escape(conf.paramDel or "") or DEFAULT_GET_POST_DELIMITER, re.escape(conf.paramDel or "") or DEFAULT_GET_POST_DELIMITER), conf.parameters[PLACE.GET]):
        paramKey = (conf.hostname, conf.path, PLACE.GET, parameter[0])
        print paramKey
        if paramKey not in kb.testedParams:
            testSqlInj = True
            break
else:
    paramKey = (conf.hostname, conf.path, None, None)
    if paramKey not in kb.testedParams:
        testSqlInj = True

```
比如传入
```
 python .\sqlmap.py -u http://demo.lorexxar.pw/get.php?user=user1
```

就获得
```
(u'demo.lorexxar.pw', u'/get.php', 'GET', u'user')
```

# check waf #

然后开始检查waf
## check waf ##

这里的check逻辑算法来自**Reference: http://seclists.org/nmap-dev/2011/q2/att-1005/http-waf-detect.nse**

通过随机数加payload构造最终payload
```
payload = "%d %s" % (randomInt(), IDS_WAF_CHECK_PAYLOAD)
```
这里IDS_WAF_CHECK_PAYLOAD就是上面来源上的payload
```
AND 1=1 UNION ALL SELECT 1,NULL,'<script>alert("XSS")</script>',table_name FROM information_schema.tables WHERE 2>1--/**/; EXEC xp_cmdshell('cat ../../../etc/passwd')#
```

拼接请求
```
value = "" if not conf.parameters.get(PLACE.GET) else conf.parameters[PLACE.GET] + DEFAULT_GET_POST_DELIMITER
    value += agent.addPayloadDelimiters("%s=%s" % (randomStr(), payload))
```

添加延迟
```
pushValue(conf.timeout)
conf.timeout = IDS_WAF_CHECK_TIMEOUT
```

发起请求
```
try:
    retVal = Request.queryPage(place=PLACE.GET, value=value, getRatioValue=True, noteResponseTime=False, silent=True)[1] < IDS_WAF_CHECK_RATIO
except SqlmapConnectionException:
    retVal = True
finally:
    kb.matchRatio = None
    conf.timeout = popValue()
```

## 识别waf ##

预先加载函数

```
if not kb.wafFunctions:
    setWafFunctions()
```
这里包含的waf检测规则非常复杂，sqlmap在这方面还是做的比较好

循环检测

```
for function, product in kb.wafFunctions:
    try:
        logger.debug("checking for WAF/IDS/IPS product '%s'" % product)
        found = function(_)
    except Exception, ex:
        errMsg = "exception occurred while running "
        errMsg += "WAF script for '%s' ('%s')" % (product, getSafeExString(ex))
        logger.critical(errMsg)

        found = False

    if found:
        retVal = product
        break
```

相应的显示
```
if retVal:
    errMsg = "WAF/IDS/IPS identified as '%s'. Please " % retVal
    errMsg += "consider usage of tamper scripts (option '--tamper')"
    logger.critical(errMsg)

    message = "are you sure that you want to "
    message += "continue with further target testing? [y/N] "
    output = readInput(message, default="N")

    if output and output[0] not in ("Y", "y"):
        raise SqlmapUserQuitException
else:
    warnMsg = "WAF/IDS/IPS product hasn't been identified"
    logger.warn(warnMsg)
```

# 测试空连接 #

测试空连接的思路来自**Reference: http://www.wisec.it/sectou.php?id=472f952d79293**

```
try:
    pushValue(kb.pageCompress)
    kb.pageCompress = False

    page, headers, _ = Request.getPage(method=HTTPMETHOD.HEAD)

    if not page and HTTP_HEADER.CONTENT_LENGTH in (headers or {}):
        kb.nullConnection = NULLCONNECTION.HEAD

        infoMsg = "NULL connection is supported with HEAD header"
        logger.info(infoMsg)
    else:
        page, headers, _ = Request.getPage(auxHeaders={HTTP_HEADER.RANGE: "bytes=-1"})

        if page and len(page) == 1 and HTTP_HEADER.CONTENT_RANGE in (headers or {}):
            kb.nullConnection = NULLCONNECTION.RANGE

            infoMsg = "NULL connection is supported with GET header "
            infoMsg += "'%s'" % kb.nullConnection
            logger.info(infoMsg)
        else:
            _, headers, _ = Request.getPage(skipRead = True)

            if HTTP_HEADER.CONTENT_LENGTH in (headers or {}):
                kb.nullConnection = NULLCONNECTION.SKIP_READ

                infoMsg = "NULL connection is supported with 'skip-read' method"
                logger.info(infoMsg)

except SqlmapConnectionException, ex:
    errMsg = getSafeExString(ex)
    raise SqlmapConnectionException(errMsg)

finally:
    kb.pageCompress = popValue()
```

# 开始注入逻辑 #

从这里开始，终于开始真正的注入逻辑了。