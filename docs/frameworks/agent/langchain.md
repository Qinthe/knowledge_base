# LangChain 说明

## 1. 什么是 LangChain？

LangChain 是一个用于构建 **大语言模型（LLM）应用** 的开源框架，2022 年由 Harrison Chase 发起，由 LangChain 团队维护。

它解决的核心痛点：**直接调用大模型 API 很麻烦**。不同厂商（OpenAI、DeepSeek、通义千问、文心一言）的接口格式各不相同，而真实应用还需要提示词管理、多步骤编排、记忆、工具调用、知识库检索等一整套东西。

LangChain 的价值在于**统一接口 + 标准组件**：

- **统一接口**：换模型只改一行配置，其余代码不变。
- **标准组件**：把"选模型、写提示词、串流程、加记忆、查资料"打包成可复用的组件。

> 通俗地说：LangChain 是"LLM 应用的标准工具箱"，你自己搭应用时要什么零件，它都备好了。

## 2. LangChain 的生态与包结构

LangChain 按功能拆成了多个包，安装时按需选择：

| 包 | 作用 |
| :--- | :--- |
| `langchain-core` | 核心抽象：模型、提示词、输出解析器等基础接口 |
| `langchain` | 高层入口，组合各组件（链、Agent、工具） |
| `langchain-openai` | OpenAI 及 OpenAI 兼容接口（DeepSeek 也走这个包） |
| `langchain-community` | 社区集成：各种第三方工具、文档加载器 |

> 因为 DeepSeek 提供 OpenAI 兼容接口，所以用的是 `langchain-openai`，而不是专门的 `langchain-deepseek`。

## 3. 核心概念详解

### 3.1 Model（模型）

封装大模型调用，统一不同厂商的接口。两种常见类型：

- **LLM**：输入文本、输出文本（一次性完成）。
- **ChatModel**：面向对话，接收消息列表（system/user/assistant）、返回消息。现在绝大多数场景用 ChatModel。

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com/v1",
    api_key="...",
)
```

### 3.2 Prompt（提示词）

组织输入给模型的话术模板。`ChatPromptTemplate` 支持用 `{变量}` 占位，运行时再填入：

```python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}。"),
    ("user", "{question}"),
])
# 运行时：prompt.invoke({"role": "中文技术助手", "question": "什么是数据湖"})
```

### 3.3 Chain（链）

把多个步骤串成一条流水线。用 LCEL（LangChain Expression Language）的 `|` 运算符连接：

```python
chain = prompt | llm | output_parser
```

`|` 左边组件的输出，自动作为右边组件的输入，像管道一样。

### 3.4 LCEL（LangChain 表达式语言）

LCEL 是 LangChain 推荐的组合方式，核心运算符就是 `|`。它带来的好处：

- **异步支持**：同一个链可以 `.invoke()`（同步）、`.ainvoke()`（异步）、`.batch()`（批量）。
- **流式输出**：`.stream()` 逐字输出。
- **自动重试/回退**：可配置 `with_retry()`、`with_fallbacks()`。

### 3.5 Memory（记忆）

在多轮对话中记住上下文。做法是把历史消息拼进每次请求的 messages 里：

```python
messages = []                       # 对话历史
messages.append(("user", "我叫小明"))
messages.append(("assistant", "你好小明！"))
# 下次调用时把 messages 一起传给模型
```

> 进阶方案是用 LangGraph 管理多轮状态，比 LangChain 自带的老版 Memory 更清晰。

### 3.6 Retriever（检索）+ RAG

从文档/知识库检索相关内容，再喂给模型，这就是 **RAG（检索增强生成）**：

```text
问题 → 从知识库检索相关片段 → 拼进提示词 → 模型生成答案
```

好处是让模型能回答"训练数据里没有"的私有知识。

### 3.7 Agent（智能体）

让模型自主决定**调用哪些工具、调用几次**：

```text
用户提问 → 模型思考 → 需要查数据库? → 调用工具 → 再思考 → 给出最终答案
```

Agent 是 LangChain 的进阶用法，通常配合 LangGraph 实现。

## 4. 与 LangGraph 的关系

- **LangChain**：提供基础组件（模型、提示词、工具、检索器、链）。
- **LangGraph**：负责**编排**，构建有状态、多步骤、可循环、可分支的 Agent 工作流。

| 场景 | 推荐 |
| :--- | :--- |
| 简单一问一答、线性流程 | LangChain 的 Chain |
| 多步骤 Agent、循环、人工介入 | LangGraph |

> 一句话：LangChain 是"零件"，LangGraph 是"装配线"。
