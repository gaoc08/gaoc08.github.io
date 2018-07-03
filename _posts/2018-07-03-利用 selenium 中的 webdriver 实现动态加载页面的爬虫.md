---
layout:     post
title:      利用 selenium 中的 webdriver 实现动态加载页面的图片爬虫
subtitle:   
date:       2018-07-03
author:     G
header-img: img/post-bg-2015.jpg
catalog: true
tags:
    - python
    - 爬虫
    - selenium
    - webdriver
    - 动态页面爬虫
---

#背景

对于一个静态页面，我们可以很容易的取到页面的 html，然后通过正则表达式找到我们所需要的图片的url，然后下载，达到爬图片的目的。

对于动态加载的页面，比如需要下滑才会 loadmore 的页面，我们就无法使用上面的简单方式完成目标。

# Selenium

> Selenium 是一款强大的基于浏览器的开源自动化测试工具，最初由 Jason Huggins 于 2004 年在 ThoughtWorks 发起，它提供了一套简单易用的 API，模拟浏览器的各种操作，方便各种 Web 应用的自动化测试。它的取名很有意思，因为当时最流行的一款自动化测试工具叫做 QTP，是由 Mercury 公司开发的商业应用。Mercury 是化学元素汞，而 Selenium 是化学元素硒，汞有剧毒，而硒可以解汞毒，它对汞有拮抗作用。

> Selenium 的核心组件叫做 Selenium-RC（Remote Control），简单来说它是一个代理服务器，浏览器启动时通过将它设置为代理，它可以修改请求响应报文并向其中注入 Javascript，通过注入的 JS 可以模拟浏览器操作，从而实现自动化测试。但是注入 JS 的方法存在很多限制，譬如无法模拟键盘和鼠标事件，处理不了对话框，不能绕过 JavaScript 沙箱等等。就在这个时候，于 2006 年左右，Google 的工程师 Simon Stewart 发起了 WebDriver 项目，WebDriver 通过调用浏览器提供的原生自动化 API 来驱动浏览器，解决了 Selenium 的很多疑难杂症。不过 WebDriver 也有它不足的地方，它不能支持所有的浏览器，需要针对不同的浏览器来开发不同的 WebDriver，因为不同的浏览器提供的 API 也不尽相同，好在经过不断的发展，各种主流浏览器都已经有相应的 WebDriver 了。最终 Selenium 和 WebDriver 合并在一起，这就是 Selenium 2.0，有的地方也直接把它称作 WebDriver。Selenium 目前最新的版本已经是 3.9 了，WebDriver 仍然是 Selenium 的核心。

从上面介绍上我们可以看到，[Selenium](http://selenium-python.readthedocs.io/) 可以通过模拟用户操作来实现动态网页的图片爬虫。

# WebDriver 的安装

1. 按照官网教程安装 selenium。
2. 下载相应浏览器的 webdriver 文件。本文以 mac 环境下的 chromedriver 为例。`brew install chromedriver` 直接安装。


# 实现

1. 初始化 webdriver
	
	```
	options = Options()
	options.add_argument("--headless")
	driver = webdriver.Chrome(options = options)
	driver.get(url)
	```
2. 获取初始页面的 html
	
	```
	html = driver.page_source
	```
3. 定义个 loadmore 的函数.

	```
	def load_more(driver):
		driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")
		time.sleep(5)
	```
	> 这里用 time sleep 不好，应该用 WebDriverWait，有时间再优化。

4. loadmore 之后获取 loadmore 之后的 html。

	```
	def getNextPageImages(driver):
		load_more(driver)
		html = driver.execute_script("return document.documentElement.outerHTML;")
		if not html:
			return
		getImage(html)
		time.sleep(5)
	```
	> 对这个函数做循环，就可以不停的 loadmore 并爬图片啦。

# 注意!!!

其实上面说了那么多，最主要想说的就是最后这个 *注意事项*，也是这个点花了我大量的时间：

- `driver.page_source` 只会获取动态加载前的html。
- 如果想要获取动态加载后的 html 需要使用 `driver.execute_script("return document.documentElement.outerHTML;")` 来实现。

