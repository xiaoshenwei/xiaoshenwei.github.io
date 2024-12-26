

# LangChain

**LangChain** is a framework for developing applications powered by large language models (LLMs).

# 组件

![](https://raw.githubusercontent.com/xiaoshenwei/xiaoshenwei.github.io/master/assets/images/202412231529243.png)

langchain==0.3.7

思考?

- 如何让大模型返回 json 数据？
- 如何让大模型 使用本地代码？
- 如何让大模型 实现实时搜索?

做一道逻辑题

``` markdown
逻辑题：三个人的年纪
有三个人：A、B、C，他们的年纪分别是 15、20、25 岁，但我们不知道谁对应哪一个年龄。已知以下信息：

A 说：“B 比我小。”
B 说：“C 比我大。”
C 说：“A 比我小。”
请问：谁对应哪一个年龄？
```

# 输入输出

输入提示 > 调用模型 > 输出解析

在模型 I/O 的每个环节，LangChain 都为提供了模板和工具，快捷地形成调用各种语言模型的接口。

1. 提示模板：使用模型的第一个环节是把提示信息输入到模型中，你可以创建 LangChain 模板，根据实际需求动态选择不同的输入，针对特定的任务和应用调整输入。
2. 语言模型：LangChain 允许你通过通用接口来调用语言模型。这意味着无论你要使用的是哪种语言模型，都可以通过同一种方式进行调用，这样就提高了灵活性和便利性。
3. 输出解析：LangChain 还提供了从模型输出中提取信息的功能。通过输出解析器，你可以精确地从模型的输出中获取需要的信息，而不需要处理冗余或不相关的数据，更重要的是还可以把大模型给回的非结构化文本，转换成程序可以处理的结构化数据。

## 提示模版

语言模型是个无穷无尽的宝藏，人类的知识和智慧，好像都封装在了这个“魔盒”里面了。但是，怎样才能解锁其中的奥秘，那可就是仁者见仁智者见智了。所以，现在“提示工程”这个词特别流行，所谓 Prompt Engineering，就是专门研究对大语言模型的提示构建。

``` python
from langchain_ollama import ChatOllama

model = ChatOllama(
    base_url='http://0.0.0.0:11434',
    model="qwen2.5:latest",
)

from langchain_core.prompts import PromptTemplate

template = """你是一名专业的 {language} 语言开发者"""
prompt = PromptTemplate.from_template(template)

model.invoke(prompt.format(language="python"))
```

## 输出解析

在开发具体应用的过程中，很明显我们不仅仅需要文字，更多情况下我们需要的是程序能够直接处理的、结构化的数据。

``` python
from langchain_core.prompts import PromptTemplate
from langchain.output_parsers import PydanticOutputParser
from pydantic import BaseModel, Field

class Language(BaseModel):
    feature: str = Field(description="此开发语言的特点")
    usage: str = Field(description="此开发语言的使用场景")

parser = PydanticOutputParser(pydantic_object=Language)

prompt = PromptTemplate(
    template="Answer the user query.\n{format_instructions}\n{query}\n",
    input_variables=["query"],
    partial_variables={"format_instructions": parser.get_format_instructions()},
)

prompt_and_model = prompt | model
output = prompt_and_model.invoke({"query": "介绍一下python开发语言"})
parser.invoke(output)
```

json

``` py
from langchain_core.prompts import PromptTemplate
from langchain.output_parsers import ResponseSchema, StructuredOutputParser

response_schems = [
    ResponseSchema(name="feature", description="此开发语言的特点"),
    ResponseSchema(name="usage", description="此开发语言的使用场景"),
]

parser = StructuredOutputParser.from_response_schemas(response_schems)

prompt = PromptTemplate(
    template="Answer the user query.\n{format_instructions}\n{query}\n",
    input_variables=["query"],
    partial_variables={"format_instructions": parser.get_format_instructions()},
)

prompt_and_model = prompt | model
output = prompt_and_model.invoke({"query": "介绍一下python开发语言"})
parser.invoke(output)
```

````shell
# print(parser.get_format_instructions())
The output should be a markdown code snippet formatted in the following schema, including the leading and trailing "```json" and "```":

```json
{
	"feature": string  // 此开发语言的特点
	"usage": string  // 此开发语言的使用场景
}
```
````

## 提示工程

![](https://raw.githubusercontent.com/xiaoshenwei/xiaoshenwei.github.io/master/assets/images/b77a15cd83b66bba55032d711bcf3c16.webp)

-  指令（Instuction）告诉模型这个任务大概要做什么、怎么做，比如如何使用提供的外部信息、如何处理查询以及如何构造输出。这通常是一个提示模板中比较固定的部分。一个常见用例是告诉模型“你是一个有用的 XX 助手”，这会让他更认真地对待自己的角色。
- 上下文（Context）则充当模型的额外知识来源。这些信息可以手动插入到提示中，通过矢量数据库检索得来，或通过其他方式（如调用 API、计算器等工具）拉入。一个常见的用例时是把从向量数据库查询到的知识作为上下文传递给模型。
- 提示输入（Prompt Input）通常就是具体的问题或者需要大模型做的具体事情，这个部分和“指令”部分其实也可以合二为一。但是拆分出来成为一个独立的组件，就更加结构化，便于复用模板。这通常是作为变量，在调用模型之前传递给提示模板，以形成具体的提示。
- 输出指示器（Output Indicator）标记要生成的文本的开始。这就像我们小时候的数学考卷，先写一个“解”

## COT

CoT 这个概念来源于学术界，是谷歌大脑的 Jason Wei 等人于 2022 年在论文《Chain-of-Thought Prompting Elicits Reasoning in Large Language Models（自我一致性提升了语言模型中的思维链推理能力）》中提出来的概念。它提出，如果生成一系列的中间推理步骤，就能够显著提高大型语言模型进行复杂推理的能力。

``` tex
Let's think step by step.
```

```
1 + 2+3 +4 = ?

1 + 2+3 +4 = ? Let's think step by step. 
```

用了思维链后，大模型把任务做了拆分并展示了每一步思考的过程。

如果是一个复杂任务，或者我们想知道某个问题的具体解决步骤，那显然使用思维链效果会更好。

这其实就是让大模型不要着急回答问题，而是先推理拆解问题，然后再回答，这样一句简单的 prompt 就让大模型有了推理能力。

# Chain

复杂的应用程序，那么就需要通过 “Chain” 来链接 LangChain 的各个组件和功能——模型之间彼此链接，或模型与其他组件链接。

说到链的实现和使用，也简单。

- 首先 LangChain 通过设计好的接口，实现一个具体的链的功能。例如，LLM 链（LLMChain）能够接受用户输入，使用 PromptTemplate 对其进行格式化，然后将格式化的响应传递给 LLM。这就相当于把整个 Model I/O 的流程封装到链里面。
- 实现了链的具体功能之后，我们可以通过将多个链组合在一起，或者将链与其他组件组合来构建更复杂的链。所以你看，链在内部把一系列的功能进行封装，而链的外部则又可以组合串联。

链其实可以被视为 LangChain 中的一种基本功能单元。

## Llmchain

``` python
# 导入所需的库
from langchain import PromptTemplate, LLMChain
from langchain_ollama import ChatOllama
# 原始字符串模板
template = "{language}的作者是?"
# 创建模型实例

llm = ChatOllama(
    base_url='http://0.0.0.0:11434',
    model="qwen2.5:latest",
)

# 创建LLMChain
llm_chain = LLMChain(
    llm=llm,
    prompt=PromptTemplate.from_template(template))
# 调用LLMChain，返回结果
llm_chain.invoke("python")

```

## Sequential Chain

顺序链（Sequential Chain ）允许用户连接多个链并将它们组合成执行特定场景的流水线（Pipeline）

``` python
from langchain_ollama import ChatOllama
from langchain.chains import LLMChain, SequentialChain
from langchain.prompts import PromptTemplate

llm = ChatOllama(
    base_url='http://0.0.0.0:11434',
    model="qwen2.5:latest",
)

# 这是第一个LLMChain，用于生成鲜花的介绍，输入为花的名称和种类
template = """
你是一个植物学家。给定花的名称和类型，你需要为这种花写一个200字左右的介绍。

花名: {name}
颜色: {color}
植物学家: 这是关于上述花的介绍:"""
prompt_template = PromptTemplate(input_variables=["name", "color"], template=template)
introduction_chain = LLMChain(llm=llm, prompt=prompt_template, output_key="introduction")

# 这是第二个LLMChain，用于根据鲜花的介绍写出鲜花的评论
template = """
你是一位鲜花评论家。给定一种花的介绍，你需要为这种花写一篇200字左右的评论。

鲜花介绍:
{introduction}
花评人对上述花的评论:"""
prompt_template = PromptTemplate(input_variables=["introduction"], template=template)
review_chain = LLMChain(llm=llm, prompt=prompt_template, output_key="review")

# 这是第三个LLMChain，用于根据鲜花的介绍和评论写出一篇自媒体的文案
template = """
你是一家花店的社交媒体经理。给定一种花的介绍和评论，你需要为这种花写一篇社交媒体的帖子，300字左右。

鲜花介绍:
{introduction}
花评人对上述花的评论:
{review}

社交媒体帖子:
"""
prompt_template = PromptTemplate(input_variables=["introduction", "review"], template=template)
social_post_chain = LLMChain(llm=llm, prompt=prompt_template, output_key="social_post_text")

# 这是总的链，我们按顺序运行这三个链
overall_chain = SequentialChain(
    chains=[introduction_chain, review_chain, social_post_chain],
    input_variables=["name", "color"],
    output_variables=["introduction","review","social_post_text"],
    verbose=True)

# 运行链，并打印结果
result = overall_chain({"name":"玫瑰", "color": "黑色"})
print(result)
```

## RouterChain

RouterChain，也叫路由链，能动态选择用于给定输入的下一个链。

1 . 构建大模型

``` python
from langchain.globals import set_debug

set_debug(True)

from langchain_ollama import ChatOllama

llm = ChatOllama(
    base_url='http://0.0.0.0:11434',
    model="qwen2.5:latest",
)

```

2. 定义三种职业

``` python
from langchain.chains import LLMChain
from langchain.prompts import PromptTemplate

# 定义用于医生的链：提供诊断建议
doctor_template = """
你是一位医生，请根据以下症状给出诊断建议：
- 症状：{input}
- 请提供可能的诊断结果并给出治疗建议。
"""
doctor_prompt = PromptTemplate(input_variables=["input"], template=doctor_template)
doctor_chain = LLMChain(llm=llm, prompt=doctor_prompt, verbose=True)

# 定义 Python 开发者的链：提供编程建议或解决编程问题
python_developer_template = """
你是一位 Python 开发者，请根据以下编程问题提供解决方案：
- 问题：{input}
- 请提供详细的 Python 编程解决方案或优化建议。
"""
python_developer_prompt = PromptTemplate(input_variables=["input"], template=python_developer_template)
python_developer_chain = LLMChain(llm=llm, prompt=python_developer_prompt, verbose=True)

# 定义厨师的链：提供食谱或烹饪建议
chef_template = """
你是一位厨师，请根据以下食材提供烹饪建议或食谱：
- 食材：{input}
- 请提供一道美味的菜肴食谱，包含所需食材、步骤和烹饪时间。
"""
chef_prompt = PromptTemplate(input_variables=["input"], template=chef_template)
chef_chain = LLMChain(llm=llm, prompt=chef_prompt, verbose=True)

from langchain.chains import ConversationChain
# 默认路由
default_chain = ConversationChain(llm=llm, output_key="text", verbose=True)
```

3. 定义路由链

``` python
destination_chains = {
    "doctor": doctor_chain,
    "python_developer": python_developer_chain,
    "chef": chef_chain,
}
describe_prompt = """
doctor: 擅长回答医学类问题
python_developer: 擅长回答python相关问题
chef: 擅长回答烹饪相关的问题
"""

from langchain.chains.router.multi_prompt_prompt import MULTI_PROMPT_ROUTER_TEMPLATE

router_template = MULTI_PROMPT_ROUTER_TEMPLATE.format(destinations=describe_prompt)

router_template

from langchain.chains.router.llm_router import RouterOutputParser, LLMRouterChain
# 通过路由提示词模板，构建路由提示词
router_prompt = PromptTemplate(
    template=router_template,
    input_variables=["input"],
    output_parser=RouterOutputParser(),
)
router_chain = LLMRouterChain.from_llm(llm, router_prompt, verbose=True)

```

4. 最终的链

``` python
from langchain.chains.router import MultiPromptChain
chain = MultiPromptChain(
    router_chain=router_chain,
    destination_chains=destination_chains,
    default_chain=default_chain,
    verbose=True,
)
# chain.invoke("我最近感到头痛，应该怎么办？")
# chain.invoke("如何定义一个class")
chain.invoke("我想成为亿万富翁")
```

# Agent

## Tool Call

虽然名称“工具调用”意味着模型直接执行某些操作，但实际上并非如此！该模型仅生成工具的参数，实际运行（或不运行）该工具取决于用户.

``` python
from langchain_ollama import ChatOllama

llm = ChatOllama(
    base_url='http://0.0.0.0:11434',
    model="qwen2.5:latest",
)

from langchain_core.tools import tool


@tool
def add(a: int, b: int) -> int:
    """Adds a and b."""
    return a + b

@tool
def multiply(a: int, b: int) -> int:
    """Multiplies a and b."""
    return a * b


tools = [add, multiply]

llm_with_tools = llm.bind_tools(tools)

from langchain_core.messages import HumanMessage

query = "What is 3 * 12? Also, what is 11 + 49?"

messages = [HumanMessage(query)]

ai_msg = llm_with_tools.invoke(messages)

print(ai_msg.tool_calls)

messages.append(ai_msg)

for tool_call in ai_msg.tool_calls:
    selected_tool = {"add": add, "multiply": multiply}[tool_call["name"].lower()]
    tool_msg = selected_tool.invoke(tool_call)
    messages.append(tool_msg)

llm_with_tools.invoke(messages)
```

查询天气

``` python
from langchain_ollama import ChatOllama

llm = ChatOllama(
    base_url='http://0.0.0.0:11434',
    model="qwen2.5:latest",
)

import requests
import json
import pandas as pd

# 高德天气API查询函数
def search_adcode(city: str) -> str:
    API_KEY = "xxx"  # 请替换为你自己的高德API Key
    url = f"https://restapi.amap.com/v3/weather/weatherInfo?city={city}&key={API_KEY}&extensions=base"
    
    response = requests.get(url)
    data = response.json()
    return json.dumps(data)

def get_weather(city_name: str):
    df = pd.read_excel("data/AMap_adcode_citycode.xlsx")
    # 按行遍历
    for index, row in df.iterrows():
        if city_name in row['中文名']:
            return "城市名称: %s, adcode: %d" % (row['中文名'], row['adcode'])
    return "为找到城市的adcode"
    
    
from langchain.agents import initialize_agent, Tool, AgentType

# 定义查询天气的Tool
weather_tool = Tool(
    name="WeatherTool",
    func=get_weather,
    description="Use this tool to get the current weather in a specified adcode."
)

adcode_search_tool = Tool(
    name="SearchAdcodeTool",
    func=search_adcode,
    description="Use this tool to get the city adcode in a specified city name."
)

# 使用Agent进行查询
tools = [weather_tool, adcode_search_tool]
agent = initialize_agent(tools, llm, agent_type=AgentType.ZERO_SHOT_REACT_DESCRIPTION, verbose=True)

agent.invoke("今天滨江天气怎么样?")
```



## 作用

每当你遇到这种需要模型做自主判断、自行调用工具、自行决定下一步行动的时候，Agent（也就是代理）就出场了。

代理就像一个多功能的接口，它能够接触并使用一套工具。根据用户的输入，代理会决定调用哪些工具。它不仅可以同时使用多种工具，而且可以将一个工具的输出数据作为另一个工具的输入数据。



在 LangChain 中使用代理，我们只需要理解下面三个元素。

- 大模型：提供逻辑的引擎，负责生成预测和处理输入。
- 与之交互的外部工具：可能包括数据清洗工具、搜索引擎、应用程序等。
- 控制交互的代理：调用适当的外部工具，并管理整个交互过程的流程。

![](https://raw.githubusercontent.com/xiaoshenwei/xiaoshenwei.github.io/master/assets/images/202412241622599.png)

问题:

- 如何确定 调用哪个工具?
- 如何确认可以开始下一步？

LangChain 中的代理是怎样自主计划、自行判断，并执行行动的呢？



## ReAct 框架

思考: 如果遇到一个任务， 如何做出决策并完成下一步动作?

例如： 我开了一家 花店，每天早上如何为鲜花定价呢 ？

1. 搜索: 玫瑰价格
2. 观察：市场价格
3. 思考: 如何定价
4. 行动: 确认售价

ReAct 框架的灵感正是来自“行动”和“推理”之间的协同作用，这种协同作用使得咱们人类能够学习新任务并做出决策或推理。这个框架，也是大模型能够作为“智能代理”，自主、连续、交错地生成推理轨迹和任务特定操作的理论基础。

引导模型生成一个任务解决轨迹：观察环境 - 进行思考 - 采取行动，也就是观察 - 思考 - 行动。那么，再进一步进行简化，就变成了推理 - 行动，也就是 Reasoning-Acting 框架。

![](https://raw.githubusercontent.com/xiaoshenwei/xiaoshenwei.github.io/master/assets/images/202412251603748.png)

ReAct 发明的初衷是为了解决两个问题，

- 第一，大模型执行的结果不可观测，导致出现“幻觉”；
- 第二，大模型不能与外部环境交互，导致无法回答一些特定垂直领域或者实时问题。

LangChain 正是通过 Agent 类，将 ReAct 框架进行了完美封装和实现，这一下子就赋予了大模型极大的自主性（Autonomy），你的大模型现在从一个仅仅可以通过自己内部知识进行对话聊天的 Bot，飞升为了一个有手有脚能使用工具的智能代理。

### 通过代理实现 ReAct 框架

任务: 获取黄金当前的价格，然后计算出加价 20% 后的新价格。

``` python
# 设置OpenAI和SERPAPI的API密钥
import os
os.environ["SERPAPI_API_KEY"] = 'xxx'

from langchain.agents import load_tools
from langchain.agents import initialize_agent
from langchain.agents import AgentType


tools = load_tools(["serpapi", "llm-math"], llm=llm)

agent = initialize_agent(tools, llm, agent=AgentType.ZERO_SHOT_REACT_DESCRIPTION, verbose=True)

agent.run("目前市场上黄金的平均价格每克多少人民币？如果我在此基础上加价20%卖出，应该如何定价？")
```

思考？

Chain 和 Agent 区别是什么？



操作的序列并非硬编码在代码中，而是使用语言模型（如 GPT-3 或 GPT-4）来选择执行的操作序列。



### 关键组件

1. 代理（Agent）：这个类决定下一步执行什么操作。它由一个语言模型和一个提示（prompt）驱动。提示可能包含代理的性格（也就是给它分配角色，让它以特定方式进行响应）、任务的背景（用于给它提供更多任务类型的上下文）以及用于激发更好推理能力的提示策略（例如 ReAct）。LangChain 中包含很多种不同类型的代理。
2. 工具（Tools）：工具是代理调用的函数。这里有两个重要的考虑因素：一是让代理能访问到正确的工具，二是以最有帮助的方式描述这些工具。如果你没有给代理提供正确的工具，它将无法完成任务。如果你没有正确地描述工具，代理将不知道如何使用它们。LangChain 提供了一系列的工具，同时你也可以定义自己的工具。
3. 工具包（Toolkits）：工具包是一组用于完成特定目标的彼此相关的工具，每个工具包中包含多个工具。比如 LangChain 的 Office365 工具包中就包含连接 Outlook、读取邮件列表、发送邮件等一系列工具。当然 LangChain 中还有很多其他工具包供你使用。
4. 代理执行器（AgentExecutor）：代理执行器是代理的运行环境，它调用代理并执行代理选择的操作。执行器也负责处理多种复杂情况，包括处理代理选择了不存在的工具的情况、处理工具出错的情况、处理代理产生的无法解析成工具调用的输出的情况，以及在代理决策和工具调用进行观察和日志记录。

https://smith.langchain.com/hub/hwchase17/react

```markdown
Answer the following questions as best you can. You have access to the following tools:

Search: A search engine. Useful for when you need to answer questions about current events. Input should be a search query.
Calculator: Useful for when you need to answer questions about math.

Use the following format:

Question: the input question you must answer

Thought: you should always think about what to do
Action: the action to take, should be one of [Search, Calculator]
Action Input: the input to the action
Observation: the result of the action
(this Thought/Action/Action Input/Observation can repeat N times)

Thought: I now know the final answer
Final Answer: the final answer to the original input question
```

# 相关链接

https://github.com/langchain-ai/langchain

https://smith.langchain.com/hub/