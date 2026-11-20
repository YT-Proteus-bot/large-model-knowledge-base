---
title: 怎么修改一个类中的private修饰的String参数的值
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/ZSQOwL9Z7igrGYkud4YckoARnBh
source_path: AI Agent+后端精选八股文/语言基础篇/Java/怎么修改一个类中的private修饰的String参数的值.md
questions: 0
chars: 1730
build_date: '2026-11-20'
images_removed: 0
---

# 怎么修改一个类中的private修饰的String参数的值

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/怎么修改一个类中的private修饰的String参数的值.md

🔓 修改 private 的 String 字段，是不是只能通过反射？

是的，最直接的办法就是反射。private 只是编译期检查，运行时可以通过 Field.setAccessible(true) 暴力访问。如果字段还是 final 的，则需要多一步——把 final 修饰符也“拔掉”。不过随着 JDK 版本的升高，这种操作越来越受限，理解其原理和边界非常重要。

## 场景：类定义与目标

假设有这样一个类：

```
public class User {
    private String name = "张三";
    private final String id = "ID-001";
    // 省略 getter
}
```

我们想在不改源码的前提下，把 name 改为 "李四"，甚至把 final 的 id 也改掉。

## 步骤一：普通 private 字段（非 final）

这种情况下只需要标准的反射三步：

1. 获取目标类的 Class 对象
1. 通过 getDeclaredField("字段名") 拿到 Field
1. 调用 field.setAccessible(true) 关闭访问检查
1. 用 field.set(对象实例, 新值) 修改

代码示例：

```
User user = new User();
Field nameField = User.class.getDeclaredField("name");
nameField.setAccessible(true);
nameField.set(user, "李四");
System.out.println(user.getName()); // 输出：李四
```

✅ 原理：setAccessible(true) 会抑制 Java 语言访问控制检查，让反射可以绕过 private。

## 步骤二：如果字段是 final 的

final 修饰的字段在编译后可能被内联，运行时的修改不仅要突破 private，还要突破 final 的限制。Java 的 Field 对象内部用一个 modifiers 整型来表示字段的修饰符（public/private/static/final 等），我们可以把这个整型值里的 final 标志位清除掉。

做法：

1. 拿到 Field 对象，setAccessible(true)
1. 获取 Field 类中的 modifiers 字段（它是 java.lang.reflect.Field 的私有成员）
1. 把这个 modifiers 的值改成 field.getModifiers() & ~Modifier.FINAL
1. 再执行 field.set(实例, 新值)

代码示例：

```
User user = new User();
Field idField = User.class.getDeclaredField("id");
idField.setAccessible(true);
// 去掉 final 修饰符
Field modifiersField = Field.class.getDeclaredField("modifiers");
modifiersField.setAccessible(true);
modifiersField.setInt(idField, idField.getModifiers() & ~Modifier.FINAL);
idField.set(user, "ID-999");
System.out.println(user.getId()); // 输出：ID-999
```

⚠️ 注意：这种修改只在当前 JVM 实例中有效，且对于 static final 的基本类型或字符串常量，如果被编译时常量折叠，修改可能不会反映到所有使用点。

## 高版本 JDK 的障碍（JDK 12+）

从 JDK 12 开始，Java 加强了对内部 API 的封装。如果你直接运行上面的代码，可能会遇到：
