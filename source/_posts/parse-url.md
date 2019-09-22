---
title: parse_url
date: 2017-12-07 21:29:31
tags: [php,web,记录]
categories:

---



## 0x00.前言

又是一道关于parse_url的题目，上次也说了一道http://myndtt.com/2017/10/22/url%E4%B8%AD%E7%9A%84%E4%B8%8B%E5%88%92%E7%BA%BF/

## 0x01.开门见山

题目都没什么稀奇的，题目链接：http://solveme.safflower.kr/prob/url_filtering/

代码：

```php
<?php
    error_reporting(0);
    require __DIR__."/lib.php";

    $url = urldecode($_SERVER['REQUEST_URI']);
    $url_query = parse_url($url, PHP_URL_QUERY);

    $params = explode("&", $url_query);
    foreach($params as $param){

        $idx_equal = strpos($param, "=");
        if($idx_equal === false){
            $key = $param;
            $value = "";
        }else{
            $key = substr($param, 0, $idx_equal);
            $value = substr($param, $idx_equal + 1);
        }

        if(strpos($key, "do_you_want_flag") !== false || strpos($value, "yes") !== false){
            die("no hack");
        }
    }

    if(isset($_GET['do_you_want_flag']) && $_GET['do_you_want_flag'] == "yes"){
        die($flag);
    }

    highlight_file(__FILE__);
```

<!-- more -->

## 0x02.循序渐进

在分析题目之前，首先需要说一下其他的知识。

### URI与URL

URI,全称`uniform resource identifier`统一资源标识符，用来唯一标识一个资源，同时可以代表相对的绝对的的一个资源信息。

URL似乎我们比较熟悉,全称`uniform resource locator`统一资源定位器，需要定位资源的具体信息，不可以是相对的。

在php中有如下

```php
http://localhost/aaa/index.php
结果：
$_SERVER['QUERY_STRING'] = "";
$_SERVER['REQUEST_URI']  = "/aaa/";
$_SERVER['SCRIPT_NAME']  = "/aaa/index.php";
$_SERVER['PHP_SELF']     = "/aaa/index.php";
```



### Parse_url 

函数原型`mixed parse_url ( string $url [, int $component = -1 ] )`

按php文档的解释就是该函数专门解析URL的。

![40](https://myndtt.github.io\images\40.png)

同时这个函数的返回值可能是false

![41](https://myndtt.github.io\images\\41.png)

### 回到代码

代码如下两句首先对我们URI的进行相应解析

```php
    $url = urldecode($_SERVER['REQUEST_URI']);
    $url_query = parse_url($url, PHP_URL_QUERY);
```

随后代码对其进行参数及值抽取

```php
//输入http://xxx.com/1/index.php?a=2
$params = explode("&", $url_query);  //得到数组0 => 2
    foreach($params as $param){

        $idx_equal = strpos($param, "=");//获得参数
        if($idx_equal === false){
            $key = $param;
            $value = "";
        }else{
            $key = substr($param, 0, $idx_equal);//参数a
            $value = substr($param, $idx_equal + 1);//参数值 这里值为2
        }
      if(strpos($key, "do_you_want_flag") !== false || strpos($value, "yes") !== false){
            die("no hack");
        }//检测到这样就没了

```

但是后面又需要我们get提交 do_you_want_flag=yes

```php
 if(isset($_GET['do_you_want_flag']) && $_GET['do_you_want_flag'] == "yes"){
        die($flag);
    }

```

这是自相矛盾的，但是上文提到了`parse_url`是可能返回错误的，这就是关键，一旦错误后面就不会解析。直接跳过了,也就不会` die("no hack");`,那么如何返回错误么，老套路，还是看文档。

![42](https://myndtt.github.io\images\\42.png)

这里提到了除了file不能三个`///`,否则其他都不允许！试试这个。

http://solveme.safflower.kr///prob/url_filtering/index.php?do_you_want_flag=yes

果然。

## 0x03.总结

说起来就是一个`parse_url`的解析错误，翻翻文档就可以了，但是我想思路就是琢磨出来的，有时灵感一现就有了，但有时也会困惑，但不管怎样遇到问题看文档总是一个正确的选择，无论是php还是mysql。
