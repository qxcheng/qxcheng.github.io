---
title: 8.使用langgraph实现带条件分支的RAG问答系统
date: 2025-01-21 09:09:56
tags:
- 大模型             
categories: 
- langchain
description: '在这篇教程中，我们将深入探讨如何使用 langgraph 框架构建一个带有条件分支的智能问答系统。我们将通过逐步分析代码，了解每个部分的功能，并解释其背后的原理。最终的目标是使您能够创建一个能够从网页文档中提取信息并选择是否使用提取信息来回答用户问题的 AI 助手。'
---

在这篇教程中，我们将深入探讨如何使用 langgraph 框架构建一个带有条件分支的智能问答系统。我们将通过逐步分析代码，了解每个部分的功能，并解释其背后的原理。最终的目标是使您能够创建一个能够从网页文档中提取信息并选择是否使用提取信息来回答用户问题的 AI 助手。

## 1. 引入所需的库

首先，我们需要导入一些必要的库和模块。这些库将帮助我们处理网页数据、进行文本分割、嵌入和检索等操作。

```python
import bs4
from langchain import hub
from langchain_community.document_loaders import WebBaseLoader
from langchain_core.documents import Document
from langchain_text_splitters import RecursiveCharacterTextSplitter
from typing_extensions import List, TypedDict
from langchain_openai import ChatOpenAI
from langchain_openai import OpenAIEmbeddings
from langchain_core.vectorstores import InMemoryVectorStore
from langgraph.graph import MessagesState, StateGraph
from langchain_core.tools import tool
from langchain_core.messages import SystemMessage
from langgraph.graph import END
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.checkpoint.memory import MemorySaver
```


## 2. 初始化语言模型

接下来，我们将初始化一个聊天模型，这里我们使用的是 OpenAI 的 GPT-4o 模型。`temperature=0`：设置温度为 0，意味着模型将生成更确定的答案，而不是随机的。

```python
# 大模型
llm = ChatOpenAI(model="gpt-4o", temperature=0)
```


## 3. 加载网页文档

我们使用 `WebBaseLoader` 从指定的 URL 加载文档，并仅解析列表项（`li` 标签）。

```python
# 文档加载
loader = WebBaseLoader(
    web_paths=("https://github.com/jobbole/awesome-python-cn/blob/master/README.md",),
    bs_kwargs=dict(
        parse_only=bs4.SoupStrainer("li")
    ),
)
docs = loader.load()
```


## 4. 文档分割

加载文档后，我们需要将其分割成更小的部分，以便进行处理和嵌入。

```python
# 文档分割
text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
all_splits = text_splitter.split_documents(docs)
```


## 5. 文档嵌入存储

接下来，我们将为分割后的文档生成嵌入，并将其存储在内存向量存储中。

```python
# 文档嵌入存储
embeddings = OpenAIEmbeddings(model="text-embedding-3-large")
vector_store = InMemoryVectorStore(embeddings)
_ = vector_store.add_documents(documents=all_splits)
```


## 6. 创建检索工具

我们将创建一个工具，用于根据用户的查询检索相似文档。

```python
@tool(response_format="content_and_artifact")
def retrieve(query: str):
    """
    将检索步骤转化为工具
    """
    retrieved_docs = vector_store.similarity_search(query, k=2)
    serialized = "\n\n".join(
        (f"Source: {doc.metadata}\n" f"Content: {doc.page_content}")
        for doc in retrieved_docs
    )
    return serialized, retrieved_docs

tools = ToolNode([retrieve])
```

- `retrieve` 函数：接收查询并返回与之相似的文档。
- `similarity_search`：在向量存储中查找与查询最相似的文档。


## 7. 处理用户查询

我们定义一个函数来处理用户的查询并生成响应。

```python
# 生成可能包含工具调用的 AIMessage
def query_or_respond(state: MessagesState):
    llm_with_tools = llm.bind_tools([retrieve])
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}
```

- `query_or_respond`：语言模型通过绑定检索工具，从而可以根据需要决定是否使用检索工具，response可能包含两种类型的内容：普通的文本响应或工具调用请求，这是后续条件分支判断走哪条路的依据。


## 8. 生成最终响应

接下来，我们将生成最终的答案。

```python
# 使用检索到的内容生成回复
def generate(state: MessagesState):
    """生成答案"""
    recent_tool_messages = []
    for message in reversed(state["messages"]):
        if message.type == "tool":
            recent_tool_messages.append(message)
        else:
            break
    tool_messages = recent_tool_messages[::-1]

    docs_content = "\n\n".join(doc.content for doc in tool_messages)
    system_message_content = (
        "You are an assistant for question-answering tasks. "
        "Use the following pieces of retrieved context to answer "
        "the question. If you don't know the answer, say that you "
        "don't know. Use three sentences maximum and keep the "
        "answer concise."
        "\n\n"
        f"{docs_content}"
    )
    conversation_messages = [
        message
        for message in state["messages"]
        if message.type in ("human", "system")
        or (message.type == "ai" and not message.tool_calls)
    ]
    prompt = [SystemMessage(system_message_content)] + conversation_messages

    response = llm.invoke(prompt)
    return {"messages": [response]}
```

- `generate` 函数：该函数的主要任务是从检索到的文档中生成答案。它首先获取最近的工具消息，然后将这些消息格式化为适合传递给语言模型的提示。
  - **获取工具消息**：通过逆序遍历 `state["messages"]`，我们可以找到最近的工具消息，这些消息包含了与用户查询相关的上下文信息。然后将获取到的工具消息反转，以确保它们按照正确的顺序进行处理。
  - **连接文档内容**：通过 `docs_content`，将所有工具消息的内容连接起来，形成一个完整的上下文，以便语言模型使用。
  - **构建系统消息**：`system_message_content` 指示模型其角色和回答问题的方式，强调回答的简洁性和准确性，并加入检索到的文档内容。
  - **提取对话消息**：`conversation_messages` 从消息状态中提取人类和系统消息，以便在生成最终响应时使用。
  - **生成提示**：最终的提示 `prompt` 结合了系统消息和对话消息，作为输入传递给语言模型。
  - **生成响应**：调用语言模型生成的响应，最终返回给调用者。


## 9. 构建状态图

我们使用 `StateGraph` 来管理整个流程的状态。

![](https://img.bplan.top/20250110091632612.png)

```python
# 图定义
graph_builder = StateGraph(MessagesState)
graph_builder.add_node(query_or_respond)
graph_builder.add_node(tools)
graph_builder.add_node(generate)

graph_builder.set_entry_point("query_or_respond")
graph_builder.add_conditional_edges(
    "query_or_respond",
    tools_condition,
    {END: END, "tools": "tools"},
)
graph_builder.add_edge("tools", "generate")
graph_builder.add_edge("generate", END)

memory = MemorySaver()
graph = graph_builder.compile(checkpointer=memory)
```

- `StateGraph`：这是一个用于定义和管理状态流转的结构。它允许我们将不同的处理步骤组织成一个图形结构，以便在处理用户输入时能够灵活地进行状态转移。
- **添加节点**：通过 `add_node` 方法，我们可以将不同的处理步骤（如 `query_or_respond`、`tools` 和 `generate`）添加到状态图中。这些节点代表了系统在处理用户请求时的不同阶段。
- **设置入口点**：`set_entry_point` 方法用于指定状态图的起始节点。在这里，我们将入口点设置为 `query_or_respond`，这意味着系统将从处理用户查询开始。
- **添加条件边**：`add_conditional_edges` 方法用于定义在特定条件下的状态转移。在本例中，我们根据是否调用工具的结果来决定是否继续到 `tools` 节点或结束流程。
- **添加边**：通过 `add_edge` 方法，我们可以定义节点之间的连接关系，确保在处理流程中能够顺利地从一个步骤转移到下一个步骤。
- **内存保存**：`MemorySaver` 用于在状态图中保存当前的状态，以便在需要时能够恢复。这对于长时间运行的对话或多轮交互非常重要。
- **编译图形**：最后，通过 `compile` 方法，我们将状态图编译为可执行的形式，以便在后续的用户交互中使用。

## 10. 流式处理用户输入

最后，我们可以流式处理用户输入并生成响应。

```python
# 对于不需要额外检索步骤的消息，它会做出适当的回应
input_message = "Hello"
for step in graph.stream(
    {"messages": [{"role": "user", "content": input_message}]},
    stream_mode="values",
    config=config,
):
    step["messages"][-1].pretty_print()

# 在执行搜索时，我们可以流式传输步骤以观察查询生成、检索和答案生成：
input_message = "请问 maya 是什么工具？"
for step in graph.stream(
    {"messages": [{"role": "user", "content": input_message}]},
    stream_mode="values",
    config=config,
):
    step["messages"][-1].pretty_print()
```

响应内容：

```
================================ Human Message =================================

Hello
================================== Ai Message ==================================

Hi there! How can I assist you today?
================================ Human Message =================================

请问 maya 是什么工具？
================================== Ai Message ==================================
Tool Calls:
  retrieve (chatcmpl-Jcknfda608GbybspWw9fe51T0VPGl)
 Call ID: chatcmpl-Jcknfda608GbybspWw9fe51T0VPGl
  Args:
    query: Maya 是什么工具
================================= Tool Message =================================
Name: retrieve

Source: {'source': 'https://github.com/jobbole/awesome-python-cn/blob/master/README.md'}
Content: Python 本身。pyarmor：一个用于加密 python 脚本的工具，也可以将加密后的脚本绑定到固件上，或设置已加密脚本的有效 期。shiv：一个命令行工具，可用于构建完全独立的 zip 应用（PEP 441 所描述的那种），同时包含了所有的依赖项。buildout：一个 构建系统，从多个组件来创建，组装和部署应用。BitBake：针对嵌入式 Linux 的类似 make 的构建工具。fabricate：对任何语言自动 找到依赖关系的构建工具。PlatformIO：多平台命令行构建工具。PyBuilder：纯 Python 实现的持续化构建工具。SCons：软件构建工具。IPython：功能丰富的工具，非常有效的使用交互式 Python。bpython：界面丰富的 Python 解析器。ptpython：高级交互式 Python  解析器， 构建于 python-prompt-toolkit 之上。Jupyter Notebook (IPython)：一个能够让你最大限度地以交互式方式使用 Python 的丰富工具包。

Source: {'source': 'https://github.com/jobbole/awesome-python-cn/blob/master/README.md'}
Content: Club：用于图形结构化数据的无监督机器学习工具箱。NIPY：神经影响学工具箱集合。ObsPy：地震学 Python 工具箱。QuTiP ：Python 版 Quantum 工具箱。SimPy：一个基于过程的离散事件模拟框架。matplotlib：一个 Python 2D 绘图库。bokeh：用 Python  进行交互式 web 绘图。ggplot：ggplot2 给 R 提供的 API 的 Python 版本。plotly：协同 Python 和 matplotlib 工作的 web 绘图库。pyecharts：基于百度 Echarts 的数据可视化库。pygal：一个 Python SVG 图表创建工具。pygraphviz：Graphviz 的 Python 接口。PyQtGraph：交互式实时 2D/3D/ 图像绘制及科学/工程学组件。SnakeViz：一个基于浏览器的 Python's cProfile 模块输出结果查看工 具。vincent：把 Python 转换为 Vega 语法的转换工具。VisPy：基于 OpenGL 的高性能科学可视化工具。Altair：用于 Python 的声明式统计可视化库。bqplot：Jupyter Notebook 的交互式绘图库。Cartopy：具有 matplotlib 支持的 Python 制图库。Dash：构建在 Flask、React 和 Plotly 之上，旨在用于分析 Web 应用程序。
================================== Ai Message ==================================

Maya 是一个 Python 库，用于简化日期和时间处理，提供了简单的接口来解析、格式化和处理日期时间数据。如果您对 Python 中的日 期时间操作感兴趣，Maya 可以提供很大的帮助。
```

## 11.总结

在本教程中，我们详细介绍了如何使用 langgraph 构建一个智能问答系统。我们从加载网页文档开始，经过文本分割、嵌入存储、工具创建、用户查询处理，最终生成响应。通过这种方式，您可以创建一个能够从网络文档中提取信息并选择是否使用提取信息来回答用户问题的 AI 助手。

## 12.完整代码

```python
import bs4
from langchain import hub
from langchain_community.document_loaders import WebBaseLoader
from langchain_core.documents import Document
from langchain_text_splitters import RecursiveCharacterTextSplitter
from typing_extensions import List, TypedDict
from langchain_openai import ChatOpenAI
from langchain_openai import OpenAIEmbeddings
from langchain_core.vectorstores import InMemoryVectorStore
from langgraph.graph import MessagesState, StateGraph
from langchain_core.tools import tool
from langchain_core.messages import SystemMessage
from langgraph.graph import END
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.checkpoint.memory import MemorySaver

# 大模型
llm = ChatOpenAI(model="gpt-4o", temperature=0)

# 网络文档加载
loader = WebBaseLoader(
    web_paths=("https://github.com/jobbole/awesome-python-cn/blob/master/README.md",),
    bs_kwargs=dict(
        parse_only=bs4.SoupStrainer("li")
    ),
)
docs = loader.load()

# 文档分割
text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
all_splits = text_splitter.split_documents(docs)

# 文档嵌入存储
embeddings = OpenAIEmbeddings(model="text-embedding-3-large")
vector_store = InMemoryVectorStore(embeddings)
_ = vector_store.add_documents(documents=all_splits)


# 将检索步骤转化为工具
@tool(response_format="content_and_artifact")
def retrieve(query: str):
    """
    将检索步骤转化为工具
    """
    retrieved_docs = vector_store.similarity_search(query, k=2)
    serialized = "\n\n".join(
        (f"Source: {doc.metadata}\n" f"Content: {doc.page_content}")
        for doc in retrieved_docs
    )
    return serialized, retrieved_docs

tools = ToolNode([retrieve])


# 生成可能包含工具调用的 AIMessage
def query_or_respond(state: MessagesState):
    llm_with_tools = llm.bind_tools([retrieve])
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}


# 使用检索到的内容生成回复
def generate(state: MessagesState):
    """生成答案"""
    recent_tool_messages = []
    for message in reversed(state["messages"]):
        if message.type == "tool":
            recent_tool_messages.append(message)
        else:
            break
    tool_messages = recent_tool_messages[::-1]

    docs_content = "\n\n".join(doc.content for doc in tool_messages)
    system_message_content = (
        "You are an assistant for question-answering tasks. "
        "Use the following pieces of retrieved context to answer "
        "the question. If you don't know the answer, say that you "
        "don't know. Use three sentences maximum and keep the "
        "answer concise."
        "\n\n"
        f"{docs_content}"
    )
    conversation_messages = [
        message
        for message in state["messages"]
        if message.type in ("human", "system")
        or (message.type == "ai" and not message.tool_calls)
    ]
    prompt = [SystemMessage(system_message_content)] + conversation_messages

    response = llm.invoke(prompt)
    return {"messages": [response]}


# 图定义
graph_builder = StateGraph(MessagesState)
graph_builder.add_node(query_or_respond)
graph_builder.add_node(tools)
graph_builder.add_node(generate)

graph_builder.set_entry_point("query_or_respond")
graph_builder.add_conditional_edges(
    "query_or_respond",
    tools_condition,
    {END: END, "tools": "tools"},
)
graph_builder.add_edge("tools", "generate")
graph_builder.add_edge("generate", END)

memory = MemorySaver()
graph = graph_builder.compile(checkpointer=memory)
config = {"configurable": {"thread_id": "abc123"}}

# 对于不需要额外检索步骤的消息，它会做出适当的回应
input_message = "Hello"
for step in graph.stream(
    {"messages": [{"role": "user", "content": input_message}]},
    stream_mode="values",
    config=config,
):
    step["messages"][-1].pretty_print()

# 在执行搜索时，我们可以流式传输步骤以观察查询生成、检索和答案生成：
input_message = "请问 maya 是什么工具？"
for step in graph.stream(
    {"messages": [{"role": "user", "content": input_message}]},
    stream_mode="values",
    config=config,
):
    step["messages"][-1].pretty_print()
```