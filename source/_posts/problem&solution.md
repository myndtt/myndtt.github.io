---
title: problem&solution
date: 2017-12-04 14:25:18
tags: [php,记录]
categories:
---

## 0x00.前言

这完全是一篇水文，只是记录了一下平时遇到的一些问题及查找到的解决方法。

## 0x01.琴声悠悠

### 一.echo "{${phpinfo()}}"可以输出php环境

这个问题很早以前就有人提出来了，但是一直都没什么人解答，我其实也不知道真实的情况具体是什么，只是根据测试去推测可能是什么。

#### 1.phpinfo()

首先phpinfo()函数是php的内置函数，函数原型是

![36](https://myndtt.github.io/images/36.png)

<!-- more -->

对，这个函数返回值是一个布尔类型，但是它是可以输出配置环境的。

![37](https://myndtt.github.io/images/37.png)

`可以说只要这个函数得到了调用,只要调用print echo var_dump等函数都可以将php环境信息输出到屏幕上。`

#### 2.php变量

![38](https://myndtt.github.io/images/38.png)

众所周知，php中有效变量由$+变量名字组成，变量名字必须为`(字母或者下划线开头)`,除此之外，变量名字可以由其他函数返回值或者变量代替，如：

```php
<?php
$aaa="man";
$man="hello";
echo $$aaa;  //输出hello
?>
```

```php
<?php
function test(){
	return "man";
}
$man="hello";
echo ${test()};//输出hello;
?>
```

#### 3.php双引号中的大括号

网上对这一部分有了许多解析，简单总结就是一个界定的作用，如上面看到的那样。

`${test()}` ,这里就将test函数括起来，要求php去执行、解析。



有了上面三个小问题的简单解释`echo "{${phpinfo()}}"` 也就没什么了。

首先在字符串在双引号中，所以最外面一层双引号进行界定分开，表示其中需要解析，然而发现里面是${}结构，则需要解析这个变量名字，也即解析phpinfo()函数，此函数解析又有前面的打印函数，自然将php环境输出至屏幕，而phpinfo()执行成功返回1，不允许这样的变量，页面最后面报错。

```php
<?php
//echo "{${phpversion()}}"; //可以从返回信息中得到版本，其他预置函数都可以
echo "{${phpinfo()}}";

?>
```

![39](https://myndtt.github.io/images/39.png)

### 二.select语句执行流程

一直想知道select语句执行顺序到底是怎样的，找到一篇十分详细的文章解决了我的问题。

https://www.cnblogs.com/annsshadow/p/5037667.html    

(看原文档更好)

select语句的原型可视为：

```mysql
SELECT DISTINCT
    < select_list >
FROM
    < left_table > < join_type >
JOIN < right_table > ON < join_condition >
WHERE
    < where_condition >
GROUP BY
    < group_by_list >
HAVING
    < having_condition >
ORDER BY
    < order_by_condition >
LIMIT < limit_number >
```

执行顺序是：

```mysql
FROM <left_table>
ON <join_condition>
<join_type> JOIN <right_table>
WHERE <where_condition>
GROUP BY <group_by_list>
HAVING <having_condition>
SELECT 
DISTINCT <select_list>
ORDER BY <order_by_condition>
LIMIT <limit_number>
```

简单解析可为：首先看看select的字段来自哪里，即先解析`from`得到后面的表，如何有多表且是`用join`联合起来则就按`on`条件进行联合。得到字段来自的表后，将表加入的内存中。随后解析`where`按一定条件进行筛选，得到新的虚表。有`group by `则将该虚表按条件进行重组，重组后如果有`having`则对按条件对重组后的表进行过滤得到全新的虚表，随后`select`进行字段选择，如果有`disinct`存在就将重复的删除(指定索引唯一去除重复数据)。z再然后就是`order by`排序和`limit`进行限定。举一个简单小栗子，sql注入常会出现这样的句子

```mysql
select id from user where username='a' or 1
```

首先mysql进行词法分析，发现第一个字为select关键字，随即找from关键字确定表为user，将user表加载至内存得到虚表1，此时指针指向虚表1中第一条数据，然后去找where关键字进行条件判断该数据是否符合要求。这里条件为`username='a' or 1` 恒返回1为真，也即虚表1中第一条数据符合要求，将 数据加载至虚表2中，表中所有数据检索完后，执行select,从虚表2中进行字段选择得到虚表3，随后将虚表3进行返回。（这一段表达可能有误，但是大致过程是如此）

## 0x02.篱笆外

mysql中udf库，链接：https://dn.jarvisoj.com/challengefiles/udf.so.02f8981200697e5eeb661e64797fc172

这是一道web题，提示 `咦，奇怪，说好的WEB题呢，怎么成逆向了？不过里面有个help_me函数挺有意思的哦`

根据文件udf.so查找资料到有这么一句话：`用户可以自己建立包含自定义函数得动态库来创建自定义函数，简称udf.这里是关键`。由于是.so动态库则用linux(ubuntu)环境进行测试。

#### 1.过程

ubuntu下载mysql，将下载得`udf.so.02f8981200697e5eeb661e64797fc172`放入mysql默认.so文件的目录(/usr/lib/mysql/plugin)下,登入mysql依次执行：

```mysql
create function help_me returns string soname 'udf.so.02f8981200697e5eeb661e64797fc172';//激活该//函数
select help_me();//根据提示执行后面
create function getflag returns string soname 'udf.so.02f8981200697e5eeb661e64797fc172';
select getflag();
```

#### 2.其他

可以利用此.so文件对mysql进行提权。如果有sql注入点可以上传文件，由此那么我们可以上传一个可以执行系统函数的.so文件进行提权。

http://www.91ri.org/16540.html









