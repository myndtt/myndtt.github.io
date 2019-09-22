---
title: scrapy 浅分析(四)
date: 2018-02-01 19:45:11
tags: [scrapy,python]
categories:
---

# 0x00.前言

[前面一](http://myndtt.com/2018/01/30/scrapy-%E6%B5%85%E5%88%86%E6%9E%90-%E4%B8%80/)

[前面二](http://myndtt.com/2018/01/31/scrapy-%E6%B5%85%E5%88%86%E6%9E%90-%E4%BA%8C/)

[前面三](http://myndtt.com/2018/02/01/scrapy-%E6%B5%85%E5%88%86%E6%9E%90-%E4%B8%89/)

这次接着上回要提到下载的地方

```python
def _next_request_from_scheduler(self, spider):
        slot = self.slot
        request = slot.scheduler.next_request()
        if not request:
            return
        #去下载
        d = self._download(request, spider)
        #处理下载返回结果
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
```

<!-- more -->

列出相关函数得

```python
def _download(self, request, spider):
        slot = self.slot
        slot.add_request(request)
        def _on_success(response):
           #成功返回
            assert isinstance(response, (Response, Request))
            #返回的是response则执行如下操作
            if isinstance(response, Response):
                response.request = request # tie request to response received
                logkws = self.logformatter.crawled(request, response, spider)
                logger.log(*logformatter_adapter(logkws), extra={'spider': spider})
                self.signals.send_catch_log(signal=signals.response_received, \
                    response=response, request=request, spider=spider)
            return response

        def _on_complete(_):
          #完成后进行下一次
            slot.nextcall.schedule()
            return _
		#去执行那个downloader的函数
        dwld = self.downloader.fetch(request, spider)
        #回调到上面那个函数
        dwld.addCallbacks(_on_success)
        #回调到上面那个函数
        dwld.addBoth(_on_complete)
        return dwld
```

更进`fetch`函数

```python
def fetch(self, request, spider):
        def _deactivate(response):
      		#删除这个request
            self.active.remove(request)
            return response
		#加入该request
        self.active.add(request)
        #调用下载的中间件，回调到_enqueue_request函数
        dfd = self.middleware.download(self._enqueue_request, request, spider)
        #最后调用上面那个函数也就是删除操作
        return dfd.addBoth(_deactivate)
```

OK，进入`download`函数

```Python
def download(self, download_func, request, spider):
        @defer.inlineCallbacks
        def process_request(request):
        #按字面意思来如果中间件有这个方法就执行
        #这里需要说明request，response，下载器要对调度器的请求和下载后的返回结果进行不同对待
            for method in self.methods['process_request']:
                response = yield method(request=request, spider=spider)
                assert response is None or isinstance(response, (Response, Request)), \
                        'Middleware %s.process_request must return None, Response or Request, got %s' % \
                        (six.get_method_self(method).__class__.__name__, response.__class__.__name__)
                if response:
                    defer.returnValue(response)
             #否则执行download_func,这个就是传进来的self._enqueue_request
            defer.returnValue((yield download_func(request=request,spider=spider)))

        @defer.inlineCallbacks
        def process_response(response):
        #按字面意思来如果中间件有这个方法就执行，后面同理
            assert response is not None, 'Received None in process_response'
            if isinstance(response, Request):
                defer.returnValue(response)

            for method in self.methods['process_response']:
                response = yield method(request=request, response=response,
                                        spider=spider)
                assert isinstance(response, (Response, Request)), \
                    'Middleware %s.process_response must return Response or Request, got %s' % \
                    (six.get_method_self(method).__class__.__name__, type(response))
                if isinstance(response, Request):
                    defer.returnValue(response)
            defer.returnValue(response)

        @defer.inlineCallbacks
        def process_exception(_failure):
            exception = _failure.value
            for method in self.methods['process_exception']:
                response = yield method(request=request, exception=exception,
                                        spider=spider)
                assert response is None or isinstance(response, (Response, Request)), \
                    'Middleware %s.process_exception must return None, Response or Request, got %s' % \
                    (six.get_method_self(method).__class__.__name__, type(response))
                if response:
                    defer.returnValue(response)
            defer.returnValue(_failure)
		
        deferred = mustbe_deferred(process_request, request)
        #差错检测
        deferred.addErrback(process_exception)
        #进行回调
        deferred.addCallback(process_response)
        return deferred
```

等于说下载时，对request进行一些中间件操作，操作完以后进入`_enqueue_request`函数，如下：

```python
def _enqueue_request(self, request, spider):
        key, slot = self._get_slot(request, spider)
        request.meta['download_slot'] = key

        def _deactivate(response):
            slot.active.remove(request)
            return response
		
        slot.active.add(request)
        deferred = defer.Deferred().addBoth(_deactivate)
        #加入下载队列
        slot.queue.append((request, deferred))
        #处理队列
        self._process_queue(spider, slot)
        return deferred
```

跟进`_process_queue`,相关函数也一并列出

```python
def _process_queue(self, spider, slot):
        if slot.latercall and slot.latercall.active():
            return

        # Delay queue processing if a download_delay is configured
        now = time()
        delay = slot.download_delay()
        if delay:
            penalty = delay - now + slot.lastseen
            if penalty > 0:
                slot.latercall = reactor.callLater(penalty, self._process_queue, spider, slot)
                return

        # Process enqueued requests if there are free slots to transfer for this slot
        while slot.queue and slot.free_transfer_slots() > 0:
            slot.lastseen = now
            request, deferred = slot.queue.popleft()
            #开始真正下载
            dfd = self._download(slot, request, spider)
            dfd.chainDeferred(deferred)
            # prevent burst if inter-request delays were configured
            if delay:
                self._process_queue(spider, slot)
                break
def _download(self, slot, request, spider):
        # The order is very important for the following deferreds. Do not change!

        # 1. Create the download deferred
      #注册handlers下的download_request方法
        dfd = mustbe_deferred(self.handlers.download_request, request, spider)

        # 2. Notify response_downloaded listeners about the recent download
        # before querying queue for next request
        def _downloaded(response):
            self.signals.send_catch_log(signal=signals.response_downloaded,
                                        response=response,
                                        request=request,
                                        spider=spider)
            #调用后返回
            return response
         #注册好了就调用
        dfd.addCallback(_downloaded)

        # 3. After response arrives,  remove the request from transferring
        # state to free up the transferring slot so it can be used by the
        # following requests (perhaps those which came from the downloader
        # middleware itself)
        slot.transferring.add(request)
		#结束后的操作
        def finish_transferring(_):
            slot.transferring.remove(request)
            #处理队列
            self._process_queue(spider, slot)
            return _

        return dfd.addBoth(finish_transferring)
```

官方注释的很清楚啊！！先跟进`download_request`方法，列出相关函数

```python
def download_request(self, request, spider):
  		#获取scheme，也就是连接是什么http,https,ftp之类的
        scheme = urlparse_cached(request).scheme
      #不同的schema返回不同
        handler = self._get_handler(scheme)
        if not handler:
            raise NotSupported("Unsupported URL scheme '%s': %s" %
                               (scheme, self._notconfigured[scheme]))
            #ok去吧去 下载吧！！
        return handler.download_request(request, spider)
#下载就不进去了
def download_request(self, request, spider):
        p = urlparse_cached(request)
        scheme = 'https' if request.meta.get('is_secure') else 'http'
        bucket = p.hostname
        path = p.path + '?' + p.query if p.query else p.path
        url = '%s://%s.s3.amazonaws.com%s' % (scheme, bucket, path)
        if self.anon:
            request = request.replace(url=url)
        elif self._signer is not None:
            import botocore.awsrequest
            awsrequest = botocore.awsrequest.AWSRequest(
                method=request.method,
                url='%s://s3.amazonaws.com/%s%s' % (scheme, bucket, path),
                headers=request.headers.to_unicode_dict(),
                data=request.body)
            self._signer.add_auth(awsrequest)
            request = request.replace(
                url=url, headers=awsrequest.headers.items())
        else:
            signed_headers = self.conn.make_request(
                    method=request.method,
                    bucket=bucket,
                    key=unquote(p.path),
                    query_args=unquote(p.query),
                    headers=request.headers,
                    data=request.body)
            request = request.replace(url=url, headers=signed_headers)
        return self._download_http(request, spider)
```

下载这里就结束，回到最上面提到的处理下载返回结果那里，那里调用`self._handle_downloader_output`函数，在这里官方框架的第四五步结束了。以下对返回的结果进行分别对待，进行不同操作。后面官方架构步骤就会都完成。

```python
def _handle_downloader_output(self, response, request, spider):
  #三种结果选一种
        assert isinstance(response, (Request, Response, Failure)), response
        # downloader middleware can return requests (for example, redirects)
      #如果response是Request则再次调用该方法进入调度器，就是说的URL在此进入队列继续。
        if isinstance(response, Request):
            self.crawl(response, spider)
            return
        # response is a Response or Failure
        d = self.scraper.enqueue_scrape(response, request, spider)
        d.addErrback(lambda f: logger.error('Error while enqueuing downloader output',
                                            exc_info=failure_to_exc_info(f),
                                            extra={'spider': spider}))
        return d
```

同理官方注释很清楚。这回进入`enqueue_scrape`函数，列出相关函数，`函数不一定在一个文件里面`

```python
class Slot(object):
    """Scraper slot (one per running spider)"""

    MIN_RESPONSE_SIZE = 1024

    def __init__(self, max_active_size=5000000):
        self.max_active_size = max_active_size
        self.queue = deque()
        self.active = set()
        self.active_size = 0
        self.itemproc_size = 0
        self.closing = None

    def add_response_request(self, response, request):
        deferred = defer.Deferred()
        #加入队列
        self.queue.append((response, request, deferred))
        if isinstance(response, Response):
            self.active_size += max(len(response.body), self.MIN_RESPONSE_SIZE)
        else:
            self.active_size += self.MIN_RESPONSE_SIZE
        return deferred

#该类后面还有

def enqueue_scrape(self, response, request, spider):
        slot = self.slot
        dfd = slot.add_response_request(response, request)
        def finish_scraping(_):
            slot.finish_response(response, request)
            self._check_if_closing(spider, slot)
            self._scrape_next(spider, slot)
            return _
        dfd.addBoth(finish_scraping)
        dfd.addErrback(
            lambda f: logger.error('Scraper bug processing %(request)s',
                                   {'request': request},
                                   exc_info=failure_to_exc_info(f),
                                   extra={'spider': spider}))
        self._scrape_next(spider, slot)
        return dfd
def _scrape_next(self, spider, slot):
  #拿出要处理的队列
        while slot.queue:
            response, request, deferred = slot.next_response_request_deferred()
            self._scrape(response, request, spider).chainDeferred(deferred)

def _scrape(self, response, request, spider):
        """Handle the downloaded response or failure through the spider
        callback/errback"""
        assert isinstance(response, (Response, Failure))
		#进入_scrape2
        dfd = self._scrape2(response, request, spider) # returns spiders processed output
        dfd.addErrback(self.handle_spider_error, request, response, spider)
        #从_scrape2回来后进入`handle_spider_output`
        dfd.addCallback(self.handle_spider_output, request, response, spider)
        return dfd

def _scrape2(self, request_result, request, spider):
        """Handle the different cases of request's result been a Response or a
        Failure"""
    	#进行区分
        if not isinstance(request_result, Failure):
          #如果是Response类型的调用中间件函数`scrape_response`进行处理，这个传进去call_spider函数
            return self.spidermw.scrape_response(
                self.call_spider, request_result, request, spider)
        else:
            # FIXME: don't ignore errors in spider middleware
            dfd = self.call_spider(request_result, request, spider)
            return dfd.addErrback(
                self._log_download_errors, request_result, request, spider)
          
def scrape_response(self, scrape_func, response, request, spider):
        fname = lambda f:'%s.%s' % (
                six.get_method_self(f).__class__.__name__,
                six.get_method_function(f).__name__)

        def process_spider_input(response):
            for method in self.methods['process_spider_input']:
                try:
                    result = method(response=response, spider=spider)
                    assert result is None, \
                            'Middleware %s must returns None or ' \
                            'raise an exception, got %s ' \
                            % (fname(method), type(result))
                except:
                    return scrape_func(Failure(), request, spider)
            return scrape_func(response, request, spider)

        def process_spider_exception(_failure):
            exception = _failure.value
            for method in self.methods['process_spider_exception']:
                result = method(response=response, exception=exception, spider=spider)
                assert result is None or _isiterable(result), \
                    'Middleware %s must returns None, or an iterable object, got %s ' % \
                    (fname(method), type(result))
                if result is not None:
                    return result
            return _failure

        def process_spider_output(result):
            for method in self.methods['process_spider_output']:
                result = method(response=response, result=result, spider=spider)
                assert _isiterable(result), \
                    'Middleware %s must returns an iterable object, got %s ' % \
                    (fname(method), type(result))
            return result
        dfd = mustbe_deferred(process_spider_input, response)
        #差错检测
        dfd.addErrback(process_spider_exception)
        #进行回掉
        dfd.addCallback(process_spider_output)
        return dfd

```

先进入`call_spider`函数

```python
def call_spider(self, result, request, spider):
        result.request = request
        dfd = defer_result(result)
        #自己定义的回调否则就是parse函数（我们项目经常写的）
        dfd.addCallbacks(request.callback or spider.parse, request.errback)
        #下面这个函数里面就一句话return arg_to_iter(result)
        return dfd.addCallback(iterate_spider_output)
```

好的回去进入`handle_spider_output`函数

```python
def handle_spider_output(self, result, request, response, spider):
        if not result:
            return defer_succeed(None)
      #处理错误
        it = iter_errback(result, self.handle_spider_error, request, response, spider)
        #得嘞又 注册了_process_spidermw_output
        dfd = parallel(it, self.concurrent_items,
            self._process_spidermw_output, request, response, spider)
        return dfd
```

进入`_process_spidermw_output`

```python
def _process_spidermw_output(self, output, request, response, spider):
        """Process each Request/Item (given in the output parameter) returned
        from the given spider
        """
   		 #如果是Request，则加入调度器继续
        if isinstance(output, Request):
            self.crawler.engine.crawl(request=output, spider=spider)
        elif isinstance(output, (BaseItem, dict)):
            self.slot.itemproc_size += 1
            #调用Pipeline的process_item，你看看你写的项目是不是有这个
            dfd = self.itemproc.process_item(output, spider)
            dfd.addBoth(self._itemproc_finished, output, response, spider)
            return dfd
        elif output is None:
            pass
        else:
            typename = type(output).__name__
            logger.error('Spider must return Request, BaseItem, dict or None, '
                         'got %(typename)r in %(request)s',
                         {'request': request, 'typename': typename},
                         extra={'spider': spider})
```

官方注释可以啊，后面具体细节就不去看了。

# 0x02.小结

这次在全局搜索代码的时候发现好多一样的代码，可想框架很多地方是可以重写的。我们将官方的图补上

![7](http://myndtt.com/images/58.png)

完成了4，5，6，7，8部分。这回这张图就完美了,引擎将下载结果给`蜘蛛`做`鉴定`，不同则进入不同地方。

scrapy框架看下来很多地方可以重写，但是一环扣一环的，主框架流程不能动。用到了异步，去重等...这么看来其实它也不神秘，但有些小细节我肯定没注意到。说不定错过精彩的地方也不一定，就这样吧..