---
title: Spring MVC 与 WebFlux
topic: AI Agent+后端精选八股文
sub_topic: Spring _ MVC _ Mybatis FrameWork篇
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/EvyJwQ60XiC313kQqbUcyFhEnAh
source_path: AI Agent+后端精选八股文/Spring _ MVC _ Mybatis FrameWork篇/Spring MVC 与 WebFlux.md
questions: 2
chars: 1690
build_date: '2026-11-20'
images_removed: 0
---

# Spring MVC 与 WebFlux

> 主题：AI Agent+后端精选八股文 › Spring _ MVC _ Mybatis FrameWork篇　｜　原始文件：AI Agent+后端精选八股文/Spring _ MVC _ Mybatis FrameWork篇/Spring MVC 与 WebFlux.md

### Spring MVC 请求处理流程

## 题1：Spring MVC 处理一个 HTTP 请求的完整流程

🔹 一句话概括

请求先到前端控制器 DispatcherServlet，然后它调度各种组件，最终返回响应。理解这个流程，就掌握了 Spring MVC 的精髓。

🔹 文字流程图（带图标说明）

```
用户请求
   ↓
① 🎯 DispatcherServlet（前端控制器）
   |  是整个流程的“总指挥”，接收请求并分派。
   ↓
② 🗺️ HandlerMapping（处理器映射器）
   |  根据请求 URL、方法等找到对应的处理器 Handler（即控制器方法）。
   ↓
③ ⚙️ HandlerAdapter（处理器适配器）
   |  负责真正调用 Handler。因为 Handler 有多种形式（@Controller、HttpRequestHandler 等），
   |  通过适配器统一调用，并处理参数绑定、返回值处理等。
   ↓
④ 🔧 Handler（开发者编写的 Controller 方法）
   |  执行业务逻辑，通常会返回一个 ModelAndView（传统）或直接用 @ResponseBody 返回对象。
   ↓
⑤ 🔁 如果有拦截器(Interceptor)的 postHandle，视图渲染前执行。
   ↓
⑥ 🖼️ ViewResolver（视图解析器） / HttpMessageConverter（消息转换器）
   |  如果返回视图名，则 ViewResolver 解析为具体 View 模板（如 JSP、Thymeleaf）；
   |  如果是 @ResponseBody 或 REST 接口，则通过消息转换器（如 MappingJackson2HttpMessageConverter）把对象转为 JSON。
   ↓
⑦ 📄 视图渲染 / 响应体输出
   |  将模型数据填充到视图，生成 HTML；或者直接输出 JSON/XML 等。
   ↓
⑧ 🧹 拦截器的 afterCompletion 执行（视图渲染后，用于资源清理）。
   ↓
响应返回给用户
```

🔹 示例代码：体验 @ResponseBody 如何走消息转换器

```
@RestController
public class UserController {
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        // 模拟查询用户
        User user = new User(id, "老周", "laozhou@example.com");
        return user;  // 因为没有视图名，默认走 @ResponseBody -> JSON
    }
}
```

当请求 /users/1 时，DispatcherServlet 通过 HandlerMapping 找到 getUser 方法，HandlerAdapter 执行方法并得到 User 对象。因为没有返回视图，Spring MVC 会使用 HttpMessageConverter 将 User 转成 JSON 写入响应体。

🔹 面试加分小贴士

可以顺便提一下：在前后端分离时代，ViewResolver 环节经常会被弱化，核心变成了消息转换器。另外，流程中还会穿插拦截器、异常解析器等，体现了 Spring MVC 的可扩展性。

📌 这样梳理下来，是不是比死记硬背清晰多了？下次可以尝试画一画，面试时连画带讲，效果拔群。

## 题2：拦截器（Interceptor）和过滤器（Filter）的区别

很多同学傻傻分不清，本质上是因为它们处于不同层级。我们用一张对比表 + 代码感受一下。

🔹 核心区别对比表

> 〔原文此处为飞书内嵌组件，内容未随导出〕
