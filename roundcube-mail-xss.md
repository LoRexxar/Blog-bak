---
title: Roundcube mail 3 Xss
date: 2020-06-10 16:55:44
tags: 
- roundcube
- xss
---

---



3个没用的Xss

<!--more-->


# Store Xss in /installer/test.php

in install step2

![image.png-66.2kB][1]


Access trigger
```
http://127.0.0.1/roundcubemail-1.4.4/installer/index.php?_step=3
```
![image.png-178.8kB][2]


## code anlysize

input in step2
```
if (!empty($_POST['submit'])) {
  $_SESSION['config'] = $RCI->create_config();

  if ($RCI->save_configfile($_SESSION['config'])) {
     echo '<p class="notice">The config file was saved successfully into <tt>'.RCMAIL_CONFIG_DIR.'</tt> directory of your Roundcube installation.';

```

follow into`create_config`在`\program\include\rcmail_install.php line 216`
```
if ($prop == 'db_dsnw' && !empty($_POST['_dbtype'])) {
    if ($_POST['_dbtype'] == 'sqlite') {
        $value = sprintf('%s://%s?mode=0646', $_POST['_dbtype'],
            $_POST['_dbname'][0] == '/' ? '/' . $_POST['_dbname'] : $_POST['_dbname']);
    }
    else if ($_POST['_dbtype']) {
        $value = sprintf('%s://%s:%s@%s/%s', $_POST['_dbtype'],
            rawurlencode($_POST['_dbuser']), rawurlencode($_POST['_dbpass']), $_POST['_dbhost'], $_POST['_dbname']);
    }
}
```

Regarding the configuration of db, only when it is not sqlite, the user and pass are filtered, and other parameters are not filtered. Direct incoming will be put into config.


in step3，`installer/test.php`，directly export config out.

![image.png-40kB][3]

# store xss in smtp config

![image.png-29.4kB][4]

![image.png-33.5kB][5]

## code analysize

There is no filtering when entering variables

When the output is not defined by the html_inputfield class, it will take effect if the output is obtained directly

![image.png-42.5kB][6]


# store xss in email in database

![image.png-37.8kB][7]


![image.png-46.1kB][8]


## code analysize

When accessing index.php, the corresponding page will render the template, when obtaining the template, from

```
program/include/rcmail_output_html.php line 2094


public function current_username($attrib)
{
    static $username;

    // alread fetched
    if (!empty($username)) {
        return $username;
    }
    // Current username is an e-mail address
    if (strpos($_SESSION['username'], '@')) {
        $username = $_SESSION['username'];
    }
    // get e-mail address from default identity
    else if ($sql_arr = $this->app->user->get_identity()) {
        $username = $sql_arr['email'];
    }
    else {
        $username = $this->app->user->get_username();
    }
    
    return rcube_utils::idn_to_utf8($username);
}
```

in program/lib/roundcube/rcube_user.php line 304
```
    function get_identity($id = null)
    {
        $id = (int)$id;
        // cache identities for better performance
        if (!array_key_exists($id, $this->identities)) {
            $result = $this->list_identities($id ? "AND `identity_id` = $id" : '');
            $this->identities[$id] = $result[0];
        }

        return $this->identities[$id];
    }
```

Directly from the database identities.

And this value is inserted into the database by call function `insert_identity`

```
program/lib/roundcube/rcube_user.php line 397

function insert_identity($data)
{
    if (!$this->ID) {
        return false;
    }

    unset($data['user_id']);

    $insert_cols   = array();
    $insert_values = array();

    foreach ((array)$data as $col => $value) {
        $insert_cols[]   = $this->db->quote_identifier($col);
        $insert_values[] = $value;
    }

    $insert_cols[]   = $this->db->quote_identifier('user_id');
    $insert_values[] = $this->ID;

    $sql = "INSERT INTO ".$this->db->table_name('identities', true).
        " (`changed`, ".implode(', ', $insert_cols).")".
        " VALUES (".$this->db->now().", ".implode(', ', array_pad(array(), count($insert_values), '?')).")";

    $insert = $this->db->query($sql, $insert_values);

    // clear the cache
    $this->identities = array();
    $this->emails     = null;

    return $this->db->affected_rows($insert) ? $this->db->insert_id('identities') : false;
}
```

This function is mainly called in two places, one is the installation place, and the other is the login place.

```
/program/lib/roundcube/rcube_user.php lime 648

$rcube->user = $user_instance;
$mail_domain = $rcube->config->mail_domain($data['host']);
$user_name   = $data['user_name'];
$user_email  = $data['user_email'];
$email_list  = $data['email_list'];

if (empty($email_list)) {
    if (empty($user_email)) {
        $user_email = strpos($data['user'], '@') ? $user : sprintf('%s@%s', $data['user'], $mail_domain);
    }
    $email_list[] = $user_email;
}
```


[1]: http://static.zybuluo.com/LoRexxar/owf1cwattgs754us8a50b9ql/image.png
[2]: http://static.zybuluo.com/LoRexxar/ch865oo7kx1ox4qq6s2yan8h/image.png
[3]: http://static.zybuluo.com/LoRexxar/3fauiwg2at94fitzt0q0fifb/image.png
[4]: http://static.zybuluo.com/LoRexxar/oxxzk7ue5r65ox75guzzm40f/image.png
[5]: http://static.zybuluo.com/LoRexxar/w1limk4us55aqvsgmwcm25my/image.png
[6]: http://static.zybuluo.com/LoRexxar/y7k2beho0ay3w91oyoxv0z6w/image.png
[7]: http://static.zybuluo.com/LoRexxar/0ybzvbssnksdg8fbim9tk2df/image.png
[8]: http://static.zybuluo.com/LoRexxar/g7bi6zgtixwuijgxnj7555da/image.png