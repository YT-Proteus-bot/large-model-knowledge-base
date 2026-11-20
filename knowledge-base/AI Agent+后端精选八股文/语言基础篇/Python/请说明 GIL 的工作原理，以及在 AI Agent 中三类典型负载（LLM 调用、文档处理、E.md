---
title: 请说明 GIL 的工作原理，以及在 AI Agent 中三类典型负载（LLM 调用、文档处理、Embedding 计算）应该如何选择并发方案？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/Mzi1wYMKJi0uvck97PNcAzcrn7f
source_path: AI Agent+后端精选八股文/语言基础篇/Python/请说明 GIL 的工作原理，以及在 AI Agent 中三类典型负载（LLM 调用、文档处理、E.md
questions: 0
chars: 1878
build_date: '2026-11-20'
images_removed: 1
---

# 请说明 GIL 的工作原理，以及在 AI Agent 中三类典型负载（LLM 调用、文档处理、Embedding 计算）应该如何选择并发方案？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/请说明 GIL 的工作原理，以及在 AI Agent 中三类典型负载（LLM 调用、文档处理、E.md

好的，这个问题其实正好把 Python 并发里最让人头疼的 GIL 和真实 AI Agent 的场景串起来了。我按“原理 → 分场景选方案”这条线来聊，中间会画个简单的结构图。

## GIL 到底怎么工作

GIL（全局解释器锁）可以看作 CPython 解释器里的一把互斥锁。它的存在主要是为了保护 Python 对象引用计数和内存管理，因为这些操作不是线程安全的。

- 任何一个线程，想执行 Python 字节码，必须先拿到 GIL。
- 它每执行一小段时间（比如 sys.getswitchinterval() 默认 5ms）或遇到 IO 时，就会释放 GIL，让其他线程有机会抢。
- 所以：多线程跑纯 Python 计算时，看着是并行的，实际在 CPU 层面是串行的，而且频繁切换还带来额外开销。

这个特性直接决定了我们在 Agent 里怎么选并发方案，说白了就是看任务是计算密集还是 IO 密集，以及底层库会不会主动释放 GIL。

## 三类负载的并发方案拆解

我把三种负载的特点和方案画成一张分发图，更直观：

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

我分别细说一下为什么这么配。

## 1️⃣ LLM 调用：毫无疑问用 asyncio

调 OpenAI、Claude 这些 API，本质就是发一个 HTTP 请求，然后干等网络响应。等待时 GIL 会被释放，线程或协程都不占 CPU。这场景下，asyncio 最合适：

- 一个事件循环就能管理成百上千个连接，内存开销极小。
- 配合 aiohttp 或 httpx.AsyncClient，用 asyncio.gather 轻松并发十几个 Agent 的请求。
- 比线程池省掉上下文切换和线程栈开销，在大量长连接等待时优势明显。

实际写的时候，我会把所有 LLM 调用封装成一个 async def chat_completion(...)，里头用 Semaphore 控制窗口，外面随便 gather。

## 2️⃣ 文档处理：看情况，保守用多进程

文档处理很暧昧，它可能是：

- IO 部分：读 PDF、Word 文件，用异步 IO 没问题。
- CPU 部分：解析布局、提取文字、正则清洗、Pandas 做表格处理，这些是实打实的 Python 计算，长时间占着 GIL。

如果你直接 pymupdf.open(...) 之类的库，很多 C 实现的库在内部会释放 GIL（比如 fitz 处理页面渲染时）。这种情况下用线程池就行，性能很好。但如果你拿不准第三方库是否释放 GIL，或者用纯 Python 做复杂清洗，那直接用 ProcessPoolExecutor 最保险：

- 把每个文档的处理任务扔到一个独立进程里，GIL 不打架，多核真正并行。
- 缺点：进程启动开销大，数据传参要序列化。对于批量离线文档处理，这不是问题；实时在线场景可以考虑预先 fork 一个常驻进程池。

我一般会先小流量用线程池压测，看 CPU 是不是有效并行，不行再切多进程。

## 3️⃣ Embedding 计算：先分本地还是远程

情况 A：调用远程 Embedding API（如 OpenAI Ada）

跟 LLM 调用一模一样，纯 IO，走 asyncio，和 LLM 放同一个事件循环里一起调度。

情况 B：本地跑模型（sentence-transformers、transformers）

这是最有迷惑性的地方。很多人以为 Python 多线程不行，得用多进程。但实际上 PyTorch/TensorFlow 在推理时，底层 C++/CUDA 计算会释放 GIL。所以用 ThreadPoolExecutor 就能实现多路并行推理，而且可以共享模型实例（注意模型本身的线程安全性，一般 encode 可以并发）。

- 我会把模型加载到内存，然后定义一个 def embed(texts) 的同步函数，内部调模型。
- 在 asyncio 协程里通过 loop.run_in_executor(thread_pool, embed, batch) 把任务丢到线程池，这样主事件循环不阻塞，又能并行使用多个 CPU 核（或 GPU 流）。
- 如果 GPU 推理，还可以配合 cuda stream 进一步并行，线程池数量一般等于 CPU核数*2 或根据 GPU 并发量调。

用多进程反而要复制几份模型，显存或内存爆炸，不划算。
