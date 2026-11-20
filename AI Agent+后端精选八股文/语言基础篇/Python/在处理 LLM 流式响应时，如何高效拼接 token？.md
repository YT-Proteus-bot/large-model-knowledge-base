# 在处理 LLM 流式响应时，如何高效拼接 token？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/Hwr2w2TlziBvSek2kNUcSfhrn4c

✂️ 处理 LLM 流式 token 时，一个直觉是用 += 来拼。但这是“看起来对，实际上慢”的典型。

---

#### 🧱 直接拼接：每次都在“拆楼重建”

Python 字符串是不可变的，text += token 的本质是：

1. 分配一块新内存，大小 = 原 text 长度 + token 长度
1. 把旧内容复制过去
1. 追加新内容
1. 丢弃旧字符串

随着拼接次数增加，越往后复制的东西越多，总开销是 O(n²)。

```
代码块
token1 + token2 + token3 ...
  └─ 复制1个字符
       └─ 复制前2个字符
            └─ 复制前3个字符
                 └─ ...
```

#### 🚀 高效方式：list 收集 + ''.join

先拿列表把 token 收起来，最后一次性 join：

```
代码块
chunks = []
for token in stream_llm(prompt):
    chunks.append(token)
full_text = ''.join(chunks)
```

- 时间复杂度 O(n)：join 内部会先算总长度，一次分配内存，然后顺序拷贝。没有重复复制。
- 内存友好：列表里只是放了字符串对象的引用，没有额外的字符拷贝。

```
代码块

# ❌ 不要这样
text = ""
for token in stream:
    text += token       # 每次都重新分配内存

# ✅ 应该这样
buffer = []
for token in stream:
    buffer.append(token)
response = ''.join(buffer)
```

#### 📊 一图对比

> （此处为内嵌电子表格，需登录飞书查看）

---

#### 🔹 进阶技巧：边传边拼，不必全存

如果你的场景是流式返回给前端（如 SSE），根本不需要拼出完整字符串，拿到一个 token 就 yield 或 send 出去，连 list 都可以省掉：

```
代码块
for token in llm_stream:
    yield token        # 直接流式传递，零拷贝拼接
```

除非你最后需要完整文本做后处理（比如保存、摘要），才需要收集拼接。即便如此，list + join 也是最高效的选择。
