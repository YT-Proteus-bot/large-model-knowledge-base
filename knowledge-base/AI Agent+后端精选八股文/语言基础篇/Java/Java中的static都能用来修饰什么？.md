---
title: Java中的static都能用来修饰什么？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/LuYtw7VMcieiGTkKFXZcblw4neh
source_path: AI Agent+后端精选八股文/语言基础篇/Java/Java中的static都能用来修饰什么？.md
questions: 0
chars: 1322
build_date: '2026-11-20'
images_removed: 0
---

# Java中的static都能用来修饰什么？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/Java中的static都能用来修饰什么？.md

☕ Java 中的 static 能修饰什么？

static 是 Java 里的“反对象”修饰符，凡是被它标记的成员，就脱离了具体的对象实例，变成了类级别的共有财产。面试时别只背概念，我会分 5 类讲清楚，并附上容易忽略的细节。

## 1️⃣ 静态变量（类变量）—— 所有实例共享

```
public class User {
    public static int onlineCount = 0; // 类变量，只有一份
}
```

- 特点：内存中只存储一份，被所有实例共享。适合记录全局状态（如在线人数、配置参数）。
- 加载时机：类加载时分配内存并初始化（非 final 的静态变量在 <clinit> 方法里赋值）。
- 访问方式：推荐 类名.变量名，虽然用实例也能访问，但会引发警告。
- 注意：实例变量每个对象一份，静态变量整个类一份，别混用。

## 2️⃣ 静态方法（类方法）—— 不依赖对象即可调用

```
public class MathUtils {
    public static int add(int a, int b) {
        return a + b;
    }
}
// 调用：MathUtils.add(3,5)
```

- 特点：
- 直接通过类名调用，不需要 new 对象。
- 只能访问静态变量和静态方法，不能直接访问实例变量/方法（因为没有 this）。
- 应用：工具类方法（Math.max、Collections.sort）、工厂方法。
- 注意：静态方法不能被重写（但可以在子类中“隐藏”同名方法）。

## 3️⃣ 静态代码块 —— 类加载时执行，只一次

```
public class DBConfig {
    static {
        // 加载驱动、读取配置文件等
        System.out.println("类加载时执行，且只执行一次");
    }
}
```

- 作用：用于初始化复杂的静态变量，或在类加载时做一次性工作（如加载本地库）。
- 执行顺序：如果类中有多个静态块，按代码顺序依次执行。父类的静态块优先于子类。
- 对比实例代码块：实例块在每次创建对象时执行，静态块只执行一次。

## 4️⃣ 静态内部类 —— 不依赖外部类实例

```
public class Outer {
    static class StaticInner {
        void print() { System.out.println("静态内部类"); }
    }
}
// 创建：new Outer.StaticInner();
```

- 特点：
- 可以直接实例化，不需要先创建外部类对象。
- 只能访问外部类的静态成员（因为不持有外部实例引用）。
- 优势：普通非静态内部类会隐式持有外部类引用，容易造成内存泄漏；静态内部类更安全、更独立。
- 典型应用：HashMap 的 Node、建造者模式中的 Builder。

## 5️⃣ 静态导入 —— 直接使用类里的静态成员，不加类名
