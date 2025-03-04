---
title: 1.从零开始使用Python构建ReAct模式的AI智能体
date: 2025-03-04 14:22:20
tags:
- 大模型             
categories: 
- agent
description: 为了深入了解大模型智能体的运行机制，这篇博客和大家分享如何从零开始搭建一个基于ReAct模式的AI智能体。
---
为了深入了解大模型智能体的运行机制，这篇博客和大家分享如何从零开始搭建一个基于ReAct模式的AI智能体。

## 一、ReAct模式原理解析

ReAct（Reasoning + Acting）是一种创新的AI交互模式，它将语言模型的推理能力（Reasoning）与实际行动执行（Acting）相结合。这种模式让AI系统能够：
- 通过推理理解问题的本质
- 规划解决问题的步骤
- 调用合适的工具执行操作
- 根据执行结果调整后续行动

在代码层面表现为，在一个循环中，大模型对用户提出的问题进行推理，然后要么给出最终答案，要么给出一个工具调用的指令。通过解析这个指令，我们实际地去调用这个工具，将工具的结果扔给大模型，然后大模型根据这个结果再次对问题进行推理，以此类推，直到获取最终答案为止。

![](https://img.bplan.top/20250225191445806.png)

## 二、项目整体架构

### 2.1 目录结构

```python
myagent/
├── main.py        # 主程序入口
├── agent.py       # 智能体核心逻辑
├── actions.py     # 工具函数实现
├── helpers.py     # 辅助工具函数
├── prompts.py     # 系统提示词模板
└── .env           # 配置文件
```

### 2.2 提示词模板 (prompts.py)
```python
system_prompt = """
You run in a loop of Thought, Action, PAUSE, Action_Response.
At the end of the loop you output an Answer.

Use Thought to understand the question you have been asked.
Use Action to run one of the actions available to you - then return PAUSE.
Action_Response will be the result of running those actions.

Your available actions are:

calculate:
e.g. calculate: 4 * 7 / 3
Runs a calculation and returns the number - uses Python so be sure to use floating point syntax if necessary

wikipedia:
e.g. wikipedia: Django
Returns a summary from searching Wikipedia

Example session:

Question: What is the capital of France?
Thought: I should look up France on Wikipedia
Action: 
{
  "function_name": "wikipedia",
  "function_parms": {
    "q": "France"
  }
}
PAUSE 

You will be called again with this:

Action_Response: France is a country. The capital is Paris.

You then output:

Answer: The capital of France is Paris.

Example session:

Question: What is the result of 2 times 2?
Thought: I need to multiply 2 by 2
Action: 
{
  "function_name": "calculate",
  "function_parms": {
    "operation": "2 * 2"
  }
}
PAUSE

You will be called again with this: 

Action_Response: 4

You then output:

Answer: The result of 2 times 2 is 4.
"""
```

提示词的开头定义了一个清晰的思维链路：思考(Thought) -> 行动(Action) -> 暂停(PAUSE) -> 响应(Action_Response)，然后定义了每个步骤的职责。

接着工具定义部分，告诉了模型当前可用的工具，并给出了工具的使用示例和职责。

最后给出了完整的示例会话，展示了如何将自然语言查询转化为结构化的工具调用，然后再将工具调用的结果传递给模型。这个转化过程正是ReAct模式的精髓，将推理(Reasoning)和行动(Acting)有机结合。

### 2.3 智能体实现 (agent.py)

```python
from openai import OpenAI

from prompts import system_prompt
from actions import available_actions
from helpers import extract_json

class Agent:
    def __init__(self, client: OpenAI):
        self.client = client
        self.messages: list = []
        self.messages.append({"role": "system", "content": system_prompt})
        self.max_turns = 5

    def execute(self):
        completion = self.client.chat.completions.create(
            model="gpt-4o",
            messages=self.messages
        )
        return completion.choices[0].message.content

    def run(self, message: str = ""):
        turn_count = 1
        self.messages.append({"role": "user", "content": message})

        while turn_count < self.max_turns:
            print(f"------Loop: {turn_count}")
            turn_count += 1

            response = self.execute()
            print(response)
            self.messages.append({"role": "assistant", "content": response})
            
            if 'Action' in response:
                # 解析出Action
                json_function = extract_json(response)
                if not json_function:
                    raise Exception(f"Invalid JSON in response: {response}")
                
                function_name = json_function[0]['function_name']
                function_parms = json_function[0]['function_parms']
                if function_name not in available_actions:
                    raise Exception(f"Unknown action: {function_name}: {function_parms}")
                                
                # 调用Action
                print(f"***Running action: {function_name} {function_parms}")
                action_function = available_actions[function_name]
                result = action_function(**function_parms)
                function_result_message = f"Action_Response: {result}"
                self.messages.append({"role": "user", "content": function_result_message})
                print(f"***Running result: {function_result_message}")
                continue

            if 'Answer' in response:
                break
```

在run方法中，为了防止无限循环，添加了最大循环次数。在模型思考输出后，将思考结果添加到消息历史中。然后检测模型输出中是否包含"Action"标记，如果有就提取JSON，再通过字典查找的方式`available_actions[function_name]`动态调用对应的工具函数，接着将工具的执行结果添加到消息历史中传递给模型，继续循环。

通过这样交替添加模型响应和工具执行结果，代码构建了一个完整的对话上下文。这让模型能够参考之前的思考和行动结果，实现连贯的多轮对话。

最后如果模型输出了"Answer"标记，代表已经得到答案，结束循环。


### 2.4 工具函数实现 (actions.py)

这段代码实现了两个工具：`wikipedia`和`calculate`。`wikipedia`用于搜索Wikipedia，返回摘要，`calculate`用于计算数学表达式，返回结果。

```python
import httpx

def wikipedia(q: str):
    return httpx.get("https://en.wikipedia.org/w/api.php", params={
        "action": "query",
        "list": "search",
        "srsearch": q,
        "format": "json"
    }).json()["query"]["search"][0]["snippet"]

def calculate(operation: str) -> float:
    return eval(operation)

available_actions = {
    "wikipedia": wikipedia,
    "calculate": calculate,
}
```

### 2.5 辅助函数实现 (helpers.py)

这段代码实现了一个名为`extract_json`的辅助函数，用于从输入字符串中提取JSON对象。

```python
import json

def extract_json(input_string, return_dict=True):
    # 用于跟踪潜在 JSON 对象的变量
    potential_jsons = []
    stack = []
    start_index = None
    
    for i, char in enumerate(input_string):
        if char == '{' and not stack:
            # 第一个开括号
            start_index = i
            stack.append(char)
        elif char == '{':
            # 嵌套的开括号
            stack.append(char)
        elif char == '}' and stack:
            # 闭括号
            stack.pop()
            if not stack:
                # 找到一个完整的类 JSON 字符串
                json_candidate = input_string[start_index:i+1]
                potential_jsons.append(json_candidate)
    
    # 验证每个候选字符串并转换为字典（如果需要）
    valid_jsons = []
    for candidate in potential_jsons:
        try:
            # 尝试解析 JSON
            parsed_dict = json.loads(candidate)
            if return_dict:
                valid_jsons.append(parsed_dict)  # 返回 Python 字典
            else:
                valid_jsons.append(candidate)    # 返回原始 JSON 字符串
        except json.JSONDecodeError:
            # 不是有效的 JSON，跳过
            continue
    
    return valid_jsons
```

### 2.6 环境配置

创建`.env`文件：
```
OPENAI_API_KEY=your_api_key_here
OPENAI_API_BASE=your_api_base_here
```

### 2.7 主程序入口 (main.py)

这段代码实现了主程序的入口点。它加载环境变量，初始化OpenAI客户端，并调用`Agent.run`方法来运行Agent。

```python
import os
from openai import OpenAI
from dotenv import load_dotenv

from agent import Agent

# Load environment variables
load_dotenv()

# Initialize OpenAI client
openai_client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"), base_url=os.getenv("OPENAI_API_BASE"))

Agent(client=openai_client).run("What is current age of Mr. Nadrendra Modi multiplied by 2?")
```

我们向agent提问：Nadrendra Modi先生现在的年龄乘以2是多少？

agent在第一轮调用了wikipedia工具进行年龄查询，在第二轮调用了calculate工具进行计算，并在第三轮给出了最终结果。

输出结果：
```sh
------Loop: 1
Thought: I need to find the current age of Mr. Narendra Modi. I'll search for his birthdate on Wikipedia, calculate his current age, and then multiply that by 2.

Action:
{
  "function_name": "wikipedia",
  "function_parms": {
    "q": "Narendra Modi"
  }
}
PAUSE
***Running action: wikipedia {'q': 'Narendra Modi'}
***Running result: Action_Response: <span class="searchmatch">Narendra</span> Damodardas <span class="searchmatch">Modi</span> (born 17 September 1950) is an Indian politician who has served as the prime minister of India since 2014. <span class="searchmatch">Modi</span> was the chief
------Loop: 2
Thought: Narendra Modi was born on 17 September 1950. I will calculate his age as of today and then multiply it by 2.

Action:
{
  "function_name": "calculate",
  "function_parms": {
    "operation": "(2023 - 1950) * 2"
  }
}
PAUSE
***Running action: calculate {'operation': '(2023 - 1950) * 2'}
***Running result: Action_Response: 146
------Loop: 3
Answer: The current age of Mr. Narendra Modi multiplied by 2 is 146.
```

## 三、结语

这个项目展示了如何构建一个基于ReAct模式的AI智能体。通过模块化设计和清晰的代码结构，我们实现了一个可扩展的智能体框架。关键在于理解每个组件的职责和它们之间的协作方式。

希望这篇详细的代码解读能帮助你理解AI智能体的工作原理，并在此基础上构建出更强大的应用。
