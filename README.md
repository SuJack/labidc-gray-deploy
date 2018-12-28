
# labidc-gray-deploy 
##### spring cloud 非侵入式轻量级灰度发布插件
##### 基于spring-boot-starter-parent 2.0.1.RELEASE开发
#### 前言
##### 本来该插件设想是非暴力修改，扩展方式重写 LoadBalancerRule 相关负载均衡器规则，但发现会有并发问题，所以这里暴力重写了LoadBalancerRule 相关类，以后时间允许的情况下再做调整与修改（不忘初心）
##### 该文档使用了 mermaid 规范制作了流程图，如果需要查看流程图，请在ide里安装mermaid插件再预览，vscode有mermaid插件
##### 该灰度发布组件也可以用做测试环境多版本微服务并行测试使用。


## 开始
##### 1. 该插件开发了两个版本 分别支持 consul 和 eureka 两种注册中心。
##### 2. demo项目也对其有分别的实现与配置，分别实现了 zuul 和 gateway 两个网关案例。
##### 3. 但这里还是要详细说明一下使用方法，防止掉坑
## 注意：有可能出现的坑，该插件没有加入【调用链】与【熔断器】一起测过， 还有就是上传文件未测，长链接未测，可能会有坑, 欢迎测试。

### 实现原理：
1. 一般我们微服务调用是多层级的，比如 

``` mermaid
graph LR
    start[正常请求] --> input[网关 gateway/zuul ]
    input --> conditionA{负载均衡算法}
    conditionA -- 命中 --> B{服务1-实例1}
    conditionA -- 未命中 --> C{服务1-实例2}
    B --> D{服务2}
    D --> E{服务3}
```
2. 这个时候，如果我们整个链条下的某个服务开发了新版本需要灰度发布，但是其他服务是正式版本, 比如 服务1 有个灰度版本需要上线

``` mermaid
graph LR
    start[正常请求] --> input[网关 gateway/zuul ]
    start2[灰度请求] -. version:1.0.1 .-> input[网关 gateway/zuul ]
    input --> conditionA{负载均衡算法}
    input -. version:1.0.1 .-> 服务1-version:1.0.1
    服务1-version:1.0.1 --> D{服务2}
    conditionA -- 命中 --> B{服务1-实例1}
    conditionA -- 未命中 --> C{服务1-实例2}
    B --> D{服务2}
    D --> E{服务3}
```

3. 如果我们 服务2 有灰度版本，如下

``` mermaid
graph LR
    start[正常请求] --> input[网关 gateway/zuul ]
    start2[灰度请求] -. version:1.0.1 .-> input[网关 gateway/zuul ]
    input --> conditionA{负载均衡算法}
    input -. version:1.0.1 .-> conditionA{负载均衡算法}
    conditionA -- 命中 --> B{服务1-实例1}
    conditionA -. version:1.0.1 .-> B{服务1-实例1}
    conditionA -- 未命中 --> C{服务1-实例2}
    B --> D{服务2}
    B -. version:1.0.1 .-> F{服务2-version:1.0.1}
    D --> E{服务3}
    F --> E{服务3}
```


4. 或者所有服务都有灰度版本

``` mermaid
graph LR
    start[正常请求] --> input[网关 gateway/zuul ]
    input --> conditionA{负载均衡算法}
    conditionA -- 命中 --> B{服务1-实例1}
    conditionA -- 未命中 --> C{服务1-实例2}
    B --> D{服务2}
    D --> E{服务3}
    start2[灰度请求] -. version:1.0.1 .->  input[网关 gateway/zuul ]
    input -. version:1.0.1 .->  conditionA2{负载均衡算法}
    conditionA2 -. version:1.0.1命中 .-> B2{服务1-version:1.0.1-实例1}
    conditionA2 -. version:1.0.1未命中 .-> C2{服务1-version:1.0.1-实例2}
    B2 -. version:1.0.1命中 .-> D2{服务2-version:1.0.1}
    D2 -. version:1.0.1命中 .-> E2{服务3-version:1.0.1}
```

5. 基本原理介绍完了，说一下细节，上面流程图中，服务3，没有调用下层服务了，相当于他是最底层服务【叶级服务】。
所以这个层级的服务，只需要设置 标识就行了，标注他是不是灰度服务，同时标注他的版本号, consul和eureka 设置方法不同。
同时不需要安装插件

6. 对于服务2，服务1 整个调用链中间的服务，同样需要适应上面第5条，如果是灰度服务就需要标注灰度版本，同时需要安装插件。

7. 由于 gateway 基于 netty+webflux，和zuul 核心原理不同，zuul 基于 Servlet 容器, 每个请求独立单线程处理，
webflux 已经不止一个线程跑到底，所以在当前请求线程上下文拿不到原请求的ThreadLocal 对象。 因此zuul 可以直接使用该插件
gateway 无法直接使用，所以 gateway 需要一种变通的方式来处理，下面会讲到，实例也会有对应的配置。

8. 重点：核心原则，服务如果没有加入【版本标识】，该插件会识别成正式服务，如果有【版本标识】该插件会识别成灰度服务。
但是【正常请求】，如果访问下层服务没有正式服务，只有灰度服务的时候，不会调用，会报告异常，也就是说，整个调用链条所有服务必须要有正式服务，
当【灰度请求】，如果访问下层服务，下层服务有对应版本的灰度服务，会调用；如果没有对应版本的灰度服务，会调用正式版本服务，这样也就做到了
调用链当中部分【服务灰度发布的需求】
### 这个符合逻辑的，连正式版本都没有，哪来灰度版本呢。


9. 如何发起【灰度请求】呢，前端程序请求的时候，增加一个自定义头 version, 自定义头内容为版本号，这里的版本号会对应上面【第5条】提到
服务版本标识。注意哟，请求头的版本号要和服务的版本标识完全对应上哟，不然无法识别，会调用正式服务去，再啰嗦一句，正式服务不要加版本识，空值也不行


## 开始安装
### 一、网关：gateway，注册中心：consul ，以上面讲的案例为例子。
1.  【服务3】，添加tag
```yaml
spring:
   cloud: 
     consul: 
       discovery:
         tags:
           - version=1.0.1
```
2. 【服务1】，【服务2】 添加tag
```yaml
spring:
   cloud: 
     consul: 
       discovery:
         tags:
           - version=1.0.1
```
3. 【服务1】，【服务2】 maven添加插件
``` 
  <dependency>
    <groupId>com.labidc</groupId>
    <artifactId>labidc-gray-deploy-eureka</artifactId>
    <version>1.0.30</version>
  </dependency>

```
相关负载均衡规则修改配置, 有 WeightedResponseTimeGrayDeployRule, RandomGrayDeployRule, AvailabilityFilteringGrayDeployRule, BestAvailableGrayDeployRule, ZoneAvoidanceGrayDeployRule, RetryGrayDeployRule, RoundRobinGrayDeployRule(默认值)
```yaml
spring:
  gray:
    deploy:
      ribbonRule: BestAvailableGrayDeployRule
```
4. gateway, 由于gateway 基与webflux, 所以使用变通方法解决路由到 【服务1】
``` yaml
spring:
  application:
    name: ${app_name:service-geteway}
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        healthCheckInterval: 15s
        instanceId: ${spring.application.name}:${vcap.application.instance_id:${spring.application.instance_id:${random.value}}}
        tags:
          - 网关路由
        instance-group: 工具
        query-passing: true
        prefer-ip-address: true
    # gateway 自带了tag 可以做分流和限流操作，自行百度
    gateway:
      discovery:
        locator:
          enabled: true
          #route-id-prefix: test_
          #include-expression: metadata['edge'] != ''
          #lower-case-service-id: true
      routes:
      #===========================灰度发布设置主要这里开始
      - id: gray_demo # 灰度请求路由
        # 新版本请求会走这里，带version请求的，全是这条路由
        uri: lb://product-service-demo1-gray # 启动一个最上层灰度服务，当然这里你也可以启动一个正式版本服务
        predicates:
        # 断言到根路径是test_service 走这条路由
        - Path=/test_service/**
        # 断言到请求头有version 走这条路由
        - Header=version, (.*?)
        filters:
          # 转发给服务的时候取消掉test_service路径
          - StripPrefix=1
          # 路由熔断器
          - name: Hystrix
            args:
              name: fallbackcmd
              fallbackUri: forward:/fallback
      - id: prod_demo # 正常请求路由
        # 正式版本，没有version请求头的，全是正常请求
        uri: lb://product-service-demo1  # 正式服务
        predicates:
        - Path=/test_service/**
        filters:
          - StripPrefix=1
          - name: Hystrix
            args:
              name: fallbackcmd
              fallbackUri: forward:/fallback
      #===========================灰度发布设置主要这里结束
```


### 二、网关：zuul2，注册中心：eureka ，以上面讲的案例为例子。
1.  【服务3】，添加元数据map
```yaml
eureka:
  instance:
    metadata-map:
      version: 1.0.0
```
2. 【服务1】，【服务2】 添加元数据map
```yaml
eureka:
  instance:
    metadata-map:
      version: 1.0.0
```
3. 【服务1】，【服务2】 maven添加插件
``` 
  <dependency>
      <groupId>com.labidc</groupId>
      <artifactId>labidc-gray-deploy-eureka</artifactId>
      <version>1.0.1</version>
  </dependency>

```
相关负载均衡规则修改配置, 有 WeightedResponseTimeGrayDeployRule, RandomGrayDeployRule, AvailabilityFilteringGrayDeployRule, BestAvailableGrayDeployRule, ZoneAvoidanceGrayDeployRule, RetryGrayDeployRule, RoundRobinGrayDeployRule(默认值)
```yaml
spring:
  gray:
    deploy:
      ribbonRule: BestAvailableGrayDeployRule
```

4. zuul2 
``` yaml
# 路由规则配置
zuul:
  #忽略所有默认服务配置
  ignored-services: "*"
  routes:
    product-provider:
      # 这个设置非常关键，要手动设置敏感的请求header, 否则所有请求头都会被过滤，这样设置
      # 主要是为了放行version请求头
      sensitive-headers: Cookie,Set-Cookie,Authorization
      # 路由请求前缀 gray_service
      path: /gray_service/**
      # 取消掉gray_service 访问到后端服务
      stripPrefix: true
      serviceId: product-service-demo1 
```
添加maven 配置
``` 
  <dependency>
      <groupId>com.labidc</groupId>
      <artifactId>labidc-gray-deploy-eureka</artifactId>
      <version>1.0.1</version>
  </dependency>

```


## [DEMO项目地址：https://github.com/labidc/labidc-gray-deploy-demo](https://github.com/labidc/labidc-gray-deploy-demo).