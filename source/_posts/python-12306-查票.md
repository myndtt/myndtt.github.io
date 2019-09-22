---
title: python-12306-查票
id: 128
date: 2017-07-16 12:58:25
tags:
    - python
---

已经是暑期了，许多小伙伴们都已经回家，说到回家突然想到以前的12306查余票的python脚本不能用了，是官方的json变了，不管怎么变，其实我们要的信息都在那里面就好了，关键是如何提取出来。借助之前网上有的例子。我们首先得到城市站名与其对应得编号（这个网上有），关键是这条链接[https://kyfw.12306.cn/otn/resources/js/framework/station_name.js?station_version=1.9006](https://kyfw.12306.cn/otn/resources/js/framework/station_name.js?station_version=1.9006)，然后正则匹配出来即可。接下来就是如何解析了，其实不要太多花里胡哨的功能，本来就是用来用的，不是好看的。拿自己本人为例找一找成都到南昌的：

<!-- more -->

```python
#coding:utf-8
#from stations import stations
import requests
from prettytable import PrettyTable

header = u'车次 车站 时间 历时 一等 二等 高级软卧 软卧 硬卧 硬座 无座'.split()
def userinfo():
    fromstation='CDW'#初始站编号
    tostation='NCG'#到达站编号
    date='2017-07-25'#时间
    url='https://kyfw.12306.cn/otn/leftTicket/query?leftTicketDTO.train_date={}&leftTicketDTO.from_station={}&leftTicketDTO.to_station={}&purpose_codes=ADULT'.format(date, fromstation, tostation)
    #try:
    res=requests.get(url,verify=False)
    #except requests.RequestException as e:
        #print "故障发生"
    #print res.json()
    resulttrains=res.json()['data']['result']#获取火车信息
    resultstation=res.json()['data']['map']#获取车站编号信息
    #print date+stations.get(fromstation)+"至"+stations.get(tostation)+"火车信息"需要反转字典
    print date+u"成都至南昌火车表"
    rprintit(resulttrains,resultstation)
    
def rprintit(resulttrains,resultstation):
    pt = PrettyTable()
    pt._set_field_names(header)#表的标题
    for train in resulttrains:
        #print train
        onetrain=[]
        traininfo=train.split("|")#用|分割信息
        a=traininfo[3]#车次
        onetrain.append(a)
        onetrain.append('\n'.join([resultstation[traininfo[6]],resultstation[traininfo[7]]]))#初始站
        #print trainfromandtostaion
        onetrain.append('\n'.join([traininfo[8],traininfo[9]]))#初始时间
        onetrain.append(traininfo[10])#总时间
        #if 'K' in a:
            #print train
        trainfirst=(traininfo[-5] if traininfo[-5] else '--')
        #print trainfirst
        trainsecond=(traininfo[-6] if traininfo[-6] else '--')
        traingreatsoft=(traininfo[-14] if traininfo[-14] else '--')
        trainsoftbed=(traininfo[-13] if traininfo[-13] else '--')
        trainhardbed=(traininfo[-8] if traininfo[-8] else '--')
        trainhardseat=(traininfo[-7] if traininfo[-7] else '--')
        trainnoseat=(traininfo[-10] if traininfo[-10] else '--')
        onetrain.append(trainfirst)
        onetrain.append(trainsecond)
        onetrain.append(traingreatsoft)
        onetrain.append(trainsoftbed)
        onetrain.append(trainhardbed)
        onetrain.append(trainhardseat)
        onetrain.append(trainnoseat)
        pt.add_row(onetrain)
    print(pt)
if __name__=='__main__':
    userinfo()
```
运行结果

![](http://101.200.62.181/wp-content/uploads/2017/07/NOPFCT7B6DMR5Z@OY.png)

&nbsp;

&nbsp;

&nbsp;

&nbsp;

实际结果

![](http://101.200.62.181/wp-content/uploads/2017/07/6GXM0@XLM1B3NEWYJRI0.png)