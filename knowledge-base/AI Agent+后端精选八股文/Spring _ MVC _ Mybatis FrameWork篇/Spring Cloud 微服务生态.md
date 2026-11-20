---
title: Spring Cloud 微服务生态
topic: AI Agent+后端精选八股文
sub_topic: Spring _ MVC _ Mybatis FrameWork篇
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/CDpWwiHlFibwCokmhxDcZEhbnQb
source_path: AI Agent+后端精选八股文/Spring _ MVC _ Mybatis FrameWork篇/Spring Cloud 微服务生态.md
questions: 8
chars: 2168
build_date: '2026-11-20'
images_removed: 0
---

# Spring Cloud 微服务生态

> 主题：AI Agent+后端精选八股文 › Spring _ MVC _ Mybatis FrameWork篇　｜　原始文件：AI Agent+后端精选八股文/Spring _ MVC _ Mybatis FrameWork篇/Spring Cloud 微服务生态.md

## 题1：Spring Cloud Gateway 的工作原理是什么？

## 题2：1 从请求到转发的四个核心组件

Spring Cloud Gateway 是基于 WebFlux + Netty 的反应式网关，它的整个处理过程可以抽象成一个管道，有三个关键角色：

> 〔原文此处为飞书内嵌组件，内容未随导出〕

一个请求经过网关时，会先遍历所有 Route，找到第一个匹配的 Predicate 集合，然后执行该路由上配置的 Filter 链，最终通过 Netty 客户端转发到下游服务。

## 题3：2 底层执行流程（WebFlux 非阻塞模型）

```
Client Request → HttpWebHandlerAdapter
→ RoutePredicateHandlerMapping（匹配路由）
→ FilteringWebHandler（组装过滤器链）
→ NettyRoutingFilter（发起代理请求）
→ 响应反向经过 Filter 链 → 返回客户端
```

整个过程中 没有阻塞线程，完全基于 Reactor 的事件循环。每个请求在 Netty 连接池中获取一个客户端连接，通过 WebClient 发出异步 HTTP 调用。

## 题4：3 扩展点和常见用法

- 自定义全局过滤器：实现 GlobalFilter 接口，可以全局控制鉴权、日志、流量染色。
- 路由动态刷新：通过 GatewayControllerEndpoint 或集成配置中心（Nacos/Apollo）动态修改路由，不用重启。

它和 Zuul 1.x 最大的区别是异步非阻塞和函数式编程，从而在面对高并发时消耗更少的线程资源。

## 题5：Spring Cloud Feign 的底层原理是什么？如何实现负载均衡和重试？

## 题6：1 Feign 的本质：动态代理 + 契约解析

Feign 本身是一个 声明式 HTTP 客户端，它通过 JDK 动态代理，把接口方法调用转换成 HTTP 请求。

```
@FeignClient(name = "order-service", configuration = FeignConfig.class)
public interface OrderClient {
    @GetMapping("/orders/{id}")
    Order getOrder(@PathVariable Long id);
}
```

当调用 getOrder(123) 时，背后发生了什么？

- Feign 在运行时为 OrderClient 生成代理对象。
- 通过 Contract（默认是 Spring MVC 的 SpringMvcContract）解析方法上的注解，生成 RequestTemplate。
- 代理方法拦截后，将参数值填充到 RequestTemplate，交给 Client 组件执行 HTTP 请求。
- Spring Cloud Feign 使用 FeignBlockingLoadBalancerClient 或早期的 LoadBalancerFeignClient 替换默认的 HTTP 客户端，集成了负载均衡。

## 题7：2 负载均衡集成

Spring Cloud Feign 依赖 Spring Cloud LoadBalancer（或 Ribbon）实现客户端负载均衡。过程：

1. 解析 @FeignClient(name = "order-service") 中的服务名。
1. 通过服务发现组件（Nacos/Consul）获取该服务名对应的所有实例。
1. 内置的 ReactorServiceInstanceLoadBalancer 根据策略（轮询/随机/权重）选出一个实例。
1. 将 RequestTemplate 中的目标 URL 替换为选中的真实地址，发起请求。

如果你想替换负载均衡算法，只需要自定义一个 ServiceInstanceListSupplier 或 LoadBalancer Bean。

## 题8：3 重试机制如何实现

Feign 自带了重试功能，但需要显式开启：

- Feign 内置重试器：在 Feign.Builder 中配置 Retryer.Default，可以设置最大重试次数、初始间隔等。不过这只适用于幂等方法，GET 请求默认是安全的。
- 配合 Spring Retry：在配置中增加 spring.cloud.loadbalancer.retry.enabled=true，并提供一个 RetryTemplate Bean。这样底层 LoadBalancer 在发现连接失败时会自动重试下一个实例，起到“故障转移”的效果。

```
spring:
  cloud:
    loadbalancer:
      retry:
        enabled: true
feign:
  client:
    config:
      default:
        retryer: feign.Retryer.Default
```
