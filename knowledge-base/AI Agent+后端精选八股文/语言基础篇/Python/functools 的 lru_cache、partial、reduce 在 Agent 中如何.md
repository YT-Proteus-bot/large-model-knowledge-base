---
title: functools 的 lru_cache、partial、reduce 在 Agent 中如何应用，各自有什么限制？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/BeCIwVg4oizFU5kMxFEc0maBnOi
source_path: AI Agent+后端精选八股文/语言基础篇/Python/functools 的 lru_cache、partial、reduce 在 Agent 中如何.md
questions: 0
chars: 1920
build_date: '2026-11-20'
images_removed: 0
---

# functools 的 lru_cache、partial、reduce 在 Agent 中如何应用，各自有什么限制？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/functools 的 lru_cache、partial、reduce 在 Agent 中如何.md

面试中如果问到 functools 里这几个工具在 Agent 中的应用，面试官通常不是考 API 怎么用，而是想看你如何用函数式思维降低系统复杂度。我会把这三个分别对应到 Agent 的三个典型痛点：重复计算、参数膨胀、流式聚合。

## lru_cache —— 给 Agent 的记忆加个“缓存条”

本质：记住函数调用的结果，相同输入直接返回缓存，不用再算。

在 Agent 中的典型用法

- 缓存工具调用结果：比如天气查询、汇率转换，短时间内相同参数的结果是稳定的。Agent 反复调用时，直接命中缓存，既快又省 API 配额。
- 缓存 LLM 响应：如果某些 prompt 是固定的模板+参数，可对组装后的请求做缓存。但要注意，LLM 本身有随机性，通常只缓存 temperature=0 的确定性调用。

```
from functools import lru_cache
@lru_cache(maxsize=128)
def get_weather(city: str, date: str):
    return weather_api.fetch(city, date)
```

⚠️ 限制与坑

> 〔原文此处为飞书内嵌组件，内容未随导出〕

## partial —— 冻结高频参数，让 Agent 的指令更简洁

本质：预先填好函数的一部分参数，生成一个新函数，调用时只需补上剩余参数。

在 Agent 中的典型用法

- 预设工具默认值：比如公司内部的搜索工具总是搜内网、用特定语言，每次调都传一样参数很繁琐。partial 把底细藏起来，给 Agent 暴露一个简洁接口。
- 统一回调接口：Agent 各模块之间需要回调，但不同模块的签名不同。可以用 partial 把上下文（如 session_id）提前绑死，让接口统一为 callback(result)。
- 为 LLM 函数调用瘦身：OpenAI function calling 定义的参数有时很灵活，通过 partial 可以生成“特化版”函数，减少 Agent 决策时选错参数的概率。

```
from functools import partial

# 原始工具：search(query, source, lang)
internal_search = partial(search, source="internal_wiki", lang="zh")

# 现在 Agent 只需调用 internal_search(query)
```

⚠️ 限制与坑

- 过早求值：partial 绑定的参数在定义时就求值了，如果绑了一个可变对象（比如一个列表），后来它变了，partial 里还是旧的。解决方案是绑不可变数据或对象引用。
- 可读性下降：连续几个 partial 嵌套后，debug 时看到函数叫 partial(func, ...) 会丧失签名信息。可以在定义后手动修改 name 或使用 functools.wraps。
- 参数覆盖风险：调用时如果又传了已固定的参数，会抛出 TypeError: got multiple values，在动态 Agent 拼装参数时容易踩。

## reduce —— 把一连串工具输出揉成一个结果

本质：对一个序列反复应用一个二元函数，最终聚合成一个值。

在 Agent 中的典型用法

- 多路搜索合并：Agent 同时调了 3 个搜索引擎，返回了 3 段文本。用 reduce + operator.concat 或自定义合并函数，把结果串成一段完整的上下文喂给 LLM。
- 置信度累积：多个分析模块各自给出一个可信度分数（0~1），用 reduce 相乘或取平均，得到整体置信度，决定是否需要人工介入。
- 管道式数据处理：把清洗、过滤、排序等操作串成一条链，reduce 搭配 partial 可以写出非常紧凑的数据流。

```
from functools import reduce
import operator
results = [
    "北京今日晴朗，25度。",
    "北京今天多云，26度。",
    "北京当前温度24度。"
]
merged = reduce(operator.add, results)

# 或者更复杂的聚合：
def merge_two(a, b):
    return f"{a}\n---\n{b}"
context = reduce(merge_two, results)
```
