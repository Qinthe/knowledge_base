# LangGraph 开始使用

从零跑通一个最小 StateGraph，再升级成带循环的示例。

## 1. 安装

```bash
pip install langgraph
```

> 如需在图中调用大模型，再装 `langchain-openai`（见 LangChain 开始使用）。

## 2. 最小示例：两个节点的流水线

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

# 1. 定义状态：节点间传递的数据结构
class State(TypedDict):
    messages: list

# 2. 定义节点：每个节点是一个函数，接收 state、返回要更新的字段
def node_a(state: State):
    return {"messages": state["messages"] + ["节点A"]}

def node_b(state: State):
    return {"messages": state["messages"] + ["节点B"]}

# 3. 建图：注册节点 + 连线
builder = StateGraph(State)
builder.add_node("a", node_a)
builder.add_node("b", node_b)
builder.add_edge(START, "a")   # 起点 → 节点A
builder.add_edge("a", "b")     # 节点A → 节点B
builder.add_edge("b", END)     # 节点B → 终点

graph = builder.compile()

# 4. 运行
result = graph.invoke({"messages": []})
print(result["messages"])
```

**运行结果**：

```text
['节点A', '节点B']
```

**逐行解释**：

- `StateGraph(State)`：创建一张以 `State` 为共享状态的图。
- `add_node("a", node_a)`：注册名为 `a` 的节点，绑到 `node_a` 函数。
- `add_edge(START, "a")`：从起点连到节点 a。
- `compile()`：编译成可执行图。
- `invoke({"messages": []})`：传入初始状态，运行整张图。

## 3. 带循环的示例：条件边

下面的图模拟了 Agent 的"循环直到满足条件"结构（不调用真实工具，只演示循环）：

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    count: int

def think(state: State):
    print(f"思考第 {state['count']} 次...")
    return {"count": state["count"] + 1}

def decide(state: State):
    # 模拟"是否还需要继续"
    return "continue" if state["count"] < 3 else "stop"

builder = StateGraph(State)
builder.add_node("think", think)
builder.add_edge(START, "think")
builder.add_conditional_edges(
    "think",
    decide,
    {"continue": "think", "stop": END},   # 根据 decide 的返回值选路径
)

graph = builder.compile()
result = graph.invoke({"count": 0})
print("最终:", result["count"])
```

**运行结果**：

```text
思考第 0 次...
思考第 1 次...
思考第 2 次...
最终: 3
```

**关键点**：`add_conditional_edges` 让图能根据状态**循环或退出**，这是 Agent 的核心机制。

## 4. 下一步

- 把 `think` 节点换成真正的 LLM 调用、`decide` 换成"模型是否请求调用工具"的判断，就是一个最小 Agent。
- 用 `checkpointer` 开启状态持久化，实现多轮对话。
- 官方文档：<https://langchain-ai.github.io/langgraph/>
