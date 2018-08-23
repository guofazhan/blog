title: Eureka之服务注册数据模型
author: Mr.G
tags:
  - 微服务
  - Eureka
categories:
  - 微服务
date: 2018-08-23 17:43:00
---
> Eureka 作为注册中心，担任着服务节点的注册以及发现，并提供着动态的管理（坏点摘除，服务检测等）。可以说是整个系统的基础设施。

- 注册列表的数据模型

![image](https://github.com/guofazhan/image/blob/master/eureka%E6%B3%A8%E5%86%8C%E5%88%97%E8%A1%A8%E6%95%B0%E6%8D%AE%E6%A8%A1%E5%9E%8B.png?raw=true)

**1. Applications**
> 注册列表，包含注册到当前注册中心上的所有实例信息，每个注册中心在内存中保持一份数据。在Applications中持有Application的队列

**2. Application**
> 注册到注册中心的服务集群信息，Application可以看做为一组对外提供相同服务的集群列表。在Application包含一组提供相同服务的InstanceInfo集合

**3. InstanceInfo**
> 注册列表里的单个服务实例信息，代表系统中提供一项服务的某个节点，里面包含节点的基础信息，以及唯一标识instanceId。

**4. DataCenterInfo**
> 服务实例所在的数据中心，默认为MyOwn，为InstanceInfo子属性

**5. LeaseInfo**
> 服务实例的租约信息，用于对服务做健康检查，续租，过期等服务可用性的管理。为InstanceInfo子属性

---
通过数据模型图可以清楚的看到Eureka将我们系统中的服务信息抽象成5个对象进行精细化的管理。我们注册进去的服务信息都将实例化成上述对象存储在eureka的服务端内存中，从而对外提供服务的注册以及发现。