---
title: 10.使用LangGraph进行长文本摘要总结
date: 2025-01-21 10:37:16
tags:
- 大模型             
categories: 
- langchain
description: '在本文中，我们将探索如何使用 LangChain 和 LangGraph 构建一个强大的文档摘要系统。这个系统能够处理长文本，通过分块、并行处理和递归合并的方式，最终生成一个连贯的总结。'
---

在本文中，我们将探索如何使用 LangChain 和 LangGraph 构建一个强大的文档摘要系统。这个系统能够处理长文本，通过分块、并行处理和递归合并的方式，最终生成一个连贯的总结。

## 0.短文本总结

当token足够容纳文档时，可不进行文档分割，直接生成摘要。

```python
from langchain_community.document_loaders import WebBaseLoader
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain.chains.combine_documents import create_stuff_documents_chain

# 加载文档
loader = WebBaseLoader("https://github.com/jobbole/awesome-python-cn/blob/master/README.md")
docs = loader.load()

# 大模型
llm = ChatOpenAI(model="gpt-4o-mini")

# 定义prompt
prompt = ChatPromptTemplate.from_messages(
    [("system", "Write a concise summary of the following:\\n\\n{context}")]
)

# 创建链
chain = create_stuff_documents_chain(llm, prompt)

# 直接为docs生成总结
result = chain.invoke({"context": docs})
print(result)
# 流式输出
# for token in chain.stream({"context": docs}):
#     print(token, end="|")

# The README.md file for the "awesome-python-cn" repository, maintained by the "开源前哨" and "Python开发者" WeChat public account teams, provides a comprehensive list of Python resources in Chinese. It includes various tools and libraries categorized under themes such as environment management, package management, web frameworks, data visualization, machine learning, and more. Each category lists specific libraries and tools along with brief descriptions, covering a wide range of functionalities from handling HTTP requests to performing scientific calculations and creating graphical user interfaces. The project aims to facilitate the development of Python applications by providing access to valuable resources and community contributions.
```

## 1. 环境准备和依赖导入

当token不足够容纳文档时，我们需要进行文档分割和合并摘要。首先，让我们导入所需的库和组件：

```python
from langchain_community.document_loaders import WebBaseLoader
from langchain_openai import ChatOpenAI
from langchain.chains.llm import LLMChain
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_text_splitters import CharacterTextSplitter
import asyncio
import operator
from typing import Annotated, List, Literal, TypedDict
from langchain.chains.combine_documents.reduce import (
    acollapse_docs,
    split_list_of_docs,
)
from langchain_core.documents import Document
from langgraph.constants import Send
from langgraph.graph import END, START, StateGraph
```

## 2. 文档加载和预处理

接下来，我们实现文档的加载和初始化处理：

```python
# 加载文档
loader = WebBaseLoader("https://github.com/jobbole/awesome-python-cn/blob/master/README.md")
docs = loader.load()

# 使用tiktoken编码器进行文档分割
text_splitter = CharacterTextSplitter.from_tiktoken_encoder(
    chunk_size=1000, chunk_overlap=0
)
split_docs = text_splitter.split_documents(docs)

# 初始化大语言模型
llm = ChatOpenAI(model="gpt-4o-mini")
```

这段代码实现了文档的加载和预处理过程。chunk_overlap=0 的设置意味着我们的文本块之间没有重叠。这个选择是为了提高处理效率，避免重复处理相同的内容。在某些特殊场景下，如果需要保持上下文的连续性，我们可以设置适当的重叠值。

## 3. 构建摘要链

现在，我们来构建用于生成摘要的处理链：

```python
# 单文档摘要链
map_prompt = ChatPromptTemplate.from_messages(
    [("system", "Write a concise summary of the following:\\n\\n{context}")]
)
map_chain = map_prompt | llm | StrOutputParser()

# 摘要合并链
reduce_template = """
The following is a set of summaries:
{docs}
Take these and distill it into a final, consolidated summary
of the main themes.
"""
reduce_prompt = ChatPromptTemplate([("human", reduce_template)])
reduce_chain = reduce_prompt | llm | StrOutputParser()
```

在这段代码中，我们构建了两个关键的处理链：map_chain 和 reduce_chain。这种设计采用了 Map-Reduce 模式，这是一种强大的分布式处理范式，特别适合处理大规模数据。
- map_chain 负责处理单个文档块的摘要生成。
- reduce_chain 的职责是将多个摘要合并成一个连贯的整体。


## 4. 状态管理和工具函数

接下来，我们定义系统的状态管理和辅助函数：

```python
# 设置最大token限制
token_max = 1000

def length_function(documents: List[Document]) -> int:
    """计算输入文档列表的总token数"""
    return sum(llm.get_num_tokens(doc.page_content) for doc in documents)

# 定义状态类型
class OverallState(TypedDict):
    contents: List[str]
    summaries: Annotated[list, operator.add]
    collapsed_summaries: List[Document]
    final_summary: str

class SummaryState(TypedDict):
    content: str
```

OverallState 类定义了系统的整体状态，包含四个关键字段：
- contents：存储原始文档内容
- summaries：存储生成的摘要，使用 operator.add 注解支持摘要的合并操作
- collapsed_summaries：存储合并后的摘要文档
- final_summary：存储最终生成的总结


## 5. 核心处理函数

每个处理函数都有其特定的职责和优化策略：

```python
async def generate_summary(state: SummaryState):
    """生成单个文档块的摘要"""
    response = await map_chain.ainvoke(state["content"])
    return {"summaries": [response]}

def map_summaries(state: OverallState):
    """将文档内容映射到摘要生成节点"""
    return [
        Send("generate_summary", {"content": content}) 
        for content in state["contents"]
    ]

def collect_summaries(state: OverallState):
    """收集生成的摘要"""
    return {
        "collapsed_summaries": [Document(summary) for summary in state["summaries"]]
    }

async def collapse_summaries(state: OverallState):
    """合并摘要"""
    doc_lists = split_list_of_docs(
        state["collapsed_summaries"], length_function, token_max
    )
    results = []
    for doc_list in doc_lists:
        results.append(await acollapse_docs(doc_list, reduce_chain.ainvoke))
    return {"collapsed_summaries": results}

def should_collapse(state: OverallState) -> Literal["collapse_summaries", "generate_final_summary"]:
    """判断是否需要继续合并摘要"""
    num_tokens = length_function(state["collapsed_summaries"])
    return "collapse_summaries" if num_tokens > token_max else "generate_final_summary"

async def generate_final_summary(state: OverallState):
    """生成最终摘要"""
    response = await reduce_chain.ainvoke(state["collapsed_summaries"])
    return {"final_summary": response}
```

这段代码定义了系统的核心处理函数，每个函数都有其特定的职责和优化策略。generate_summary 负责生成单个文档块的摘要，map_summaries 将文档内容映射到摘要生成节点，collect_summaries 收集生成的摘要，collapse_summaries 合并摘要，should_collapse 判断是否需要继续合并摘要，generate_final_summary 生成最终摘要。

## 6. 构建处理图

![](https://img.bplan.top/20250117174635083.png)

```python
# 创建状态图
graph = StateGraph(OverallState)

# 添加节点
graph.add_node("generate_summary", generate_summary)  
graph.add_node("collect_summaries", collect_summaries)
graph.add_node("collapse_summaries", collapse_summaries)
graph.add_node("generate_final_summary", generate_final_summary)

# 添加边和条件
graph.add_conditional_edges(START, map_summaries, ["generate_summary"])
graph.add_edge("generate_summary", "collect_summaries")
graph.add_conditional_edges("collect_summaries", should_collapse)
graph.add_conditional_edges("collapse_summaries", should_collapse)
graph.add_edge("generate_final_summary", END)

# 编译图
app = graph.compile()
```

在这段代码中，我们使用 LangGraph 构建了一个基于 OverallState 的状态图来管理整个摘要生成过程。

在节点添加阶段，我们定义了四个关键节点：
1. generate_summary 节点接收文档内容作为输入，输出对应的摘要。
2. collect_summaries 节点将多个并行生成的摘要整合在一起。
3. collapse_summaries 节点实现了摘要的合并逻辑。当摘要总长度超过阈值时，这个节点会将多个摘要合并。
4. generate_final_summary 节点将所有处理后的摘要整合成一个连贯的最终摘要。

边的配置是图结构中最复杂的部分。我们使用了两种类型的边：
- 普通边（add_edge）：用于表示确定性的处理流程，如从 generate_summary 到 collect_summaries 的转换。
- 条件边（add_conditional_edges）：用于实现动态处理流程。系统中有两个关键的条件边：
  1. 从 collect_summaries 到后续节点的转换：这个条件边基于 should_collapse 函数的判断结果来决定下一步操作。如果当前收集的摘要的总token数超过了 token_max（1000），系统会选择 collapse_summaries 路径进行合并处理；否则，系统会选择 generate_final_summary 路径直接生成最终摘要。
  2. 从 collapse_summaries 到后续节点的转换：这个条件边同样使用 should_collapse 函数进行判断。在合并后，系统会再次检查token数量，决定是继续合并还是生成最终摘要。这种循环检查确保了最终的摘要始终保持在可处理的大小范围内。

特别值得注意的是起始点（START）和终止点（END）的处理：
- START 节点通过 map_summaries 函数连接到 generate_summary 节点，这个过程实现了初始文档内容到摘要生成任务的映射。
- END 节点标志着处理流程的完成，它只能从 generate_final_summary 节点到达，确保了处理流程的完整性。


## 7. 运行系统

系统运行时的关键特性：

```python
async def main():
    async for step in app.astream(
        {"contents": [doc.page_content for doc in split_docs]},
        {"recursion_limit": 10},
    ):
        print(list(step.keys()))
    print(step)

asyncio.run(main())
```

这段代码运行了系统，展示了系统的关键特性。系统采用了异步流处理，支持实时状态监控和递归控制。系统的性能优化考虑了并行处理策略、内存管理优化和状态追踪机制。

## 8.输出

```sh
Created a chunk of size 1254, which is longer than the specified 1000
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['generate_summary']
['collect_summaries']
['collapse_summaries']
['collapse_summaries']
['collapse_summaries']
['collapse_summaries']
['collapse_summaries']
['collapse_summaries']
['generate_final_summary']
{'generate_final_summary': {'final_summary': '### Consolidated Summary of Python Resources and Libraries\n\nThe Python ecosystem offers a diverse range of resources, libraries, and frameworks that cater to various programming needs, highlighting its versatility and extensive functionality:\n\n1. **Web Development**: Frameworks like Django and Flask, along with libraries such as BeautifulSoup, support web applications and web scraping.\n\n2. **Data Handling and Processing**: Libraries like pandas, NumPy, and Dask facilitate efficient data manipulation and processing, while tools like PySpark and Ray enhance data analysis capabilities. \n\n3. **Scientific Computing and Machine Learning**: Essential libraries for data analysis include SciPy and visualization tools such as matplotlib and Plotly, complemented by machine learning frameworks like TensorFlow and PyTorch.\n\n4. **Asynchronous Programming and Task Automation**: Libraries such as asyncio, Celery, and APScheduler improve application responsiveness and workflow management.\n\n5. **Development Tools and Quality Assurance**: Tools like Jupyter Notebook, Flake8, and pytest focus on improving code quality, testing, and maintenance.\n\n6. **Environment and Package Management**: Tools like pip and conda streamline the management of packages and development environments.\n\n7. **Security and Data Integrity**: Libraries like cryptography ensure secure data handling and management.\n\n8. **Cloud Integration and DevOps**: Tools such as boto3 and Ansible facilitate integration with cloud services and infrastructure management.\n\n9. **Gaming and GUI Development**: Frameworks like Pygame and libraries like Tkinter offer options for game and graphical user interface development.\n\n10. **Networking, API Development, and Specialized Applications**: Libraries like Mininet and scapy support networking tasks, while tools like Graphene assist with API development, catering to specialized areas such as robotics and finance.\n\n11. **Documentation and Community Engagement**: Tools like Sphinx and curated tutorials foster community contributions and project documentation.\n\nThis summary underscores the comprehensive and open-source nature of the Python toolkit, emphasizing its broad applicability across various domains and industries.'}}
```

## 9.结语

这个文档摘要系统展示了如何将现代AI技术与优秀的软件工程实践相结合。通过合理的架构设计，系统实现了高效的并行处理、智能的资源分配和可靠的错误处理。系统不仅解决了长文档摘要的具体需求，还为构建类似的AI处理系统提供了实用的参考范例。

## 10.完整代码

```python
from langchain_community.document_loaders import WebBaseLoader
from langchain_openai import ChatOpenAI
from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain.chains.llm import LLMChain
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_text_splitters import CharacterTextSplitter
import asyncio
import operator
from typing import Annotated, List, Literal, TypedDict

from langchain.chains.combine_documents.reduce import (
    acollapse_docs,
    split_list_of_docs,
)
from langchain_core.documents import Document
from langgraph.constants import Send
from langgraph.graph import END, START, StateGraph

# 加载文档
loader = WebBaseLoader("https://github.com/jobbole/awesome-python-cn/blob/master/README.md")
docs = loader.load()

# 分割文档
text_splitter = CharacterTextSplitter.from_tiktoken_encoder(
    chunk_size=1000, chunk_overlap=0
)
split_docs = text_splitter.split_documents(docs)

# 大模型
llm = ChatOpenAI(model="gpt-4o-mini")


#2 长文本总结
# 生成一段摘要的链
map_prompt = ChatPromptTemplate.from_messages(
    [("system", "Write a concise summary of the following:\\n\\n{context}")]
)

map_chain = map_prompt | llm | StrOutputParser()


# 将多个摘要合并的链
reduce_template = """
The following is a set of summaries:
{docs}
Take these and distill it into a final, consolidated summary
of the main themes.
"""

reduce_prompt = ChatPromptTemplate([("human", reduce_template)])

reduce_chain = reduce_prompt | llm | StrOutputParser()


# 设置最大token
token_max = 1000

# 计算输入文档列表的总token
def length_function(documents: List[Document]) -> int:
    """Get number of tokens for input contents."""
    return sum(llm.get_num_tokens(doc.page_content) for doc in documents)


# 图的总体状态：包含输入文档内容、文档相应的摘要、合并的摘要、最终摘要。
class OverallState(TypedDict):
    # 使用operator.add将我们生成的所有摘要从各个节点合并回一个列表
    contents: List[str]
    summaries: Annotated[list, operator.add]
    collapsed_summaries: List[Document]
    final_summary: str

# 节点状态：生成摘要节点的状态
class SummaryState(TypedDict):
    content: str


# 生成摘要
async def generate_summary(state: SummaryState):
    response = await map_chain.ainvoke(state["content"])
    return {"summaries": [response]}

# 将输入文档的内容映射到生成摘要的节点
def map_summaries(state: OverallState):
    # 每个 Send 对象由图中节点的名称组成以及要发送到该节点的状态
    return [
        Send("generate_summary", {"content": content}) for content in state["contents"]
    ]

# 收集摘要
def collect_summaries(state: OverallState):
    return {
        "collapsed_summaries": [Document(summary) for summary in state["summaries"]]
    }

# 合并摘要
async def collapse_summaries(state: OverallState):
    doc_lists = split_list_of_docs(
        state["collapsed_summaries"], length_function, token_max
    )
    results = []
    for doc_list in doc_lists:
        results.append(await acollapse_docs(doc_list, reduce_chain.ainvoke))

    return {"collapsed_summaries": results}

# 判断是否需要合并摘要
def should_collapse(
    state: OverallState,
) -> Literal["collapse_summaries", "generate_final_summary"]:
    num_tokens = length_function(state["collapsed_summaries"])
    if num_tokens > token_max:
        return "collapse_summaries"
    else:
        return "generate_final_summary"

# 生成最终摘要
async def generate_final_summary(state: OverallState):
    response = await reduce_chain.ainvoke(state["collapsed_summaries"])
    return {"final_summary": response}


# 构建图
# 节点:
graph = StateGraph(OverallState)
graph.add_node("generate_summary", generate_summary)  
graph.add_node("collect_summaries", collect_summaries)
graph.add_node("collapse_summaries", collapse_summaries)
graph.add_node("generate_final_summary", generate_final_summary)

# 边:
# 当状态图从 START 节点开始时，它会调用 map_summaries 函数。
# 如果 map_summaries 返回有效的 Send 对象，状态图将继续向 generate_summary 节点转移，
# 执行摘要生成的操作。
graph.add_conditional_edges(START, map_summaries, ["generate_summary"])
graph.add_edge("generate_summary", "collect_summaries")
graph.add_conditional_edges("collect_summaries", should_collapse)
graph.add_conditional_edges("collapse_summaries", should_collapse)
graph.add_edge("generate_final_summary", END)

app = graph.compile()
# from IPython.display import Image
# Image(app.get_graph().draw_mermaid_png())

async def main():
    async for step in app.astream(
        {"contents": [doc.page_content for doc in split_docs]},
        {"recursion_limit": 10},
    ):
        print(list(step.keys()))

    print(step)

asyncio.run(main())
```