---
title: finally中代码一定会执行吗？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/EUzfwPHSViLrjqk2ulEc6ZM1nCf
source_path: AI Agent+后端精选八股文/语言基础篇/Java/finally中代码一定会执行吗？.md
questions: 1
chars: 645
build_date: '2026-11-20'
images_removed: 0
---

# finally中代码一定会执行吗？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/finally中代码一定会执行吗？.md

“大部分情况下会，但不能说‘一定’。我早期写过一段代码，在 try 里调了 os._exit()，结果 finally 里的清理全部跳过，内存泄漏查了半天。

🛡️ 一句话：finally 保证在离开 try 块时执行，但如果进程直接‘暴毙’，神仙也救不了它。”

### 会执行的情况（99% 的场景）

> 〔原文此处为飞书内嵌组件，内容未随导出〕

小验证：

```
try:
    sys.exit(1)
finally:
    print("还是跑到了")   # ✅ 会输出
```

### 不会执行的情况（进程“物理消失”）

> 〔原文此处为飞书内嵌组件，内容未随导出〕

## 题1：快速分辨：会不会执行？

```
import os
try:
    os._exit(0)          # ❌ 进程直接死，没有任何 finally
finally:
    print("不会执行到这里")
```

```
try:
    raise SystemExit    # ✅ 这是异常，finally 会执行
finally:
    print("会执行")
```

### 面试用一句话总结

🔧 “finally 保的是‘正常的离开路径’——异常、返回、跳出，它都能截住；

但面对 os._exit() 和 kill -9 这种‘暴力拆迁’，它一点办法都没有。

写关键清理逻辑，别只靠 finally，结合 atexit 和信号处理才稳。”
