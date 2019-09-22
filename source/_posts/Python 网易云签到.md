---
title: python-网易云音乐签到，词云
id: 174
date: 2017-08-29 17:03:40
tags:
	- python
---

时间过的真快，8月份悄悄地就没了，这段时间一直在回顾以前的电子书，做一个累计，也在一点点找回说英文和写汉字的感觉，电脑打字久了，感觉自己都不会提笔了...
一：网易云签到
就是模拟浏览器去做这些个事情,其实就是练练手，没多大意义,代码如下

<!-- more -->

```python
# -*- coding: utf-8 -*-
import traceback
from selenium import webdriver
#coding=utf-8
import selenium.webdriver.support.ui as ui
from selenium.webdriver.common.desired_capabilities import DesiredCapabilities
import time
import traceback
import random
def dengru(url):
	#driver = webdriver.PhantomJS()#,executable_path='/Users/mrlevo/phantomjs-2.1.1-macosx/bin/phantomjs')  # 注意填上路径
	driver = webdriver.Chrome()
	driver.get(url)
	try:
		driver.find_element_by_link_text(u'登录').click()
		wait = ui.WebDriverWait(driver,5)
		wait.until(lambda driver: driver.find_element_by_class_name('u-btn2-2').is_displayed())  # 根据link获取元素
		driver.find_element_by_class_name('u-btn2-2').click()
		driver.find_element_by_css_selector("input[placeholder=\"请输入手机号\"]").send_keys('###')
		driver.find_element_by_css_selector("input[placeholder=\"请输入密码\"]").send_keys('###')
		time.sleep(0.2)
		driver.find_element_by_class_name('js-primary').click()
		wait = ui.WebDriverWait(driver,5)
		driver.switch_to_frame('g_iframe')
		wait.until(lambda driver: driver.find_element_by_class_name('sign').is_displayed())  # 根据link获取元素
		driver.find_element_by_class_name('sign').click()
		driver.quit()
	except Exception as ex:
		traceback.print_exc()
	finally:
		driver.quit()
if __name__ == '__main__':
	for url in ['http://music.163.com/']:  
		time.sleep(random.randint(2, 4)) 
		url_playlist = dengru(url)
```


二：词云，也是练手，看看自己网易云喜欢的音乐中的歌手记录（这个网上有，造着别人做的）
做出来结果是：

![](http://101.200.62.181:8080/wp-content/uploads/2017/08/mymusic2-300x169.png)
看我喜欢的歌手是 “群星”
其实呢 python中有一个库装门用来“玩”网易云音乐的，但是呢，我没有用过，这里给个链接

[http://xiyoumc.0x2048.com/ncmbot/#/?id=ncmbot](http://xiyoumc.0x2048.com/ncmbot/#/?id=ncmbot)