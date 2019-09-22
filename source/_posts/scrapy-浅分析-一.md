---
title: scrapy 浅分析(一)
date: 2018-01-30 15:13:40
tags: [scrapy,python]
categories:
---

# 0x00.前言

相信大家都用过python的scrapy框架写给爬虫。然后就想看看这个框架的简单执行流程。

# 0x01.大体架构

官方架构图如下

![architecture](http://myndtt.github.io/images/58.png)

图大体架构很清晰，实际以例子来看看，一般我们的爬虫以如下代码开始运行(命令行是一样的)

```python
from scrapy import cmdline

cmdline.execute("scrapy crawl xxx".split())
```

<!-- more -->

![57](http://myndtt.github.io/images/57.png)

找到scrapy安装的包文件，上述代码即为运行`cmdline.py`文件里面的`execute()`函数，该函数较长，阶段看

如下：

```python
def execute(argv=None, settings=None):
    if argv is None:
        argv = sys.argv

    # --- backwards compatibility for scrapy.conf.settings singleton ---
    if settings is None and 'scrapy.conf' in sys.modules:
        from scrapy import conf
        if hasattr(conf, 'settings'):
            settings = conf.settings
    # ------------------------------------------------------------------

```

这里得到外部参数，然后即为注释所说为兼容性操作,此处继续为`cmdline.py`文件。

```python
   if settings is None:
        settings = get_project_settings()
        # set EDITOR from environment if available
        try:
            editor = os.environ['EDITOR']
        except KeyError: pass
        else:
            settings['EDITOR'] = editor
    check_deprecated_settings(settings)

    # --- backwards compatibility for scrapy.conf.settings singleton ---
    import warnings
    from scrapy.exceptions import ScrapyDeprecationWarning
    with warnings.catch_warnings():
        warnings.simplefilter("ignore", ScrapyDeprecationWarning)
        from scrapy import conf
        conf.settings = settings
    # ------------------------------------------------------------------
```

执行`get_project_settings()`函数，全局寻找跟进得，此处为`utils`文件夹下的`project.py`文件：

```python
def get_project_settings():
  #ENVVAR = 'SCRAPY_SETTINGS_MODULE',变量所在文件中已经定义了
    if ENVVAR not in os.environ:
        project = os.environ.get('SCRAPY_PROJECT', 'default')
        #project此时为default，下面进行初始化
        init_env(project)
```

`Init_env()`函数如下：

```python
def init_env(project='default', set_syspath=True):
    """Initialize environment to use command-line tool from inside a project
    dir. This sets the Scrapy settings module and modifies the Python path to
    be able to locate the project module.
    """
    cfg = get_config()
    if cfg.has_option('settings', project):
        os.environ['SCRAPY_SETTINGS_MODULE'] = cfg.get('settings', project)
    closest = closest_scrapy_cfg()
    if closest:
        projdir = os.path.dirname(closest)
        if set_syspath and projdir not in sys.path:
            sys.path.append(projdir)
```

如注释所说，初始化环境,循环递归找到用户项目中的配置文件`settings.py`,并且将其设置到环境变量`SCRAPY_SETTINGS_MODULE`中。然后继续`get_project_settings`函数：

```Python
 #创建settings实例，根据包导入情况，from scrapy.settings import Settings，进入settings文件夹可发 #现此过程同时加载默认配置文件default_settings.pys
  	settings = Settings()
    #得到用户配置文件路径
    settings_module_path = os.environ.get(ENVVAR)
    #更新配置，有则覆盖
    if settings_module_path:
        settings.setmodule(settings_module_path, priority='project')
	#以下都为一些更新操作，setdict函数里面是一些set_update函数
    # XXX: remove this hack
    pickled_settings = os.environ.get("SCRAPY_PICKLED_SETTINGS_TO_OVERRIDE")
    if pickled_settings:
        settings.setdict(pickle.loads(pickled_settings), priority='project')

    # XXX: deprecate and remove this functionality
    env_overrides = {k[7:]: v for k, v in os.environ.items() if
                     k.startswith('SCRAPY_')}
    if env_overrides:
        settings.setdict(env_overrides, priority='project')

    return settings
```

至此，`get_project_settings()`该函数结束，如函数名字一样，最后返回项目配置。回到最初`cmdline.py`文件

```python
 	#容差兼容性检测
  	check_deprecated_settings(settings)
	#兼容
    # --- backwards compatibility for scrapy.conf.settings singleton ---
    import warnings
    from scrapy.exceptions import ScrapyDeprecationWarning
    with warnings.catch_warnings():
        warnings.simplefilter("ignore", ScrapyDeprecationWarning)
        from scrapy import conf
        conf.settings = settings
    # ------------------------------------------------------------------
```

此处执行`check_deprecated_settings`函数

```python
def check_deprecated_settings(settings):
    deprecated = [x for x in DEPRECATED_SETTINGS if settings[x[0]] is not None]
    if deprecated:
        msg = "You are using the following settings which are deprecated or obsolete"
        msg += " (ask scrapy-users@googlegroups.com for alternatives):"
        msg = msg + "\n    " + "\n    ".join("%s: %s" % x for x in deprecated)
        warnings.warn(msg, ScrapyDeprecationWarning)
```

做一些容差性检测。接下来`cmdline.py`函数`execute()`代码如下

```python
	#是否环境在项目中	
 	inproject = inside_project()
    #做字典
    cmds = _get_commands_dict(settings, inproject)
    
```

`inside_project`函数如下：

```python
def inside_project():
    scrapy_module = os.environ.get('SCRAPY_SETTINGS_MODULE')
    if scrapy_module is not None:
        try:
            import_module(scrapy_module)
        except ImportError as exc:
            warnings.warn("Cannot import scrapy settings module %s: %s" % (scrapy_module, exc))
        else:
            return True
    return bool(closest_scrapy_cfg())
```

检测是否有`SCRAPY_SETTINGS_MODULE`，这是前面有的啊，如何当真没有则执行`closest_scrapy_cfg()`,该函数如下：

```python
def closest_scrapy_cfg(path='.', prevpath=None):
    """Return the path to the closest scrapy.cfg file by traversing the current
    directory and its parents
    """
    if path == prevpath:
        return ''
    path = os.path.abspath(path)
    cfgfile = os.path.join(path, 'scrapy.cfg')
    if os.path.exists(cfgfile):
        return cfgfile
    return closest_scrapy_cfg(os.path.dirname(path), path)
```

循环查找`scrapy.cfg`,如何有就返回，同时上面代码`bool`返回真。如函数名字所示，该`inside_project`函数检测环境是否于项目中。随后执行` cmds = _get_commands_dict(settings, inproject)`,涉及如下：

```python
def _iter_command_classes(module_name):
    # TODO: add `name` attribute to commands and and merge this function with
    # scrapy.utils.spider.iter_spider_classes
    for module in walk_modules(module_name):
        for obj in vars(module).values():
            if inspect.isclass(obj) and \
                    issubclass(obj, ScrapyCommand) and \
                    obj.__module__ == module.__name__ and \
                    not obj == ScrapyCommand:
                yield obj

def _get_commands_from_module(module, inproject):
    d = {}
    #将module也即commands文件夹文件带入
    for cmd in _iter_command_classes(module):
        if inproject or not cmd.requires_project:
            cmdname = cmd.__module__.split('.')[-1]
            d[cmdname] = cmd()
    return d

def _get_commands_from_entry_points(inproject, group='scrapy.commands'):
    cmds = {}
    for entry_point in pkg_resources.iter_entry_points(group):
        obj = entry_point.load()
        if inspect.isclass(obj):
            cmds[entry_point.name] = obj()
        else:
            raise Exception("Invalid entry point %s" % entry_point.name)
    return cmds

def _get_commands_dict(settings, inproject):
   #将commands文件夹带入该函数中
    cmds = _get_commands_from_module('scrapy.commands', inproject)
    cmds.update(_get_commands_from_entry_points(inproject))
    cmds_module = settings['COMMANDS_MODULE']
    if cmds_module:
        cmds.update(_get_commands_from_module(cmds_module, inproject))
    return cmds
  	#最后返回一个字典
```

将该函数返回结果进行打印，如下：

```
{'bench': <scrapy.commands.bench.Command object at 0x180d597f98>, 'check': <scrapy.commands.check.Command object at 0x180d5975c0>, 'crawl': <scrapy.commands.crawl.Command object at 0x180d57ef98>, 'edit': <scrapy.commands.edit.Command object at 0x180d57ee48>, 'fetch': <scrapy.commands.fetch.Command object at 0x180d57eb70>, 'genspider': <scrapy.commands.genspider.Command object at 0x180d57eb38>, 'list': <scrapy.commands.list.Command object at 0x180d57edd8>, 'parse': <scrapy.commands.parse.Command object at 0x180d5b1080>, 'runspider': <scrapy.commands.runspider.Command object at 0x180d5832e8>, 'settings': <scrapy.commands.settings.Command object at 0x180d5833c8>, 'shell': <scrapy.commands.shell.Command object at 0x180d5b54a8>, 'startproject': <scrapy.commands.startproject.Command object at 0x180d5b5438>, 'version': <scrapy.commands.version.Command object at 0x180d5b5470>, 'view': <scrapy.commands.view.Command object at 0x180d5b55c0>}
```

这些均为commands文件夹下文件，可以发现这些个文件几乎都存在run函数，都是可以执行的命令，有没有发现一个特别的`crawl`呢。继续

```python
		#得到你输入的命令
  	cmdname = _pop_command_name(argv)
    parser = optparse.OptionParser(formatter=optparse.TitledHelpFormatter(), \
        conflict_handler='resolve')
    if not cmdname:
        _print_commands(settings, inproject)
        sys.exit(0)
      #字典中寻找，找到对应的命令
    elif cmdname not in cmds:
        _print_unknown_command(settings, cmdname, inproject)
        sys.exit(2)
	#找到了，还有一些处理
    cmd = cmds[cmdname]
    parser.usage = "scrapy %s %s" % (cmdname, cmd.syntax())
    parser.description = cmd.long_desc()
    settings.setdict(cmd.default_settings, priority='command')
    cmd.settings = settings
    cmd.add_options(parser)
    opts, args = parser.parse_args(args=argv[1:])
    _run_print_help(parser, cmd.process_options, args, opts)
	#创建CrawlerProcess实例
    cmd.crawler_process = CrawlerProcess(settings)
    #将对应的命令执行，这里是crawl
    _run_print_help(parser, _run_command, cmd, args, opts)
    sys.exit(cmd.exitcode)
```

```python
def _run_command(cmd, args, opts):
    if opts.profile:
        _run_command_profiled(cmd, args, opts)
    else:
        cmd.run(args, opts)
```

# 0x02.小结

这里一路下来就是，得到配置文件，照着输入的命令于commands文件夹中寻找对应命令并执行。这里还没有得到太多细节，下次再聊。