---
title: PyTorch 知识库：从入门到工程实践
topic: 教学知识库
sub_topic: PyTorch
doc_type: kb-doc
level: 入门
chapters: 11
chars: 113165
build_date: '2026-11-25'
---

# PyTorch 知识库：从入门到工程实践

从环境安装讲到张量、自动求导、模型构建、数据加载、训练循环，再深入显存优化、分布式训练、保存部署与调试排查，最后给出生态与工程实践清单。

> 适合人群：零基础或刚入门的开发者、需要带新人的工程师；每章独立成节，可按需跳读，也适合整篇灌入 RAG 系统做检索。

## 目录

1. 环境与工具链
2. 张量基础
3. 自动求导
4. nn.Module 与模型构建
5. 数据加载
6. 训练循环与优化
7. 显存与性能优化
8. 分布式训练
9. 保存、加载与部署
10. 调试与常见报错
11. 生态与工程实践

## 1. 环境与工具链

在深度学习项目里，环境问题造成的失败远多于模型问题。同一份训练代码在同事机器上跑得通、到自己机器上报「内核镜像不可用」，原因通常不在代码，而在解释器、轮子与驱动三者的版本没有对齐。让 PyTorch 真正可用，要按顺序做完五件事：隔离 Python 环境、选对安装通道、对齐 CUDA 版本、跑通自检脚本、配好调试器。每一件都给出可直接执行的命令和判断依据。

### 1.1 环境隔离：venv、conda 与 uv 怎么选

虚拟环境（Virtual Environment，一份独立的 Python 解释器副本加独立的第三方包目录，让不同项目互不干扰）是第一个必须建立的习惯。原因很具体：PyTorch 的每个版本都对 Python 版本、numpy 版本以及自身的 CUDA 运行时做过编译期绑定，当两个项目分别要求 `torch==2.1` 和 `torch==2.7` 时，装进同一套 site-packages 必然有一个要坏。而直接往系统 Python 里装包，出问题后连回滚都做不到，只能重装解释器。

三种主流方案的差别可以用一句话概括：venv 是标准库自带的最轻隔离，环境目录就是项目里的一个文件夹，删掉即清理干净，缺点是完全管不了非 Python 的二进制依赖；conda 能连带管理编译器和系统库，代价是解析依赖慢、环境体积通常是 venv 的三到五倍，装好 PyTorch 的环境占五到八 GB 很常见；uv 用 Rust 实现，命令兼容 pip，解析与下载速度通常是 pip 的数倍，适合依赖多、需要反复重建环境的工程，缺点是生态较新，个别冷门包仍要退回 pip。

```bash
python -m venv .venv && source .venv/bin/activate      # Windows 用 .\.venv\Scripts\Activate.ps1 激活
conda create -n pt27 python=3.11 -y && conda activate pt27    # 环境名 pt27，Python 3.11
uv venv --python 3.11 .venv && uv pip install torch --index-url https://download.pytorch.org/whl/cu126
```

三条纪律：一个项目一个环境；把 `.venv/` 写进 `.gitignore` 而不是提交进仓库；动手前用 `python -c "import sys; print(sys.executable)"` 确认当前解释器路径。还有一条容易被忽略：不要在 conda 的 base 环境里装项目依赖，base 一旦装坏，连 `conda` 命令本身都可能失效，恢复成本远高于重建一个子环境。

判断环境是否干净有个笨办法：新建环境后立刻执行 `python -m pip list`，正常只有 pip、setuptools、wheel 寥寥几个包；如果列出几十个，说明激活没生效，命令落在别的环境里。用 conda 的话再补一句 `conda list | grep -i torch` 与 `pip list | grep -i torch`，两处都出现同一个包，就说明这个环境被混装过，后面迟早出问题。团队内部最好统一到同一种方案和同一份激活说明，三个人用三种工具时，最难排查的不是代码问题，而是「我这边是好的」。

### 1.2 安装 PyTorch：先定 CUDA 版本，再选索引

PyTorch 的 pip 包分为 CPU 版与 CUDA 版，包名相同，只靠下载索引区分。默认源给出的是 CPU 版，装完 `torch.cuda.is_available()` 恒为 `False`——这不是装坏了，是装错了通道。另一个常被误解的点：pip 安装的轮子里已经打包了 CUDA 运行时库（libcudart、cuBLAS、cuDNN 等），系统上不需要另外安装 CUDA Toolkit，只需要一个足够新的显卡驱动。CUDA Toolkit 是含 nvcc 编译器与头文件的开发套件，体积好几个 GB，只有在自己编译 CUDA 算子时才需要。

```bash
nvidia-smi                                 # 看驱动能力上限，右上角 "CUDA Version: 12.6" 即上限
python -m pip install --upgrade torch torchvision torchaudio \
  --index-url https://download.pytorch.org/whl/cu126    # cu118 / cu124 / cu126 / cu128 按需替换
```

是否装对，只看一行：`python -c "import torch; print(torch.__version__, torch.version.cuda)"`。`torch.version.cuda` 为 `None` 说明拿到的是 CPU 版，必须重装；正常应输出类似 `2.7.0+cu126 12.6` 的结果。版本后缀也值得留意：带 `+cu126` 说明来自 CUDA 索引，带 `+cpu` 说明是 CPU 轮子，两个来源混着装会在升级时互相覆盖。

版本匹配有两条硬约束。第一，torch、torchvision、torchaudio 必须来自同一次安装，单独升级其中一个会让 `import torchvision` 直接报运算符未定义；第二，Python 版本要落在官方轮子覆盖的范围内，通常 3.9 到 3.13，太新（刚发布的 3.14）与太旧（3.8 以下）都会得到「找不到匹配版本」，工程上建议 3.10 到 3.12。用 `pip config set global.index-url` 可以把镜像地址固化成默认值，但命令行的 `--index-url` 优先级更高，装 CUDA 版时别忘显式写出。

新手最常见的错误是照着某篇教程抄 `--index-url .../cu118`，而自己的卡是 50 系，装完直接报算力不匹配。正确顺序永远是以本机驱动和显卡为准反推版本，教程里的命令只当模板。若一张卡同时服务多个项目，让每个项目钉在各自的轮子版本上，比把全机器统一升到最新更安全——跨大版本的算子实现与数值精度都可能变化，昨天能复现的实验今天未必能。

国内直连官方索引常超时，可行做法有三种：给 pip 加代理（`--proxy` 参数或 `HTTPS_PROXY` 环境变量）；在能联网的同平台机器上用 `pip download` 存下 whl 文件再离线安装；用清华、阿里等国内镜像，但这类镜像同步的通常是 PyPI 上的 CPU 轮子，CUDA 版往往缺失或滞后，只适合装 CPU 环境。安装前还要确认磁盘余量，一个 CUDA 轮子解包后占两到五 GB。换用新的镜像站时，先用一个小包验证连通与证书，再装体积大的 torch，能省掉一次十几分钟的干等。

### 1.3 CUDA、cuDNN 与驱动的对应关系

三个概念容易混淆。显卡驱动（Driver，操作系统访问 GPU 的唯一通道，由 NVIDIA 安装包提供）是唯一必须装在宿主机上的东西；CUDA 运行时（CUDA Runtime，PyTorch 调用 GPU 的底层 API 集合）随轮子打包；cuDNN（CUDA Deep Neural Network library，NVIDIA 的深度神经网络算子加速库，卷积、归一化、Attention 的底层实现）同样随轮子打包，无需单独下载。

| 驱动版本 | 支持的最高 CUDA 运行时 | 建议轮子 | 对应 cuDNN |
| --- | --- | --- | --- |
| 520.x 及以上 | 11.8 | cu118 | 8.7 |
| 525.x 及以上 | 12.1 | cu121 | 8.9 |
| 550.x 及以上 | 12.4 | cu124 | 9.x |
| 560.x 及以上 | 12.6 | cu126 | 9.x |
| 570.x 及以上 | 12.8 | cu128 | 9.x |

兼容规则只有一条：新驱动能跑旧的 CUDA 运行时，旧驱动跑不了新的。所以排查顺序永远是先看驱动、再看轮子。两个高频误判要记住：`nvidia-smi` 右上角的 CUDA Version 是驱动能力上限，不代表机器装了该版本运行时；`nvcc --version` 反映的是本地 Toolkit，与 PyTorch 实际使用的运行时无关，两者不一致完全正常。

CUDA 12.x 内部还有一层次版本兼容：装 12.6 运行时的程序可以在只支持 12.1 的老驱动上跑起来，前提是不使用 12.1 之后新增的 API。这条机制解释了不少「驱动看起来不够但确实能跑」的现象，不过别把它当设计前提，跨大版本（11.x 与 12.x 之间）完全没有这种宽容。

「内核镜像不可用」这类报错的根因是显卡算力（Compute Capability，用主次两个数字描述 GPU 指令集代际的编号）不在轮子的编译目标里。常见对应关系：T4 为 7.5，V100 为 7.0，A100 为 8.0，RTX 30 系为 8.6，RTX 40 系为 8.9，H100 为 9.0，RTX 50 系为 12.0。cu126 的轮子覆盖到 9.0 为止，50 系新卡必须用 cu128 及以上。有人想用 `TORCH_CUDA_ARCH_LIST` 环境变量补救，这个变量只在从源码编译时生效，对官方预编译轮子没有任何作用。顺带一提，同一块卡上 cuDNN 9 相对 8 在卷积类算子上的提升通常有百分之十几，这本身就是升级轮子的一项实在收益。

在容器或 WSL 里，`nvidia-smi` 显示的是宿主机驱动，容器内既不需要也装不了驱动，启动时加 `--gpus all` 即可，可用上限仍由宿主驱动决定。共享服务器上不要随手升级驱动或执行整机升级，驱动是全局资源，一次升级会打断所有人的训练；要动就约维护窗口，或用容器把自己隔离起来。日常记录机器基线用 `nvidia-smi --query-gpu=name,driver_version,memory.total --format=csv`，比截图更便于比对。

独立机器上换驱动的正确做法是先彻底卸载旧版本再装新版本。新旧驱动库混在一起时，`nvidia-smi` 照样能出画面，但 CUDA 初始化会失败，这种故障比版本不匹配更难查，因为所有版本号看起来都对。卸载后重启再装，是唯一稳妥的顺序。

### 1.4 安装自检脚本

自检脚本可以保存为 `check_env.py`，每次新建环境后跑一遍，能在训练开始前暴露绝大多数环境问题。

```python
"""环境自检：python check_env.py"""
import platform, sys, time, torch

print(sys.version.split()[0], platform.platform(), "|", sys.executable)
print("torch", torch.__version__, "cu", torch.version.cuda)   # cu 为 None 即 CPU 版
print("cudnn", torch.backends.cudnn.version(), "GPU 数", torch.cuda.device_count())
if not torch.cuda.is_available():
    raise SystemExit("CUDA 不可用：先查是否 CPU 版，再对照驱动版本表")
for i in range(torch.cuda.device_count()):
    p = torch.cuda.get_device_properties(i)
    print(i, p.name, f"{p.total_memory / 1024**3:.1f}GiB", f"算力{p.major}.{p.minor}")

a = torch.randn(4096, 4096, device="cuda", dtype=torch.float16)
torch.cuda.synchronize(); t0 = time.perf_counter()   # 计时前必须同步，否则测的是下发耗时
for _ in range(20):
    a @ a
torch.cuda.synchronize(); dt = (time.perf_counter() - t0) / 20
print(f"FP16 矩阵乘 {dt * 1e3:.2f}ms 约 {2 * 4096**3 / dt / 1e12:.0f} TFLOPS")

x = torch.randn(64, 3, 224, 224, device="cuda")
net = torch.nn.Sequential(                            # 卷积+BN+反传，确认 cuDNN 链路通
    torch.nn.Conv2d(3, 32, 3, padding=1), torch.nn.BatchNorm2d(32), torch.nn.ReLU(),
    torch.nn.AdaptiveAvgPool2d(1), torch.nn.Flatten(), torch.nn.Linear(32, 10)).cuda()
net(x).sum().backward()
print(f"显存峰值 {torch.cuda.max_memory_allocated() / 1024**2:.0f}MiB")
```

判读有四条。实测 TFLOPS 与显卡规格差一个数量级，说明跑在 PCIe 降速、省电模式或 ECC 受限状态；显存峰值远高于按参数量估算的结果，说明显存里还留着未释放的张量，通常是把带梯度的中间结果存进了列表；「算力」一行与轮子的编译目标不匹配，正是内核镜像报错的直接原因；cudnn 显示 `None` 则说明这个轮子是 CPU 版或未随包分发，卷积会退回慢速实现，性能凭空掉一半。

一个真实案例可以说明脚本的价值：同事的脚本报内核镜像不可用，自检显示运行时 12.1、驱动 550、显卡是算力 8.9 的 RTX 4060 Ti，三项都不老，问题出在他装的是两年前的旧轮子——那批轮子的编译目标最高到 8.6，8.9 的卡只能走 PTX 即时编译，性能打折，个别算子干脆失败，换成本季度发布的轮子后立刻恢复。把每次自检的输出贴进实验记录，换机器、换镜像、升级轮子时都有对照物，出问题能立刻分辨是环境变了还是代码变了。脚本还可以按团队需要扩展，比如加一段 `torch.cuda.mem_get_info()` 打印空闲显存，或检查 TF32 开关是否被全局关掉，做成 `make env-check` 后，新人入职第一件事就是跑它。多人共用的训练镜像里，把自检挂到容器启动脚本上更彻底：环境有问题时容器直接退出，而不是等训练跑了半天才崩在第一个卷积上。

### 1.5 IDE 与调试配置

调试 PyTorch 有两个额外要求：必须锁定解释器路径，否则 IDE 用全局 Python，装好的包全都 import 不到；必须能步入框架内部，否则断点一进 `torch` 就跳出，无法定位是哪一层把形状改坏了。VS Code 在 `.vscode/launch.json` 里填四项关键配置即可：`python` 写死虚拟环境解释器（Windows 为 `${workspaceFolder}/.venv/Scripts/python.exe`，Linux 与 macOS 换成 `.venv/bin/python`）；`justMyCode` 设为 `false`，断点才能步入 PyTorch 与第三方库源码；`env` 里写 `CUDA_VISIBLE_DEVICES` 锁定一张卡，既防止调试任务挤掉别人的训练，也让代码里的 `cuda:0` 指向确定的物理卡；`PYTHONPATH` 指向工作区根目录，保证包内相对导入能解析。PyCharm 的对应做法是把 Project Interpreter 指向 `.venv`，在 Run Configuration 里勾选把内容根加入 PYTHONPATH，并填入同样的环境变量。

调试 `DataLoader` 时注意，`num_workers` 大于 0 时数据加载发生在子进程，主进程断点进不去，调试阶段先设为 0。断点本身也可以带条件，例如在形状变化的那个函数上写 `x.shape[2] != 224`，只在异常样本上停下，比在循环里手动翻几十次高效得多。若反向传播报错指向一个不相干的算子，在训练循环开头加 `torch.autograd.set_detect_anomaly(True)`，它会在前向时记录起源，把 NaN 或原地修改的定位信息打全，代价是速度下降数倍，只在排查期开启。

显存相关的排查另有工具：`torch.cuda.memory_summary()` 会打印分配明细与碎片情况，`memory_allocated()` 与 `memory_reserved()` 的差值越大，说明缓存块被切得越碎。在 Jupyter 里出错后不必重跑整个单元格，`%debug` 直接进入事后调试，`%pdb on` 则让每次异常自动落入调试器，交互式实验时非常顺手。

远程开发时本地 IDE 通过 SSH 连到训练机，这时 `python` 要填远端虚拟环境里的绝对路径，`CUDA_VISIBLE_DEVICES` 也要按远端的卡号设置，而不是照抄本机的 0 号卡。断点位置飘移或变量显示成奇怪的占位符，通常说明调试器加载的是远端另一份代码，先核对两端的路径映射与 torch 版本，再怀疑代码。

### 1.6 安装失败的原因与排查顺序

| 现象 | 根因 | 对策 |
| --- | --- | --- |
| `Could not find a version that satisfies the requirement torch` | Python 版本过新过旧、32 位解释器、网络拦截索引 | 换 3.10~3.12 的 64 位解释器，换索引或加代理 |
| `is_available()` 为 False 但 `nvidia-smi` 正常 | 装成 CPU 版、驱动低于轮子要求、显卡被独占 | 查 `torch.version.cuda` 是否为 None |
| 内核镜像不可用 | 显卡算力不在轮子编译目标内，新卡配旧轮子 | 换更高 CUDA 版本的轮子，50 系需 cu128 及以上 |
| `WinError 126` 或 `DLL load failed` | 缺 VC++ 运行库，或旧 CUDA DLL 排在 PATH 前面 | 装 VC++ 2015-2022 运行库，清干净后重装 |
| 下载卡住、`ReadTimeoutError` | 直连官方索引慢或代理不稳定 | 换代理、离线 whl，临时调大 `--timeout` 与 `--retries` |
| `import torch` 耗时十几秒 | site-packages 在网络盘或机械盘，被杀毒实时扫描 | 环境放本地 SSD，解释器目录加白名单 |

固定排查顺序，不要跳步：第一，`python -c "import sys; print(sys.executable)"` 确认解释器；第二，`python -m pip -V` 确认这个 pip 属于同一解释器；第三，`pip show torch` 看版本与安装位置；第四，`nvidia-smi` 看驱动与显卡占用；第五，回到上表对号入座。多数「装不上」在第二步就暴露了真实原因——PATH 里排在前面的是另一个 Python 的 pip。

三个具体案例值得记住。企业网络下报证书错误或 `403 Forbidden`，是代理在做证书拦截，把公司根证书导入系统信任链，或在 pip 配置里指定证书路径；报 `No space left on device` 却看磁盘还有空间，多半是临时目录所在分区满了，用 `pip cache dir` 找到缓存位置清理；同一环境里先用 conda 装再用 pip 覆盖，典型症状是 `ImportError: undefined symbol` 或 cudnn 版本不符，此时不要逐个修库，删掉环境重建是最快的路径。

还有两类与环境本身相关的坑。一是权限与路径：把 Python 装在需要管理员权限的目录下，普通用户跑脚本时会读不到部分文件；Windows 上环境路径过深会撞 260 字符限制，项目放在 `D:\proj\xxx` 这类短路径下最省事。二是诊断手段：`pip install -v` 会打印真实的下载地址，用来确认到底走了哪个索引，比盯着报错猜要可靠得多。

### 1.7 从零搭环境的顺序清单

把环境搭建拆成固定顺序，可以避免绝大部分返工。第一步，确认项目要求的 Python 与 PyTorch 版本，依据是项目里的依赖清单或历史镜像标签，而不是网上教程；第二步，创建并激活独立环境；第三步，先升级 pip 与 wheel 这类基础工具，再装业务依赖；第四步，按本机驱动版本选定索引装 torch 三件套；第五步，跑自检脚本，确认运行时版本、算力、显存峰值三项都对；第六步，装项目其余依赖后执行 `pip check`，它会校验包之间的依赖声明是否自洽，比等到运行时报错更早发现问题；第七步，`python -m pip freeze > requirements.txt` 存成基线；第八步，把同一串命令写进 CI 或 `Makefile`，用一条命令替代口口相传的文档。

版本锁定有个容易被忽略的细节：`requirements.txt` 里写 `torch==2.7.0` 只锁住版本号，锁不住 CUDA 变体，因为变体由索引地址决定。稳妥做法是把 `--index-url` 直接写到文件顶部，并在注释里注明目标 CUDA 版本与显卡型号，让后来人一眼看出这份清单的适用边界。照抄别人的清单是最常见的翻车原因：对方的卡是 40 系，你的卡是 50 系，同一串命令装出来的结果完全不同。

### 1.8 常见坑

- **习惯用裸 `pip` 而不是 `python -m pip`**：PATH 上第一条 `pip` 可能属于另一个解释器，命令返回成功，包却装进了别的环境，运行时依旧 `ModuleNotFoundError`。
- **在同一个环境里混用 conda 与 pip 装 PyTorch**：两套包管理器各自记录依赖，后装的一方会覆盖前一方写下的 cudnn、nccl 版本，最终出现 `torch.version.cuda` 正常但卷积报错，修不好只能重建。
- **把 `nvidia-smi` 里的 CUDA Version 当成已安装版本**：那只是驱动支持的上限，真实运行时版本只以 `torch.version.cuda` 为准，两者不一致是常态而非故障。
- **多卡机器上不设 `CUDA_VISIBLE_DEVICES`**：代码里写 `cuda:0` 会命中默认可见的第一张卡，那张卡可能已被别人的任务占满，表现为无规律的 OOM，换个时间又复现不了。固定卡号既让实验可复现，也是共享机器的基本礼貌。

## 2. 张量基础

### 2.1 张量与它的四件套

张量（Tensor，PyTorch 中承载全部数据与参数的多维数组）是这门框架里最小的数据单位。标量是 0 维张量，向量是 1 维，矩阵是 2 维，一批彩色图片是 4 维 `(N, C, H, W)`，一批文本 token 是 2 维 `(B, L)`。层与层之间流动的一切都是张量，模型参数本身也是张量。

看懂一个张量要同时看四件事，缺一件就会在某一处报错：数值（里面存了什么）、形状 shape（每一维多长）、数据类型 dtype（每个元素占几个字节、按整数还是浮点解释）、设备 device（数据在 CPU 内存还是某张 GPU 显存里）。这四项共同决定了一个张量能参与什么运算、占多少内存、能和谁相加。

与 NumPy 的 ndarray 相比，张量多出三项能力：可以搬到 GPU 上计算；记录运算过程以支持自动求导（Autograd，自动计算梯度的机制）；与深度学习算子（卷积、注意力、归一化）直接对接。代价是创建、搬移、类型转换都有明确规则，写错一步就是运行时报错或静默的精度问题，不像 Python 列表那样宽容。

内存占用算起来很直接：字节数 = 元素个数 × 单个元素字节数。一个 `(1024, 1024)` 的 float32 张量是 4 MB，同形状的 float64 或 int64 都是 8 MB。估显存时先算这个数，再乘上需要同时存活的份数（参数、梯度、优化器状态、激活值），比凭感觉调 batch size 可靠得多。

### 2.2 创建张量与 dtype 选择

新建张量的入口分三类：从已有数据构造、按形状填充、按规律生成序列。

```python
import torch

a = torch.tensor([1, 2, 3])                        # 从列表构造，整数自动推断为 int64
b = torch.tensor([1.0, 2.0], dtype=torch.float32)  # 显式指定浮点类型，推荐写法
z = torch.zeros(2, 3)                              # 全 0，默认 float32
o = torch.ones_like(z)                             # 与 z 同形状同 dtype 全 1，常用来造 mask
e = torch.empty(4, 4)                              # 只分配内存不初始化，用前必须填充
r = torch.randn(3, 4)                              # 标准正态分布，权重初始化的原料
i = torch.randint(0, 10, (2, 5))                   # [0,10) 随机整数，可模拟标签或索引
s = torch.arange(0, 12, 2)                         # 步长 2 的序列 [0,2,4,6,8,10]
f = torch.linspace(0, 1, 5)                        # 闭区间等分 5 个点
g = torch.eye(3)                                   # 3x3 单位矩阵
```

dtype 的默认规则只有两条，记住就不会被莫名其妙的类型报错绊住：含小数的字面量推断为 `torch.float32`，纯整数推断为 `torch.int64`。`torch.Tensor(...)`（大写 T 的构造器）是历史遗留写法，永远返回 float32，会把整数也转成浮点，新代码不要用它。

| dtype | 每元素字节 | 典型用途 | 注意点 |
| --- | --- | --- | --- |
| `torch.float32` | 4 | 模型参数与训练的默认精度 | 绝大多数层期望的输入类型 |
| `torch.float16` | 2 | 混合精度训练的运算类型 | 表示范围小、易上溢，需 GradScaler 配合 |
| `torch.bfloat16` | 2 | 大模型训练与推理 | 尾数少精度低，范围与 float32 相同，不易溢出 |
| `torch.int64` | 8 | 索引、类别标签、token id | `nn.Embedding` 的输入必须是它 |
| `torch.int32` | 4 | 省内存的索引 | 部分算子不支持，转换前先查 |
| `torch.bool` | 1 | 掩码、比较结果 | 不能直接与浮点张量做算术 |

带 `_like` 后缀的构造函数（`torch.zeros_like`、`torch.randn_like`、`torch.full_like`）会完整继承输入张量的 dtype 与 device，这是写设备无关代码最省事的做法：模型搬到 GPU 后，所有基于参数形状新建的临时张量自动跟着上卡，不必逐个传 `device=`。

选型有三条经验。模型参数与激活值默认 float32；确定在 GPU 上且硬件支持时，训练用 bfloat16（Ampere 及更新架构）或 float16 加梯度缩放，推理可以再降到 int8；凡是索引、类别标签、token id 一律 int64，喂给 Embedding 或交叉熵之前用 `.long()` 对齐。

类型转换要当成一次复制来理解。`.float()`、`.half()`、`.long()`、`.to(dtype)` 都会新建一块内存并逐元素转换，在大张量上循环调用会让显存翻倍；转换只做一次，放在数据进入模型之前的边界处。旧代码里的 `.type(torch.FloatTensor)` 已不推荐，统一用 `.to()` 或具体类型方法。

取单个元素的值用 `.item()`，它把 0 维张量转成 Python 数字。打印 loss、累加统计量时用它，直接 `print(loss)` 会带出张量结构。

```python
loss = torch.tensor(0.7321)
print(loss)          # tensor(0.7321)
print(loss.item())   # 0.7321，纯 Python float，可安全写日志
```

打印张量默认只显示 4 位小数并折叠中间维度，调试形状够用；要看数值细节时调 `torch.set_printoptions(precision=6, sci_mode=False)`。

### 2.3 device：CPU 与 GPU 的分界线

device（设备，标明张量存储与计算所在硬件的属性）在代码里是字符串或 `torch.device` 对象：`"cpu"`、`"cuda"`、`"cuda:0"`、`"mps"`。同一台机器上 CPU 内存与 GPU 显存是两块物理隔离的空间，张量绝不会自动搬运。

搬运只有两个方向要记：`.to(device)` 返回一个在新设备上的张量（原张量不动），`Module.to(device)` 是原地修改，把模块的全部参数与 buffer 搬过去。`x.cuda()` 等价于 `x.to("cuda")`，但写死 cuda 的代码在没有 GPU 的机器上直接崩，工程代码统一走 `.to(device)`。

```python
import torch

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
print(device, torch.cuda.device_count())
if device.type == "cuda":
    print(torch.cuda.get_device_name(0))   # 打印卡型号，确认没选错机器

x = torch.randn(2, 3)
x = x.to(device)                            # 返回新张量，必须赋值回原变量
x = x.to(torch.float64)                     # to 也能同时改 dtype
y = torch.randn(2, 3, device=device)        # 创建时直接落在目标设备，省一次搬运
```

最高频的报错是设备不一致：`RuntimeError: Expected all tensors to be on the same device, but found at least two devices, cuda:0 and cpu!`。原因通常是某个中间张量用了 `torch.zeros(...)`、`torch.arange(...)` 这类没带 `device=` 的构造，或从 NumPy 转进来的张量天生在 CPU。排查动作固定：把报错行涉及的两个张量都打印 `x.device`，而不是猜。

第二个高频坑是取值前必须搬回 CPU。`.item()`、`.numpy()`、`print` 取值都要 CPU 张量，在 GPU 张量上 `.item()` 会触发一次隐式同步，训练循环里频繁调用会明显拖慢速度。累加指标时先在 GPU 上相加，一个 epoch 结束再搬一次。

搬移本身也有成本。小张量逐个 `.to(device)` 会把时间耗在启动内核上，能批量搬的就批量搬；DataLoader 侧用 `pin_memory=True` 配合 `non_blocking=True` 让拷贝与计算重叠，这是省时间的常规手段。

### 2.4 形状操作：视图与复制

形状操作分两类：只改元信息、与源张量共享内存的视图（View，不复制数据的新形状解释），以及必须复制数据的新张量。分清这两类，才能解释「为什么改了 y，x 也变了」。

| 方法 | 作用 | 是否共享内存 | 典型用途 |
| --- | --- | --- | --- |
| `view(...)` | 按新形状重新解释 | 是，要求连续 | 展平特征、拆分 batch 维 |
| `reshape(...)` | 同上，不满足条件时自动复制 | 尽量共享 | 不确定是否连续时的安全选择 |
| `unsqueeze(dim)` | 在 dim 处插入长度为 1 的维 | 是 | 给标量加 batch 维、对齐广播 |
| `squeeze(dim)` | 删除指定长度为 1 的维 | 是 | 去掉多余的 1 维 |
| `transpose(d1, d2)` | 交换两个维度 | 是，结果通常不连续 | 矩阵转置、把 `(B,L)` 变 `(L,B)` |
| `permute(...)` | 按给定顺序重排所有维度 | 是，结果通常不连续 | `(B,C,H,W)` 转 `(B,H,W,C)` |
| `flatten(start, end)` | 合并连续若干维 | 视情况 | 卷积特征送进全连接层 |
| `expand(...)` | 把长度为 1 的维扩成更大 | 是（零拷贝） | 广播前显式扩展 |
| `repeat(...)` | 按次数复制整块数据 | 否 | 真正需要多份数据时 |

```python
x = torch.arange(24).reshape(2, 3, 4)   # 形状 (2,3,4)
print(x.shape, x.numel(), x.stride())   # numel 元素总数；stride 每维步长

x.unsqueeze(0).shape                    # (1,2,3,4)，最前面加 batch 维
x.unsqueeze(-1).shape                   # (2,3,4,1)，最后加一维
x.squeeze().shape                       # 没有长度为 1 的维，原样返回
torch.zeros(1, 3, 1).squeeze().shape    # (3,)，删掉所有 1 维，可能误删 batch 维

x.permute(2, 0, 1).shape                # (4,2,3)，按索引顺序重排
x.transpose(0, 1).shape                 # (3,2,4)，只换两维
x.flatten(0, 1).shape                   # (6,4)，合并前两维
x.reshape(2, -1).shape                  # (2,12)，-1 表示该维长度自动推断
```

要理解 `view` 的限制，先理解步长。shape 描述「每一维多长」，stride 描述「沿这一维走一步，内存地址要跳几个元素」。`torch.arange(12).reshape(3, 4)` 的 stride 是 `(4, 1)`：换一行跳 4 个元素，换一列跳 1 个，与行优先的物理布局完全吻合。`view` 的本质是只换一组 shape/stride 而不碰数据，所以目标形状必须能由原步长拼出来；转置后 stride 变成 `(1, 4)`，元素在内存里不再按逻辑顺序排列，任何要求「紧密排列」的新形状都没有合法解。

`view` 只接受能靠改步长实现的形状变化，对转置后的张量调用它会失败：

```text
RuntimeError: view size is not compatible with input tensor's size and stride
(at least one dimension spans across two contiguous subspaces). Use .reshape(...) instead.
```

这不是 bug。转置后内存里元素的物理顺序没变，只是步长变了，无法用一组新步长表达目标形状。`reshape` 遇到同样情况会默默复制一份，因此更安全也更慢，并且失去了与源张量的内存共享。

`squeeze()` 不带参数会删掉所有长度为 1 的维，这是隐蔽事故的来源：batch size 恰好为 1 时，`(1, 3, 224, 224)` 被挤成 `(3, 224, 224)`，下游层要么把它当作无 batch 的输入，要么直接形状报错。永远写 `squeeze(dim)` 指明维度。

`permute` 与 `transpose` 返回的都是视图，结果的步长被打乱，后续操作若要求连续内存就会报错，处理手段是对结果调用 `.contiguous()` 复制一份连续数据。

还有一个容易被忽略的语义差别：对 `permute` 结果调用 `reshape(-1)` 或 `flatten()`，得到的元素顺序按物理内存排列，而不是按逻辑上的行优先顺序。想先把维度换对再拉平成一维，正确顺序是 `x.permute(0, 2, 3, 1).contiguous().view(N, -1)`，先落成连续布局再展平，否则得到的特征顺序与训练时不一致，模型能跑但效果莫名变差。

### 2.5 广播规则

广播（Broadcasting，形状不同的张量按固定规则隐式扩展成兼容形状后再逐元素运算）让代码变短，也带来一类难查的错误。规则三条：从最右维开始对齐；维数不足的一方在左侧补 1；每一维要么长度相等，要么其中一方为 1。

```python
a = torch.randn(4, 1, 3)
b = torch.randn(2, 3)
(a + b).shape    # (4,2,3)：b 左补一维成 (1,2,3)，两个 1 维分别扩成 4 和 2
```

训练中最常见三个广播场景：标准化时 `(B, C, H, W)` 减去按通道的均值 `(C, 1, 1)`，得到每个通道独立的偏移；注意力里 `(B, 1, 1, L)` 的掩码与 `(B, H, L, L)` 的分数相加；逐样本加权时 `(B, 1)` 的权重乘 `(B, D)` 的特征。

反例比正例更能说明规则。`(3, 4)` 与 `(3,)` 相加失败：右对齐后后者是 `(1, 3)`，最后一维 4 与 3 既不相等也没有 1，报 `The size of tensor a (4) must match the size of tensor b (3) at non-singleton dimension 1`。想按行做逐行缩放，正确写法是把后者 reshape 成 `(3, 1)`，或者先确认语义再转置。

广播不复制数据，只在计算时按步长 0 复用同一块内存，`(1, 1000, 1000)` 与 `(64, 1, 1)` 相加不会真的分配 64 份输入。但结果张量是实打实的新内存，`(64, 1000, 1000)` 的 float32 结果要 256 MB，显存吃紧时先怀疑某次广播把形状放大了。

显式扩展有时更清楚：`expand` 只能扩展长度为 1 的维，零拷贝；`repeat` 按次数真正复制。需要多份独立数据（例如构造对比学习的正负样本）时用 `repeat`，只是为对齐形状时用 `expand`。

排查广播问题时不要靠心算。PyTorch 提供了 `torch.broadcast_shapes(a.shape, b.shape)`，直接返回两者能广播出的形状，不兼容则抛错；把可疑的两个形状丢进去，一行就能定位是哪一维对不上。另一种情况是不报错但结果维度比预期多一维，通常是某个张量本该 `squeeze` 掉却没处理，被广播悄悄扩成了新的维度。

### 2.6 索引、切片与 gather

基础索引与 NumPy 几乎一致，先建立一张对照。

| 写法 | 含义 | 结果形状 |
| --- | --- | --- |
| `x[0]` | 取第 0 个 | 去掉最左维 |
| `x[:, 1]` | 所有行的第 1 列 | `(行数,)` |
| `x[1:3, ::2]` | 行切片 + 列步长 | 与切片范围一致 |
| `x[..., -1]` | 最后一维取最后一个 | 保留前面所有维 |
| `x[x > 0]` | 布尔掩码取值 | 一维，长度不定 |
| `x[[0, 2]]` | 整数列表索引 | 按列表长度取行 |

布尔掩码返回一维张量，元素个数取决于命中数量，这种形状不确定的结果不适合直接进入网络层，多用于统计与过滤。用张量做索引（高级索引）返回的是副本而不是视图，改它不影响原张量。

索引也能出现在赋值左侧，这是清洗数据最常用的句式：`x[x != x] = 0` 把 NaN 清零（NaN 不等于自身，这个写法靠的是 IEEE 754 的定义），`x[mask] = 0` 把掩码位置置零。注意给需要梯度的张量做赋值会破坏计算图，训练中的掩码操作要用 `masked_fill` 或 `torch.where` 这类返回新张量的函数，而不是直接改原张量。

gather（按索引在指定维度上取值的算子）是分类任务里取目标类别分数的标准写法，也是形状最容易写错的地方。它的语义是：对输出中每个位置，沿 `dim` 用同位置的 index 值去输入里取，dim=1 时即 `out[i][j] = input[i][index[i][j]]`。约束是 index 与 input 维数相同，且除 `dim` 外其余维长度不超过 input。

```python
logits = torch.randn(32, 10)                      # 32 个样本，10 类
target = torch.randint(0, 10, (32,))              # 每个样本的正确类别
picked = logits.gather(1, target.unsqueeze(1))    # (32,1)，取到每条样本自己的分数
picked = picked.squeeze(1)                        # (32,)
loss = -picked.log().mean()                       # 交叉熵的核心部分
```

三个细节：index 必须是 int64；`dim` 写错会静默取到错误的值而不报错，先打印一行手工核对；`scatter` 是 gather 的反向操作（按索引把值写回），实现 one-hot 与标签平滑时用到。另外两个常用工具：`index_select(dim, index)` 沿一维按索引取子集，语义比 gather 简单；`masked_fill(mask, value)` 把 mask 为 True 的位置填成指定值，注意力里把不该看的位置填 `-inf` 再 softmax 就出自这里，逐元素三目选择用 `torch.where(cond, a, b)`。

### 2.7 拼接与切分

`cat` 与 `stack` 的差别只有一句话：`cat` 沿已有维度接起来，维数不变；`stack` 新建一个维度，维数加一。

| 操作 | 输入要求 | 参数 | 输出 | 场景 |
| --- | --- | --- | --- | --- |
| `torch.cat` | 除 dim 外形状一致 | `dim` | 维数不变 | 拼 batch、拼多路特征 |
| `torch.stack` | 形状完全一致 | `dim` | 维数加一 | 把样本列表变成 batch |
| `torch.split` | 一个张量 | 每块大小, `dim` | 张量元组 | 按固定大小切 |
| `torch.chunk` | 一个张量 | 块数, `dim` | 尽量均分 | 大 batch 分块推理 |
| `torch.unbind` | 一个张量 | `dim` | 去掉该维的元组 | 逐样本处理 |

```python
a = torch.randn(2, 3)
b = torch.randn(2, 3)
torch.cat([a, b], dim=0).shape     # (4,3)
torch.cat([a, b], dim=1).shape     # (2,6)
torch.stack([a, b], dim=0).shape   # (2,2,3)，新建的第 0 维长度等于列表长度

x = torch.randn(10, 4)
for part in torch.split(x, 3):     # 3,3,3,1，最后一块可能不足
    print(part.shape)
for part in torch.chunk(x, 3):     # 4,3,3，按块数均分
    print(part.shape)
```

`split` 按每块大小切，`chunk` 按块数切，块数除不尽时前面的块大一点。分块推理（把大 batch 塞进显存）用 `chunk` 更自然，但要保证切出的块非空，空张量进入某些算子会直接报错。

维度参数写错是拼接类报错的主因。`cat` 要求除拼接维外所有维长度相等，报错信息会列出两个形状，逐维比对即可；`stack` 要求形状完全一致，混进一个形状不同的样本就失败。

一个真实的高频错误：手写 `collate_fn` 把 `list[(3, 4)]` 的样本聚成 batch 时用了 `torch.cat`，得到 `(3N, 4)`，而模型看到的是「N 倍长的一条样本」，维度上完全不报错，训练也能跑，只是 batch 语义已经错了。正确做法是 `torch.stack` 得到 `(N, 3, 4)`。判断口诀：要在最前面多出一个 batch 维，用 `stack`；要在已有维上续接，才用 `cat`。

### 2.8 原地操作、内存共享与 contiguous

方法名以下划线结尾表示原地修改（In-place，直接在原张量内存上写入），`add_`、`mul_`、`zero_`、`clamp_`、`relu_` 都属于这一类。原地操作省内存，但会带来三类事故。

破坏自动求导：对需要梯度的叶子张量做原地修改，报 `RuntimeError: a leaf Variable that requires grad is being used in an in-place operation`，参数更新要在 `torch.no_grad()` 下做或交给优化器。静默改掉别处的数据：视图与源张量共享内存，`y = x.view(-1)` 之后 `y.add_(1)`，`x` 也变了。覆盖计算图仍需的值：某个张量在前向里被原地改过，反向传播用的是改后的值，梯度偏了却不报错——这类错误最难查，因为它不抛异常。

判断两个张量是否共享内存，比 `data_ptr()`：地址相同就是同一块存储。`.storage()` 已被弃用，用 `untyped_storage()` 或直接比指针。

```python
x = torch.arange(6)
y = x.view(2, 3)
print(y.data_ptr() == x.data_ptr())   # True，视图共享存储

z = x.clone()
print(z.data_ptr() == x.data_ptr())   # False，clone 复制数据，仍在计算图里

w = x.detach()
print(w.data_ptr() == x.data_ptr())   # True，detach 只切断梯度，不复制数据
```

contiguous（连续，指张量元素在内存中按行优先顺序紧密排列）是很多算子对内存布局的隐含要求。转置、`permute`、切片通常得到不连续张量，它们的 shape 看着正常，stride 却已经乱了。

```python
x = torch.arange(12).reshape(3, 4)
t = x.t()
print(t.shape, t.stride(), t.is_contiguous())   # (4,3) (1,4) False

t.view(-1)                  # 报错：view size is not compatible ...
t.reshape(-1)               # 可以，内部先复制
t.contiguous().view(-1)     # 显式复制成连续布局再 view，意图最清楚
```

要不要无脑加 `.contiguous()`：不要。它是一次真实的内存复制，在热路径上反复调用会明显增加耗时与显存峰值。判断标准是下一步操作是否要求连续——`view` 与部分老算子明确要求连续，`.numpy()` 在负步长时不支持，逐元素运算与矩阵乘通常不需要。看到 `is_contiguous()` 为 False 时先想清楚为什么，再决定复制还是调整上游写法，例如 `permute` 之后接 `reshape`，让复制只发生一次。

原地操作并非只在优化器里有正当用途。给不需要梯度的统计量清零（`running_mean.zero_()`）、在 `torch.no_grad()` 里把某列赋零、给已 detach 的中间结果做归一化，都是合法且省内存的写法；`x += 1` 这种复合赋值运算符也等价于原地修改，是否安全取决于 `x` 是不是需要梯度的叶子张量。判断标准很简单：先问这个张量还会不会参与反向传播，再决定用 `x.add_(1)` 还是 `x = x + 1`。

排查形状与布局问题的固定动作：打印 `x.shape`、`x.stride()`、`x.is_contiguous()`、`x.dtype`、`x.device` 五个值。多数「形状看着对但跑不通」的问题，答案就在 stride 那一项里。

### 2.9 与 NumPy 互转

PyTorch 与 NumPy 的转换有共享内存与复制两条路，选错会得到「改了这边那边也变」的现象，或者意外的性能开销。

| 写法 | 是否共享内存 | 说明 |
| --- | --- | --- |
| `torch.from_numpy(arr)` | 是 | 要求 ndarray 类型受支持，修改互相可见 |
| `torch.as_tensor(arr)` | 尽量共享 | 已是张量则原样返回，最省 |
| `torch.tensor(arr)` | 否 | 总是复制，最安全 |
| `tensor.numpy()` | 是 | 要求 CPU 且 `requires_grad=False` |
| `tensor.tolist()` | 否 | 转 Python 列表，仅调试用 |

三个必踩的坑。第一，`numpy()` 不能用在需要梯度的张量上，报 `RuntimeError: Can't call numpy() on Tensor that requires grad. Use tensor.detach().numpy() instead.`。第二，带负步长的张量（`torch.flip` 之后的结果）转 NumPy 会失败，报 `ValueError: some of the strides of a given numpy array are negative`，先 `.contiguous()` 再转。第三，NumPy 默认浮点是 float64，`np.random.randn(...)` 造出的数组转成张量就是 float64，再与 float32 的权重做矩阵乘会报 `expected scalar type Float but found Double`，边界处统一 `astype(np.float32)` 或 `.float()`。

```python
import numpy as np
import torch

arr = np.random.randn(3, 4).astype(np.float32)   # 显式 float32，避免 dtype 不一致
t = torch.from_numpy(arr)                        # 共享内存，零拷贝
t.mul_(2)                                        # 原地修改，arr 同步变化
print(arr[0, 0])                                 # 已翻倍

back = t.detach().cpu().numpy()                  # 回 NumPy 的标准写法
copy = torch.tensor(arr)                         # 复制一份，之后互不影响
```

由于共享内存，`from_numpy` 之后不要让原数组被回收或复用，多进程 DataLoader 里容易读到被覆盖的数据。数据进入训练循环前统一复制一次（`.clone()` 或 `torch.tensor`）成本很低，能省掉一整类难查的 bug。

反向转换还有一种写法：`np.asarray(tensor)` 与 `tensor.numpy()` 等价，前提同样是张量在 CPU 且不需要梯度。需要把 GPU 张量交给只接受 NumPy 的库（OpenCV、matplotlib、sklearn）时，标准链路是 `t.detach().cpu().numpy()`，漏掉 `detach` 报梯度错，漏掉 `cpu` 报设备错，这两个报错在字面上完全不同，记住链路顺序能省下大量试错。

### 2.10 速查表与常见坑

| 需求 | 写法 | 备注 |
| --- | --- | --- |
| 看形状与元素总数 | `x.shape` / `x.numel()` | 顺手看 `x.stride()` |
| 加一维 | `x.unsqueeze(0)` | `x[None]` 等价 |
| 展平保留 batch | `x.flatten(1)` 或 `x.reshape(x.size(0), -1)` | 不要无脑 `flatten()` |
| 换设备 | `x = x.to(device)` | 返回新张量，必须赋值 |
| 换类型 | `x.float()` / `x.long()` | 索引与标签必须 long |
| 取标量 | `x.item()` | 会触发 GPU 同步 |
| 拼成 batch | `torch.stack(list, 0)` | 输入形状必须一致 |
| 沿维拼接 | `torch.cat([a, b], dim)` | 其余维必须一致 |
| 按索引取值 | `x.gather(dim, idx)` | idx 为 int64、维数相同 |
| 变连续 | `x.contiguous()` | 真实复制，热路径慎用 |

- 用 `squeeze()` 不带参数，batch size 为 1 时维度被误删；一律写 `squeeze(dim)`。
- 直接对 `permute`/`transpose` 的结果 `view`，报 stride 不兼容；先 `contiguous()` 或改用 `reshape`。
- 新建中间张量忘写 `device=`，训练中途报两个设备上的张量不能相加。
- NumPy 的 float64 数据混进 float32 模型，报 `expected scalar type Float but found Double`；在数据边界处统一转 float32。

## 3. 自动求导

### 3.1 计算图与 requires_grad：谁在图里，谁在图外

自动求导（Automatic Differentiation，让程序按链式法则自动算出导数的技术）是 PyTorch 与 NumPy 最本质的分界。它用的是反向模式（Reverse Mode）：一次前向算出结果，再一次反向就能得到「一个标量对全部参数的偏导」，而训练需要的恰好就是这个方向。手写导数公式在深度网络里不可行，数值差分（`(f(x+h)-f(x))/h`）又在百万参数上慢到无法使用，自动求导是唯一同时满足精确与高效的方案。

计算图（Computational Graph，把张量运算记录成有向无环图，节点是算子、边表示张量与算子之间的依赖）不是提前编译的，而是每跑一次前向就现建一次，反向跑完立刻释放。这种「定义即运行」（Define-by-Run）风格让 `if` 分支、循环、变长序列都能被自然记录，代价是图只活一次，想再反传一次必须显式声明。

为什么一定是反向模式而不是前向模式（Forward Mode，从输入出发逐个传播导数）？前向模式每求一个输入的导数都要完整走一遍图，参数量上百万时成本就是百万次前向；反向模式只需要一次前向加一次反向，总成本约为前向的两三倍，与参数多少无关。深度学习恰好是「参数极多、输出只有一个 loss」的形态，反向模式因此是唯一可用的选择。

这张图还有一个隐性成本：前向时凡是被反向用到的中间结果都要留在显存里，反向前向保存、反向消费、结束释放。这就是同一模型训练比推理多吃几倍显存的主要原因之一（另外两块是参数本身与优化器状态），也是 `retain_graph=True` 会让显存翻倍的直接原因——它把本该释放的中间量继续扣着。

只有 `requires_grad=True` 的浮点张量才会被跟踪。整数张量不能求导，`torch.tensor([1, 2]).requires_grad_(True)` 会直接抛出 `RuntimeError: only Tensors of floating point and complex dtype can require gradients`——分类标签用 `long`，本身就是设计上不允许反传。

| 观察点 | 叶子张量（Leaf Tensor，用户直接创建、不由其他算子产出的张量） | 非叶子张量（由算子计算得到） |
| --- | --- | --- |
| 创建方式 | `w = torch.randn(3, requires_grad=True)` | `y = w * 2` |
| `is_leaf` | `True` | `False` |
| `grad_fn` | `None` | `MulBackward0` |
| 反向后的 `.grad` | 自动写入 | 默认 `None`，需 `y.retain_grad()` |
| 能否交给优化器 | 能 | 不能 |

模型里的参数正是叶子张量：`nn.Linear` 的 `weight`、`bias` 都是 `nn.Parameter`，创建时自动 `requires_grad=True`，因此 `model.parameters()` 交给优化器就是「把全部叶子张量登记为待更新」。

```python
import torch

x = torch.tensor([1.0, 2.0, 3.0], requires_grad=True)
w = torch.tensor([2.0, 2.0, 2.0], requires_grad=True)
b = torch.tensor(1.0, requires_grad=True)

y = (w * x + b).sum()        # 非叶子张量，grad_fn 记录了这一串算子
print(y.is_leaf, y.grad_fn)  # False <SumBackward0 object at 0x...>
y.backward()                 # 标量输出，backward 可省略参数
print(w.grad)                # tensor([1., 2., 3.])：d(sum(w*x+b))/dw 等于 x
print(x.grad)                # tensor([2., 2., 2.])：对 x 求导得到 w
print(b.grad)                # tensor(1.)
```

`requires_grad` 可以被临时开关：`x.requires_grad_(False)` 会让后续依赖它的算子全部不建图，冻结预训练层就是靠这个开关，比手写两套前向逻辑干净得多。局部需要打开时用 `with torch.enable_grad():` 包住那几行。

跟踪规则是「或」而不是「与」：只要有一个输入需要梯度，输出就需要梯度，`grad_fn` 指向那个产生它的算子。因此把标签 `Y` 误设成 `requires_grad=True` 不会有明显报错，只会让整张图多出一条通往标签的分支，白算一份梯度并多占一份显存；这是「训练慢了但结果没错」类问题的常见来源。判断某个张量是否在图里，看 `tensor.requires_grad` 与 `tensor.grad_fn` 两个属性即可，不要靠猜。

### 3.2 backward：标量、非标量与梯度累积

`loss.backward()` 的语义是「从 `loss` 出发，沿计算图反向走一遍，把每个叶子的梯度写进它的 `.grad`」。它要求起点是标量，因为反向模式每次只能传播一个「上游梯度」，标量天然对应 1.0；输出不是标量时必须显式给出这个上游梯度，等价于先对输出做加权求和再求导，数学上叫雅可比向量积（Vector-Jacobian Product，用向量与雅可比矩阵相乘，避免显式构造巨大的雅可比矩阵）。

```python
w = torch.tensor([1.0, 2.0], requires_grad=True)
y = w * 3                              # 形状 (2,)，非标量
y.backward(torch.tensor([1.0, 1.0]))   # 直接 y.backward() 会报 grad can be implicitly created only for scalar outputs
print(w.grad)                          # tensor([3., 3.])，等价于对 y.sum() 求导
```

上游梯度并不是可有可无的装饰。`y.backward(v)` 真正计算的是「`v` 与雅可比矩阵的乘积」，`v` 就是你希望在 `y` 的每个分量上施加的权重。多任务学习里把两个 loss 分别反传、或者把某个辅助损失的权重设为 0.1，本质上都是在选择这个 `v`；而 `y.sum()` 只是 `v` 全为 1 的特例。

第二件必须记住的事：`.grad` 是累加而不是覆盖。反向时 PyTorch 把新算出的梯度加到已有值上，所以训练循环里不清零，第 n 步的梯度就是前 n 步之和，表现为 loss 先降后炸、参数越更新越离谱。

```python
w.grad.zero_()                 # 先清空上一段留下的 [3., 3.]
for step in range(3):
    loss = (w * w).sum()
    loss.backward()            # 每轮都在旧 .grad 上累加，不清零
print(w.grad)                  # tensor([6., 12.])，即三轮 2w 之和，而不是一轮的 [2., 4.]

w.grad.zero_()                 # 写法一：原地清零；写法二为 optimizer.zero_grad(set_to_none=True)
```

梯度累积（Gradient Accumulation，多次前向累积梯度后再更新一次参数）是显存不够时的常规手段：把 batch 切成 `k` 份分别前向反传，但只在第 `k` 份之后调用 `optimizer.step()` 与 `zero_grad()`，等效于 `k` 倍大的 batch。这里要顺手把 loss 除以 `k`，否则梯度量级会放大 `k` 倍。

```python
accum_steps = 4
for i, (xb, yb) in enumerate(loader):
    loss = criterion(model(xb), yb) / accum_steps   # 归一化，保证等效 batch 的梯度量级
    loss.backward()                                 # 每份都对全部参数累加梯度
    if (i + 1) % accum_steps == 0:                  # 攒够 k 份才真正更新
        optimizer.step()
        optimizer.zero_grad()
```

注意 `step()` 与 `zero_grad()` 必须成对出现在同一个 `if` 里，只写 `step()` 不写清零等于把两个累积窗口的梯度混在一起；而 `BatchNorm` 这类依赖 batch 统计的层，在累积模式下看到的是切分后的小 batch，统计量本身会偏，此时要改用 `GroupNorm` 或同步 BN。

`.grad` 的形状与 dtype 永远和参数本身一致，不会因为被广播过而变大：`b` 的形状是 `(1,)`，前向时被广播到 `(64,)`，反向时框架自动对广播出来的维度求和，写回 `b.grad` 的仍是 `(1,)`。这条规则解释了「为什么梯度不需要手动 reshape」，也解释了广播用错时为什么梯度看起来对、训练却不动——求和把本该分开的信号平均掉了。另外，`zero_()` 是原地写零，`set_to_none=True` 是直接置空，后者少一次全量内存写、在梯度稀疏时也更省，梯度裁剪与 `nan_to_num` 这类操作要记得判空。

### 3.3 断开梯度：detach、no_grad 与 inference_mode

三个工具都让「这一段不参与求导」，但作用范围与副作用不同，混用会造成两类事故：该断的没断，显存一路涨到 OOM；断过头，参数永远拿不到梯度。

| 方式 | 是否建图 | 结果 `requires_grad` | 典型用途 |
| --- | --- | --- | --- |
| `tensor.detach()` | 否，仅该张量 | `False` | 取数值、把目标值固定住 |
| `torch.no_grad()` | 否，整段代码 | `False` | 参数更新、验证、指标计算 |
| `torch.enable_grad()` | 是 | 按输入推断 | 在 `no_grad` 内部临时恢复建图 |
| `torch.inference_mode()` | 否 | `False`，且带 inference 标记 | 纯推理；其产物不能再参与求导 |
| `tensor.data` | 否 | `False` | 不推荐：绕过版本计数，原地改会静默算错 |

`detach()` 与源张量共享同一块存储，改动其中一个另一个也变，区别只在于它被从图上摘了下来，后续对它的运算不会再回传到原张量。这个性质在「监督信号来自模型自身」的场景里不可或缺：自编码器的重建目标、知识蒸馏的教师输出、强化学习里的目标 Q 值、动量更新用的影子网络，都必须先把目标张量 `detach()` 掉，否则反传会顺着目标分支一路流回教师或影子网络，把不该更新的参数也改掉。常见的固定目标写法是先算 `target = teacher(x).detach()`，再在更新教师参数时用 `teacher_param.data.copy_(momentum * teacher_param.data + (1 - momentum) * student_param.data)` 这类操作，两步都不建图。

`no_grad()` 则是整段关闭记录，常用于参数更新：更新公式 `w -= lr * w.grad` 本身不该被记录进图，写进 `no_grad` 里既快又不会污染。它既能当上下文用，也能当装饰器包在评估函数上，还能用 `with torch.no_grad():` 嵌套在训练循环内部只包住 `step()` 那一行，粒度比全局设置细。

```python
with torch.no_grad():                      # 教师只做前向，天然不建图
    teacher_logits = teacher(x)
student_logits = student(x)
loss = kl_div(student_logits, teacher_logits.detach())   # 目标值固定，双保险
```

```python
total = 0.0
for xb, yb in loader:
    loss = criterion(model(xb), yb)
    total += loss.item()       # 只取 Python 浮点数，图随 loss 释放
    # total += loss            # 错误：把计算图挂到 total 上，几十步后显存爆炸
    loss.backward()
    optimizer.step()
    optimizer.zero_grad()

acc = (logits.argmax(1) == y).float().mean().item()   # 指标同样只取数值，不进图
```

`inference_mode()` 比 `no_grad()` 更激进：它额外关闭了「版本计数器」等追踪开销，速度略快、显存略省，但产出的张量一旦在之后被用于需要梯度的运算，会报 `Inference tensors cannot be saved for backward`。线上推理服务用它，训练流程里用 `no_grad()`。

还有一个语义陷阱：`model.eval()` 只改变 `Dropout` 与 `BatchNorm` 的行为，完全不关闭求导。验证集上显存不降、速度不变，往往就是少写了一层 `with torch.no_grad():`；`model.eval()` 与 `torch.no_grad()` 要同时使用，前者管「层的行为」，后者管「是否建图」，两件事互不替代。需要全脚本关闭时可用 `torch.set_grad_enabled(False)`，它等价于在全局套了一个 `no_grad`。

### 3.4 retain_graph：计算图的寿命与二次反传

默认情况下反向一结束，中间结果就被释放，图也随之销毁。第二次调用 `backward()` 会得到 `RuntimeError: Trying to backward through the graph a second time`，这就是「图已被释放」的直白说法。需要复用同一张图时传 `retain_graph=True`，让中间张量留到下一次反向结束。

两个场景会用到它：一个前向对应多个 loss（多任务、对抗训练、梯度惩罚）；以及做梯度分析时要对同一输出分别求不同输入的梯度。

```python
x = torch.tensor(2.0, requires_grad=True)
y = x ** 3                            # y = 8

dy = torch.autograd.grad(y, x, create_graph=True)[0]   # 3x^2 = 12，保留图以便继续求导
d2y = torch.autograd.grad(dy, x)[0]                    # 6x = 12，二阶导
print(dy, d2y)
```

`torch.autograd.grad(outputs, inputs)` 与 `backward()` 的区别值得单独记：前者返回梯度值、不写 `.grad`，且可以只对指定输入求导，适合分析；后者直接累加到 `.grad`，是训练路径。`create_graph=True` 让「求导这件事本身」也被记录，从而支持二阶导、梯度惩罚这类需要梯度的梯度的损失。

`retain_graph` 也是最容易被滥用的参数。多损失场景的正确做法是让它们共享一次反向：`(loss_a + loss_b).backward()`，或者分别反传但只在最后一次传 `retain_graph=True`；如果每步都开着它，图会一直挂到下一次前向，显存稳步上涨，症状与「忘记 release」完全一样。真正需要多次反传又想省显存时，替代方案是梯度检查点（Gradient Checkpointing，前向不保存中间激活，反向时重算一遍），它用计算换显存，比自己手管 `retain_graph` 安全。

排查 NaN 与 Inf 时，`torch.autograd.set_detect_anomaly(True)` 会在反向途中定位到具体算子，代价是速度下降数倍，只在调试时打开。

### 3.5 自定义 autograd.Function 与 gradcheck

当一个运算没有现成实现、或者需要用更快的近似公式替换反向时，可以自定义算子。写法是继承 `autograd.Function`，`forward` 里用 `ctx.save_for_backward` 保存反向要用的张量，`backward` 按 `forward` 的参数顺序返回梯度。

```python
import torch
from torch.autograd import Function

class SquarePlusOne(Function):
    @staticmethod
    def forward(ctx, x):
        ctx.save_for_backward(x)      # 保存中间量，反向时取回
        return x * x + 1

    @staticmethod
    def backward(ctx, grad_output):   # grad_output 是 dL/d(本算子输出)
        (x,) = ctx.saved_tensors
        return grad_output * 2 * x    # 与 forward 参数一一对应，非张量参数返回 None

f = SquarePlusOne.apply               # 用 apply 调用，不要直接调 forward
x = torch.tensor([1.0, 2.0], requires_grad=True)
y = f(x)
y.sum().backward()
print(x.grad)                         # tensor([2., 4.])
```

`save_for_backward` 而非 `ctx.x = x` 的原因：前者会登记版本号，原地修改时能报错；后者会悄悄拿到被改过的值，梯度错了却不报错。反向里若某个输入不需要梯度，可用 `ctx.needs_input_grad` 判断以省计算。

三个进阶约定能避免大部分自定义算子的隐性问题：`backward` 的返回值个数必须与 `forward` 的输入个数完全一致，非张量输入（整数、布尔）对应返回 `None`；若算子本身不可二次求导，用 `@torch.autograd.function.once_differentiable` 装饰 `backward`，否则二阶导会得出错误结果而不是报错；`forward` 中产生的、不需要梯度的输出（如索引、掩码）要用 `ctx.mark_non_differentiable(y)` 标记，否则它会带着一个无意义的 `grad_fn` 参与后续计算，浪费显存又可能崩在反向。

绝大多数需求不该写自定义算子。只要运算能由现有算子组合出来，组合写法自动获得正确的反向、正确的广播与正确的高阶导，可读性也更好；自定义的收益只出现在三种情形：需要接入 PyTorch 没有的底层实现（自定义 CUDA 核、稀疏算子）、反向有更省算的闭式解、以及需要把一个不可导的近似步骤包成可导的直通估计。引入自定义算子就是引入一份自己维护的导数公式，写错时不会报错，只会悄悄训得不对，所以下面的检验步骤不是可选项。

写完必须验证。梯度检验（Gradient Check，用数值差分近似导数，再与解析梯度比对）能抓住符号写反、漏乘 `grad_output` 这类错误；它要求双精度、小尺寸、避免不可导点。若算子需要支持二阶导，还要再跑一次 `gradgradcheck`，否则 `create_graph=True` 的场景会在训练中途才暴露错误。

```python
from torch.autograd import gradcheck

x = torch.randn(4, dtype=torch.double, requires_grad=True)   # 必须 float64
print(gradcheck(f, (x,), eps=1e-6, atol=1e-4))               # True 表示两者一致
```

### 3.6 完整反传示例：手写线性回归并对照手算梯度

把前面几块拼起来：构建叶子参数、前向建图、反向取梯度、在 `no_grad` 里更新、清零后再来一轮。跑完用解析公式核对一遍，是确认「框架算得对、自己理解也对」的最快方式。

```python
import torch

torch.manual_seed(0)
N, D = 64, 3
X = torch.randn(N, D)
true_w, true_b = torch.tensor([2.0, -3.0, 0.5]), 1.7
Y = X @ true_w + true_b + 0.1 * torch.randn(N)   # 加噪，避免完美拟合

w = torch.zeros(D, requires_grad=True)
b = torch.zeros(1, requires_grad=True)
lr = 0.1

for epoch in range(200):
    y_hat = X @ w + b                    # 前向：shape (64,)，图随之建立
    loss = ((y_hat - Y) ** 2).mean()     # 均方误差，标量
    loss.backward()                      # 反向：dloss/dw 与 dloss/db 写入 .grad

    with torch.no_grad():                # 更新参数不能建图
        w -= lr * w.grad
        b -= lr * b.grad
    w.grad.zero_()                       # 先清零再下一轮，否则梯度累积
    b.grad.zero_()

print(w.detach(), b.detach())            # 接近 [2., -3., 0.5] 与 1.7

y_hat = X @ w + b                        # 再跑一次前向，准备与手算梯度比对
((y_hat - Y) ** 2).mean().backward()
manual = 2 / N * (X.T @ (y_hat - Y).detach())
print(torch.allclose(w.grad, manual, atol=1e-6))   # True
```

三个细节决定这段代码是否可靠。其一，`w -= lr * w.grad` 是原地操作，若写在 `no_grad` 外面会触发 `a leaf Variable that requires grad is being used in an in-place operation`。其二，`loss` 必须保持标量，`((y_hat-Y)**2).sum()` 也能训，但梯度会随 batch 大小放大 `N` 倍，学习率要同步调整。其三，`print(w.detach())` 而不是 `print(w)`，带梯度的张量直接被日志框架序列化时会连带图一起留引用。

把反传的每一步写出来，就能理解框架在做什么。设残差 `r = y_hat - Y`，损失 `L = (1/N) Σ r²`。第一步对 `y_hat` 求导得 `dL/dy_hat = 2r/N`，形状与 `y_hat` 相同为 `(64,)`；第二步用链式法则穿过矩阵乘法：`y_hat = Xw + b`，故 `dL/dw = Xᵀ (dL/dy_hat)`，形状 `(3, 64) @ (64,) = (3,)`；对 `b` 则是把 `dL/dy_hat` 沿 batch 维求和，得到 `(1,)`。三次运算的维度全部闭合，这是检查手写反向是否正确的第一道关卡——维度对不上一定是链式法则用错了位置。

这段代码同时解释了三个常见困惑。为什么第 0 步的梯度一定不为零？因为初始 `w=0` 时 `y_hat=0`，残差 `-Y` 不为零，梯度自然非零；如果连初始梯度都是全零，问题一定在 `requires_grad` 或数据搬进了 `no_grad`。为什么损失曲线会先快速下降再缓慢爬行？均方误差在参数靠近真值时梯度趋近于零，固定学习率下收敛会变慢，这是学习率调度要解决的问题而不是自动求导的问题。为什么验证时不需要手动反传？评估只用前向，`no_grad` 一包就不再建图，既省显存也避免无意中累积 `.grad`。

### 3.7 常见坑与速查表

自动求导相关的问题有一个统一的排查顺序：先确认参数 `requires_grad` 是 `True`、`grad_fn` 不为 `None`，排除「根本没建图」；再看 `.grad` 是 `None` 还是全零，区分「没回流」与「回流了但被清零/覆盖」；接着打印梯度的范数，判断是量级异常（累积、未归一化）还是数值异常（NaN/Inf）；最后才怀疑原地操作与自定义算子。这个顺序能覆盖八成现场，也解释了为什么「梯度为 None」和「梯度为 0」必须当成两种病来治。

| 现象 | 原因 | 处置 |
| --- | --- | --- |
| 第二次 `backward()` 报图已释放 | 默认反向完就清图 | 该轮补 `retain_graph=True`，或检查是否重复反传 |
| 梯度是上一轮的几倍 | 忘了 `zero_grad()` | 每轮 `step()` 后清零，或 `set_to_none=True` |
| 参数 `.grad` 恒为 `None` | 参数不在图里，或用了 `.data` 更新 | 确认 `requires_grad=True` 且更新写在 `no_grad` 内 |
| 显存随训练步数线性上涨 | 用 `total += loss` 累积了图 | 改 `total += loss.item()` 或 `loss.detach()` |
| 报 `modified by an inplace operation` | 反传需要的张量被原地改过 | 去掉原地写，改 `x = x.clone()` 后再改 |
| 自定义算子梯度对不上 | `backward` 漏乘 `grad_output` 或顺序错 | 用 `gradcheck` 定位，检查返回值个数 |

四条高频坑按踩到的概率排序：把 `loss` 直接累加进日志变量，长训练必 OOM；用 `.data` 代替 `detach()`，版本计数失效后梯度静默错误；在 `no_grad` 里意外创建了需要梯度的张量却没开 `enable_grad`，导致该层永远学不动；以及冻结层时只设了 `requires_grad=False` 却仍在优化器里传了这些参数，看似冻结实则被 weight decay 继续推动。

## 4. nn.Module 与模型构建

在 PyTorch 里搭模型不是写一个接收张量、返回张量的函数，而是定义继承 `nn.Module`（模块，PyTorch 中所有神经网络组件的基类）的类。可训练的模型除了前向计算，还要回答四个工程问题：哪些张量交给优化器、它们如何组成可递归的树、如何切换训练与推理、如何在不改代码的前提下窥探中间结果。`nn.Module` 用 `__setattr__` 拦截属性赋值自动登记参数与子模块，用 `__call__` 包住 `forward` 以插入钩子，用 `train()`/`eval()` 递归广播状态，用 `state_dict()` 输出带层级前缀的权重字典。

### 4.1 调用协议与生命周期

`model(x)` 不等于 `model.forward(x)`：前者走 `Module.__call__`，依次执行 forward 前钩子、`forward` 本体、forward 后钩子；后者只是普通方法调用，钩子全部被绕过。工程代码一律写 `model(x)`。

模块只有两段生命周期：`__init__` 声明结构，只在实例化时执行一次；`forward` 每次前向执行，不能保存跨 batch 状态，写 `self.xxx = ...` 会污染下一次前向并打断梯度图。子类必须显式调用 `super().__init__()`，父类构造器初始化 `_parameters`、`_modules`、`_buffers` 等注册表，漏掉时报 `cannot assign module before Module.__init__() call`。

```python
import torch
from torch import nn

class MLP(nn.Module):
    def __init__(self, in_dim: int, hidden: int, out_dim: int, p: float = 0.1):
        super().__init__()                    # 初始化内部注册表
        self.fc1 = nn.Linear(in_dim, hidden)  # 赋值即注册为子模块
        self.act = nn.GELU()
        self.drop = nn.Dropout(p)
        self.fc2 = nn.Linear(hidden, out_dim)

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        x = self.act(self.fc1(x))             # (B, in_dim) -> (B, hidden)
        return self.fc2(self.drop(x))         # (B, hidden) -> (B, out_dim)

print(MLP(64, 128, 10)(torch.randn(8, 64)).shape)   # torch.Size([8, 10])
```

设备迁移用 `model.to(device)`，原地搬运参数与 buffer 并返回 `self`。

### 4.2 参数、buffer 与注册机制

`nn.Parameter`（参数，一种默认 `requires_grad=True` 的 Tensor 子类）是可训练权重的唯一合法载体。把 `Parameter` 赋给模块属性时，`__setattr__` 会同时把它写进 `_parameters` 字典，于是 `parameters()` 能枚举到它、优化器能拿到它、`state_dict()` 能保存它。普通 Tensor 赋给属性则什么都不发生：不进 `state_dict`，`.to(device)` 也不搬它。

另一类张量随模型保存但不训练，例如 BatchNorm 的滑动均值、位置编码表、因果注意力掩码，用 `register_buffer` 登记为缓冲（buffer，模块内不训练但属于模型状态的张量）。

| 登记方式 | 出现在 `parameters()` | 出现在 `state_dict()` | 随 `.to(device)` 迁移 | 典型用途 |
| --- | --- | --- | --- | --- |
| `nn.Parameter` | 是 | 是 | 是 | 权重、偏置、可学习缩放 |
| `register_buffer` | 否 | 是（`persistent=True`） | 是 | 滑动统计量、位置编码、掩码 |
| 普通属性 Tensor | 否 | 否 | 否 | 临时常量，切勿用于模型状态 |
| 子 `nn.Module` | 递归包含 | 递归包含 | 是 | 层、块、整个子网络 |

```python
class CausalScore(nn.Module):
    def __init__(self, max_len: int = 1024):
        super().__init__()
        self.temperature = nn.Parameter(torch.ones(1))   # 可训练，进 parameters()
        # 可重算的常量登记为 buffer；persistent=False 表示不进 state_dict
        self.register_buffer("mask", torch.tril(torch.ones(max_len, max_len)),
                             persistent=False)

    def forward(self, scores):                    # scores: (..., T, T) 注意力打分
        t = scores.size(-1)
        return scores.masked_fill(self.mask[:t, :t] == 0, -1e9) / self.temperature
```

`persistent=False` 意味着该 buffer 不进 `state_dict`，换代码或换 `max_len` 时不会因 key 对不上而加载失败；真正的状态（如 `running_mean`）必须 `persistent=True`，否则推理漂移。

### 4.3 常用层与形状契约

层错用几乎都源于形状契约记错，下表六个基础件覆盖绝大多数模型骨架。

| 层 | 形状契约 | 关键行为 |
| --- | --- | --- |
| `nn.Linear(in, out)` | `(..., in) -> (..., out)` | 只变换最后一维，前面维度原样保留 |
| `nn.Conv2d(cin, cout, k, s, p)` | `(N, cin, H, W) -> (N, cout, H', W')` | `H' = (H + 2p - k) / s + 1` |
| `nn.BatchNorm2d(c)` | 形状不变，按通道统计 | 训练用当前 batch 统计量，推理用滑动统计量 |
| `nn.LayerNorm(d)` | 形状不变，对最后 `d` 维归一化 | 与 batch 无关，序列模型首选 |
| `nn.Dropout(p)` | 形状不变 | 训练时以概率 `p` 置零并放大 `1/(1-p)`，推理恒等 |
| `nn.Embedding(num, dim)` | `(N, T)` int64 `-> (N, T, dim)` | 查表，输入必须是整型索引而非 one-hot |

`BatchNorm` 与 `Dropout` 是行为随模式变化的层，开关是模块树上的 `training` 标志：`model.eval()` 递归置 `False`，`model.train()` 置回 `True`。忘记切换时验证指标会抖动，因为推理仍在用当前 batch 统计量归一化。

注意力组件由打分、掩码、归一化、加权求和四步组成，PyTorch 2.x 一行融合调用即可：`F.scaled_dot_product_attention(q, k, v, attn_mask=mask, dropout_p=0.1 if training else 0.0)`。它在支持的设备上自动选择 FlashAttention 或内存高效核，`q`、`k`、`v` 形状均为 `(B, H, T, D)`，推理阶段 `dropout_p` 必须显式为 0。

`nn.MultiheadAttention` 默认 `batch_first=False`，期望输入 `(T, B, D)`；按习惯传 `(B, T, D)` 而不设 `batch_first=True`，它会把时间维当 batch 处理，结果错但不报错。

### 4.4 容器：Sequential、ModuleList 与 ModuleDict

容器负责把子模块登记进模块树，选择标准是前向逻辑的形状。

| 容器 | 前向是否自动生成 | 适用结构 | 不能做的事 |
| --- | --- | --- | --- |
| `nn.Sequential` | 是，按顺序串联 | 无分支的 MLP、编码器主干 | 跳连、多输入、条件分支 |
| `nn.ModuleList` | 否，需自己写 `forward` | 层数可变、需按索引访问的堆叠块 | 直接 `self.layers(x)` 调用 |
| `nn.ModuleDict` | 否，按键取模块 | 多任务头、按名字选分支 | 依赖插入顺序 |

```python
class MultiTaskHead(nn.Module):
    def __init__(self, dim: int, tasks: dict[str, int]):
        super().__init__()
        # ModuleDict 保证每个 head 都被注册，且能按任务名取用
        self.heads = nn.ModuleDict({
            name: nn.Linear(dim, n_out) for name, n_out in tasks.items()
        })

    def forward(self, x, task: str):
        return self.heads[task](x)   # 只计算被请求的那个头
```

参数名自动带上容器路径（`heads.cls.weight`），因此多任务头可以按任务单独冻结。堆叠同构块时，`nn.ModuleList([Block(dim) for _ in range(n)])` 配 pre-norm 残差 `x + self.ff(self.norm(x))` 是最常见的结构片段。最典型的错误是用原生 `list` 装层：`self.layers = [nn.Linear(8, 8) for _ in range(4)]`。列表不是 `Module`，赋值不触发登记，这些层既不在 `parameters()` 也不在 `state_dict` 里，优化器看不到它们，表现为 loss 卡住不动。

### 4.5 初始化：默认够用，但要占对位置

`nn.Linear` 与 `nn.Conv2d` 自带 `reset_parameters()`，按 Kaiming 均匀分布初始化权重，一般够用。需要显式初始化的是上百层的深网络、带自定义 `Parameter` 的层，以及从零训练的 Transformer。

```python
def init_weights(m: nn.Module) -> None:
    if isinstance(m, nn.Linear):
        nn.init.trunc_normal_(m.weight, std=0.02)   # 深网络抑制方差累积
        if m.bias is not None:
            nn.init.zeros_(m.bias)
    elif isinstance(m, nn.LayerNorm):
        nn.init.ones_(m.weight)
        nn.init.zeros_(m.bias)

model.apply(init_weights)   # 深度优先遍历，作用到所有子模块
```

顺序纪律是：先建模型，再 `apply` 初始化，最后 `load_state_dict`。把 `apply` 放在加载之后会覆盖刚读入的权重，现象是训练多轮后效果等同随机初始化。

### 4.6 嵌套、遍历与冻结

遍历 API 的区别在于是否递归、是否带名字：`named_children()` 只给直接子模块，`named_modules()` 递归给出全部，`named_parameters()` 递归给出参数与全名。参数全名即 `state_dict` 的 key，改名等于让旧 checkpoint 失效。

冻结（freeze，把部分参数的 `requires_grad` 置为 `False`）是迁移学习的常规操作，配套动作有二：优化器只接收 `requires_grad=True` 的参数；被冻结的 BatchNorm 要额外切到 `eval` 模式，否则统计量仍被小 batch 改写。

```python
for name, p in model.named_parameters():
    if name.startswith("backbone."):
        p.requires_grad_(False)

trainable = [p for p in model.parameters() if p.requires_grad]
optimizer = torch.optim.AdamW(trainable, lr=1e-4)
print(sum(p.numel() for p in trainable))   # 确认数量级符合预期
```

### 4.7 hooks：不改模型代码取中间结果

钩子（hook，注册在模块上、在特定时机被自动调用的回调）解决一个矛盾：想看中间层输出或梯度，又不想改动 `forward`。`register_forward_pre_hook` 在前向前触发，`register_forward_hook` 在前向后触发并拿到输入输出，`register_full_backward_hook` 在反向后触发并拿到梯度；返回的句柄用完必须 `remove()`。

```python
feats = {}

def grab(name):
    def hook(module, inputs, output):
        feats[name] = output.detach()   # detach 断开计算图，避免显存泄漏
    return hook

handle = model.fc1.register_forward_hook(grab("fc1"))
_ = model(torch.randn(4, 64))
handle.remove()                         # 不移除会持续持有引用并反复写入
print(feats["fc1"].shape)               # torch.Size([4, 128])
```

### 4.8 常见坑

- **用原生 `list` 或 `dict` 装子模块**：不进 `parameters()` 与 `state_dict`，优化器静默跳过，表现为 loss 不降且无报错。一律用 `nn.ModuleList` / `nn.ModuleDict`。
- **推理忘记 `model.eval()`**：Dropout 继续随机置零、BatchNorm 继续用当前 batch 统计量，同一张图两次推理结果不同。成对写法是 `model.eval()` 加 `with torch.no_grad():`。
- **先 `load_state_dict` 再初始化**：`apply` 覆盖刚加载的权重，训练结果与随机初始化无异。固定顺序为建模型 → 初始化 → 加载 → 迁移设备。
- **属性改名或 buffer 漏设 `persistent=False` 导致 key 不匹配**：先打印两侧 key 的差集定位，不要盲目加 `strict=False`，它会把结构错误一起吞掉。

## 5. 数据加载

模型训练的时间往往不花在前向与反向上，而是花在等数据上。GPU 利用率长期停在 20%、`nvidia-smi` 里显存忽高忽低、第一个 batch 等十几秒才出来，这些都是数据加载流水线没有跑满的典型信号。PyTorch 把这件事拆成三件互相独立的工作：Dataset 定义「一条样本长什么样、怎么取出来」，Sampler 决定「这一轮按什么顺序取、取哪些」，DataLoader 负责「攒成 batch、开子进程并行、把结果搬到显存」。理解这条链路的分工，比记住参数默认值更有用。

衡量加载流水线只看两个指标：吞吐（throughput，每秒能产出多少条样本）与首样本延迟（从启动到第一个 batch 进入显存的耗时）。训练进入稳态后盯吞吐，短任务或频繁重建迭代器的场景盯首样本延迟。经验值是每个 worker 每秒应能产出几十到上百条小样本；当单样本处理耗时乘以批大小明显超过 GPU 单批计算耗时，瓶颈就必然在数据侧，此时调模型结构没有丝毫用处。

### 5.1 Dataset、Sampler 与 DataLoader 的分工

数据集（Dataset，一个必须实现取长度与按下标取样本两个方法的类）是唯一需要写业务逻辑的地方；采样器（Sampler，产出下标序列的可迭代对象）由框架提供现成的几种，也可以自己写；数据加载器（DataLoader，把下标变成 batch 张量并负责并行与预取的调度器）几乎不需要继承，只通过参数配置。

| 组件 | 需要实现什么 | 典型改动场景 | 出错表现 |
| --- | --- | --- | --- |
| Dataset | `__len__` 与 `__getitem__` | 读文件、解码、做数据增强、返回 dict | 下标越界、解码报错、返回值类型不一致 |
| Sampler | `__iter__`，产出下标 | 按长度分桶、按类别平衡、按分片切给不同 rank | 样本重复或漏采、每个 epoch 顺序相同 |
| DataLoader | 不继承，只传参 | 批大小、并行进程数、拼批函数、锁页内存 | 卡住、OOM、显存带宽浪费、batch 形状不齐 |

一个实用的判断方法：如果问题是「这条样本的内容不对」，改 Dataset；如果问题是「样本喂进去的顺序或数量不对」，改 Sampler；如果问题是「吞吐不够」或「进程卡死」，改 DataLoader 参数。

不用 `for i in range(0, len(ds), batch_size)` 手写取数循环，换来的是四件事：预取（worker 提前把下一批算好，GPU 不必等 IO）、跨进程并行、锁页内存直传、以及每个 epoch 自动重排。手写循环要实现这些得自己写队列与进程池，而 DataLoader 的版本已经在大量边界场景上打磨过。代价是报错栈里多出一层进程边界，调试时先把 `num_workers` 归零能省下大量时间。

一个 batch 的完整旅程分四步：Sampler 产出一批下标；各 worker 在自己进程里调 `__getitem__` 取样本并做增强；worker 里的 collate 把样本列表拼成张量、主进程把张量写进锁页内存；训练循环再把它异步拷到显存。四步都可替换，也就都能成为瓶颈：下标生成太慢（例如每取一条都要查一次数据库）、样本解码太重、collate 里写了 Python 循环、锁页内存不足导致拷贝同步等待。按这四步逐个计时，比盲目调参有效得多。

### 5.2 map-style 与 iterable-style：Dataset 与 IterableDataset

PyTorch 有两种数据集风格。基于映射的（map-style）实现 `__getitem__(index)`，因此支持随机访问，可以打乱、可以按任意下标取，代价是整个数据集必须能被「按下标定位」，通常意味着数据在本地文件里且索引常驻内存。基于迭代的（iterable-style，`IterableDataset`）只实现 `__iter__`，一条条往外吐样本，适合放不进内存的超大数据集、网络流、日志流。

| 对比项 | map-style Dataset | IterableDataset |
| --- | --- | --- |
| 必须实现 | `__len__`、`__getitem__` | `__iter__` |
| 随机访问 | 支持 | 不支持 |
| shuffle | 由 Sampler 打乱下标完成，代价低 | 只能靠缓冲池局部打乱 |
| 多进程切分 | 框架按下标自动切给各 worker | 需自己用 `worker_info` 切分，否则每个 worker 都从头读全量 |
| 断点续训 | 记住 epoch 与下标即可 | 需要自己记录已消费条数 |
| 适用数据 | 本地图片、小规模表格、持久化索引 | 巨型 jsonl、tar 分片、数据库游标、实时流 |

下面是 map-style 的最小实现，注意 `__init__` 只记录路径与索引，不读内容——把重活留到 `__getitem__`，才能让 `num_workers` 真正并行。

```python
import json
from torch.utils.data import Dataset


class JsonlDataset(Dataset):
    """每行一个 json 的大文件；__init__ 只建行号偏移表，不加载正文。"""

    def __init__(self, path: str, tokenizer, max_len: int = 512):
        self.path = path
        self.tokenizer = tokenizer
        self.max_len = max_len
        self.offsets = []                      # 每行的字节偏移，32 位整数即可省内存
        with open(path, "rb") as f:
            pos = 0
            for line in f:
                if line.strip():
                    self.offsets.append(pos)
                pos += len(line)

    def __len__(self) -> int:
        return len(self.offsets)

    def __getitem__(self, index: int) -> dict:
        with open(self.path, "rb") as f:       # 每个 worker 各自持有文件句柄，跨进程安全
            f.seek(self.offsets[index])
            row = json.loads(f.readline())
        enc = self.tokenizer(
            row["text"], truncation=True, max_length=self.max_len, padding=False
        )
        return {"input_ids": enc["input_ids"], "label": int(row["label"])}
```

越界、损坏行、异常样本都应该在这里就地处理（跳过或打日志），不要让异常穿透到 collate 阶段，那样报错信息会指向一个与业务无关的位置。

`__getitem__` 的耗时可以用一个不等式衡量：批大小乘以单条样本耗时再除以 worker 数，应当小于 GPU 单批前反向耗时。举例，每批 64 条、单条解码 20 毫秒，一个 worker 处理完一批需要 1.28 秒，而 GPU 算完这批只要 120 毫秒，于是大约需要 11 个 worker 才能勉强跟上。这个比值解释了为什么图像与音频任务普遍要开 8 个以上 worker，而纯向量数据 2 个就够。

`IterableDataset` 的关键约束是每个 worker 必须只处理自己那一份分片，否则开了 4 个 worker 等于把数据重复读 4 遍。

```python
import glob
from torch.utils.data import IterableDataset, get_worker_info


class ShardStream(IterableDataset):
    """按 tar/jsonl 分片读取，worker 之间按分片分配，互不重叠。"""

    def __init__(self, pattern: str, shuffle_shards: bool = True):
        self.shards = sorted(glob.glob(pattern))
        self.shuffle_shards = shuffle_shards

    def __iter__(self):
        info = get_worker_info()
        wid, nworker = (0, 1) if info is None else (info.id, info.num_workers)
        # 每个 worker 只拿 shard_id % nworker == wid 的分片
        mine = self.shards[wid::nworker]
        if self.shuffle_shards:
            import random
            random.shuffle(mine)
        for shard in mine:
            with open(shard, encoding="utf-8") as f:
                for line in f:
                    yield {"text": line.strip()}
```

### 5.3 DataLoader 的参数怎么配

| 参数 | 默认值 | 作用 | 实践取值 |
| --- | --- | --- | --- |
| `batch_size` | 1 | 每批样本数 | 按显存上限试，取到刚好不 OOM 的 80% |
| `shuffle` | False | 每个 epoch 打乱下标 | 训练集 True，验证集 False |
| `sampler` | None | 自定义取数顺序 | 传入后必须把 `shuffle` 置 False，否则报错 |
| `batch_sampler` | None | 直接产出整批下标 | 变长样本按长度分桶时使用 |
| `num_workers` | 0 | 加载子进程数 | 从 4 起试，不超过物理核数，GPU 训练建议 ≥ 4 |
| `collate_fn` | None | 把样本列表拼成一个 batch | 变长序列、多模态、dict 嵌套时必写 |
| `pin_memory` | False | 把 batch 放进锁页内存 | GPU 训练设 True，加速 H2D 拷贝 |
| `drop_last` | False | 丢弃最后一个不足批 | 训练集设 True，避免 BatchNorm 在单样本上报错 |
| `persistent_workers` | False | epoch 之间不销毁 worker | 数据加载开销大且 epoch 多时设 True |
| `prefetch_factor` | 2 | 每个 worker 预取的批数 | 内存宽裕可调到 4，内存吃紧降到 1 |
| `worker_init_fn` | None | 每个 worker 启动时的回调 | 设置 numpy/random 种子，保证增强可复现 |
| `timeout` | 0 | 取一个 batch 的超时秒数 | 设 60 以上，让卡死的 worker 抛错而不是永久挂起 |

```python
from torch.utils.data import DataLoader

train_loader = DataLoader(
    train_set,
    batch_size=64,
    shuffle=True,               # 与 sampler 不能同时生效
    num_workers=8,
    collate_fn=pad_collate,
    pin_memory=True,            # 仅 GPU 训练有意义
    drop_last=True,
    persistent_workers=True,    # 必须与 num_workers>0 同时使用
    prefetch_factor=4,
    worker_init_fn=seed_worker,
)
```

调优顺序是固定的：先确认单进程能取到数据（`num_workers=0`、`batch_size=2` 跑两个 batch），再逐步加 `num_workers` 看 GPU 利用率是否上升，最后才考虑 `pin_memory` 与预取。反过来做，一旦出问题就分不清是数据逻辑错还是并发错。

DataLoader 支持迭代协议，`for batch in loader` 每轮都会新建一个迭代器，这也是 `shuffle` 能在每个 epoch 重新打乱的原因；而 `len(loader)` 依赖 `len(dataset)`，对 `IterableDataset` 会直接抛 `TypeError`，此时训练循环只能用固定步数或 `while` 加计数。判断一个 loader 是否被正确消费还有一个笨办法：在两个 epoch 里各取一个 batch 打印第一维大小，若最后一个 batch 明显小于设定值，说明 `drop_last` 没开。

三个最容易被误解的参数值得单独说。`pin_memory` 指锁页内存（pinned memory，不会被操作系统换出到磁盘的一块主机内存），GPU 的 DMA 引擎可以直接从这块内存读取数据，省掉一次中间拷贝；它只对 GPU 训练有意义，纯 CPU 训练设 True 纯属浪费内存。`drop_last` 除了回避 BatchNorm 在单样本批上的报错，还让每个 epoch 的步数固定下来，对按 step 计数的学习率调度与日志对齐都有影响。`persistent_workers` 保留的是 worker 进程而不是数据，它省掉的是每个 epoch 启进程、重建索引的开销，`__init__` 越慢的数据集收益越大。

### 5.4 collate_fn：把样本列表拼成 batch

默认拼批规则只覆盖简单情况：张量沿第 0 维堆叠、Python 数字转张量、字符串原样放进 list、dict 按 key 递归合并。一旦样本长度不一致，默认规则会直接报 `stack expects each tensor to be equal size`，这时就要自定义拼批函数。

```python
import torch


def pad_collate(batch: list[dict]) -> dict:
    """变长序列拼批：右侧补 pad，同时生成 attention mask。"""
    seqs = [torch.tensor(b["input_ids"], dtype=torch.long) for b in batch]
    lengths = torch.tensor([len(s) for s in seqs])
    # batch_first=True 才是 [B, L]，默认是 [L, B]，方向弄反不会报错但结果全错
    input_ids = torch.nn.utils.rnn.pad_sequence(
        seqs, batch_first=True, padding_value=0
    )
    mask = torch.arange(input_ids.size(1))[None, :] < lengths[:, None]
    labels = torch.tensor([b["label"] for b in batch], dtype=torch.long)
    return {"input_ids": input_ids, "attention_mask": mask.long(), "labels": labels}
```

拼批函数在 worker 进程里执行，所以它必须是模块顶层函数（可被 pickle），不能是 lambda 或闭包，否则在 Windows 或 `spawn` 模式下会抛 `Can't pickle local object`。

它还承担统一类型的责任。默认规则下 Python 的 `int` 会变成 `int64` 张量，字符串会原样存进 list，若某个样本的 `label` 来自 numpy 或被写成了浮点，拼出来的类型就可能与下游 loss 期待的不一致，报 `expected scalar type Long but found Float`。在 collate 里显式指定 dtype，比在训练循环里到处 `float()` 更省事。

另一条隐性约束是同一批样本的字段必须一致：有的样本带 `attention_mask`、有的不带，默认规则会把缺失字段悄悄补齐成长度不一的列表，或者直接报 key 不存在。让 Dataset 在所有分支上都返回相同的 key 集合，是让拼批保持可预测的前提。

### 5.5 transforms 与数据增强

变换（transform，作用在单条样本上的可调用对象，典型如随机裁剪、翻转、归一化）属于 Dataset 的内部实现细节，由 `__getitem__` 调用。这意味着增强发生在 worker 进程里，天然并行；也意味着增强的随机性来源在每个 worker 里各自独立，需要 `worker_init_fn` 才能复现。

```python
from torchvision import transforms as T

train_tf = T.Compose([
    T.RandomResizedCrop(224, scale=(0.6, 1.0)),   # 先裁再缩放，比先缩放再裁更抗形变
    T.RandomHorizontalFlip(),
    T.ColorJitter(0.2, 0.2, 0.2),
    T.ToTensor(),                                 # 同时把 0~255 归一到 0~1
    T.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
])
val_tf = T.Compose([T.Resize(256), T.CenterCrop(224), T.ToTensor(),
                    T.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225])])
```

三条纪律：训练集用随机增强、验证与测试集只用确定性变换；`Normalize` 的均值方差必须与预训练权重一致，换成自建数据集时要重算；分割、检测任务不能用只操作图像、不同步变换标注的旧接口，应改用 `torchvision.transforms.v2`，它能把图像与 mask、bbox 一起变换。

增强强度本身就是超参数。裁剪比例、颜色抖动幅度开得过大时，会出现训练损失照常下降而验证指标停滞甚至倒退，本质是增强改坏了语义。排查办法很简单：留一个小验证集，分别跑「不加增强」与「加增强」两版各 2 个 epoch，若后者验证准确率显著更低，先降强度，不要急着换模型。另一个常见错误是把 `ToTensor` 与 `Normalize` 的顺序写反，或者忘记 `ToTensor`，此时 `Normalize` 会作用在 `uint8` 图像上，数值被截断到 0 与 1，训练一开始 loss 就异常，报错却不会出现。

当增强的算力开销超过模型本身时，可以把一部分变换搬到 GPU 上做：collate 只负责拼成 `uint8` 张量并 pin 到锁页内存，训练循环里再用 `torchvision.transforms.v2` 的 GPU 版本做裁剪与翻转。这样省掉的是 CPU 侧的重复计算与进程间数据拷贝，但会占用显存，小显存卡上要权衡。

### 5.6 多进程加载的五个真坑

`num_workers > 0` 时，PyTorch 用 `multiprocessing` 启动子进程。Linux 默认 `fork`，子进程直接继承父进程内存；Windows 与 macOS 默认 `spawn`，子进程重新 import 主模块再重建 Dataset。这个差异导致了大部分跨平台问题：

- **启动代码必须在 `if __name__ == "__main__":` 里**。`spawn` 会重新执行主模块，若构造 DataLoader 的代码暴露在模块顶层，子进程又会构造 DataLoader，形成进程递归，报 `RuntimeError: An attempt has been made to start a new process before the current process has finished its bootstrapping phase`。
- **Dataset 必须可 pickle**。持有 sqlite3 连接、h5py 文件句柄、lambda、线程锁的 Dataset 在 `spawn` 下会直接失败；正确做法是在 `__init__` 里只存路径，在 `__getitem__` 里按 worker 惰性打开句柄。
- **内存暴涨**。每个 worker 各有一份 Dataset 副本，`spawn` 下是完整拷贝而非共享，图片解码缓冲与预取队列还会再放大；worker 数设成 CPU 核数 2 倍是常见错误。定位方法是同时看 `top` 的 RES 与 Queue 深度，把 `prefetch_factor` 降到 1、`num_workers` 减半，若内存随之下降即为预取积压。
- **死锁与卡住**。在 Dataset 或 collate 里调用 CUDA、在 worker 里使用父进程创建的锁、`timeout=0` 时某个 worker 因读到坏文件而静默退出，都会表现为训练停在某个 step 不动。给 `timeout` 一个非零值，就能把「永久挂起」换成一条可读的报错。
- **线程超订**。每个 worker 里的 PyTorch 与 numpy 默认会按物理核数开 OpenMP 线程，8 个 worker × 16 线程会在 16 核机器上产生上百个线程互相抢占，表现为 CPU 跑满但吞吐不涨。设环境变量 `OMP_NUM_THREADS=1`、`MKL_NUM_THREADS=1` 再试一次，吞吐通常会明显回升。

随机种子的传递也有讲究：`worker_init_fn` 在每个 worker 启动时被调用一次，需要在里面同时设置 `torch`、`random`、`numpy` 三套种子；批顺序的随机性则由 `generator` 参数控制。只有这三处都固定，两次运行切出的 batch 才能逐元素一致，否则「同样的种子、跑出不同的 loss」几乎必然发生在数据侧。

```python
import torch


def seed_worker(worker_id: int) -> None:          # worker_id 从 0 开始
    import random
    import numpy as np
    base = torch.initial_seed() % 2**32            # 由生成器派生的基种子
    random.seed(base + worker_id)
    np.random.seed(base + worker_id)
```

```python
import torch
from torch.utils.data import DataLoader, Dataset


class MyDataset(Dataset):
    def __init__(self, path: str):
        self.path = path          # 只存路径，不在这里 open，避免 spawn 拷贝句柄

    def __len__(self) -> int:
        return 10000

    def __getitem__(self, i: int) -> dict:
        return {"x": torch.randn(3, 224, 224), "y": i % 10}


def main() -> None:               # 所有 DataLoader 构造都在函数内，受 __main__ 保护
    loader = DataLoader(MyDataset("data"), batch_size=32, num_workers=4, timeout=60)
    for step, batch in enumerate(loader):
        if step == 0:
            print(batch["x"].shape)


if __name__ == "__main__":        # Windows 上缺这一行必然报错
    main()
```

### 5.7 流式加载：放不进内存的数据怎么喂

当数据集大到单机磁盘都吃紧，或者要从对象存储实时拉取时，把数据打包成分片（shard，一个包含上万条样本的 tar 或二进制文件）再流式读取，比逐个小文件随机读快一个量级：顺序 IO 带宽高、打开文件次数少、元数据开销可忽略。代价是没有全局随机性，只能用缓冲池做近似打乱。

```python
import random


def stream_shuffle(pattern_files, buffer_size=10000, skip=0):
    """流式近似打乱 + 断点续训：skip 为已消费条数。"""
    buf, seen = [], 0
    for path in pattern_files:
        with open(path, encoding="utf-8") as f:
            for line in f:
                if seen < skip:          # 续训时跳过已消费样本
                    seen += 1
                    continue
                seen += 1
                if len(buf) < buffer_size:
                    buf.append(line)
                    continue
                i = random.randrange(buffer_size)   # 蓄水池式替换，局部乱序
                yield buf[i]
                buf[i] = line
    random.shuffle(buf)
    yield from buf
```

缓冲池越大乱序性越好，但内存线性增长，常见取值 1 万到 10 万条。若需要精确的 epoch 语义与恢复点，应使用支持分片清单与状态保存的格式（如 WebDataset），把「分片列表 + 分片内偏移」作为 checkpoint 的一部分存下来。

流式方案不是默认选择。判断标准很直接：数据集能否在 30 秒内顺序读完一遍。能，就用 map-style 加随机下标，随机性与调试便利都更好；不能，或者数据在对象存储、需要边下载边训练，才值得引入分片与清单管理。另外，流式加载下每个 worker 的样本数不再天然均衡，验证集指标要按实际消费条数计算，不能直接用 `len(loader) × batch_size`。

介于两者之间的做法是内存映射（memory-mapped file，把磁盘文件映射进进程地址空间，让操作系统按页调度）：把样本顺序写进一个二进制文件，另存一份「下标 → 字节偏移」的索引，`__getitem__` 里用 `numpy.memmap` 或 `torch.from_file` 只把需要的那一段读进内存。它既有随机访问，又不需要把整个数据集读进物理内存，代价是要求样本定长或索引足够精确，且随机小读在机械盘上会很慢。

### 5.8 加载慢与加载崩的排查速查表

| 现象 | 大概率原因 | 处理动作 |
| --- | --- | --- |
| GPU 利用率低、step 时间被数据主导 | `num_workers` 太小、增强太重、单文件随机读 | 加 worker、把增强换成 GPU 侧或预计算、转分片顺序读 |
| 第一个 batch 等很久，之后正常 | worker 启动与首批预取开销 | 设 `persistent_workers=True`，属正常现象无需修 |
| 训练一会儿报 `DataLoader worker (pid xxx) is killed` | worker 内存超限被系统 OOM Killer 杀掉 | 减 `num_workers`、降低 `prefetch_factor`、检查增强是否有大图 |
| Windows 上 `num_workers>0` 直接崩 | 缺 `if __name__ == "__main__"`、Dataset 不可 pickle | 加主模块保护、把 Dataset 定义在可导入模块顶层 |
| `RuntimeError: stack expects each tensor to be equal size` | 样本长度不一致且用了默认 collate | 写自定义 `collate_fn` 做 padding |
| 报 `pin_memory` 相关错误或显存访问变慢 | CPU 版 torch、或锁页内存耗尽 | 确认 CUDA 版、减小 batch 或关掉 `pin_memory` |
| 训练卡在某个 step 不动 | worker 死锁、共享句柄冲突、坏样本阻塞 | 设 `timeout=60`、换成 `num_workers=0` 复现、逐条验证样本 |

### 5.9 常见坑

- **在 `__init__` 里把整个数据集读进内存**：`num_workers>0` 时每个 worker 都会各自执行一次 `__init__`，8 个 worker 就是 8 倍内存，且启动时间成倍增加。只存路径与索引，把读取延后到 `__getitem__`。
- **同时传 `shuffle=True` 和 `sampler`**：DataLoader 会直接抛 `ValueError: sampler option is mutually exclusive with shuffle`，而用 `batch_sampler` 时连 `batch_size`、`shuffle`、`drop_last` 都不允许再传。
- **用 `len(loader)` 当训练步数却忘了 `drop_last`**：最后一个不完整 batch 会让 BatchNorm 在只剩一条样本时报 `Expected more than 1 value per channel`，也让学习率调度按错误的步数前进。训练集设 `drop_last=True`。
- **调试时把断点打在 `__getitem__` 里**：`num_workers>0` 时该方法在子进程中执行，主进程调试器进不去，表现为断点不生效。调试数据逻辑先把 `num_workers` 设为 0，确认逻辑正确后再开多进程看吞吐。

## 6. 训练循环与优化

训练循环（Training Loop，把取数据、前向传播、算损失、反向传播、更新参数这几步按固定顺序反复执行的代码骨架）是 PyTorch 项目里唯一真正决定结果的部分。模型结构写得再漂亮，只要循环里顺序错一处——在 `backward()` 之后才清梯度，或者验证时忘了切 `eval()`——训练照样跑完、loss 照样在降，产出的权重却不可用，而且这种错误不会抛出任何异常。把循环写成只有一份实现、其余地方都调用它，是避免这类静默错误最有效的手段。

### 6.1 标准训练循环骨架

一个可复用的单轮训练函数由九个动作组成，顺序固定：置为训练模式 → 清空梯度 → 前向 → 算损失 → 反向 → 裁剪梯度 → 更新参数 → 更新学习率 → 累计指标。清空梯度必须在反向之前，因为 PyTorch 的 `backward()` 是把新梯度**累加**进 `param.grad` 而不是覆盖它，忘了清零的结果是梯度随步数线性增长，几十步后参数直接变成 NaN。更新学习率放在参数更新之后，是因为绝大多数调度器按「已完成的步数」计算学习率，提前调用会让第一步就用上第 1 步的学习率。

```python
import torch
from torch.utils.data import DataLoader


def train_one_epoch(model, loader, loss_fn, optimizer, scheduler, device, clip=1.0):
    model.train()                             # 训练模式：BN 用批统计量，Dropout 生效
    loss_sum, seen = 0.0, 0
    for step, (x, y) in enumerate(loader):
        x = x.to(device, non_blocking=True)   # 非阻塞拷贝，需配合 DataLoader(pin_memory=True)
        y = y.to(device, non_blocking=True)

        optimizer.zero_grad(set_to_none=True)  # 清上一轮梯度；置 None 比填 0 少一次写显存
        logits = model(x)
        loss = loss_fn(logits, y)

        loss.backward()                        # 梯度累加进 param.grad，不清零会一直相加
        if clip and clip > 0:
            torch.nn.utils.clip_grad_norm_(model.parameters(), clip)
        optimizer.step()                       # 用梯度改参数
        scheduler.step()                       # 逐 step 型调度器放这里，按 epoch 型放循环外

        loss_sum += loss.item() * y.size(0)    # 按样本数加权，最后一个不满的 batch 才不拉偏均值
        seen += y.size(0)
    return loss_sum / seen
```

四个容易忽略的细节。第一，`y.size(0)` 加权是必须的：直接对每批 loss 取算术平均，会让最后一个只装 8 条样本的 batch 和前 99 条样本的 batch 等权，验证集上的均值会系统性偏高。第二，每步调 `loss.item()` 会强制 CPU 等待 GPU 同步，步数上万时开销可观，训练中改为每 10 到 50 步记录一次即可。第三，`zero_grad(set_to_none=True)` 把 `grad` 置为 `None` 而不是全零张量，省一次显存写带宽，也让「这一步是否算过梯度」变成可判定的状态。第四，`non_blocking=True` 只有在 `DataLoader(pin_memory=True)` 时才真正异步，否则参数无效。

外层按 epoch 组织，验证与保存按固定间隔触发：

```python
for epoch in range(start_epoch, epochs):
    train_loss = train_one_epoch(model, train_loader, criterion, opt, sched, device)
    if (epoch + 1) % eval_every == 0:            # 每 N 轮验证一次，省下验证开销
        val_loss, val_acc = evaluate(model, val_loader, criterion, device)
        if val_acc > best_acc:                   # 只在指标提升时落盘，避免磁盘被占满
            best_acc = val_acc
            save_ckpt(ckpt_path, model, opt, sched, scaler, epoch, best_acc)
    print(f"epoch {epoch} train {train_loss:.4f} lr {sched.get_last_lr()[0]:.2e}")
```

要做梯度累积（Gradient Accumulation，多个小 batch 累加梯度再更新一次，用时间换显存）时，改动只有三处：`loss` 先除以累积步数、`optimizer.step()` 与 `scheduler.step()` 包在 `if (step + 1) % accum == 0` 里、`zero_grad()` 挪到更新之后而不是循环开头。漏掉第一处，等效学习率就被放大了 `accum` 倍，现象是「换成大 batch 后完全训不动」。

### 6.2 损失函数的选择

损失函数（Loss Function，衡量预测与标签差距、并据此产生梯度的标量函数）选错的典型后果不是报错，而是「loss 不降」或「loss 降但指标不涨」。判断只看两点：任务形态决定用哪个函数，输入形态决定该往里传什么。

| 任务 | 推荐函数 | 传入什么 | 最常见错误 |
| --- | --- | --- | --- |
| 单标签多分类 | `nn.CrossEntropyLoss` | 原始 logits `(N, C)` 与类别索引 `(N,)` | 先手动 `softmax` 再传，等于做了两次 |
| 二分类 / 多标签 | `nn.BCEWithLogitsLoss` | logits 与同形状 float 标签 | 用 `BCELoss` 却忘了自己 `sigmoid` |
| 回归 | `nn.MSELoss` / `nn.SmoothL1Loss` | 同形状浮点张量 | 目标未归一化，梯度量级失控 |
| 语言模型 | `nn.CrossEntropyLoss` | 摊平的 `(N*T, V)` 与 `(N*T,)` | 维度顺序是 `(N, C, T)` 时忘了 `permute` |
| 类别不平衡 | 同上加 `weight=` / `pos_weight=` | — | 权重设得过大，直接训崩 |

```python
criterion = torch.nn.CrossEntropyLoss(label_smoothing=0.1, ignore_index=-100)
loss = criterion(logits, target)          # 多分类：内部已含 softmax，喂 raw logits，(B, C) / (B,)
loss = criterion(logits.reshape(-1, logits.size(-1)),
                 tokens.reshape(-1))      # 语言模型：(B, T, V) 摊平成 (B*T, V)，标签同步摊平
```

`label_smoothing` 把目标从 one-hot 软化成 `(1-ε)` 的分布，是抑制过拟合与过度自信最便宜的手段，分类任务给 0.05 到 0.1 通常白赚；`ignore_index=-100` 让 padding 位置不参与损失，序列任务必备。做类别不平衡处理时，`weight` 是按类给的乘数、`pos_weight` 是正样本相对负样本的权重比，两者都给得太大时会把梯度抬到裁剪阈值之上，等效于一直在裁剪，此时应先降学习率再看效果。另外记住 `reduction` 默认 `'mean'`，做梯度累积时要自己再除一次累积步数，否则梯度尺度被放大。

还有一类隐蔽问题：loss 与业务指标不是一回事。交叉熵在类别极度不平衡时可以被「全预测多数类」压到很低，而准确率看起来是 99%，召回率却是 0。选损失函数时先问一句「这个标量降下去，业务指标会一起降吗」，不会就要换加权损失或改评估口径。

### 6.3 优化器与参数分组

优化器（Optimizer，用梯度计算参数更新量的算法）之间的真正差别，是「用了多少历史信息」和「每个参数额外占多少显存」。

| 优化器 | 每参数额外状态 | 关键超参 | 适用与代价 |
| --- | --- | --- | --- |
| `SGD` | 0 | `lr` | 极省显存，收敛慢，需精调学习率 |
| `SGD(momentum=0.9)` | 1 份（动量） | `lr`、`momentum` | 视觉任务大 batch 常出最好泛化 |
| `Adam` | 2 份（一阶、二阶矩） | `lr=1e-3`、`betas`、`eps` | 快速拿到基线，显存约为 SGD 的 3 倍 |
| `AdamW` | 2 份 | `lr=3e-4`、`weight_decay=0.01~0.1` | Transformer 与大模型默认选择 |

`Adam` 与 `AdamW` 的分歧在权重衰减（Weight Decay，把参数绝对值往零拉以抑制过拟合的正则项）的实现位置。`Adam` 把 L2 惩罚加进梯度，这个惩罚随后会被自适应学习率按参数各自缩放，结果是梯度大的参数反而衰减得少，正则强度与参数尺度绑在一起；`AdamW` 把衰减项从梯度里摘出来，直接乘在参数上，正则强度与梯度尺度解耦。参数量大、训练步数多时，两者的差别会明显体现在验证集曲线上，选 `AdamW` 更省心。

权重衰减不该一视同仁地施加到所有参数上：偏置与归一化层的缩放参数被拉向零会直接削掉模型的表达能力。标准做法是参数分组，把二维以上的权重矩阵和一维参数分开：

```python
decay, no_decay = [], []
for name, p in model.named_parameters():
    if not p.requires_grad:
        continue
    # 一维参数（bias、LayerNorm/BatchNorm 的 weight）不参与权重衰减
    if p.ndim < 2 or name.endswith(".bias"):
        no_decay.append(p)
    else:
        decay.append(p)

optimizer = torch.optim.AdamW(
    [{"params": decay, "weight_decay": 0.1},
     {"params": no_decay, "weight_decay": 0.0}],
    lr=3e-4, betas=(0.9, 0.95), eps=1e-8,
)
```

三个超参的经验值：`betas` 的两个数分别控制一阶矩与二阶矩的滑动窗口，`(0.9, 0.99)` 是默认，长序列或大模型常用 `(0.9, 0.95)` 让二阶矩跟得更快；`eps` 是防止除零的平滑项，`1e-8` 对 fp32 够用，训练早期梯度极小时调大到 `1e-6` 更稳；学习率的数量级上，从头训 CNN 用 `1e-1`（SGD）或 `1e-3`（Adam），微调预训练模型用 `1e-5` 到 `1e-4`，全参数微调要比新增的分类头小一个数量级，否则预训练权重会被几轮学习率淹没。

一条硬约束：先 `model.to(device)`，再构造优化器。优化器持有的是参数对象的引用，先把参数搬到 GPU 再交给优化器，可以避免「优化器更新了一份、前向用的是另一份」这类极难定位的错乱。显存预算也要提前算：`Adam` 给每个参数额外存两份 fp32 状态，10 亿参数就是 8 GB，这部分不随混合精度缩小。

### 6.4 学习率调度与 warmup

学习率调度器（LR Scheduler，按训练进度调整学习率的策略）的核心区别不在公式，而在调用频率：按 epoch 调还是按 step 调。位置错了不会报错，只会让曲线变成一条与预期完全不同的折线。

| 调度器 | 调用频率 | 特点 |
| --- | --- | --- |
| `StepLR` | 每 epoch | 每 `step_size` 轮乘一次 `gamma`，简单粗暴 |
| `MultiStepLR` | 每 epoch | 指定里程碑处衰减，可控性强 |
| `CosineAnnealingLR` | 每 epoch 或每 step | 余弦曲线平滑降到 `eta_min` |
| `OneCycleLR` | 必须每 step | 先升后降，`total_steps` 算错就全乱 |
| `LambdaLR` | 每 step | 用函数自定义，可组合预热与退火 |
| `ReduceLROnPlateau` | 每 epoch，需传指标 | 指标不降才降，调用要传 `val_loss` |

大模型训练几乎都用「线性预热 + 余弦退火」。预热（Warmup，训练最初若干步把学习率从接近零线性升到目标值）的作用是让 Adam 的二阶矩估计先积累出可靠的统计量：这个估计由全零初始化，前几步的 `v` 极小，除以 `sqrt(v)` 会把梯度放大成巨大的更新，跳过预热直接上大学习率，常把权重打成 NaN。实践中预热步数取总步数的 1% 到 5%，数据越脏、batch 越大，取值越靠上。

```python
import math

total_steps = len(loader) * epochs
warmup_steps = max(1, len(loader) * 2)      # 前两个 epoch 线性升温


def lr_lambda(step):
    if step < warmup_steps:
        return (step + 1) / warmup_steps
    progress = (step - warmup_steps) / max(1, total_steps - warmup_steps)
    return 0.5 * (1.0 + math.cos(math.pi * progress))   # 余弦从 1 平滑降到 0


scheduler = torch.optim.lr_scheduler.LambdaLR(optimizer, lr_lambda)
print(scheduler.get_last_lr())               # 确认当前学习率的唯一可靠方式
```

`LambdaLR` 乘的是优化器的基准 `lr`，构造时就会按 `lr_lambda(0)` 设定初始值，所以学习率的起点是 `lr / warmup_steps` 而不是 `lr`，看到第一步学习率极小属于正常现象。`get_last_lr()` 返回的是列表，参数分组时里面有多个数，只取 `[0]` 会漏掉其他组的实际值——排查「某些层没在学」时应当打印整个列表。如果只想快速拿到效果、不想写公式，`OneCycleLR` 一行覆盖「预热 + 退火 + 收尾降到底」，代价是必须传准 `total_steps` 并严格按 step 调用一次；它内部已含预热，再叠加一层 `LambdaLR` 预热会得到先降后升的怪曲线。

另外要区分「按 epoch 调度的调度器放在 epoch 循环末尾」这条规则：`CosineAnnealingLR` 如果按 epoch 调，`T_max` 就要传给 `epochs`；按 step 调则传给 `total_steps`。两种写法都能用，但混着写会得到一半周期就归零的学习率。

### 6.5 train/eval 与 BN、Dropout 的行为差异

`model.train()` 与 `model.eval()` 递归设置所有子模块的 `training` 标志，它改变的是层的行为，不是梯度的开关。批归一化（Batch Normalization，用当前 batch 的均值方差归一化特征，并维护一份滑动平均供推理使用）在训练模式下用批统计量并更新 `running_mean`、`running_var`，在评估模式下改用滑动平均值；Dropout 在训练模式下按概率随机置零并把保留值放大 `1/(1-p)`，在评估模式下退化为恒等映射。

由此产生两个后果。第一，评估若忘了 `eval()`，Dropout 会让同一张图片连续两次推理得到不同结果，BN 还会用当前 batch 的统计量去归一化，同时把验证数据混进滑动平均——表现为验证指标每轮抖动、最优轮次完全靠运气、复现实验时结论对不上。第二，评估忘了 `no_grad()` 只是白吃显存：梯度照样建图，验证 batch 一大就 OOM，而且这些梯度永远不会被 `step()` 用掉。

```python
@torch.no_grad()                          # 不建计算图，验证显存约为训练的三分之一
def evaluate(model, loader, loss_fn, device):
    model.eval()                          # BN 用滑动统计量，Dropout 关闭
    loss_sum, seen = 0.0, 0
    for x, y in loader:
        x, y = x.to(device), y.to(device)
        loss_sum += loss_fn(model(x), y).item() * y.size(0)
        seen += y.size(0)
    model.train()                         # 防御性切回，避免调用方漏了这一步
    return loss_sum / seen
```

还有三个边界值得记住。`torch.inference_mode()` 比 `no_grad()` 更快，但它产出的张量不能再参与需要梯度的运算，只适合纯推理。BN 的滑动平均更新系数 `momentum` 默认 0.1，意味着统计量大约由最近 10 个 batch 主导，小 batch（如 2、4）训练检测或分割模型时这个估计噪声大到几乎无法收敛，此时换 `GroupNorm` 或 `LayerNorm` 才是正解，调学习率救不回来。最后，`eval()` 与 `no_grad()` 是两件独立的事，缺任何一个都会出问题，把它们一起写进评估函数内部，比依赖调用方记得切换可靠得多。

### 6.6 梯度裁剪与梯度监控

梯度裁剪（Gradient Clipping，当梯度范数超过阈值时按比例整体缩小）是防梯度爆炸最直接的手段。它只该出现在两个位置之间：`backward()` 之后、`optimizer.step()` 之前。放在反向之前等于裁了个空，放在更新之后则这一轮已经用脏梯度改过参数了。

```python
scaler.scale(loss).backward()             # 混合精度下 loss 先放大，梯度也带着放大因子
scaler.unscale_(optimizer)                # 必须先还原梯度尺度，否则裁剪阈值失效
grad_norm = torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
if not torch.isfinite(grad_norm):
    optimizer.zero_grad(set_to_none=True)  # 遇到坏 batch 直接跳过，别让它污染权重
else:
    scaler.step(optimizer)
scaler.update()
```

`clip_grad_norm_` 的返回值是裁剪前的全局范数，这个值比 loss 更能说明训练健康度：正常训练中它应在个位数到几十之间小幅波动；突然跳到几百，说明这个 batch 的数据有问题或学习率过大；长期贴在阈值上，说明裁剪一直在生效，模型收到的是被压扁的梯度方向，此时应先查数据与学习率，而不是把阈值调大。按范数裁剪优于按值裁剪（`clip_grad_value_`），因为范数保留了各参数之间的相对比例，按值裁会把大梯度和中等梯度一起压成同一个数。

开启混合精度（AMP，用 fp16/bf16 计算以省显存提速）时有一个固定顺序：必须先 `unscale_` 再裁剪。跳过它就是在裁剪被 `GradScaler` 放大过的梯度，`max_norm=1.0` 实际上裁到了真实范数的 `1/scale`，阈值等于失效。这个过程不会抛任何异常，`GradScaler` 只根据梯度里有没有 inf/nan 决定是否跳过更新，所以只能靠日志里 `grad_norm` 的量级异常（恒等于阈值，或大得离谱）发现。

### 6.7 Checkpoint 与断点续训

检查点（Checkpoint，把恢复训练所需的全部状态落盘的文件）只存 `state_dict()`，不要用 `torch.save(model)` 存整个对象：后者把类定义路径一并 pickle 进去，换目录、改类名就加载失败，还存在反序列化执行任意代码的安全风险。

```python
import os


def save_ckpt(path, model, optimizer, scheduler, scaler, epoch, best):
    tmp = path + ".tmp"
    torch.save({
        "model": model.state_dict(),
        "optimizer": optimizer.state_dict(),   # 含动量与 Adam 的 m、v，缺了等于冷启动
        "scheduler": scheduler.state_dict(),   # 缺了学习率会从预热起点重来
        "scaler": scaler.state_dict(),
        "epoch": epoch, "best": best,
        "rng": torch.get_rng_state(),
        "cuda_rng": torch.cuda.get_rng_state_all(),
    }, tmp)
    os.replace(tmp, path)                      # 原子替换，断电不会留下半个文件
```

保存用「先写临时文件再改名」的原子写法，训练几小时后遇到断电或进程被杀，不至于只剩一个 0 字节的 checkpoint。加载用 `torch.load(path, map_location="cpu")` 再 `model.load_state_dict(ckpt["model"])`，可以在没有 GPU 的机器上读取显卡上存下的文件；`load_state_dict` 默认 `strict=True`，层名对不上会直接报错，这比静默加载一部分参数安全。

保存策略建议「最近 N 个 + 最优 1 个」。只留最优的，后期指标震荡时没有可回退的点；只留最近的，早期偶然出现的最优权重会被覆盖。恢复训练后要确认三件事：`epoch` 从断点续、`scheduler` 的步数对得上、`best` 指标没有被重置。漏了 `scheduler` 状态最常见，现象是「续训后学习率突然变大、指标先掉一截再爬回来」，看起来像模型被破坏了，其实是调度器从预热起点重来。

### 6.8 日志指标与随机种子

训练的日志至少覆盖五类量：训练 loss 的滑动平均、验证指标、当前学习率、梯度范数、吞吐（每秒样本数或每秒 token 数）。只看 loss 会漏掉「loss 正常但学习率已经掉到 1e-8」这类隐性停滞；只看吞吐会漏掉「速度没变但梯度范数在缓慢攀升」这类爆炸前兆。指标口径也要写清楚：吞吐按样本算还是按 token 算、loss 是按 batch 平均还是按样本加权，口径不一致的两条曲线放在同一张图上，会得出完全错误的结论。

随机种子（Random Seed，伪随机数发生器的初始值，相同种子产生相同序列）必须覆盖所有随机源，只设 `torch.manual_seed` 远远不够：

```python
import random

import numpy as np
import torch


def set_seed(seed: int = 42, deterministic: bool = True):
    random.seed(seed)
    np.random.seed(seed)
    torch.manual_seed(seed)                  # CPU 与当前 GPU
    torch.cuda.manual_seed_all(seed)         # 多卡时每张卡都要设
    if deterministic:
        torch.backends.cudnn.deterministic = True
        torch.backends.cudnn.benchmark = False   # 关掉自动挑最快卷积算法，它会引入不确定
        torch.use_deterministic_algorithms(True, warn_only=True)


def worker_init_fn(worker_id: int):
    s = (torch.initial_seed() + worker_id) % 2**32
    np.random.seed(s)
    random.seed(s)                           # 每个 DataLoader 子进程独立且确定的种子
```

`DataLoader` 这边还要显式给 `generator=torch.Generator().manual_seed(seed)`，否则 `shuffle=True` 的随机顺序来自全局状态，换进程就复现不了；`num_workers>0` 时不给 `worker_init_fn`，数据增强的随机性就与主进程脱钩，两次跑出不同的增强序列。顺带一提，`cudnn.benchmark=True` 与 `cudnn.deterministic=True` 是互斥的取舍：前者自动挑最快的卷积算法、每次启动耗时略长且结果不稳定，后者保证同一输入同一输出但可能慢 10% 到 20%。

可复现也有边界：换 GPU 型号、换 cuDNN 或驱动版本、改变多卡通信的归约顺序，都可能让最后几位小数不同，进而在几千步后放大成完全不同的权重。工程上的目标应定为「同机器同配置可复现」，而不是「任何环境完全一致」；`torch.use_deterministic_algorithms(True, warn_only=True)` 会在遇到没有确定性实现的算子时给出警告而不中断，这既是提示也是性能损耗的来源，正式长跑时可以关掉。

### 6.9 常见坑

- **`zero_grad()` 写在 `backward()` 之后**：梯度在参数上不断累加，等效学习率越来越大，几百步后直接 NaN，而日志上前期完全正常。固定顺序是「清梯度 → 前向 → 反向 → 更新」，只写一份实现，不要在多处手写。
- **`scheduler.step()` 位置与调度器类型不匹配**：把 `OneCycleLR` 放在 epoch 循环末尾，它只被调用 `epochs` 次而 `total_steps` 是 `len(loader)*epochs`，学习率曲线被压成粗糙折线；反过来把 `StepLR` 每 step 调一次，学习率会按步数狂降。判断依据只有一个：这个调度器的总步数是按什么单位算的。
- **验证阶段忘了 `model.eval()`**：指标抖动、最优轮次随机、验证数据污染 BN 的滑动统计量，且这些现象在训练 loss 上看不出来。把 `model.eval()` 和 `@torch.no_grad()` 一起写进评估函数内部，比靠调用方记得切换可靠。
- **续训时只加载了模型权重**：优化器状态与调度器状态缺失，动量、Adam 矩估计、学习率进度全部归零，表现为「续训后指标先掉一截」。checkpoint 里少存一样，续训就不是续训，而是带初始权重的重新开始。

## 7. 显存与性能优化

把 batch size 从 4 调到 8 就报 `torch.cuda.OutOfMemoryError`，可 `nvidia-smi` 显示显存只占了七成——这类现象说明显存问题很少是"总量不够"，多数是账没算清、分配方式不对。显存优化有三条主线：先把显存账算清楚，知道每一块钱花在哪里；再用混合精度、梯度累积、梯度检查点，让同样的显存装下更大的有效 batch；最后用 profiler 与 compile 把算力利用率提上来。

这三条主线有严格的先后关系。跳过估算直接改 batch，通常是把 8 改成 4 之后发现 GPU 利用率从 85% 掉到 40%，单步时间只少了三成，总训练时间反而变长；跳过 profiler 直接上 `torch.compile`，可能因为图里有 `.item()` 打断而一行收益都拿不到。可复用的顺序是：估算显存预算 → 定位 OOM 属于哪一块 → 选择对应手段 → 用 profiler 确认瓶颈转移 → 再决定是否值得继续压。

### 7.1 显存账本：五块开销与一次估算

显存（GPU Memory，显卡上的独立高速内存，消费级卡 8~24GB，数据中心卡 40~80GB）由五部分构成，其中前四块与参数量成正比，第五块与 batch 和序列长度成正比。

| 组成 | 何时产生 | 每参数字节 | 说明 |
| --- | --- | --- | --- |
| 模型参数 | 初始化或加载权重 | 2 | 前向计算用的权重副本，bf16 |
| 梯度 | 第一次反向传播 | 2 | dtype 与参数一致 |
| 优化器状态 | 构造优化器时 | 8 | AdamW 的一阶矩、二阶矩，fp32 |
| fp32 主权重 | 构造优化器时 | 4 | 避免小梯度更新被舍入成 0 |
| 激活值 | 每次前向 | 与 batch×seq 成正比 | 反向要用，是 batch 的主要约束 |
| 上下文与碎片 | 进程启动、反复分配 | 0.3~1GB 固定开销 | CUDA context、cuDNN workspace、碎片 |

前四项合计约 16 字节/参数，这个数字值得记住，它决定了"模型能不能训"的下限。为什么要单独存一份 fp32 主权重？因为 bf16 只有 7 位尾数，学习率 1e-4 量级的更新作用在 0.1 量级的权重上时，相对变化约 1e-3，已经接近 bf16 的分辨率极限，直接累加会被舍入掉，训练表现为 loss 长时间不动。AdamW 为什么占 8 字节？它要为每个参数保存一阶矩与二阶矩两个 fp32 张量，本身也是主要的显存开销来源，这也是 8bit Adam、Adafactor 这类省显存优化器的立足点。

| 模型规模 | 参数+梯度+优化器状态 | 单卡可行性判断 |
| --- | --- | --- |
| 0.5B | 约 8GB | 24GB 卡可全参微调 |
| 1.5B | 约 24GB | 24GB 卡必须减 batch 或开检查点 |
| 7B | 约 112GB | 单卡放不下，需 ZeRO/FSDP 或只训 LoRA |
| 13B | 约 208GB | 8×40GB 起步 |
| 70B | 约 1.1TB | 多机分片，必须并行策略配合 |

这张表刻意不含激活值，而激活往往才是 OOM 的直接原因。激活可用经验公式估算：未开检查点时，每层约 34 字节/元素（fp16 下含 QKV、注意力矩阵、MLP 中间结果），总激活约 `层数 × 34 × batch × seq × hidden` 字节；开启全量检查点后只保留每层输入，降到约 `层数 × 2 × batch × seq × hidden`。

```python
# 翻译自 Megatron-LM 论文的激活显存经验公式，用于判断 batch 能做到多大
GB = 2 ** 30

def activation_gb(layers, batch, seq, hidden, ckpt=False):
    if ckpt:                      # 全量检查点：只留每层输入，约 2 字节/元素
        return layers * 2 * batch * seq * hidden / GB
    return layers * 34 * batch * seq * hidden / GB   # 不检查点：约 34 字节/元素

# 24 层、hidden=1024 的小模型，seq=2048
print(activation_gb(24, 1, 2048, 1024))             # 约 1.6GB
print(activation_gb(24, 8, 2048, 1024))             # 约 12.8GB，batch 上不去的真凶
print(activation_gb(24, 8, 2048, 1024, ckpt=True))  # 约 0.75GB
```

把两张表叠起来看，结论就很清楚：7B 模型全参微调的权重侧下限是 112GB，比任何单卡的容量都大，所以要么用分片（ZeRO-3、FSDP）把状态切到多卡，要么干脆冻结主干只训 LoRA——LoRA 时基座权重仍按 2 字节/参数常驻，但梯度与优化器状态只作用于占比很小的一两亿个适配器参数，整卡占用能压到十几 GB。推理场景的账本又是另一套：只需参数加上缓存，长上下文时会出现与 `层数 × seq × hidden` 成正比的 KV Cache，它才是推理侧真正的显存吞金兽。

KV Cache 的算法是 `2 × 层数 × batch × seq × kv_dim × 字节数`，前一个 2 表示要同时存 Key 与 Value。以 32 层、kv 维度 1024、bf16、batch 8、seq 4096 为例，单这一项就要约 32GB，比很多模型的权重本身还大。这解释了为什么长上下文推理的并发数上不去，也解释了 MLA、GQA 这类压缩 KV 的结构改动为什么值得做。

### 7.2 混合精度：AMP、bf16 与 GradScaler

混合精度（Mixed Precision，让矩阵乘、卷积这类吞吐敏感的算子用 16 位浮点计算，而 softmax、LayerNorm、损失等数值敏感的算子仍用 32 位的技术）。它是性价比最高的一项优化，通常带来 1.5~2 倍加速与 20%~40% 的显存下降，改动量只有两个上下文管理器加一个 scaler。

| 格式 | 位宽 | 指数位 | 尾数位 | 数值范围 | 是否需要 loss scaling |
| --- | --- | --- | --- | --- | --- |
| fp32 | 32 | 8 | 23 | 约 1e-38 ~ 3e38 | 不需要 |
| fp16 | 16 | 5 | 10 | 约 6e-8 ~ 65504 | 需要，否则小梯度下溢成 0 |
| bf16 | 16 | 8 | 7 | 与 fp32 同阶 | 不需要 |

fp16 的问题出在指数位只有 5 位：可表示的最大值约 65504，最小正规数约 6e-5。训练的梯度普遍在 1e-5 到 1e-7 之间，反传时很容易下溢成 0，参数就再也不更新。GradScaler 的做法是把 loss 乘上一个动态系数（初始 65536），梯度整体放大到安全区间再反传，更新前再除回来；一旦某次反传出现 inf 或 nan，说明放得太大，就把系数减半并跳过这次更新。

```python
import torch

scaler = torch.amp.GradScaler("cuda")          # 只有 fp16 需要缩放，纯 bf16 可设 enabled=False

for batch in loader:
    optimizer.zero_grad(set_to_none=True)
    # autocast 只把白名单算子（matmul、conv、linear）降到 bf16，其余保持 fp32
    with torch.amp.autocast("cuda", dtype=torch.bfloat16):
        loss = model(**batch).loss

    scaler.scale(loss).backward()               # 先把 loss 放大，防止 fp16 梯度下溢
    scaler.unscale_(optimizer)                  # 反缩放，否则裁剪阈值失去意义
    torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)
    scaler.step(optimizer)                      # 检测到 inf/nan 就跳过本次更新并降低 scale
    scaler.update()
```

四条经验。30/40 系与 A100/H100 原生支持 bf16，优先用 bf16，省掉缩放逻辑且几乎不会溢出；T4、V100 只有 fp16，必须配 `GradScaler`，否则训练十几个 step 后 loss 变 NaN。不要在 `autocast` 块里手动 `.half()`，那会绕过算子白名单，把 LayerNorm 也降到半精度，出现 `expected scalar type Float but found Half` 一类报错。断点续训要把 `scaler.state_dict()` 一起存进 checkpoint，否则恢复后 loss scale 从头摸索，前几十步容易震荡。最后，`unscale_` 必须在裁剪之前调用，顺序颠倒的结果是裁剪按放大后的梯度算，阈值形同虚设。

### 7.3 梯度累积：用小显存凑出大 batch

梯度累积（Gradient Accumulation，连续做多次前向与反向、把梯度攒起来再更新一次参数的技术）。数学依据很简单：梯度是线性的，把 8 个 micro batch 的梯度求和再除以 8，结果与用 8 倍大 batch 训一次等价，只是多花了 7 次反传的时间。

```python
accum_steps = 8                        # 等效 batch = micro_batch × 8
for step, batch in enumerate(loader):
    with torch.amp.autocast("cuda", dtype=torch.bfloat16):
        loss = model(**batch).loss / accum_steps   # 先除，梯度才是平均值
    loss.backward()                    # 梯度累加到 param.grad，不清零
    if (step + 1) % accum_steps == 0:  # 攒够一个窗口才更新
        torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)  # 只在 step 前裁一次
        optimizer.step()
        optimizer.zero_grad(set_to_none=True)   # 比 fill_(0) 省一次全量写
```

四个易错点。忘记除以 `accum_steps`，等效学习率会被放大 8 倍，表现是训练前几十步 loss 剧烈震荡甚至发散。梯度裁剪放进每个 micro step，会让裁剪阈值与真实梯度尺度错配，正确做法是只在 `optimizer.step()` 前裁一次。最后一个不完整窗口要有兜底 step，或者给 DataLoader 设 `drop_last=True`，否则尾部样本被静默丢弃。用 DDP 时每个 micro step 都会触发一次 all-reduce，带宽全浪费在中间步骤上，应当用 `with model.no_sync():` 包住非窗口末尾的 backward。

还有两处语义差异需要记住。BatchNorm 按 micro batch 统计均值方差，梯度累积后的等效 batch 与真实大 batch 并不可等价，视觉任务要改用 GroupNorm 或 SyncBN；NLP 模型普遍用 LayerNorm，不受影响。学习率的调整上，梯度累积把等效 batch 放大 k 倍后，按线性缩放规则把学习率也乘 k 起步、再观察若干百步的 loss 曲线决定是否回退，比直接照搬原学习率稳得多。

### 7.4 梯度检查点：用算力换显存

梯度检查点（Gradient Checkpointing，前向时只保存少量中间结果、反向时重新计算被丢弃激活的技术），也叫激活重计算（activation recomputation）。代价明确：反向多算一次前向，训练时间增加约 30%，换来激活显存下降 60%~70%，于是 batch 能放大 2~4 倍，总体吞吐往往反而更高。

```python
import torch
from torch.utils.checkpoint import checkpoint

class Block(torch.nn.Module):
    def __init__(self):
        super().__init__()
        self.attn, self.mlp = Attention(), MLP()
        self.norm1 = self.norm2 = torch.nn.LayerNorm(1024)

    def forward(self, x):
        # 只保留 block 输入，block 内部的中间激活在反向传播时重算
        x = x + checkpoint(self.attn, self.norm1(x), use_reentrant=False)
        x = x + checkpoint(self.mlp, self.norm2(x), use_reentrant=False)
        return x
```

Hugging Face 模型一行即可打开，但配 LoRA 时有额外条件：

```python
model.gradient_checkpointing_enable(
    gradient_checkpointing_kwargs={"use_reentrant": False})  # 新版推荐，兼容 compile
model.enable_input_require_grads()   # 冻结 embedding + 只训 LoRA 时必须，否则梯度链断掉
```

三个坑。`use_reentrant=True` 的旧实现在"部分参数不参与计算图"时会报 requiring grad 相关错误，与 `torch.compile` 也不兼容，新代码一律显式传 `False`。被 checkpoint 的函数内部不要有修改输入的原地操作，重算时输入已经不是原值，梯度会算错。checkpoint 自带的 RNG 状态保存能处理 Dropout，但自定义的随机算子要自己存取 RNG state，否则两次前向的随机结果不同，梯度就对不上。

也要知道什么时候不该用：模型只有几层、序列很短时，重算的固定开销占比高而省下的显存只有几百 MB，收益为负；推理阶段根本不存激活，开检查点纯属浪费。

如果只有个别层的激活格外大，还可以做选择性检查点（selective checkpointing）：只对注意力部分重算、保留 MLP 的激活，时间多花约 10% 就能省掉接近一半的激活显存。判断哪些层值得重算的方法很直接，用 profiler 的显存视图按层排序，把占用最高的几层挑出来，不必整模型一刀切。

### 7.5 batch size 与显存的定量关系

激活显存与 micro batch 近似线性，而注意力部分与序列长度是平方关系——这解释了两个常见现象：batch 翻倍显存大致翻倍，但序列长度从 1024 涨到 4096，显存可能涨到四五倍，batch 只能退到 1。工程上不要靠猜，写一个探测脚本把最大可用 batch 试出来。

```python
import torch

def probe_max_batch(make_batch, model, optimizer, start=1, max_try=64):
    """倍增法探测单卡最大 micro batch，OOM 在 PyTorch 里是可恢复错误"""
    lo, hi = 1, start
    while hi <= max_try:
        try:
            batch = make_batch(hi)                 # 造与真实数据同形状的假数据
            with torch.amp.autocast("cuda", dtype=torch.bfloat16):
                loss = model(**batch).loss
            loss.backward()
            optimizer.step()
            optimizer.zero_grad(set_to_none=True)
            lo, hi = hi, hi * 2                    # 成功就翻倍继续
        except torch.cuda.OutOfMemoryError:
            torch.cuda.empty_cache()               # 释放失败尝试留下的缓存块
            break
    return lo                                      # 真实训练取 lo 的 80%

# 变长样本的长尾要靠分桶解决，而不是靠把 batch 压到 1
```

探测结果要留 10%~15% 的余量，因为真实语料的长度分布在尾部，最长的那几条才会触发 OOM。`drop_last=False` 时最后一个 batch 可能偏小，这不会带来风险，反而说明长度分布不均匀。

比调 batch 更有效的手段是按长度分桶（length bucketing）：把长度相近的样本组进同一个 batch，能把 padding 从 50% 降到 5% 以内。padding 不只是浪费显存，它同样进入矩阵乘，等于白算。当单条样本本身就超过显存时，只能靠分块注意力（FlashAttention 一类实现）把注意力矩阵的计算切成小块，用重算换取不落地完整注意力矩阵。这也解释了为什么同样的 seq 长度，换用 FlashAttention 后显存能从几十 GB 降到几 GB。

### 7.6 torch.profiler：先定位再优化

性能优化最忌凭直觉改代码。`torch.profiler` 能同时采 CPU 与 GPU 的时间线，回答"时间到底花在哪个 kernel 上"这个必须回答的问题。

```python
from torch.profiler import profile, ProfilerActivity, schedule

with profile(
    activities=[ProfilerActivity.CPU, ProfilerActivity.CUDA],
    schedule=schedule(wait=1, warmup=1, active=3, repeat=1),  # 跳过前两步预热
    record_shapes=True,
    profile_memory=True,
) as prof:
    for i, batch in enumerate(loader):
        train_step(batch)
        prof.step()
        if i >= 5:
            break

print(prof.key_averages().table(sort_by="self_cuda_time_total", row_limit=15))
prof.export_chrome_trace("trace.json")   # 拖进 chrome://tracing 看逐流时间轴
```

读表只需对照几个典型结论：

| 现象 | 含义 | 对策 |
| --- | --- | --- |
| GPU 利用率低，小 kernel 密集 | 启动开销吞掉收益 | 增大 batch、用 torch.compile 融合 kernel |
| 大量 `aten::copy_`、`_to_copy` | 主机与设备间反复搬运 | `pin_memory=True` + `non_blocking=True` |
| 时间轴大片空白、H2D 间隔长 | DataLoader 喂不饱 | 加 `num_workers`、`prefetch_factor` |
| 单个 matmul kernel 占 70% | 算力瓶颈在矩阵乘 | 换 bf16、上 FlashAttention、减少 padding |
| CPU 侧 `self_cpu_time` 很高 | Python 解释开销或动态 shape | `torch.compile`、固定输入形状 |

一个真实案例：某次训练单步 320ms，`nvidia-smi` 显示利用率在 35%~50% 之间波动。profile 结果显示 GPU 时间只占 40%，其余是 CPU 侧几百个 `aten::to` 与 Python 循环，根因是每个 step 在 CPU 上做了一次图像解码加一次 dtype 转换。把解码挪到 DataLoader 的 worker 进程、张量一次性 `.to(device, non_blocking=True)` 之后，单步降到 150ms。如果一开始就去调 kernel 或者开 compile，只会优化那 40% 里的一小部分。

查显存归属时加 `profile_memory=True`，按 `self_cuda_memory_usage` 排序，能直接看出是哪一层、哪个算子分配的字节最多，比逐层打印 `shape` 猜要快得多。

### 7.7 torch.compile：一行代码换 10%~30%

`torch.compile` 在运行时捕获计算图，交给 TorchInductor 生成融合后的 kernel，训练与推理都能受益，典型收益 10%~30%，小算子密集的模型更明显。

```python
model = torch.compile(
    model,
    mode="default",        # default / reduce-overhead（CUDA Graphs）/ max-autotune（自动调优）
    fullgraph=False,       # True 时遇到 graph break 直接报错，利于排查
    dynamic=False,         # 输入形状固定时设 False，编译结果最优
)
```

四个前提条件。首次调用要编译几十秒到几分钟，短任务可能亏本，长期训练才划算。Python 里的 `print`、`.item()`、依赖张量值的 `if`、numpy 调用都会打断计算图，用 `TORCH_LOGS="graph_breaks"` 环境变量能看到断点位置与原因。输入形状频繁变化会触发反复重编译，长尾样本多的场景设 `dynamic=True` 或用 `torch._dynamo.mark_dynamic` 标记维度。编译产物落在 `TORCHINDUCTOR_CACHE_DIR`，CI 里持久化这个目录能省掉重复编译。

组合使用的注意点：梯度检查点必须用 `use_reentrant=False`，否则 compile 会在检查点处断图；`mode="reduce-overhead"` 借助 CUDA Graphs，要求输入地址稳定，适合小模型高吞吐场景，遇到动态显存申请容易出错；`max-autotune` 会为每个 matmul 试多种实现，编译时间可达数分钟，只在确定要榨干性能时开。

验证收益要按吞吐量算，不能按单步耗时算。预热若干步（把编译时间排除在外），再用固定步数除以总时间得到每秒样本数，与未编译版本对比；同一组数据跑三轮取中位数，避免被显卡降频干扰。也要留意显存：融合后的 kernel 可能一次性申请更大的缓冲区，原本刚好够用的 batch 在编译后会 OOM，这时把 batch 减一档再测，否则会把 OOM 误判成 compile 不兼容。

### 7.8 OOM 排查流程与显存碎片

报 OOM 时不要立刻改小 batch，按下面顺序走，八成问题在前两步就能定位。

```python
import torch
print(torch.cuda.memory_summary(abbreviated=True))   # 分块统计，看 allocated/reserved/inactive
print(f"allocated={torch.cuda.memory_allocated()/2**30:.2f}GB "
      f"reserved={torch.cuda.memory_reserved()/2**30:.2f}GB "
      f"peak={torch.cuda.max_memory_allocated()/2**30:.2f}GB")
```

1. `nvidia-smi` 看是否已有别的进程占卡，同组同事的 notebook 残留是最常见的"显存凭空消失"。
2. 固定 `batch=1`、最短序列复现。仍然 OOM，说明权重、梯度、优化器状态本身就超了，调 batch 无意义，只能上 ZeRO、LoRA 或换卡。
3. 对比 allocated 与 reserved：两者差距大是碎片；reserved 顶到上限而 allocated 很小，是缓存没还回去。
4. 找泄漏：`total_loss += loss` 会把整张计算图钉在显存里；把 tensor append 进 list 做指标记录；eval 阶段忘了 `torch.no_grad()`；常开 `retain_graph=True`。
5. 逐级降显存：混合精度 → 梯度累积 → 梯度检查点 → 冻结底层或改 LoRA → 优化器状态 offload 到 CPU。
6. 兜底：`PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True` 缓解碎片（2.x 推荐常开），减小 seq、按长度分桶，或换更大的卡。

判断是否存在泄漏，不要靠观察 OOM 迟早发生，直接看 `allocated` 随 step 的走势：正常训练在固定区间内锯齿波动（激活随 batch 反复申请释放），曲线单调上升就是泄漏。要精确定位是哪个张量没被释放，用分配器自带的快照功能记录每次申请的调用栈。

```python
# 每 50 步打一次，正常应锯齿波动；单调上升说明有引用没释放
if step % 50 == 0:
    print(step, f"alloc={torch.cuda.memory_allocated()/2**30:.2f}GB",
          f"reserved={torch.cuda.memory_reserved()/2**30:.2f}GB")

# 需要定位到具体代码行时，录制分配历史再导出快照
torch.cuda.memory._record_memory_history(max_entries=100000)
# ... 跑几十个 step ...
torch.cuda.memory._dump_snapshot("mem_snapshot.pickle")  # 用 pytorch.org/memory_viz 打开
```

关于碎片：PyTorch 的缓存分配器按块管理显存，反复申请释放不同尺寸的块会留下无法利用的空洞，表现为 `reserved` 远大于 `allocated` 却仍 OOM。典型场景是训练中途 eval、样本长度抖动、动态生成不同 shape 的 mask。`expandable_segments:True` 让分配器用可扩展的虚拟内存段，从机制上减少这类空洞，比反复 `empty_cache` 有效。`torch.cuda.empty_cache()` 只把空闲块还给驱动，不解决泄漏，而且每次调用都同步设备，绝不要放进训练循环。

### 7.9 手段对比与组合顺序

单点手段各自解决一块开销，工程上真正要回答的是"我的显存缺口该用哪几个换"。判断依据只有两条：能省下多少显存，以及付出多少时间与改造成本。

| 手段 | 典型显存节省 | 主要代价 | 适用条件 |
| --- | --- | --- | --- |
| bf16/fp16 混合精度 | 20%~40% | 数值风险，fp16 需 scaler | 几乎所有训练，首选 |
| 梯度累积 | 单步不变，等效 batch 变大 | 时间线性增加 | batch 太小导致收敛不稳 |
| 梯度检查点 | 激活降 60%~70% | 时间增加约 30% | 层数多、激活占大头 |
| 冻结主干 / LoRA | 权重侧降 80% 以上 | 表达能力受限 | 下游任务与预训练分布接近 |
| 优化器状态 offload | 每参数 8 字节移到 CPU | 带宽瓶颈，速度降 20%~50% | 单卡极限、勉强跑得动 |
| 按长度分桶 | 10%~40% | 需要改采样逻辑 | 变长样本、padding 多 |

组合顺序有一份可复用的经验：显存缺口在 30% 以内，混合精度加长度分桶基本够用；缺口在 2 倍左右，混合精度 + 梯度检查点 + 梯度累积三件套能顶住；缺口到 10 倍以上（例如单卡全参微调 7B），任何单卡手段都无解，只能转向分片并行或 LoRA，继续在 batch 上抠只会把训练拖慢十倍。

速度侧的顺序恰好相反：先确保数据管道不空转（`num_workers`、`pin_memory`、预取），再开混合精度，再考虑 `torch.compile`，最后才动自定义 kernel。衡量指标统一用吞吐（每秒样本数或每秒 token 数），单步耗时会被 batch 大小误导——batch 减半单步时间往往只减三成，吞吐其实是降的。

### 7.10 常见坑

- 用 `total_loss += loss` 累计日志指标，计算图一直活着，跑上百步必 OOM；写成 `total_loss += loss.item()` 或 `loss.detach()`。
- 推理或评测忘记 `torch.inference_mode()`，中间变量照样建图，峰值显存能比正常推理高两三倍，速度也慢。
- 训练循环里调 `torch.cuda.empty_cache()` 想"省显存"，实际每次同步设备、丢掉可复用块，速度掉一大截而显存并不会少。
- 把 `nvidia-smi` 的占用当成模型真实开销：那里包含缓存与上下文，`torch.cuda.memory_allocated()` 才是当前张量真正占用的字节数。

## 8. 分布式训练

### 8.1 数据并行：把一个大 batch 摊到多张卡上

分布式训练（Distributed Training，把一个训练任务拆到多张 GPU 或多台机器上同时执行）要解决两件事：单卡显存装不下模型，以及单卡算力跑不完数据。按「切什么」可以分成三类：数据并行（Data Parallelism，每张卡持有完整模型、各吃一部分数据）、张量并行（Tensor Parallelism，把单个矩阵乘按行或按列切开分到多张卡上算）、流水线并行（Pipeline Parallelism，把不同的层放到不同卡上按阶段执行）。数据并行改动最小、通用性最好，是绝大多数团队上的第一套方案。

数据并行的数学依据很直接。设全局 batch 为 B，切成 N 份后每张卡拿到 B/N 条样本。第 i 张卡算出的梯度是对「它那 B/N 条样本的平均梯度」，是全局梯度的无偏估计；把所有卡的梯度求和再除以 N，得到的就是「用整个 B 训练一步」的梯度，与单卡大 batch 数学等价。所以数据并行的核心动作只有一个：每步反向结束后做一次梯度归约（All-Reduce，所有进程把各自的张量汇总成一个相同结果并写回每个进程）。

代价在通信。一次 all-reduce 搬运的数据量等于梯度张量的字节数：参数量 P、fp16 精度时约 2P 字节。反向传播又必须等所有卡都算完才能同步，于是通信时间与计算时间的比值决定了加速比的上限。同机内走 NVLink/PCIe 足够快，4 卡通常能拿到 3.5 倍以上加速；跨机走普通以太网时，往往要配合更大 batch 或梯度压缩才划算。

```python
# 量级感受：1.3B 参数模型在 fp16 下，每个训练步要同步的梯度字节数
params, bytes_per_param = 1.3e9, 2
grad_gib = params * bytes_per_param / 1024**3   # ≈ 2.42 GiB
print(f"{grad_gib:.2f} GiB per step")
```

梯度对齐了，学习率也要跟着调。全局 batch 放大 N 倍后若保持原学习率，每步更新幅度相对不足，收敛会变慢。常用的线性缩放规则（Linear Scaling Rule，把学习率按全局 batch 的倍数同比放大）配合几百步 warmup 使用，可以先从 N 倍里取 √N 倍这种保守值试起，避免前期发散。

### 8.2 DP 与 DDP：名字很像，实现完全不同

PyTorch 有两套数据并行实现，网上教程里最容易抄到旧的那套。

`nn.DataParallel`（DP）是单进程多线程：只有一个 Python 解释器，主卡（device 0）负责把模型复制到各卡、切分输入、收集输出、算 loss，再反向把梯度 scatter 回去。它的问题很硬：受全局解释器锁限制，多卡并不能真正并行跑 Python 逻辑；每步都复制一次模型参数；通信逐层串行，带宽利用率低；BatchNorm 的统计量只在主卡更新，行为和单卡不一致；主卡既要汇总梯度又要当参数服务器，显存占用明显更高。它只适合「两张卡随手试试」，生产环境不再推荐。

`DistributedDataParallel`（DDP）是多进程：每个进程一个独立解释器、绑定一张卡，进程间用 NCCL 通信。反向传播时梯度按 bucket 分桶，某个桶的梯度一算完就立刻触发 all-reduce，与后续层的反向计算重叠，通信被藏在计算里。这是它能线性扩展的关键，也是 DDP 与「先算完再统一同步」的朴素实现的区别。

| 对比项 | DP（nn.DataParallel） | DDP（DistributedDataParallel） |
| --- | --- | --- |
| 进程模型 | 单进程多线程，GIL 限制 | 每卡一个进程，真正并行 |
| 通信内容 | 每步复制模型 + 逐层传梯度 | 只传梯度，分桶与计算重叠 |
| 主卡瓶颈 | 有，主卡显存与带宽压力大 | 没有，各卡地位对等 |
| BatchNorm | 统计量只在主卡上更新 | 各进程独立统计，等价于小 batch 口径 |
| 数据切分 | 自动切分输入张量 | 必须配 DistributedSampler |
| 启动方式 | `python train.py` | `torchrun` 或 `mp.spawn` |
| 适用场景 | 调试、双卡玩具实验 | 单机多卡与多机的标准方案 |

BatchNorm 那一行值得单独提醒：DDP 下每张卡只看到自己的 B/N 条样本，BN 的均值方差是按这个局部 batch 算的。想让多卡 BN 与单卡大 batch 完全等价，要把 BN 换成 `nn.SyncBatchNorm.convert_sync_batchnorm(model)`，代价是每层多一次跨卡同步。另外 DDP 默认桶大小 25MB，由构造参数 `bucket_cap_mb` 控制，模型很小或某层特别大时可以调。

通信后端也有一条硬规则：GPU 训练一律 `backend="nccl"`，它针对 NVIDIA 显卡拓扑做过优化，能直接走 NVLink 与 InfiniBand；`gloo` 只用于纯 CPU 训练或 CPU 张量通信，拿它跑 GPU 数据并行会慢到不可接受。不过 gloo 的报错信息更友好，怀疑连接问题时可以用两个进程、`gloo` 后端跑一次最小实验，先分清是网络不通还是 NCCL 配置不对。

### 8.3 单机多卡实操：DDP 的五步改造

把一个单卡脚本改成 DDP 只需五处改动，顺序固定：初始化进程组、绑定本地设备、模型包 DDP、数据换 DistributedSampler、只在主进程保存和打印。

```python
import os, torch, torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP
from torch.utils.data import DataLoader, DistributedSampler

def main():
    # torchrun 会把 rank/world_size 注入环境变量，不用自己算
    dist.init_process_group(backend="nccl")          # 1. 建进程组，自动读环境变量
    local_rank = int(os.environ["LOCAL_RANK"])       # 本机第几张卡
    torch.cuda.set_device(local_rank)                # 2. 当前进程绑定这张卡
    device = torch.device("cuda", local_rank)

    model = MyNet().to(device)
    model = DDP(model, device_ids=[local_rank])      # 3. 包装，构造时广播一次参数保证起点一致

    dataset = MyDataset()
    sampler = DistributedSampler(dataset, shuffle=True, drop_last=True)  # 4. 每个 rank 只取自己那份
    loader = DataLoader(dataset, batch_size=32, sampler=sampler,
                        num_workers=4, pin_memory=True)

    optimizer = torch.optim.AdamW(model.parameters(), lr=1e-3)
    for epoch in range(10):
        sampler.set_epoch(epoch)                     # 必须：否则每个 epoch 的分片顺序完全一样
        model.train()
        for x, y in loader:
            x = x.to(device, non_blocking=True)
            y = y.to(device, non_blocking=True)
            optimizer.zero_grad(set_to_none=True)    # 置 None 比填 0 省一次显存写
            loss = model(x).loss(y)
            loss.backward()                          # 反向内部自动 all-reduce 梯度
            optimizer.step()
        if dist.get_rank() == 0:                     # 5. 只有主进程写盘与打印
            torch.save(model.module.state_dict(), f"ckpt_e{epoch}.pt")
    dist.destroy_process_group()                     # 收尾，避免端口与通信资源残留

if __name__ == "__main__":
    main()
```

所有进程都会完整跑一遍训练循环，只有 rank 0 的输出该被看到；`print` 不加 rank 判断就会打印 N 份，日志里 loss 看起来像在剧烈震荡，实际只是多份拷贝。

三个容易写错的地方。一是模型必须先 `.to(device)` 再包 DDP；包装之后原始模型在 `model.module` 里，直接改外层属性无效。二是 `device_ids=[local_rank]` 只在单机多卡场景给，多机时该传 `None`，否则报 `RuntimeError: device_ids must be a list of length 1`。三是 `set_epoch` 漏掉不会报错，只会让每个 epoch 的样本顺序完全相同、且同一个 rank 永远看不到某部分数据，表现为训练集准确率异常高而验证集不动。

### 8.4 torchrun 启动与关键参数

`torchrun` 是 `torch.distributed.run` 的命令行入口，为每个进程写好 `RANK`、`LOCAL_RANK`、`WORLD_SIZE`、`MASTER_ADDR`、`MASTER_PORT`；任一进程异常退出时它会杀掉整组进程，不留僵尸 rank。

```bash
# 单机 4 卡，最常用的一行
torchrun --nproc_per_node=4 --master_port=29501 train.py --epochs 10 --lr 1e-3

# 两机各 8 卡，在第 0 台执行；第 1 台把 --node_rank 改成 1
torchrun --nnodes=2 --node_rank=0 --nproc_per_node=8 \
         --master_addr=10.0.0.11 --master_port=29500 train.py
```

`--` 之后的内容原样传给训练脚本，脚本里的 argparse 看不到 torchrun 自己的参数。

| 参数 | 含义 | 典型取值与坑 |
| --- | --- | --- |
| --nproc_per_node | 每台机器启动几个进程 | 等于本机可用 GPU 数，超过会互相抢显存 |
| --nnodes | 参与训练的机器数 | 单机写 1，所有机器必须一致 |
| --node_rank | 当前机器的序号 | 从 0 开始，两台写同一个值会端口冲撞 |
| --master_addr | rank 0 所在机器的地址 | 多机要写对方可达的内网 IP，不能写 127.0.0.1 |
| --master_port | 进程组通信端口 | 默认 29500，被占用时换成 29501 之类 |

旧写法 `python -m torch.distributed.launch --nproc_per_node=4 train.py` 仍能跑，但它依赖 `--use_env` 之类的隐式行为、不支持弹性伸缩，新代码统一用 torchrun。若训练脚本要读 `--local_rank`，注意新版 torchrun 不再默认注入这个命令行参数，改为从 `LOCAL_RANK` 环境变量取。

### 8.5 检查点：存什么、谁能读

DDP 下 `model.state_dict()` 的 key 全部带 `module.` 前缀，因为它外面多包了一层容器。直接保存会让单卡推理时全部 key 不匹配。保存时统一用 `model.module.state_dict()`；手上的旧 ckpt 若已带前缀，最稳的做法是先剥掉再加载，而不是靠 `strict=False` 蒙混过关。

```python
# 剥掉历史 ckpt 里多余的 "module." 前缀
ckpt = torch.load("old.pt", map_location="cpu")
ckpt = {k.replace("module.", "", 1): v for k, v in ckpt.items()}
model.load_state_dict(ckpt)
```

断点续训要存的不只是权重，缺一项都会让恢复后的曲线和原来对不上：

| 保存项 | 为什么必须存 |
| --- | --- |
| `model.module.state_dict()` | 权重本体，注意剥前缀 |
| `optimizer.state_dict()` | Adam 的 m/v 动量，缺了恢复后前几步会明显抖动 |
| `scheduler.state_dict()` | 学习率进度，否则 warmup 与衰减从头再来 |
| `epoch` 与 `global_step` | 决定从哪继续、日志横轴是否正确 |
| `scaler.state_dict()` | 混合精度的 loss scale，缺了容易前几步梯度溢出 |
| `torch.get_rng_state()` 与 CUDA RNG | 可复现性，数据增强与 dropout 依赖它 |

写盘只让 rank 0 做，但保存一结束其他 rank 可能已经跑到下一步，要严格同步就在 `torch.save` 后面加一句 `dist.barrier()`。读取则建议每个 rank 各读一份：统一 `map_location="cpu"` 再 `.to(device)`，既避开「rank 0 正在写、别人读到半个文件」的竞态，也避免把整份 ckpt 的显存压力集中在一张卡上。

### 8.6 FSDP 与 ZeRO：连参数都切开

DDP 并没有降低单卡显存：每张卡都存完整参数、梯度和优化器状态。混合精度加 Adam 时，每个参数大约占 16 字节——fp16 参数 2、fp16 梯度 2、fp32 主权重 4、Adam 一阶与二阶动量 8。7B 模型光这一项就是 112 GB，单张 80G 卡装不下。

ZeRO（Zero Redundancy Optimizer，DeepSpeed 提出的显存去冗余方案，把原本每卡各存一份的状态切分到各卡）分三级：Stage 1 切优化器状态，Stage 2 再切梯度，Stage 3 连参数一起切。FSDP（Fully Sharded Data Parallel，PyTorch 原生的参数分片数据并行）走的是 Stage 3 思路：参数平时以扁平分片的形式分散在各卡，前向或反向用到某层时临时 all-gather 出完整层参数，用完立刻释放，反向再用 reduce-scatter 把梯度归约并按分片写回。

| 方案 | 切分对象 | 每卡显存（约） | 通信量 |
| --- | --- | --- | --- |
| DDP | 都不切 | 16 字节/参数 | 1 倍 |
| ZeRO-1 / FSDP 关分片参数 | 优化器状态 | 4 + 12/N 字节/参数 | 1 倍 |
| ZeRO-2 | 优化器状态 + 梯度 | 2 + 14/N 字节/参数 | 1 倍 |
| ZeRO-3 / FSDP FULL_SHARD | 全部三项 | 16/N 字节/参数 | 约 1.5 倍 |

```python
from torch.distributed.fsdp import FullyShardedDataParallel as FSDP
from torch.distributed.fsdp import ShardingStrategy, MixedPrecision

model = FSDP(
    model,
    sharding_strategy=ShardingStrategy.FULL_SHARD,   # 等价 ZeRO-3，显存吃紧时首选
    mixed_precision=MixedPrecision(param_dtype=torch.bfloat16,
                                   reduce_dtype=torch.bfloat16),
    device_id=torch.cuda.current_device(),
)
```

FSDP 对「按层 wrap」很敏感：整个模型包一层等于没切，反而更慢，必须用 auto_wrap_policy 按 Transformer block 之类的边界逐层包。它还可以把分片后的参数与梯度卸载到 CPU（`CPUOffload(offload_params=True)`），显存再紧一档，代价是每步多出主机与显存之间的来回拷贝，吞吐通常掉三成以上。它的通信量约为 DDP 的 1.5 倍，小模型上不一定更快。合理的路线是先用 DDP 把数据与训练流程跑对，显存确实不够再平移到 FSDP，参数规模在十亿以下时 DDP 通常更省心。

### 8.7 报错排查：端口、NCCL、rank

分布式训练的报错大多不是算法问题，而是「某个 rank 没跟上」。先开足日志再猜：

```bash
# 打开 NCCL 细节日志，看卡在哪一步握手；指定网卡避免多网卡选错
NCCL_DEBUG=INFO NCCL_SOCKET_IFNAME=eth0 torchrun --nproc_per_node=4 train.py

# 查看进程组的分组与超时设置
TORCH_DISTRIBUTED_DEBUG=DETAIL torchrun --nproc_per_node=4 train.py
```

| 现象 | 常见原因 | 对策 |
| --- | --- | --- |
| `Address already in use` | 上次进程未退干净，或两台机器 node_rank 重复 | `netstat -ano \| findstr 29500` 找到并杀掉，或换 `--master_port` |
| 卡在 init_process_group 不动 | MASTER_ADDR 不可达、防火墙拦端口 | 先在 rank 0 机器上 `telnet 目标IP 端口` 验证连通 |
| `NCCL error: unhandled system error` | 多网卡选错、IB 与以太网混用 | 用 `NCCL_SOCKET_IFNAME` 指定内网网卡，或加 `NCCL_IB_DISABLE=1` |
| 某卡 OOM 而其他卡正常 | 各 rank 局部 batch 不同，或 rank 0 多存了 ckpt | 检查 DataLoader 的 batch_size 是否为每卡口径 |
| 训练到一半全体挂死 | 某个 rank 提前 `break`/抛异常，其余 rank 卡在 all-reduce | 查最小 rank 的 traceback，保证所有 rank 走同样的控制流 |
| loss 与单卡差异大 | 漏了 `sampler.set_epoch`，或学习率未按全局 batch 缩放 | 补 set_epoch，按线性缩放规则调 lr |

排查时按这个顺序走：先确认所有进程都进来了（打印 `rank/world_size`），再确认端口与地址连通，最后才怀疑 NCCL 与网卡。分布式问题的最小复现是把 `--nproc_per_node` 降到 2，用几十条假数据跑 3 个 step，能复现就与模型和数据集无关。

**常见坑**

1. 所有 rank 都 `print` 和 `torch.save`，日志刷屏、权重文件互相覆盖，只剩最后写的那份能读。
2. 忘了 `sampler.set_epoch(epoch)`：不报错，但每个 epoch 的样本顺序一模一样，训练集指标虚高。
3. 直接存 `model.state_dict()`，ckpt 里全是 `module.` 前缀，换到单卡加载时 key 全不匹配。
4. 把 `batch_size` 当全局值用：DataLoader 里的 `batch_size` 是每卡口径，全局 batch 要再乘 `world_size`，学习率与日志里的吞吐都跟着错。

## 9. 保存、加载与部署

### 9.1 state_dict：推荐保存的最小单元

state_dict（模型状态字典，`nn.Module` 中所有需要持久化的张量按名字索引组成的 OrderedDict）里有两类条目：parameters（可训练参数，即 `requires_grad=True` 的 `nn.Parameter`，如 Linear 的 weight 与 bias）和 buffers（不参与梯度、但属于模型状态的张量，如 BatchNorm 的 running_mean、running_var、num_batches_tracked）。两类都要存，模型的推理行为才完整；只存参数、重建模型后再加载，BN 的统计量会回到初始值（均值 0、方差 1），同一份输入的输出与训练结束时不一致，这是「离线指标正常、上线后精度掉一截」的常见来源。

state_dict 的本质是「字符串键 + 纯张量」，不含类的定义和代码结构。因此加载流程固定为两步：先在代码里搭出结构完全相同的模型，再把数值灌进去。好处是权重文件与源码解耦，改目录、改包名都不影响加载；代价是结构一致性要自己保证。键名由模块层级拼接而成，`Sequential` 里是 `0.weight` 这类数字前缀，用 `named_parameters()` 能打印出真实键名，这也是排查加载失败的第一手信息。

```python
import torch, torch.nn as nn

net = nn.Sequential(nn.Linear(4, 8), nn.BatchNorm1d(8), nn.ReLU(), nn.Linear(8, 3))
sd = net.state_dict()
print([k for k in sd][:3])              # ['0.weight', '0.bias', '1.weight']：键名来自层号
print(sd["1.running_mean"].shape)       # torch.Size([8])：buffer 也在 state_dict 里
torch.save(sd, "net.pt")                # 只落数值，不落代码

net2 = nn.Sequential(nn.Linear(4, 8), nn.BatchNorm1d(8), nn.ReLU(), nn.Linear(8, 3))
state = torch.load("net.pt", map_location="cpu")   # 无 GPU 的机器必须指定 map_location
missing, unexpected = net2.load_state_dict(state, strict=False)
print(missing, unexpected)              # 两个列表都为空，说明结构完全对齐
```

`map_location` 决定张量重建到哪个设备。用 GPU 训练、在只有 CPU 的机器上加载时，不加这个参数会抛 `Attempting to deserialize object on a CUDA device but torch.cuda.is_available() is False`。

`strict` 默认 True，要求键集合严格一致，否则报 `Missing key(s)` 或 `Unexpected key(s)`。设成 False 会只加载交集、忽略差异，适合结构改过一部分的场景，但必须检查返回值：如果返回的 missing 里包含全部键，实际一个参数都没加载上，程序照常跑、指标却等于随机初始化。上线脚本里应把这两个列表打进日志。

### 9.2 三种存法：只存权重、存整模型、存 checkpoint

`torch.save(model)` 把整个模块对象 pickle 下来，加载时靠 pickle 里记录的「模块路径 + 类名」重新 import 那个类。重构目录、改包名、重命名类之后加载会直接失败，报 `ModuleNotFoundError` 或 `AttributeError: Can't get attribute 'Net'`。pickle 反序列化还能执行任意代码，来源不可信的 `.pt` 文件等同于运行陌生程序。

| 存法 | 写法 | 体积 | 对代码结构的要求 | 典型用途 |
| --- | --- | --- | --- | --- |
| 只存权重 | `torch.save(model.state_dict(), p)` | 最小，约参数量 × 精度字节 | 结构一致即可，最稳 | 发布权重、推理部署 |
| 存整个模型 | `torch.save(model, p)` | 略大 | 依赖类路径，重构即失效 | 本地临时试验、notebook |
| 存 checkpoint 字典 | `torch.save({...}, p)` | 最大 | 自带元信息，可断点续训 | 训练中途保存与恢复 |

checkpoint 不是一个张量，而是一个字典，训练相关的状态要尽量收全：模型 `state_dict`、优化器 `state_dict`、学习率调度器 `state_dict`、混合精度 `GradScaler` 的 `state_dict`、当前 epoch 与 global_step、最佳验证指标，以及随机数状态。优化器状态常被漏掉：Adam 系列会为每个参数维护一阶动量 exp_avg 与二阶动量 exp_avg_sq（外加 step 计数），不存它们，续训后这两项从零开始，前几十步的更新方向与幅度都会偏，表现为 loss 曲线出现一个小台阶，长训练里损失可观。

```python
import random, numpy as np, torch

def save_ckpt(path, model, opt, sched, scaler, epoch, best):
    torch.save({
        "model": model.state_dict(),
        "optimizer": opt.state_dict(),       # 含 Adam 的动量，断点续训必需
        "scheduler": sched.state_dict(),
        "scaler": scaler.state_dict(),       # AMP 的动态缩放系数
        "epoch": epoch,
        "best_acc": best,
        "rng": {                              # 复现数据增强与 dropout 的随机性
            "torch": torch.get_rng_state(),
            "cuda": torch.cuda.get_rng_state_all(),
            "numpy": np.random.get_state(),
            "python": random.getstate(),
        },
        "config": {"hidden": 256, "vocab": 32000, "tokenizer": "sp-v3"},  # 结构超参，缺了没法重建
    }, path)

def load_ckpt(path, model, opt=None, sched=None, scaler=None):
    ck = torch.load(path, map_location="cpu")
    model.load_state_dict(ck["model"])
    if opt is not None:
        opt.load_state_dict(ck["optimizer"])
    if sched is not None:
        sched.load_state_dict(ck["scheduler"])
    if scaler is not None:
        scaler.load_state_dict(ck["scaler"])
    torch.set_rng_state(ck["rng"]["torch"])
    return ck["epoch"], ck["best_acc"]
```

checkpoint 的体积约等于「参数量 × 精度字节 × (2~3)」：模型一份、优化器动量两份，fp32 训练 1 亿参数的模型，单个 checkpoint 就是 1.1GB 量级。保存频率控制在「每 N 步一次 + 保留最近 K 个 + 只留最佳一个」，并且只在主进程写盘。

### 9.3 迁移学习与冻结：加载「能对上的那部分」

迁移学习（Transfer Learning，把在大数据集上训练好的权重拿来做新任务）分两步：加载预训练权重，替换不匹配的输出层。需要注意，`strict=False` 只能解决「键缺失或不匹配名字」，解决不了「名字相同但形状不同」——把 1000 类的 fc 换成 10 类后再加载旧权重，会报 `size mismatch for fc.weight`。正确顺序是先换结构，再加载，让不匹配的键自然成为「缺失」或「多余」。

```python
import torch.nn as nn
from torchvision.models import resnet18, ResNet18_Weights

model = resnet18(weights=ResNet18_Weights.IMAGENET1K_V1)   # 加载 ImageNet 预训练权重
model.fc = nn.Linear(model.fc.in_features, 10)             # 先换头：新层随机初始化

for p in model.parameters():
    p.requires_grad = False                                # 整体冻结主干
for p in model.fc.parameters():
    p.requires_grad = True                                 # 只放开新分类头

for m in model.modules():                                  # 冻结时 BN 保持 eval
    if isinstance(m, nn.BatchNorm2d):
        m.eval()

head = list(model.fc.parameters())
trunk = [p for n, p in model.named_parameters() if not n.startswith("fc.")]
optimizer = torch.optim.AdamW([                            # 参数分组，学习率不同
    {"params": head, "lr": 1e-3},
    {"params": trunk, "lr": 1e-5},                         # 冻结时无梯度，不会更新
], weight_decay=1e-2)

trainable = sum(p.numel() for p in model.parameters() if p.requires_grad)
print(f"可训练参数 {trainable/1e6:.2f}M")                  # 只占总量的一小部分
```

冻结的两个细节容易出错。第一，`model.train()` 会把 BN 切回训练态并更新 running_mean/running_var，冻结主干时必须在每个 epoch 调用 `train()` 之后，再把 BN 子模块单独设回 `eval()`，否则预训练权重会被新数据的小 batch 统计量一步步污染。第二，`requires_grad=False` 的参数传给优化器不会报错，只是没有梯度、不会被更新；真正决定是否更新的是梯度是否存在，不是优化器里有没有这个参数。

训练策略上常用「先冻主干训头、再解冻后几层小学习率微调」：前 1~2 个 epoch 只训头，让随机初始化的新层先稳定下来，避免大梯度冲坏主干；之后按层解冻，越靠近输入的层学习率越小。数据量只有几千张图时全量微调极易过拟合，冻结主干通常更稳。

### 9.4 TorchScript：把模型变成不依赖 Python 的图

TorchScript（PyTorch 的静态图中间表示，把模型转成可脱离 Python 解释器执行的 IR）解决的是部署端没有 Python 环境、或需要 C++ 加载的场景。生成方式有两种：追踪（Tracing，喂一组示例输入把模型跑一遍，记录实际发生的算子序列）和脚本化（Scripting，分析源码把 Python 语法转译成 IR）。

| 对比项 | `torch.jit.trace` | `torch.jit.script` |
| --- | --- | --- |
| 生成方式 | 记录一次真实执行路径 | 解析源码转译 |
| 数据相关控制流 | 被固化成一个分支，换输入也不变 | 保留为图中的真实分支 |
| 输入形状 | 记住示例 shape，动态维度需重新 trace | 由类型标注决定，可动态 |
| 对代码的要求 | 几乎无，只要能跑通 | 需要类型标注，不能用部分动态写法 |
| 常见失败 | 逻辑静默错误（不报错但结果错） | 语法不支持时直接编译报错 |

trace 最危险的失败是「不报错但行为错」：`if x.sum() > 0` 这种依赖数据的判断在 trace 时被记成一个常量分支，上线后无论如何都走那一支。所以含数据相关分支的模型必须用 script，或者把判断挪到模型外面。

```python
import torch

class Net(torch.nn.Module):
    def __init__(self):
        super().__init__()
        self.fc = torch.nn.Linear(8, 2)
    def forward(self, x):
        return self.fc(x)

m = Net().eval()                                   # trace 前必须 eval，否则记录的是训练态
example = torch.randn(1, 8)
traced = torch.jit.trace(m, example)
traced.save("net_scripted.pt")                     # C++ 端用 torch::jit::load 加载

loaded = torch.jit.load("net_scripted.pt")
with torch.no_grad():
    ok = torch.allclose(loaded(example), m(example), atol=1e-6)
print(ok)                                          # True 表示数值一致
```

导出后先做数值一致性校验再交付，这是唯一能发现 trace 静默错误的办法。

### 9.5 ONNX 导出与算子兼容

ONNX（Open Neural Network Exchange，跨框架的模型交换格式）导出后可以交给 onnxruntime、TensorRT、OpenVINO 等运行时执行，适合需要跨语言、跨硬件部署的场景。

```python
import torch
dummy = torch.randn(4, 3, 224, 224)     # 示例输入：导出时会跑一遍，因此 batch 要覆盖真实取值
torch.onnx.export(
    model, (dummy,), "model.onnx",
    input_names=["input"], output_names=["logits"],
    dynamic_axes={"input": {0: "batch"}, "logits": {0: "batch"}},  # 不声明就锁死 batch=1
    opset_version=17,               # 算子集版本，越高支持的算子越多，但运行时需同步升级
    do_constant_folding=True,       # 常量折叠，能提前算掉的都算掉
)
```

`dynamic_axes` 是最常被忽略的参数。不声明时导出的图把示例输入的每个维度都写成常量，推理端喂 batch=8 会报维度不匹配。业务上 batch 会变的维度（batch、序列长度、图像高宽）都要声明成符号名。

```python
import numpy as np, onnxruntime as ort
sess = ort.InferenceSession("model.onnx", providers=["CPUExecutionProvider"])
out = sess.run(None, {"input": dummy.numpy()})[0]
print(np.abs(out - model(dummy).detach().numpy()).max())   # fp32 一般 < 1e-4
```

算子不兼容的排查顺序：先看报错里被点名的是哪个算子（`UnsupportedOperatorError` 会给出算子名与 opset），再去查该算子在目标 opset 中是否已注册，把 `opset_version` 提到支持它的版本；仍不行就改写模型，例如把自定义的归一化换成 `F.layer_norm`、把 `interpolate` 的 `scale_factor` 换成固定 `size`、把自定义 autograd Function 的 forward 逻辑抄成纯算子组合。ONNX 只能表达「前向计算图」，带训练语义的算子（如可学习的、依赖随机数的）要么去掉，要么用 mask 形式改写。

导出失败还有一类原因在模型状态：`model.train()` 导出的图会带上 Dropout 的随机分支，`onnxruntime` 下每次结果不同。导出前统一 `model.eval()` 并用 `torch.no_grad()` 包住。

### 9.6 推理优化：先关梯度，再谈精度与批处理

推理优化的收益从大到小有个固定顺序：关掉自动求导、切 eval 模式、降精度、加大 batch、编译加速。

`no_grad()` 与 `inference_mode()` 都关闭梯度记录，区别在彻底程度。`inference_mode` 连版本计数（tensor 的 version counter，用于检测原地修改）都不再维护，因此更省内存与时间，但它产生的张量是「推理张量」，之后若参与需要梯度的计算会报错。需要保留可微性的场景（如可解释性分析、对抗样本生成）只能用 `no_grad`。

| 手段 | 写法 | 主要收益 | 注意 |
| --- | --- | --- | --- |
| 关梯度 | `with torch.inference_mode():` | 省激活显存与建图开销 | 张量不可再参与反向 |
| 切 eval | `model.eval()` | BN 用 running 统计量、Dropout 关闭 | 与关梯度是两件事，必须都做 |
| 半精度 | `model.half()` + 输入 `.half()` | 显存减半，GPU 上吞吐提升 | CPU 上 fp16 常常更慢；bf16 更稳 |
| 批处理 | 合并请求为大 batch | 提升 GPU 利用率与吞吐 | 延迟上升，P99 要看排队时间 |
| 编译 | `torch.compile(model)` | 算子融合，常见 1.2~1.5 倍 | 首次调用有编译耗时，动态 shape 加 `dynamic=True` |

批处理是吞吐与延迟的直接交换：单条推理 GPU 利用率可能只有 10%，凑成 batch=32 能到 70% 以上，但最后一条请求要等前面凑齐。在线服务常用「微批 + 超时」策略：等待几毫秒或攒满 N 条就发一次，先到先执行，这就是动态批处理（Dynamic Batching，把短时间窗口内的请求合并成一个 batch 执行）的基本思路。

`model.half()` 要连输入一起转，否则报 `Input type (float) and bias type (c10::Half) should be the same`。对数值范围敏感的算子（LayerNorm、softmax 等）在纯 fp16 下可能溢出，改用 bf16 通常无需缩放即可用。

### 9.7 服务化：TorchServe 与 Triton 概览

模型服务框架要解决的是同一批问题：模型怎么打包、请求怎么进来、预处理与后处理挂在哪、多模型怎么共存、怎么扩缩容。

TorchServe（PyTorch 官方与 AWS 联合维护的模型服务框架）用 `.mar` 归档文件打包模型权重与 handler（handler 是定义 `preprocess`/`inference`/`postprocess` 三个函数的 Python 类），配置文件 `config.properties` 指定 `inference_address`、`management_address`、`number_of_gpu`、`batch_size` 与 `max_batch_delay`。它的优势是与 PyTorch 生态贴合、自定义逻辑直接写 Python；弱点是只服务 PyTorch 模型、动态批处理能力有限。

Triton Inference Server（NVIDIA 的通用推理服务器）以模型仓库目录为单位组织，每个模型一个子目录，里面放模型文件与 `config.pbtxt`。关键参数包括 `max_batch_size`、`dynamic_batching { max_queue_delay_microseconds: 5000 }`（等待 5ms 凑批）、`instance_group`（每卡几个实例）、`platform`（`onnxruntime_onnx`、`tensorrt_plan`、`pytorch_libtorch` 等）。它支持多框架后端、模型集成（ensemble）与并发模型执行，适合多模型、多框架、追求吞吐的生产环境；代价是配置以 protobuf 文本为主，调试链路更长。

| 对比项 | TorchServe | Triton Inference Server |
| --- | --- | --- |
| 模型格式 | PyTorch（eager 或 TorchScript） | ONNX / TensorRT / TorchScript / Python 后端等 |
| 动态批处理 | 有，能力较弱 | 成熟，参数可细调 |
| 自定义前后处理 | Python handler，写起来直接 | Python backend 或 preprocessing 目录 |
| 多模型共存 | 支持，管理较简单 | 强项，模型仓库 + 版本目录 |
| 适用场景 | 纯 PyTorch、团队以算法为主 | 多框架、高吞吐、已有 GPU 推理平台 |

无论用哪个框架，部署前都应固定三件事：模型版本号（请求里带 version，便于灰度与回滚）、输入校验（shape、dtype、取值范围，别把异常数据送进模型）、以及一份「离线推理结果 vs 服务返回结果」的一致性对比脚本，确保导出、量化、批处理没有改变输出语义。

### 9.8 常见坑

- 加载成功但精度崩塌：`strict=False` 静默跳过大量键，务必打印 missing/unexpected 列表并断言为空或符合预期。
- 续训后 loss 跳变：checkpoint 漏存优化器状态，Adam 动量归零，把 `optimizer.state_dict()` 一并保存。
- 导出后 batch 锁死为 1：ONNX 未声明 `dynamic_axes`，或 trace 时只喂了单一 shape。
- 部署端结果与训练端不一致：导出前忘了 `model.eval()`，BN 与 Dropout 仍是训练态行为。

## 10. 调试与常见报错

PyTorch 的报错看起来五花八门，其实有稳定的"指纹"：异常类型、期望值与实际值的对比、设备名和 dtype、以及调用栈里第一个属于自己代码的帧。会调试的人不是记得更多报错，而是有一套固定动作：先读懂这一行在说什么，再把问题压缩到二十行能复现，最后用一个能证伪的检查确认原因。这套动作用在七类问题上——形状、设备、数值、显存、数据加载、编译与不收敛，它们的处理顺序并不相同。

### 10.1 把报错读成一句话

一条 traceback 里信息密度最高的不是最后一行，而是**异常类型加消息主体**。最后一行告诉你"在哪炸"，第一行告诉你"为什么炸"。读的时候只找三样东西：

| 要素 | 典型形式 | 用法 |
| --- | --- | --- |
| 异常类型 | `RuntimeError`、`ValueError`、`IndexError`、`torch.cuda.OutOfMemoryError` | 决定大类：RuntimeError 多为形状/设备/图结构，IndexError 多为索引越界 |
| 期望与实际 | `expected 256, got 128`、`must match`、`should be the same` | 括号里通常左边是实际、右边是期望，或反之，先确定哪一维不等 |
| 出事位置 | 栈中第一个不是 `torch/` 或 `site-packages` 的帧 | 那才是你的代码；继续往上读 torch 内部实现没有意义 |

除了改代码，还有一组"开关"能显著提高报错的信息量，代价是变慢，只在调试时开：

| 开关 | 作用 | 代价 |
| --- | --- | --- |
| `CUDA_LAUNCH_BLOCKING=1` | 让 CUDA 内核同步执行，报错行号指向真正的出错算子 | 速度降数倍 |
| `TORCH_SHOW_CPP_STACKTRACES=1` | 打印 C++ 侧调用栈，定位 ATen 内部抛错 | 输出很长 |
| `torch.autograd.set_detect_anomaly(True)` | 反向传播时检查 NaN/Inf，并打印产生它的前向算子 | 慢 2~10 倍 |
| `breakpoint()` | 在 `forward` 里停下，交互式查看张量 | 需要可交互终端 |

CUDA 调用默认异步，这是"报错位置漂移"的根源：真正的错误发生在第 10 步，报错却出现在第 50 步的某个无关算子上。看到栈指向奇怪位置的 CUDA 错误，先用 `CUDA_LAUNCH_BLOCKING=1` 重跑一遍再判断。

还要先分清确定性错误与概率性错误。每次都崩在同一行的，是确定性错误，直接读栈即可；跑十次崩三次、或换台机器就好的，是概率性错误，成因集中在四处：数据顺序（`shuffle` 与 worker 的随机种子）、多进程（`num_workers>0` 时的进程竞争）、非确定性算子（原子加、cuDNN 自动调优）、显存碎片。对这类问题，处置顺序是先把变量一个个钉死——`num_workers=0`、`shuffle=False`、固定种子——确认能稳定复现，再逐个放开，看放到哪一个又崩。

```python
# 排查第一步：把环境四元组打出来，贴 issue 或对比"能跑的同事"都用得上
import torch, platform
print(torch.__version__, torch.version.cuda)          # 例如 2.4.1 12.1
print(platform.platform(), platform.python_version())
print(torch.cuda.is_available(), torch.cuda.device_count())
print(torch.cuda.get_device_name(0), torch.cuda.get_device_capability(0))  # 算力 (8, 0) 即 sm_80
```

### 10.2 最小复现法

能在二十行里复现的问题，通常十分钟内能定位；只能在八百行项目里复现的问题，可能耗掉两天。最小复现（Minimal Reproducible Example）不是"为了提问才做"的礼仪，而是定位手段本身：每删掉一块代码问题就消失，那一块就是嫌疑人。

操作顺序固定为五步：

1. 固定随机性。`torch.manual_seed(0)`、`numpy.random.seed(0)`、`random.seed(0)` 一起设，`DataLoader` 的 `shuffle` 先关掉。
2. 换掉数据。用 `torch.randn` 造与真实 batch 同形状、同 dtype、同 device 的假数据。问题仍在，说明是模型、损失或训练逻辑；问题消失，说明是数据管道。
3. 换掉模型。从一个 `Linear` 开始，逐层加回去；层数多就二分，一半一半加。
4. 换掉步骤。只保留一次 `forward`；再加 `loss.backward()`；再加 `optimizer.step()`。爆在哪一步，范围就缩到那一步。
5. 留档。把版本四元组、完整 traceback、复现脚本三样存在同一个 issue 里，问题复现和回归验证都用它。

```python
import torch, random, numpy as np

def set_seed(s=0):
    # 四个随机源都要固定，否则"时好时坏"的问题永远复现不出来
    random.seed(s); np.random.seed(s); torch.manual_seed(s)
    torch.cuda.manual_seed_all(s)
    torch.backends.cudnn.deterministic = True
    torch.backends.cudnn.benchmark = False

set_seed(0)
# 假数据：形状 dtype device 与真实 batch 完全一致，但内容随机
x = torch.randn(8, 3, 224, 224, device="cuda")
y = torch.randint(0, 10, (8,), device="cuda")
model = MyNet().cuda()   # MyNet 是占位，换成你自己的模型
loss = torch.nn.functional.cross_entropy(model(x), y)
loss.backward()   # 这一步单独跑，先确认反传本身没问题
```

要验证一个假设，用"断言"而不是"观察"。观察是"看起来 loss 正常"，断言是 `assert torch.isfinite(loss), loss`——后者会立刻停下并带上现场，前者会让你在多步之后才发现异常。

### 10.3 形状不匹配

形状错误是出现频率最高、也最容易读的一类，因为报错直接把两个形状摆在一起。按报错原文对照处理：

| 报错原文（节选） | 含义 | 修法 |
| --- | --- | --- |
| `mat1 and mat2 shapes cannot be multiplied (32x128 and 256x64)` | `Linear` 的 `in_features` 与输入最后一维不等 | 改 `in_features`，或在上一处调整输出维度 |
| `The size of tensor a (128) must match the size of tensor b (64) at non-singleton dimension 2` | 该维既不相等也不为 1，广播失败 | 显式 `unsqueeze`/`expand`，或核对两路数据的来源 |
| `Expected 4-dimensional input for 4-dimensional weight [64, 3, 3, 3], but got 3-dimensional input` | 单张图缺 batch 维 | `x = x.unsqueeze(0)` |
| `Expected input batch_size (32) to match target batch_size (16)` | logits 是 `[N, C]`、target 是 `[N]`，两个 N 对不上 | 对齐样本数与类别维，别把 `C` 当 `N` |
| `view size is not compatible with input tensor's size and stride` | 非连续张量不能直接 `view` | `.reshape(...)` 或 `.contiguous().view(...)` |
| `too many values to unpack (expected 4)` | 模型返回值从张量变成了 tuple/dict | 按实际返回结构解包 |

定位靠打印，但不要在每层手写 `print`。用 `forward hook` 一次性拿到全部叶子层的输入输出形状，哪一层的输出与下一层的输入不一致，问题就在那一层：

```python
def trace_shapes(model, sample):
    # 给所有叶子层挂 forward hook，一次前向打印全部形状
    hooks = []
    def make(name):
        def hook(module, inputs, output):
            i = tuple(inputs[0].shape) if isinstance(inputs[0], torch.Tensor) else type(inputs[0])
            o = tuple(output.shape) if isinstance(output, torch.Tensor) else type(output)
            print(f"{name:32s} {str(i):24s} -> {o}")
        return hook
    for name, m in model.named_modules():
        if not list(m.children()):                      # 只看叶子层，避免容器层噪音
            hooks.append(m.register_forward_hook(make(name)))
    with torch.no_grad():
        model(sample)
    for h in hooks:
        h.remove()                                     # hook 不摘会一直累积，且占用显存

trace_shapes(model, torch.randn(1, 3, 224, 224, device="cuda"))
```

在关键边界加形状断言，比事后猜快得多：`assert x.dim() == 4 and x.size(1) == 3, x.shape`。断言失败时的第二参数会把真实形状打出来，省掉一轮往返。

比"报错"更难查的是"不报错但算错了"。广播规则会把 `[N, 1]` 与 `[N]` 一起算成 `[N, N]`，把本意是逐样本的加权求和变成一个全连接式的矩阵乘法：形状合法、梯度正常、loss 也下降，只是模型在学一个完全不同的任务。判据是看维度数为 1 的位置是不是自己有意的：所有"用于广播的 1"都应该能说出理由，说不出就在那里加 `unsqueeze` 并在注释里写明；发现输出比预期多出一维，先查是不是两路张量的形状本就该对齐。配套工具可以装 `torchinfo`，一行 `summary(model, input_size=(1, 3, 224, 224))` 就能看到每层输出形状与参数量，省去手写 hook。

### 10.4 device 与 dtype 不一致

这两类错误的消息都很直白，但成因往往相隔很远——错误发生在第 40 行的加法上，真正的疏漏在第 3 行的 `Dataset.__getitem__` 里。

| 报错原文（节选） | 含义 | 常见成因 |
| --- | --- | --- |
| `Expected all tensors to be on the same device, but found at least two devices, cuda:0 and cpu!` | 参与同一运算的张量分居两处 | 循环里新建的 `torch.zeros`、位置编码、mask 没跟着 `.to(device)` |
| `Input type (torch.cuda.FloatTensor) and weight type (torch.FloatTensor) should be the same` | 数据在 GPU、模型还在 CPU | 忘了 `model.cuda()`；或加载了 CPU 上 `torch.load` 的权重 |
| `expected scalar type Long but found Float` | 需要整型索引的地方收到浮点 | `Embedding`/`CrossEntropyLoss` 的 target 不是 `long` |
| `expected scalar type Float but found Double` | 精度来自 NumPy 默认的 float64 | `torch.from_numpy` 后没有 `.float()` |

一个容易记错的语义：`Tensor.to()` 返回新张量，**必须接住**，`x.to(device)` 单独一行是无效操作；而 `nn.Module.to()` 是原地修改参数并返回自身，写不写赋值都生效。所以"张量不生效、模型生效"是正常现象，不是 bug。

```python
def device_report(model, *tensors):
    # 报 device 错误前先跑一次，能省掉大部分猜测
    p = next(model.parameters())
    print(f"model  device={p.device} dtype={p.dtype}")
    for i, t in enumerate(tensors):
        print(f"tensors[{i}] shape={tuple(t.shape)} device={t.device} dtype={t.dtype}")

# 统一搬运：collate 出来的 batch 是 dict 时，逐项 to，别对容器调用 .to()
batch = {k: v.to("cuda", non_blocking=True) for k, v in batch.items() if torch.is_tensor(v)}
```

还要注意 `pin_memory=True` 只把数据放进锁页内存（pinned memory），那是 CPU 侧的概念，不等于搬到 GPU；它的作用是让后续 `.to(device, non_blocking=True)` 的拷贝能和计算重叠。

混合精度还会制造一类只在特定层出现的 dtype 错误：`autocast` 把矩阵乘降到半精度，而 LayerNorm、损失、以及自定义 kernel 仍要求 fp32，于是报 `expected scalar type BFloat16 but found Float` 之类。这类错误的处置不是全局 `.half()`，而是在需要的边界显式转换：`x.float()` 进敏感算子，出了再 `x.to(dtype)` 转回去。全局 `model.half()` 是另一种极端——它把 BN、LayerNorm、softmax 也压成半精度，训练几百步后数值就开始飘，报错反而来得更晚。

### 10.5 NaN/Inf 与梯度爆炸

现象是 loss 突然变成 `nan`，或参数里出现 `nan`/`inf`。关键认知：一旦参数被写成 `nan`，AdamW 的一阶矩、二阶矩也被污染，之后的更新全是 `nan`。此时"继续训练看会不会自愈"是无效的，必须回滚到上一个健康 checkpoint。

源头只有四类，按排查成本从低到高走：

1. **数据**。归一化时除以 0 的方差、padding 填了 `inf`、爬取文本里的异常值。一份代码即可验证：`assert torch.isfinite(batch["x"]).all()`，失败就打印 `(~torch.isfinite(x)).sum()` 与对应索引。
2. **不稳定的算子**。`torch.log(x)` 在 `x=0` 时得 `-inf`；`sqrt` 负数得 `nan`；softmax 输入全是 `-inf` 时整行变 `nan`。加法用 `clamp_min`：`torch.log(x.clamp_min(1e-8))`。
3. **学习率**。过大导致参数一步跨到远处，梯度随之爆炸。把 lr 降十倍重跑是最便宜的对照实验。
4. **半精度溢出**。fp16 上限是 65504，注意力 logits 或大 batch 的平方和很容易超；bf16 范围与 fp32 同阶，不会上溢，但精度低，全 mask 行仍会产生 `nan`。

定位用两步：在 `loss.backward()` 之前断言 `torch.isfinite(loss)`，把"哪一步开始坏"这一事实固定下来；再用 `set_detect_anomaly(True)` 拿到产生 `nan` 的前向算子名。梯度是否爆炸则直接看 `clip_grad_norm_` 的返回值——它返回裁剪前的总范数，正常在 0.1 到 10 之间，突然跳到 `1e4` 就是爆炸的实证。

```python
scaler = torch.amp.GradScaler("cuda", enabled=True)   # 仅 fp16 需要，bf16 可设 False
bad_steps = 0

for batch in loader:
    optimizer.zero_grad(set_to_none=True)
    with torch.amp.autocast("cuda", dtype=torch.float16):
        loss = model(**batch).loss
    scaler.scale(loss).backward()
    scaler.unscale_(optimizer)                        # 先反缩放，裁剪阈值才有意义
    gnorm = torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)
    if not torch.isfinite(gnorm):
        # 梯度非有限：跳过本次更新，让 scaler 降低 scale，绝不能让 nan 进入参数
        optimizer.zero_grad(set_to_none=True)
        scaler.update()
        bad_steps += 1
        if bad_steps > 5:                             # 连续多次就是真 bug，不是偶发溢出
            raise RuntimeError("gradient not finite for 6 consecutive steps")
        continue
    scaler.step(optimizer)
    scaler.update()
```

| 症状 | 最可能的原因 | 一步验证 |
| --- | --- | --- |
| 第 1 步就是 nan | 输入数据含 nan/inf | `torch.isfinite(x).all()` |
| 几十步后突然 nan | lr 太大或 fp16 溢出 | lr 除以 10、换 bf16 各跑一次 |
| 参数正常但 loss `inf` | `log(0)` 或除零 | 在 `log`/除法前后加 `clamp_min` |
| 只有某层出现 nan | 该层输入被全 mask 或数值范围异常 | 用 hook 打印该层 `min/max` |

### 10.6 OOM：先读懂那行数字

显存不足（Out of Memory，显存装不下当前计算所需的张量）的报错自带账本，学会读它比调参更快：

```text
torch.cuda.OutOfMemoryError: CUDA out of memory. Tried to allocate 2.00 GiB.
GPU 0 has a total capacity of 23.65 GiB of which 1.20 GiB is free.
21.90 GiB is allocated by PyTorch, and 380.00 MiB is reserved by PyTorch but unallocated.
```

四段数字分别对应：本次要申请多少、"总量与剩余"、"PyTorch 已分配给张量多少"、"PyTorch 缓存中闲置多少"。`allocated` 远小于 `reserved` 说明是碎片或缓存未归还；`allocated` 本身就顶到容量，说明张量真的多，缩 batch 才有意义。

要区分两种 OOM，它们的可恢复性完全不同：

| 消息 | 来源 | 可恢复性 | 处置 |
| --- | --- | --- | --- |
| `torch.cuda.OutOfMemoryError` | PyTorch 缓存分配器 | 可 `try/except` 捕获后换小 batch 重试 | `empty_cache()` 后降 batch |
| `CUDA error: out of memory` | cuDNN/cuBLAS 的 workspace 或驱动层 | 进程状态可能已损坏 | 重启进程，减小 `batch` 或关掉 `benchmark` |

定位分三步：先确认卡上有没有别的进程（`nvidia-smi`）；再把 `batch=1`、序列截到最短，仍 OOM 就说明权重与优化器状态本身超了；最后用内存快照看时间轴上谁在涨：

```python
torch.cuda.memory._record_memory_history(max_entries=100000)
# ... 跑几个 step ...
torch.cuda.memory._dump_snapshot("mem.pickle")   # 上传到 pytorch.org/memory_viz 可视化
```

快照能直接回答"是激活涨、还是我在 list 里存了带梯度的 loss"。至于把显存降下来用什么手段，属于优化范畴；本节只解决"这行数字在说什么、卡在哪一层"。

还有一种常被误判成显存的错误：`RuntimeError: [enforce fail at alloc_cpu.cpp] DefaultCPUAllocator: can't allocate memory`。它说的是主机内存（RAM）不足，不是显卡显存。成因是 `num_workers` 太多、每个 worker 各自持有一份数据集副本；或把整批图片预先解码成大张量留在内存里。判据很简单：`nvidia-smi` 显示显存充裕而进程被 OOM Killer 杀掉或抛出上述错误，就是内存问题，处置方向是减少 worker 数、把 `pin_memory` 关掉、用流式数据集，与显存优化无关。

### 10.7 DataLoader 卡住与死锁

数据加载卡住的典型特征是 GPU 利用率掉到 0、某个 CPU 核跑满、进度条不动，而且往往不报错。按现象对照：

| 现象 | 原因 | 处置 |
| --- | --- | --- |
| Windows/macOS 上 `num_workers>0` 直接抛 `An attempt has been made to start a new process...` | 子进程以 spawn 方式重启主模块，缺 `__main__` 保护 | 训练入口包进 `if __name__ == "__main__":` |
| `DataLoader worker (pid xxx) is killed by signal: Bus error` | Docker 的 `/dev/shm` 太小（默认 64MB） | 启动容器加 `--shm-size=8g`，或降 `num_workers` |
| `Cannot re-initialize CUDA in forked subprocess` | 在 worker 里用了 CUDA | 把张量搬运留在主进程，worker 只做 CPU 解码 |
| 每个 epoch 越来越慢、内存线性上涨 | worker 里把全量数据缓存进了实例属性 | 缓存放到主进程，或限制 `cache_size` |
| 每轮开头停顿数秒 | worker 反复重建 | `persistent_workers=True`（需 `num_workers>0`） |

排查工具是 `timeout` 参数：它把"静默卡死"变成"立刻报错"，错误信息里带上 worker 的 PID 和栈。

```python
import time

def probe(dataset, n=20):
    # 主进程串行遍历，先确认 Dataset 本身不慢、不卡
    t0 = time.time()
    for i in range(min(n, len(dataset))):
        dataset[i]
    print(f"serial {n} samples: {time.time() - t0:.2f}s")

# timeout=60：任一 worker 超过 60 秒没交出一个 batch 就抛异常并杀进程
loader = torch.utils.data.DataLoader(
    dataset, batch_size=32, num_workers=4, timeout=60,
    persistent_workers=True, prefetch_factor=2,
)
```

`probe` 快而 `DataLoader` 慢，问题在多进程或锁页拷贝；两者都慢，问题在 `Dataset.__getitem__`（磁盘、解码、远程请求）。这条对照能把范围一刀切开。

多进程下还有两个容易忽略的细节。一是每个 worker 的随机种子默认由主进程派生，如果 `__getitem__` 里做了随机增强，不同 worker 可能拿到相同的种子序列，增强的多样性打了折扣，正确做法是写 `worker_init_fn`，用 `torch.initial_seed()` 结合 worker 编号重新播种。二是 worker 数量不是越多越好：`num_workers` 超过 CPU 物理核数后，进程切换成本会盖过并行收益，图像解码这类 CPU 密集任务建议设在核数的 70%~100%，轻量任务设 4 就够。

### 10.8 CUDA 与编译错误

环境类报错几乎都能用三条命令定位：`nvidia-smi` 看驱动、`python -c "import torch; print(torch.__version__, torch.version.cuda)"` 看 runtime、`nvcc --version` 看编译器。三者版本必须同向匹配，混装是九成环境问题的来源。

| 报错原文（节选） | 原因 | 处置 |
| --- | --- | --- |
| `no kernel image is available for execution on the device` | wheel 里的编译架构不含本卡算力 | 装匹配 CUDA 版本的官方 wheel；用 `torch.cuda.get_arch_list()` 核对 |
| `CUDA driver version is insufficient for CUDA runtime version` | 驱动旧于 runtime | 升级驱动，或改装低 CUDA 版本的 wheel |
| `libcudart.so.12: cannot open shared object file` | 多份 CUDA runtime 混装或 `LD_LIBRARY_PATH` 指错 | `pip list` 查 `nvidia-*`，清干净后按官方命令重装 |
| `undefined symbol: ...` | torch 与 torchvision/自定义扩展 ABI 不匹配 | 同一条 pip 命令装齐同批次版本 |
| `CUDA error: device-side assert triggered` | 内核断言失败，最常见是 embedding 或标签索引越界 | 开 `CUDA_LAUNCH_BLOCKING=1` 重跑，修索引范围 |
| `nvcc: command not found` | 只装了 runtime，没有 toolkit | 装 `cuda-toolkit`，或改用预编译 wheel |

`device-side assert triggered` 要特别注意：标签越界（比如 10 类却出现 label=10）会先写坏 CUDA 上下文，之后再跑什么都会失败，必须重启进程。它与"形状对不上"不同，不会在 Python 侧给你清晰提示，所以看到它就先怀疑索引。

自己编译扩展时，报错多半落在工具链而非代码上：`RuntimeError: Ninja is required to load C++ extensions` 说明缺 `ninja`（`pip install ninja` 即可）；`error: command 'gcc' failed` 在 Windows 上是 MSVC 版本不足，在 Linux 上常是 gcc 版本高于 CUDA 支持范围；`the detected CUDA version mismatches the version that was used to compile PyTorch` 则是本地 toolkit 与 torch 自带的 runtime 版本不一致，要么改用同一版本，要么确认只做运行不做编译。判断该不该编译：只使用官方算子就不需要任何编译器，只有写自定义 CUDA 算子或用 `torch.utils.cpp_extension.load` 才需要完整工具链。

### 10.9 不收敛的排查顺序

"loss 不降"不是一种病因，而是一堆病因的共同症状。顺序很重要，从"证明代码正确"开始，而不是从调超参开始。

看曲线要分两个时间尺度。百步之内看的是"有没有在学"：正常训练前 100 步 loss 应有明显下降，一条水平线说明第 1 步或第 4 步有问题。跨 epoch 看的是"学得好不好"：训练 loss 降而验证 loss 升是过拟合，两者都不降才是配置或代码问题。把这两件事混在一起看，最常见的误判是把"学习率太小"当成"模型容量不够"，于是去加深网络。

1. **能否过拟合一个 batch**。取 32 条样本反复喂 200 步，loss 应降到接近 0。做不到，就是数据、标签、损失或反传有硬 bug，此时调 lr、换优化器全是徒劳。
2. **打乱标签做对照**。正常模型在标签打乱后准确率应掉到随机水平（`1/类别数`）。如果指标没变化，说明标签没被真正用上，或评估代码在自欺欺人。
3. **学习率扫描**。从 `1e-5` 到 `1e-1` 各跑一百步，比较 loss 曲线：都不降看第 1 步，只有大 lr 震荡看第 4 步。
4. **看梯度**。逐层打印 `grad.norm()`：全为 0 说明被 `detach` 或冻结，数量级超过 `1e3` 说明需要裁剪或降 lr。
5. **看模式与预处理**。忘写 `model.train()` 会让 Dropout 全程生效、BN 不更新统计量；验证集误用随机增强、归一化参数与预训练权重不一致，都会表现为"训练集能降、验证集不动"。
6. **看指标实现**。`argmax` 用错维度、累加后除以 batch 数而非样本数、多分类误用二分类损失，都是"loss 正常但指标离谱"的常见原因。

```python
def overfit_one_batch(model, batch, steps=200, lr=1e-3):
    """万能探针：连 32 条样本都记不住，问题一定在代码，不在数据分布"""
    opt = torch.optim.AdamW(model.parameters(), lr=lr)
    model.train()
    for i in range(steps):
        opt.zero_grad(set_to_none=True)
        out = model(**batch)
        loss = getattr(out, "loss", None)
        if loss is None:
            loss = torch.nn.functional.cross_entropy(out, batch["labels"])
        loss.backward()
        opt.step()
        if i % 50 == 0:
            print(f"step {i:3d} loss {loss.item():.4f}")
    return loss.item()   # 期望 < 0.05；否则先修 bug，再谈调参
```

顺带一个高频陷阱：`optimizer` 必须在模型搬到 GPU、并且所有需要训练的参数都注册好之后再构造。先建优化器再 `model.to("cuda")` 一般没事（参数对象不换），但如果之后又 `model.half()` 或替换了子模块，优化器里持有的还是旧参数，更新会静默失效。

### 10.10 报错-原因-对策速查表

| 报错关键词 | 一类原因 | 第一个动作 |
| --- | --- | --- |
| `shapes cannot be multiplied` | `Linear` 输入维度不符 | 打印上一层输出形状 |
| `must match ... non-singleton dimension` | 广播失败 | 对齐该维或显式 `unsqueeze` |
| `Expected 4-dimensional input` | 缺 batch 维 | `unsqueeze(0)` |
| `Expected all tensors to be on the same device` | 张量分居 CPU/GPU | 跑 `device_report` |
| `should be the same`（type） | 模型与数据 dtype 或设备不一致 | 统一 `model.to(device)` 与 `x.float()` |
| `expected scalar type Long` | 索引用浮点 | `.long()` |
| `one of the variables needed for gradient computation has been modified by an inplace operation` | 原地操作改掉了反传要用的张量 | 换非原地写法，或调换语句顺序 |
| `Trying to backward through the graph a second time` | 图已被释放又反传 | 需要多次反传才用 `retain_graph=True` |
| `CUDA out of memory` | 显存不足 | 读 `allocated`/`reserved` 再定策 |
| `device-side assert triggered` | 索引越界 | `CUDA_LAUNCH_BLOCKING=1` 重跑 |
| loss 变 `nan` | 数据异常、lr 过大、算子不稳定 | 断言 `isfinite` + 降 lr |
| loss 不降 | 数据/标签/损失/模式配置 | 先过拟合一个 batch |

### 10.11 常见坑

- 只盯着栈的最后一行，甚至读进 `torch/nn/modules/module.py` 里去找 bug；异常主体的第一行才是"为什么"，栈里第一个自己的帧才是"在哪"。
- 报错后第一反应是改小 batch、调低 lr，跳过最小复现；结果把标签越界、索引 dtype 错误这类硬 bug 当成"资源不足"，越调越远。
- 把 `set_detect_anomaly(True)`、`CUDA_LAUNCH_BLOCKING=1` 留在正式训练或性能基准里，速度慢数倍，还据此得出"这个模型很慢"的错误结论。
- 用 `try/except` 捕获 OOM 加 `empty_cache()` 当成常态重试机制，掩盖了真实的显存增长（比如把带梯度的 loss 累加进 list），线上表现为越跑越慢、最后必崩。

## 11. 生态与工程实践

### 11.1 生态分层：先判断你要补的是哪一层

PyTorch 本体只提供四样东西：张量、自动求导、优化器、`nn.Module`。凡是不属于这四样的能力，都由周边库提供。把常用库按职责分成四层，选型时就不容易装一堆功能重叠的包。

| 层次 | 代表库 | 解决什么问题 | 不引入的代价 |
| --- | --- | --- | --- |
| 数据与增强 | torchvision、torchaudio、albumentations | 数据集封装、图像解码、增强流水线 | 自己写解码与增强，易与预训练权重预处理不一致 |
| 模型与权重 | torchvision.models、timm、transformers | 现成网络结构 + 预训练权重 | 从零搭结构、无预训练、小数据训不动 |
| 训练流程 | PyTorch Lightning、Accelerate、torchrun | 训练循环、混合精度、分布式、断点续训 | 每个项目重写一遍模板代码，出错点多 |
| 观测与配置 | TensorBoard、W&B、Hydra、pytest | 指标记录、参数注入、自动化验证 | 实验结果无法复现，改动靠口头传达 |

判断规则很直接：能靠一层库解决的，不要叠第二层。例如只做图像分类，`timm` 已经包含骨干网络、预训练权重和配套的预处理配置，不需要再引 `transformers`；只做单卡小模型训练，手写循环比引入 Lightning（即 PyTorch Lightning，把训练循环、精度与分布式收敛进 `Trainer` 的框架）更省认知成本。反过来，一旦训练要跑多机多卡、要按论文复现实验、要给别人交接，把流程层和观测层补上就是划算的。

选型还有一条硬约束是版本对齐。`torchvision`、`timm`、`transformers` 都直接依赖 `torch` 的内部 API，版本错配时报错往往出现在 import 阶段甚至运行中途（`undefined symbol`、`torch.classes` 找不到）。稳妥做法是三者与 `torch` 同批升级，升级后立刻跑一遍冒烟测试，而不是在生产镜像上单独升某一个包。

### 11.2 torchvision：数据管道与 transforms v2

torchvision（PyTorch 官方计算机视觉扩展库，提供数据集、图像变换和经典网络结构）是视觉任务的地基。它有两套变换 API：老的 `transforms`（基于 PIL，单张处理）和 `transforms.v2`（同时支持图像、视频、边界框、掩码，且能作用于 batch）。新项目直接用 v2，因为检测与分割任务需要标签随图像一起变换，老 API 做不到这一点。

```python
import torch
from torchvision.transforms import v2

train_tf = v2.Compose([
    v2.RandomResizedCrop(224, antialias=True),   # 不加 antialias 会让边缘出现锯齿，验证精度略降
    v2.RandomHorizontalFlip(),
    v2.ToImage(),                                # 统一成 Tensor 图像类型
    v2.ToDtype(torch.float32, scale=True),       # uint8 0-255 → float32 0-1，scale 必须开
    v2.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
])
```

`v2.ToDtype(..., scale=True)` 是最常被漏掉的一步：不缩放的话输入是 0-255 量级，第一层卷积的输出放大近 255 倍，前几百步都在把权重往回拉，表现为 loss 长时间不降。顺序也有讲究：归一化必须放在 `ToDtype` 之后，否则在整型上做浮点运算会被截断。

与 `DataLoader` 结合时，增强只在训练集开，验证与测试只做 resize + 归一化，且验证的 resize 策略要与权重训练时一致（ImageNet 权重通常是短边 256、中心裁 224）。把 `train_tf` 和 `val_tf` 写成两个常量并在 `Dataset` 里按 `is_train` 选择，比在循环里改全局变换更安全。

增强强度要和训练轮数匹配。小数据集上用 `RandomResizedCrop(scale=(0.08, 1.0))` 这种强增强，前几个 epoch 指标会明显偏低，属于正常现象；但如果 30 轮之后验证指标仍低于不加增强的基线，说明增强过强或数据量不足以支撑，应该退回较弱策略。另一个高频问题是 `num_workers > 0` 时 `Dataset` 里的随机数落在子进程，主进程设置的种子管不到它，增强序列在多次运行间不同，复现失败。标准做法是在 `worker_init_fn` 里按 `torch.initial_seed()` 重置该 worker 的随机种子，并在 `DataLoader` 上显式传入。

### 11.3 timm：骨干网络与「配套预处理」

timm（PyTorch Image Models，第三方视觉模型库，收录上千个骨干网络及其预训练权重）解决的是「换个骨干看效果」这个高频需求。一行 `create_model` 就能拿到任意结构，但真正容易出错的是预处理：每个权重都对应一套自己的输入尺寸、归一化参数和插值方式，用错就会掉几个点且极难归因。

```python
import timm

model = timm.create_model("resnet18", pretrained=True, num_classes=10)
model.eval()

cfg = timm.data.resolve_model_data_config(model)   # 从权重的 config 里读出真实预处理参数
tf = timm.data.create_transform(**cfg, is_training=False)
print(cfg["input_size"], cfg["mean"], cfg["std"], cfg["interpolation"])
```

`resolve_model_data_config` 读的是权重自带的元数据，所以换任何模型都不用改预处理代码。上线时要把这份 `cfg` 和权重一起存进产物目录，推理服务按它构造变换，避免训练用一套、服务用另一套。另外 `pretrained=True` 会把权重下载到 `~/.cache/torch/hub`，离线环境要先下载好再用 `pretrained_cfg_overlay=dict(file="...")` 指向本地文件。

迁移学习时还有两个细节。第一，指定 `num_classes` 会重建分类头并随机初始化，头部的学习率通常要比骨干大一个量级，用参数分组实现而不是全局一个 `lr`。第二，冻结骨干要连 BatchNorm 一起处理：只把 `requires_grad` 关掉、不切 `eval()`，BN 的 running 统计量仍会被当前 batch 更新，小 batch 下会把预训练统计量污染掉，微调效果反而变差。

```python
import timm, torch

model = timm.create_model("resnet18", pretrained=True, num_classes=10)
head = model.get_classifier()                        # 分类头的名字随架构变（resnet 是 fc、vit 是 head）
head_ids = {id(p) for p in head.parameters()}        # 按对象身份划分，不靠名字前缀猜
backbone = [p for p in model.parameters() if id(p) not in head_ids]
opt = torch.optim.AdamW([
    {"params": backbone, "lr": 1e-4},                      # 骨干小学习率，避免破坏预训练特征
    {"params": list(head.parameters()), "lr": 1e-3},       # 新初始化的头需要更快收敛
], weight_decay=0.01)
```

同一个参数绝不能同时出现在两个参数组里，PyTorch 会直接抛 `some parameters appear in more than one parameter group`；因此按名字前缀过滤（如 `name.startswith("head")`）是危险的写法，换成 VGG 或 ViT 就失效，按对象身份排除才与架构无关。

### 11.4 transformers：AutoClass 与权重持久化

transformers（Hugging Face 维护的预训练模型库，用统一接口加载上千种模型结构、分词器与权重）是 NLP 与多模态任务的默认入口。它的核心抽象是 AutoClass：`AutoModel`、`AutoTokenizer`、`AutoConfig` 读 `config.json` 里的 `model_type` 与 `architectures` 字段，动态决定实例化哪个具体的类，所以同一段代码可以加载 BERT、RoBERTa、DeBERTa。

```python
from transformers import AutoTokenizer, AutoModelForSequenceClassification

name = "distilbert-base-uncased"
tok = AutoTokenizer.from_pretrained(name, cache_dir="./.hf")      # 缓存进项目目录，CI 可复用
model = AutoModelForSequenceClassification.from_pretrained(name, num_labels=2)

batch = tok(["训练不收敛", "loss 正常下降"], padding=True,
            truncation=True, max_length=128, return_tensors="pt")   # truncation 必须显式开
logits = model(**batch).logits                                     # (2, 2)

model.save_pretrained("./ckpt/distilbert-cls")   # 存 config + 权重，不用 torch.save(model)
tok.save_pretrained("./ckpt/distilbert-cls")     # 分词器必须同目录保存
```

三件事必须同时做对才能上线：一是分词器与权重一起保存，只存模型会导致线上分词规则（大小写、子词切分、特殊 token）与训练时不同；二是 `max_length` 与训练时一致，推理端截断长度变化会让长文本的分数分布漂移；三是锁版本，`transformers` 的小版本升级会改变注意力实现、padding 侧默认值和生成参数默认值，同一份权重在不同版本下输出不同。依赖里写精确版本（`transformers==4.44.2`），并把整个 `ckpt` 目录作为二进制产物归档。

上线前建议先统计一次 token 长度分布再定 `max_length`，而不是拍一个 512：把语料按 128、256、512 分桶统计被截断的样本比例，如果 512 之外仍有超过 5% 的样本被截断，要么加长度，要么在业务上截断长文本并告知调用方。截断比例是个容易被忽略的指标，它不体现在验证集 loss 上，只体现在线上长文本的准确率下降里。

需要一个通用的训练器时，`transformers.Trainer` 可以直接接管 `Dataset`、评估与保存，代价是它对非标准流程的改造比较费劲，而且每个版本的参数名会调整，升级前要读 changelog。

### 11.5 Lightning 与手写训练循环的取舍

PyTorch Lightning（基于 PyTorch 的训练流程封装框架，把循环、分布式、混合精度、checkpoint 收敛到一组约定好的钩子里）让「换策略不改业务代码」成为可能。是否引入它，取决于训练流程能不能被「一次 `fit`」描述。

| 维度 | 手写循环 | Lightning |
| --- | --- | --- |
| 上手成本 | 低，代码即流程 | 需理解 hook 顺序与 Trainer 行为 |
| 分布式 | 自己配 DDP、sampler、保存 | `Trainer(strategy="ddp")` 一行 |
| 混合精度 | 手写 `autocast` + `GradScaler` | `precision="16-mixed"` |
| 断点续训 | 自己存全部状态 | `ckpt_path` 自动恢复 |
| 调试 | 断点随意打 | 需要进入 `fit` 内部，稍绕 |
| 适合 | 研究原型、非常规流程（RL、多模型交替、外部环境交互） | 标准监督训练、多卡、要交接的实验 |

```python
import lightning as L
import torch

class Classifier(L.LightningModule):
    def __init__(self, lr=3e-4):
        super().__init__()
        self.save_hyperparameters()          # 超参写进 ckpt，复现时不用翻日志
        self.net = torch.nn.Linear(768, 2)

    def training_step(self, batch, idx):
        logits = self.net(batch["x"])
        loss = torch.nn.functional.cross_entropy(logits, batch["y"])
        self.log("train_loss", loss, prog_bar=True, on_step=True, on_epoch=True)
        return loss

    def configure_optimizers(self):
        return torch.optim.AdamW(self.parameters(), lr=self.hparams.lr)

trainer = L.Trainer(max_epochs=10, precision="16-mixed",
                    accelerator="auto", devices="auto", log_every_n_steps=20)
trainer.fit(model, datamodule=dm)   # 反向、梯度裁剪、保存都交给 Trainer
```

混用是可行且常见的：LightningModule 内部照样可以调用外部模拟器做 rollout，只是把「一次迭代」的边界想清楚。真正的坑是两边都管：在 `training_step` 里又手动 `optimizer.step()`，梯度会被更新两次，日志步数也会翻倍。

### 11.6 实验追踪：TensorBoard 与 W&B

TensorBoard（TensorFlow 出品的可视化工具，PyTorch 通过 `torch.utils.tensorboard` 写入标量、图像、直方图）适合本地、内网、无外网环境。W&B（Weights & Biases，云端实验追踪平台，记录指标、配置与产物并支持跨实验对比）适合多人协作。

```python
from torch.utils.tensorboard import SummaryWriter

writer = SummaryWriter(log_dir=f"runs/{exp_name}")
for step, batch in enumerate(loader):
    loss, acc = train_one_step(batch)
    writer.add_scalar("loss/train", loss, step)
    writer.add_scalar("acc/train", acc, step)
    writer.add_scalar("lr", opt.param_groups[0]["lr"], step)   # 学习率必记，排查不收敛先看它
writer.add_graph(model, torch.randn(1, 768))   # 只放一次，否则文件迅速膨胀
writer.close()                                  # 不 close 会丢最后一段缓冲区
```

```python
import wandb

wandb.init(project="cls", name=exp_name,
           config={"lr": 3e-4, "batch_size": 64, "backbone": "resnet18"})
wandb.log({"loss/train": loss, "acc/train": acc}, step=step)
wandb.finish()
```

无论用哪个，每次实验至少要留下三样东西：指标曲线、超参、代码与数据版本（git commit、数据快照 id、依赖锁文件）。缺了 commit 的曲线在一周后无法解释，也无法判断两次实验是否真的只有一处不同。日志策略上，`add_scalar` 的 step 用全局步而不是 epoch，短训练才看得出前几十步的异常。

| 能力 | TensorBoard | W&B |
| --- | --- | --- |
| 部署方式 | 本地或内网，`tensorboard --logdir runs` 起服务 | SaaS 或自托管，需要账号与网络 |
| 实验对比 | 手工勾选多个 run，弱 | 内置表格、并行坐标、swarm 视图 |
| 超参与产物 | 自己写文本或 hparams | `config`、artifact 版本原生支持 |
| 团队协作 | 分享目录或端口 | 链接即分享，带评论 |
| 离线环境 | 完全可用 | 需 `wandb offline` 后手动同步 |

选择策略简单：单机、离线、只是自己看趋势，TensorBoard 足够；多人对比几十组实验、需要把数据集与权重做版本化，W&B 的表格与 artifact 省下的时间远超接入成本。两者不冲突，可以同时写：TensorBoard 给本地调试，W&B 给团队留档。

### 11.7 配置管理：argparse、YAML、Hydra

argparse（Python 标准库自带的命令行解析器，把 `--key value` 解析成对象）在参数十个以内时最省事。超过十个、或者要做网格搜索、要把配置入库时，把参数写进 YAML（一种用缩进表达层级结构的文本配置格式）文件，再用 dataclass（Python 标准库的类装饰器，用于声明字段、类型与默认值）读出并校验，胜过无数个 `--flag`。

```yaml
# configs/base.yaml
seed: 42
train:
  epochs: 30
  batch_size: 64
  lr: 3.0e-4
model:
  name: resnet18
  num_classes: 10
```

```python
from dataclasses import dataclass, field
import yaml

@dataclass
class TrainCfg:
    epochs: int = 30
    batch_size: int = 64
    lr: float = 3e-4

@dataclass
class Cfg:
    seed: int = 42
    train: TrainCfg = field(default_factory=TrainCfg)

def load_cfg(path: str) -> Cfg:
    with open(path, encoding="utf-8") as f:
        raw = yaml.safe_load(f)
    return Cfg(seed=raw["seed"], train=TrainCfg(**raw["train"]))  # 嵌套显式转换，避免到处是 dict
```

命令行参数只做「少量覆盖」，主体仍从配置文件读，这样既保留了快速试参的便利，也不会让参数散落到十几个 flag 里：

```python
import argparse

def parse_args():
    p = argparse.ArgumentParser()
    p.add_argument("--config", default="configs/base.yaml")
    p.add_argument("--lr", type=float, default=None)     # 默认 None 表示「不覆盖配置文件」
    p.add_argument("--dry-run", action="store_true")     # 只跑 10 步，用于验证管道
    args = p.parse_args()
    cfg = load_cfg(args.config)
    if args.lr is not None:
        cfg.train.lr = args.lr                           # 命令行优先级高于配置文件
    return cfg, args
```

Hydra（Facebook 开源的配置组合框架，支持分层配置、命令行覆盖和自动实验目录）适合多组实验：命令行写 `python train.py train.lr=1e-3 train.batch_size=32 seed=123` 就能覆盖任意层级字段，运行目录自动切到 `outputs/日期/时间/`，每次实验的配置都被完整快照，天然可回溯。

Hydra 的默认行为也最容易踩：它会改变进程工作目录，脚本里所有 `./data`、`./ckpt` 相对路径都会指向 `outputs/...` 而报文件不存在。解决办法是在装饰器里加 `hydra.job.chdir=False`，或所有路径统一用 `${hydra:runtime.cwd}` 拼绝对路径。另外 `yaml.safe_load` 会把 `3e-4` 这种写法解析成字符串而非浮点数，dataclass 不做类型强制，优化器拿到字符串后训练直接崩在 `lr` 类型上，加载后应显式 `float()` 转换。

### 11.8 目录组织：入口只有一个

工程化的第一标志是「可执行入口唯一、逻辑可被 import」。下面的结构适用于绝大多数训练项目，配置与代码分离，测试与训练脚本分离。

```text
project/
├── configs/              # 只放 yaml，不放代码
├── src/
│   ├── data/             # Dataset、collate_fn、增强
│   ├── models/           # nn.Module 定义，可单独 import
│   ├── engine/           # train_one_epoch / evaluate
│   ├── utils/            # seed、logging、metrics
│   └── train.py          # 唯一入口：读配置 → 建对象 → 跑循环
├── tests/
├── scripts/              # 启动、导出 ONNX、评测
├── requirements.txt      # 运行依赖，精确版本
├── requirements-dev.txt  # 测试与静态检查工具
└── README.md             # 一条命令能复现主结果
```

模型类必须定义在 `src/models/` 而不是 `train.py` 里。写在训练脚本里，测试要么 import 整个脚本（连带执行一遍参数解析），要么复制一份结构，两种都会导致「测试通过但训练用的不是同一个模型」。同理，`train_one_epoch` 与 `evaluate` 要能在无 `Trainer`、无命令行参数的环境里被调用，这是能写单测的前提。

`src` 能被直接 import 依赖两件事：包里有 `__init__.py`（或在 `pyproject.toml` 里声明 `packages`），以及仓库根目录在 `sys.path` 上。以 `pip install -e .` 安装成本地包是最省心的做法，比在每个脚本顶部 `sys.path.append` 更可靠，因为后者在 CI 与容器里的工作目录经常不同。

依赖分三个文件维护：`requirements.txt` 放运行必需且版本精确的包；`requirements-dev.txt` 放 pytest、ruff、pre-commit 这类只在开发与 CI 用的工具；镜像或 `environment.yml` 记录 CUDA 与系统库版本。数据、权重、`runs/` 目录一律写进 `.gitignore`，仓库只存代码、配置与切分脚本，否则仓库会在几周内膨胀到无法 clone。

### 11.9 单测与 CI

pytest（Python 最常用的单元测试框架，按文件名与函数名约定自动发现用例）里，训练代码的测试分三层，从上到下成本递增、价值递增。第一层形状测试：喂极小张量，断言输出 shape 与 dtype。第二层数据管道测试：取一个 batch，断言不含 NaN、标签范围合法、增强后 dtype 仍是 float32。第三层冒烟测试：用 8 个样本训 200 步，断言能过拟合到接近零损失——连 8 个样本都学不会，说明存在比超参更基础的 bug。

```python
# tests/test_model.py
import torch
from src.models.mlp import MLP

def test_forward_shape():
    out = MLP(in_dim=16, num_classes=3)(torch.randn(4, 16))
    assert out.shape == (4, 3) and out.dtype == torch.float32

def test_overfit_tiny_batch():
    torch.manual_seed(0)                     # 固定种子，避免偶发失败
    m = MLP(in_dim=16, num_classes=3)
    opt = torch.optim.Adam(m.parameters(), lr=1e-2)
    x, y = torch.randn(8, 16), torch.randint(0, 3, (8,))
    for _ in range(200):
        loss = torch.nn.functional.cross_entropy(m(x), y)
        opt.zero_grad(); loss.backward(); opt.step()
    assert loss.item() < 0.05
```

CI（持续集成，代码提交后自动执行检查与测试的流水线）要跑在 CPU 上，并显式装 CPU 版 PyTorch，镜像小、启动快，也避免为了跑单测去申请 GPU 机器。

```yaml
# .github/workflows/ci.yml
name: ci
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - run: pip install torch --index-url https://download.pytorch.org/whl/cpu
      - run: pip install -r requirements-dev.txt
      - run: pytest -q tests          # 冒烟测试控制在 5 分钟内
```

CI 最怕两件事：跑得太慢和偶发失败。慢的根因通常是单测里加载了预训练权重或跑了真实数据集，正确做法是测试用随机初始化的小模型加合成张量，需要真实权重的用例打上 `@pytest.mark.slow` 并在夜间流水线单独跑。偶发失败的根因几乎都是随机性没固定，或者用浮点数做精确比较，前者靠种子，后者改成 `torch.allclose(atol=1e-5)`。

代码风格检查放进同一流水线，成本几乎为零：`ruff check src tests` 与 `ruff format --check` 能拦下未使用变量、可变默认参数、裸 `except` 这类问题，`pre-commit` 让它在提交前就跑一次，比在 review 里反复提同一句更有效。mypy 对张量形状无能为力，所以类型检查的收益主要在配置对象与函数签名上，可选。

### 11.10 从实验到生产的检查清单

实验能跑通和生产可用之间隔着一整套约束。发布前逐条核对下表，任何一条不通过都不应上线。

| 类别 | 检查项 | 通过标准 |
| --- | --- | --- |
| 可复现 | 随机种子 | `random`、`numpy`、`torch`、`cuda` 四处都设，值写进配置 |
| 可复现 | 环境锁定 | 依赖精确版本 + 镜像 tag，不使用 `latest` |
| 数据 | 训练/验证隔离 | 同一实体不跨集，切分脚本入库而非手工 |
| 模型 | 预处理绑定权重 | 归一化参数、输入尺寸随权重一起持久化 |
| 模型 | 导出等价性 | 导出成 ONNX（跨框架模型交换格式）或 TorchScript（PyTorch 自带序列化格式）后，与 eager 输出最大误差小于 1e-4 |
| 性能 | 推理延迟 | 目标硬件上实测 P95 达标，而非只看单条耗时 |
| 观测 | 可追溯 | 由一次线上预测能反查到权重版本、配置、commit |

其中「导出等价性」最容易被跳过：`model.eval()` 与 `torch.no_grad()` 之外，动态 shape、`interpolate` 的 `align_corners`、自定义算子在导出后都可能变味，必须用同一批输入对比两份输出，而不是只看导出的文件能否生成。

上线本身也要设计退路。新模型先跑影子流量（同一批请求同时打给新旧两版，只记录不返回），对比两者的预测分布与延迟，再切 5% 真实流量观察一天。回滚条件要提前写死并量化，例如「核心指标下降超过 1 个百分点，或 P95 延迟上涨超过 20% 持续 10 分钟，自动回滚到上一版权重」。权重文件、预处理配置、分词器、推理代码要打成一个不可变版本号（如 `cls-2024-06-01-a`），回滚只改指向，不重新构建，才能把恢复时间压到分钟级。

### 11.11 常见坑

- **预处理与权重不配套**：手写 `mean/std` 或换了 resize 尺寸，精度掉几个点却查不出原因。用 `timm.data.resolve_model_data_config` 或权重自带的 `preprocessor_config.json` 生成变换，并把它和权重一起归档。
- **只存模型不存分词器**：`save_pretrained` 只调了模型，线上分词规则与训练不同，短文本分数抖动、长文本被静默截断。模型与分词器必须保存在同一目录并整体发布。
- **不锁版本导致结果漂移**：`transformers`、`timm` 小版本升级会改默认注意力实现与预处理行为，同一份权重输出不同。依赖写精确版本，并在 CI 里跑一次固定输入的数值回归。
- **Lightning 与手写 step 双管**：在 `training_step` 里手动 `optimizer.step()`，梯度更新两次、日志步数翻倍，表现为 loss 下降异常快而指标不涨。优化器只在 `configure_optimizers` 创建，步进交给 `Trainer`。
