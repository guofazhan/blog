title: Eureka之REST API
author: Mr.G
tags:
  - 微服务
  - Eureka
categories:
  - 微服务
date: 2018-08-23 17:50:00
---
> Eureka服务端与客户之间的通讯方式为http，服务端通过提供rest API风格的接口与客户端或其它服务端交互。

#### 交互数据格式
> JSON/XML

#### 提供资源服务的包路径
> **com.netflix.eureka.resources**
#### 提供可使用的REST API接口
> **/{version}/apps 路径**

![image](https://github.com/guofazhan/image/blob/master/apps.png?raw=true)

> **/{version}/instances 路径**

![image](https://github.com/guofazhan/image/blob/master/rs.png?raw=true)

> **/{version}/status 路径**

![image](https://github.com/guofazhan/image/blob/master/status.png?raw=true)

> **/{version}/peerreplication 路径**

![image](https://github.com/guofazhan/image/blob/master/pree.png?raw=true)

> **/{version}/vips 路径**

![image](https://github.com/guofazhan/image/blob/master/vip.png?raw=true)

> **/{version}/svips 路径**

![image](https://github.com/guofazhan/image/blob/master/svip.png?raw=true)