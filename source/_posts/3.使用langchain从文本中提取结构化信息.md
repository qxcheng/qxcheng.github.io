---
title: 3.使用langchain从文本中提取结构化信息
date: 2025-01-13 12:46:56
tags:
- 大模型             
categories: 
- langchain
description: '在处理非结构化数据时，我们经常需要从文本、图像或其他媒体中提取结构化信息。本文将介绍如何使用 LangChain 和大语言模型来实现这一目标。'
---


在处理非结构化数据时，我们经常需要从文本、图像或其他媒体中提取结构化信息。本文将介绍如何使用 LangChain 和大语言模型来实现这一目标。

## 1.技术方案

### 1. 定义数据模型

首先，我们需要定义要提取的数据结构。使用 Pydantic 可以轻松创建类型安全的数据模型：

```python
from typing import List, Optional
from pydantic import BaseModel, Field

class Person(BaseModel):
    """Information about a person."""
    name: Optional[str] = Field(default=None, description="The name of the person")
    hair_color: Optional[str] = Field(
        default=None, description="The color of the person's hair if known"
    )
    height_in_meters: Optional[str] = Field(
        default=None, description="Height measured in meters"
    )

class Data(BaseModel):
    """Extracted data about people."""
    people: List[Person]
```

这里有几个关键设计点：

1. 所有字段都是可选的（Optional），这允许模型在无法确定某个属性时返回空值
2. 每个字段都有清晰的描述，这些描述会帮助语言模型更准确地提取信息
3. 使用嵌套模型结构支持提取多个实体

### 2. 配置提示模板

接下来，我们需要设置提示模板来指导语言模型如何提取信息：

```python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    (
        "system",
        "You are an expert extraction algorithm. "
        "Only extract relevant information from the text. "
        "If you do not know the value of an attribute asked to extract, "
        "return null for the attribute's value.",
    ),
    ("human", "{text}"),
])
```

这个提示模板包含：
- 系统指令：定义模型的角色和基本行为规则
- 占位符：用于插入需要处理的文本

### 3. 设置执行管道

最后，我们创建一个执行管道来处理文本：

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4-mini", temperature=0)
runnable = prompt | llm.with_structured_output(schema=Data)
```

## 2.实际应用示例

让我们看一个具体的例子：

```python
text = "我叫杰夫，我的头发是黑色的，身高 6 英尺。安娜的头发颜色和我一样。"
res = runnable.invoke({"text": text})
print(res)
```

输出结果：
```python
people=[
    Person(name='杰夫', hair_color='黑色', height_in_meters='1.83'),
    Person(name='安娜', hair_color='黑色', height_in_meters=None)
]
```

模型成功地：
- 识别出了两个人物
- 正确提取了各自的特征
- 自动将英尺转换为米
- 推断出安娜的头发颜色
- 对未知的信息返回了空值

## 3.优化建议

1. **添加示例**：在提示模板中加入少量示例可以显著提升提取质量
2. **调整温度**：将temperature设为0可以获得更稳定的输出
3. **完善字段描述**：为每个字段提供详细的描述，帮助模型更好理解要提取的内容
4. **考虑上下文**：可以在提示中加入文档元数据等上下文信息

## 4.总结

使用大语言模型进行结构化数据提取具有以下优势：
- 无需编写复杂的规则
- 具有强大的理解和推理能力
- 可以处理各种非标准格式
- 容易扩展和维护

通过合理的模型设计和提示工程，我们可以构建出强大而灵活的数据提取系统。

## 5.完整代码

下面是本文所有代码的完整版本：

```python
from typing import List, Optional
from pydantic import BaseModel, Field
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI


class Person(BaseModel):
    """Information about a person."""

    # ^ Doc-string for the entity Person.
    # This doc-string is sent to the LLM as the description of the schema Person,
    # and it can help to improve extraction results.

    # Note that:
    # 1. Each field is an `optional` -- this allows the model to decline to extract it!
    # 2. Each field has a `description` -- this description is used by the LLM.
    # Having a good description can help improve extraction results.
    name: Optional[str] = Field(default=None, description="The name of the person")
    hair_color: Optional[str] = Field(
        default=None, description="The color of the person's hair if known"
    )
    height_in_meters: Optional[str] = Field(
        default=None, description="Height measured in meters"
    )

class Data(BaseModel):
    """Extracted data about people."""

    # Creates a model so that we can extract multiple entities.
    people: List[Person]


# Define a custom prompt to provide instructions and any additional context.
# 1) You can add examples into the prompt template to improve extraction quality
# 2) Introduce additional parameters to take context into account (e.g., include metadata
#    about the document from which the text was extracted.)
prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            "You are an expert extraction algorithm. "
            "Only extract relevant information from the text. "
            "If you do not know the value of an attribute asked to extract, "
            "return null for the attribute's value.",
        ),
        # Please see the how-to about improving performance with
        # reference examples.
        # MessagesPlaceholder('examples'),
        ("human", "{text}"),
    ]
)

llm = ChatOpenAI(model="gpt-4-mini", temperature=0)

runnable = prompt | llm.with_structured_output(schema=Data)

text = "我叫杰夫，我的头发是黑色的，身高 6 英尺。安娜的头发颜色和我一样。"
res = runnable.invoke({"text": text})
print(res)
# people=[Person(name='杰夫', hair_color='黑色', height_in_meters='1.83'), Person(name='安娜', hair_color='黑色', height_in_meters=None)]
```