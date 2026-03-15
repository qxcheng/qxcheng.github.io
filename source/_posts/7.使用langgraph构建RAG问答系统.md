---
title: 7.使用langgraph构建RAG问答系统
date: 2025-01-21 09:07:08
tags:
- 大模型             
categories: 
- langchain
description: '在这篇教程中，我们将学习如何使用 langgraph 构建一个智能文档检索系统。该系统能够从网页中提取信息，进行智能分段，并通过查询分析、向量检索实现精准的问答功能。'
---

在这篇教程中，我们将学习如何使用 langgraph 构建一个智能文档检索系统。该系统能够从网页中提取信息，进行智能分段，并通过查询分析、向量检索实现精准的问答功能。

## 1.安装依赖

`pip install beautifulsoup4`

## 2. 导入必要的库

```python
import bs4
from typing import Literal
from typing_extensions import List, TypedDict, Annotated
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_core.vectorstores import InMemoryVectorStore
from langchain_community.document_loaders import WebBaseLoader
from langchain_core.documents import Document
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langgraph.graph import START, StateGraph
from langchain_core.prompts import PromptTemplate
```

## 3. 网页内容加载

WebBaseLoader 是 LangChain 提供的一个强大的网页内容加载器，它的工作流程如下：

1. **URL 获取**：使用 urllib 库从指定的 URL 获取原始 HTML 内容
2. **HTML 解析**：使用 BeautifulSoup4 库解析 HTML 内容
3. **内容过滤**：通过 `bs_kwargs` 参数可以自定义解析规则
   - 在我们的例子中，使用 `SoupStrainer("li")` 只提取列表项内容
   - 这样可以有效过滤掉网页中的导航栏、页脚等无关内容

```python
loader = WebBaseLoader(
    web_paths=("https://github.com/jobbole/awesome-python-cn/blob/master/README.md",),
    bs_kwargs=dict(
        parse_only=bs4.SoupStrainer("li")
    ),
)
docs = loader.load()
```

## 4. 文档智能分割

文本分割器采用递归策略，具体步骤如下：

1. **初始分割**：首先尝试使用最高级别的分隔符（如换行符、段落符号）
2. **递归处理**：如果分割后的块仍然过大，则使用次级分隔符（如句号、分号）继续分割
3. **重叠处理**：
   - chunk_overlap=200 表示每个相邻块之间共享200个字符
   - 这种重叠设计确保了上下文的连续性，防止句子被生硬切断
   - 例如，如果一个重要概念横跨两个块，通过重叠可以在检索时完整捕获这个概念

```python
text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
all_splits = text_splitter.split_documents(docs)
```

## 5. 元数据增强

为了实现更智能的检索，我们为文档添加位置相关的元数据。这种元数据可以帮助我们在检索时进行更精确的过滤：

```python
total_documents = len(all_splits)
third = total_documents // 3
for i, document in enumerate(all_splits):
    if i < third:
        document.metadata["section"] = "beginning"
    elif i < 2 * third:
        document.metadata["section"] = "middle"
    else:
        document.metadata["section"] = "end"
```

通过添加 section 元数据，我们可以：
- 在检索时进行定向搜索
- 只搜索文档开头、中间或结尾部分的内容
- 提高检索的精确度

## 6. 定义查询模式

使用 TypedDict 定义查询的数据结构，确保查询的规范性和可维护性：

```python
class Search(TypedDict):
    """Search query."""
    query: Annotated[str, ..., "Search query to run."]
    section: Annotated[
        Literal["beginning", "middle", "end"],
        ...,
        "Section to query.",
    ]
```

## 7. 向量存储设置

InMemoryVectorStore 提供了高效的向量存储和检索功能：
   - 使用 OpenAI 的 text-embedding-3-large 模型将文本转换为高维向量
   - 每个文档块都会被转换为一个独特的向量表示

```python
embeddings = OpenAIEmbeddings(model="text-embedding-3-large")
vector_store = InMemoryVectorStore(embeddings)
_ = vector_store.add_documents(documents=all_splits)
```

## 8. 设置语言模型和提示模板

提示词模板的设计考虑了以下几个关键点：

1. **上下文注入**：
   - 将检索到的文档内容作为上下文提供给语言模型
   - 使用 {context} 和 {question} 占位符动态插入内容

2. **回答约束**：
   - 限制回答最多三个句子，保持简洁
   - 明确指示在不确定时承认不知道，避免编造答案
   - 添加固定的结束语"thanks for asking!"，保持一致的交互风格

```python
llm = ChatOpenAI(model="gpt-4o-mini")

template = """Use the following pieces of context to answer the question at the end.
If you don't know the answer, just say that you don't know, don't try to make up an answer.
Use three sentences maximum and keep the answer as concise as possible.
Always say "thanks for asking!" at the end of the answer.

{context}

Question: {question}

Helpful Answer:"""
prompt = PromptTemplate.from_template(template)
```

## 9. 构建处理流程

LangGraph 提供了一个灵活的工作流程管理系统，它允许我们将复杂的处理流程分解为多个独立的步骤，并通过状态管理来协调这些步骤之间的数据流转。

### 9.1 状态管理

首先，我们定义了一个 TypedDict 来管理整个处理流程中的状态：

```python
class State(TypedDict):
    question: str      # 用户的原始问题
    query: Search      # 结构化的查询信息
    context: List[Document]  # 检索到的相关文档
    answer: str       # 最终的回答
```

这个状态字典包含了处理流程中的所有关键数据：
- question：存储用户输入的原始问题
- query：存储经过分析后的结构化查询（使用之前定义的 Search 类型）
- context：存储从向量数据库检索到的相关文档
- answer：存储最终生成的答案

### 9.2 处理步骤

处理流程被分解为三个主要步骤，每个步骤都是一个独立的函数，接收当前状态并返回更新后的状态部分：

1. **查询分析（analyze_query）**：
```python
def analyze_query(state: State):
    # 使用 LLM 将自然语言问题转换为结构化查询
    structured_llm = llm.with_structured_output(Search)
    # 调用 LLM 进行结构化输出
    query = structured_llm.invoke(state["question"])
    # 返回更新后的状态部分
    return {"query": query}
```
这个步骤的作用是：
- 接收用户的自然语言问题
- 使用 LLM 分析问题并生成结构化查询
- 确定查询应该在文档的哪个部分（开始、中间、结尾）进行搜索

2. **文档检索（retrieve）**：
```python
def retrieve(state: State):
    query = state["query"]
    # 使用向量存储进行相似度搜索
    retrieved_docs = vector_store.similarity_search(
        query["query"],  # 使用结构化查询中的查询文本
        filter=lambda doc: doc.metadata.get("section") == query["section"],  # 使用元数据过滤
    )
    # 返回检索到的文档
    return {"context": retrieved_docs}
```
这个步骤的功能包括：
- 从状态中获取结构化查询
- 使用查询文本在向量存储中搜索相似文档
- 使用 section 元数据过滤文档
- 返回最相关的文档列表

3. **答案生成（generate）**：
```python
def generate(state: State):
    # 将检索到的文档内容合并
    docs_content = "\n\n".join(doc.page_content for doc in state["context"])
    # 使用提示模板构造输入消息
    messages = prompt.invoke({
        "question": state["question"],  # 原始问题
        "context": docs_content         # 合并的文档内容
    })
    # 使用 LLM 生成回答
    response = llm.invoke(messages)
    # 返回生成的答案
    return {"answer": response.content}
```
这个步骤的处理流程是：
- 将所有检索到的文档内容合并成一个文本
- 使用提示模板构造包含上下文和问题的输入
- 调用 LLM 生成最终答案
- 返回生成的答案文本

## 10. 组装处理图

使用 LangGraph 将各个处理步骤串联成有向无环图：
- 每个步骤的输出会自动更新状态，供下一步使用
- 支持条件分支和并行处理（本例中使用简单的线性流程）

```python
graph_builder = StateGraph(State).add_sequence([analyze_query, retrieve, generate])
graph_builder.add_edge(START, "analyze_query")
graph = graph_builder.compile()
```

## 11.使用示例

```python
for message, metadata in graph.stream(
    {"question": "请列出文章末尾部分推荐的Python库有哪些？"}, stream_mode="messages"
):
    print(message.content, end="")
# 文章末尾部分推荐的Python库包括：Pyro、PyUserInput、scapy、wifi、Pingo、keyboard、mouse、Python-Future、Six和modernize。感谢提问！
```

## 12.总结

这个项目展示了如何使用 langgraph 构建一个完整的智能文档检索系统。系统的主要特点包括：

1. 智能网页内容提取
2. 文档的智能分割和元数据增强
3. 向量化存储和相似度检索
4. 基于 LLM 的智能问答
5. 流程化的处理架构

通过这个系统，我们可以轻松地实现对大型文档的智能检索和问答功能。这种架构不仅适用于网页内容，还可以扩展到其他类型的文档处理场景。

## 13.完整代码

```python
import bs4
from typing import Literal
from typing_extensions import List, TypedDict, Annotated
from langchain_openai import ChatOpenAI
from langchain_openai import OpenAIEmbeddings
from langchain_core.vectorstores import InMemoryVectorStore
from langchain import hub
from langchain_community.document_loaders import WebBaseLoader
from langchain_core.documents import Document
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langgraph.graph import START, StateGraph
from langchain_core.prompts import PromptTemplate


# 网络内容加载 
# WebBaseLoader使用 urllib 从网络 URL 加载 HTML，并使用 BeautifulSoup 将其解析为文本。
# 我们可以通过将参数传递给 BeautifulSoup 解析器来定制 HTML 到文本的解析过程，
# 这里只解析带有“li”类的 HTML 标签
loader = WebBaseLoader(
    web_paths=("https://github.com/jobbole/awesome-python-cn/blob/master/README.md",),
    bs_kwargs=dict(
        parse_only=bs4.SoupStrainer("li")
    ),
)
docs = loader.load()
# print("docs: ", docs)

# 文档分割
# 递归字符文本分割器将使用常见分隔符（如新行）递归地分割文档，直到每个块的大小适当为止。这是通用文本用例推荐的文本分割器。
text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
all_splits = text_splitter.split_documents(docs)

# 查询分析：为文档添加元信息，以对原始用户输入进行转换或构建优化的搜索查询
total_documents = len(all_splits)
third = total_documents // 3
for i, document in enumerate(all_splits):
    if i < third:
        document.metadata["section"] = "beginning"
    elif i < 2 * third:
        document.metadata["section"] = "middle"
    else:
        document.metadata["section"] = "end"
# 为我们的搜索查询定义一个模式
class Search(TypedDict):
    """Search query."""
    query: Annotated[str, ..., "Search query to run."]
    section: Annotated[
        Literal["beginning", "middle", "end"],
        ...,
        "Section to query.",
    ]


# 文档嵌入存储
embeddings = OpenAIEmbeddings(model="text-embedding-3-large")
vector_store = InMemoryVectorStore(embeddings)
_ = vector_store.add_documents(documents=all_splits)

# 大语言模型
llm = ChatOpenAI(model="gpt-4o-mini")

# 提示词模板
template = """Use the following pieces of context to answer the question at the end.
If you don't know the answer, just say that you don't know, don't try to make up an answer.
Use three sentences maximum and keep the answer as concise as possible.
Always say "thanks for asking!" at the end of the answer.

{context}

Question: {question}

Helpful Answer:"""
prompt = PromptTemplate.from_template(template)

# 定义图的状态，包含：问题、查询模式、上下文和回答
class State(TypedDict):
    question: str
    query: Search
    context: List[Document]
    answer: str

# 查询分析步骤：提取输入问题信息到指定的查询模式Search
def analyze_query(state: State):
    structured_llm = llm.with_structured_output(Search)
    query = structured_llm.invoke(state["question"])
    return {"query": query}

# 检索步骤：使用输入问题进行相似性搜索
def retrieve(state: State):
    query = state["query"]
    retrieved_docs = vector_store.similarity_search(
        query["query"],
        filter=lambda doc: doc.metadata.get("section") == query["section"],
    )
    return {"context": retrieved_docs}

# 生成步骤：将检索到的上下文和原始问题格式化为聊天模型的提示
def generate(state: State):
    docs_content = "\n\n".join(doc.page_content for doc in state["context"])
    messages = prompt.invoke({"question": state["question"], "context": docs_content})
    response = llm.invoke(messages)
    return {"answer": response.content}

# 编译图
# 将检索和生成步骤连接成一个单一的序列
graph_builder = StateGraph(State).add_sequence([analyze_query, retrieve, generate])
graph_builder.add_edge(START, "analyze_query")
graph = graph_builder.compile()

for message, metadata in graph.stream(
    {"question": "请列出文章末尾部分推荐的Python库有哪些？"}, stream_mode="messages"
):
    print(message.content, end="")
# 文章末尾部分推荐的Python库包括：Pyro、PyUserInput、scapy、wifi、Pingo、keyboard、mouse、Python-Future、Six和modernize。感谢提问！
```