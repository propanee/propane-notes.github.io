# Overview

项目地址：https://github.com/zhoushuguang/beyond

b站：https://www.bilibili.com/video/BV1op4y177iS/?spm_id_from=333.788&vd_source=62e671c84986495aa734adb02dcc32eb

- 微服务架构，从单体架构到微服务架构，如何拆分
- 高并发，模拟高并发场景，面对高并发时的性能优化以及架构调整

## 架构

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/0930%20gozero%E5%BE%AE%E6%9C%8D%E5%8A%A1%E6%9E%B6%E6%9E%84%E5%9B%BE.png" alt="架构图" style="zoom:25%;" />

## 核心功能点

- [zero-examples](https://github.com/zeromicro/zero-examples) 所有功能全部在项目中体现
- 服务注册与发现，负载均衡
- 监控 && 告警 && 链路追踪
- 缓存异常的处理，穿透、击穿、雪崩、缓存数据库一致性保证
- 秒杀场景，超高并发的读写（10w QPS，压测）
- 请求耗时高如何优化
- 聊天室
- 问题排查定位方法
- CI/CD 完整部署上线，并可访问

## 项目结构

```Shell
├── README.md
├── application(大仓，存放所有的微服务)
│   ├── applet
│   │   ├── README.md
│   │   ├── applet.api
│   │   ├── applet.go //入口文件，存放main函数
│   │   ├── etc // 配置文件
│   │   └── internal // 不希望对外提供访问
│   ├── article
│   │   ├── README.md
│   │   └── rpc
│   ├── chat
│   │   ├── README.md
│   │   └── rpc
│   ├── concerned
│   │   ├── README.md
│   │   └── rpc
│   ├── member
│   │   ├── README.md
│   │   └── rpc
│   ├── message
│   │   ├── README.md
│   │   └── rpc
│   ├── qa
│   │   ├── README.md
│   │   └── rpc
│   └── user
│       ├── README.md
│       └── rpc
├── db
│   └── user.sql
├── go.mod
├── go.sum
└── pkg(各种微服务依赖的一些通用方法和工具)
    ├── encrypt
    │   ├── encrypt.go
    │   └── encrypt_test.go
    ├── jwt
    │   └── token.go
    └── util
        └── util.go
```

# 服务拆分

[Go实战干货二 微服务拆分&&项目结构 && 服务初始化 && 调用流程 && jwt验证 && 验证码注册 && 缓存 && 服务注册与发现](https://pwmzlkcu3p.feishu.cn/docx/XX6xdpB0UoH0auxgPYlcxmninDb)

## 微服务

微服务提倡将单体应划分成一组小的服务，服务之间互相协调、互相配合，为用户提供最终价值。每个服务运行在其独立的进程中，服务与服务间采用轻量级的通信机制互相沟通（通常是基于RPC或者RESTful API）。每个服务都围绕着具体业务进行构建，并且能够独立地部署到生产环境

微服务相当于把服务组件化了，通过组装不同的微服务组件快速实现业务功能

## DDD

### 限界上下文

DDD（Domain-Driven Design领域驱动设计） 在战略设计上提出了“限界上下文”这个概念，用来确定语义所在的领域边界。

**理论上限界上下文就是微服务的边界**。我们将限界上下文内的领域模型映射到微服务，就完成了从问题域到软件的解决方案。可以说，限界上下文是微服务设计和拆分的主要依据。在领域模型中，如果不考虑技术异构、团队沟通等其它外部因素，一个限界上下文理论上就可以设计为一个微服务。定义好了限界上下文，微服务的边界也就定义好了。

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/0930%E9%99%90%E7%95%8C%E4%B8%8A%E4%B8%8B%E6%96%87.jpeg" alt="a8fc2d7b9bb24ab1a0ddda8eec3c1865" style="zoom:33%;" />

### 基于DDD的分层

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/0930DDD%E6%9E%B6%E6%9E%84.jpeg" alt="output" style="zoom:50%;" />

## 垂直功能

按照所在的职能部门，独立业务功能进行划分，比如评论服务，用户服务，购物车服务等就可以独立的划分为一个微服务：**文章(article)**、**问答(qa)**、**消息中心(message)**、**聊天(chat)**、**关注(concerned)**独、**会员(member)**、**用户中心(user)**独立为一个微服务

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/0930%E5%9E%82%E7%9B%B4%E6%9C%8D%E5%8A%A1%E5%88%92%E5%88%86.png" alt="image-20230930214138731" style="zoom: 33%;" />

### 微服务架构

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/image-20230930214626422.png" alt="image-20230930214626422" style="zoom: 50%;" />

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/image-20230930214651665.png" alt="image-20230930214651665" style="zoom: 50%;" />

# 项目初始化

##  生成api服务

定义applet.api文件，通过goctl命令生成项目

```Shell
cd ~/beyond/application/applet
goctl api go --dir=./ --api applet.api
```

## 生成rpc项目

```Shell
cd ~/beyond/application/user/rpc
goctl rpc protoc ./user.proto --go_out=. --go-grpc_out=. --zrpc_out=./
```

## 启动user-rpc服务

采用etcd作为注册中心

```Shell
## 查看etcd中是否已注册
etcdctl get --prefix user.rpc

user.rpc/7587873030137136398
192.168.2.170:8080
```

```Shell
## 查看租约
etcdctl lease list
```

```Shell
## 查看租约剩余时间
etcdctl lease timetolive 694d8a5192174527
```

## 服务注册与发现

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1008%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C%E4%B8%8E%E5%8F%91%E7%8E%B0.png" alt="image-20231008213521808" style="zoom:33%;" />

- 服务提供方（user rpc）将其ip注册到注册中心；

- 注册中心部署多个副本（多个ip）；

- 负载均衡处理客户端请求的具体ip。go-zero中放在客户端，在ip调用时调用负载均衡，从注册中心拿出一批ip，通过算法返回一个给调用方，调用方通过直连调用对应的server。

  
