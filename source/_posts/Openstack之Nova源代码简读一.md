---
title: Openstack之Nova源代码简读一
date: 2019-11-21 19:42:01
tags: [记录,nova,openstack]
categories:
---

## 一、前言

学习openstack,对其大体框架已经熟悉，学习其内部基本原理，聊以记录。

Compute Service Nova 是 OpenStack 最核心的服务，负责维护和管理云环境的计算资源。OpenStack 作为 IaaS 的云操作系统，虚拟机生命周期管理也就是通过 Nova 来实现的。openstack中其他的组件都可以看做为Nova服务。Nova组件有以下六部分组成： 

```python
1 API服务器 API Server（Nova-api） 
2 计算工作者Compute Workers（Nova-compute） 
3 网络控制器Network Controller（Nova-network） 
4 卷工作者Volume Worker（Nova-volume） 
5 调度器Schedule（Nova-schedule） 
6 消息队列Message Queue（rabbitmq server）
```

具体概念可查看[这篇](<https://blog.csdn.net/ohenry88/article/details/75267742>)，或者[这篇](<https://mp.weixin.qq.com/s?__biz=MzIwMTM5MjUwMg==&mid=2653587855&idx=1&sn=06cc0f77ed94ec69b983d27a0e657b13&chksm=8d308196ba4708809892f84dc6367cade3864b1ba7fec89d792a8baefe3c00fb80b74557f720&scene=21#wechat_redirect>)

需要注意的是Openstack Nova 仅仅是作为云计算虚拟机的管理工具，其本身并不提供任何的虚拟化技术，而是交由具体的 Hypervisor 来实现虚拟机的创建和管理。

<!-- more -->

## 二、安装

官网[安装](<https://docs.openstack.org/nova/latest/install/controller-install-ubuntu.html#prerequisites>)

## 三、nova-api

nova-api 是整个 Nova 组件的门户，所有对 Nova 的请求都首先由 nova-api 处理。nova-api 向外界暴露若干 HTTP REST API 接口 。具体能做的事情，包括下图。

![4](https://myndtt.github.io/images/67.jpg)



有点多。

```python
def main():
    config.parse_args(sys.argv)
    logging.setup(CONF, "nova")
    utils.monkey_patch()
    objects.register_all()
    gmr_opts.set_defaults(CONF)
    if 'osapi_compute' in CONF.enabled_apis:
        # NOTE(mriedem): This is needed for caching the nova-compute service
        # version.
        objects.Service.enable_min_version_cache()
    log = logging.getLogger(__name__)

    gmr.TextGuruMeditation.setup_autorun(version, conf=CONF)

    launcher = service.process_launcher()
    started = 0
    for api in CONF.enabled_apis:#启动各类服务
        should_use_ssl = api in CONF.enabled_ssl_apis
        try:
            server = service.WSGIService(api, use_ssl=should_use_ssl)
            launcher.launch_service(server, workers=server.workers or 1)
            started += 1
        except exception.PasteAppNotFound as ex:
            log.warning("%s. ``enabled_apis`` includes bad values. "
                        "Fix to remove this warning.", ex)
```

具体服务在`CONF/SERVICE.PY`中定义

```python
    cfg.ListOpt('enabled_apis',
                item_type=cfg.types.String(choices=['osapi_compute',
                                                    'metadata']),
                default=['osapi_compute', 'metadata'],
                help="List of APIs to be enabled by default."),
    cfg.ListOpt('enabled_ssl_apis',
                default=[],
                help="""
```

像之前的glance中的`glance-api`启动一样。找到nova的配置文件

```python
[composite:osapi_compute]
use = call:nova.api.openstack.urlmap:urlmap_factory
/: oscomputeversions
# v21 is an exactly feature match for v2, except it has more stringent
# input validation on the wsgi surface (prevents fuzzing early on the
# API). It also provides new features via API microversions which are
# opt into for clients. Unaware clients will receive the same frozen
# v2 API feature set, but with some relaxed validation
/v2: openstack_compute_api_v21_legacy_v2_compatible
/v2.1: openstack_compute_api_v21
 #。。。。。
[app:osapi_compute_app_v21]
paste.app_factory = nova.api.openstack.compute:APIRouterV21.factory#跟进
```

根据这些配置，由 paste.deploy 库生成 app，然后通过 eventlet 这个python 的多线程网络库创建线程池，去运行这些 app，响应 API 请求。 跟进`nova.api.openstack.compute:APIRouterV21.factory`，寻找到路由。

```python

```

以创建实例为例

```python
    def create(self, req, body):
        """Creates a new server for a given user."""
        context = req.environ['nova.context']
        server_dict = body['server']
        password = self._get_server_admin_password(server_dict)
        name = common.normalize_name(server_dict['name'])
        description = name
        if api_version_request.is_supported(req, min_version='2.19'):
            description = server_dict.get('description')

        # Arguments to be passed to instance create function
        create_kwargs = {}#以上获取post的各类参数，计划创建。

        # TODO(alex_xu): This is for back-compatible with stevedore
        # extension interface. But the final goal is that merging
        # all of extended code into ServersController.
        self._create_by_func_list(server_dict, create_kwargs, body)#跟进
```

```python
        server_create_func_list = [
        block_device_mapping.server_create,
        block_device_mapping_v1.server_create,
        config_drive.server_create,
        keypairs.server_create,
        multiple_create.server_create,
        scheduler_hints.server_create,
        security_groups.server_create,
        user_data.server_create,
    ]
    #。。。。。
    def _create_by_func_list(self, server_dict,
                             create_kwargs, req_body):
        for func in self.server_create_func_list:
            func(server_dict, create_kwargs, req_body)#配置各类变量
```

回到`create`函数

```python
        try:
            inst_type = flavors.get_flavor_by_flavor_id(
                    flavor_id, ctxt=context, read_deleted="no")

            supports_multiattach = common.supports_multiattach_volume(req)
            (instances, resv_id) = self.compute_api.create(context,
                            inst_type,
                            image_uuid,
                            display_name=name,
                            display_description=description,
                            availability_zone=availability_zone,
                            forced_host=host, forced_node=node,
                            metadata=server_dict.get('metadata', {}),
                            admin_password=password,
                            requested_networks=requested_networks,
                            check_server_group_quota=True,
                            supports_multiattach=supports_multiattach,
                            **create_kwargs)#关键，跟进
```

通过`flavor_id`获取对应传递实例信息，进而创建实例。

```python
 return self._create_instance(
                       context, instance_type,
                       image_href, kernel_id, ramdisk_id,
                       min_count, max_count,
                       display_name, display_description,
                       key_name, key_data, security_groups,
                       availability_zone, user_data, metadata,
                       injected_files, admin_password,
                       access_ip_v4, access_ip_v6,
                       requested_networks, config_drive,
                       block_device_mapping, auto_disk_config,
                       filter_properties=filter_properties,
                       legacy_bdm=legacy_bdm,
                       shutdown_terminate=shutdown_terminate,
                       check_server_group_quota=check_server_group_quota,
                       tags=tags, supports_multiattach=supports_multiattach)#跟进

```

```python
# Normalize and setup some parameters
        if reservation_id is None:
            reservation_id = utils.generate_uid('r')
        security_groups = security_groups or ['default']
        min_count = min_count or 1
        max_count = max_count or min_count
        block_device_mapping = block_device_mapping or []
        tags = tags or []

        if image_href:
            image_id, boot_meta = self._get_image(context, image_href)#获取镜像信息
        else:
            image_id = None
            boot_meta = self._get_bdm_image_metadata(
                context, block_device_mapping, legacy_bdm)#如果用户没有给出image相关的参数，则去查找镜像相关元信息

        self._check_auto_disk_config(image=boot_meta,
                                     auto_disk_config=auto_disk_config)

```

这部分代码主要是设置一些变量，如果用户没有设定，则系统自动生成一个或使用缺省值，security_groups表示网络安全组。继续跟进。

```python
block_device_mapping = self._check_and_transform_bdm(context,
            base_options, instance_type, boot_meta, min_count, max_count,
            block_device_mapping, legacy_bdm)#进行调度

        # We can't do this check earlier because we need bdms from all sources
        # to have been merged in order to get the root bdm.
        self._checks_for_create_and_rebuild(context, image_id, boot_meta,
                instance_type, metadata, injected_files,
                block_device_mapping.root_bdm())

        instance_group = self._get_requested_instance_group(context,
                                   filter_properties)

        tags = self._create_tag_list_obj(context, tags)

        instances_to_build = self._provision_instances(
            context, instance_type, min_count, max_count, base_options,
            boot_meta, security_groups, block_device_mapping,
            shutdown_terminate, instance_group, check_server_group_quota,
            filter_properties, key_pair, tags, supports_multiattach)#进行调度

```

```python
for rs, build_request, im in instances_to_build:
    build_requests.append(build_request)
    instance = build_request.get_new_instance(context)
    instances.append(instance)
    request_specs.append(rs)

```

```python
self.compute_task_api.build_instances(context,
        instances=instances, image=boot_meta,
        filter_properties=filter_properties,
        admin_password=admin_password,
        injected_files=injected_files,
        requested_networks=requested_networks,
        security_groups=security_groups,
        block_device_mapping=block_device_mapping,
        legacy_bdm=False)
else:
    self.compute_task_api.schedule_and_build_instances(
        context,
        build_requests=build_requests,
        request_spec=request_specs,
        image=boot_meta,
        admin_password=admin_password,
        injected_files=injected_files,
        requested_networks=requested_networks,
        block_device_mapping=block_device_mapping,
        tags=tags)

return instances, reservation_id

```

## 四、总结

nova工作原理图如下所示

![nova](https://myndtt.github.io/images/68.jpg)

但是目前，我们只是简单略读了整个过程，并且似乎与从传统的nova组件过程相冲突，因为没有提到queue。nova更多的细节还没看到。
