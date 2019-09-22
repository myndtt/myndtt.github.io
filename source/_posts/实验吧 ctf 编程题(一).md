---
title: 实验吧 CTF 编程题 （一）
id: 147
date: 2017-07-23 12:26:16
tags: [ctf,实验吧,编程]	
---

我进过一些安全类的QQ群，发现挺多”小白“（当然我也是^_^）遇到不懂得就一顿糊涂拿到群里来问，其实很多问题在网上就能找到，并且许多的刚入这个方向的伙伴们喜欢用工具，不喜欢了解其中道理，当然这可能需要一个过程，当时我也是这样，喜欢拿个御剑到处乱扫^_^。不说了，说多了大家还以为我很牛呢。不管怎样，要想搞好安全，一定要会编程，这是当年教我入门的老师跟我说的一句语重心长的话！就拿实验吧的题目来写写吧。话说这个网站真的很好，题目质量很好，但是好像好久没更了，并且有些题出错了，不过每周四都会有老师开直播教一些简单基本实用的东西，免费的，很实在。题目链接[http://www.shiyanbar.com/ctf/practice](http://www.shiyanbar.com/ctf/practice)

一：百米

这道题很基础，速度要快，就是用代码呗
```python
#coding=utf-8
import requests
from bs4 import BeautifulSoup 
import urllib
def getit():
	headers = {
		'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/47.0.2526.80 Safari/537.36'
	}
	url=u'http://ctf5.shiyanbar.com/jia/index.php'
	response=session.get(url,headers=headers)
	a=response.text
	b=BeautifulSoup(a,"html.parser")#beautfiulsoup解析
	c=b.find("div",{"name":"my_expr"})
	d=c.get_text()#提取出那个计算式
	e=d.replace('x','*')#用正儿八经的*代替那个字母x
	print e
	print eval(e)#eval很好用，直接就算出来了
	f=eval(e)
	data={
		"pass_key":f
		}
	h=session.post(url,data=data,headers=headers)#发包
	print h.text#看结果
if __name__=='__main__':
	with requests.Session() as session:#建立会话
		getit()
```

<!-- more -->

但是这个代码并没有用，不知道是我姿势不对，还是题有问题，先放着

二：迷宫大逃亡

这道题就是深度遍历，之前刷过一些OJ题，这道题还是很基本的
```c
#include 
#include 
#include 
using namespace std;
int sx,sy,ex,ey;
int num;
int n;
char maps[1000][1000];//存图
int flag[1000][1000];//标志是否已经来过
int dir[4][2]={-1,0,0,-1,1,0,0,1};//方向
string request;//保存结果

int check(int x,int y){
	if(x>0&&x<=n&&y>0&&y<=n&&flag[x][y]==0&&maps[x][y]=='O')
		return 1;
	return 0;
}
void dfs(int x,int y){//深度遍历,一直找,直到堵死了
	flag[x][y]=1;
	for(int i=0;i<4;i++){
		if(check(x+dir[i][0],y+dir[i][1]))
			dfs(x+dir[i][0],y+dir[i][1]);

	}
}
int main(){
	freopen("in.txt","r",stdin);//将这个文件作为输入的流
	scanf("%d",&num);//这时候是读取文件的，而不是默认的键盘
	while(num--){
	memset(flag,0,sizeof(flag));
	scanf("%d%d%d%d%d",&n,&sx,&sy,&ex,&ey);
	getchar();//清除缓存,清除读取一行后的'\n'
	for(int i=1;i<=n;i++){
		for(int j=1;j<=n;j++)
			scanf("%c",&maps[i][j]);
		getchar();//清除缓存
	}
	dfs(sx,sy);
	if(flag[ex][ey]==1)//看看经过深度遍历后,这个点是否到达了
		request+='1';
	else
		request+='0';
	}
	cout<<request;
}
```
这仅仅是得到一串字符串，C/C++也是很好处理的，但是我还是比较懒，之前写过类似的python脚本，把字符串变一下就好了 

```python
#import base64
u=u'011000010100100001010010001100000110001101000100011011110111011001001100001100110110010000110011011001000111100100110101011110100110000101000111011011000011010101011001010101110011010101101001010110010101100001001001011101010101100100110010001110010111010001001100001100100100111000110000010110100110100100111000011110000100111101000100010010010011000001010010011011010111100001101000010110100011000001101100011110100101100101010110010101100101001101010100010000010011110100111101'
a=''
for i in range(0,len(u),8):
	x=u[i:i+8]
	a+=chr((int(x,2)))
print a
#print base64.b64decode(a)
```

结果是一段很明显的base64加密的字符串。解密一下就好了(去掉上面的注释即可)

写在最后：

提交不对没关系，实干出真知