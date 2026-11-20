# String a = "ab"; String b = "a" + "b"; a == b 吗？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/XIIOwC9HgiRflekQKyCcuwM7nrb

#### 面试回答

面试官： String a = "ab"; String b = "a" + "b"; a == b 吗？

你：
是 true。 因为编译期会做常量折叠，"a" + "b" 直接优化成 "ab"，两个引用指向常量池里同一个字符串对象。

![image](../../../../assets/HOdwdIdTYobrSaxzIpRcjSh7njh/J057b7woIoycr2xD0VDcwEwlngc.png)
