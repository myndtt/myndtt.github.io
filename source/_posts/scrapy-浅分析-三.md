---
title: scrapy 浅分析(三)
date: 2018-02-01 10:05:18
tags: [scrapy,python]
categories:
---

# 0x00.前言

[前面一](http://myndtt.com/2018/01/30/scrapy-%E6%B5%85%E5%88%86%E6%9E%90-%E4%B8%80/)

[前面二](http://myndtt.com/2018/01/31/scrapy-%E6%B5%85%E5%88%86%E6%9E%90-%E4%BA%8C/)

这次看看引擎里面的东西。在此之前看看文档中的流程

```reStructuredText
Scrapy中的数据流由执行引擎控制，其过程如下:

1.引擎打开一个网站(open a domain)，找到处理该网站的Spider并向该spider请求第一个要爬取的URL(s)。
2.引擎从Spider中获取到第一个要爬取的URL并在调度器(Scheduler)以Request调度。
3引擎向调度器请求下一个要爬取的URL。
4.调度器返回下一个要爬取的URL给引擎，引擎将URL通过下载中间件(请求(request)方向)转发给下载器(Downloader)。
------------------------------------------------------------------（下面下一篇提到）
5.一旦页面下载完毕，下载器生成一个该页面的Response，并将其通过下载中间件(返回(response)方向)发送给引擎。
6.引擎从下载器中接收到Response并通过Spider中间件(输入方向)发送给Spider处理。
7.Spider处理Response并返回爬取到的Item及(跟进的)新的Request给引擎。
8.引擎将(Spider返回的)爬取到的Item给Item Pipeline，将(Spider返回的)Request给调度器。
9.(从第二步)重复直到调度器中没有更多地request，引擎关闭该网站。
```



# 0x01.引擎

![59](https://myndtt.github.io/images/59.png)

<!-- more -->

```python
class ExecutionEngine(object):

    def __init__(self, crawler, spider_closed_callback):
        self.crawler = crawler
        self.settings = crawler.settings
        #信号
        self.signals = crawler.signals
        #日志
        self.logformatter = crawler.logformatter
        self.slot = None
        self.spider = None
        self.running = False
        self.paused = False
        #scheduler_cls没有实例化
        self.scheduler_cls = load_object(self.settings['SCHEDULER'])
        downloader_cls = load_object(self.settings['DOWNLOADER'])
        #实例化
        self.downloader = downloader_cls(crawler)
        #引擎以此连接爬虫类
        self.scraper = Scraper(crawler)
        self._spider_closed_callback = spider_closed_callback
```

初始化中。分别通过配置加载调度器对象和下载器对象。初始化后执行引擎的`open_spider`方法(上篇提到了)。

```python
	@defer.inlineCallbacks
    def open_spider(self, spider, start_requests=(), close_if_idle=True):
        assert self.has_capacity(), "No free spider slot when opening %r" % \
            spider.name
        logger.info("Spider opened", extra={'spider': spider})
        #将引擎的_next_request方法传进CallLaterOnce中
        nextcall = CallLaterOnce(self._next_request, spider)
        #此时scheuler_cls开始实例化了
        scheduler = self.scheduler_cls.from_crawler(self.crawler)
        #调用spidermw的process_start_requests方法
        start_requests = yield self.scraper.spidermw.process_start_requests(start_requests, spider)
        slot = Slot(start_requests, close_if_idle, nextcall, scheduler)
        self.slot = slot
        self.spider = spider
		# 调用scheduler的open，主要去重request
        yield scheduler.open(spider)
        #调用scraper.open_spider,主要初始化一下输出的事情
        yield self.scraper.open_spider(spider)
        #stats与STATS_CLASS = 'scrapy.statscollectors.MemoryStatsCollector'有关
        #一个简单的数据收集器。其在 spider 运行完毕后将其数据保存在内存中。数据可以通过 spider_stats 			#属性访问
        self.crawler.stats.open_spider(spider)
        #信号日志
        yield self.signals.send_catch_log_deferred(signals.spider_opened, spider=spider)
        #开始(第一次)调度
        slot.nextcall.schedule()
        slot.heartbeat.start(5)
```

先看看`from scrapy.utils.reactor import CallLaterOnce` 进入reactor.py文件

```python
class CallLaterOnce(object):
    """Schedule a function to be called in the next reactor loop, but only if
    it hasn't been already scheduled since the last time it ran.
    """

    def __init__(self, func, *a, **kw):
        self._func = func
        self._a = a
        self._kw = kw
        self._call = None

    def schedule(self, delay=0):
        if self._call is None:
            self._call = reactor.callLater(delay, self)

    def cancel(self):
        if self._call:
            self._call.cancel()

    def __call__(self):
        self._call = None
        return self._func(*self._a, **self._kw)
```

按注释来看，这里利用reactor异步循环调用导入的`_next_request`方法，当运行起来时循环调用该函数。在进入`spidermw.py`文件里面看看

```python
def process_start_requests(self, start_requests, spider):
        return self._process_chain('process_start_requests', start_requests, spider)
```

一直跟进所示的函数

```python
def _process_chain(self, methodname, obj, *args):
        return process_chain(self.methods[methodname], obj, *args)
```

```python
def process_chain(callbacks, input, *a, **kw):
    """Return a Deferred built by chaining the given callbacks"""
    d = defer.Deferred()
    for x in callbacks:
        d.addCallback(x, *a, **kw)
    d.callback(input)
    return d
```

这是用户项目里`middleres.py`对应的函数

```python
def process_start_requests(start_requests, spider):
        # Called with the start requests of the spider, and works
        # similarly to the process_spider_output() method, except
        # that it doesn’t have a response associated.

        # Must return only requests (not items).
        for r in start_requests:
            yield r
```

这里什么都没有,方法被重写，其实我们都可以在其中写一些东西来处理开始请求前的事情,并将request返回。

接下来是用Slot把request进行封装

```python
class Slot(object):

    def __init__(self, start_requests, close_if_idle, nextcall, scheduler):
        self.closing = False
        self.inprogress = set() # requests in progress
        self.start_requests = iter(start_requests)
        self.close_if_idle = close_if_idle
        self.nextcall = nextcall
        self.scheduler = scheduler
        self.heartbeat = task.LoopingCall(nextcall.schedule)

    def add_request(self, request):
        self.inprogress.add(request)

    def remove_request(self, request):
        self.inprogress.remove(request)
        self._maybe_fire_closing()

    def close(self):
        self.closing = defer.Deferred()
        self._maybe_fire_closing()
        return self.closing

    def _maybe_fire_closing(self):
        if self.closing and not self.inprogress:
            if self.nextcall:
                self.nextcall.cancel()
                if self.heartbeat.running:
                    self.heartbeat.stop()
            self.closing.callback(None)
```

接下来就是调度开始：`yield scheduler.open(spider)`进入core下`scheduler.py`文件下。由此进入框架的第二步。

在此之前看看其中的`from_crawler`函数和该类初始化

```python
    @classmethod
    def from_crawler(cls, crawler):
        settings = crawler.settings
        dupefilter_cls = load_object(settings['DUPEFILTER_CLASS'])
        #实例化
        dupefilter = dupefilter_cls.from_settings(settings)
        #队列优先级
        pqclass = load_object(settings['SCHEDULER_PRIORITY_QUEUE'])
        #磁盘队列
        dqclass = load_object(settings['SCHEDULER_DISK_QUEUE'])
        #内存队列
        mqclass = load_object(settings['SCHEDULER_MEMORY_QUEUE'])
        logunser = settings.getbool('LOG_UNSERIALIZABLE_REQUESTS', settings.getbool('SCHEDULER_DEBUG'))
        return cls(dupefilter, jobdir=job_dir(settings), logunser=logunser,
                   stats=crawler.stats, pqclass=pqclass, dqclass=dqclass, mqclass=mqclass)
	def __init__(self, dupefilter, jobdir=None, dqclass=None, mqclass=None,
                 logunser=False, stats=None, pqclass=None):
        self.df = dupefilter#df来自于这个dupefilter，上面提到其在from_crawler已经实例化了
        self.dqdir = self._dqdir(jobdir)
        self.pqclass = pqclass
        self.dqclass = dqclass
        self.mqclass = mqclass
        self.logunser = logunser
        self.stats = stats

```

```python
def open(self, spider):
        self.spider = spider
    	#建立优先级队列
        self.mqs = self.pqclass(self._newmq)
      	#非空则实例化磁盘优先级队列
        self.dqs = self._dq() if self.dqdir else None
        #df来自与配置中DUPEFILTER_CLASS = 'scrapy.dupefilters.RFPDupeFilter'
        return self.df.open()
```

首先dqdir与`_dqdir`有关

```python
def _dqdir(self, jobdir):
        if jobdir:
            dqdir = join(jobdir, 'requests.queue')
            if not exists(dqdir):
                os.makedirs(dqdir)
            return dqdir
```

也即：

启动调度器时, 调度器会读取配置中的”JOBDIR”设置. 如果这个变量不存在, 则不使用磁盘队列, 而内存队列不需要这个设置, 因此, 内存队列始终存在, 而磁盘队列只有在设置了”JOBDIR”这个变量之后才会使用。

随后看看`dupefilters.py`文件下的对应那个类

```python
class RFPDupeFilter(BaseDupeFilter):
    """Request Fingerprint duplicates filter"""

    def __init__(self, path=None, debug=False):
        self.file = None
        self.fingerprints = set()
        self.logdupes = True
        self.debug = debug
        self.logger = logging.getLogger(__name__)
        if path:
            self.file = open(os.path.join(path, 'requests.seen'), 'a+')
            self.file.seek(0)
            self.fingerprints.update(x.rstrip() for x in self.file)

    @classmethod
    def from_settings(cls, settings):
        debug = settings.getbool('DUPEFILTER_DEBUG')
        return cls(job_dir(settings), debug)

    def request_seen(self, request):
        fp = self.request_fingerprint(request)
        if fp in self.fingerprints:
            return True
        self.fingerprints.add(fp)
        if self.file:
            self.file.write(fp + os.linesep)

    def request_fingerprint(self, request):
        return request_fingerprint(request)

    def close(self, reason):
        if self.file:
            self.file.close()

    def log(self, request, spider):
        if self.debug:
            msg = "Filtered duplicate request: %(request)s"
            self.logger.debug(msg, {'request': request}, extra={'spider': spider})
        elif self.logdupes:
            msg = ("Filtered duplicate request: %(request)s"
                   " - no more duplicates will be shown"
                   " (see DUPEFILTER_DEBUG to show all duplicates)")
            self.logger.debug(msg, {'request': request}, extra={'spider': spider})
            self.logdupes = False

        spider.crawler.stats.inc_value('dupefilter/filtered', spider=spider)
```

按注释来其即对重复的请求进行过滤，即：

scrpay默认使用自带的去重组件为”RFPDupeFilter”(请求指纹重复过滤器). 这个组件通过python自带的set数据类型, 通过判断新请求链接是否在”集合”中, 来判断这个请求链接是否重复. `yield scheduler.open(spider`执行完后就是` yield self.scraper.open_spider(spider)`进入core文件夹下`scrapy.py`文件得：

```python
@defer.inlineCallbacks
    def open_spider(self, spider):
        """Open the given spider for scraping and allocate resources for it"""
        self.slot = Slot()
        yield self.itemproc.open_spider(spider)
```

需要调用itemproc的open_spider方法，从文件中找到如下信息：

```python
itemproc_cls = load_object(crawler.settings['ITEM_PROCESSOR'])
self.itemproc = itemproc_cls.from_crawler(crawler)
ITEM_PROCESSOR = 'scrapy.pipelines.ItemPipelineManager'
```

则进入pipelines文件夹对应文件下

```python
class ItemPipelineManager(MiddlewareManager):

    component_name = 'item pipeline'

    @classmethod
    def _get_mwlist_from_settings(cls, settings):
        return build_component_list(settings.getwithbase('ITEM_PIPELINES'))

    def _add_middleware(self, pipe):
        super(ItemPipelineManager, self)._add_middleware(pipe)
        if hasattr(pipe, 'process_item'):
            self.methods['process_item'].append(pipe.process_item)

    def process_item(self, item, spider):
        return self._process_chain('process_item', item, spider)
```

加入中间件，我们知道pipeline处理输出，这里加一些中间件，做一点初始化工作。最后回到调度这里开始

`slot.nextcall.schedule()`,前面说了这里会调用`_next_request`方法，设计到的主要函数如下

```Python
def crawl(self, request, spider):
        assert spider in self.open_spiders, \
            "Spider %r not opened when crawling: %s" % (spider.name, request)
      #进行调度
        self.schedule(request, spider)
        #下一次调度
        self.slot.nextcall.schedule()
def schedule(self, request, spider):
        self.signals.send_catch_log(signal=signals.request_scheduled,
                request=request, spider=spider)
    	#request放入调度中
        if not self.slot.scheduler.enqueue_request(request):
            self.signals.send_catch_log(signal=signals.request_dropped,
                                        request=request, spider=spider)
#具体request进入队列情况，core文件夹scheduler.py和default_setting下均有，代码一样，可重写
#其中上面有提到的两种形式内存或者磁盘，同时还会就是重复过滤操作
def enqueue_request(self, request):
        if not request.dont_filter and self.df.request_seen(request):
            self.df.log(request, self.spider)
            return False
        dqok = self._dqpush(request)
        if dqok:
            self.stats.inc_value('scheduler/enqueued/disk', spider=self.spider)
        else:
            self._mqpush(request)
            self.stats.inc_value('scheduler/enqueued/memory', spider=self.spider)
        self.stats.inc_value('scheduler/enqueued', spider=self.spider)
        return True
def _next_request(self, spider):
        slot = self.slot
        if not slot:
            return
		#暂停
        if self.paused:
            return
		#等待
        while not self._needs_backout(spider):
          #第一次调度中无下一个，等后面放进去了就有了
            if not self._next_request_from_scheduler(spider):
                break
		#下一次有request并且不需要等待
        if slot.start_requests and not self._needs_backout(spider):
            try:
              #拿出下一次request
                request = next(slot.start_requests)
            except StopIteration:
                slot.start_requests = None
            except Exception:
                slot.start_requests = None
                logger.error('Error while obtaining start requests',
                             exc_info=True, extra={'spider': spider})
            else:
              #没错误就执行crawl，将request放入调度中
                self.crawl(request, spider)
		#空则关闭
        if self.spider_is_idle(spider) and slot.close_if_idle:
            self._spider_idle(spider)
 def _needs_backout(self, spider):
        slot = self.slot
    	#等待条件
        return not self.running \
            or slot.closing \
            or self.downloader.needs_backout() \
            or self.scraper.slot.needs_backout()

 def _next_request_from_scheduler(self, spider):
        slot = self.slot
        request = slot.scheduler.next_request()
        if not request:
            return
        #去下载
        d = self._download(request, spider)
        d.addBoth(self._handle_downloader_output, request, spider)
        d.addErrback(lambda f: logger.info('Error while handling downloader output',
                                           exc_info=failure_to_exc_info(f),
                                           extra={'spider': spider}))
        d.addBoth(lambda _: slot.remove_request(request))
        d.addErrback(lambda f: logger.info('Error while removing request from slot',
                                           exc_info=failure_to_exc_info(f),
                                           extra={'spider': spider}))
        d.addBoth(lambda _: slot.nextcall.schedule())
        d.addErrback(lambda f: logger.info('Error while scheduling new request',
                                           exc_info=failure_to_exc_info(f),
                                           extra={'spider': spider}))
        return d
 def spider_is_idle(self, spider):
        if not self.scraper.slot.is_idle():
            # scraper is not idle
            return False

        if self.downloader.active:
            # downloader has pending requests
            return False

        if self.slot.start_requests is not None:
            # not all start requests are handled
            return False

        if self.slot.scheduler.has_pending_requests():
            # scheduler has pending requests
            return False

        return True
```

到此，官方框架的第三步也结束了，要去做第四步下载。

# 0x02小结：

这次就是这么几个过程：

```
1.引擎打开一个网站(open a domain)，找到处理该网站的Spider并向该spider请求第一个要爬取的URL(s)。
2.引擎从Spider中获取到第一个要爬取的URL并在调度器(Scheduler)以Request调度。
3。引擎向调度器请求下一个要爬取的URL。
4.调度器返回下一个要爬取的URL给引擎，引擎将URL通过下载中间件(请求(request)方向)转发给下载器(Downloader)。
```

调度器将请求弄好成一个队列形式，加上去重处理，待引擎调用时返回给引擎，其中会有一些pipelines的简单初始化和中间件等处理。许多地方可由自己操作，如引擎给调度器request可以作处理，去重与否，队列采取与否都可以自己设置。