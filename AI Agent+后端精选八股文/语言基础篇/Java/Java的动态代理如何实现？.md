# Java的动态代理如何实现？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/Uvx5wHWbJikrlZkWSRfccrBBnec

要理解Java动态代理，我觉得可以先从它解决了什么问题入手。

我们平时的代理模式，往往是先写一个接口，然后写一个代理类去实现它，在代理类里把额外的逻辑（比如日志、事务）和真正的业务对象组合起来。这就是静态代理。但问题是，每代理一个接口就得手写一个类，接口一多，代理类就会爆炸。动态代理就是为了省掉这些重复的代理类，让程序在运行时自动生成。

Java的动态代理主要靠java.lang.reflect包下的两个东西：Proxy类和InvocationHandler接口。实现的核心思路可以拆成三步：

1. 写一个调用处理器
   你要先实现InvocationHandler接口，把横切逻辑写在它的invoke方法里。这个方法会在代理对象的任何方法被调用时触发，你能拿到代理对象、被调用的Method对象以及参数。比如：

```
代码块
public class LogHandler implements InvocationHandler {
    private Object target;
    public LogHandler(Object target) { this.target = target; }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("before...");
        Object result = method.invoke(target, args);
        System.out.println("after...");
        return result;
    }
}
```

用Proxy.newProxyInstance生成代理对象
然后调用Proxy.newProxyInstance，告诉它三样东西：

- 用哪个ClassLoader
- 要代理哪些接口（必须至少一个接口）
- 以及刚才写的InvocationHandler实例。

```
代码块
MyService service = new MyServiceImpl();
MyService proxy = (MyService) Proxy.newProxyInstance(
    service.getClass().getClassLoader(),
    service.getClass().getInterfaces(),
    new LogHandler(service)
);
proxy.doSomething();
```

1. 方法调用自动转发到invoke
   当你调用proxy.doSomething()的时候，并不是执行原始对象的方法，而是进入LogHandler的invoke方法，在里面你可以做前置处理、调用真实方法、做后置处理。这样就把“增强”逻辑和业务逻辑解耦了。

底层原理其实很直接：运行时拼接字节码，当场生成了一个代理类。
JDK内部会调用ProxyGenerator.generateProxyClass，动态拼装出一个形如$Proxy0的类，这个类继承了Proxy并实现了你指定的接口。我们可以反编译看看它大概长这样：

```
代码块
public final class $Proxy0 extends Proxy implements MyService {
    // 内部有一个 Method 类型的静态变量，保存接口方法的引用
    public final void doSomething() {
        super.h.invoke(this, m3, null);  // m3 就是 doSomething 的 Method 对象
    }
}
```

可以看到，每个接口方法的实现，都只是去调InvocationHandler的invoke，并把对应的方法对象传进去。正因为代理类已经继承了Proxy，Java又是单继承，所以JDK动态代理只能代理接口，不能代理普通类。

调用链可以用一个简单的图来表示：

![image](../../../../assets/GSOtdAaDqoH138x1r2pcbHk3ncg/OTOTbyv7JoFQCXxqXEucNLD7n0g.png)
