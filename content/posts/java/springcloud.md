---
title: "Spring Cloud原生中间件"
date: 2025-06-09T13:57:23+08:00
draft: false
description: "Spring Cloud中间件的介绍"
tags: ["Java", "Spring Cloud"]
---

## Consul（服务注册与发现 + 分布式配置管理）

> 拥有服务治理功能，实现微服务之间的动态注册与发现
>
> ❌不在使用Eureka：1. 停更进维 2. 注册中心独立且和微服务功能解耦

[Consul官网](https://developer.hashicorp.com/consul)

[Spring官方介绍](http://spring.io/projects/spring-cloud-consul)

### 三个注册中心区别

| 组件名    | 语言 | CAP  | 服务健康检查 | 对外暴露接口 | Spring Cloud集成 |
| --------- | ---- | ---- | ------------ | ------------ | ---------------- |
| Eureka    | Java | AP   | 可配支持     | HTTP         | 已集成           |
| Consul    | Go   | CP   | 支持         | HTTP/DNS     | 已集成           |
| Zookeeper | Java | CP   | 支持         | 客户端       | 已集成           |
| Nacos     | Java | AP   | 支持         | 客户端       | 已集成           |

#### CAP

> 一个分布式系统最多只能同时满足其中的两个属性。

- Consistency: 强一致性，每次读取都能获取到最近一次成功写入的数据。
- Availablity: 可用性，每次请求都会在有限时间内返回结果，无论结果是否为最新。
- Partition tolerance: 分区容错性，系统在遇到网络分区（节点之间无法通信）时仍能继续运作。（必须有）

经典CAP：

- AP（Eurake）：当网络分区出现后，为了保证可用性，系统B可以返回旧值，保证系统的可用性。

  ![CAP_AP](/images/20250612214547.png)

- CP：当网络分区出现后，为了保证一致性，系统返回错误信息。

  ![CAP_CP](/images/20250612214611.png)

当数据出现不一致时，虽然A, B上的注册信息不完全相同，但每个Eureka节点依然能够正常对外提供服务，这会出现查询服务信息时如果请求A查不到，但请求B就能查到。如此保证了可用性但牺牲了一致性结论：违背了一致性C的要求，只满足可用性和分区容错，即AP

当网络分区出现后，为了保证一致性，**就必须拒接请求**，否则无法保证一致性，Consul 遵循CAP原理中的CP原则，保证了强一致性和分区容错性，且使用的是Raft算法，比zookeeper使用的Paxos算法更加简单。虽然保证了强一致性，但是可用性就相应下降了，例如服务注册的时间会稍长一些，因为 Consul 的 raft 协议要求必须过半数的节点都写入成功才认为注册成功 ；在leader挂掉了之后，重新选举出leader之前会导致Consul 服务不可用。结论：违背了可用性A的要求，只满足一致性和分区容错，即CP

### 使用

```bash
./consul --version # 查看版本号
./consul agent -dev # 开发模式启动 http://localhost:8500

```

## LoadBalance（服务调用复杂均衡）

> 负载均衡：平摊请求，减少服务器压力
>
> Spring-cloud-starter-loadbalancer：spring官方提供的客户端负载均衡器，在SpringCloud-commons中，用来替代以前的Ribbon，支持RestTemplate、Web Flux。
>
> 客户端与服务器端负载均衡区别：服务器端如Nginx，将客户端发起的请求到通过服务器端部署的Nginx，转发到各个服务器上。LoadBlance本地负载均衡，调用微服务接口时，在注册中心上获取信息服务列表之后缓存到JVM本地，从而在本地实现RPC远程服务调用。

[Spring官方介绍](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html)

### 负载均衡算法

实现ReactiveLoadBalancer接口，默认使用的是RoundRobinLoadBalancer。

[官方对负载均衡算法的介绍](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html#switching-between-the-load-balancing-algorithms)

- RoundRobinLoadBalancer: 轮询
- RandomLoadBalancer: 随机

## OpenFeign（服务调用复杂均衡）

> Feign是一个声明式web服务客户端，用来替代RestTemplate，只需创建一个Rest接口并在该接口上添加注解@FeignClient。
>
> OpenFeign基本上就是当前微服务之间调用的事实标准。
>
> 可以结合LoadBalancer实现负载均衡，结合Sentinel实现熔断降级。

[Spring官方介绍](https://docs.spring.io/spring-cloud-openfeign/reference/spring-cloud-openfeign.html) [Github](https://github.com/spring-cloud/spring-cloud-openfeign)

### 超时控制

[超时控制说明](https://docs.spring.io/spring-cloud-openfeign/reference/spring-cloud-openfeign.html#timeout-handling)

- connectTimeout: 连接超时时间
- readTimeout: 请求处理超时时间，默认超时时间是60s

### 重试机制

[重试机制说明](https://docs.spring.io/spring-cloud-openfeign/reference/spring-cloud-openfeign.html#spring-cloud-feign-overriding-defaults)

默认情况下会创建Retry.NEVER_RETRY类型的Retry的bean，这将禁用重试，这种重试行为与Feign默认行为不同，他会自动重试IOExceptions，将它们视为网络相关的瞬态异常，以及从ErrorDecoder抛出的任何RetryableException。

### 默认HTTPClient修改

[默认HTTPClient修改说明](https://docs.spring.io/spring-cloud-openfeign/reference/spring-cloud-openfeign.html#spring-cloud-feign-overriding-defaults)

如果不做特殊配置，OpenFeign默认使用JDK自带的HttpURLConnection发送HTTP请求，

由于默认HttpURLConnection没有连接池、性能和效率比较低 ，如果采用默认，无法发挥最大性能，故使用Apach的HTTPClient 5替换默认HTTPURLConnection。

⚠️注意：httpclient的版本对齐

### 请求/响应压缩

[请求/响应压缩说明](https://docs.spring.io/spring-cloud-openfeign/reference/spring-cloud-openfeign.html#feign-request-response-compression)

Spring Cloud OpenFeign支持对请求和响应进行GZIP压缩，以减少通信过程中的性能损耗。

### 日志打印功能

[日志打印功能说明](https://docs.spring.io/spring-cloud-openfeign/reference/spring-cloud-openfeign.html#feign-logging)

Feign 提供了日志打印功能，我们可以通过配置来调整日志级别，

日志级别：

- NONE：默认的，不显示任何日志；
- BASIC：仅记录请求方法、URL、响应状态码及执行时间；
- HEADERS：除了 BASIC 中定义的信息之外，还有请求和响应的头信息；
- FULL：除了 HEADERS 中定义的信息之外，还有请求和响应的正文及元数据

## CircuitBreaker（断路器）

⚠️较为繁琐，资料少，不适合自学，面试必考

> 解决服务雪崩问题，对于有问题的节点，快速熔断（快速返回失败处理或者返回默认兜底数据【服务降级】）
>
> “断路器”本身是一种开关装置，当某个服务单元发生故障之后，通过断路器的故障监控（类似熔断保险丝），向调用方返回一个符合预期的、可处理的备选响应(FallBack)，而不是长时间的等待或者抛出调用方无法处理的异常，这样就保证了服务调用方的线程不会被长时间、不必要地占用，从而避免了故障在分布式系统中的蔓延，乃至雪崩。
>
> 当一个组件或服务出现故障时，CircuitBreaker会迅速切换到开放OPEN状态(保险丝跳闸断电)，阻止请求发送到该组件或服务从而避免更多的请求发送到该组件或服务。这可以减少对该组件或服务的负载，防止该组件或服务进一步崩溃，并使整个系统能够继续正常运行。同时，CircuitBreaker还可以提高系统的可用性和健壮性，因为它可以在分布式系统的各个组件之间自动切换，从而避免单点故障的问题。

[Spring官方介绍](https://spring.io/projects/spring-cloud-circuitbreaker)

Spring Cloud提供的两个实现类：Resilience4j、Spring Retry

### Resilience4j

[Resilience4j官网](https://github.com/resilience4j/resilience4j)

> Resilience4j 是一个轻量级的容错库，专为函数式编程设计。Resilience4j 提供了高阶函数（装饰器），可以增强任何函数式接口、lambda 表达式或方法引用，添加断路器、速率限制器、重试或舱壁。您可以在任何函数式接口、lambda 表达式或方法引用上堆叠多个装饰器。优点是可以自由选择所需的装饰器，而不需要其他任何东西。

#### 断路（Circuit Breaker）

##### 断路器状态

- OPEN
- CLOSED
- HALF_OPEN
- DISABLED(特殊状态)
- FORCED_OPEN(特殊状态)

##### 3大状态之间的转换

![官方状态转换图](https://files.readme.io/39cdd54-state_machine.jpg)

- 当熔断器关闭时，所有请求都会通过熔断器
  - 失败率超过设定的阈值，熔断器就会从CLOSED转到OPEN，这时所有的请求都会被拒绝
  - 当经过一段时间后，熔断器会从OPEN变为HALF_OPEN，这时有一定数量的请求会被放入，并重新计算失败率  
  - 如果失败率超过阈值，则变为OPEN状态，如果失败率低于阈值，则变为CLOSED状态
- 断路器使用滑动窗口来存储和统计调用的结果
  - 基于访问数量的滑动窗口：统计了最近N次调用的返回结果
  - 基于时间的滑动窗口：统计了最近N秒的调用返回结果
- 特殊状态
  - 这两个状态不会生成熔断时间，并且不回记录事件的成功或失败
  - 退出这两个状态的唯一方法是触发状态转换或者熔断器

##### 配置

| **failure-rate-threshold**                       | **以百分比配置失败率峰值**                                   |
| ------------------------------------------------ | ------------------------------------------------------------ |
| **sliding-window-type**                          | **断路器的滑动窗口期类型: 基于“次数”（COUNT_BASED）、“时间”（TIME_BASED）进行熔断，默认是COUNT_BASED。** |
| **sliding-window-size**                          | **若COUNT_BASED，则N次调用中有failure-rate-threshold%失败（即5次）打开熔断断路器；若为TIME_BASED则，此时还有额外的两个设置属性，含义为：在N秒内（sliding-window-size）100%（slow-call-rate-threshold）的请求超过N秒（slow-call-duration-threshold）打开断路器。** |
| **slowCallRateThreshold**                        | **以百分比的方式配置，断路器把调用时间大于slowCallDurationThreshold的调用视为慢调用，当慢调用比例大于等于峰值时，断路器开启，并进入服务降级。** |
| **slowCallDurationThreshold**                    | **配置调用时间的峰值，高于该峰值的视为慢调用。**             |
| **permitted-number-of-calls-in-half-open-state** | **运行断路器在HALF_OPEN状态下时进行N次调用，如果故障或慢速调用仍然高于阈值，断路器再次进入打开状态。** |
| **minimum-number-of-calls**                      | **在每个滑动窗口期样本数，配置断路器计算错误率或者慢调用率的最小调用数。比如设置为5意味着，在计算故障率之前，必须至少调用5次。如果只记录了4次，即使4次都失败了，断路器也不会进入到打开状态。** |
| **wait-duration-in-open-state**                  | **从OPEN到HALF_OPEN状态需要等待的时间**                      |

例子：

6次访问中当执行方法的失败率达到50%时CircuitBreaker将进入开启OPEN状态(保险丝跳闸断电)拒绝所有请求。

等待5秒后，CircuitBreaker将自动从开启OPEN状态过渡到半开HALF_OPEN状态，允许一些请求通过以测试服务是否恢复正常。

如还是异常CircuitBreaker将重新进入开启OPEN状态；如正常将进入关闭CLOSE闭合状态恢复正常处理请求。

```yaml
failure-rate-threshold: 50 # 设置50%的失败率阈值，超过失败请求百分比CircuitBreaker变为open
sliding-window-type: COUNT_BASED # 滑动窗口的类型
sliding-window-size: 6 # 滑动窗口的大小，单位为请求数
minimum-number-of-calls: 6 # 断路器计算失败率或慢调用之前所需的最小样本（每个滑动窗口周期）
automatic-transition-from-open-to-half-open-enabled: true # 是否启用自动从开启到半开启状态，默认值为false
wait-duration-in-open-state: 5s # 从OPEN到HALF_OPEN状态的等待时间
permitted-number-of-calls-in-half-open-state: 2 # 半开状态允许的最大请求数，默认值为10.
```

```yaml
failure-rate-threshold: 50 # 设置50%的失败率阈值，超过失败请求百分比CircuitBreaker变为open
slow-call-duration-threshold: 2s # 慢调用时间阈值，高于此时间的调用将被视为慢调用并增加调用比例。
slow-call-rate-threshold: 30 # 慢调用百分比阈值，超过此百分比的慢调用将触发断路器。
sliding-window-type: TIME_BASED
sliding-window-size: 2 # 滑动窗口的大小配置，配置TIME_BASED表示2秒
minimum-number-of-calls: 2 # 断路器计算失败率或慢调用之前所需的最小样本（每个滑动窗口周期）
permitted-number-of-calls-in-half-open-state: 2 # 半开状态允许的最大请求数，默认值为10.
wait-duration-in-open-state: 5s # 从OPEN到HALF_OPEN状态的等待时间
record-exceptions:
  - java.lang.Exception
```

#### 舱壁隔离（Bulkhead）

> 依赖隔离&负载保护：用来限制对于下游服务的最大并发的限制

[舱壁隔离说明](https://github.com/lmhmhl/Resilience4j-Guides-Chinese/blob/main/core-modules/bulkhead.md)

##### 两种隔离方式

- 信号量舱壁（SemaphoreBulkhead）：
  - 当信号量有空闲时，进入系统的请求会直接获取信号量并开始业务处理。
  - 当信号量全备占用时，接下来的请求将会进入阻塞状态，SemaphoreBulkhead提供了一个阻塞计时器
    - 如果阻塞状态的请求在阻塞计时内无法获取到信号量则系统会拒绝这些请求。
    - 若请求在阻塞计时内获取到了信号量，那将直接获取信号量并执行相应的业务处理
- FixedThreadPoolBulkhead使用了有界队列和固定大小线程池
  - 当线程池中存在空闲时，则此时进入系统的请求将直接进入线程池开启新线程或使用空闲线程来处理请求
  - 当线程中无空闲时，接下来的请求进入等待队列
    - 若等待队列仍然无剩余空间时接下来的请求将直接被拒绝
    - 在队列中的请求等待线程池出现空闲时，将进入线程池进行业务处理
  - 另，ThreadPoolBulkhead只对CompletableFuture方法有效，所以必须创建返回CompletableFuture类型的方法

#### 速率限制（Rate Limiter）

[速率限制说明](https://github.com/lmhmhl/Resilience4j-Guides-Chinese/blob/main/core-modules/ratelimiter.md)

##### 限流算法

- 漏斗算法（Leaky Bucket）
  - 一个固定容量的漏桶，按照设定常量固定速率流出水滴，类似医院打吊针，不管你源头流量多大，我设定匀速流出。
  - 如果流入水滴超出了桶的容量，则流入的水滴将会溢出了(被丢弃)，而漏桶容量是不变的。
  - 缺点：对于存在突发特性的流量来说缺乏效率。
- **令牌桶算法（Token Bucket）**
  - Spring Cloud 默认使用的算法
  - 当用户发起请求，先判断桶空不空
    - 令牌桶算法会匀速的添加令牌至令牌桶中，桶可容纳令牌的数量是有限的。
    - 用户每次发起请求，先检查桶是否为空。若桶空，则丢弃请求；若桶不空，则申请获得令牌，获得令牌则可排队让处理器处理当前请求
- 滚动时间窗口算法（Tumbling time window）
  - 允许固定数量的请求进入(比如1秒取4个数据相加，超过25值就over)超过数量就拒绝或者排队，等下一个时间段进入。
  - 由于是在一个时间间隔内进行限制，如果用户在上个时间间隔结束前请求（但没有超过限制），同时在当前时间间隔刚开始请求（同样没超过限制），在各自的时间间隔内，这些请求都是正常的。
  - 缺点：间隔临界的一段时间内的请求就会超过系统限制，可能导致系统被压垮。
- **滑动时间窗口算法（Sliding time window）**
  - 滑动窗口算法是把固定时间片进行划分并且随着时间移动，移动方式为开始时间点变为时间列表中的第2个时间点，结束时间点增加一个时间点。
  - 不断重复，通过这种方式可以巧妙的避开计数器的临界点的问题。

## Sleuth(Micrometer) + Zipkin（分布式链路追踪）

> 分布式链路追踪（Distributed Tracing），就是将一次分布式请求还原成调用链路，进行日志记录，性能监控并将一次分布式请求的调用情况集中展示。比如各个服务节点上的耗时、请求具体到达哪台机器上、每个服务节点的请求状态等等。
>
> ⚠️ Sleuth停更进维， Sleuth替代方案Micrometer

[Micrometer官方文档](https://micrometer.io/docs/tracing)

### 分布式链路追踪原理

两个关键id：Track Id（链路id）、Span Id（节点id）、Parent Id（父级节点id，Span Id）

主要通过Span Id记录整条链路

### Dashboard

- [ZipKin](https://zipkin.io/)：由Twitter公司开源，开放源代码分布式的跟踪系統，用于收集服务的定时数据，以解决微服务架构中的延迟问
  题，包括：数据的收集、存储、查找和展现。结合spring-cloud-sleuth使用较为简单，集成方便，但是功能较
  简单。
- [Cat](https://github.com/dianping/cat)：由大众点评开源，基于Java开发的实时应用监控千台，包括实时应用监控，业务监控。集成方案是通过代码埋
  点的方式来实现监控，比如：拦截器，过滤器等。对代码的侵入性很大，集成成本较高。风险较大。
- [Pinpoint](https://github.com/pinpoint-apm/pinpoint)：Pinpoint是一款开源的基于字节码注入的调用链分析，以及应用监控分析工具。特点是支持多种插件，UI功能
  强大，接入端无代码侵入。
- [Skywalking](https://skywalking.apache.org/)： SkyWalking是国人开源的基于字节码注入的调用链分析，以及应用监控分析工具。特点是支持多
  种插件，UI功能较强，接入端无代码侵入。

### 整合Micrometer Tracing

由于Micrometer Tracing是一个门面工具自身并没有实现完整的链路追踪系统，具体的链路追踪另外需要引入的是第三方链路追踪系统的依赖：

- micrometer-tracing-bom  导入链路追踪版本中心，体系化说明
- micrometer-tracing  指标追踪
- micrometer-tracing-bridge-brave  一个Micrometer模块，用于与分布式跟踪工具 Brave 集成，以收集应用程序的分布式跟踪数据。Brave是一个开源的分布式跟踪工具，它可以帮助用户在分布式系统中跟踪请求的流转，它使用一种称为"跟踪上下文"的机制，将请求的跟踪信息存储在请求的头部，然后将请求传递给下一个服务。在整个请求链中，Brave会将每个服务处理请求的时间和其他信息存储到跟踪数据中，以便用户可以了解整个请求的路径和性能。
- micrometer-observation  一个基于度量库 Micrometer的观测模块，用于收集应用程序的度量数据。
- feign-micrometer  一个Feign HTTP客户端的Micrometer模块，用于收集客户端请求的度量数据。
- zipkin-reporter-brave  一个用于将 Brave 跟踪数据报告到Zipkin 跟踪系统的库。
- 补充包：spring-boot-starter-actuator  SpringBoot框架的一个模块用于监视和管理应用程序

## Gateway（服务网关 ）

> 以前都是用Zuul，但是Zuul更新太水了，Spring Cloud 自己研发了Gateway替代Zuul
>
> Spring Cloud Gateway组件的核心是一系列的过滤器，通过这些过滤器可以将客户端发送的请求转发(路由)到对应的微服务。 Spring Cloud Gateway是加在整个微服务最前沿的防火墙和代理器，隐藏微服务结点IP端口信息，从而加强安全保护。Spring Cloud Gateway本身也是一个微服务，需要注册进服务注册中心。

[Spring官方介绍](https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webmvc)

### 三大核心

- 路由（Route）：网关的基本构建块。它由一个ID、一个目标URI、一组断言和一组过滤器定义。如果断言为True，则匹配路由。
- 断言（Predicate）：参考的是Java8中的java.util.function.Predicate,  允许匹配HTTP请求中的任何内容，例如头或参数。如果请求与断言相匹配则进行路由。
- 过滤器（Filter）：指的是Spring框架中GatewayFilter的实例，使用过滤器，可以在请求被路由前或者之后对请求进行修改。

### Gateway工作流程

[工作流程说明](https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webmvc/how-it-works.html)

客户端向 Spring Cloud Gateway 发出请求。然后在 Gateway Handler Mapping 中找到与请求相匹配的路由，将其发送到 Gateway Web Handler。Handler 再通过指定的过滤器链来将请求发送到我们实际的服务执行业务逻辑，然后返回。

过滤器之间用虚线分开是因为过滤器可能会在发送代理请求之前(Pre)或之后(Post)执行业务逻辑。

在“pre”类型的过滤器可以做参数校验、权限校验、流量监控、日志输出、协议转换等;

在“post”类型的过滤器中可以做响应内容、响应头的修改，日志的输出，流量监控等有着非常重要的作用。

核心：路由转发 + 断言判断 + 执行过滤器链

### Route以微服务名 - 动态获取服务URI

在配置route的uri时使用 lb://服务名 即可

### Predicate断言

[Predicate断言说明](https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/request-predicates-factories.html)

Spring Cloud中Gateway有一个RoutePredictFactory，通过RoutePredicate工厂类可以创建Predict对象，Predict对象用于Route的匹配。Spring Cloud Gateway中包含多个内置的Route Predicate Factories：After、Before、Between、Cookie、Header、Host、Method、Path、Query、ReadBody、RemoteAddr、XForwardedRemoteAddr、Weight、CloudFoundryRouteService，当然也可以自定义。

### Filter过滤

[Filter过滤说明](https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/gatewayfilter-factories.html)

功能上类似SpringMVC里的拦截器Interceptor，Servlet里的过滤器

"pre"和"post"分别会在请求被执行前调用和被执行后调用，用来修改请求和响应信息

作用：请求鉴权、异常处理、记录接口调用时长统计

过滤器分类

- 单一内置过滤器
- 自定义过滤器
