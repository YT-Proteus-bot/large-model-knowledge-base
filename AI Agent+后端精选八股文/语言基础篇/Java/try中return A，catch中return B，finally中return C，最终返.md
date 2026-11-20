# try中return A，catch中return B，finally中return C，最终返回值是什么？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/AF94w0MmSiDsnbkBPeWcNlZKnpf

这个问题，其实是在考你对 finally 执行优先级 和 JVM 返回机制 的理解，属于看起来简单、一深究就露馅的经典题。

直接先说结果：

```
代码块
public static int test() {
    try {
        return 1;   // A
    } catch (Exception e) {
        return 2;   // B
    } finally {
        return 3;   // C
    }
}
// 最终返回值：3
```

👉 finally 里的 return C 会把前面 try 和 catch 里的 return 值直接“截胡”。

#### 为什么是 3，而不是 1？

你脑袋里可以想象 JVM 在执行这段代码时的“微操”：

1. 执行 try 块里的代码，遇到了 return 1 —— 这时 JVM 不会立刻返回，而是先记下“我准备返回 1 了”，然后跑去执行 finally 块。
1. 进到 finally 里，结果碰上一个 return 3 —— 这相当于对 JVM 说：“刚才那个 1 不要了，现在就给我返回 3”。
1. 于是 finally 里的 return 直接覆盖了之前任何准备返回的值，方法最终返回 3。

🔹 哪怕 try 里没异常，走的也是这个流程，catch 根本不会触发，但 finally 横插一脚照抢不误。

#### 再说个更容易混淆的变种

如果 finally 里没有 return，只是修改了变量的值呢？

```
代码块
public static int test() {
    int x = 0;
    try {
        return x;      // ① 准备返回 0
    } finally {
        x = 10;        // ② 把 x 改成 10，但不 return
    }
}
// 最终返回值：0
```

⚠️ 这里反而是很多人的翻车点。为什么还是 0？
因为 return x 在字节码层面，是先把 x 的值（基本类型）拷贝到一个临时槽位里，然后不管 finally 里怎么改 x 本身，最终返回的都是那个已经拷贝好的值。
如果 x 是引用类型，改的是对象内部状态，那才会影响返回值（因为引用拷贝的只是地址，指向的还是同一个对象）。

🔹 这题还藏着一个隐蔽考点：finally 到底什么时候不执行？
除了刚才说的 System.exit(0)，还有：守护线程挂掉、JVM 崩溃、死循环卡住。其他情况下，即使 try/catch 里抛了异常，finally 也一定会跑完——这是它存在的意义。

你说这题像不像个“俄罗斯套娃”？你刚觉得自己理解了 return 被覆盖，它又用变量修改给你挖个新坑。面试能答到变量拷贝和引用类型这一层，基本就能和背八股的拉开差距了。如果再补一句“其实在字节码里，每个 return 前面都会被插入 finally 块的一个副本”，面试官大概就知道你是真翻过源码或看过字节码的人。
