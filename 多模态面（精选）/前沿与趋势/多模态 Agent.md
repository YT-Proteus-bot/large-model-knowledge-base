# 多模态 Agent

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/XaPWw0zuTilP7NkGkoBc1TFcnwc

## 多模态Agent：从GUI自动化到具身智能的感知、推理与行动

多模态Agent是现代AI从“感知理解”走向“自主行动”的前沿范式。它需要将视觉、语言、环境状态等信息统一起来，进行推理、规划并最终输出可执行的动作。以下深入剖析多模态Agent在GUI、Web、具身智能等场景中的核心架构、关键技术及挑战。

![image](../../../assets/J7UWdJkmAoTJAdxcoDscVYu7nac/WRojboHBko0h8vxymVhcyGvPnRg.png)

---

### 1. 多模态 Agent 在 GUI 自动化操作中，如何理解屏幕并生成正确的动作？

GUI自动化要求Agent像人一样“看”屏幕截图，理解界面元素布局和功能，然后输出精确的点击、拖拽、输入等操作。其技术栈通常包含视觉编码-语义理解-动作生成三阶段。

#### 1.1 屏幕感知：将像素转化为结构化语义

核心技术方案：

> （此处为内嵌电子表格，需登录飞书查看）

主流实践：SoM结合多模态LLM是目前工业界最常用方案。例如，用UI检测模型（如YOLO微调）提取所有可交互元素，框出并编号，生成标注后的截图。LLM看到标注截图后，输出“点击[14]”或“在[5]输入‘Hello’”，后处理层将ID映射为真实坐标并执行。

#### 1.2 从理解到动作：动作空间的抽象

GUI动作需要标准化为模型可输出的格式。典型动作空间定义为：

```
代码块
{
  "action": "CLICK" | "TYPE" | "SCROLL" | "SWIPE" | "LONG_PRESS" | "BACK",
  "element_id": 14,           // 对应SoM标注的ID
  "text": "input content",   // 用于TYPE动作
  "coordinate": [x, y],      // 备选：直接坐标
  "direction": "up" | "down" // 用于SCROLL/SWIPE
}
```

动作生成的两种范式：

- 端到端生成：将当前截图、历史操作序列、用户目标作为prompt，直接让LLM输出结构化动作JSON。
- 分离式生成：LLM输出高层次意图（“打开设置”），再由一个轻量级执行模块匹配底层UI坐标。

#### 1.3 错误恢复与自适应

GUI Agent常因弹窗、网络延迟、意外界面跳转而执行失败。鲁棒方案需在每一轮动作后重新截屏作为新的视觉观察，形成“观察-动作-再观察”的闭环。若动作执行后界面无预期变化（如弹窗未关闭），Agent可检测到异常并自动重试或改用替代策略。此即ReAct范式在视觉GUI中的直接应用。

---

### 2. Web Agent 使用 HTML 和屏幕截图作为输入，如何设计融合方案？

Web Agent拥有比App更丰富的输入：HTML DOM提供结构化语义和层级关系，截图提供视觉外观和布局信息。融合两者能结合“精确元素定位”与“视觉上下文理解”。

#### 2.1 融合架构

![image](../../../assets/J7UWdJkmAoTJAdxcoDscVYu7nac/AIlibBt4AoKppsxVjlkcVymhnsc.png)
