---
title: 如何用动态代理透明地为 Agent 所有工具调用加上日志和耗时监控？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/MPBLwegbEiVXh7kJSmIcchQDn1f
source_path: AI Agent+后端精选八股文/语言基础篇/Python/如何用动态代理透明地为 Agent 所有工具调用加上日志和耗时监控？.md
questions: 0
chars: 2423
build_date: '2026-11-20'
images_removed: 0
---

# 如何用动态代理透明地为 Agent 所有工具调用加上日志和耗时监控？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/如何用动态代理透明地为 Agent 所有工具调用加上日志和耗时监控？.md

## 问题核心

如何在不修改任何工具源码的前提下，为 Agent 的所有工具调用统一加上日志和耗时监控？

答案很明确：基于接口的动态代理 + 统一的 InvocationHandler，在方法调用前后织入计时和日志逻辑，再把代理对象悄悄替换掉原来的工具实例，对 Agent 完全透明。

## 关键设计：让增强“零侵入”

想要透明，必须具备两个条件：

1. 统一的工具接口 – 所有工具实现同一个接口（例如 Tool，方法 execute）。
1. 代理对象与真实对象类型一致 – Agent 持有的是接口引用，分不清代理还是真实对象。

JDK 动态代理正好满足：Proxy.newProxyInstance 创建一个实现相同接口的代理实例，调用任何方法都会经过 InvocationHandler，我们在这里植入日志和耗时计算。

## 落地实现（面试可展示的核心代码）

① 工具接口

```
public interface Tool {
    String execute(String input);
}
```

② 日志&耗时代理处理器

```
public class ToolLogTimingProxy implements InvocationHandler {
    private final Tool target;
    public ToolLogTimingProxy(Tool target) {
        this.target = target;
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            // 调用真实工具
            return method.invoke(target, args);
        } finally {
            long cost = System.currentTimeMillis() - start;
            System.out.printf("[Tool:%s] method=%s args=%s cost=%dms%n",
                target.getClass().getSimpleName(),
                method.getName(),
                Arrays.toString(args),
                cost);
        }
    }
}
```

③ 透明织入 Agent

```
如何用动态代理透明地为 Agent 所有工具调用加上日志和耗时监控？
答案很明确：基于接口的动态代理 + 统一的 InvocationHandler，在方法调用前后织入计时和日志逻辑，再把代理对象悄悄替换掉原来的工具实例，对 Agent 完全透明。
⚙️ 关键设计：让增强“零侵入”
想要透明，必须具备两个条件：
统一的工具接口 – 所有工具实现同一个接口（例如 Tool，方法 execute）。
代理对象与真实对象类型一致 – Agent 持有的是接口引用，分不清代理还是真实对象。
JDK 动态代理正好满足：Proxy.newProxyInstance 创建一个实现相同接口的代理实例，调用任何方法都会经过 InvocationHandler，我们在这里植入日志和耗时计算。
💻 落地实现（面试可展示的核心代码）
① 工具接口
public interface Tool {
    String execute(String input);
}
② 日志&耗时代理处理器
public class ToolLogTimingProxy implements InvocationHandler {
    private final Tool target;
    public ToolLogTimingProxy(Tool target) {
        this.target = target;
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            // 调用真实工具
            return method.invoke(target, args);
        } finally {
            long cost = System.currentTimeMillis() - start;
            System.out.printf("[Tool:%s] method=%s args=%s cost=%dms%n",
                target.getClass().getSimpleName(),
                method.getName(),
                Arrays.toString(args),
                cost);
        }
    }
}
③ 透明织入 Agent
```
