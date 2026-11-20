# char能存储中文吗？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/QwRCwbpzLi9g3ekQLW5crQUbnIh

#### 面试回答

面试官： Java 中的 char 能存储一个中文汉字吗？

你：
能，但不是所有。一个 char 可以存储大部分常用汉字，但极少数生僻字或特殊汉字一个 char 存不下。

这要从 char 的存储原理说起。Java 的 char 是 16 位无符号整数，范围是 0 ~ 65535，它采用的是 UTF-16 编码。Unicode 把所有字符分成了 17 个平面，其中第 0 平面称为基本多语言平面（BMP），码点范围 U+0000 ~ U+FFFF，正好和 char 的取值范围完美对应。常用的中文字符（包括 GB2312、GBK、常用繁体等）几乎全都在 BMP 内，比如“我”的码点是 U+6211，“爱”是 U+7231，它们都只需要一个 char 就能存储。

但 Unicode 总字符量远不止 65535 个。那些超出 BMP 的字符，比如一些生僻汉字（如“𠀀”，码点 U+20000）、emoji 表情（如“😀”，码点 U+1F600），就需要用两个 char 组成的代理对来表示。这种情况下，一个 char 是存不了一个完整汉字的。

所以结论是： char c = '中'; 完全没问题，char c = '𠀀'; 编译直接报错，因为编译器知道这个字面量超出了 char 的范围，必须用两个 char 或直接用 String 表示。

实际开发中更建议直接用 String。 比如 String s = "𠀀"; 虽然会占用两个 char 的位置，但对调用者完全透明，不用操心底层是不是代理对，避免在字符处理时出现“半个字符”的 bug。

![image](../../../../assets/TC95dvhTWo6qkUxdTEjcbqomnDd/JSoNbGPSmoSybZxTLxQc9elznuh.png)
