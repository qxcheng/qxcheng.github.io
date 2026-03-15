---
title: 1.langchain的概念和基本使用
date: 2025-01-09 14:09:29
tags:
- 大模型
categories:
- - langchain
description: 'LangChain 是一个革命性的开源框架，为构建基于大语言模型（LLMs）的智能应用提供了强大支持。作为连接 LLMs 与实际应用场景的桥梁，它极大简化了复杂应用的开发流程，使得开发者能够快速构建从简单的聊天机器人到复杂的知识问答系统等各类智能应用。'
---

**LangChain** 是一个革命性的开源框架，为构建基于大语言模型（LLMs）的智能应用提供了强大支持。作为连接 LLMs 与实际应用场景的桥梁，它极大简化了复杂应用的开发流程，使得开发者能够快速构建从简单的聊天机器人到复杂的知识问答系统等各类智能应用。

## 1. 核心优势与特性

1. **智能链式处理**
    - LangChain 基于"链"的创新理念，实现了模型调用的优雅串联
    - 支持灵活的多级处理流程，如信息提取、分析和生成的无缝衔接
    - 通过模块化设计，确保处理流程的可维护性和扩展性
2. **无缝数据集成能力**
    - 提供丰富的数据源连接器，支持实时对接数据库、文档和各类 API
    - 实现数据的动态获取和更新，确保 LLM 始终基于最新信息进行决策
    - 支持结构化和非结构化数据的统一处理
3. **智能记忆管理**
    - 先进的上下文管理机制，实现对话的连贯性和上下文理解
    - 灵活的记忆策略，支持短期和长期记忆的动态调节
    - 智能的状态追踪，确保多轮对话的准确性
4. **强大的工具生态**
    - 内置丰富的工具集成接口，支持计算器、搜索引擎等外部功能
    - 插件化架构设计，便于开发者扩展自定义功能
    - 支持工具的动态调用和组合使用
5. **广泛的模型兼容性**
    - 全面支持主流大语言模型，包括 OpenAI GPT 系列、Anthropic Claude
    - 支持国内领先的模型，如百度千帆等
    - 灵活的适配机制，便于接入新型语言模型

## 2. 应用场景

1. **智能对话系统**
    - 构建具有深度理解能力的对话助手
    - 支持多轮对话推理和上下文管理
    - 实现个性化的用户交互体验
2. **知识问答系统**
    - 打造基于专业知识库的智能问答平台
    - 支持多源数据的融合查询
    - 实现精准的答案定位和解析
3. **智能工作流自动化**
    - 自动化文档处理和数据分析
    - 智能报告生成和内容总结
    - 多语言翻译和本地化处理
4. **创意内容生成**
    - 智能文案创作和内容优化
    - 代码生成和技术文档编写
    - 创意设计方案生成

## 3. 核心架构组件

1. **PromptTemplates（提示模板）**
    - 专业的提示工程设计工具
    - 支持动态参数注入和模板复用
    - 优化模型输入以提升响应质量
2. **Chains（处理链）**
    - 灵活的任务流程编排工具
    - 支持条件分支和并行处理
    - 确保处理流程的可控性和可追踪性
3. **Memory（记忆系统）**
    - 智能的上下文状态管理
    - 支持多种记忆策略的动态切换
    - 确保对话的连贯性和准确性
4. **Agents（智能代理）**
    - 自主决策的智能处理单元
    - 动态规划任务执行路径
    - 优化资源调用效率
5. **Data Loaders（数据加载器）**
    - 统一的数据接入标准
    - 支持多种格式数据的智能解析
    - 确保数据处理的可靠性和效率

## 4.基本问答

首先安装langchain包：

```python
pip install langchain langchain-openai langchain-core
```

我们使用openai的大模型，需要设置环境变量`OPENAI_API_KEY`，如果你使用的是代理地址，需要使用环境变量`OPENAI_API_BASE`指定：

```python
import os

os.environ["OPENAI_API_BASE"] = "https://xxx.com"
os.environ["OPENAI_API_KEY"] = "sk-xxx"
```

完成一次基本问答：

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, SystemMessage

# 创建大模型
model = ChatOpenAI(model="gpt-4o-mini")

# 向模型发送的系统消息和用户消息
messages = [
    SystemMessage(content="Translate the following from English into Chinese"),
    HumanMessage(content="hello woeld"),
]

# 向模型发送消息
res = model.invoke(messages)

# 打印结果
print(res.content)
```

## 5.提示词模板和链

使用提示词模板来注入动态参数，并与模型用管道操作符连接成链。

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

system_template = "Translate the following from English into {language}"

# 构建提示模板
prompt_template = ChatPromptTemplate.from_messages(
    [("system", system_template), ("user", "{text}")]
)

model = ChatOpenAI(model="gpt-4o-mini")

# 将提示模板与模型连接，形成一个处理链（chain）
chain = prompt_template | model

# 执行链的调用
response = chain.invoke({"language": "Italian", "text": "hi!"})
print(response.content) 
```