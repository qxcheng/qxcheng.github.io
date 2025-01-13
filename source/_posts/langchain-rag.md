---
title: 4.使用langchain构建简单的RAG系统
date: 2025-01-13 12:50:20
tags:
- 大模型
categories:
- - langchain
description: '在这篇教程中，我们将学习如何使用 LangChain 构建一个基础的检索增强生成（RAG）系统。我们将逐步介绍从文档加载到问答的完整流程。'
---

在这篇教程中，我们将学习如何使用 LangChain 构建一个基础的检索增强生成（RAG）系统。我们将逐步介绍从文档加载到问答的完整流程。

## 0.安装依赖

`pip install langchain-chroma langchain-community langchain-text-splitters pypdf`

## 1. 环境准备和依赖导入

首先，我们需要导入必要的依赖：

```python
from langchain_core.documents import Document
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_core.runnables import RunnableLambda, RunnablePassthrough
from langchain_core.prompts import ChatPromptTemplate 
from langchain_community.document_loaders import PyPDFLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
```

这些导入包含了文档处理、向量存储、语言模型和提示模板等核心组件。

## 2. 文档加载

我们提供了两种文档加载方式：直接创建文本文档和加载PDF文件。

### 2.1 文本文档加载

```python
def get_text_docs():
    docs = [
        Document(
            page_content="我的名字叫张三",
            metadata={"source": "name-doc"},
        ),
        Document(
            page_content="我的同学名字叫李四",
            metadata={"source": "name-doc"},
        ),
        Document(
            page_content="我喜欢打乒乓球",
            metadata={"source": "hobby-doc"},
        ),
        Document(
            page_content="李四喜欢打蓝球",
            metadata={"source": "hobby-doc"},
        ),
    ]
    return docs
```

这个函数创建了包含个人信息和爱好的简单文档集合。每个文档都包含内容和元数据。

### 2.2 PDF文档加载

```python
def get_pdf_docs(file_path = "test.pdf"):
    loader = PyPDFLoader(file_path)
    docs = loader.load()
    print(len(docs))
    print(f"{docs[0].page_content[:200]}\n")
    print(docs[0].metadata)
    return docs
```

这个函数可以加载PDF文件，并输出一些基本信息用于验证。

## 3. 文档处理和向量化

接下来，我们将文档分割成更小的片段，并将其转换为向量表示：

```python
# 文档加载
docs = get_text_docs()

# 分割
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000, chunk_overlap=200, add_start_index=True
)
all_splits = text_splitter.split_documents(docs)

# 向量嵌入和存储
vectorstore = Chroma.from_documents(
    all_splits,
    embedding=OpenAIEmbeddings(model="text-embedding-3-large"),
)
```

这一步骤包括：
1. 加载文档
2. 使用递归字符分割器将文档分成小块
3. 使用OpenAI的embedding模型将文本转换为向量
4. 将向量存储在Chroma向量数据库中

## 4. 构建RAG管道

最后，我们构建完整的RAG查询管道：

```python
# 检索器
retriever = RunnableLambda(vectorstore.similarity_search).bind(k=1) 

# 大模型
llm = ChatOpenAI(model="gpt-4o-mini")

# 提示模板
message = """
Answer this question using the provided context only.
{question}
Context:
{context}
"""
prompt = ChatPromptTemplate.from_messages([("human", message)])

# 模型链
rag_chain = {"context": retriever, "question": RunnablePassthrough()} | prompt | llm
```

这个管道包含以下组件：
- 检索器：用于从向量存储中检索相关文档
- 语言模型：使用OpenAI的GPT模型
- 提示模板：定义如何构建查询
- RAG链：将所有组件组合成一个完整的处理流程

在这个管道中，`RunnablePassthrough()` 的作用是：
1. 直接传递输入：它会将输入值原封不动地传递给下一个组件，不做任何修改
2. 在这里具体作用是：
   - 当我们调用 `rag_chain.invoke("我是谁？")` 时
   - `{"context": retriever}` 会通过检索器获取相关文档
   - `{"question": RunnablePassthrough()}` 会将原始问题 "我是谁？" 直接传递下去
   - 这样 prompt 模板就能同时获得检索到的上下文（context）和原始问题（question）

## 5. 使用示例

让我们测试这个RAG系统：

```python
response = rag_chain.invoke("我是谁？")
print(response.content)
response = rag_chain.invoke("李四喜欢干什么？")
print(response.content)
```

输出结果：
```
你叫张三。
李四喜欢打篮球。
```

系统成功地从文档中检索到相关信息并给出了准确的回答。

## 6.总结

通过这个教程，我们学习了如何使用LangChain构建一个基础的RAG系统。这个系统可以：
1. 加载和处理不同格式的文档
2. 将文档转换为向量表示
3. 存储和检索相关信息
4. 使用大语言模型生成答案

这为构建更复杂的文档问答系统奠定了基础。你可以通过调整各个组件的参数，或添加更多功能来增强系统的性能。

## 7.完整代码

```python
from langchain_core.documents import Document
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_core.runnables import RunnableLambda, RunnablePassthrough
from langchain_core.prompts import ChatPromptTemplate 
from langchain_community.document_loaders import PyPDFLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter

def get_text_docs():
    # 文档加载
    docs = [
        Document(
            page_content="我的名字叫张三",
            metadata={"source": "name-doc"},
        ),
        Document(
            page_content="我的同学名字叫李四",
            metadata={"source": "name-doc"},
        ),
        Document(
            page_content="我喜欢打乒乓球",
            metadata={"source": "hobby-doc"},
        ),
        Document(
            page_content="李四喜欢打蓝球",
            metadata={"source": "hobby-doc"},
        ),
    ]
    return docs

def get_pdf_docs(file_path = "test.pdf"):
    # 加载PDF
    loader = PyPDFLoader(file_path)
    docs = loader.load()
    print(len(docs))
    print(f"{docs[0].page_content[:200]}\n")  # 页面字符串内容
    print(docs[0].metadata)  # 包含文件名和页码的元数据
    return docs


# 文档加载
docs = get_text_docs()

# 分割
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000, chunk_overlap=200, add_start_index=True
)
all_splits = text_splitter.split_documents(docs)

# 向量嵌入和存储
vectorstore = Chroma.from_documents(
    all_splits,
    embedding=OpenAIEmbeddings(model="text-embedding-3-large"),
)

# 检索器
retriever = RunnableLambda(vectorstore.similarity_search).bind(k=1) 

# 大模型
llm = ChatOpenAI(model="gpt-4o-mini")

# 提示模板
message = """
Answer this question using the provided context only.
{question}
Context:
{context}
"""
prompt = ChatPromptTemplate.from_messages([("human", message)])

# 模型链
rag_chain = {"context": retriever, "question": RunnablePassthrough()} | prompt | llm

# 基于文档问答
response = rag_chain.invoke("我是谁？")
print(response.content)
response = rag_chain.invoke("李四喜欢干什么？")
print(response.content)
# 你叫张三。
# 李四喜欢打篮球。
```

