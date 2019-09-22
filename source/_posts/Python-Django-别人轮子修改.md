---
title: python Django别人的轮子修改
id: 140
date: 2017-07-22 14:20:37
tags:
	 - python
---

最近突然想用python写一写微信公众号后台，说起微信挺那个啥的，公众号接口似乎必须占着80端口。不知道为毛线。扯远了，由于以前用JAVA写过微信公众号后台，当时的一些基本回复消息还有事件的处理都是有现成的轮子，这一部分都封装好了的，只要到时根据自己需要修改增加相应代码即可。我就想会不会有python写的这一方面轮子。果然有[http://ningning.today/2015/02/21/python/django-python%E5%BE%AE%E4%BF%A1%E5%BC%80%E5%8F%91%E4%B9%8B%E4%B8%80%EF%BC%8D%E5%87%86%E5%A4%87%E5%B7%A5%E4%BD%9C/](http://ningning.today/2015/02/21/python/django-python%E5%BE%AE%E4%BF%A1%E5%BC%80%E5%8F%91%E4%B9%8B%E4%B8%80%EF%BC%8D%E5%87%86%E5%A4%87%E5%B7%A5%E4%BD%9C/)，其实这些个轮子都是根据微信的开发文档设计而来的，写出来难度不大，只是要时间而已。然后我就用了这个轮子，就有了接下来的故事。

<!-- more -->

将这位大佬前辈的代码移植好后发现简单的文本回复没有问题，但是文章和音乐回复有问题，不应该啊，原以为是我方法没用对，后来检查无误，我就对这些代码看了看，发现问题出现在这位大佬自己写的一个消息类转为xml的方法上




```python
          else:
                    for tmpkey, tmpvalue in vars(obj.__getattribute__(key)).items():
                        #print tmproot,tmpkey
                        tmpkey_ele = etree.SubElement(tmproot, tmpkey)
                    if u'time' in tmpkey.lower() or u'count' in tmpkey.lower():
                           tmpkey_ele.text = unicode(tmpvalue)
                    else:    # CDATA tag for str
                           tmpkey_ele.text = etree.CDATA(unicode(tmpvalue))
                           #print tmpkey_ele.text,tmpvalue
```
这里的if else语句一定是要针对于每一个temkey_ele的，所以if else应该在for语句里面，将其缩进进去即可，拿出wechatUtil和wechatReply两个文件，写个简单测试类

```python
#coding=utf-8
from wechatReply import *
from wechatUtil import *
import time
t=MusicReply()
s=Music()
s.setTitle(u'123')
t.setToUserName(u"wo")
t.setFromUserName(u"ni")
t.setMsgType(u'music')
t.setCreateTime(time.time())
s.setDescription(u'wangyiyun')
s.setMusicUrl(u'123')
t.setMusic(s)
print MessageUtil.class2xml(t)#修改前
print "-------------"
print MessageUtil.class2xmlself(t)#修改后
```


结果：

![](http://101.200.62.181:8080/wp-content/uploads/2017/07/OZ5S4NI8P@F8A3260W2.png)

写在后面：

平凡不可怕 就怕甘于平庸