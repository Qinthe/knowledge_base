# LangGraph 说明

## 1. 什么是 LangGraph？

LangGraph 是 LangChain 团队推出的**图编排框架**，用于构建**有状态、多步骤、可循环**的 LLM 应用，尤其是 Agent。

要理解它，先看普通 Chain 的局限：

```text
普通 Chain：提示词 → 模型 → 输出   （一条直线，只能走一遍）
```

真实 Agent 往往需要：

```text
模型思考 → 需要调用工具? → 是 → 调用工具 → 回到模型再思考
                         → 否 → 输出最终答案
```

这里有**循环**和**分支**，普通 Chain 表达不了，LangGraph 用"图"来解决。

> 通俗地说：LangChain 的 Chain 是"一条直线流水线"，LangGraph 是"带红绿灯和岔路的交通网"。

## 2. 核心概念

### 2.1 State（状态）

在节点之间传递的共享数据，是图的"记忆"。用 `TypedDict` 定义，每次经过节点都会被更新。

```python
from typing import TypedDict

class State(TypedDict):
    messages: list   # 对话历史
    step: int        # 当前步数
```

### 2.2 Node（节点）

图里的一个处理步骤，本质是一个**函数**：接收 State，返回要更新的字段。

```python
def my_node(state: State):
    # 处理逻辑...
    return {"step": state["step"] + 1}   # 只更新 step 字段
```

### 2.3 Edge（边）

节点之间的连接关系，决定执行顺序：

- **普通边（add_edge）**：固定从 A 到 B。
- **条件边（add_conditional_edges）**：根据状态决定下一步走哪。

### 2.4 Graph（图）

节点 + 边的整体。建好后 `compile()` 得到一个可执行的"程序"。

## 3. LangGraph 的关键能力

| 能力 | 说明 | 典型场景 |
| :--- | :--- | :--- |
| 循环 | 图可以有环，节点能反复执行 | Agent 反复调用工具 |
| 状态持久化（Checkpoint） | 每一步的状态可保存/恢复 | 断点续跑、多轮对话 |
| 人工介入（Human-in-the-loop） | 关键步骤停下来等人工确认 | 审批、审核 |
| 流式 | 逐节点输出过程 | 让用户看到 Agent 的每一步 |

## 4. 什么时候用 LangGraph？

- 需要 **多步骤 + 循环** 的 Agent。
- 需要 **人工确认** 的关键流程。
- 需要 **失败重试、条件分支** 的复杂工作流。

简单的一问一答，直接用 LangChain 的 Chain 即可，不必上 LangGraph。
