# Agent 框架冷启动慢，怀疑是 torch、sentence-transformers 导入耗时，怎么优化？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/FIO1wdJ5oiGMJ1k3TCFcnDrdnNc

Agent 框架冷启动慢，尤其拖在 import torch 和 sentence-transformers 上，这很常见。
核心矛盾：框架启动时一次性加载了很多重量级依赖，其实大部分功能并非入口所需。
优化思路就是四个字：按需加载，延迟绑定。

---

#### 🧊 分析启动耗时的大头

- import torch 会初始化 CUDA 环境、加载动态库、检测 GPU，哪怕你暂时不用，这一下就 2~5 秒没了。
- import sentence_transformers 内部会触发 transformers、tokenizers 等库的导入，有时还会检查模型缓存、甚至在线校验，哪怕你还没加载模型，已经吃掉好几秒。

---

#### 🛠️ 实战优化策略

##### 1️⃣ 延迟导入：把 import 塞进函数里（最直接有效）

不要在所有 import 都放在文件顶部，只在真正需要调用模型的地方才 import。

```
代码块

# ❌ 旧写法：框架一启动就导入
import torch
from sentence_transformers import SentenceTransformer
class EmbeddingTool:
    def __init__(self):
        self.model = SentenceTransformer('all-MiniLM-L6-v2')

# ✅ 懒加载写法
class EmbeddingTool:
    _model = None
    def embed(self, texts):
        # 第一次调用时才真正导入和加载
        import torch
        from sentence_transformers import SentenceTransformer
        if self._model is None:
            self._model = SentenceTransformer('all-MiniLM-L6-v2')
        return self._model.encode(texts)
```

这样，框架启动时完全不会碰这两个库，直到第一个向量化请求到来才加载。

代价：第一次调用会有“预热”延时，可以配合下文的热身机制。

---

##### 2️⃣ 使用轻量级替代或按需降级

如果不需要完整的 Transformer 模型，可以考虑：

- text2vec-base 等基于 ONNX 的小模型，直接通过 onnxruntime 加载，导入极快且无需 torch。
- 用 sentencepiece 等轻量分词器 + 提前量化好的模型。

但通常 Agent 框架已绑定 sentence-transformers，我们可以保留 API 但内部做懒加载，不改接口。

---

##### 3️⃣ 启动后异步预热

框架启动完成后再慢慢加载模型，避免用户等待。

```
代码块
import asyncio
class AgentApp:
    def __init__(self):
        self.embedding_tool = None
    async def startup(self):
        # 先让服务就绪，再后台预热模型
        asyncio.create_task(self._warm_up())
    async def _warm_up(self):
        # 初始化嵌入工具，触发导入
        self.embedding_tool = EmbeddingTool()
        # 可以跑一个假请求，强制加载到显存
        await self.loop.run_in_executor(None, self.embedding_tool.embed, ["warmup"])
```
