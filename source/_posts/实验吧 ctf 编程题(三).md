---
title: 实验吧 CTF 编程题（三）
id: 151
date: 2017-07-23 20:16:44
tags: [ctf,实验吧,编程]
---

六：循环

送分题，这里判断奇数偶数的时候我用了逻辑与，其实一个数是否为奇数还是偶数就是看二进制的最后一位

```python
max=0
for a in range(900,10001):
n=a	
num=0
while(n!=1):
	num+=1
	if(n&amp;1==0):
		n=n/2
	else:
		n=3*n+1
if(num&gt;max):
	max=num
	#print max
	print max
```

<!-- more -->

七：小球下落
这道题在刘老师的紫书《算法竞赛入门经典》中提到过，算是一道模拟题吧，可以通过过程分析找出其中规律，也可以直接重复该过程，这里使用后者

```c
#include 
int main()
{
int flag[1&lt;&lt;16]; //结点个数
int i;
memset(flag,0,sizeof(flag));  //每个点开关预设为0即关闭状态
int k,n=(1&lt;&lt;16)-1;           //ｎ是最大结点编号
for(i=0;i&lt;12345;i++) //连续让小球下落 { k=1; for(;;) { flag[k]=!flag[k];//改变状态 k=flag[k]?k*2:k*2+1; //根据开关状态选择下落方向 if(k&gt;n)             //判断是否已经落地了
                break;
        }
    }
printf("%d\n",k/2);         //落地之前的叶子编号
return 0;
}
```
八：求底运算

这道题涉及到大数问题，果断用python
```python
import math
for a in range(100,2000):
num=pow(a,7)
#print num
if num ==4357186184021382204544:
	print a
	exit()
```
但是总是用python是不好的，东西封装的太多，没有跟C语言一样更接近底层，所以性能比不上，总之C语言很重要！
九：普里姆运算
这道题算是目前来说比较复杂的，但仅仅是复杂，其思路其实很清楚的。首先标明a至b的所有素数。然后从a至b一步步增加，判断当前数是否为素数，如果是，接着改变这个数的某一位，看看这些个改变中那些是素数。有思路就好办，就好写代码了
<pre>


```c
#include 
using namespace std;
define Max 10000
bool flag[Max],is_prime[Max];//一个标记是否找过这个数，一个标记数是否为素数
int num[Max];
int a=1373;
int b=8017;
void getprime()
{
int i,j;
memset(is_prime,false,sizeof(is_prime));
for(i=1000;i&lt;=9999;i++)
{
	for(j=2;j&lt;=sqrt(i);j++) if(i%j==0) break; if(j&gt;sqrt(i))
		is_prime[i]=true;
}
return ;
}
int judge(int v,int i,int j)
{
if (!is_prime[v])
    return 0;   //不是素数返回0
if (i == 1 &amp;&amp; j == 0)
    return 0;  //首位为0返回0
char s[6] = "\0";
sprintf(s, "%d", v);
s[i - 1] = j + '0';   //将第i位改成j
sscanf(s, "%d", &amp;v);
if (!flag[v])
    return v;   //返回改变后的值
return 0;
}
int bfs()
{
int i,j;
queue  q;//建立一个队列保持步数
q.push(a);
flag[a]=1;//表示访问过
while(!q.empty())
{
int v=q.front();//取出队列第一个数
q.pop();
for(i=1;i&lt;=4;i++)
	for(j=0;j&lt;=9;j++)
	{
	int t=judge(v,i,j);
	if(t){
	flag[t]=1;
	num[t]=num[v]+1;//转到数字t需要经过转到数字V的步数+1
	if(t==b)
		return num[t];
	q.push(t);
		}
	}
}
return -1;
}
int main()
{
getprime();
memset(flag, false, sizeof(flag));
int request = bfs();
if (request == -1)
	printf("no\n");
else
	printf("%d\n", request);
return 0;
}
```

这里用到了sprintf，sprintf 最常见的应用之一莫过于把整数打印到字符串中，但请记住这个函数要谨慎使用，这个函数不注意会造出缓存区溢出。

十：大数模运算

就跟以往提到的一样，python对大数处理很好，这题主要考究数学功底和对指数相乘，如果你想知道更多，这里有一个传送门,是一枚大佬的，很详细[http://blog.csdn.net/lyy289065406/article/details/6648539](http://blog.csdn.net/lyy289065406/article/details/6648539)

```python
import math
a=12345
d=12345
p=[]
n=[]
alltotal=1
b=int(math.sqrt(a))+1
for i in range(2,b):
#print i
sum=0
if (a%i)==0:
	p.append(i)
	while(a%i==0):
		sum+=1
		a=a/i
	sum=sum*d
	n.append(sum)
#print sum
p.append(a)
n.append(1*d)
for a in range(0,len(p)):
	mid=(pow(p[a],(n[a]+1))-p[a])/(p[a]-1)+1
	alltotal=alltotal*mid
	print mid
print alltotal%9901
```
&nbsp;