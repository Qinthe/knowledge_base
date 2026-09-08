# LangChain 开始使用

本教程以 **DeepSeek**（OpenAI 兼容接口）为例，从安装到跑通几个常见用法。

## 1. 环境准备

### 1.1 创建虚拟环境（推荐）

```powershell
# Windows PowerShell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

### 1.2 安装依赖

```bash
pip install langchain langchain-openai
```

> `langchain` 会自动带上 `langchain-core`；`langchain-openai` 用于调用 OpenAI 兼容接口（DeepSeek 就是这类）。

## 2. 配置 DeepSeek API Key

1. 到 [DeepSeek 开放平台](https://platform.deepseek.com) 注册并创建 API Key。
2. 关键参数：
   - **模型名**：`deepseek-chat`
   - **接口地址（base_url）**：`https://api.deepseek.com/v1`
3. 把 Key 存到环境变量，避免写死在代码里：

```powershell
# PowerShell（当前会话生效）
$env:DEEPSEEK_API_KEY = "sk-你的Key"
```

```python
import os

api_key = os.environ["DEEPSEEK_API_KEY"]
```

## 3. 用法一：直接调用模型

```python
import os
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com/v1",
    api_key=os.environ["DEEPSEEK_API_KEY"],
)

resp = llm.invoke("用一句话解释什么是数据湖")
print(resp.content)
```

**运行结果（示例）**：

```text
数据湖是一个以原始格式集中存储海量数据的仓库，允许在读取时再定义数据结构。
```

## 4. 用法二：提示词模板 + 模型（链）

```python
import os
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个乐于助人的中文技术助手，回答尽量通俗易懂。"),
    ("user", "{question}"),
])

llm = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com/v1",
    api_key=os.environ["DEEPSEEK_API_KEY"],
)

chain = prompt | llm   # 用 | 把模板和模型串成一条链

result = chain.invoke({"question": "什么是数据湖？"})
print(result.content)
```

## 5. 用法三：流式输出

适合聊天场景，逐字打印：

```python
for chunk in chain.stream({"question": "什么是数据湖？"}):
    print(chunk.content, end="", flush=True)
```

## 6. 用法四：多轮对话（手动维护历史）

```python
from langchain_core.messages import HumanMessage, SystemMessage

messages = [
    SystemMessage(content="你是一个中文助手。"),
    HumanMessage(content="我叫小明。"),
]

resp = llm.invoke(messages)
print(resp.content)

# 把回答追加进历史，再问下一轮
messages.append(resp)
messages.append(HumanMessage(content="我叫什么？"))
resp2 = llm.invoke(messages)
print(resp2.content)
```

## 7. 常见问题

| 问题 | 可能原因 | 处理 |
| :--- | :--- | :--- |
| 连接超时 / Connection error | 网络不通 | 确认能访问 api.deepseek.com，代理可用 |
| `401 Authentication Fails` | API Key 错误或未设置 | 检查 `DEEPSEEK_API_KEY` |
| `model not found` | 模型名写错 | DeepSeek 用 `deepseek-chat` |
| 中文问题答成英文 | system 提示词未指定语言 | 在 system 里写明"用中文回答" |
