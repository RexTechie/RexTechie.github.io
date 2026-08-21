---
title: "Spring Boot原理篇"
date: 2025-07-15T20:48:37+08:00
draft: false
description: "Spring Cloud中间件的介绍"
categories: ["☕Java 后端"]
tags: ["Spring Boot", "自动配置", "Spring MVC"]
---

> 本文侧重📝记录SpringBoot3框架源码

[官方文档](https://docs.spring.io/spring-boot/index.html)

## 核心注解

> 理解SpringBoot源码的核心注解分别是组件注册注解、条件注解、属性绑定注解，这几个注解与SpringBoot的自动装配机制息息相关！

### 组件注册

> 组件注册是将对象存储到Spring boot的IoC容器中

- **@Component**：将类作为组件注册到IoC容器中。
- @ComponentScan
  - 扫描指定路径下需要注册到容器中的组件。
  - @SpringBootApplication注解中默认将@SpringBootApplication所标注的类所在位置标注为扫描路径。
- @Bean、@Scope：通常标注在方法上，将方法返回的对象注册到ioc容器中。
  - 默认是单例(singleton)
  - 通常用于导入第三方组件
  - 可以通过@Scope来修改容器的作用域
    - singleton：单例模式（默认值），整个 Spring 容器中只创建一个 Bean 实例。
    - prototype：原型模式，每次获取 Bean 时都会创建一个新的实例。
    - request：每个 HTTP 请求创建一个 Bean，仅在 Web 应用中有效。
    - session：每个 HTTP Session 创建一个 Bean，仅在 Web 应用中有效。
- @Import
  - 通常用与将第三方组件导入到容器中
- 其他：
  - 配置类组件注册注解：@Configuration、@SpringBootConfiguration。@Configuration = @Component， @SpringBootConfiguration =  @Configuration + @Index
  - Web开发类组件注册注解，@Controller、@Service、@Repository = @Component
  - ...

### 条件注解


`@ConditionalOnXXX`如果指定注解的条件成立，则触发指定行为，例子：@ConditionalOnClass, 如果类路径中存在这个类，则触发指定行为，

- 若将这个注解放在类级别，如果注解判断生效，则整个配置类生效
- 若放在方法级别，单独对这个方法进行注解判断

### 属性绑定

> 将容器中任意组件的属性值和配置文件的配置项的值进行绑定

`@ConfigurationProperties`、`@EnableConfigurationProperties`（常用于第三方组件的属性绑定，默认会把组件放到容器中）

绑定过程

1. 给容器中注册组件(@Bean/@Component)
2. 使用`@ConfigurationProperties`声明组件和配置文件的哪些配置项进行绑定

## 核心原理

### 启动过程

![springboot_life_cycle](/images/20250715205742.png)

1. Listener从`META-INF/spring.factories`读到
2. 引导：利用 `BootstrapContext`引导整个项目启动
   1. starting: 应用开始，只要有了`BootstrapContext`就直接调用
   2. `prepareEnvironment`: 配置环境变量，但是ioc还没创建
3. 启动：
   1. `contextPrepared`:   创建ioc容器，但是sources（主配置类）还没加载。并关闭引导上下文。
   2. `contextLoaded`：ioc容器加载。主配置类加载进去了。但是ioc容器还没刷新（我们的bean还没创建）
   3. `started`:  ioc容器刷新了（所有的bean还没创建），**但是runner没调用**。
   4. `ready`：ioc容器刷新了（所有的bean还没创建），**而且runner调用完了**。
4. 运行：
   1. 以前步骤都正确执行，代表容器running

#### 回调监听器

- `BootStrapRegistryInitializer`：感知特定阶段：感知引导初始化
  - `META-INF/spring.factories`
  - 创建引导上下文`bootstrapContext`的时候触发
  - 场景：进行密钥校对授权
- `ApplicationContextInitializer`：感知特定阶段：感知ioc容器初始化
- **`ApplicationListener`**：感知全阶段 ：基于事件机制，感知事件。
- **`SpringApplicationRunListener`**：感知全阶段： 各种阶段都能自定义操作
- `ApplicationRunner`：感知特定阶段：感知应用就绪Ready。卡死应用，就不会就绪。
- `CommandLineRunner`：感知特定阶段：感知应用就绪Ready。卡死应用，就不会就绪。

#### 9大事件

1. `ApplicationStartingEvent`：应用启动但未做任何事情，除过注册listeners and initializers
2. `ApplicationEnvironmentPreparedEvent`：Environment准备好，但context未创建
3. `ApplicationContextInitializedEvent`：ApplicationContext准备好，ApplicationContextInitializers调用，但是任何bean未加载
4. `ApplicationPreparedEvent`：容器刷新之前，bean定义信息加载
5. `ApplicationStartedEvent`：容器刷新完成，runner未调用
6. `AvailabilityChangeEvent`（存活探针）：`LivenessState.CORRECT`应用存活
7. `ApplicationReadyEvent`：任何runner被调用
8. `AvailabilityChangeEvent`（就绪探针）：`ReadinessState.ACCEPTING_TRAFFIC`应用就绪，可以接受请求
9. `ApplicationFailedEvent`：启动出错

### 自动配置原理

> 应用关注的三大核心：场景、配置、组件
>
> 自动配置原理是依赖于SPI思想
>
> SPI（**Service Provider Interface**）是一种为服务提供者（Service Providers）设计的接口，它允许服务的实现与服务的使用者（Service Consumers）解耦。通过SPI，开发者可以为某个特定的接口提供实现，而应用程序可以通过标准接口来使用这些实现。SPI 是 Java 和一些其他编程环境中常用的设计模式。

#### 自动装配完整流程

1. 导入`spring-boot-starter-web`
   1. starter导入了相关场景所有的依赖：starter-json、starter-tomcat、spring-mvc
   2. 每个starter中引入了核心starter`spring-boot-starter`
   3. **核心starter**中引入了`spring-boot-autoconfigure`依赖
   4. `spring-boot-autoconfigure`包含了所有场景所需要的配置
   5. 只要这个包下的所有类都能生效，那么相当于Spring Boot官方写好的整合功能就生效
   6. SpringBoot默认扫描不到`spring-boot-autoconfigure`下写好的所有配置类。
2. 主程序：`@SpringBootApplication`。
   1. `@SprinBootApplication` = `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`。
   2. SpringBoot默认只能扫描主程序所在的包及其子包，扫描不到`spring-boot-autoconfigure`包中官方写好的配置类。
   3. `@EnableAutoConfiguration`：SpringBoot开启自动配置的核心。
      1. `@Import(AutoConfigurationImportSelector)`批量给容器中导入组件。
      2. SpringBoot启动会默认加载156（springboot3.5.0, 不同版本数量不同）。
      3. 这156个配置来自于`spring-boot-autoconfigure`下`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`文件指定的。
      4. 项目启动的时候利用`@Import`批量导入组件机制把`autoconfigure`包下的156个`XxxAutoConfiguration`类导入进来（自动配置类）。
   4. 虽然导入了156个自动配置类，但并不是所有自动配置类都能生效，每一个自动配置类，都有条件注解`ConditionalOnXxx`，只有条件成立，才能生效。
3. `XxxAutoConfiguration`自动配置类
   1. 给容器中使用`@Bean`给容器放一堆组件。
   2. 每个自动配置类都可能有`@EnableConfigurationProperties(XxxProperties.class)`,用来把配置文件中的属性值封装到`XxxProperties`属性类中。
   3. 以Tomcat为例：把服务器的所有配置都是以server开头的封装到属性类中。
   4. 只需要改配置文件的值，核心组件的底层参数都能修改
4. 业务开发

#### 功能开关

- 自动配置：全部都配置好，什么都不用管
- 手动配置：`@EnableXxxx`手动控制那些功能启动，都是利用`@Import`把对应功能导入进来

#### 自定义starter流程

1. 创建`自定义starter`项目，引入`spring-boot-starter`依赖
2. 编写starter功能代码，引入相关依赖
3. 实现自动装配
   1. 编写`XxxAutoConfiguration`自动配置类，帮其他项目导入这个模块需要的所有组件
   2. `@EnableXxx`（可选）：模仿`@EnableWebMvc`写`@EnableXxx`，可实现通过在启动类上加上这个注解就能实现自动装配
   3. 全自动方式（可选）：编写配置文件`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
4. 其他项目引入即可

## Web篇

[Spring Boot Web介绍](https://docs.spring.io/spring-boot/reference/web/servlet.html)

### 默认配置

1. 包含了`ContentNegotiatingViewResolver`和`BeanNameViewResolve`组件，用于视图解析。
2. 默认进行了静态资源处理机制：静态资源在static文件夹下即可直接访问。
3. 自动注册了`Converter`、`GenericConverter`、`Formatter`组件，适配常见的数据类型转换和格式化需求。
4. 支持`HttpMessageConverters`，方便`json`类型数据。
5. 注册`MessageCodesResolver`，方便国际化及错误消息处理。
6. 支持静态`index.html`
7. 自动使用`ConfigurableWebBindingInitializer`，实现`消息处理`，`数据绑定`、`类型转换`等功能。

‼️重要

- 如果要保持boot mvc的默认配置，并且自定义更多的mvc配置，如：interceptors、formatters、view controller等。可以使用`@Configuration`注解添加一个WebMvcConfigure类型的配置类，并不要标注@EnableWebMvc。
- 如果想要保持boot mvc的配置，但要自定义核心组件实例，比如：`RequestMappinghHandlerMapping`、`RequestMappingHandlerAdapter`或`ExceptionHandlerExceptionResolver`，给容器中放一个`WebMvcRegistrations`组件即可。
- 如果想全面接管Spring MVC，`@Configuration`标注一个配置类，并加上`@EnableWebMvc`注解，实现`WebMvcConfigurer`接口。

这部分自定义配置的原理

- WebMvcAutoConfiguration是一个自动配置类，里面有一个`EnableWebMvcConfiguration`
- `EnableWebMvcConfiguration`继承与`DelegatingWebMvcConfiguration`，这两个都生效
- `DelegatingWebMvcConfiguration`利用依赖注入获取容器中所有的`WebMvcConfiguration`
- 别人调用`DelegatingWebMvcConfiguration`的方法配置底层方法，而它调用所有`WebMvcCoinfiguration`的配置底层方法

### 原理

WebMvcAutoConfiguration原理

1. 放了两个过滤器: 
   1. `HiddenHttpMethodFilter`: 页面表单提交REST请求（GET、POST、PUT、DELETE...）。
   2. `FormContentFilter`:  表单内容过滤器，一版来说GET（url）、POST（请求体）请求可以携带数据，PUT、DELETE的请求体数据会被忽略数据。
2. 给容器中放了`WebMvcAutoConfigurationAdapter组件.
   1. 该组件实现了`WebConfig`；给SpringMVC添加各种定制功能.
   2. 这个组件绑定了两个Properties.
      1. WebMvcProperties: `spring.mvc`
      2. WebProperties: `spring.web`
3. `WebMvcConfigure`接口
   1. configurePathMatch: 配置路径匹配
   2. configureContentNegotiation: 配置内容协商
   3. configureAsyncSupport: 配置异步支持，处理异步请求
   4. configureDefaultServletHandling: 配置默认处理，默认接收:/
   5. addFormatters: 添加格式化器
   6. addInterceptors: 添加拦截器
   7. **addResourceHandlers**: 添加资源处理器，用于处理静态资源
   8. addCorsMappings: 添加跨域
   9. addViewControllers: 添加视图控制器，如令/a直接跳转到xxx.html
   10. configureViewResolvers: 配置视图解析
   11. addArgumentResolvers: 添加参数解析器
   12. addReturnValueHandlers: 添加返回值处理器，跳转页面/返回数据
   13. configureMessageConverters: 配置消息转换器
   14. extendMessageConverters: 扩展消息转换器
   15. configureHandlerExceptionResolvers: 配置异常解析器
   16. extendHandlerExceptionResolvers: 扩展异常解析起
   17. addErrorResponseInterceptors: 添加错误响应拦截器

### 静态资源

`WebMvcAutoConfigurationAdapter`中`addResourceHandlers`源码

```java
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
  if (!this.resourceProperties.isAddMappings()) {
    logger.debug("Default resource handling disabled");
    return;
  }
  addResourceHandler(registry, this.mvcProperties.getWebjarsPathPattern(),
      "classpath:/META-INF/resources/webjars/");
  addResourceHandler(registry, this.mvcProperties.getStaticPathPattern(), (registration) -> {
    registration.addResourceLocations(this.resourceProperties.getStaticLocations());
    if (this.servletContext != null) {
      ServletContextResource resource = new ServletContextResource(this.servletContext, SERVLET_LOCATION);
      registration.addResourceLocations(resource);
    }
  });
}
```

1. 规则一：访问：`/webjars/**`路径就去`classpath:/META-INF/resources/webjars`下找资源。
2. 规则二：访问：`/**`路径就去静态资源默认的四个位置找资源：`classpath:/META-INF/resources/`、`classpath:/resources/`、 `classpath:/static/`、 `classpath:/public/`
3. 规则三：静态资源默认都有缓存规则的设置，如果浏览器访问了一个静态资源`index.js`，如果服务器这个资源没有发生变化，下次访问的时候就可以直接让浏览器用自己缓存中的东西，而不用给服务器发请求。
   1. 所有缓存的设置，直接通过配置文件: `spring.web`
   2. cachePeriod: 缓存周期，多久不用找服务器要新的。默认没有，以s为单位
   3. cacheControl: HTTP缓存控制
   4. userLastModified: 是否使用最后一次修改，配合HTTP Cache规则

#### 自定义 - 配置方式

`spring.mvc`: 静态资源访问前缀路径

`spring.web`:

- 静态资源目录
- 静态资源缓存策略

#### 自定义 - 代码方式

> 容器中只要有一个WebMvcConfigure组件。配置的底层行为都会生效
>
> @EnableWebMvc  //禁用boot的默认配置

```java
@Configuration
public class MyConfig implements WebMvcConfigurer {
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        WebMvcConfigurer.super.addResourceHandlers(registry);

        registry.addResourceHandler("/static/**")
                .addResourceLocations("classpath:/static/")
                .setCacheControl(CacheControl.maxAge(7200, TimeUnit.SECONDS));

    }
}
```

### 路径匹配

> Spring5.3之后加入了更多的请求路径匹配的实现策略：`PathPatternParser`
>
> 以前只支持`AntPathMatcher`
>
> 默认使用新版的匹配路径，可以配置spring.mvc.pathmatch.matching-strategy来修改匹配策略

#### 两者区别

- `PathPatternParser`在jmh基准测试下，有6~8倍吞吐量提升，降低30%~40%空间分配率
- `PathPatternParser`兼容`AntPathMatcher`语法，并支持更多类型的路径模式
- `PathPatternParser`中`**`多目录匹配的支持仅运行在模式末尾使用

### 内容协商

> 一套系统适配多端数据返回

#### HttpMessageConverter原理

1. `@ResponseBody`由`HttpMessageConverter`处理
   1. 若有类被`@ResponseBody`标注
      1. 请求先经过`DispatcherServlet`的`doDispatch()`进行处理
      2. 找到一个`HandlerAdapter`适配器，利用适配器执行目标方法
      3. `RequestMappingHandlerAdapter`来执行，调用`invokeHandlerMethod()`来执行方法
      4. 目标方法执行前，准备好两个东西
         1. `HandlerMethodArgumentResolver`：参数解析器，确定目标方法的每个参数值
         2. `HandlerMethodReturnValueHandler`：返回值解析器，确定目标方法的返回值怎么处理
      5. `RequestMappingHandlerAdapter`里面的`invokeAndHandle()`真正执行目标方法
      6. 目标方法执行完成，会返回返回值对象
      7. 找到一个合适的返回值处理`HandlerMethodReturnValueHandler`，即`RequestResponseBodyMethodProcessor`
      8. `RequestResponseBodyMethodProcessor`利用`writeWithMessageConverters`把返回值写出去
   2. `HttpMessageConverter`会先进行内容协商
      1. 遍历所有的`HttpMessageConverter`看谁支持这种内容类型的数据
      2. 最终因为要`json`所以使用`MappingJackson2HttpMessageConverter`
      3. `jackson`用`ObjectMapper`把对象写出去
2. `WebAutoConfiguration`提供的默认`HttpMessageConverters`
   1. ByteArrayHttpMessageConverter：字节数据读写
   2. StringHttpMessageConverter：字符串读写
   3. ResourceHttpMessageConverter：资源读写
   4. ResourceRegionHttpMessageConverter：分区资源读写
   5. AllEncompassingFormHttpMessageConverter：表单读写
   6. MappingJackson2HttpMessageConverter：请求响应体Json读写
   7. MappingJackson2XmlHttpMessageConverter：请求响应体Xml读写

### 异常处理

> 错误处理的自动配置都在`ErrorMvcAutoConfiguration`中，两大核心
>
> - SpringBoot会自适应处理错误，响应页面或JSON数据
> - Spring MVC的错误处理机制依然保留，MVC处理不了，才会交给Boot进行处理

#### 原理

1. SpringBoot中的异常处理首先会依次经过@ExceptionHandler标注的方法、@ResponseStatus标注的方法、@SpringMVC定义的默认错误响应处理方法进行处理，若不能处理则跳转至`/error`页面，而`/error`页面是由`BasicErrorController`进行处理的。

2. `BasicErrorController`中`errorHTML`会处理异常跳转的逻辑

   ```java
   @RequestMapping(produces = MediaType.TEXT_HTML_VALUE)
   public ModelAndView errorHtml(HttpServletRequest request, HttpServletResponse response) {
       HttpStatus status = getStatus(request);
       Map<String, Object> model = Collections
           .unmodifiableMap(getErrorAttributes(request, getErrorAttributeOptions(request, MediaType.TEXT_HTML)));
       response.setStatus(status.value());
       ModelAndView modelAndView = resolveErrorView(request, response, status, model);
       return (modelAndView != null) ? modelAndView : new ModelAndView("error", model);
   }
   ```

   

   1. 首先他会从容器中找到能够处理该错误状态（304、404）的视图页面

      ```java
      @Override
      public ModelAndView resolveErrorView(HttpServletRequest request, HttpStatus status, Map<String, Object> model) {
          ModelAndView modelAndView = resolve(String.valueOf(status.value()), model);
          if (modelAndView == null && SERIES_VIEWS.containsKey(status.series())) {
             modelAndView = resolve(SERIES_VIEWS.get(status.series()), model);
          }
          return modelAndView;
      }
      ```

      1. 如果发生了304、404等错误，默认在classpath:/templates/error/状态码.html
      2. 如果没有对应的视图页面，在静态资源文件夹下找`状态码.html`
      3. 如果无法匹配精确的状态码的页面，则模糊找`5xx.html`、`4xx.html`,
      4. 如果没有对应的视图页面，在静态资源文件夹下找`5xx.html`、`4xx.html`

   2. 若没有能处理的，则跳转到/error页面

#### 自定义错误响应

- 前后端分离
  - 后台发生的所有错误，`@ControllerAdvice + @ExceptionHandler`进行统一异常处理
- 服务端渲染
  - 给`classpath:/templates/error/`下放错误页面: 500.html、5xx.html。。。
- 发生业务错误
  - 核心业务，每一种错误，都应该代码控制，跳转到自己定制的错误页
  - 通用业务，`classpath:/templates/error.html`页面，显示错误信息

### Web嵌入式容器

> 管理、运行Servlet组件（Servlet、Filter、Listener）的环境，一般指服务器

#### 原理

1. `ServletWebServerFactoryAutoConfiguration`自动配置了容器场景,绑定了`ServerProperties`，配置了嵌入式三大服务器（Tomcat、Jettty、Undertow）

2. 通过条件注解，来确定使哪个嵌入式服务器生效

3. 每个服务器都有`ServletWebServerFactory`，每个Web服务器工厂都有`getWebServer`方法来获取web服务器

4. `ServletWebServerApplicationContext`ioc容器启动时会调用创建web服务器`createWebServer()`,而`createWebServer()`会调用`getWebServer()`方法

5. Spring容器刷新（启动）的时候，会预留一个时机，刷新子容器。`onRefresh()`,而容器刷新的时候会调用`createWebServer()`方法

   ```java
   @Override
   protected void onRefresh() {
       super.onRefresh();
       try {
          createWebServer();
       }
       catch (Throwable ex) {
          throw new ApplicationContextException("Unable to start web server", ex);
       }
   }
   ```

6. Refresh容器刷新经典12大步的刷新子容器会调用onRefresh

### Spring Boot中的SpringMVC

#### 原理

`WebMvcAutoConfiguration`web自动配置类分析

> Spring MVC自动配置场景配置了如下所有场景

1. `OrderedHiddenHttpMethodFilter`：支持RESTful的filter
2. `OrderedFormContentFilter`：支持非POST请求，请求提携带数据
3. `WebMvcAutoConfigurationAdapter`配置生效，它是一个`WebMvcConfigurer`，定义mvc底层组件
   1. 定义好`WebMvcConfigurer`底层组件默认功能
   2. `InternalResourceViewResolver`、`BeanNameViewResolver`：视图解析器
   3. `ContentNegotiatingViewResolver`：内容协商解析器
   4. `RequestContextFilter`：请求上下文的过滤器， 使得请求在任意位置都可以获取当前请求
4. `EnableWebMvcConfiguration`
   1. `RequestMappingHandlerAdapter`
   2. `WelcomePageHandlerMapping`：欢迎页功能支持（模板引擎目录、静态资源目录）
   3. `LocaleResolver`：国际化解析器
   4. `ThemeResolver`：主题解析器（已弃用）
   5. `FlashMapManager`：临时数据解析器
   6. `FormattingConversionService`：数据格式化、类型转换
   7. `Validator`：数据校验'JSR'提供的数据校验功能
   8. `WebBindingInitializer`：请求参数的封装与绑定
   9. `RequestMappingHandlerMapping`：处理请求的映射关系
   10. `ExceptionHandlerExceptionResolver`：默认的异常解析器
   11. `ContentNegotiationManager`：内容协商管理器
5. `ResourceChainResourceHandlerRegistrationCustomizer`：静态资源链规则
6. `ProblemDetailsErrorHandlingConfiguration`：错误详情，捕获SpringMVC内部常见异常

`@EnableWebMvc`会禁用`WebMvcAutoConfiguration`配置的信息

1. 给容器导入了`DelegatingWebMvcConfiguration`，他是`WebMvcConfigurationSupport`

2. 而`WebMvcAutoConfiguration`中有条件注解

   ```java
   @ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
   ```

3. 因此`@EnableWebMvc`标注的注解，导入了`WebMvcConfigurationSupport`，使得`WebMvcAutoConfiguration`失效

## 数据访问

### jdbc场景的自动配置

> 配置了数据源等基本信息

1. `mybatis-spring-boot-starter`导入了`spring-boot-starter-jdbc`，jdbc是操作数据库的场景
2. JDBC场景中的自动配置
   1. `DataSourceAutoConfiguration`：数据源的自动配置。默认使用`HikariDataSource`
   2. `JdbcTemplateAutoConfiguration`： 给容器中放了`JdbcTemplate`
   3. `JndiDataSourceAutoConfiguration`：允许数据源使用JNDI的方式进行连接
   4. `XADataSourceAutoConfiguration`：基于XA二阶段提交协议的分布式事务数据源
   5. `DataSourceTransactionManagerAutoConfiguration`：支持事务

### mybatis场景的自动配置

1. `mybatis-spring-boot-starter`默认加载两个自动配置类
   1. `MybatisLanguageDriverAutoConfiguration`，语言驱动
   2. `MybatisAutoConfiguration`
      1. 必须在数据源配置好之后进行配置
      2. `SqlSessionFactory`：通过工厂创建`SqlSession`，即数据库的操作绘画
      3. `SqlSessionTemplate`：操作数据库的工具
2. MapperScan原理：利用`@Import`注解导入`MapperScannerRegistrar`类
   1. `registerBeanDefinitions()`解析指定包路径里面的每一个类，为每一个Mapper接口类，创建Bean定义信息，注册到容器
