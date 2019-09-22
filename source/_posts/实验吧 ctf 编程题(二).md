---
title: 实验吧 CTF 编程题（二）
id: 149
date: 2017-07-23 16:35:18
tags: [ctf,实验吧,编程]
---

三：奖券

这是水题，可以暴力
```python
num=0
for a in range(100000,1000000):
	b=str(a)
	if '4' not in b:
		num+=1
print num
```
<!-- more -->

四：三羊献瑞

也是水题,这里提一点，许多人看见这个就直接遍历，其实看看题目可以知道一些东西，比如这个“祥”字加上“三”字进位了，并且进位后最高位为“三”字，那么可以确定“三”字为1，则“祥”字为9了，减少遍历
```python
for f in range(10):
 for g in range(10):
  for h in range(10):
   if (a!=b) and (a!=c) and (a!=d) and (a!=f) and (a!=g) and (a!=h):
    if (b!=c)and (b!=d)and (b!=e) and (b!=f) and (b!=g) and (b!=h):
     if (c!=d)and (c!=e) and (c!=f) and (c!=g) and (c!=h):
      if (d!=e) and (d!=f) and (d!=g) and (d!=h):
       if (e!=f) and (e!=g) and (e!=h):
        if(f!=g) and (f!=h):
         if (g!=h):
          sum=(a+e)*1000+(f+b)*100+(g+c)*10+d+b
          cal=10000+1000*f+100*c+10*b+h
          if(sum==cal):
            print e,f,g,b
            exit()
```
五：找素数
很简单，涉及到的东西就是枚举加上素数的判定 
```python
import math
def isPrime(n):    
	if n <= 1:    
		return False   
	for i in range(2, int(math.sqrt(n)) + 1):    
		if n % i == 0:
			return False
		return True
if __name__=='__main__':
	num=0
	for a in range(367,100000000,186):
		flag=isPrime(a)
		if(flag):
			num+=1
			if(num==151):
				print a
				exit()
```
写在最后：
切莫浮沙建堡垒