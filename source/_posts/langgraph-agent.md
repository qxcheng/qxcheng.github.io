---
title: 6.langgraph创建agent代理并使用搜索工具
date: 2025-01-15 12:30:29
tags:
- 大模型             
categories: 
- langchain
description: '在这篇教程中，我们将学习如何使用 LangGraph 和 LangChain 构建一个具有记忆功能的智能对话代理。这个代理能够使用搜索工具来回答问题，并保持对话的连续性。'
---

在这篇教程中，我们将学习如何使用 LangGraph 和 LangChain 构建一个具有记忆功能的智能对话代理。这个代理能够使用搜索工具来回答问题，并保持对话的连续性。

## 1.获取搜索工具的apikey

登录搜索工具网站: `https://tavily.com/`，按提示生成api-key后有1000的免费调用额度。然后我们需要在python代码中设置环境变量 `TAVILY_API_KEY`为获取到的api-key。

```python
os.environ["TAVILY_API_KEY"] = "tvly-*****"
```

## 2.安装依赖

```python
pip install tavily-python langchain_community 
```

## 3. 依赖导入

首先，我们需要导入必要的组件：

```python
from langchain_openai import ChatOpenAI
from langchain_community.tools.tavily_search import TavilySearchResults
from langchain_core.messages import HumanMessage
from langgraph.checkpoint.memory import MemorySaver
from langgraph.prebuilt import create_react_agent
```

这些导入包括：
- ChatOpenAI：OpenAI的聊天模型接口
- TavilySearchResults：网络搜索工具
- HumanMessage：用于构建人类消息的类
- MemorySaver：用于保存对话历史的记忆组件
- create_react_agent：用于创建 ReAct 范式的代理

## 4. 组件配置

### 4.1 设置语言模型

```python
model = ChatOpenAI(model="gpt-4o-mini")
```

我们使用 OpenAI 的 GPT-4 模型作为代理的大脑。这个模型将负责理解用户输入并决定如何使用工具。

### 4.2 配置记忆系统

```python
memory = MemorySaver()
```

MemorySaver 组件允许代理记住之前的对话内容，这对于维持连贯的对话至关重要。

### 4.3 设置搜索工具

```python
search = TavilySearchResults(max_results=2)
tools = [search]
```

我们使用 Tavily 搜索工具，它能够：
- 执行网络搜索
- 返回最相关的前两个结果
- 帮助代理获取最新信息

## 5. 创建智能代理

```python
agent_executor = create_react_agent(model, tools, checkpointer=memory)
```

create_react_agent 函数将所有组件组合成一个完整的代理系统：
1. 使用指定的语言模型作为决策核心
2. 集成搜索工具供代理使用
3. 添加记忆系统来保存对话历史

## 6. 交互式对话实现

```python
config = {"configurable": {"thread_id": "abc123"}}

while True:
    query = input("请输入问题:")
    for chunk in agent_executor.stream(
        {"messages": [HumanMessage(content=query)]}, config
    ):
        print(chunk)
        print("----")
    print('\n')
```

这段代码实现了：
1. 创建一个持续的对话循环
2. 接收用户输入的问题
3. 使用流式输出显示代理的思考和回答过程
4. 通过 thread_id 跟踪对话上下文

## 7.使用说明

使用这个智能代理系统时：
1. 代理会实时显示其思考过程
2. 可以问任何问题，代理会根据需要使用搜索工具
3. 代理会记住之前的对话内容，可以进行连续的对话
4. 如果需要退出对话，可以使用 Ctrl+C 终止程序

## 8.示例

```json
请输入问题:明天上海的天气如何？
{'agent': {'messages': [AIMessage(content='', additional_kwargs={'tool_calls': [{'id': 'call_Wq2uNmcLynn6MSURC8yXJuiY', 'function': {'arguments': '{"query":"明天上海天气预报"}', 'name': 'tavily_search_results_json'}, 'type': 'function'}], 'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 23, 'prompt_tokens': 86, 'total_tokens': 109, 'completion_tokens_details': None, 'prompt_tokens_details': None}, 'model_name': 'gpt-4o-mini', 'system_fingerprint': 'fp_04751d0b65', 'finish_reason': 'tool_calls', 'logprobs': None}, id='run-65d3026c-7047-4679-b868-89a9f6dde55a-0', tool_calls=[{'name': 'tavily_search_results_json', 'args': {'query': '明天上海天气预报'}, 'id': 'call_Wq2uNmcLynn6MSURC8yXJuiY', 'type': 'tool_call'}], usage_metadata={'input_tokens': 86, 'output_tokens': 23, 'total_tokens': 109, 'input_token_details': {}, 'output_token_details': {}})]}}
----
{'tools': {'messages': [ToolMessage(content='[{"url": "https://www.weather-atlas.com/zh/china/shanghai-weather-tomorrow", "content": " 明天天气 - 上海市, 中国 Weather Atlas 搜索国家或城市 国家（多个） 单位 °摄氏度 °华氏度 语言 english español 中文 联系 联系 关于我们 术语表 使用条款 隐私政策 Cookie 政策 免责声明 国家（多个） 中国 上海市 明天 明天的天气预报 上海市, 中国 今天 明天 长期 历 年一月二月三月四月五月六月七月八月九月十月十一月十二月 °摄氏度°华氏度 分享更多 Email Viber WhatsApp 扩大 目录 明天的天气预报 每小时天气预报 您所在位置的天气预报 明天的天气预报和温度 - 星期三, 20. 十一月 CST 大部分地区多云 16°摄氏度 / 9°摄氏度 风: 14公里 每小时 352° 湿度: 59% 降水概率: 4% 降水: 0毫米 紫外线指数: 3 每小时天气预报 - 上海市, 中国 0小时 12°摄氏度 感觉就像: 11°摄氏度 局部多云 风: 10公里每小时 347° 压力: 1027毫巴 湿度: 73% 降水概率: 4% 12°摄氏度 11°摄氏度 11°摄氏度 10°摄氏度 11°摄氏度 10°摄氏 度 13°摄氏度 12°摄氏度 15°摄氏度 14°摄氏度 16°摄氏度 15°摄氏度 15°摄氏度 15°摄氏度 15°摄氏度 14°摄氏度 14°摄氏度 13°摄氏度 13° 摄氏度 12°摄氏度 13°摄氏度 12°摄氏度 发布者: Weather Atlas | 关于我们 数据来源 | weather.com | Weather 上海市, 中国 明天的天气 预报 © 2002-2024|联系|关于我们|"}, {"url": "https://www.weather.com.cn/shanghai/weather.shtml", "content": "上海天气预报，上海 天气预报还提供上海各区县的生活指数、 健康指数、交通指数、旅游指数，及时发布上海气象预警信号、各类气象资讯。 ... 今天最高温度20 度，明天最低温度15度。火险等级气象指数：4级，易燃。上班（6-9时）：晴到多云，11-15度；下班（16"}]', name='tavily_search_results_json', id='494dc3bb-0fd0-4367-8e81-034e87848c45', tool_call_id='call_Wq2uNmcLynn6MSURC8yXJuiY', artifact={'query': '明天上海天 气预报', 'follow_up_questions': None, 'answer': None, 'images': [], 'results': [{'title': '明天天气 - 上海市, 中国 - Weather Atlas', 'url': 'https://www.weather-atlas.com/zh/china/shanghai-weather-tomorrow', 'content': '明天天气 - 上海市, 中国 Weather Atlas 搜索国家或城市 国家（多个） 单位 °摄氏度 °华氏度 语言 english español 中文 联系 联系 关于我们 术语表 使用条款 隐私政策 Cookie 政策 免责声明 国家（多个） 中国 上海市 明天 明天的天气预报 上海市, 中国 今天 明天 长期 历年一月二月三月四月五月六月七月八月 九月十月十一月十二月 °摄氏度°华氏度 分享更多 Email Viber WhatsApp 扩大 目录 明天的天气预报 每小时天气预报 您所在位置的天气预报 明天的天气预报和温度 - 星期三, 20. 十一月 CST 大部分地区多云 16°摄氏度 / 9°摄氏度 风: 14公里每小时 352° 湿度: 59% 降水概率: 4% 降水: 0毫米 紫外线指数: 3 每小时天气预报 - 上海市, 中国 0小时 12°摄氏度 感觉就像: 11°摄氏度 局部多云 风: 10公里每小时 347° 压力: 1027毫巴 湿度: 73% 降水概率: 4% 12°摄氏度 11°摄氏度 11°摄氏度 10°摄氏度 11°摄氏度 10°摄氏度 13°摄氏度 12°摄氏度 15°摄氏度 14°摄氏度 16°摄氏度 15°摄氏度 15°摄氏度 15°摄氏度 15°摄氏度 14°摄氏度 14°摄氏度 13°摄氏度 13°摄氏度 12°摄氏度 13°摄氏度 12°摄氏度 发布者: Weather Atlas | 关于我们 数据来源 | weather.com | Weather 上海市, 中国 明天的天气预报 © 2002-2024|联系|关于我们|', 'score': 0.9973888, 'raw_content': None}, {'title': '上海首页 - 上海 - 中国天气网', 'url': 'https://www.weather.com.cn/shanghai/weather.shtml', 'content': '上海天气预报，上海天气预报还提供上海各区县的生活指数、 健康指数、交通指数、旅游指数，及时发布上海气象 预警信号、各类气象资讯。 ... 今天最高温度20度，明天最低温度15度。火险等级气象指数：4级，易燃。上班（6-9时）：晴到多云，11-15度 ；下班（16', 'score': 0.99589807, 'raw_content': None}], 'response_time': 2.85})]}}
----
{'agent': {'messages': [AIMessage(content=' 明天（11月20日）上海的天气预报显示：\n\n- **天气状况**：大部分地区多云\n- **气温**：最高温度16°C，最低温度9°C\n- **风速**：14公里每小时\n- **湿度**：59%\n- **降水概率**：4%\n- **降水量**：0毫米\n\n整体来说，明天的天气适合出行，您可以查看更详细的信息 [这 里](https://www.weather-atlas.com/zh/china/shanghai-weather-tomorrow)。', additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 124, 'prompt_tokens': 727, 'total_tokens': 851, 'completion_tokens_details': None, 'prompt_tokens_details': None}, 'model_name': 'gpt-4o-mini', 'system_fingerprint': 'fp_04751d0b65', 'finish_reason': 'stop', 'logprobs': None}, id='run-a4ac2976-a80b-4b4c-8844-d249f94ce1d3-0', usage_metadata={'input_tokens': 727, 'output_tokens': 124, 'total_tokens': 851, 'input_token_details': {}, 'output_token_details': {}})]}}
```

## 9.总结

通过这个教程，我们学习了如何：
1. 使用 LangGraph 构建智能代理
2. 集成大语言模型作为决策核心
3. 添加实用工具扩展代理能力
4. 实现记忆功能保持对话连续性

这个系统为构建更复杂的智能代理奠定了基础。你可以通过添加更多工具或自定义代理的行为来增强系统的功能。

## 10.完整代码

```python
from langchain_openai import ChatOpenAI
from langchain_community.tools.tavily_search import TavilySearchResults
from langchain_core.messages import HumanMessage
from langgraph.checkpoint.memory import MemorySaver
from langgraph.prebuilt import create_react_agent

# 定义模型
model = ChatOpenAI(model="gpt-4o-mini")

# 定义记忆存储
memory = MemorySaver()

# 定义搜索工具
search = TavilySearchResults(max_results=2)
tools = [search]

# 创建代理（使用模型、工具、记忆存储)
agent_executor = create_react_agent(model, tools, checkpointer=memory)

# 使用代理进行聊天
config = {"configurable": {"thread_id": "abc123"}}

while True:
    query = input("请输入问题:")
    for chunk in agent_executor.stream(
        {"messages": [HumanMessage(content=query)]}, config
    ):
        print(chunk)
        print("----")
    print('\n')
```