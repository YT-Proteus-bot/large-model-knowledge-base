# 为什么对Java中的负数取绝对值结果不一定是正数？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/FDbRwJTWDi6w77kxK4pcrokWnLc

面试回答

面试官： 为什么Java里对负数取绝对值，结果不一定是正数？

你：

这个坑出在整数溢出上。Java里int的范围是-2147483648到2147483647，正数比负数少一个。Integer.MIN_VALUE是-2147483648，你取它的绝对值应该是2147483648，但这个数已经超出了int能表示的最大值，溢出了，转了一圈变回-2147483648，还是负数。

根因在于补码不对称。 计算机用补码存整数，补码表示范围天然不对称，负数比正数多一个。所以绝对值函数Math.abs对绝大多数负数有效，唯独Integer.MIN_VALUE不行。Long同理，Long.MIN_VALUE的绝对值也溢出。

怎么避免？ 调用Math.abs之前先判断是不是Integer.MIN_VALUE，如果是就别直接取绝对值，或者先转成long再取，long范围宽，基本够用。

![image](../../../../assets/Hw17djWWlorTFRxIb00cRA1Snlb/ATeGbqD2Uoak1gxpwzocb676n8e.png)
