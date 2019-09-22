---
title: Openstack之Glance源代码简读一
date: 2019-09-22 13:13:59
tags: [记录,glance,openstack]
categories:
---

## 一、前言

学习openstack,对其大体框架已经熟悉，学习其内部基本原理，聊以记录。

作为 OpenStack 的基础支持服务，Keystone 做下面这几件事情：

```php
OpenStack Glance是一种提供发现，注册，和下载的镜像服务。OpenStack Glance是一个提供虚拟机镜像的集中式仓库。通过Glance的RESTful API，可以查询镜像元数据下载镜像。虚拟机的镜像可以很方便的存储在各种地方，从简单的文件系统到对象存储系统(如OpenStack Swift项目)。
```

更多基本概念的介绍可以查看[一文读懂OpenStack Glance是什么 ](<https://blog.csdn.net/dylloveyou/article/details/80530838>)。

## 二、安装

我的机器是ubuntu 18，可以按照[官网](<https://docs.openstack.org/glance/latest/install/install-ubuntu.html>)安装。

<!-- more -->

## 三、分析glance-APi

一般来说，你这样安装`apt install glance`的话，glance会在`/usr/lib/python2.7/dist-packages/glance`下。查看 /usr/bin的关于glance的指令,有不少，目前先看 `glance-api`,与`keystone-admin`差不多，最初一点比较明显的是keystone是由apache启动，而`glance-api`则直接脚本启动：

![1](https://myndtt.github.io/images/62.jpg)

进入cmd目录下的`api.py`，启动服务。

```python
def main():
    try:
        config.parse_args()
        config.set_config_defaults()
        wsgi.set_eventlet_hub()#协程
        logging.setup(CONF, 'glance')
        notifier.set_defaults()

        if CONF.profiler.enabled:
            osprofiler.initializer.init_from_conf(
                conf=CONF,
                context={},
                project="glance",
                service="api",
                host=CONF.bind_host
            )

        server = wsgi.Server(initialize_glance_store=True)
        server.start(config.load_paste_app('glance-api'), default_port=9292)
        server.wait()
    except KNOWN_EXCEPTIONS as e:
        fail(e)


if __name__ == '__main__':
    main()
```

进入`cli.py`文件的`main`函数，关注`server.start(config.load_paste_app('glance-api'), default_port=9292)`

它的功能是根据paste配置文件建立并返回一个WSGI应用程序。glance-api的paste配置文件是源代码中的`/etc/glance-api-paste.ini`，并根据传入的参数`glance-api`来找到对应的配置,然而并没有。

```PYTHON
[composite:rootapp]
paste.composite_factory = glance.api:root_app_factory
/: apiversions
/v1: apiv1app
/v2: apiv2app
```

但找到`composite:rootapp]`request进来后第一个通过的Section，表示需要将一个HTTP URL Request调度到一个或多种Application上。来到对应函数

```PYTHON
def root_app_factory(loader, global_conf, **local_conf):
    if not CONF.enable_v1_api and '/v1' in local_conf:
        del local_conf['/v1']
    if not CONF.enable_v2_api and '/v2' in local_conf:
        del local_conf['/v2']
    return paste.urlmap.urlmap_factory(loader, global_conf, **local_conf)
```

接下来就是进入api文件夹V1,V2目录的`router.py`寻找对应路由。`glance-registry`的启动也大概是这样。不过不同的是glance-registry处理的是与镜像元数据相关的RESTful请求。Glance-api接收到用户的RESTful请求后，如果该请求与元数据相关，则将其转发给glance-registry服务。glance-registry会解析请求的内容，并与数据库进行交互，存取或更新镜像的元数据，这里的元数据是指保存在数据库中的关于镜像的一些信息，Glance的DB模块存储的仅仅是镜像的元数据。

## 四、实践几个问题

### 1. 列举镜像过程

![2](https://myndtt.github.io/images/63.jpg)

`image_repo = self.gateway.get_repo(req.context)`进入

```python
class ImageRepo(object):

    def __init__(self, context, db_api):
        self.context = context
        self.db_api = db_api

    def get(self, image_id):
        try:
            db_api_image = dict(self.db_api.image_get(self.context, image_id))
            if db_api_image['deleted']:
                raise exception.ImageNotFound()
        except (exception.ImageNotFound, exception.Forbidden):
            msg = _("No image found with ID %s") % image_id
            raise exception.ImageNotFound(msg)
        tags = self.db_api.image_tag_get_all(self.context, image_id)
        image = self._format_image_from_db(db_api_image, tags)
        return ImageProxy(image, self.context, self.db_api)

    def list(self, marker=None, limit=None, sort_key=None,
             sort_dir=None, filters=None, member_status='accepted'):
        sort_key = ['created_at'] if not sort_key else sort_key
        sort_dir = ['desc'] if not sort_dir else sort_dir
        db_api_images = self.db_api.image_get_all(
            self.context, filters=filters, marker=marker, limit=limit,
            sort_key=sort_key, sort_dir=sort_dir,
            member_status=member_status, return_tag=True)
        images = []
        for db_api_image in db_api_images:
            db_image = dict(db_api_image)
            image = self._format_image_from_db(db_image, db_image['tags'])
            images.append(image)
        return images
```

从数据库里面查取就好了。取出后还需要层层验证。

```python
 def get_repo(self, context):
        image_repo = glance.db.ImageRepo(context, self.db_api)#取出
        store_image_repo = glance.location.ImageRepoProxy(
            image_repo, context, self.store_api, self.store_utils)
        quota_image_repo = glance.quota.ImageRepoProxy(
            store_image_repo, context, self.db_api, self.store_utils)
        policy_image_repo = policy.ImageRepoProxy(
            quota_image_repo, context, self.policy)
        notifier_image_repo = glance.notifier.ImageRepoProxy(
            policy_image_repo, context, self.notifier)
        if property_utils.is_property_protection_enabled():
            property_rules = property_utils.PropertyRules(self.policy)
            pir = property_protections.ProtectedImageRepoProxy(
                notifier_image_repo, context, property_rules)
            authorized_image_repo = authorization.ImageRepoProxy(
                pir, context)
        else:
            authorized_image_repo = authorization.ImageRepoProxy(
                notifier_image_repo, context)

        return authorized_image_repo
```

每一层的功能简介

| 层                  | 描述                                                         |
| ------------------- | ------------------------------------------------------------ |
| Authorization       | 认证层：提供了一个镜像本身或其属性是否可以改变的验证         |
| Property protection | 属性保护层：该层是可选的，如果你在配置文件中设置了property_protection_file参数，它就变得可用。可以通过配置文件指明访问权限。 |
| Notifier            | 消息通知层：关于镜像变化的消息和使用镜像时发生的错误和警告都会被添加到消息队列中。 |
| Policy              | 规则定义层：定义镜像操作的访问规则，规则在/etc/policy.json文件中定义，该层进行监视并实施。 |
| Quota               | 配额限制层：如果管理者对某用户定义了镜像大小的镜像上传上限，则若该用户上传了超过该限额的镜像，则会上传失败。 |
| Location            | 镜像位置定位层：通过glance_store与后台存储进行交互，例如上传、下载和管理图像位置。1. 添加新位置时检查位置URI是否正确；2. 镜像位置改变时，删除存储后端保存的镜像数据；3. 防止镜像位置重复； |
| Database            | 数据库层：实现与数据库进行交互的API。                        |

### 2.镜像上传过程

1.获取image schema（镜像所支持的属性字典定义）

2.调用glance image create接口创建image，创建完成后image处于queued状态（返回的是image信息）

3.调用glance image data的upload接口，上传image数据；在上传过程中，image处于saving状态；上传完成后，image处于active状态

4.调用glance image get接口，获取image详细信息，并在client中显示

```
def create(self, req, image, extra_properties, tags):
        image_factory = self.gateway.get_image_factory(req.context)
        image_repo = self.gateway.get_repo(req.context)
        try:
            image = image_factory.new_image(extra_properties=extra_properties,
                                            tags=tags, **image)
```

```python
    def new_image(self, image_id=None, name=None, visibility='shared',
                  min_disk=0, min_ram=0, protected=False, owner=None,
                  disk_format=None, container_format=None,
                  extra_properties=None, tags=None, os_hidden=False,
                  **other_args):
        extra_properties = extra_properties or {}
        self._check_readonly(other_args)
        self._check_unexpected(other_args)
        self._check_reserved(extra_properties)

        if image_id is None:
            image_id = str(uuid.uuid4())
        created_at = timeutils.utcnow()
        updated_at = created_at
        status = 'queued'

        return Image(image_id=image_id, name=name, status=status,
                     created_at=created_at, updated_at=updated_at,
                     visibility=visibility, min_disk=min_disk,
                     min_ram=min_ram, protected=protected,
                     owner=owner, disk_format=disk_format,
                     container_format=container_format,
                     os_hidden=os_hidden,
                     extra_properties=extra_properties, tags=tags or [])
```

开辟镜像,标记镜像id,时间等等，并且检查权限之类的。随后进行add

```python
  def add(self, image):
        image_values = self._format_image_to_db(image)
        if (image_values['size'] is not None
           and image_values['size'] > CONF.image_size_cap):
            raise exception.ImageSizeLimitExceeded
        # the updated_at value is not set in the _format_image_to_db
        # function since it is specific to image create
        image_values['updated_at'] = image.updated_at
        new_values = self.db_api.image_create(self.context, image_values)#进入
        self.db_api.image_tag_set_all(self.context,
                                      image.image_id, image.tags)
        image.created_at = new_values['created_at']
        image.updated_at = new_values['updated_at']
```

每一层的add都有被重写，实现对应层的目的。包括镜像的元数据入库等等。进入upload函数

![3](https://myndtt.github.io/images/64.jpg)

修改状态并保存。

```python
    image_repo.save(image, from_state='queued')#信息入库
    image.set_data(data, size)#上传至对应目录(配置文件提供)
```

## 四、总结

Glance组件的启动离不开Glance domian 控制器的良好封装。整张图配合文中的表，最好理解了。

![5](https://myndtt.github.io/images/65.png)



几个库的重要性。

1.Eventlet (绿色协程)

2.webob (WebOb是一个Python库，它围绕WSGI请求环境提供包装器，还有一个对象来帮助创建WSGI响应。 对象映射了HTTP的指定行为，包括头解析，内容协商以及条件和范围请求的正确处理。)

openstack的学习其实就是对python的各类库的学习！