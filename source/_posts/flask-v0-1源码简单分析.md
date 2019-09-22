---
title: flask v0.1源码简单分析
date: 2018-01-22 13:50:17
tags: [python,flask,记录]
categories:
---

## 0x00.前言

想学习python的flask，看了《FlaskWeb开发：基于Python的Web应用开发实战》这本书，觉得很OK👌。之前彭师兄分析过flask v0.1的源码，听说代码不长，加之网上解释甚多，所以自己也想看看，就其本身，本文无意义，有错误更是可能，只是记录而已。

## 0x01.获取

第一步获取`git clone https://github.com/pallets/flask.git`,这是最新版本的

第二步 `git tag`看版本

第三步`git reset --hard xx`回退到对应版本

最后是这样

![54](http://myndtt.github.io/images/54.png)

主要内容在flask.py文件里面

<!-- more -->

## 0x02.简要分析

### 1.包

flask.py导入的包情况

```python
from __future__ import with_statement
import os
import sys

from threading import local
from jinja2 import Environment, PackageLoader, FileSystemLoader
from werkzeug import Request as RequestBase, Response as ResponseBase, \
     LocalStack, LocalProxy, create_environ, cached_property, \
     SharedDataMiddleware
from werkzeug.routing import Map, Rule
from werkzeug.exceptions import HTTPException, InternalServerError
from werkzeug.contrib.securecookie import SecureCookie

# utilities we import from Werkzeug and Jinja2 that are unused
# in the module but are exported as public interface.
from werkzeug import abort, redirect
from jinja2 import Markup, escape
```

Werkzeug是[Python](https://baike.baidu.com/item/Python)的[WSGI](https://baike.baidu.com/item/WSGI)规范的实用函数库，而对于WSGI,廖先生解释的很好啊https://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/001386832689740b04430a98f614b6da89da2157ea3efe2000

盗一张简书的图

![55](https://upload-images.jianshu.io/upload_images/1563354-26e27df8f32e9d55.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/545)

而jinja2是flask其默认的页面模版，其格式有点像最初的将java代码写在jsp页面中...

### 2.上下文

flask.py最下面是这几行，要看懂flask需要了解其上下文机制

```python
_request_ctx_stack = LocalStack()#建立request栈
current_app = LocalProxy(lambda: _request_ctx_stack.top.app)#栈当前app
request = LocalProxy(lambda: _request_ctx_stack.top.request)#栈当前request
session = LocalProxy(lambda: _request_ctx_stack.top.session)#栈当前session
g = LocalProxy(lambda: _request_ctx_stack.top.g)#当栈前g
```

书中有提到

![55](http://myndtt.github.io/images/55.png)

对于上下文的解释更有https://blog.tonyseek.com/post/the-context-mechanism-of-flask/

解释的很好。

### 3.简单流程

下面一大串其实主要为了解释上门那张图

#### 一.`app=Flask(__name__)`

```python
#程序入口 建立app实例
class Flask(object):
```

并且其中初始化一些东西

```python
self.template_context_processors = [_default_template_ctx_processor]

        self.url_map = Map()#URL endpoint function 对应的map

        if self.static_path is not None:
            self.url_map.add(Rule(self.static_path + '/<filename>',
                                  build_only=True, endpoint='static'))
            if pkg_resources is not None:
                target = (self.package_name, 'static')
            else:
                target = os.path.join(self.root_path, 'static')
            self.wsgi_app = SharedDataMiddleware(self.wsgi_app, {#静态
                self.static_path: target
            })

        #: the Jinja2 environment.  It is created from the
        #: :attr:`jinja_options` and the loader that is returned
        #: by the :meth:`create_jinja_loader` function.
        self.jinja_env = Environment(loader=self.create_jinja_loader(),
                                     **self.jinja_options)
        self.jinja_env.globals.update(
            url_for=url_for,
            get_flashed_messages=get_flashed_messages
        )
```

这里还是很关键的

```python
Flask应用使用werkzeug库中的Map类和Rule类来处理URL的模式匹配，每一个URL模式对应一个Rule实例，这些Rule实例最终会作为参数传递给Map类构造包含所有URL模式的一个“地图”。
Flask使用SharedDataMiddleware来对静态内容的访问支持，也即是static目录下的资源可以被外部，
```

flask的route设计思路，这篇足以https://segmentfault.com/a/1190000004213652

#### 二.`app.run()`

开始运行flask类下的`run()`函数,run函数最后一句

```python
return run_simple(host, port, self, **options)#开始运行,这里的run_simple是werkzeug里面的
```

在对应·的`/lib/python版本/site-packes/werkzeug/serving.py`文件中可以找到该函数,该函数属于

`class ForkingWSGIServer(ForkingMixIn, BaseWSGIServer):`类

```python
def run_simple(hostname, port, application, use_reloader=False,
               use_debugger=False, use_evalex=True,
               extra_files=None, reloader_interval=1,
               reloader_type='auto', threaded=False,
               processes=1, request_handler=None, static_files=None,
               passthrough_errors=False, ssl_context=None):
    """Start a WSGI application. Optional features include a reloader,
    multithreading and fork support.
```

该函数下同时执行`inner()`函数,

```python
    _log('info', ' * Running on %s://%s:%d/ %s',
             ssl_context is None and 'http' or 'https',
             display_hostname, port, quit_msg)

    def inner():
        try:
            fd = int(os.environ['WERKZEUG_SERVER_FD'])
        except (LookupError, ValueError):
            fd = None
        srv = make_server(hostname, port, application, threaded,
                          processes, request_handler,
                          passthrough_errors, ssl_context,
                          fd=fd)
        if fd is None:
            log_startup(srv.socket)
        srv.serve_forever()
```

这里执行了`make_server`,make_server()函数默认生成一个WSGIServer类,同时最后一句执行`serve_forever()`函数，该函数属于`class BaseWSGIServer(HTTPServer, object):`类，跟进

```python
    def serve_forever(self):
        self.shutdown_signal = False
        try:
            HTTPServer.serve_forever(self)
        except KeyboardInterrupt:
            pass
        finally:
            self.server_close()
```

执行`HTTPServer.serve_forever(self)` 进入对应`socketserver.py`文件跟进

#### 三.进入socketserver.py

-----------------------------------------------------------------------------------------------------------------------------------------------------------

```python
def serve_forever(self, poll_interval=0.5):
        """Handle one request at a time until shutdown.

        Polls for shutdown every poll_interval seconds. Ignores
        self.timeout. If you need to do periodic tasks, do them in
        another thread.
        """
        self.__is_shut_down.clear()
        try:
            while not self.__shutdown_request:
                # XXX: Consider using another file descriptor or
                # connecting to the socket to wake this up instead of
                # polling. Polling reduces our responsiveness to a
                # shutdown request and wastes cpu at all other times.
                r, w, e = _eintr_retry(select.select, [self], [], [],
                                       poll_interval)
                if self in r:
                    self._handle_request_noblock()

                self.service_actions()
        finally:
            self.__shutdown_request = False
            self.__is_shut_down.set()
```

其中` if self in r:self._handle_request_noblock()`如果请求成功进入`_handle_request_noblock()`函数

跟进得

```python

    def _handle_request_noblock(self):
        """Handle one request, without blocking.

        I assume that select.select has returned that the socket is
        readable before this function was called, so there should be
        no risk of blocking in get_request().
        """
        try:
            request, client_address = self.get_request()
        except socket.error:
            return
        if self.verify_request(request, client_address):
            try:
                self.process_request(request, client_address)
            except:
                self.handle_error(request, client_address)
                self.shutdown_request(request)
```

这里调用`get_request()`函数得到请求，这个函数会pass,同时后面调用了`process_request()`函数处理请求，跟进得

```python
def process_request(self, request, client_address):
        """Call finish_request.

        Overridden by ForkingMixIn and ThreadingMixIn.

        """
        self.finish_request(request, client_address)
        self.shutdown_request(request)
```

先调用`finish_request()`函数

```python
def finish_request(self, request, client_address):
        """Finish one request by instantiating RequestHandlerClass."""
        self.RequestHandlerClass(request, client_address, self)
```

```python
class WSGIRequestHandler(BaseHTTPRequestHandler)://werkzeug的serving.py

class BaseHTTPRequestHandler(socketserver.StreamRequestHandler)://

class StreamRequestHandler(BaseRequestHandler): ／／socketserver.py


finish_request方法中执行了self.RequestHandlerClass(request, client_address, self)
self.RequestHandlerClass = RequestHandlerClass（就在__init__方法中）。所以finish_request方法本质上就是创建了一个服务处理实例,当我们创建服务处理类实例时，就会运行handle()方法，而handle()方法则一般是我们处理事务逻辑的代码块。
```

#### 四.在此回到werkzeug的serving.py

进入werkzeug的handle函数得

```python
def handle(self):
        """Handles a request ignoring dropped connections."""
        rv = None
        try:
            rv = BaseHTTPRequestHandler.handle(self)
        except (socket.error, socket.timeout) as e:
            self.connection_dropped(e)
        except Exception:
            if self.server.ssl_context is None or not is_ssl_error():
                raise
        if self.server.shutdown_signal:
            self.initiate_shutdown()
        return rv
```

调用了`BaseHTTPRequestHandler.handle(self)`,进入得

```python
 def handle(self):
        """Handle multiple requests if necessary."""
        self.close_connection = 1

        self.handle_one_request()
        while not self.close_connection:
            self.handle_one_request()
```

调用了`handle_one_request()`该方法重写跳一步回到werkzeug的该函数中

```python
def handle_one_request(self):
        """Handle a single HTTP request."""
        self.raw_requestline = self.rfile.readline()
        if not self.raw_requestline:
            self.close_connection = 1
        elif self.parse_request():
            return self.run_wsgi()
```

调用了`run_wsgi()`函数，该函数内部进行写和开始回复请求等函数，同时还有一个`execute()`函数

```python
def execute(app):
            application_iter = app(environ, start_response)
            try:
                for data in application_iter:
                    write(data)
                if not headers_sent:
                    write(b'')
            finally:
                if hasattr(application_iter, 'close'):
                    application_iter.close()
                application_iter = None
```

execute函数中通过application_iter = app(environ, start_response)调用了Flask应用实例app,最后实际执行为`__call__`函数，最后`__call__`函数上,该函数最后一句，调用wsgi_app函数

```python
return self.wsgi_app(environ, start_response)
```

`wsgi_app`函数中有

```python
    #将environ参数放入request_context中开始创建请求上下文函数
        #上诉函数实际即为压栈操作
        with self.request_context(environ):
            rv = self.preprocess_request()#预处理请求
            if rv is None:
                rv = self.dispatch_request()#分发处理请求，得到对应编写函数的结果
            response = self.make_response(rv)#将结果指定到制造返回的函数中
            response = self.process_response(response)
            return response(environ, start_response)
```

其中在dispatch_request()中这句尤为关键,其函数中有：

```python
return self.view_functions[endpoint](**values)#通过endpoint找到对应的函数然后就是运行
```

## 0x03.总结

看代码还是舒服😌，但是不排除有错误...

![ok](http://myndtt.github.io/images/56.png)