---
title: "Spring Cloud Alibaba原生中间件"
date: 2025-06-14T16:23:39+08:00
draft: false
description: "Spring Cloud Alibaba中间件的介绍"
tags: ["Java", Spring Cloud]
---

[Spring官方介绍](https://spring.io/projects/spring-cloud-alibaba#overview) [⚠️Spring官方对Spring Cloud Alibaba的更新不及时]

[Spring Cloud Alibaba官网](https://sca.aliyun.com/)

## Nacos（服务注册与发现）

> Nacos(Dynamic **Na**ming and **Co**nfiguration **S**ervice,  Nacos)，一个易于构建 Al Agent 应用的动态服务发现、配置管理和AI智能体管理平台。
>
> Nacos = Eureka + Config + Bus = Spring Cloud Consul

[Nacos官网](https://nacos.io/)

| 组件名    | 语言 | CAP  | 服务健康检查 | 对外暴露接口 | Spring Cloud集成 |
| --------- | ---- | ---- | ------------ | ------------ | ---------------- |
| Eureka    | Java | AP   | 可配支持     | HTTP         | 已集成           |
| Consul    | Go   | CP   | 支持         | HTTP/DNS     | 已集成           |
| Zookeeper | Java | CP   | 支持         | 客户端       | 已集成           |
| Nacos     | Java | AP   | 支持         | 客户端       | 已集成           |

Nacos 在阿里巴巴内部有超过 10 万的实例运行，已经过了类似双十一等各种大型流量的考验，Nacos默认是AP模式，

但也可以调整切换为CP，一般用默认AP即可。

[下载与安装](https://nacos.io/docs/latest/quickstart/quick-start/?spm=5238cd80.2ef5001f.0.0.3f613b7ct98a8H&source=blog)

- spring-cloud-starter-alibaba-nacos-config: 实现配置的动态更新
- spring-cloud-starter-alibaba-nacos-discovery: 实现服务的注册与发现

### 注册中心

主要配合spring-cloud-starter-alibaba-nacos-discovery

服务发现：yml配置nacos相关配置、主启动配置EnableDiscoveryClient注解

调用注册中心中的服务：只需在配置中增加RestTemplate的bean，并增加loadbalancer注解，调用服务的接口时将host写成服务名称即可

### 配置中心

主要配合spring-cloud-starter-alibaba-nacos-discovery

nacos端配置文件DataId的命名规则:

${prefix}-${spring.profiles.active}.${file-extension}

### Namespace，Group，DataId

目的是解决 多环境、多项目、多配置文件的管理

Nacos 数据模型 Key 由三元组（Namespace，Group，DataId）唯一确定, Namespace默认是空串，公共命名空间（public），分组默认是 DEFAULT_GROUP。

## Sentinel（服务熔断和降级）

> 面向分布式、多语言异构化服务架构的流量治理组件。
>
> 从流量路由、流量控制、流量整形、熔断降级、系统自适应过载保护、热点流量防护等多个维度来帮助开发者保障微服务的稳定性

[Sentinel官网](https://sentinelguard.io/)/[Github](https://github.com/alibaba/Sentinel)

Sentinel后台端口8719，前台端口8080，前台用户名密码都是sentinel

### SentinelResource注解

SentinelResource是一个流量防卫组件注解，用于指定防护资源，对配置的资源进行流量控制、熔断降级等功能。

[SentinelResource注解配置说明](https://sentinelguard.io/zh-cn/docs/annotation-support.html)

注意fallback与blockHandler的区别

- blockHandler，主要针对sentinel配置后出现的违规情况处理
- fallback，程序异常了，JVM抛出的异常服务降级

### 流控规则

[流控规则说明](https://sentinelguard.io/zh-cn/docs/flow-control.html)

> Sentinel能够对流量进行控制，主要是监控应用的QPS流量或者并发线程数等指标，如果达到指定的阈值时，就会被流量进行控制，以避免服务被瞬时的高并发流量击垮，保证服务的高可靠性。

流控模式

- 直接：默认的流控模式，当接口达到限流条件时，直接开启限流功能。
- 关联：当关联的资源达到阈值，就限流自己；当与A关联的资源B达到阈值后，就限流A自己。(A惹事B挂了)
- 链路： 来自不同链路的请求对同一个目标访问时，实施针对性的不同限流措施，比如C请求来访问就限流，D请求来访问就是OK

流控效果

- 快速失败：直接失败，抛出异常。Blocked by Sentinel (flow limiting)
- Warm up：限流，冷启动。当流量增大的时候，希望系统从空闲状态到繁忙状态的切换时间长一些。公式：阈值除以冷却因子coldFactor（默认值为3），经过预热时长后才会达到阈值。默认coldFactor为3，即请求QPS从threshold/3开始，经预热时长逐渐升至设定的QPS阈值
- 排队等待：主要处理间隔性突发流量，当一秒内发起大量请求，后几秒却是空闲状态。排队等待重排请求，使得服务器匀速处理请求。

### 熔断规则

> Sentinel 熔断降级会在调用链路中某个资源出现不稳定状态时（例如调用超时或异常比例升高），对这个资源的调用进行限制，让请求快速失败，避免影响到其它的资源而导致级联错误。当资源被降级后，在接下来的降级时间窗口之内，对该资源的调用都自动熔断（默认行为是抛出 DegradeException）。

[熔断规则说明](https://sentinelguard.io/zh-cn/docs/circuit-breaking.html)

- 慢调用比例(SLOW_REQUEST_RATIO) ：在统计时长内，实际请求数目 > 设定的最小请求数 且 实际慢调用比例 > 比例阈值，进入熔断状态。
- 异常比例(ERROR_RATION): 异常比例达到阈值，进入熔断状态
- 异常数(ERROR_COUNT): 数量达到阈值，进入熔断状态

### 热点规则

> 热点即经常访问的数据，很多时候我们希望统计或者限制某个热点数据中访问频次最高的TopN数据，并对其访问进修限流或者其他操作

[热点规则说明](https://sentinelguard.io/zh-cn/docs/parameter-flow-control.html)

简单来说就是对指定入参限流，sentinel还可以有参数例外项，即对特定的参数值不限流，其他的参数值都限流

### 授权规则

> 对请求分黑白名单

[来源访问控制说明](https://sentinelguard.io/zh-cn/docs/origin-authority-control.html)

⚠️注意要实现RequestOriginParser再配合可视化端

## Seata（分布式事务）

⚠️面试重灾区，Seata分布式三大模式的原理需要仔细了解

背景：一次业务操作需要跨多个数据源或需要跨多个系统进行远程调用，就会产生分布式事务问题；但是关系型数据库提供的能力是单机事务，一但遇到分布式事务场景，就需要通过更多其他技术手段来解决问题。

> Seata(Simple Extensible Autonomous Transaction Architecture), 是一款开源的分布式事务解决方案，致力于在微服务架构下提供高性能和简单易用的分布式事务服务。

[Seata官网](https://seata.apache.org/)/[Github](https://github.com/apache/incubator-seata), Alibaba开发，2023.10.29捐赠给Apache

[Seata部署](https://seata.apache.org/zh-cn/docs/ops/deploy-guide-beginner)

### Seata三大模式

- [Seata AT](https://seata.apache.org/zh-cn/docs/dev/mode/at-mode)
- [Seata TCC](https://seata.apache.org/zh-cn/docs/dev/mode/tcc-mode)
- [Seata SAGA](https://seata.apache.org/zh-cn/docs/dev/mode/saga-mode)
- [Seata XA](https://seata.apache.org/zh-cn/docs/dev/mode/xa-mode)

### 工作流程

Seata对分布式事务的协调和控制就是1+3

- 1个XID：XID是全局事务的唯一标识，他可以在服务的调用链路中传递，绑定到服务的事务上下文中。
- 3（TC->TM->RM）
  - TC(Transaction Coordinator)事务协调者
    - 维护全局和分支事务的状态，驱动全局事务提交或回滚。
  - TM(Transaction Manager)事务管理器
    - 定义全局事务的范围：开始全局事务、提交或回滚全局事务。
  - RM(Resource Manager)资源管理器
    - 管理分支事务处理的资源，与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚。

三个组件相互协作，TC以Seata服务器形式独立部署，TM和RM则是以Seata Client的形式集成在微服务中运行

流程：

1. TM向TC申请开启一个全局事务，全局事务创建成功并生成一个全局唯一的XID。
2. XID在微服务调用链路的上下文中传播。
3. RM向TC注册分支事务，将其纳入XID对应全局事务的管辖。
4. TM向TC发起针对XID的全局提交或回滚决议。
5. TC调度XID下管辖的全部分支事务完成提交或回滚请求。
