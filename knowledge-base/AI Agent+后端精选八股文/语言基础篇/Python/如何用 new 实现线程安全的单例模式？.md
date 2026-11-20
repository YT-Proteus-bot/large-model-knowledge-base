---
title: 如何用 new 实现线程安全的单例模式？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/L4o5wiP8iiY4mqkL9CycAtzRnLb
source_path: AI Agent+后端精选八股文/语言基础篇/Python/如何用 new 实现线程安全的单例模式？.md
questions: 0
chars: 1577
build_date: '2026-11-20'
images_removed: 0
---

# 如何用 new 实现线程安全的单例模式？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/如何用 new 实现线程安全的单例模式？.md

单例模式最核心的矛盾就是：又要全局唯一，又要懒加载，还要线程安全。

用 new 来实现的话，常见有三条路，我按推荐程度从高到低说一下。

## 推荐：静态内部类（Holder 模式）

既能懒加载（真正调用时才初始化），又是线程安全（JVM 类加载机制保证），代码还简单。

```
public class Singleton {
    private Singleton() {}   // 构造私有
    private static class Holder {
        // 内部类加载时才执行这个 new，且只执行一次
        private static final Singleton INSTANCE = new Singleton();
    }
    public static Singleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

✅ 原理：Singleton 类加载时，内部类 Holder 不会被加载。只有 getInstance() 第一次被调用，触发 Holder 类的初始化，JVM 会确保这个初始化过程是线程安全的，new Singleton() 只执行一次。

✅ 没有锁，没有 volatile，性能最高。

## 备选：双重检查锁定（DCL）

如果面试官问“怎么用 new 加锁实现”，那这就是经典写法。必须配合 volatile，否则指令重排可能让其他线程拿到“半成品”对象。

```
public class Singleton {
    private static volatile Singleton instance;   // volatile 防止指令重排
    private Singleton() {}
    public static Singleton getInstance() {
        if (instance == null) {                   // 第一次检查，避免不必要的同步
            synchronized (Singleton.class) {
                if (instance == null) {           // 第二次检查，保证只 new 一次
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

⚠️ 为什么必须 volatile？

new Singleton() 实际分三步：

① 分配内存

② 调用构造函数初始化

③ 将引用指向内存地址

如果没有 volatile，②和③可能被重排，导致另一个线程在判断 instance != null 时拿到一个还未初始化完成的对象。volatile 禁止这种重排。

## 最简单粗暴：饿汉式

类一加载就直接 new，天生线程安全，但不懒加载。如果单例创建成本高且不一定会用到，可能浪费资源。

```
public class Singleton {
    private static final Singleton INSTANCE = new Singleton();
    private Singleton() {}
    public static Singleton getInstance() {
        return INSTANCE;
    }
}
```
