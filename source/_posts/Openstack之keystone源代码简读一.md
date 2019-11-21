---
title: Openstack之keystone源代码简读一
date: 2019-11-21 17:38:18
tags: [记录,keystone,openstack]
categories:
---

## 一、前言

学习openstack,对其大体框架已经熟悉，学习其内部基本原理，聊以记录。

作为 OpenStack 的基础支持服务，Keystone 做下面这几件事情：

```php
1、管理用户及其权限
    不同用户拥有不同权限。这里的权限不仅仅指的是用户对于某项资源的权限，也包括对某项用户的权限。
2、维护 OpenStack Services 的 Endpoint
Endpoint 是一个网络上可访问的地址，通常是一个 URL。Service 通过 Endpoint 暴露自己的 API，Keystone 负责管理和维护每个 Service 的 Endpoint。（关键）
3.Authentication（认证）和 Authorization（鉴权）
```

更多基本概念的介绍可以查看[OpenStack keystone详解及调优](https://blog.csdn.net/zhongbeida_xue/article/details/78654676)和[理解 Keystone 核心概念 - 每天5分钟玩转 OpenStack(18)](https://mp.weixin.qq.com/s?__biz=MzIwMTM5MjUwMg==&mid=2653587894&idx=1&sn=877a9cb2c23f242e45e7c113f2d2440f&chksm=8d3081afba4708b9295debc08ac8159927733a1429a5f488ce136370984be15e6039b3496b46&scene=21#wechat_redirect)，[推荐看](https://www.cnblogs.com/linhaifeng/p/6248086.html)

<!-- more -->

## 二、安装

我的机器是ubuntu 18，可以按照[官网](https://docs.openstack.org/keystone/latest/install/keystone-install-ubuntu.html#install-and-configure-components)安装,或者你不喜欢看简单的英文话，可以参考[这篇](https://www.cnblogs.com/jsonhc/p/7692418.html)或者[这篇](https://www.cnblogs.com/linhaifeng/p/6269707.html),都讲的很详细，当然因为版本不同，大家不要一股脑就直接复制，粘贴安装。会有问题。

## 三、分析openstack-admin

一般来说，你这样安装`apt install keystone`的话，keystone会在`/usr/lib/python2.7/dist-packages/keystone`下。查看 `/usr/bin`的关于keystone的指令,有`keystone-manage,keystone-wsgi-admin,keystone-wsgi-public`. 先查看`keystone-manage`,我们来到`cmd`目录下的`manage.py`文件。

```python
# entry point.
def main():
    dev_conf = os.path.join(possible_topdir,
                            'etc',
                            'keystone.conf')#/usr/lib/python2.7/dist-packages/etc/keystone.conf

    config_files = None
    if os.path.exists(dev_conf):
        config_files = [dev_conf]

    cli.main(argv=sys.argv, config_files=config_files)
```

进入`cli.py`文件的`main`函数

```python
CMDS = [
    BootStrap,
    CredentialMigrate,
    CredentialRotate,
    CredentialSetup,
    DbSync,
    DbVersion,
    Doctor,
    DomainConfigUpload,
    FernetRotate,
    FernetSetup,
    MappingPopulate,
    MappingPurge,
    MappingEngineTester,
    SamlIdentityProviderMetadata,
    TokenFlush,
]


def add_command_parsers(subparsers):
    for cmd in CMDS:
        cmd.add_argument_parser(subparsers)

command_opt = cfg.SubCommandOpt('command',
                                title='Commands',
                                help='Available commands',
                                handler=add_command_parsers)
def main(argv=None, config_files=None):
    CONF.register_cli_opt(command_opt)#着手注册各类操作

    keystone.conf.configure()#正式配置各类模块
    sql.initialize()#数据库group的注册操作依靠oslo_db
    keystone.conf.set_default_for_default_log_levels() #日志等级

    CONF(args=argv[1:],
         project='keystone',
         version=pbr.version.VersionInfo('keystone').version_string(),
         usage='%(prog)s [' + '|'.join([cmd.name for cmd in CMDS]) + ']',
         default_config_files=config_files)
    if not CONF.default_config_files:#此时为['/etc/keystone/keystone.conf']
        LOG.warning('Config file not found, using default configs.')
    keystone.conf.setup_logging()#开始记录
    #keystone.cmd.cli.对应的操作，此次与oslo的cfg cache组件相关(oslo_config.cfg.ConfigOpts)
    CONF.command.cmd_class.main()
```

关注几个问题。

### 1. default_config_files如何获得的？

```PYTHON
    CONF(args=argv[1:],
         project='keystone',
         version=pbr.version.VersionInfo('keystone').version_string(),
         usage='%(prog)s [' + '|'.join([cmd.name for cmd in CMDS]) + ']',
         default_config_files=config_files)
```

此处调用了`CONF`的`__call__`函数

```PYTHON
    def __call__(self,
                 args=None,
                 project=None,
                 prog=None,
                 version=None,
                 usage=None,
                 default_config_files=None,
                 default_config_dirs=None,
                 validate_default_values=False,
                 description=None,
                 epilog=None,
       
        self.clear()

        self._validate_default_values = validate_default_values

        prog, default_config_files, default_config_dirs = self._pre_setup(
            project, prog, version, usage, description, epilog,
            default_config_files, default_config_dirs)#该方法去获取对应的配置文件路径

        self._setup(project, prog, version, usage, default_config_files,
                    default_config_dirs, use_env)

        self._namespace = self._parse_cli_opts(args if args is not None
                                               else sys.argv[1:])
        if self._namespace._files_not_found:
            raise ConfigFilesNotFoundError(self._namespace._files_not_found)
        if self._namespace._files_permission_denied:
            raise ConfigFilesPermissionDeniedError(
                self._namespace._files_permission_denied)

        self._load_alternative_sources()

        self._check_required_opts()
```

尝试可能的路径

```python
   def _pre_setup(self, project, prog, version, usage, description, epilog,
                   default_config_files, default_config_dirs):
        """Initialize a ConfigCliParser object for option parsing."""

        if prog is None:
            prog = os.path.basename(sys.argv[0])
            if prog.endswith(".py"):
                prog = prog[:-3]

        if default_config_files is None:
            default_config_files = find_config_files(project, prog)

        if default_config_dirs is None:
            default_config_dirs = find_config_dirs(project, prog)

        self._oparser = _CachedArgumentParser(
            prog=prog, usage=usage, description=description, epilog=epilog)

        if version is not None:
            self._oparser.add_parser_argument(self._oparser,
                                              '--version',
                                              action='version',
                                              version=version)

        return prog, default_config_files, default_config_dirs
```

### 2.操作如何注册？

关注`CONF.command.cmd_class.main()`, 该函数会调用对应的操作模块，在此之前会执行模块内的`CONF = keystone.conf.CONF`，其返回的是`oslo_config.cfg.ConfigOpts`类的实例。(模块初始化时服务把默认的配置项及配置组注册至cfg中)

```PYTHON
    def register_cli_opt(self, opt, group=None):
        """Register a CLI option schema.

        CLI option schemas must be registered before the command line and
        config files are parsed. This is to ensure that all CLI options are
        shown in --help and option validation works as expected.

        :param opt: an instance of an Opt sub-class
        :param group: an optional OptGroup object or group name
        :return: False if the opt was already registered, True otherwise
        :raises: DuplicateOptError, ArgsAlreadyParsedError
        """
        if self._args is not None:
            raise ArgsAlreadyParsedError("cannot register CLI option")

        return self.register_opt(opt, group, cli=True, clear_cache=False)
    #......
    def register_opt(self, opt, group=None, cli=False):
        """Register an option schema.

        Registering an option schema makes any option value which is previously
        or subsequently parsed from the command line or config files available
        as an attribute of this object.

        :param opt: an instance of an Opt sub-class#具体实例
        :param group: an optional OptGroup object or group name#配置文件的片段
        :param cli: whether this is a CLI option #标明是否为可从命令行接受的option。
        :return: False if the opt was already registered, True otherwise
        :raises: DuplicateOptError
        """
        if group is not None:
            group = self._get_group(group, autocreate=True)
            if cli:
                self._add_cli_opt(opt, group)
            self._track_deprecated_opts(opt, group=group)
            return group._register_opt(opt, cli)

        # NOTE(gcb) We can't use some names which are same with attributes of
        # Opts in default group. They includes project, prog, version, usage,
        # default_config_files and default_config_dirs.
        if group is None:
            if opt.name in self.disallow_names:
                raise ValueError('Name %s was reserved for oslo.config.'
                                 % opt.name)

        if cli:
            self._add_cli_opt(opt, None)

        if _is_opt_registered(self._opts, opt):
            return False

        self._opts[opt.dest] = {'opt': opt, 'cli': cli}
        self._track_deprecated_opts(opt)
        return True

```

则command_opt,也即各类操作注册完成。(Group的注册类似)。

### 3.配置文件中的Frenet是什么

keystone有好几种认证方式：UUID、PKI(PKIZ)、Fernet;Frent是其中一种而已，具体这三者认证之间的区别，[这篇](https://www.cnblogs.com/dhplxf/p/7966890.html])和[这篇](https://www.jianshu.com/p/e1b206ebda5b)已经讲的很好了，不赘述了。

### 4.几条命令

回顾一下，安装时候用到的几条命令。

```c
keystone-manage db_sync   #数据库同步，初始化，生成对应的orm关系映射
#一下两条是生成认证时对应的key
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

重点看看这条

```pythoN
keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne

```

这一条就可以理解为创建管理员并引导对应的身份认证服务，不指定用户名默认为admin,在bootstrap选项里，并没有domain的设定项，所以默认的domain只能是Default。没有设定角色默认创建三个角色admin，member，reader。

### 5.其他

`keystone-wsgi-admin`

`keystone-wsgi-public`

这两个实际上是测试使用，当然现在版本的keystone是以apache可以启动，`/etc/apache2/sites-enabled`下的`keystone.conf`文件。

`  WSGIScriptAlias / /usr/bin/keystone-wsgi-public` 已经使用WSGIScriptAlias指令来指定wsgi application的启动脚本。

以 keystone-wsgi-admin为例子

```python
#! /usr/bin/python2
#PBR Generated from u'wsgi_scripts'

import threading

from keystone.server.wsgi import initialize_admin_application

if __name__ == "__main__":
    import argparse
    import socket
    import sys
    import wsgiref.simple_server as wss

    my_ip = socket.gethostbyname(socket.gethostname())

    parser = argparse.ArgumentParser(
        description=initialize_admin_application.__doc__,
        formatter_class=argparse.ArgumentDefaultsHelpFormatter,
        usage='%(prog)s [-h] [--port PORT] [--host IP] -- [passed options]')
    parser.add_argument('--port', '-p', type=int, default=8000,
                        help='TCP port to listen on')
    parser.add_argument('--host', '-b', default='',
                        help='IP to bind the server to')
    parser.add_argument('args',
                        nargs=argparse.REMAINDER,
                        metavar='-- [passed options]',
                        help="'--' is the separator of the arguments used "
                        "to start the WSGI server and the arguments passed "
                        "to the WSGI application.")
    args = parser.parse_args()
    if args.args:
        if args.args[0] == '--':
            args.args.pop(0)
        else:
            parser.error("unrecognized arguments: %s" % ' '.join(args.args))
    sys.argv[1:] = args.args
    server = wss.make_server(args.host, args.port, initialize_admin_application())#关键！！！！

    print("*" * 80)
    print("STARTING test server keystone.server.wsgi.initialize_admin_application")
    url = "http://%s:%d/" % (server.server_name, server.server_port)
    print("Available at %s" % url)
    print("DANGER! For testing only, do not use in production")
    print("*" * 80)
    sys.stdout.flush()

    server.serve_forever()
else:
    application = None
    app_lock = threading.Lock()

    with app_lock:
        if application is None:
            application = initialize_admin_application()


```

关键在于`initialize_admin_application` 和`wsgiref.simple_server `，下次再说吧！！

## 四、总结

keystone组件的启动离不开oslo这个openstack公开库。主要跟oslo.config有很大关系。不仅如此，似乎其他组件的服务启动也是如此。该库的基本使用可以[参考](https://blog.csdn.net/canxinghen/article/details/51711457),详细的可以[参考](https://blog.csdn.net/Bill_Xiang_/column/info/18110)


