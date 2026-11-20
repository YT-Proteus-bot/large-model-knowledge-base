# finally中代码一定会执行吗？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/EUzfwPHSViLrjqk2ulEc6ZM1nCf

“大部分情况下会，但不能说‘一定’。我早期写过一段代码，在 try 里调了 os._exit()，结果 finally 里的清理全部跳过，内存泄漏查了半天。

🛡️ 一句话：finally 保证在离开 try 块时执行，但如果进程直接‘暴毙’，神仙也救不了它。”

---

#### ✅ 会执行的情况（99% 的场景）

> （此处为内嵌电子表格，需登录飞书查看）

小验证：

```
代码块
try:
    sys.exit(1)
finally:
    print("还是跑到了")   # ✅ 会输出
```

#### 💥 不会执行的情况（进程“物理消失”）

> （此处为内嵌电子表格，需登录飞书查看）

---

#### 🧪 快速分辨：会不会执行？

```
代码块
import os
try:
    os._exit(0)          # ❌ 进程直接死，没有任何 finally
finally:
    print("不会执行到这里")
```

```
代码块
try:
    raise SystemExit    # ✅ 这是异常，finally 会执行
finally:
    print("会执行")
```

#### 🎯 面试用一句话总结

🔧 “finally 保的是‘正常的离开路径’——异常、返回、跳出，它都能截住；
但面对 os._exit() 和 kill -9 这种‘暴力拆迁’，它一点办法都没有。
写关键清理逻辑，别只靠 finally，结合 atexit 和信号处理才稳。”
