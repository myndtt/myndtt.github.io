---
title: Openstack之keystone源代码简读二
date: 2019-11-21 17:37:48
tags: [记录,keystone,openstack]
categories:
---

## 一、前言

上次谈到`initialize_admin_application` ，今天来看看。

## 二、APP的启动

`wsgi.py`文件的有两个初始化函数

```python
def initialize_admin_application():
    return initialize_application(name='admin',
                                  config_files=_get_config_files())


def initialize_public_application():
    return initialize_application(name='main',
                                  config_files=_get_config_files())

```

都指向了`initialize_application`，进入得：

<!-- more -->

```python
def initialize_application(name,
                           post_log_configured_function=lambda: None,
                           config_files=None):
    possible_topdir = os.path.normpath(os.path.join(
                                       os.path.abspath(__file__),
                                       os.pardir,
                                       os.pardir,
                                       os.pardir))

    dev_conf = os.path.join(possible_topdir,
                            'etc',
                            'keystone.conf')
    if not config_files:
        config_files = None
        if os.path.exists(dev_conf):
            config_files = [dev_conf]#去找到可能存在的配置文件

    common.configure(config_files=config_files)#初始化配置文件

    # Log the options used when starting if we're in debug mode...
    if CONF.debug:
        CONF.log_opt_values(log.getLogger(CONF.prog), log.DEBUG)

    post_log_configured_function()

    def loadapp():
        return keystone_service.loadapp(
            'config:%s' % find_paste_config(), name)

    _unused, application = common.setup_backends(
        startup_application_fn=loadapp)#回调

    # setup OSprofiler notifier and enable the profiling if that is configured
    # in Keystone configuration file.
    profiler.setup(name)

    return application
```

跟进` common.configure`，其实这代码有点像上篇的那一部分。

```python
def configure(version=None, config_files=None,
              pre_setup_logging_fn=lambda: None):
    keystone.conf.configure()
    sql.initialize()
    keystone.conf.set_config_defaults()

    CONF(project='keystone', version=version,
         default_config_files=config_files)

    pre_setup_logging_fn()
    keystone.conf.setup_logging()

    if CONF.insecure_debug:
        LOG.warning(
            'insecure_debug is enabled so responses may include sensitive '
            'information.')

```

重点看看`common.setup_backends`，跟进得

```python
def setup_backends(load_extra_backends_fn=lambda: {},
                   startup_application_fn=lambda: None):
    drivers = backends.load_backends()
    drivers.update(load_extra_backends_fn())
    res = startup_application_fn()#传进去的回调函数
    return drivers, res
```

进入`backends.load_backends`

```python
def load_backends():

    # Configure and build the cache
    cache.configure_cache()
    cache.configure_cache(region=catalog.COMPUTED_CATALOG_REGION)
    cache.configure_cache(region=assignment.COMPUTED_ASSIGNMENTS_REGION)
    cache.configure_cache(region=revoke.REVOKE_REGION)
    cache.configure_cache(region=token.provider.TOKENS_REGION)
    cache.configure_cache(region=identity.ID_MAPPING_REGION)
    cache.configure_invalidation_region()

    managers = [application_credential.Manager, assignment.Manager,
                catalog.Manager, credential.Manager,
                credential.provider.Manager, resource.DomainConfigManager,
                endpoint_policy.Manager, federation.Manager,
                identity.generator.Manager, identity.MappingManager,
                identity.Manager, identity.ShadowUsersManager,
                limit.Manager, oauth1.Manager, policy.Manager,
                resource.Manager, revoke.Manager, assignment.RoleManager,
                trust.Manager, token.provider.Manager,
                persistence.PersistenceManager]

    drivers = {d._provides_api: d() for d in managers}

    # NOTE(morgan): lock the APIs, these should only ever be instantiated
    # before running keystone.
    provider_api.ProviderAPIs.lock_provider_registry()#drivers字典，里面存放的是一些列manager，每一个都对应keystone目录下对应模块core.py中的Manger对象，这里只负责实例化对象，所以对这些对象加锁。

    auth.core.load_auth_methods()#进入认证,并加载

    return drivers
```

```python
def load_auth_method(method):
    plugin_name = CONF.auth.get(method) or 'default'
    namespace = 'keystone.auth.%s' % method
    try:
        driver_manager = stevedore.DriverManager(namespace, plugin_name,
                                                 invoke_on_load=True)#
        return driver_manager.driver
    except RuntimeError:
        LOG.debug('Failed to load the %s driver (%s) using stevedore, will '
                  'attempt to load using import_object instead.',
                  method, plugin_name)

    driver = importutils.import_object(plugin_name)

    msg = (_(
        'Direct import of auth plugin %(name)r is deprecated as of Liberty in '
        'favor of its entrypoint from %(namespace)r and may be removed in '
        'N.') %
        {'name': plugin_name, 'namespace': namespace})
    versionutils.report_deprecated_feature(LOG, msg)

    return driver
```

主要用到stevedore这里加载插件的方法。具体stevedore用法可以看官方文档，或者[这篇](https://my.oschina.net/hochikong/blog/477665),[这篇](https://blog.csdn.net/gqtcgq/article/details/49620279),至此，该看的差不多看了。对应的认证路由是在`auth`文件夹下的`routers.py`文件下，而对应的认证方法写在`controllers.py`文件夹下。

对应endpoint的理解[可看](https://www.cnblogs.com/eric-nirnava/p/endpoint.html)

## 三、总结

keystone整体流程大致如此。总结可以看着位[前辈的](<https://blog.csdn.net/dylloveyou/article/details/80329732>)，画的图也有助于理解。

![3](https://myndtt.github.io/images/66.png)

OpenStack系统要求的REST风格,这一点很标准。

目前来看，keystone的功能实现，主要涉及额外的公共python库有

1. OSLO oslo_config  oslo_cache

2. stevedore (动态加载代码)

openstack的学习其实就是对python的各类库的学习！
