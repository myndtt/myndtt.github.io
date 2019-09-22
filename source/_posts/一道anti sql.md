---
title: 从一道anti sql题讲起
date: 2017-11-09 20:00:07
tags: [sqli,web,audit]
categories:
---

## 0x00.前言

题目链接 http://antisqli.thinkout.rf.gd

```php
 <?php
    // It's 'Anti SQLi' problem of 'Solve Me'.
    error_reporting(0);
    $con = mysqli_connect("127.0.0.1", "root", "root","mytest")
            or die('SQL server down');

    $id = $_GET['id'];
    $pw = $_GET['pw'];
    var_dump($pw);
    echo "</br>";
    echo  $id.','.$pw."</br>";
    echo "SELECT * FROM `user` WHERE `id`='{$id}' AND `pwssword`=md5('{$pw}');"."</br>";
    if(isset($id, $pw)){
        preg_match(
            '/\.|\`|"|\'|\\|\xA0|\x0B|0x0C|\t|\r|\n|\0|'.
            '=|<|>|\(|\)|@@|\|\||&&|#|\/\*.*\*\/|--[\s\xA0]|'.
            '0x[0-9a-f]+|0b[01]+|x\'[0-9a-f]+\'|b\'[01]+\'|'.
            '[\s\xA0\'"]+(as|or|and|r*like|regexp)[\s\xA0\'"]+|'.
            'union[\s\xA0]+select|[\s\xA0](where|having)|'.
            '[\s\xA0](group|order)[\s\xA0]+by|limit[\s\xA0]+\d|'.
            'information_schema|procedure\s+analyse\s*/is',
            $id.','.$pw
        ) and die('Hack detected');
```

<!-- more -->

```php
   $result = mysqli_fetch_array(
            mysqli_query(
                $con, 
                "SELECT * FROM `user` WHERE `id`='{$id}' AND `pwssword`=md5('{$pw}');"
            )
        );
        mysqli_close($con);
        echo var_dump($result)."</br>";
        if(isset($result)){
            if($result['id'] === '1'){
                echo "great";
            }else{
                echo 'Hello, ', $result['id'];
            }
        }else{
            echo 'Login failed';
        }
        echo '<hr>';
    }

    highlight_file(__FILE__); 
```

很早之前就看见过这道题。最初想法是:

正则过滤了大部分东西,单引号,各种注释，但似乎没有过滤掉反斜杠,变量被单引号包含,想用反斜杠吃掉一个单引号,同时注释掉后面的不要的。

![1](http://myndtt.github.io/images/21.png)

本地测试没有问题，但是这是建立在没有过滤注释的情况。然后我想不出其他方法了,然后问了龙师兄，龙师兄说他本地测试成功了，用的是“-- ”注释，(注意是两个减号 最后是有一个空格的),FUZZ出一个可以成功绕过正则中的\s，但是没记录下来忘了.....并且在原题网站上是没有成功复现的，额......略显尴尬。最后就不了了之了。

## 0x01.回头

起初就是看书看累了，想休息休息看看代码之类的，突然想起上诉那道题目，就重新拿起来看看会不会有什么新发现。

### 1.先说说反斜杠的问题

上面正则中明明有"`|\\|\xA0|`"，这不是匹配了反斜杠么，为什么还可以输入反斜杠? 错！第1个'\'转义 字符串的第2个'\'，结果为字符串为'\',要想真正匹配反斜杠，你得需要4个:

![30](http://myndtt.github.io/images/30.png)

```
'\\\\' 
第1个'\'转义字符串的第2个'\'，字符串为'\'
第3个'\'转义第4个'\'，相当于字符串'\'
字符合起来为'\\' 两个'\\' 正则表达式看做'\'
```

这样反斜杠就没问题了

ps ：补上一张图

![31](http://myndtt.github.io/images/31.png)

### 2.the next

前段时间做出一道url中下划线的题，

 http://myndtt.com/2017/10/22/url%E4%B8%AD%E7%9A%84%E4%B8%8B%E5%88%92%E7%BA%BF/

![7](http://myndtt.github.io/images/7.png)

其中用的是无效字符，就想试试在这里有什么新的发现，果然:

![22](http://myndtt.github.io/images/22.png)



“%1A”这类的url无效字符虽然无效但是是没有过滤掉的，我们用chrome来看更直接

![23](http://myndtt.github.io/images/23.png)

恰好不知什么原因”--“可以在mysql某个版本(我自己测试的是Ver 14.14 Distrib 5.5.53, for Win32 (AMD64)版本，其他版本没试过)中可以用来注释（上诉的无效字符都可以）

![24](http://myndtt.github.io/images/24.png)

----------------------------------------------------------------------------------------------------------------------------------------

![25](http://myndtt.github.io/images/25.png)

这里在测试中遇到了这个:

![26](http://myndtt.github.io/images/26.png)

是直接复制上诉成功的那句话，这里不成功应该跟命令行字符有关

还遇到了这个:

![27](http://myndtt.github.io/images/27.png)

三条查询语句分别为

```sql
SELECT* FROM `user` WHERE `id`='1'--;             成功  查到的是id=1
SELECT* FROM `user` WHERE `id`='1'--1;            成功  查到的是id=2
SELECT* FROM `user` WHERE `id`='1'--';            不成功
```

想了想，这里的第一条，第二条不是注释，而是跟mysql运算符先后有关，后来查了确实如此,id='1'--1,应该是‘1’-(-1),即1减去负1，所以第二条查到的是id=2的数据,至于第一条到底是不是注释还是减去负0就不得而知了(正经脸)

![28](http://myndtt.github.io/images/28.png)

ps: 想想“-- ”是不是表示减去负数的空白符呢，而恰恰是这样就有着注释的含义。猜想而已，得去看看人家代码到底怎么写的，目前水平不够。

## 0x02.end

最后还是没有在原来题目上复现成功，哈哈，又水了一篇。花了一点时间，爆破id,用上面的payload，然后将id从1遍历至31337还是没有结果。要不从负数开始?哈哈算求....  哦，对了，闲暇时搜了一下31337是什么意思

![29](http://myndtt.github.io/images/29.png)



ps：

最后payload为:

```
id=\&pw=union all select 31337,31337,31337 from antisqli --%1A
```

![32](http://myndtt.github.io/images/32.png)

