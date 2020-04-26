---
layout:     post               
title:      Scrapy+Selenium+Phantomjs的Demo
date:       2016-08-27       
author:     ChayCao    
header-img: img/post-bg-2015.jpg 
catalog: true 
tags:  Python                            
---

　　前段时间学习了用Python写爬虫，使用Scrapy框架爬取京东的商品信息。商品详情页的价格是由js生成的，而通过Scrapy直接爬取的源文件中无价格信息。
　　通过Selenium、Phantomjs便能实现。下面先介绍Phantomjs。
### Phantomjs
　　作为一个基于webkit内核的没有UI界面的**浏览器**，大家不必被这个名字吓到，以为是很复杂的技术，其实只是一个浏览器啦。而其中的一些点击、翻页等操作则由代码实现。Phantomjs提供javascript API接口，即我们可以通过js与webkit内核交互。
　　只需通过Phantomjs test.js的命令就能成功访问页面，并将js生成后的内容下载下来。
```javascript
// a phantomjs example
// test.js
var page = require('webpage').create();  //获取操作dom或web网页的对象
var fs = require('fs');                  //获取文件系统对象，通过它可以操作操作系统的文件操作，包括read、write、move、copy、delete等。
phantom.outputEncoding="utf-8";
//通过page对象打开url链接，并可以回调其声明的回调函数
page.open("http://chaycao.github.io", function(status) {   
if ( status === "success" ) {
	console.log("成功");
	fs.write('web.html',page.content,'w');
	} else {
	console.log("Page failed to load.");
}
phantom.exit(0);
}); 
```
　　关于Phantomjs的详细介绍可以参照这篇博文：http://blog.csdn.net/tengdazhang770960436/article/details/41320079
### Selenium
　　作为一个用于Web应用程序测试的工具，其测试直接运行在浏览器中，框架底层使用JavaScript模拟真实用户对浏览器的操作，从终端用户的角度测试应用程序。将Selenium与Phantomjs联系起来，便是我们可以**通过使用Selenium操作Phantomjs**访问网页以获得js生成后的网页。

　　在我看来，这里Selenium的作用就像是Scrapy调用Phantomjs的中介。我也存有疑问：*“Scrapy是否能直接调用Phantomjs？”* 期望回复

### Scrapy使用Selenium
需要弄懂一个概念，**下载中间件**，可以阅读下官方文档http://scrapy-chs.readthedocs.io/zh_CN/1.0/topics/downloader-middleware.html

在settings.py加入
```python
DOWNLOADER_MIDDLEWARES = {
    'jdSpider.middlewares.middleware.JavaScriptMiddleware': 543, #键为中间件类的路径，值为中间件的顺序
    'scrapy.downloadermiddlewares.useragent.UserAgentMiddleware':None, #禁止内置的中间件
}
```
第一行为中间件的位置，需要与自己目录对应，我的目录是这样的：
![文件目录](http://upload-images.jianshu.io/upload_images/2489662-fce77b54a9fca3fc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果运行遇到robots.txt的问题，需要将settings.py中的
```python
ROBOTSTXT_OBEY = False
```
下面贴上，自定义中间件的代码：
```python
from selenium import webdriver
from scrapy.http import HtmlResponse
import time

class JavaScriptMiddleware(object):

def process_request(self, request, spider):
    if spider.name == "jd":
        print "PhantomJS is starting..."
        driver = webdriver.PhantomJS() #指定使用的浏览器
        # driver = webdriver.Firefox()
        driver.get(request.url)
        time.sleep(1)
        js = "var q=document.documentElement.scrollTop=10000" 
        driver.execute_script(js) #可执行js，模仿用户操作。此处为将页面拉至最底端。       
        time.sleep(3)
        body = driver.page_source
        print ("访问"+request.url)
        return HtmlResponse(driver.current_url, body=body, encoding='utf-8', request=request)
    else:
        return
```

在spiders/jd.py中parse()方法接收到的response则是我们自定义中间件返回的结果。我们得到的便是js生成后的界面。
```python
# -*- coding: utf-8 -*-
import scrapy

class JdSpider(scrapy.Spider):
    name = "jd"
    allowed_domains = ["jd.com"]
    start_urls = (
        'http://search.jd.com/Search?keyword=三星s7&enc=utf-8&wq=三星s7&pvid=tj0sfuri.v70avo',
    )

    def parse(self, response):
        print response.body
```