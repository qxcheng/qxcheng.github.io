---
title: 9. 使用LangGraph构建基于 SQL 数据的问答系统
date: 2025-01-21 09:11:41
tags:
- 大模型             
categories: 
- langchain
description: '在这篇博客中，我们将深入探讨如何使用 LangChain 和 LangGraph 构建一个智能的 SQL 查询助手。这个助手能够将自然语言问题转换为 SQL 查询，执行查询，并提供人性化的答案。更重要的是，它支持人机协同工作流程，让用户可以在关键步骤进行干预。'
---

在这篇博客中，我们将深入探讨如何使用 LangChain 和 LangGraph 构建一个智能的 SQL 查询助手。这个助手能够将自然语言问题转换为 SQL 查询，执行查询，并提供人性化的答案。更重要的是，它支持人机协同工作流程，让用户可以在关键步骤进行干预。

## 0.初始化sql数据

我们新建文件命名为 `create_and_insert.sql`

```sql
CREATE TABLE users (
    姓名 TEXT NOT NULL,
    年龄 INTEGER NOT NULL,
    爱好 TEXT,
    职业 TEXT
);

INSERT INTO users (姓名, 年龄, 爱好, 职业) VALUES
('张三', 25, '篮球、游泳', '软件工程师'),
('李四', 30, '阅读、旅行', '教师'),
('王五', 22, '音乐、游戏', '学生'),
('赵六', 28, '绘画、摄影', '设计师'),
('孙七', 35, '足球、跑步', '医生'),
('周八', 24, '舞蹈、瑜伽', '健身教练'),
('吴九', 29, '电影、美食', '厨师'),
('郑十', 27, '写作、书法', '作家'),
('刘一', 32, '爬山、钓鱼', '销售经理'),
('陈二', 26, '唱歌、看剧', '行政助理');
```

然后在本地安装sqlite3后执行：

```sh
sqlite3 test.db 
.read create_and_insert.sql
```

## 1. 基础设置和依赖导入

```python
from langchain_community.utilities import SQLDatabase
from langchain_openai import ChatOpenAI
from langchain import hub
from langchain_community.tools.sql_database.tool import QuerySQLDataBaseTool
from langgraph.graph import START, StateGraph
from langgraph.checkpoint.memory import MemorySaver
from typing_extensions import TypedDict, Annotated

# 初始化数据库连接
db = SQLDatabase.from_uri("sqlite:///test.db")
```

这段代码导入了必要的依赖并建立了数据库连接。SQLDatabase 是 LangChain 提供的一个实用工具，它能够将数据库操作无缝集成到 LLM 工作流中。

## 2. 定义应用程序状态

```python
class State(TypedDict):
    question: str    # 用户的自然语言问题
    query: str       # 生成的 SQL 查询
    result: str      # SQL 查询结果
    answer: str      # 最终的自然语言答案
```

使用 TypedDict 定义应用程序状态，这不仅提供了类型提示，还明确了工作流中数据的流动方式。

## 3. 初始化语言模型

```python
llm = ChatOpenAI(model="gpt-4-mini")
query_prompt_template = hub.pull("langchain-ai/sql-query-system-prompt")
```

我们使用 OpenAI 的 GPT-4-mini 模型，并从 LangChain Hub 获取预定义的 SQL 查询提示模板。这个模板经过优化，能够生成高质量的 SQL 查询。

## 4. 核心功能实现

### 4.1 SQL 查询生成

```python
class QueryOutput(TypedDict):
    """Generated SQL query."""
    query: Annotated[str, ..., "Syntactically valid SQL query."]

def write_query(state: State):
    """将自然语言问题转换为 SQL 查询"""
    prompt = query_prompt_template.invoke({
        "dialect": db.dialect,
        "top_k": 10,
        "table_info": db.get_table_info(),
        "input": state["question"],
    })
    structured_llm = llm.with_structured_output(QueryOutput)
    result = structured_llm.invoke(prompt)
    return {"query": result["query"]}
```

这个函数负责将用户的自然语言问题转换为有效的 SQL 查询。它使用了：
- 结构化输出确保生成语法正确的 SQL
- 数据库模式信息以生成准确的查询
- 预定义的提示模板优化查询生成

### 4.2 查询执行

```python
def execute_query(state: State):
    """执行 SQL 查询"""
    execute_query_tool = QuerySQLDataBaseTool(db=db)
    return {"result": execute_query_tool.invoke(state["query"])}
```

这个函数安全地执行生成的 SQL 查询，使用 LangChain 的 QuerySQLDataBaseTool 确保查询执行的安全性和可靠性。

### 4.3 答案生成

```python
def generate_answer(state: State):
    """使用查询结果生成自然语言答案"""
    prompt = (
        "Given the following user question, corresponding SQL query, "
        "and SQL result, answer the user question.\n\n"
        f'Question: {state["question"]}\n'
        f'SQL Query: {state["query"]}\n'
        f'SQL Result: {state["result"]}'
    )
    response = llm.invoke(prompt)
    return {"answer": response.content}
```

这个函数将技术性的 SQL 结果转换为用户友好的自然语言答案。

## 5. 工作流编排与人工干预机制

![](https://img.bplan.top/20250116074911661.png)

### 5.1 基础工作流设置

```python
memory = MemorySaver()
graph_builder = StateGraph(State).add_sequence([
    write_query,
    execute_query,
    generate_answer
])
graph_builder.add_edge(START, "write_query")
```

### 5.2 人工干预点配置

```python
graph = graph_builder.compile(
    checkpointer=memory,
    interrupt_before=["execute_query"]  # 关键配置：在执行查询前中断
)
```

人工干预机制的实现主要依赖于以下几个关键点：

1. **中断点设置**：
   - 使用 `interrupt_before=["execute_query"]` 参数指定在执行查询前进行中断
   - 这确保了在执行可能影响数据库的操作前，用户有机会审查和确认

2. **状态管理**：
   - `MemorySaver` 用于保存工作流的状态
   - 这使得工作流可以在中断后继续执行，不会丢失之前的处理结果

3. **流式执行**：

```python
config = {"configurable": {"thread_id": "1"}}

# 第一阶段：执行到中断点
for step in graph.stream(
    {"question": "请问有多少用户？"},
    config,
    stream_mode="updates",
):
    print(step)  # 显示每个步骤的执行结果
```

4. **用户交互**：

```python
try:
    user_approval = input("Do you want to go to execute query? (yes/no): ")
except Exception:
    user_approval = "no"  # 异常情况下默认拒绝执行
```

5. **条件继续执行**：

```python
if user_approval.lower() == "yes":
    # 继续执行剩余的工作流
    for step in graph.stream(None, config, stream_mode="updates"):
        print(step)
else:
    print("Operation cancelled by user.")
```

### 5.3 工作流执行过程

完整的执行流程如下：

1. 首先执行 `write_query` 步骤，生成 SQL 查询
2. 到达 `execute_query` 前自动中断
3. 显示生成的查询并等待用户确认
4. 根据用户的选择：
   - 如果确认，继续执行查询和生成答案
   - 如果拒绝，终止操作

### 5.4 示例输出

```
{'write_query': {'query': 'SELECT COUNT(*) as 用户数量 FROM users;'}}
{'__interrupt__': ()}
Do you want to go to execute query? (yes/no): yes
{'execute_query': {'result': '[(10,)]'}}
{'generate_answer': {'answer': '根据SQL查询的结果，用户数量为10。'}}
```

这种人工干预机制的优势在于：

- **安全性**：防止未经审查的查询直接执行
- **可控性**：用户可以在关键节点进行干预
- **透明性**：清晰展示每个步骤的执行结果
- **灵活性**：可以根据需要在不同节点添加中断点


## 6.总结

这个项目展示了如何将多个强大的工具（LangChain、LangGraph、GPT-4）组合起来，构建一个智能且安全的 SQL 查询助手。通过分层设计和人机协同，我们既保证了系统的自动化程度，又确保了操作的安全性。

关键特点：
- 自然语言理解和 SQL 生成
- 类型安全和错误处理
- 人机协同的工作流程
- 可扩展的模块化设计

这个方案可以作为构建其他 AI 驱动的数据库工具的参考架构。

## 7.完整代码

```python
from langchain_community.utilities import SQLDatabase
from langchain_openai import ChatOpenAI
from langchain import hub
from langchain_community.tools.sql_database.tool import QuerySQLDataBaseTool
from langgraph.graph import START, StateGraph
from langgraph.checkpoint.memory import MemorySaver
from typing_extensions import TypedDict, Annotated


db = SQLDatabase.from_uri("sqlite:///test.db")


# 应用程序状态
class State(TypedDict):
    question: str
    query: str
    result: str
    answer: str

llm = ChatOpenAI(model="gpt-4o-mini")

query_prompt_template = hub.pull("langchain-ai/sql-query-system-prompt")


# 将问题转换为 SQL 查询
class QueryOutput(TypedDict):
    """Generated SQL query."""

    query: Annotated[str, ..., "Syntactically valid SQL query."]

def write_query(state: State):
    """Generate SQL query to fetch information."""
    prompt = query_prompt_template.invoke(
        {
            "dialect": db.dialect,
            "top_k": 10,
            "table_info": db.get_table_info(),
            "input": state["question"],
        }
    )
    structured_llm = llm.with_structured_output(QueryOutput)
    result = structured_llm.invoke(prompt)
    return {"query": result["query"]}


# 执行查询
def execute_query(state: State):
    """Execute SQL query."""
    execute_query_tool = QuerySQLDataBaseTool(db=db)
    return {"result": execute_query_tool.invoke(state["query"])}


# 生成答案
def generate_answer(state: State):
    """Answer question using retrieved information as context."""
    prompt = (
        "Given the following user question, corresponding SQL query, "
        "and SQL result, answer the user question.\n\n"
        f'Question: {state["question"]}\n'
        f'SQL Query: {state["query"]}\n'
        f'SQL Result: {state["result"]}'
    )
    response = llm.invoke(prompt)
    return {"answer": response.content}


# 编排
memory = MemorySaver()
graph_builder = StateGraph(State).add_sequence(
    [write_query, execute_query, generate_answer]
)
graph_builder.add_edge(START, "write_query")
# 人机协同
graph = graph_builder.compile(checkpointer=memory, interrupt_before=["execute_query"])

config = {"configurable": {"thread_id": "1"}}

for step in graph.stream(
    {"question": "请问有多少用户？"},
    config,
    stream_mode="updates",
):
    print(step)

try:
    user_approval = input("Do you want to go to execute query? (yes/no): ")
except Exception:
    user_approval = "no"

if user_approval.lower() == "yes":
    # If approved, continue the graph execution
    for step in graph.stream(None, config, stream_mode="updates"):
        print(step)
else:
    print("Operation cancelled by user.")
'''
{'write_query': {'query': 'SELECT COUNT(*) as 用户数量 FROM users;'}}
{'__interrupt__': ()}
Do you want to go to execute query? (yes/no): yes
{'execute_query': {'result': '[(10,)]'}}
{'generate_answer': {'answer': '根据SQL查询的结果，用户数量为10。'}}
'''
```