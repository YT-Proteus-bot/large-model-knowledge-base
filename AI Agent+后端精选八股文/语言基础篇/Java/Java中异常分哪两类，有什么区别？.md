# Java中异常分哪两类，有什么区别？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/Vq2pwQdOnivFhskZ335cxuFTnKZ

“Java 的异常体系根节点是 Throwable，往下分三个子类：Error、Exception 和 RuntimeException。但从编译器是否强制处理的角度，我们通常把异常分成受检异常（Checked Exception）和非受检异常（Unchecked Exception）。这个分类直接决定了代码怎么写、编译能不能过。

一、受检异常：编译器逼着你处理的异常

凡是继承自 Exception 但不继承 RuntimeException 的类，都属于受检异常，比如 IOException、SQLException、ClassNotFoundException。

这类异常的特点是：调用方可以预见，并且通常有能力从错误中恢复。比如读取文件时文件可能不存在，连接网络时对方可能断连，这些都是程序外部的条件问题，不是代码的 bug。

编译器对受检异常非常严格：如果方法里抛出了受检异常，调用者必须要么用 try-catch 捕获处理，要么在方法签名上用 throws 继续向上声明。不处理，编译直接报错。这是 Java 在设计上强制程序员考虑异常情况、提高程序健壮性的手段。

二、非受检异常：编译器完全不干预的异常

非受检异常包括两类：RuntimeException 及其子类（如 NullPointerException、ArrayIndexOutOfBoundsException），以及 Error 及其子类（如 OutOfMemoryError、StackOverflowError）。

这类异常的特点是：它们通常代表程序的 bug 或者系统级灾难，调用者一般无法在运行时优雅恢复。空指针了，你能在 catch 里把它修好吗？基本不能，只能改代码。OOM 了，catch 也基本无力回天。

所以编译器对非受检异常完全不管——你可以在代码里随意抛出 NullPointerException，不声明不捕获，编译照样通过。这样的设计也避免了到处写 throws RuntimeException 这种无意义的噪音。

三、核心区别对比

> （此处为内嵌电子表格，需登录飞书查看）

四、设计意图：该不该强迫人处理？

Java 语言的设计者认为，有些异常程序员是可以提前想到的，比如文件可能没权限，网络可能断开。强迫处理可以让程序更健壮。但对于空指针这种到处都可能发生的错误，如果每次都强制 try-catch，代码会变得臃肿不堪，而且绝大多数 catch 也做不了什么有意义的恢复。所以把它们划为非受检，不强制，反而更务实。

五、实际开发中的应用

我一般遇到受检异常时，会判断当前方法是否有能力处理。如果没能力，就往上抛，在最外层统一处理（比如全局异常拦截器）。对于非受检异常，除非有特定的恢复策略，否则不主动 catch。也不要为了偷懒把受检异常包成非受检异常往外扔，那样会丢掉异常的语义，让调用方不知道该怎么处理。
