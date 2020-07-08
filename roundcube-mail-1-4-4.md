---
title: Roundcube mail代码审计笔记
date: 2020-05-29 16:57:00
tags: php 
---

---

以下是一篇不完整的文章，主要记录了在审计过程中的一些记录，在面对这类复杂的代码审计的时候，一旦被打断或者过后重新复习都会花费巨大的代价，所以这次稍微记录了一下结构。



以下笔记适用于 Roundcube mail 1.4.4


<!--more-->

# 代码结构

```
├─bin   // 涉及到更新的相关bash脚本
├─config  //配置文件
├─installer   // 安装目录
├─logs   // 错误日志
├─plugins   // 插件目录
│  ├─acl
│  ├─additional_message_headers
│  ├─archive
│  ├─attachment_reminder
│  ├─autologon
│  ├─database_attachments
│  ├─debug_logger
│  ├─emoticons
│  ├─enigma
│  ├─example_addressbook
│  ├─filesystem_attachments
│  ├─help
│  ├─hide_blockquote
│  ├─http_authentication
│  ├─identicon
│  ├─identity_select
│  ├─jqueryui
│  ├─krb_authentication
│  ├─managesieve
│  ├─markasjunk
│  ├─newmail_notifier
│  ├─new_user_dialog
│  ├─new_user_identity
│  ├─password
│  ├─redundant_attachments
│  ├─show_additional_headers
│  ├─squirrelmail_usercopy
│  ├─subscriptions_option
│  ├─userinfo
│  ├─vcard_attachments
│  ├─virtuser_file
│  ├─virtuser_query
│  └─zipdownload
├─program  // 核心类相关代码
│  ├─include
│  ├─js
│  ├─lib
│  │  └─Roundcube
│  │      ├─cache
│  │      ├─db
│  │      ├─session
│  │      └─spellchecker
│  ├─localization    // 语言文件
│  ├─resources
│  └─steps   // 核心路由文件
│      ├─addressbook
│      ├─mail
│      ├─settings
│      └─utils
├─public_html
├─skins
├─SQL  数据库备份
├─temp
└─vendor   外部引入的包
    ├─bin
    ├─composer
    ├─endroid
    ├─kolab
    ├─masterminds
    ├─pear
    │  ├─auth_sasl
    │  ├─console_commandline
    │  ├─console_getopt
    │  ├─crypt_gpg
    │  ├─mail_mime
    │  ├─net_idna2
    │  ├─net_ldap2
    │  ├─net_sieve
    │  ├─net_smtp
    │  ├─net_socket
    │  ├─pear-core-minimal
    │  └─pear_exception
    └─roundcube

```

在审计roundcube mail的代码过程中，我们可以把目标的重心放在program目录下，其中

include、lib、steps这三个目录分别包含了整个系统最核心的相关代码。

```
├─program  // 核心类相关代码
│  ├─include
│  ├─lib     // 核心类代码
│  └─steps   // 核心路由文件
```

换言之，也就是说，除了steps以外的代码只包含类以及函数定义，并没有实际的调用代码，所以我们的目标关注点主要集中在入口点steps。

# 入口路由

在弄明白roundcube的结构时，首先我们把目标放在路由入口处。

值得注意的是steps中的代码都是`.inc`结尾的，所以我们必须要从入口文件进入才能走到具体的代码部分。

首先我们要关注:
```
index.php
```

## 路由分配

in index.php line 100

```
$RCMAIL->set_task($startup['task']);
$RCMAIL->action = $startup['action'];
```

这里通过task和action做路由表的分配。

```
?_task=utils&_action=text2html
```

直接指向
```
/program/steps/utils/text2html.inc
```

当然，这一切都建立在有权限的情况下，如果没有登陆，则会在

index.php line 217-251

```
$plugin = $RCMAIL->plugins->exec_hook('unauthenticated', array(
        'task'      => 'login',
        'error'     => $session_error,
        // Return 401 only on failed logins (#7010)
        'http_code' => empty($session_error) && !empty($error_message) ? 401 : 200
));

$RCMAIL->set_task($plugin['task']);

if ($plugin['http_code'] == 401) {
    header('HTTP/1.0 401 Unauthorized');
}
$OUTPUT->send($plugin['task']);
```

跳回登录页面

相应的引入路由文件的代码如下

![image.png-91.5kB][1]

在引入每个路由文件之前，还会相应的先引入func.php。

在index.php中，除了基本的路由分配以外，还有一个重要的特性。

## csrf check

in index.php line 254
```
    // CSRF prevention
    $RCMAIL->request_security_check();

```
跟到 program/include/rcmail.php line 961
```
public function request_security_check($mode = rcube_utils::INPUT_POST)
{
    // check request token
    if (!$this->check_request($mode)) {
        $error = array('code' => 403, 'message' => "Request security check failed");
        self::raise_error($error, false, true);
    }
}
```

然后跟入 program/lib/roundcube/rcube.php line 955

```
public function check_request($mode = rcube_utils::INPUT_POST)
{
    // check secure token in URL if enabled
    if ($token = $this->get_secure_url_token()) {
        foreach (explode('/', preg_replace('/[?#&].*$/', '', $_SERVER['REQUEST_URI'])) as $tok) {
            if ($tok == $token) {
                return true;
            }
        }

        $this->request_status = self::REQUEST_ERROR_URL;

        return false;
    }

    $sess_tok = $this->get_request_token();

    // ajax requests
    if (rcube_utils::request_header('X-Roundcube-Request') === $sess_tok) {
        return true;
    }

    // skip empty requests
    if (($mode == rcube_utils::INPUT_POST && empty($_POST))
        || ($mode == rcube_utils::INPUT_GET && empty($_GET))
    ) {
        return true;
    }

    // default method of securing requests
    $token   = rcube_utils::get_input_value('_token', $mode);
    $sess_id = $_COOKIE[ini_get('session.name')];

    if (empty($sess_id) || $token !== $sess_tok) {
        $this->request_status = self::REQUEST_ERROR_TOKEN;
        return false;
    }

    return true;
}
```

可以比较清晰的看到，check request只默认检查POST的token。

你必须保证session id有效，并且token与session中存取的相同
```
if (empty($sess_id) || $token !== $sess_tok) {
    $this->request_status = self::REQUEST_ERROR_TOKEN;
    return false;
}
```

除此之外，ajax还支持把token写在header里
```
// ajax requests
if (rcube_utils::request_header('X-Roundcube-Request') === $sess_tok) {
    return true;
}
```

这个csrf check对安全性的提升是比较巨大的，可以完全防护csrf类漏洞，而且在一定程度上也保护了2次漏洞的发生（如1-click to xxx）

当然他对实际的漏洞没有防护帮助，这个token我们可以在后台的很多地方找到。

# mvc结构

roundcube的MVC结构，出口函数为
```
$OUTPUT->send();
```

跟随这个send函数，我们可以找到引入模板文件的位置

```
program/include/rcmail_output_html.php line 602

public function send($templ = null, $exit = true)
{
    if ($templ != 'iframe') {
        // prevent from endless loops
        if ($exit != 'recur' && $this->app->plugins->is_processing('render_page')) {
            rcube::raise_error(array('code' => 505, 'type' => 'php',
              'file' => __FILE__, 'line' => __LINE__,
              'message' => 'Recursion alert: ignoring output->send()'), true, false);
            return;
        }
        $this->parse($templ, false);
    }
    else {
        $this->framed = true;
        $this->write();
    }

    // set output asap
    ob_flush();
    flush();

    if ($exit) {
        exit;
    }
}
```

在602行parse主要完成引入模板的工作，跟入
```
program/include/rcmail_output_html.php line 695
```

![image.png-127.2kB][2]

从这里我们就可以看到模板被引入了，比较可惜的是，这里的模板名字无法控制，否则可以构造本地文件包含来攻击。

跟入到后面的`_write`函数可以看到对模板的编译以及替换
![image.png-82.3kB][3]

而具体到相关的模板对象编译，则到涉及到
```
program/include/rcmail_output_html.php line 1217
```
![image.png-42.2kB][4]

在`program/include/rcmail_output_html.php line 1472`，涉及到外部object的变量会通过exechook取值，并暂时赋值为临时变量
```
$hook = $this->app->plugins->exec_hook("template_object_$object", $attrib + array('content' => $content));

if (strlen($hook['content']) && !empty($external)) {
    $object_id                 = uniqid('TEMPLOBJECT:', true);
    $this->objects[$object_id] = $hook['content'];
    $hook['content']           = $object_id;
}

```

在`_write`中 postrender 函数

```
    protected function postrender($output)
    {
        // insert objects' contents
        foreach ($this->objects as $key => $val) {
            $output = str_replace($key, $val, $output, $count);
            if ($count) {
                $this->objects[$key] = null;
            }
        }

        // make sure all <form> tags have a valid request token
        $output = preg_replace_callback('/<form\s+([^>]+)>/Ui', array($this, 'alter_form_tag'), $output);

        return $output;
    }
```

相应的类变量被重新刷新回去


# 全局变量以及过滤函数


## 过滤函数

Roundcube在过滤函数上得思路比较清奇，主要集中在输出过滤上，在输入点或者过程储存上大多不会对数据做过多得处理。

数据的出口主要集中在
```
\program\include\rcmail_output_html.php

show_message 等函数
```

主要的过滤函数为
```
- rcube::Q
- html::
- new html_inputfield
```

等这类函数，其中主要的过滤函数出口类似，我们这里主要看其中1个

```
public static function Q($str, $mode = 'strict', $newlines = true)
{
    return rcube_utils::rep_specialchars_output($str, 'html', $mode, $newlines);
}
```

然后跟入program/lib/roundcube/rcube_utils.php line 165

```
public static function rep_specialchars_output($str, $enctype = '', $mode = '', $newlines = true)
{
    static $html_encode_arr = false;
    static $js_rep_table    = false;
    static $xml_rep_table   = false;

    if (!is_string($str)) {
        $str = strval($str);
    }

    // encode for HTML output
    if ($enctype == 'html') {
        if (!$html_encode_arr) {
            $html_encode_arr = get_html_translation_table(HTML_SPECIALCHARS);
            unset($html_encode_arr['?']);
        }

        $encode_arr = $html_encode_arr;

        if ($mode == 'remove') {
            $str = strip_tags($str);
        }
        else if ($mode != 'strict') {
            // don't replace quotes and html tags
            $ltpos = strpos($str, '<');
            if ($ltpos !== false && strpos($str, '>', $ltpos) !== false) {
                unset($encode_arr['"']);
                unset($encode_arr['<']);
                unset($encode_arr['>']);
                unset($encode_arr['&']);
            }
        }

        $out = strtr($str, $encode_arr);

        return $newlines ? nl2br($out) : $out;
    }

    // if the replace tables for XML and JS are not yet defined
    if ($js_rep_table === false) {
        $js_rep_table = $xml_rep_table = array();
        $xml_rep_table['&'] = '&amp;';

        // can be increased to support more charsets
        for ($c=160; $c<256; $c++) {
            $xml_rep_table[chr($c)] = "&#$c;";
        }

        $xml_rep_table['"'] = '&quot;';
        $js_rep_table['"']  = '\\"';
        $js_rep_table["'"]  = "\\'";
        $js_rep_table["\\"] = "\\\\";
        // Unicode line and paragraph separators (#1486310)
        $js_rep_table[chr(hexdec('E2')).chr(hexdec('80')).chr(hexdec('A8'))] = '&#8232;';
        $js_rep_table[chr(hexdec('E2')).chr(hexdec('80')).chr(hexdec('A9'))] = '&#8233;';
    }

    // encode for javascript use
    if ($enctype == 'js') {
        return preg_replace(array("/\r?\n/", "/\r/", '/<\\//'), array('\n', '\n', '<\\/'), strtr($str, $js_rep_table));
    }

    // encode for plaintext
    if ($enctype == 'text') {
        return str_replace("\r\n", "\n", $mode == 'remove' ? strip_tags($str) : $str);
    }

    if ($enctype == 'url') {
        return rawurlencode($str);
    }

    // encode for XML
    if ($enctype == 'xml') {
        return strtr($str, $xml_rep_table);
    }

    // no encoding given -> return original string
    return $str;
}
```

仔细观察不难发现，其实过滤的方向主要在单双引号的转义，尖括号的转义上。当然，这样的转义已经足够应对90%的情况了。



这里主要是集中在分类上，如果说这里分类到转义比较清晰的路径上，就没什么办法和绕过什么的相关。

比如函数Q设置enctype为html，mode为strict，输出时就会转义包括尖括号、双引号等和XSS相关的符号。我们就没办法绕过了。




[1]: http://static.zybuluo.com/LoRexxar/6nloxgso8aq1kfs4qu16rdve/image.png
[2]: http://static.zybuluo.com/LoRexxar/6rp1srgqkbfmum2optzl1xwz/image.png
[3]: http://static.zybuluo.com/LoRexxar/kj0fbz9gp31hscf3vlauc6va/image.png
[4]: http://static.zybuluo.com/LoRexxar/g8i3dc9ypdloiyah7x8clymi/image.png