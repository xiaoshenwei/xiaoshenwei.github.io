# 大模型助力日志分析

<img src="https://raw.githubusercontent.com/xiaoshenwei/xiaoshenwei.github.io/master/assets/images/logai.png" style="zoom:60%;" />

从大量的错误日志中获取关键信息， 快速定位问题



# 架构

![](https://raw.githubusercontent.com/xiaoshenwei/xiaoshenwei.github.io/master/assets/images/2024-08-28-1508.png)

# 从es中获取日志

 ```python
 import os
 from elasticsearch import Elasticsearch
 
 client = Elasticsearch(
     os.environ["ELASTICSEARCH_HOST"],
     api_key=os.environ["ELASTICSEARCH_API_KEY"],
 )
 
 
 def get_query_body(name: str) -> dict:
     return {
         "sort": [{"@timestamp": {"order": "desc"}}],
         "_source": ["message"],
         "size": 200,
         "query": {
             "bool": {
                 "filter": [
                     {"match": {"message": "error"}},
                     {"range": {"@timestamp": {"gte": "now-10m", "lte": "now"}}},
                     {"term": {"app.keyword": name}},
                 ]
             }
         },
     }
 
 
 def query(name: str):
     return client.search(
         index="logs-*",
         body=get_query_body(name),
     )
 
 
 if __name__ == "__main__":
     print(query("dotcom-grey"))
 
 ```

只获取 10min 内错误日志， 由于大模型token 一般都有限制， 按时间反序，只获取 200 条数据



# Langchain 调用大模型

```python
from langchain.chat_models import init_chat_model
from langchain_core.prompts import ChatPromptTemplate

llm = init_chat_model(
    "qwen2.5:7b",
    model_provider="ollama",
    base_url="http://ollama-api.k8s-test.uc.host.dxy"
)

prompt_template = """
你是一名 Java 资深运维工程师，负责分析 Java 程序的错误日志。请按以下步骤分析日志：
1. 识别错误类型（如 `NullPointerException`、`SQLException`）
2. 提取错误消息
3. 找出错误发生的代码位置（类、方法、行号）
4. 结合上下文，分析问题的根因

日志内容：
{context}
"""

prompt = ChatPromptTemplate.from_template(prompt_template)
llm_with_prompt = prompt | llm

logs = """
2023-08-01 12:00:00 ERROR [main] com.example.App - NullPointerException: null
at com.example.App.main(App.java:10)
2023-08-01 12:00:00 ERROR [main] com.example.App - SQLException: SQL syntax error
at com.example.App.main(App.java:20)
"""

result = llm_with_prompt.invoke({"context": logs})

if __name__ == "__main__":
    print(llm.get_num_tokens(logs))
    print(result.content)

```

# 思考

- 如果日志量很大， 模型 支持的最大 token 有限
- 如果逻辑复杂，chain 很复杂

有没有直接可以启动 agent 服务，由client 去调用的方式？

# LangGraph

| 能力          | LangChain  | LangGraph    |
| ------------- | ---------- | ------------ |
| 状态管理      | ❌          | ✅            |
| 分支逻辑      | ⛔ 简单支持 | ✅ 原生支持   |
| 循环逻辑      | ⛔ 很难实现 | ✅ 原生支持   |
| 多 Agent 协作 | ⚠️ 不方便   | ✅ 非常方便   |
| 复杂系统构建  | ⚠️ 容易乱   | ✅ 结构清晰   |
| 可视化        | 🚫          | ✅ 支持流程图 |

## 演示

```python
from typing import Annotated

from typing_extensions import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages


class State(TypedDict):
    # Messages have the type "list". The `add_messages` function
    # in the annotation defines how this state key should be updated
    # (in this case, it appends messages to the list, rather than overwriting them)
    messages: Annotated[list, add_messages]


graph_builder = StateGraph(State)

from langchain.chat_models import init_chat_model

llm = init_chat_model(
    "qwen2.5:7b",
    model_provider="ollama",
    base_url="http://ollama-api.k8s-test.uc.host.dxy"
)

def chatbot(state: State):
    return {"messages": [llm.invoke(state["messages"])]}


# The first argument is the unique node name
# The second argument is the function or object that will be called whenever
# the node is used.
graph_builder.add_node("chatbot", chatbot)

graph_builder.add_edge(START, "chatbot")
graph_builder.add_edge("chatbot", END)
graph = graph_builder.compile()

from IPython.display import Image, display

try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    # This requires some extra dependencies and is optional
    pass
  
def stream_graph_updates(user_input: str):
    for event in graph.stream({"messages": [{"role": "user", "content": user_input}]}):
        for value in event.values():
            print("Assistant:", value["messages"][-1].content)


while True:
    try:
        user_input = input("User: ")
        if user_input.lower() in ["quit", "exit", "q"]:
            print("Goodbye!")
            break
        stream_graph_updates(user_input)
    except:
        # fallback if input() is not available
        user_input = "What do you know about LangGraph?"
        print("User: " + user_input)
        stream_graph_updates(user_input)
        break
```



### TypedDict

把类型提示添加至字典的特殊构造器。在运行时，它是纯 [`dict`](https://docs.python.org/zh-cn/3/library/stdtypes.html#dict)。

```python
from typing import TypedDict, Literal

class User(TypedDict):
    name: str
    age: int
    gender: Literal["男", "女"]
```

### add_message

在 LangGraph 里，一个节点可能会被多个边（多条路径）并发调用，最后合并成一个状态（State）。
 为了让某些字段在合并时 **不是直接覆盖，而是“追加”或“自定义聚合”**，就要用 `Annotated[...]` 指定合并逻辑。

```python
from typing import Annotated

from typing_extensions import TypedDict


def add(messages: list[str], new_messages: str) -> list[str]:
    messages.append(new_messages)
    return messages


def replace(messages: list[str], new_messages: str) -> list[str]:
    messages.clear()
    messages.append(new_messages)
    return messages


class State(TypedDict):
    # Messages have the type "list". The `add_messages` function
    # in the annotation defines how this state key should be updated
    # (in this case, it appends messages to the list, rather than overwriting them)
    messages: Annotated[list, add]


class StateManager:
    def __init__(self, state: State):
        self.state = state

    def invoke(self, message: str):
        # 从 State 类获取 messages 字段的注解
        annotation = State.__annotations__["messages"]
        f = annotation.__metadata__[0]
        f(self.state["messages"], message)


if __name__ == "__main__":
    state = State(messages=[])
    state_manager = StateManager(state)
    state_manager.invoke("Hello")
    state_manager.invoke("World")
    state_manager.invoke("ai")
    print(state)
```

## 顺序执行

```python
from langgraph.graph import START, StateGraph
from typing_extensions import TypedDict


class State(TypedDict):
    value_1: str
    value_2: int
    
def step_1(state: State):
    return {"value_1": "a"}


def step_2(state: State):
    current_value_1 = state["value_1"]
    return {"value_1": f"{current_value_1} b"}


def step_3(state: State):
    return {"value_2": 10}

graph_builder = StateGraph(State)

# Add nodes
graph_builder.add_node(step_1)
graph_builder.add_node(step_2)
graph_builder.add_node(step_3)

# Add edges
graph_builder.add_edge(START, "step_1")
graph_builder.add_edge("step_1", "step_2")
graph_builder.add_edge("step_2", "step_3")

graph = graph_builder.compile()

from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))

graph.invoke({"value_1": "c"})
```

## 并行执行

并行节点

```python
import operator
from typing import Annotated, Any

from typing_extensions import TypedDict

from langgraph.graph import StateGraph, START, END


class State(TypedDict):
    # The operator.add reducer fn makes this append-only
    aggregate: Annotated[list, operator.add]


def a(state: State):
    print(f'Adding "A" to {state["aggregate"]}')
    return {"aggregate": ["A"]}


def b(state: State):
    print(f'Adding "B" to {state["aggregate"]}')
    return {"aggregate": ["B"]}


def c(state: State):
    print(f'Adding "C" to {state["aggregate"]}')
    return {"aggregate": ["C"]}


def d(state: State):
    print(f'Adding "D" to {state["aggregate"]}')
    return {"aggregate": ["D"]}


builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)
builder.add_node(c)
builder.add_node(d)
builder.add_edge(START, "a")
builder.add_edge("a", "b")
builder.add_edge("a", "c")
builder.add_edge("b", "d")
builder.add_edge("c", "d")
builder.add_edge("d", END)
graph = builder.compile()

from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))

graph.invoke({"aggregate": []}, {"configurable": {"thread_id": "foo"}})
```

条件分支

如果您的扇出不是确定的，您可以直接使用 add_conditional_edges。

```python
import operator
from typing import Annotated, Sequence

from typing_extensions import TypedDict

from langgraph.graph import StateGraph, START, END


class State(TypedDict):
    aggregate: Annotated[list, operator.add]
    # Add a key to the state. We will set this key to determine
    # how we branch.
    which: str


def a(state: State):
    print(f'Adding "A" to {state["aggregate"]}')
    return {"aggregate": ["A"]}


def b(state: State):
    print(f'Adding "B" to {state["aggregate"]}')
    return {"aggregate": ["B"]}


def c(state: State):
    print(f'Adding "C" to {state["aggregate"]}')
    return {"aggregate": ["C"]}


def d(state: State):
    print(f'Adding "D" to {state["aggregate"]}')
    return {"aggregate": ["D"]}


def e(state: State):
    print(f'Adding "E" to {state["aggregate"]}')
    return {"aggregate": ["E"]}


builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)
builder.add_node(c)
builder.add_node(d)
builder.add_node(e)
builder.add_edge(START, "a")


def route_bc_or_cd(state: State) -> Sequence[str]:
    if state["which"] == "cd":
        return ["c", "d"]
    return ["b", "c"]


intermediates = ["b", "c", "d"]
builder.add_conditional_edges(
    "a",
    route_bc_or_cd,
    intermediates,
)
for node in intermediates:
    builder.add_edge(node, "e")

builder.add_edge("e", END)
graph = builder.compile()

from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))

graph.invoke({"aggregate": [], "which": "cd"})
```

## MapReduce

Map-reduce 操作对于高效的任务分解和并行处理至关重要。这种方法涉及将任务分解成更小的子任务，并行处理每个子任务，并汇总所有已完成子任务的结果。



给定用户的一个通用主题，生成一系列相关主题，为每个主题生成一个笑话，并从生成的笑话列表中选择最佳笑话。在这个设计模式中，一个节点可能生成一系列对象（例如，相关主题），我们希望将这些对象（例如，主题）应用于其他节点（例如，生成笑话）。然而，出现了两个主要挑战。

- 对象的数量（例如，主题）可能在布局图之前未知
- 输入状态到下游节点的应该不同（每个生成的对象一个）。

![image-20250416163553415](/Users/shenweixiao/Library/Application Support/typora-user-images/image-20250416163553415.png)



```python
import operator
from typing import Annotated
from typing_extensions import TypedDict

from langgraph.types import Send
from langgraph.graph import END, StateGraph, START

from pydantic import BaseModel, Field

# 提示模板定义（中文）
subjects_prompt = """生成 2 到 5 个与以下主题相关的示例，使用英文逗号分隔：{topic}"""
joke_prompt = """生成一个关于 {subject} 的笑话"""
best_joke_prompt = """以下是一些关于 {topic} 的笑话。请选择其中最好的一个！返回最好的笑话的 ID（从 0 开始计数）。

{jokes}"""

# 定义返回的 subjects 数据结构
class Subjects(BaseModel):
    subjects: list[str]

# 定义返回的 joke 数据结构
class Joke(BaseModel):
    joke: str

# 定义返回的 best_joke 数据结构，ID 从 0 开始
class BestJoke(BaseModel):
    id: int = Field(description="Index of the best joke, starting with 0", ge=0)

# 初始化 Claude 模型
from langchain.chat_models import init_chat_model

model = init_chat_model(
    "qwen2.5:7b",
    model_provider="ollama",
    base_url="http://ollama-api.k8s-test.uc.host.dxy"
)

# 定义主流程状态结构
class OverallState(TypedDict):
    topic: str  # 主题，由用户提供
    subjects: list  # 生成的子主题
    jokes: Annotated[list, operator.add]  # 合并所有子节点生成的笑话
    best_selected_joke: str  # 最佳笑话

# 定义 joke 节点使用的状态结构
class JokeState(TypedDict):
    subject: str  # 单个子主题

# 节点：根据用户提供的 topic 生成子主题列表
def generate_topics(state: OverallState):
    prompt = subjects_prompt.format(topic=state["topic"])
    response = model.with_structured_output(Subjects).invoke(prompt)
    return {"subjects": response.subjects}

# 节点：根据子主题生成笑话
def generate_joke(state: JokeState):
    prompt = joke_prompt.format(subject=state["subject"])
    response = model.with_structured_output(Joke).invoke(prompt)
    return {"jokes": [response.joke]}  # 返回列表方便聚合

# 边：将多个 subject 分发到 joke 节点
def continue_to_jokes(state: OverallState):
    return [Send("generate_joke", {"subject": s}) for s in state["subjects"]]

# 节点：从多个笑话中选出最好的一个
def best_joke(state: OverallState):
    jokes = "\n\n".join(state["jokes"])  # 拼接所有笑话
    prompt = best_joke_prompt.format(topic=state["topic"], jokes=jokes)
    response = model.with_structured_output(BestJoke).invoke(prompt)
    return {"best_selected_joke": state["jokes"][response.id]}  # 取出最优笑话

# 构建流程图
graph = StateGraph(OverallState)

graph.add_node("generate_topics", generate_topics)  # 添加生成子主题节点
graph.add_node("generate_joke", generate_joke)  # 添加生成笑话节点
graph.add_node("best_joke", best_joke)  # 添加评选笑话节点

graph.add_edge(START, "generate_topics")  # 起点 -> 生成子主题
graph.add_conditional_edges("generate_topics", continue_to_jokes, ["generate_joke"])  # 子主题 -> 并发生成笑话
graph.add_edge("generate_joke", "best_joke")  # 所有笑话生成完 -> 评选

graph.add_edge("best_joke", END)  # 最终结束

# 编译为 LangGraph App
app = graph.compile()


from IPython.display import Image, display

display(Image(app.get_graph().draw_mermaid_png()))

# Call the graph: here we call it to generate a list of jokes
for s in app.stream({"topic": "animals"}):
    print(s)

print("DONE")
```

# 部署agent-server

uv + langgraph-cli

目录结构, 严格一直，否则无法识别

```
.
├── Dockerfile
├── README.md
├── docker-compose.yml
├── langgraph.json
├── main.py
├── my_agent
│   ├── __init__.py
│   ├── agent.py
│   └── utils
│       ├── __init__.py
│       ├── es.py
│       ├── prompts.py
│       ├── state.py
│       └── utils.py
├── pyproject.toml
└── uv.lock

```

## langgraph.json

```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./my_agent/agent.py:graph"
  },
  "env": ".env"
}
```

## pyproject.toml

```toml
[project]
name = "friday-ai"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
requires-python = ">=3.11"
dependencies = [
    "elasticsearch>=8.17.2",
    "langchain>=0.3.22",
    "langchain-ollama>=0.3.0",
    "langgraph==0.3.21",
    "transformers>=4.50.3",
]

```

## .env

编辑您的.env文件



## dockerfile

执行以下命令生成 Dockerfile

```bash
langgraph dockerfile Dockerfile 
```

# 部署agent-client

负责提供接口对外提供服务

```python
import json

from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from fastapi.middleware.cors import CORSMiddleware

from langgraph_sdk import get_client

app = FastAPI()


# If you're using a remote server, initialize the client with `get_client(url=REMOTE_URL)`
client = get_client(url="http://0.0.0.0:8123")


@app.get("/api/ai/log/diagnosis/{appname}")
async def get_diagnosis(appname: str):
    # List all assistants
    assistants = await client.assistants.search()
    # We auto-create an assistant for each graph you register in config.
    agent = assistants[0]

    thread = await client.threads.create()

    async def run_stream():
        input_data = {"appname": appname}
        async for chunk in client.runs.stream(thread['thread_id'], agent['assistant_id'],
                                              input=input_data,
                                              stream_mode="values"):
            if chunk.data:
                yield f"""data: {json.dumps(chunk.data)}\n\n"""

    return StreamingResponse(run_stream(), media_type="text/event-stream")

```

``` bash
 uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```



## NGINX 转发

```python
  location /api/ai/ {
          proxy_pass http://0.0.0.0:8000;
          proxy_set_header Host $host;
          proxy_buffering off;
          proxy_cache off;
  }
```

