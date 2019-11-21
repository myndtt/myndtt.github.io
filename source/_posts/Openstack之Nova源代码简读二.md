---
title: Openstack之Nova源代码简读二
date: 2019-11-21 20:14:01
tags: [记录,nova,openstack]
categories:
---



## 一、前言

学习openstack,对其大体框架已经熟悉，学习其内部基本原理，聊以记录。更简单清楚的nova各组件协调图

![](https://myndtt.github.io/images/69.jpg)

```c
1.客户（可以是OpenStack最终用户，也可以是其他程序）向API（nova-api）发送请求：“帮我创建一个虚机”；
2.API对请求做一些必要处理后，向Messaging（RabbitMQ）发送了一条消息：“让Scheduler创建一个虚机”；
3.Scheduler（nova-scheduler）从Messaging获取到API发给它的消息，然后执行调度算法，从若干计算节点中选出节点 A ；
4.Scheduler向Messaging发送了一条消息：“在计算节点 A 上创建这个虚机”
5.计算节点 A 的Compute（nova-compute）从Messaging中获取到Scheduler发给它的消息，然后在本节点的Hypervisor上启动虚机；
6.在虚机创建的过程中，Compute如果需要查询或更新数据库信息，会通过Messaging向Conductor（nova-conductor）发送消息，Conductor负责数据库访问。
```

<!-- more -->

## 二、寻找RabbitMQ 

openstack中nova的messaging机制通常是由RabbitMQ实现。RabbitMQ 是基于 AMQP协议实现的一个消息队列实现 RPC，那么什么是RPC呢。RPC（Remote Procedure Call）—[远程过程调用](https://baike.baidu.com/item/%E8%BF%9C%E7%A8%8B%E8%BF%87%E7%A8%8B%E8%B0%83%E7%94%A8/7854346)，它是一种通过[网络](https://baike.baidu.com/item/%E7%BD%91%E7%BB%9C/143243)从远程计算机程序上请求服务，而不需要了解底层网络技术的协议。

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
```

上一篇是看到这里。没有继续讨论，现在跟进这个函数。让我们先看看`compute_task_api`是如何定义的。

`self.compute_task_api = conductor.ComputeTaskAPI()`由此可知compute_task_api为`conductor`的一个computetaskapi的类，用于排列计算任务。

```PYTHON
class ComputeTaskAPI(object):
    """ComputeTask API that queues up compute tasks for nova-conductor."""

    def __init__(self):
        self.conductor_compute_rpcapi = rpcapi.ComputeTaskAPI()#跟进
```

```PYTHON
        super(ComputeTaskAPI, self).__init__()
        target = messaging.Target(topic=RPC_TOPIC,
                                  namespace='compute_task',
                                  version='1.0')#好的，看到messaging了 
        serializer = objects_base.NovaObjectSerializer()
        self.client = rpc.get_client(target, serializer=serializer)
```

```python
    def build_instances(self, context, instances, image, filter_properties,
            admin_password, injected_files, requested_networks,
            security_groups, block_device_mapping, legacy_bdm=True,
            request_spec=None, host_lists=None):
         #调用ComputeTaskManager类的build_instances函数
        self.conductor_compute_rpcapi.build_instances(context,
                instances=instances, image=image,
                filter_properties=filter_properties,
                admin_password=admin_password, injected_files=injected_files,
                requested_networks=requested_networks,
                security_groups=security_groups,
                block_device_mapping=block_device_mapping,
                legacy_bdm=legacy_bdm, request_spec=request_spec,
                host_lists=host_lists)
```

```python
   cctxt = self.client.prepare(version=version)
   cctxt.cast(context, 'build_instances', **kwargs)#调用cast方法，发送请求不用等待响应
```

用户想要启动一个实例时， 
1、nova-api作为消息生产者，将“启动实例”的消息包装成amqp消息以rpc.call的方式通过topic交换机放入消息队列； 
2、nova-compute作为消息消费者，接受该消息并通过底层虚拟化软件执行相应操作； 
3、虚拟机启动成功后，nova-compute作为消息生产者将“实例启动成功”的消息通过direct交换机放入相应的响应队列； 

4、nova-api作为消息消费者接受该消息并通知用户。

## 三、总结

利用的python库，除了之前提到的。这里主要是`oslo_messaging`对rabbitmq等ampq的封装。剩下的就是`nova-compute`中LibVirt对各类虚拟产品的管理。目前常见的企业级虚拟化产品有`VMware,HyperV,Xen,KVm`

LibVirt的设计目标是通过相同的方式管理不同的虚拟化引擎，比如KVM，Xen,HyperV,VMware ESX等。但是目前Libvirt多数情况被用到的还是KVM(毕竟一家子)。其他有自己的管理工具。

![]((https://myndtt.github.io/images/70.jpg)

![](G(https://myndtt.github.io/images/71.png)









