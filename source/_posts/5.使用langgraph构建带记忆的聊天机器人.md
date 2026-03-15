---
title: 5.使用langgraph构建带记忆的聊天机器人
date: 2025-01-15 12:28:55
tags:
- 大模型             
categories: 
- langchain
description: '本文将介绍基于langchain和langgraph来搭建带记忆的聊天机器人。LangGraph是Langchain团队开发的一个Python库，专门用于创建可以记住状态的、复杂的AI工作流和多智能体系统。'
---

本文将介绍基于langchain和langgraph来搭建带记忆的聊天机器人。LangGraph是Langchain团队开发的一个Python库，专门用于创建可以记住状态的、复杂的AI工作流和多智能体系统。

它的核心目标是解决传统AI编排中的关键痛点：
- 无法处理复杂的决策逻辑
- 难以实现智能体之间的交互
- 缺乏上下文记忆和状态管理

LangGraph通过有向图(Directed Graph)的方式，解决了这些问题。

## 1.安装库

```python
pip install langgraph
```

## 2.状态定义

```python
class State(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]  # 消息历史序列
    language: str  # 对话语言设置
```

State类用于langgraph图中的状态维护，其使用TypedDict定义了对话状态：
- messages：存储对话历史，使用Annotated标注支持消息添加功能
- language：指定回答语言，支持多语言切换。

## 3.核心组件初始化

```python
class AIChat():
    def __init__(self):
        # 初始化语言模型
        self.model = ChatOpenAI(model="gpt-4o-mini")  # 使用GPT-4系列模型
        
        # 设置提示词模板
        self.prompt = ChatPromptTemplate.from_messages([
            (
                "system",  # 系统角色提示
                "You are a helpful assistant. Answer all questions to the best of your ability in {language}.",
            ),
            MessagesPlaceholder(variable_name="messages"),  # 用户消息占位符
        ])
        
        # 初始化记忆系统
        self.memory = MemorySaver()  # 用于保存对话历史
        
        # 配置消息修剪器
        self.trimmer = trim_messages(
            max_tokens=65,  # 最大token数限制
            strategy="last",  # 保留最后的消息
            token_counter=self.model,  # 使用模型的token计数器
            include_system=True,  # 包含系统消息
            allow_partial=False,  # 不允许部分消息
            start_on="human",  # 从人类消息开始
        )
```

初始化方法设置了四个关键组件：
1. 语言模型：选择GPT-4系列模型作为对话核心
2. 提示词模板：定义了AI助手的角色和行为
3. 记忆系统：管理对话历史记录
4. 消息修剪器：防止上下文过长，保持对话效率

## 4.模型调用流程

```python
def call_model(self, state: State):
    # 组合提示词模板和模型
    chain = self.prompt | self.model
    
    # 修剪消息历史
    trimmed_messages = self.trimmer.invoke(state["messages"])
    
    # 调用模型获取响应
    response = chain.invoke(
        {"messages": trimmed_messages, "language": state["language"]}
    )
    
    # 返回模型响应
    return {"messages": [response]}
```

这个方法实现了完整的模型调用流程：
1. 创建处理链：将提示词模板与模型连接
2. 消息修剪：确保不超过token限制
3. 模型调用：传入处理后的消息和语言设置
4. 响应处理：将模型响应封装返回

## 5.对话图构建

```python
def create_graph(self):
    # 创建状态图
    workflow = StateGraph(state_schema=State)
    
    # 添加起始点到模型的边
    workflow.add_edge(START, "model")
    
    # 添加模型节点
    workflow.add_node("model", self.call_model)
    
    # 编译图并添加记忆功能
    app = workflow.compile(checkpointer=self.memory)
    return app
```

这个方法构建了对话系统的工作流：
1. 创建图：使用State类作为状态模式
2. 定义流程：从开始到模型的处理流
3. 添加功能：将call_model方法作为处理节点
4. 完成构建：编译图并集成记忆系统

## 6.交互界面

```python
def chat(self):
    # 配置对话设置
    config = {"configurable": {"thread_id": "abc123"}}  # 对话线程ID
    language = "Chinese"  # 设置回答语言
    
    # 创建对话图实例
    app = self.create_graph()
    
    # 开始交互循环
    while True:
        # 获取用户输入
        query = input("请输入问题:")
        input_messages = [HumanMessage(query)]
        
        # 流式处理响应
        for chunk, metadata in app.stream(
            {"messages": input_messages, "language": language},
            config,
            stream_mode="messages",
        ):
            # 只输出AI响应部分
            if isinstance(chunk, AIMessage):
                print(chunk.content, end="")
        print('\n')
```

交互界面实现了以下功能：
1. 配置初始化：设置对话ID和语言
2. 图实例化：创建对话处理图
3. 交互循环：持续接收用户输入
4. 流式输出：实时显示AI响应

## 7.聊天示例

```text
请输入问题：我是谁。
你是一个提问者，想要了解更多信息。如果你愿意，可以告诉我更多关于你的事情，我会尽力帮助你！

请输入问题：我是qqq
你好，QQQ！很高兴认识你。有什么我可以帮助你的吗？

请输入问题：我是谁
你是QQQ！如果你有其他问题或想聊些什么，随时告诉我！

请输入问题：我是谁
你是提问的那个人，具体是谁我并不知道。如果你愿意，可以告诉我更多关于你的信息！
```

刚开始大模型不知道我是谁，告诉它后由于添加了记忆它能回答出来，然后由于我们使用修剪器去除了前面的历史消息，它又不能回答出来。结果符合预期。

## 8.完整代码

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import BaseMessage
from langchain_core.messages import HumanMessage, SystemMessage, trim_messages, AIMessage
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import START, MessagesState, StateGraph
from langgraph.graph.message import add_messages
from typing import Sequence
from typing_extensions import Annotated, TypedDict


# 定义图中的状态
class State(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]
    language: str


class AIChat():
    def __init__(self):
        # 定义模型
        self.model = ChatOpenAI(model="gpt-4o-mini")

        # 定义提示词模板
        self.prompt = ChatPromptTemplate.from_messages(
            [
                (
                    "system",
                    "You are a helpful assistant. Answer all questions to the best of your ability in {language}.",
                ),
                MessagesPlaceholder(variable_name="messages"),
            ]
        )

        # 定义记忆
        self.memory = MemorySaver()

        # 定义修剪器
        self.trimmer = trim_messages(
            max_tokens=65,
            strategy="last",
            token_counter=self.model,
            include_system=True,
            allow_partial=False,
            start_on="human",
        )

    # 定义调用模型的函数
    def call_model(self, state: State):
        chain = self.prompt | self.model
        trimmed_messages = self.trimmer.invoke(state["messages"])
        response = chain.invoke(
            {"messages": trimmed_messages, "language": state["language"]}
        )
        return {"messages": [response]}

    # 创建图
    def create_graph(self):
        # 定义图
        workflow = StateGraph(state_schema=State)
        # 定义节点和边
        workflow.add_edge(START, "model")
        workflow.add_node("model", self.call_model)
        # 添加记忆
        app = workflow.compile(checkpointer=self.memory)
        return app

    # 聊天
    def chat(self):
        # 更改配置后，会从头开始对话，即失去之前的记忆。
        config = {"configurable": {"thread_id": "abc123"}}
        language = "Chinese"

        app = self.create_graph()

        while True:
            query = input("请输入问题:")
            input_messages = [HumanMessage(query)]
            for chunk, metadata in app.stream(
                {"messages": input_messages, "language": language},
                config,
                stream_mode="messages",
            ):
                if isinstance(chunk, AIMessage):  # Filter to just model responses
                    print(chunk.content, end="")
            print('\n')
        

if __name__ == "__main__":
    AIChat().chat()
```