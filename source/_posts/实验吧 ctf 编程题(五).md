---
title: 实验吧 CTF 编程题（五）
id: 166
date: 2017-07-24 22:49:21
tags: [ctf,实验吧,编程]
---

十六：二叉树遍历

这算是比较老的一道题了，如果是做题，可以直接画图画出来，以前写过已知二叉树前序中序求后序的代码


```c
#include 
#include 
#include
#include
using namespace std;
struct Tree{
	struct Tree *left;
	struct Tree *right;
	char num;
};
typedef struct Tree treenode;
void getit(char* inor,char* preor,int length){
	if(0==length)
		return;
	Tree * newnode = new Tree;//分配空间
	newnode->num=*preor;
	int flag=0;
	for(;flagnum;
	return;
}
int main(){
	char* preor="DBACEGF";
	char* inor="ABCDEFG";
	//printf ("%d",sizeof(preor));
	getit(inor,preor,8);
	printf("\n");
	return 0;

}
```

<!-- more -->

十七：约瑟夫环
也是一道十分经典的问题，是属于约瑟夫环好人与坏人问题之前打过表

```c
#include 
#include 
#include 
#include 
using namespace std;
//{ 0, 2, 7, 5, 30, 169, 441, 1872, 7632, 1740, 93313, 459901, 1358657, 2504881 };
int num[20]={0};
bool check(int k,int m){
	int n=2*k,s=0,resq;
	int i;
	for(i=1;i<=k;i++){
		resq=n-i+1;
		s=(s+m-1)%resq;
		if(s
```

十八：双基回文数
也是老题了
​	
```python
def check(n):
	count=0
	a=str(n)
	b=a[::-1]
	if a==b:
		return True
	else:
		return False

if __name__=='__main__':
	n=1600000
	while True:
		n+=1
		if check(n):
			print n
			exit()
```

