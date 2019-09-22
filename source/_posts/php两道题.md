---
title: php两道题
id: 266
date: 2017-09-20 00:25:37
tags: [php,记录,audit]
---

一：PHP的数组问题


```php
 <?php
/*******************************************************************
 * PHP Challenge 2015
 *******************************************************************
 * Why leave all the fun to the XSS crowd?
 *
 * Do you know PHP?
 * And are you up to date with all its latest peculiarities?
 *
 * Are you sure?
 *
 * If you believe you do then solve this challenge and create an
 * input that will make the following code believe you are the ADMIN.
 * Becoming any other user is not good enough, but a first step.
 *
 * Attention this code is installed on a Mac OS X 10.9 system
 * that is running PHP 5.4.30 !!!
 *
 * TIPS: OS X is mentioned because OS X never runs latest PHP
 *       Challenge will not work with latest PHP
 *       Also challenge will only work on 64bit systems
 *       To solve challenge you need to combine what a normal
 *       attacker would do when he sees this code with knowledge
 *       about latest known PHP quirks
 *       And you cannot bruteforce the admin password directly.
 *       To give you an idea - first half is:
 *          orewgfpeowöfgphewoöfeiuwgöpuerhjwfiuvuger
 *
 * If you know the answer please submit it to info@sektioneins.de
 ********************************************************************/
```
<!-- more -->

```php
$users = array(
        "0:9b5c3d2b64b8f74e56edec71462bd97a" ,
        "1:4eb5fb1501102508a86971773849d266",
        "2:facabd94d57fc9f1e655ef9ce891e86e",
        "3:ce3924f011fe323df3a6a95222b0c909",
        "4:7f6618422e6a7ca2e939bd83abde402c",
        "5:06e2b745f3124f7d670f78eabaa94809",
        "6:8e39a6e40900bb0824a8e150c0d0d59f",
        "7:d035e1a80bbb377ce1edce42728849f2",
        "8:0927d64a71a9d0078c274fc5f4f10821",
        "9:e2e23d64a642ee82c7a270c6c76df142",
        "10:70298593dd7ada576aff61b6750b9118"
);

$valid_user = false;

$input = $_COOKIE['user'];
$input[1] = md5($input[1]);

foreach ($users as $user)
{
        $user = explode(":", $user);
        if ($input === $user) {
                $uid = $input[0] + 0;
                $valid_user = true;
        }
}

if (!$valid_user) {
        die("not a valid user\n");
}

if ($uid == 0) {

        echo "Hello Admin How can I serve you today?\n";
        echo "SECRETS ....\n";

} else {
        echo "Welcome back user\n";
}
```

这道题，这个栗子算是比较老了的，主要涉及的知识就是

```
Description:
------------
var_dump([0 => 0] === [0x100000000 => 0]); // bool(true)
```


当然这是跟PHP版本有关系

具体可以看[http://www.sektioneins.de/blog/15-08-03-php_challenge_2015_solution.html](http://www.sektioneins.de/blog/15-08-03-php_challenge_2015_solution.html)

二：[http://solveme.safflower.kr/prob/name_check](http://solveme.safflower.kr/prob/name_check)

这道题十分有意思，我其实想了很久，突然突发奇想才想出来的
```php
<?php
error_reporting(0);
require __DIR__.'/lib.php'; 

if(isset($_GET['name'])){

    $name = $_GET['name'];
    if(preg_match("/admin|--|;|\(\)|\/\*|\\0/i", $name)){
        echo 'Not allowed input';
        goto quit;
    }

    $sql = new SQLite3('name_check.db', SQLITE3_OPEN_READWRITE);
    $res = $sql-&gt;query("
        SELECT 
        MAX('0','1','{$name}') LIKE 'a%', 
        INSTR('{$name}','d')&gt;0, 
        MIN('{$name}','b','c') LIKE '__m__', 
        SUBSTR('{$name}',-2)='in'
    ;");
    if($res === false){
        echo 'Database error';
        goto quit;
    }

    $row = $res-&gt;fetchArray(SQLITE3_NUM);
    if(
        $row[0] + $row[1] + $row[2] + $row[3] !== 4 ||
        array_sum($row) !== 4 
    ){
        echo 'Auth failed';
        goto quit;
    }

    echo $flag;

quit:
    echo '&lt;hr&gt;';
}

highlight_file(__FILE__);</pre>
```
代码十分有趣，按照代码来，我们需要select 1,1,1,1；四个1出来才能满足结果，也即下面代码每一行为真
```php
        MAX('0','1','{$name}') LIKE 'a%', 
        INSTR('{$name}','d')&gt;0, 
        MIN('{$name}','b','c') LIKE '__m__', 
        SUBSTR('{$name}',-2)='in'
```
第一行意思为你第一个必须以a开头

第二行就是你输入的需要有d

第三行就是输入的是xxmxx（xx为任意字符，中间是字符m）

第四行就是字符以in结尾

这摆明着要叫我们输入admin啊，可惜前面的正则残酷的将其过滤。

查看了文档，想过很多种方法，今天突发奇想 ||，这在sqlite中是一个字符连接的，代码的正则中没有将其过滤，也没有过滤’,所以最后就是[http://solveme.safflower.kr/prob/name_check/?name=ad%27||%27min](http://solveme.safflower.kr/prob/name_check/?name=ad%27||%27min)

出来结果。这个网站题目有点意思。再接再厉（简单的题目就不说了）

