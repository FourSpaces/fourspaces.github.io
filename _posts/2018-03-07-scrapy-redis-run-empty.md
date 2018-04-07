---
layout: post
title: Scrapy-Redis 空跑问题，redis_key链接跑完后，自动关闭爬虫
key: 20180307
tags: scrapy-redis scrapy-redis空跑 爬虫结束 python爬虫
---

## Scrapy-Redis 空跑问题，redis_key链接跑完后，自动关闭爬虫

### 问题：
- scrapy-redis框架中，reids存储的xxx:requests已经爬取完毕，但程序仍然一直运行，如何自动停止程序，结束空跑。

相信大家都很头疼，尤其是网上一堆搬来搬去的帖子，来看一下 我是如何解决这个问题的吧

### 课外了解

分布式扩展：

我们知道 scrapy 默认是单机运行的，那么scrapy-redis是如何把它变成可以多台机器协作的呢？

**首先解决爬虫等待，不被关闭的问题：**

1、scrapy内部的信号系统会在爬虫耗尽内部队列中的request时，就会触发spider_idle信号。

2、爬虫的信号管理器收到spider_idle信号后，将调用注册spider_idle信号的**处理器**进行处理。

3、当该信号的所有处理器(handler)被调用后，如果spider仍然保持空闲状态， 引擎将会关闭该spider。

scrapy-redis 中的解决方案
在信号管理器上注册一个对应在spider_idle信号下的spider_idle()方法，当spider_idle触发是，信号管理器就会调用这个爬虫中的spider_idle()， Scrapy_redis 源码如下：



```
    def spider_idle(self):
        """Schedules a request if available, otherwise waits."""
        # XXX: Handle a sentinel to close the spider.
        self.schedule_next_requests()    # 这里调用schedule_next_requests() 来从redis中生成新的请求
        raise DontCloseSpider              # 抛出不要关闭爬虫的DontCloseSpider异常，保证爬虫活着

```

### 解决思路：
- 通过前面的了解，我们知道 爬虫关闭的关键是 spider_idle 信号。
- spider_idle信号只有在爬虫队列为空时才会被触发， 触发间隔为5s。
- 那么我们也可以使用同样的方式，在信号管理器上注册一个对应在spider_idle信号下的spider_idle()方法。
- 在 spider_idle() 方法中，编写结束条件来结束爬虫

**解决方案：**
- redis_key 为空后一段时间关闭爬虫

### redis_key 为空后一段时间关闭爬虫 的实现方案：

这里在 Scrapy 中的 exensions（扩展） 中实现，当然你也可以在pipelines（管道）中实现。

扩展框架提供一个机制，使得你能将自定义功能绑定到Scrapy。
扩展只是正常的类，它们在Scrapy启动时被实例化、初始化。
关于扩展详细见： [scrapy 扩展(Extensions)](http://scrapy-chs.readthedocs.io/zh_CN/1.0/topics/extensions.html)

- 在settings.py 文件的目录下，创建一个名为 extensions.py 的文件，
- 在其中写入以下代码

```python
# -*- coding: utf-8 -*-
# Define here the models for your scraped Extensions
import logging
import time
from scrapy import signals
from scrapy.exceptions import NotConfigured
logger = logging.getLogger(__name__)


class RedisSpiderSmartIdleClosedExensions(object):

    def __init__(self, idle_number, crawler):
        self.crawler = crawler
        self.idle_number = idle_number
        self.idle_list = []
        self.idle_count = 0

    @classmethod
    def from_crawler(cls, crawler):
        # 首先检查是否应该启用和提高扩展
        # 否则不配置
        if not crawler.settings.getbool('MYEXT_ENABLED'):
            raise NotConfigured

        # 获取配置中的时间片个数，默认为360个，30分钟
        idle_number = crawler.settings.getint('IDLE_NUMBER', 360)

        # 实例化扩展对象
        ext = cls(idle_number, crawler)

        # 将扩展对象连接到信号， 将signals.spider_idle 与 spider_idle() 方法关联起来。
        crawler.signals.connect(ext.spider_opened, signal=signals.spider_opened)
        crawler.signals.connect(ext.spider_closed, signal=signals.spider_closed)
        crawler.signals.connect(ext.spider_idle, signal=signals.spider_idle)

        # return the extension object
        return ext

    def spider_opened(self, spider):
        logger.info("opened spider %s redis spider Idle, Continuous idle limit： %d", spider.name, self.idle_number)

    def spider_closed(self, spider):
        logger.info("closed spider %s, idle count %d , Continuous idle count %d",
                    spider.name, self.idle_count, len(self.idle_list))

    def spider_idle(self, spider):
        self.idle_count += 1                        # 空闲计数
        self.idle_list.append(time.time())       # 每次触发 spider_idle时，记录下触发时间戳
        idle_list_len = len(self.idle_list)         # 获取当前已经连续触发的次数

        # 判断 当前触发时间与上次触发时间 之间的间隔是否大于5秒，如果大于5秒，说明redis 中还有key
        if idle_list_len > 2 and self.idle_list[-1] - self.idle_list[-2] > 6:
            self.idle_list = [self.idle_list[-1]]

        elif idle_list_len > self.idle_number:
            # 连续触发的次数达到配置次数后关闭爬虫
            logger.info('\n continued idle number exceed {} Times'
                        '\n meet the idle shutdown conditions, will close the reptile operation'
                        '\n idle start time: {},  close spider time: {}'.format(self.idle_number,
                                                                              self.idle_list[0], self.idle_list[0]))
            # 执行关闭爬虫操作
            self.crawler.engine.close_spider(spider, 'closespider_pagecount')

```
- 在settings.py 中添加以下配置， 请将 lianjia_ershoufang 替换为你的项目目录名。

```python

MYEXT_ENABLED=True      # 开启扩展
IDLE_NUMBER=360           # 配置空闲持续时间单位为 360个 ，一个时间单位为5s

# 在 EXTENSIONS 配置，激活扩展
'EXTENSIONS'= {
            'lianjia_ershoufang.extensions.RedisSpiderSmartIdleClosedExensions': 500,
        },

```

- 完成空闲关闭扩展，爬虫会在持续空闲 360个时间单位后关闭爬虫

配置说明：

```
MYEXT_ENABLED: 是否启用扩展，启用扩展为 True， 不启用为 False
IDLE_NUMBER: 关闭爬虫的持续空闲次数，持续空闲次数超过IDLE_NUMBER，爬虫会被关闭。默认为 360 ，也就是30分钟，一分钟12个时间单位
```

### 结语
此方法只使用于 5秒内跑不完一组链接的情况，如果你的一组链接5秒就能跑完，你可以在此基础上做一些判断。原理一样，大家可以照葫芦画瓢。

哈哈，我的方式是不是特别棒呀！


### 参考链接
[scrapy与redis结合实现服务化的分布式爬虫](http://blog.csdn.net/gklifg/article/details/54950028)