# 怎么修改一个类中的private修饰的String参数的值

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/O8kDwXyV0iu3xvkfbePcIQ09nP8

#### 面试回答

面试官： 怎么修改一个类中 private 修饰的 String 参数的值？

你：
必须用反射，而且要懂 String 不可变和 final 修饰符的坑。private 只是编译期检查，反射能绕过；String 对象本身的内容没法改，但我们可以让字段指向一个新的 String 对象。

分三种情况说：

---

情况一：只 private，没有 final

这就是最常规的反射操作，三步走：

1. 拿到目标类的 Class 对象。
1. 用 getDeclaredField("字段名") 获取字段。
1. 调用 field.setAccessible(true) 暴力打开权限。
1. 用 field.set(对象实例, 新字符串) 改值。

```
代码块
User user = new User();
Field field = User.class.getDeclaredField("name");
field.setAccessible(true);
field.set(user, "李四"); // 原本是"张三"，现在引用指向"李四"
```

情况二：private 且 final

final 修饰的字段在 Java 语言层面禁止修改，JVM 也会做内联优化，但反射依然可以强行突破，只是要多处理一步——把字段的 final 修饰符从底层 Field.modifiers 里抠掉。

```
代码块
Field field = User.class.getDeclaredField("name");
field.setAccessible(true);
// 关键：移除 final 修饰符
Field modifiersField = Field.class.getDeclaredField("modifiers");
modifiersField.setAccessible(true);
modifiersField.setInt(field, field.getModifiers() & ~Modifier.FINAL);
field.set(user, "李四"); // 成功
```

需要特别注意的是，这种方法在 JDK 12+ 可能会被模块系统或安全策略拦截，如果模块没有 opens 相关包给调用方，会直接抛 InaccessibleObjectException。另外，对于 static final 的编译期常量（如 private static final String = "abc"），编译器会在编译时直接把值内联到使用处，你改了字段引用后，那些已经编译好的代码可能还是用的旧值，这是个深层坑。

情况三：JDK 16+ 的强封装

如果你的应用跑在 JDK 16 以上，且没有加 --add-opens java.base/java.lang.reflect=ALL-UNNAMED 这样的 JVM 参数，反射修改 Field.modifiers 会被直接拒绝，setAccessible 也可能被限制。这时更稳妥的做法是用 VarHandle（JDK 9+）或 Unsafe，或者从根本上承认 final 字段不应该被修改，换个设计。

---

为什么 String 本身的内容不能改？

String 内部用 private final byte[] value 存字符，没有提供任何修改这个数组的方法。你通过反射拿到 value 引用也没用，因为它是 final，而且数组内容可以改但 JDK 根本没给你合法路径，改数组内容会导致 hashCode 不一致、常量池错乱等严重问题。所以所谓的“修改 String 参数”，实际上是让原字段引用指向一个新创建的字符串对象。

---

时序图：反射修改 private 字段的完整过程

![image](../../../../assets/FqGKdwaHKobx0ZxlwnAcMv6Hnpb/W8C0b5CTpo1Rf7xu8BzcRzEZnFg.png)

一句话： private 是纸老虎，反射打洞改引用；final 是铁老虎，得先拔牙（去掉 final 修饰符），但拔牙这事在 JDK 高版本越来越难做。非必要，别这么干。
