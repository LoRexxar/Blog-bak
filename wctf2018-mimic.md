---
title: wctf2018 cyber mimic defence Writeup
date: 2018-07-13 10:51:39
tags:
- mimic
- flask
---


今年有幸作为新人赛中的一员参加了Wctf2018大师赛，比较难过的是，由于Wctf本身使用战争与分享赛制，却要求了每队必须出一道windows题目，大部分人都选择了内核驱动级别的re和pwn，只有LCBC出的“拟态防御”和智能合约审计可以一做，关于智能合约的部分有机会会再分享，这里只研究一下mimic这题。

<!--more-->

# cyber mimic defence #

代码挺简单的，flask完成，主要的功能几乎只有登陆注册，功能核心基本都在user类中，而调用到user类的view只有登陆部分，所以漏洞也就是在这里。

```
views.py

# ~*~ coding: utf-8 ~*~
import flask_login as login
import flask_admin as admin
import json
import os
import time

from collections import defaultdict
from datetime import datetime
from flask_admin import helpers, expose
from flask import redirect, url_for, request, render_template
from flask import jsonify

from datetime import timedelta
from flask import make_response, request, current_app
from functools import update_wrapper

from loginform import LoginForm
import stub as stub



class AdminIndexView(admin.AdminIndexView):
    
    def _stubs(self):
        self.nav = {
            "tasks" : stub.get_tasks(),
            "messages" : stub.get_messages_summary(),
            "alerts" : stub.get_alerts()
        }
        
        (cols, rows) = stub.get_adv_tables()
        (scols, srows, context) = stub.get_tables()
        
        self.tables = {
            "advtables" : { "columns" : cols, "rows" : rows },
            "table" : { "columns" : scols, "rows" : srows, "context" : context}
        }
        
        self.panelswells = {
            "accordion" : stub.get_accordion_items(),
            "tabitems" : stub.get_tab_items()
        }
            
    @expose('/')
    def index(self):
        # if not login.current_user.is_authenticated:
        #     return redirect(url_for('.login_view'))
            
        self._stubs()
        self.header = "Dashboard"
     
        # login.current_user.query("EXEC sp_logEvent 'View at %s', 'dashboard', 'visit';" % time.time(), 'mssql')
        print request.args.get('page', 'dashboard')
        print os.path.basename(request.args.get('page', 'dashboard'))

        page = os.path.basename(request.args.get('page', 'dashboard'))
        return render_template('sb-admin/pages/%s.html' % page, admin_view=self)
    
    @expose('/blank')
    def blank(self):        
        # if not login.current_user.is_authenticated:
        #     return redirect(url_for('.login_view'))
            
        self._stubs()    
        self.header = "Blank"
        return render_template('sb-admin/pages/blank.html', admin_view=self)

    @expose('/login/', methods=('GET', 'POST'))
    def login_view(self):
        form = LoginForm(request.form)
        if helpers.validate_form_on_submit(form):
            user = form.get_user()
            login.login_user(user)

        if login.current_user.is_authenticated:
            return redirect(url_for('.index'))
        self._template_args['form'] = form
        return render_template('sb-admin/pages/login.html', form=form)

    @expose('/logout/')
    def logout_view(self):
        login.logout_user()
        return redirect(url_for('.index'))


class BlankView(admin.BaseView):
    @expose('/')
    def index(self):
        return render_template('sb-admin/pages/blank.html', admin_view=self)

```
接着我再贴上user类
```
# ~*~ coding: utf-8 ~*~
from config import *

from collections import Counter
from random import choice

try:
    from flask.ext.login import UserMixin
except:
    from flask_login import UserMixin

class UserNotFoundError(Exception):
    pass

class User(UserMixin):

    id = None
    password = None

    def is_active(self):
        return True

    def query(self, query, driver):
        try:
            conn = self.DB_CONNECTIONS[driver]
            c = conn.cursor()
            c.execute(query)
            r = tuple(c.fetchall())
            return r
        except Exception, e:
            return ()

    def find_user(self, username, driver):

        TERMINAL_TOKENS = {
            'psql': ["'", '$$'],
            'mssql': ["'"],
            'mysql': ["'", '"'],
            'sqlite': ["'", '"']
        }

        quote = choice(TERMINAL_TOKENS.get(driver, ["'", '"']))
        query = '''select * from users where username=%s%s%s;''' % (quote, username, quote)

        # select * from users where username='1' and 1=0 union select '1','1','1234'
        return self.query(query, driver)

    def __init__(self, username):
        self.DB_CONNECTIONS = {
              'mssql': pymssql.connect('127.0.0.1', '', '', ''),
              'mysql': MySQLdb.connect(host='localhost', user='', passwd='', db=''),
              'psql': psycopg2.connect("dbname='' user='' host='localhost' password=''"),
              'sqlite': sqlite3.connect(os.path.dirname(os.path.realpath(__file__)) + '/../X.sqlite3'),
        }
        result = [self.find_user(username, driver) for driver in self.DB_CONNECTIONS]
        common = Counter(result).most_common()[0]
        user = () if common[1] < len(result) - 1 else common[0]
        if not user:
            raise UserNotFoundError()
        self._id = user[0][0]
        self.username = user[0][1]
        self.id = self.username 
        self.password = user[0][2]

    @classmethod
    def get(self_class, username):
        try:
            return self_class(username)
        except UserNotFoundError:
            return None

```

我们很容易看到，在user类中，对查询语句直接做了拼接
```
query = '''select * from users where username=%s%s%s;''' % (quote, username, quote)
```
很明显的注入，但问题在于，这里LCBC加入了所谓的拟态防御，代码如下：

```

def find_user(self, username, driver):

        TERMINAL_TOKENS = {
            'psql': ["'", '$$'],
            'mssql': ["'"],
            'mysql': ["'", '"'],
            'sqlite': ["'", '"']
        }

        quote = choice(TERMINAL_TOKENS.get(driver, ["'", '"']))
        query = '''select * from users where username=%s%s%s;''' % (quote, username, quote)
```
后端使用了4种数据库，然后不同的数据库会对应不同的闭合符号，在每次查询时都会向4个数据库同时查询，然后对比返回结果，只有3种以上相同的结果才会被返回。

```
result = [self.find_user(username, driver) for driver in self.DB_CONNECTIONS]
common = Counter(result).most_common()[0]
user = () if common[1] < len(result) - 1 else common[0]
if not user:
    raise UserNotFoundError()
self._id = user[0][0]
self.username = user[0][3]
self.id = self.username 
self.password = user[0][4]
```

或许我们很难找到这种防御方式的弱点，但是我们或许需要换个思路来思考这个问题。

我们有两个办法解决这个问题
1、找到至少3种数据库都支持的查询方式
2、只攻击其中1种数据库

这里我们很难找到支持第一种办法的注入方式，因为在不同的数据库中，储存表名列名字段的都是不同位置，我们最多只能使用最普通的`union select`语法来登陆。

```
username=root' union select 0,'root','e10adc3949ba59abbe56e057f20f883e'--&password=123456
```

用这个语句可以直接登陆，很显然，后台什么都没有。

那么我们果断是由第二种方式，既然我们的每次查询都会进数据库，那么我们直接时间盲注就好了，有个问题在于，比如mysql，我们需要处理单双引号闭合方式不同的问题，当闭合方式不同时，我们就没办法获得数据了。

有两个办法，1是通过精妙的构造来闭合两种引号，也不是很难，就是看着挺难受的，例如
```
username=1' or '"!="'!='' and sleep(2) and ''!='" or sleep(2)!="&password=123456
```

2就是通过if的正确和错误生成不同的延时来判断
```
1' or if(({}), sleep(3), sleep(1))#
```
通过只有有效语句才会sleep

这么一来我们就能注了，很显然的是，数据库里也什么都没有！！！

那让我们重新回到题目进行思考

那么在注入之后的第二步

1、注入拿flag，或者注入读文件拿flag（no）
2、需要注入触发第二个漏洞

需要登陆才能访问的路由包括
```
/blank
/

代码如下
 @expose('/')
    def index(self):
        if not login.current_user.is_authenticated:
            return redirect(url_for('.login_view'))
            
        self._stubs()
        self.header = "Dashboard"
     
        login.current_user.query("EXEC sp_logEvent 'View at %s', 'dashboard', 'visit';" % time.time(), 'mssql')
        page = os.path.basename(request.args.get('page', 'dashboard'))
        return render_template('sb-admin/pages/%s.html' % page, admin_view=self)
    
    @expose('/blank')
    def blank(self):        
        if not login.current_user.is_authenticated:
            return redirect(url_for('.login_view'))
            
        self._stubs()    
        self.header = "Blank"
        return render_template('sb-admin/pages/blank.html', admin_view=self)
```

page这里变量经过了basename的处理，没办法绕过，所以我们只能引入`sb-admin/pages/`下的
`%.html`，按照这个思路思考，我们需要找到一个写入文件的点，然后就可以通过写入模板，构造命令执行getshell！

思来想去也只有注入有可能可以写入文件，所以我们把目光放到其他数据库中，但无一例外地是权限不够，回顾源码的时候，发现了
```
login.current_user.query("EXEC sp_logEvent 'View at %s', 'dashboard', 'visit';" % time.time(), 'mssql')
```

其实当时在比赛的时候也发现这个了，所以一直在研究mssql的EXEC能不能写入文件，因为无法获取返回，所以一直找不到能验证是否成功写入文件的方法，从权限判断，则是没有写文件的权限，当时没想到的是，mssql可以查询存储过程的配置。

```
sp_helptext 'ListBandGenresInternational' # 查看存储过程定义

sp_help band_genres # 查看表结构，也可以查看存储过程的简单信息
```

值得注意的是，因为后端有多种数据库，所以即使我们开着sqlmap扫做各种限制，sqlmap也很难按照我们需要的方式帮我们完成这里的时间盲注（至少我们没成功），所以，如何在有限的时间完成不熟悉的mssql注入脚本并获得那么大的数据，就成了核心问题，这也是这个题目最大的问题！

关于mssql时间盲注可以看这篇文章

[http://drops.xmd5.com/static/drops/tips-8242.html](http://drops.xmd5.com/static/drops/tips-8242.html)

![image.png-9469.8kB][5]

这是现场分享会公布的储存过程，在其中，我们很明显可以看到写入log的目录和储存结构，唯一的问题是，我们需要想办法绕过后缀限制，其实也很好办，因为在数据库中限制了name和type的位数，分别都是40位。

![image.png-1129.1kB][6]

我们通过这种方式注入语句到spWriteupStringToFile中，构造截断就可以写入文件了。

后面的思路很清楚了，写入flask模板，然后用后台的功能引入，执行命令

![image.png-958.5kB][7]

```
page = os.path.basename(request.args.get('page', 'dashboard'))
return render_template('sb-admin/pages/%s.html' % page, admin_view=self)
```

最后请求`?page=x`即可触发


  [5]: http://static.zybuluo.com/LoRexxar/rf98mvtfeiiyu05ltb5oxx0m/image.png
  [6]: http://static.zybuluo.com/LoRexxar/6vlfhi58vrcegnsd6ciuecje/image.png
  [7]: http://static.zybuluo.com/LoRexxar/fb0kn88pavd4ducdv5gxf9l1/image.png
