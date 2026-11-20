---
title: OOM 的排查链路
topic: 显存问题-核心题
sub_topic: 四：常见显存问题诊断与调优
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/BbCrwkO3wiWmAikFy0zc2OK6nPg
source_path: 显存问题-核心题/四：常见显存问题诊断与调优/OOM 的排查链路.md
questions: 3
chars: 2882
build_date: '2026-11-20'
images_removed: 0
---

# OOM 的排查链路

> 主题：显存问题-核心题 › 四：常见显存问题诊断与调优　｜　原始文件：显存问题-核心题/四：常见显存问题诊断与调优/OOM 的排查链路.md

## 题1：训练过程中突然 OOM，你应该从哪些地方开始排查？

💡 面对突然的 OOM，不要慌，沿着“数据 → 配置 → 代码 → 框架”的脉络逐一排查，通常能在几分钟内定位元凶。 我习惯按以下顺序执行：

1. 检查是否某个 batch 的数据异常

   突然 OOM 最常见的原因是数据集中出现了超长样本。若未对文本进行截断或 padding 策略不当，个别样本长度可能是平均的 3-5 倍，导致激活值瞬间暴涨。

- 排查方法：在 DataLoader 中打印当前 batch 的最大序列长度，或在 collate_fn 中记录统计。如果是语音或图像，检查是否有超高分辨率输入。
- 临时方案：开启动态 padding 或强制截断，确保所有样本不超过预设的最大长度。
1. 确认是否开启验证（eval）阶段

   训练每隔若干步会进行验证。验证时通常会关闭梯度计算（torch.no_grad()），但可能因为使用了更大的 batch size 或缓存了所有验证结果而导致显存超出。

- 特征：OOM 发生在第一个 eval batch 或 eval 结束时。
- 解决：减小 eval batch size，或在 eval 中只保留必要的指标而不缓存所有 logits。
1. 检查是否开启了不必要的缓存或日志
- 某些调试钩子（如记录每层梯度范数）可能意外保留了张量引用，导致显存泄漏。
- 使用 torch.cuda.memory_summary() 或 torch.cuda.memory_snapshot() 检查是否有大量张量未释放。
- 如果使用了 WandB/TensorBoard 记录直方图或嵌入，可能占用额外显存。
1. 查看显存碎片情况

   即使总空闲显存足够，但若缺乏连续大块，cudaMalloc 仍然可能失败。这种情况在长时间训练后容易出现。

- 指标：nvidia-smi 显示 Memory-Usage 接近 100%，而 torch.cuda.memory_allocated() 显示已分配却远小于总显存，说明碎片严重。
- 缓解：设置环境变量 PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True 或定期调用 torch.cuda.empty_cache()（谨慎使用，可能影响性能）。
1. 排查模型/优化器状态是否意外膨胀
- 如果使用了混合精度训练，检查 loss scaler 是否产生了 NaN/Inf，导致梯度更新异常，优化器状态可能积累非法值。
- 某些优化器（如 LAMB）可能在内部维护额外的缓冲区。
- 是否无意中解冻了一些原本冻结的参数（例如在 LoRA 中误把基础模型设为可训练）。
1. 审视并行策略与通信缓冲区
- DeepSpeed ZeRO 或 FSDP 在通信时可能临时需要额外缓冲区。如果同时使用多个并行策略，检查是否开启了 overlap_comm 等，其缓冲区可能超出预期。
- 在多卡训练中，OOM 可能只发生在某一张卡上，用 pdsh 或监控查看各卡显存使用是否均衡。
1. 回退最近的代码/配置变更

   突然 OOM 往往和最近的修改有关。对比上一次稳定训练的配置（学习率、batch size、序列长度、模型结构、是否开启新优化等）。

✅ 总结排查流程

数据 → 评估阶段 → 缓存/日志泄漏 → 碎片 → 模型状态 → 并行配置。遵循这个顺序，80% 的突然 OOM 能找到直接原因，并能快速给出应对措施。

## 题2：如何通过 nvidia-smi 观察显存变化趋势？

💡 nvidia-smi 是快速诊断 GPU 显存的瑞士军刀，通过持续监控、记录日志和观察模式变化，可以判断训练中的显存是稳定、泄漏、还是突发尖峰。

🔧 常用命令与技巧

1. 持续监控模式

```
nvidia-smi dmon -s pucm -d 2
```

- -s pucm：显示功耗、利用率、时钟、显存信息。
- -d 2：每 2 秒刷新一次。

  输出中 mem 列表示当前使用的显存百分比，观察其随时间的变化曲线：如果单调递增且不回落，极可能是内存泄漏；如果出现周期性尖峰，可能与数据加载或 checkpoint 保存相关。

1. 实时查看进程级显存

```
watch -n 1 nvidia-smi
```

观察 Processes 部分，找到自己的训练进程 PID，看 Memory-Usage 慢慢上升还是突然跳变。若能结合 nvtop 工具，效果更直观。

1. 记录显存日志以便事后分析

```
nvidia-smi --query-gpu=timestamp,memory.used,memory.total --format=csv -l 1 > gpu_log.csv
```

用 Python 读取 CSV 并绘图，可以精确看到 OOM 前的显存爬升曲线。如果是阶梯状上升，通常对应梯度累积或 optimizer step 的分配；如果是陡增，则对应某个大张量分配（如注意力矩阵）。

1. 识别 OOM 时的模式
- 突然垂直上升后崩溃：往往因为一个 batch 中出现了异常大的激活张量（如超长序列）。查看 log 中最后几个时间点的显存使用量，通常会在几秒内冲顶。
- 缓慢增长直至溢出：可能为内存泄漏（某些张量未释放），也可能是碎片积累。nvidia-smi 显示的 Used 包含了 PyTorch 缓存，可能掩盖碎片问题，需结合 PyTorch 自身 API 判断。
1. 检查温度与功耗相关性

   若显存占用高同时温度飙升，可能触发了硬件保护降频，导致训练变慢但显存不减，这虽不直接 OOM，但会间接影响性能。nvidia-smi 能同时观察温度、功耗和显存。

🔬 高级用法

- 通过 nvidia-smi nvlink -e 0 查看 NVLink 传输，辅助判断多卡通信是否导致显存占用。
- 结合 nvidia-smi -i 0 -q -d MEMORY 输出详细的显存信息，包括 BAR1 使用等。

✅ 总结：用 nvidia-smi 就像看心电图，把显存使用率绘制成时间序列，能最快速识别 OOM 的宏观类型，为进一步精细化定位提供方向。

## 题3：PyTorch 的 memory_allocated() 和 memory_reserved() 在 OOM 时如何帮助定位？

💡 当 OOM 发生时，torch.cuda.memory_allocated() 告诉你 PyTorch 实际在张量上占用了多少显存，memory_reserved() 告诉你 PyTorch 缓存分配器总共向 CUDA 驱动申请了多少显存。两者之差就是分配器内部的“空闲但不归还”的缓存。分析这两个值能区分是真实显存不足还是碎片/缓存导致的 OOM。
