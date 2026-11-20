---
title: 输出解析器 (Output Parsers)
topic: LangChain面（高频）
sub_topic: 二、模型、提示与输出解析
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/FkhTw9yBsiro60k5CSIcZezcnj6
source_path: LangChain面（高频）/二、模型、提示与输出解析/输出解析器 (Output Parsers).md
questions: 2
chars: 3326
build_date: '2026-11-20'
images_removed: 1
---

# 输出解析器 (Output Parsers)

> 主题：LangChain面（高频） › 二、模型、提示与输出解析　｜　原始文件：LangChain面（高频）/二、模型、提示与输出解析/输出解析器 (Output Parsers).md

## LangChain面

> 〔原文此处为飞书内嵌组件，内容未随导出〕

## 题1：为什么要使用 Output Parser？它解决了直接从 LLM 获取结构化输出的哪些问题？

语言模型本质上是一个文本生成器，它接收一段提示（**Prompt**），然后基于概率预测下一个 Token，最终生成一段自由文本。然而，现代应用程序通常需要处理结构化数据（**如 JSON、CSV、特定格式的指令**）。直接从 LLM 获取结构化输出面临三个核心痛点，而 Output Parser 正是为根除它们而设计的。

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

**1.1 痛点一：格式的极端不确定性**

**即使我们在提示中明确要求“请用 JSON 格式输出”，模型也可能：**

- 在 JSON 前后添加“客套话”：例如，输出变成 “好的，根据您的要求，这是结果：\n{\"name\": \"张三\"}”。程序无法直接解析这个字符串。
- 遗漏关键符号：如缺少闭合的引号、花括号或逗号，导致 JSON.parse 直接抛出异常。
- 多语言符号混淆：中英文标点混用，如使用了中文分号；或中文引号“”，这些在标准 JSON 中是非法字符。

**1.2 痛点二：字段缺失或类型错误**

模型可能会“擅自发挥”，导致数据不完整或不准确：

- 遗漏必填字段：提示要求输出姓名和年龄，但模型可能只返回了姓名。
- 数据类型不匹配：要求年龄是数字，模型却输出了字符串 “二十五” 或者 “25岁”。
- 结构嵌套错误：对于复杂的嵌套对象或列表，模型可能弄错层级关系。

**1.3 痛点三：解析逻辑的脆弱性与不可维护性**

在没有专用解析器的情况下，开发者通常会写大量正则表达式或字符串分割逻辑来提取数据。这种方式极不可靠：

- 对 Prompt 修改极度敏感：一旦微调了提示词，模型输出的文本结构稍有变化，原来的正则可能完全失效。
- 难以处理复杂结构：用正则解析多层嵌套的 JSON 几乎是不可能完成的任务。
- 缺乏错误反馈机制：解析失败时，通常只能抛出异常，无法自动修复或重试。

**1.4 Output Parser 的解决之道**

LangChain 的 Output Parser 提供了一套声明式、可复用的解决方案：

- 自动生成格式指令：通过 get_format_instructions() 方法，自动生成清晰、结构化的格式说明，并注入 Prompt。这相当于给模型提供了一份“模板”，告诉它必须严格按照何种结构输出。
- 自动解析与验证：它不仅能将文本转为 Python 对象（字典、列表、Pydantic 模型），还能进行类型校验（如使用 Pydantic）。
- 自我修复与重试机制：当解析失败时，有专门的 OutputFixingParser 和 RetryWithErrorOutputParser 可以自动调用 LLM 修复错误，或者将错误信息反馈给模型重试，形成闭环。
- 无缝集成 LCEL：通过 | 管道操作符，可以像搭积木一样将解析器嵌入到链中，数据流畅流转。

## 题2：写一个 PydanticOutputParser 的例子，定义 Pydantic 类，并将其用于提取 JSON 结果。

PydanticOutputParser 是 LangChain 中最强大、最常用的解析器之一。它利用 Python 的 Pydantic 库来定义数据模型，自动生成 JSON Schema，并在解析时进行严格的类型检查和数据验证。

下面是一个详细的实战例子，演示如何从一段学术文本中提取论文信息。

```

# 导入必要的模块
from langchain_core.output_parsers import PydanticOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field
from typing import List

# 第一步：定义我们期望的数据结构
# 使用 Pydantic 的 BaseModel，可以定义字段、类型和描述
class Paper(BaseModel):
    # Field 中的 description 会帮助 LLM 理解该字段的含义
    title: str = Field(description="论文标题")
    authors: List[str] = Field(description="作者列表，即使只有一个作者也要用列表格式")
    year: int = Field(description="发表年份，必须是整数")
    keywords: List[str] = Field(description="关键词列表")

# 第二步：创建解析器实例
parser = PydanticOutputParser(pydantic_object=Paper)

# 第三步：查看并理解解析器自动生成的格式指令
# 这是 Output Parser 的精髓之一，它告诉 LLM 该怎么输出
format_instructions = parser.get_format_instructions()
print("生成的格式指令如下：\n", format_instructions)

# 输出示例：
# The output should be a JSON object with the following keys:
# - "title": (论文标题)
# - "authors": (作者列表)
# - "year": (发表年份)
# ...
# 第四步：构建 Prompt 模板
# 将格式指令嵌入到系统提示中，让 LLM 严格遵守
template = ChatPromptTemplate.from_messages([
    ("system", "你是一个专业的学术信息提取助手。请严格按照以下JSON格式输出提取结果，不要输出任何其他文字：\n{format_instructions}"),
    ("human", "请从以下文本中提取论文信息：\n{text}")
])

# 第五步：初始化模型并组装链
model = ChatOpenAI(model="gpt-4o", temperature=0)  # temperature=0 能让输出更稳定
chain = template | model | parser  # 使用 LCEL 管道符串联

# 第六步：准备数据并调用
text = "《Attention Is All You Need》由Ashish Vaswani等人于2017年发表，提出了Transformer架构，关键词包括注意力机制、机器翻译。"
result = chain.invoke({
    "text": text,
    "format_instructions": format_instructions  # 注入格式指令
})

# 第七步：查看和使用结果
print("解析后的对象类型:", type(result))  # <class '__main__.Paper'>
print("论文标题:", result.title)
print("发表年份:", result.year)
print("作者数量:", len(result.authors))

# 输出：
# 解析后的对象类型: <class '__main__.Paper'>
# 论文标题: Attention Is All You Need
# 发表年份: 2017
# 作者数量: 1
```
