---
title: 2.使用langchain的结构化输出功能实现文本分类
date: 2025-01-09 14:15:19
tags:
- 大模型
categories:
- - langchain
description: '在自然语言处理中，文本分类是一个基础且重要的任务。本文将介绍如何使用 LangChain 框架结合大语言模型，实现智能文本分类功能，将非结构化文本自动归类到预定义的类别中。'
---

在自然语言处理中，文本分类是一个基础且重要的任务。本文将介绍如何使用 LangChain 框架结合大语言模型，实现智能文本分类功能，将非结构化文本自动归类到预定义的类别中。

## 1. 文本分类系统概述

本系统能够将输入文本自动分类到以下维度：

- 情感倾向（快乐、中性、悲伤）
- 语气强度（1-5级）
- 语言类型（中文、英语、法语、德语、意大利语）

## 2. 技术实现

### 2.1 基础环境配置

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field
```

### 2.2 定义分类模型

使用 Pydantic 定义分类标准：

```python
class Classification(BaseModel):
    sentiment: str = Field(..., enum=["happy", "neutral", "sad"])
    aggressiveness: int = Field(
        ...,
        description="describes how aggressive the statement is, the higher the number the more aggressive",
        enum=[1, 2, 3, 4, 5],
    )
    language: str = Field(
        ..., enum=["chinese", "english", "french", "german", "italian"]
    )
```

分类维度说明：

- `sentiment`：情感分类（三分类）
- `aggressiveness`：语气强度分级（五分类）
- `language`：语言类型识别（五种语言）

### 2.3 配置分类提示模板

```python
tagging_prompt = ChatPromptTemplate.from_template(
"""
Extract the desired information from the following passage.

Only extract the properties mentioned in the 'Classification' function.

Passage:
{input}
"""
)
```

### 2.4 初始化分类模型

```python
llm = ChatOpenAI(temperature=0, model="gpt-4-mini").with_structured_output(
    Classification
)

# 构建分类处理链
tagging_chain = tagging_prompt | llm
```

## 3. 分类示例

```python
# 测试文本
inp = "我很高兴认识你！我想我们会成为很好的朋友！"

# 执行分类
response = tagging_chain.invoke({"input": inp})
print(response.dict())
```

分类结果：

```python
{
    'sentiment': 'happy',
    'aggressiveness': 1,
    'language': 'chinese'
}
```

结果解读：

- 情感类别：快乐（positive）
- 语气强度：1级（最温和）
- 语言类型：中文

## 4. 系统特点

1. **结构化分类**
    - 预定义分类标准
    - 输出格式统一规范
    - 分类结果可直接用于下游任务
2. **多维度分析**
    - 同时进行情感、语气和语言分类
    - 分类标准清晰明确
    - 结果易于理解和使用
3. **灵活可扩展**
    - 易于添加新的分类维度
    - 可调整分类标准
    - 支持自定义分类规则

## 5. 应用场景

这个文本分类系统适用于多种实际应用：

- 用户评论分类
- 客服对话分析
- 社交媒体内容分类
- 多语言文本处理
- 内容情感分析
- 文本标签自动化

## 6. 实施建议

1. **分类标准定制**
    - 根据具体需求调整分类维度
    - 可以增加或减少分类级别
    - 根据场景调整语言支持范围
2. **模型优化**
    - 可以通过调整 temperature 参数控制分类的确定性
    - 根据需要选择不同的底层语言模型
    - 可以通过微调提高特定领域的分类准确性

## 7.总结

LangChain 的结构化输出功能为文本分类任务提供了一个优雅的解决方案。通过预定义的分类模型和智能语言模型的结合，我们能够快速构建出功能强大的文本分类系统。这种方案不仅实现简单，而且具有良好的可扩展性和可维护性，适合各种文本分类场景的实际应用。

## 8.完整代码

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field

class Classification(BaseModel):
    sentiment: str = Field(..., enum=["happy", "neutral", "sad"])
    aggressiveness: int = Field(
        ...,
        description="describes how aggressive the statement is, the higher the number the more aggressive",
        enum=[1, 2, 3, 4, 5],
    )
    language: str = Field(
        ..., enum=["chinese", "english", "french", "german", "italian"]
    )

tagging_prompt = ChatPromptTemplate.from_template(
"""
Extract the desired information from the following passage.

Only extract the properties mentioned in the 'Classification' function.

Passage:
{input}
"""
)

llm = ChatOpenAI(temperature=0, model="gpt-4-mini").with_structured_output(
    Classification
)

# 构建分类处理链
tagging_chain = tagging_prompt | llm

# 测试文本
inp = "我很高兴认识你！我想我们会成为很好的朋友！"

# 执行分类
response = tagging_chain.invoke({"input": inp})
print(response.dict())
```