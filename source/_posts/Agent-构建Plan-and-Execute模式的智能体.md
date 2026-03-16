---
title: Agent-构建Plan-and-Execute模式的智能体
date: 2025-03-04 14:27:13
tags:
- 大模型             
categories: 
- agent
description: 这个项目实现了一个基于大型语言模型(LLM)的Plan-and-Execute模式的智能代理系统，它能够接收用户的查询，自动生成执行计划，然后逐步执行并在需要时重新规划。整个系统围绕着"规划-执行-评估"这一核心循环展开，展示了如何利用大型语言模型的推理能力来解决复杂问题。
---
这个项目实现了一个基于大型语言模型(LLM)的Plan-and-Execute模式的智能代理系统，它能够接收用户的查询，自动生成执行计划，然后逐步执行并在需要时重新规划。整个系统围绕着"规划-执行-评估"这一核心循环展开，展示了如何利用大型语言模型的推理能力来解决复杂问题。

项目由四个主要文件组成：`main.py`作为入口点初始化系统，`agent.py`定义了核心代理逻辑，`actions.py`提供了可用的工具函数，`prompts.py`包含了引导语言模型的提示模板。这种模块化的结构使得系统各部分职责明确，便于维护和扩展。

## prompts.py：定义提示词模板

`prompts.py`文件定义了三种提示模板，分别用于规划、执行和重规划阶段。这些模板的设计展示了如何通过精心构建的指令来引导大型语言模型完成特定任务。

规划提示(`planning_system_prompt`)定义了规划助手的角色、可用工具、规划指南和输出格式要求。它指导语言模型分析用户查询并生成一系列工具操作步骤，每一步只使用一个工具。这种明确的任务分解指导使模型能够将复杂问题分解为可管理的子任务。

```python
planning_system_prompt = """
% Role: 
Planning Assistant - Create a sequence of tool operations based on user query.

% Tools:
- wikipedia: Searches Wikipedia and returns a snippet (needs search query)
- calculate: Evaluates a mathematical expression and returns the result (needs operation string)

% Instructions: 
1. Analyze user query and plan necessary steps with available tools
2. Use EXACTLY ONE tool per step
3. Reference variables from memory when needed
4. Do not execute tasks, only create the plan

% Output Format:
One step per line, format: [tool_name] [variables if needed]
"""
```

执行提示(`execute_system_prompt`)针对单个任务的执行，它包含记忆部分用于插入先前任务和结果的记录，并指定了详细的JSON输出格式。这种结构化的输出格式使系统能够容易地解析语言模型的响应并执行相应的工具调用。

```python
execute_system_prompt = """
% Role: 
Execution Assistant - Execute a single tool operation.

% Tools:
- wikipedia: Searches Wikipedia and returns a snippet (needs search query)
- calculate: Evaluates a mathematical expression and returns the result (needs operation string)

% Memory:
Previous Tasks: {memory_tasks}
Previous Results: {memory_responses}

% Instructions: 
1. Select ONE appropriate tool for this specific task
2. Reference memory for context and variable values
3. Use exactly one tool per execution step

% Output Format:
JSON only:
{{
    "tool": "tool_name",
    "variables": [
        {{
            "variable_name": "name",
            "variable_value": "value"
        }}
    ]
}}
"""
```

重规划提示(`replan_system_prompt`)用于评估是否需要调整计划。它提供了四个关键记忆部分：原始计划、已执行任务、执行结果和剩余任务，引导模型分析执行情况并决定是继续原计划还是生成新计划。这种反思和调整机制使系统能够适应执行过程中的新信息和变化。

```python
replan_system_prompt = """
% Role: 
Replanning Assistant - Adjust remaining plan based on execution results.

% Tools:
- wikipedia: Searches Wikipedia and returns a snippet (needs search query)
- calculate: Evaluates a mathematical expression and returns the result (needs operation string)

% Memory:
Original Plan: {original_plan}
Executed Tasks: {memory_tasks}
Execution Results: {memory_responses}
Remaining Tasks: {remaining_tasks}

% Instructions:
1. Analyze execution results and determine if plan changes are needed
2. If no changes needed, indicate with "REPLAN: NO"
3. If changes needed, provide updated plan with "REPLAN: YES"
4. Use EXACTLY ONE tool per step in any new plan

% Output Format:
First line: "REPLAN: YES" or "REPLAN: NO"
If replanning, follow with one step per line, format: [tool_name] [variables if needed]
"""
```

这三个提示模板共同构成了一个完整的"规划-执行-评估"循环，每个模板都有明确的角色定义、上下文信息、执行指南和输出格式要求，使语言模型能够在各个阶段产生符合系统要求的输出。


## agent.py：智能代理的核心实现

`agent.py`文件定义了系统的核心类`Agent`，负责整个规划和执行流程：

```python
import json
from openai import OpenAI

from prompts import planning_system_prompt, execute_system_prompt, replan_system_prompt
from actions import available_actions


class Agent:
    def __init__(self, client: OpenAI):
        self.client = client

        ## 记忆
        self.memory_tasks = []
        self.memory_responses = []

        self.planned_actions = []

        self.planning_system_prompt = planning_system_prompt

        self.max_turns = 10
```

`Agent`类初始化时接收一个OpenAI客户端，并设置了几个关键属性：
- `memory_tasks`和`memory_responses`：跟踪已执行的任务和响应
- `planned_actions`：存储规划好的操作步骤
- `planning_system_prompt`：用于规划阶段的系统提示
- `max_turns`：最大执行回合数，防止无限循环

### LLM调用函数

```python
def openai_call(self, query: str, system_prompt: str, json_format: bool = False):
    if json_format == True:
        format_response = { "type": "json_object" }
    else:
        format_response = { "type": "text" }

    completion = self.client.chat.completions.create(
        model="gpt-4o",
        temperature=0,
        messages=[
            {"role": "system", "content": system_prompt},
            {
                "role": "user",
                "content": f'User query: {query}'
            }
        ],
        response_format=format_response
    )
    llm_response = completion.choices[0].message.content
    return llm_response
```

Agent类的`openai_call`方法封装了与OpenAI API的交互逻辑，它接受查询内容和系统提示作为输入，配置API调用参数（如模型选择、温度设置），支持指定响应格式（文本或JSON）。

### 工具调用函数

```python
def tool_call(self, task_call: str):
    try:
        task_call_json = json.loads(task_call)
    except json.JSONDecodeError as e:
        print(f"Error decoding task call: {e}")
        self.memory_responses.append(f"Error: {e}")
        return False

    tool = task_call_json.get("tool")
    variables = task_call_json.get("variables", [])
    
    # 如果工具存在于可用操作中，则执行选定的工具
    if tool in available_actions:
        # 获取工具的变量值
        variable_value = ""
        if variables and len(variables) > 0:
            variable_value = variables[0].get("variable_value", "")
        
        # 执行操作并获取结果
        try:
            result = available_actions[tool](variable_value)
            self.memory_responses.append(f"{tool}=completed. Result: {result}")
            print(f"Tool result: {result}")
            return True
        except Exception as e:
            error_message = f"Error executing {tool}: {str(e)}"
            self.memory_responses.append(error_message)
            print(error_message)
            return False
    else:
        error_message = f"Unknown tool: {tool}"
        self.memory_responses.append(error_message)
        print(error_message)
        return False
```

`tool_call`方法负责执行工具调用，它先尝试解析JSON格式的工具调用指令，验证请求的工具是否在可用工具列表中，然后提取参数并执行对应的操作。这个方法还包含了全面的错误处理机制，确保即使在工具调用失败的情况下，系统也能优雅地处理异常并记录错误信息。通过这种方式，代理能够利用外部工具（如Wikipedia查询和数学计算）来解决不同类型的问题。

### 重规划函数

```python
def replan(self, task_index: int):        
    replan_prompt = replan_system_prompt.format(
        original_plan="\n".join(self.planned_actions),
        memory_tasks=self.memory_tasks,
        memory_responses=self.memory_responses,
        remaining_tasks="\n".join(self.planned_actions[task_index:])
    )
    
    replan_result = self.openai_call("Evaluate if plan needs adjustment based on execution results", replan_prompt)
    
    # 将重新规划决策记录到记忆
    if "REPLAN: YES" in replan_result:
        print("\n\nReplanning needed. Updating plan...")
        new_plan_lines = [line for line in replan_result.split("\n") if not line.startswith("REPLAN:")]
        
        # 用新计划更新计划的操作
        self.planned_actions = self.planned_actions[:task_index] + new_plan_lines
        
        print("Updated plans:")
        for i, action in enumerate(self.planned_actions):
            print(f"  {i+1}. {action}")
        return True
    else:
        print("\n\nNo replanning needed. Continuing with original plan.")
        return False
```

`replan`方法实现了系统的动态适应能力。这个方法会调用语言模型来评估是否需要调整计划。它格式化一个包含原始计划、已执行任务、执行结果和剩余任务的重规划提示，让语言模型决定是继续原计划还是生成新计划。如果需要重规划，方法会解析新计划并更新剩余操作列表。这种自适应规划机制使系统能够应对执行过程中的不确定性和变化。

### 主执行循环

```python
def run(self, prompt: str):
    ## 规划
    planning = self.openai_call(prompt, self.planning_system_prompt)
    self.planned_actions = planning.split("\n")
    print("Planned actions:")
    for i, action in enumerate(self.planned_actions):
        print(f"  {i+1}. {action}")
            
    task_index = 0
    count = 0
    while task_index < len(self.planned_actions):
        count += 1
        if count > self.max_turns:
            print("\n\nMax turns reached. Exiting...")
            break

        task = self.planned_actions[task_index]
        print(f"\n\nExecuting task: {task}")
        execute_prompt = execute_system_prompt.format(
            memory_tasks=self.memory_tasks,
            memory_responses=self.memory_responses
        )
        task_call = self.openai_call(task, execute_prompt, True)
        self.memory_tasks.append(task)
        self.memory_responses.append(task_call)

        # 执行工具调用
        if not self.tool_call(task_call):
            self.replan(task_index)
            continue
        
        # 移动到下一个任务
        task_index += 1
        
        # 如果这是最后一个任务
        if task_index >= len(self.planned_actions):
            break
                                        
        # 执行重新规划
        self.replan(task_index)
```

`run`方法是整个代理系统的核心执行循环：
- 首先，根据用户查询生成初始计划
- 设置计数器和任务索引，防止无限循环
- 循环执行每个计划任务：
  - 为当前任务生成执行提示
  - 调用LLM确定如何执行该任务（选择工具和参数）
  - 保存任务和响应到记忆中
  - 执行工具调用，将执行结果记录到记忆中，如果失败则重规划
  - 移动到下一个任务
  - 在每个任务后考虑是否需要重规划

这个循环会一直持续到完成所有计划步骤或达到最大执行回合数。整个执行过程体现了一种"规划-行动-重规划"的模式，让代理不仅能够执行任务，还能根据反馈调整其行为。

## actions.py：工具函数

`actions.py`文件定义了代理系统可以使用的工具函数，目前包括Wikipedia查询和数学计算两个工具：

```python
import httpx


def wikipedia(q: str):
    try:
        # 先尝试获取页面摘要（更完整的信息）
        search_response = httpx.get("https://en.wikipedia.org/w/api.php", params={
            "action": "query",
            "list": "search",
            "srsearch": q,
            "format": "json"
        }).json()
        
        if not search_response.get("query", {}).get("search"):
            return f"没有找到关于'{q}'的信息"
        
        # 获取第一个结果的页面标题
        page_title = search_response["query"]["search"][0]["title"]
        
        # 用页面标题获取摘要
        extract_response = httpx.get("https://en.wikipedia.org/w/api.php", params={
            "action": "query",
            "prop": "extracts",
            "exintro": True,
            "explaintext": True,
            "titles": page_title,
            "format": "json"
        }).json()
        
        # 从响应中提取页面摘要
        pages = extract_response["query"]["pages"]
        page_id = next(iter(pages))
        extract = pages[page_id].get("extract", "")
        
        if extract:
            # 截取合理长度的摘要
            if len(extract) > 500:
                extract = extract[:500] + "..."
            return extract
        
        # 如果没有获取到摘要，回退到搜索结果片段
        snippet = search_response["query"]["search"][0]["snippet"]
        clean_snippet = snippet.replace("<span class=\"searchmatch\">", "").replace("</span>", "")
        return clean_snippet
    
    except Exception as e:
        return f"搜索维基百科时出错: {str(e)}"


def calculate(operation: str) -> float:
    return eval(operation)


available_actions = {
    "wikipedia": wikipedia,
    "calculate": calculate,
}
```

## main.py：启动入口

`main.py`是整个项目的入口点：

```python
from dotenv import load_dotenv
import os
from openai import OpenAI

from agent import Agent


load_dotenv()

openai_client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"), base_url=os.getenv("OPENAI_API_BASE")) 

plan_agent = Agent(client=openai_client)
plan_agent.run("How many times the radius of the sun is the earth?")
```

我们向代理提出了问题：太阳的半径是地球的多少倍？

代理完成了相应的任务规划，即：
1.使用wikipedia工具查询太阳的半径
2.使用wikipedia工具查询地球的半径
3.使用calculate工具计算太阳的半径除以地球的半径

在查询到结果后，代理重新规划了计算的步骤，引用了前面的查询结果。其完整输出如下：
```python
Planned actions:
  1. 1. wikipedia "Radius of Earth"
  2. 2. wikipedia "Radius of Sun"
  3. 3. calculate "Radius of Sun / Radius of Earth"

Executing task: 1. wikipedia "Radius of Earth"
Tool result: Earth radius (denoted as R🜨 or RE) is the distance from the center of Earth to a point on or near its surface. Approximating the figure of Earth by an Earth spheroid (an oblate ellipsoid), the radius ranges from a maximum (equatorial radius, denoted a) of nearly 6,378 km (3,963 mi) to a minimum (polar radius, denoted b) of nearly 6,357 km (3,950 mi).
A globally-average value is usually considered to be 6,371 kilometres (3,959 mi) with a 0.3% variability (±10 km) for the following reasons.
The I...

No replanning needed. Continuing with original plan.

Executing task: 2. wikipedia "Radius of Sun"
Tool result: Solar radius is a unit of distance used to express the size of stars in astronomy relative to the Sun. The solar radius is usually defined as the radius to the layer in the Sun's photosphere where the optical depth equals 2/3:
        1
          R
            ⊙
        =
        6.957
        ×
          10
            8
             m
    ...

Replanning needed. Updating plan...
Updated plans:
  1. 1. wikipedia "Radius of Earth"
  2. 2. wikipedia "Radius of Sun"
  3. calculate "6.957 × 10^8 m / 6,371,000 m"

Executing task: calculate "6.957 × 10^8 m / 6,371,000 m"
Tool result: 109.1979281117564
```

## 结语

这个项目展示了如何构建一个具有规划能力的AI代理，它能够分解复杂任务、执行适当的操作，并根据执行结果调整计划。这种基于规划的方法代表了AI系统向更高自主性发展的重要趋势，使它们能够解决更复杂的问题并适应不断变化的环境。


