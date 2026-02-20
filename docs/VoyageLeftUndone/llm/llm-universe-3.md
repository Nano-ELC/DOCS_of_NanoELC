---
title: LLM Universe 3
---

# 第四章：构建 RAG 应用 

### 1. 将 LLM 接入 LangChain 

这一节的核心在于利用 LangChain 框架将各大语言模型（LLM）抽象并接入我们的应用中，为后续的 RAG 链路提供处理核心。

#### 1.1 核心概念与 LCEL 语法

- **模型实例化**：LangChain 为基于 LLM 开发自定义应用提供了高效的开发框架。开发者可以在实例化时传入如 `temperature` 等超参数来控制回答的随机性与严谨度。
- **PromptTemplates (提示词模板)**：在开发大模型应用时，通常不会直接将用户的输入传递给 LLM，而是将其添加到一个包含特定任务上下文的提示模板中。例如，`ChatPromptTemplate` 可以通过定义角色（如 system, human）来生成供聊天模型使用的消息列表。
- **OutputParsers (输出解析器)**：将 LLM 生成的原始复杂对象转换为下游可以方便使用的格式。最常用的是 `StrOutputParser`，它能将消息直接转换为纯字符串。
- **LCEL (LangChain 表达式语言)**：这是一种允许我们使用 `|`（管道符）将不同组件串联成链的新语法。  * 它使得数据处理像 Unix 管道一样，将一个组件的输出作为下一个组件的输入。
  - LCEL 提供了异步、批处理、流处理支持，增加了 LLM 的并行性，并内置了日志记录以便于调试。

#### 1.2 模型接入 

LangChain 允许开发者通过统一的接口接入各种商业或开源大模型，方便“热插拔”：

- **智谱 GLM**：由于 LangChain 官方早期内置支持有限，可通过继承基类自定义 `ZhipuaiLLM` 包装器来完成适配接入。

------

### 2. 构建检索问答链 (RAG) - 核心处理流

**RAG 的本质**：就是给大模型“开卷考试”。提问 -> 查本地数据库 -> 组装上下文（类似于拼接字符串） -> 提交大模型阅读 -> 解析输出结果。

#### 2.1 加载向量数据库 

- **核心概念**：持久化加载与检索器转化 (`as_retriever`)。
- **逻辑类比**：大模型由于只靠自己背诵的知识，遇到新知识容易产生“幻觉”。这一步就是加载我们本地的资料库，并构建一个可以通过“相似度”搜索前 K 个相关文档片段的检索工具（Retriever）。

```python
import sys
sys.path.append("../C3 搭建知识库") 
from zhipuai_embedding import ZhipuAIEmbeddings
from langchain.vectorstores.chroma import Chroma
import os

# 1. 实例化 Embedding 模型（将文本转为向量坐标）
embedding = ZhipuAIEmbeddings()

# 2. 指定硬盘上 Chroma 数据库保存的相对路径
persist_directory = '../../data_base/vector_db/chroma'

# 3. 加载现有的数据库
vectordb = Chroma(persist_directory=persist_directory, embedding_function=embedding)

# 4. 将向量数据库转化为检索器对象，设置每次检索返回最相似的 3 个文档片段
retriever = vectordb.as_retriever(search_kwargs={"k": 3})
```

#### 2.2 创建检索链 

- **核心概念**：LCEL (LangChain 表达式语言) 与 流水线 (`|`)。
- **逻辑类比**：大模型需要的是一段“纯文本”，但 retriever 返回的是一个文档对象数组。我们需要写一个组合器（Combiner），提取数组中的文本并拼接。最后用 `|` 管道符将它们连起来。

```python
from langchain_core.runnables import RunnableLambda

# 1. 组合器：提取每个文档对象的 page_content，并用换行符拼接
def combine_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

combiner = RunnableLambda(combine_docs)

# 2. 使用 LCEL 管道语法构建基础检索链
# 数据流向：用户问题(Query) -> retriever(查出文档数组) -> combiner(拼接成大段文本)
retrieval_chain = retriever | combiner
```

#### 2.3 创建 LLM 

- **核心概念**：大语言模型实例化。

```python
llm = ZhipuaiLLM(model_name="glm-4-plus", temperature=0.1)
```

#### 2.4 构建检索问答链 ( RunnableParallel 与 Passthrough)

- **核心概念**：多路参数传递 (`RunnableParallel` 与 `RunnablePassthrough`)。
- **逻辑类比**：在 C 语言中，如果你要通过 `sprintf(buffer, "资料：%s，问题：%s", docs, query)` 来拼接字符串，你必须准备好 docs 和 query 两个变量。
  - `RunnablePassthrough`：可以理解为“原样透传”，就像在 C 里做了一次简单的指针赋值保留原始问题（Query）。
  - `RunnableParallel`：并行执行。一边去数据库查资料，一边保留原问题，最后打包成一个包含两个变量的结构体（字典）传给下一个环节（Prompt 模板）。

```Python
from langchain_core.prompts import PromptTemplate
from langchain_core.runnables import RunnablePassthrough, RunnableParallel
from langchain_core.output_parsers import StrOutputParser

# 1. 定义提示词模板（相当于 C 语言中带有 %s 占位符的格式化字符串）
template = """使用以下上下文来回答最后的问题。...
{context}
问题: {input}
"""
prompt = PromptTemplate(template=template)

# 2. 构建核心问答链
qa_chain = (
    # RunnableParallel 兵分两路准备参数：
    # 第一路：把输入交给 retrieval_chain 查数据库，结果赋给 "context"
    # 第二路：RunnablePassthrough() 原封不动透传输入，赋给 "input"
    RunnableParallel({"context": retrieval_chain, "input": RunnablePassthrough()})
    
    # 此时数据变成字典：{"context": "资料...", "input": "问题..."}
    # 交给 prompt 执行类似 sprintf 的填空操作
    | prompt
    # 提交给 LLM 思考生成
    | llm
    # 解析 LLM 返回的复杂对象，只提取出人类可读的字符串
    | StrOutputParser()
)
```

#### 2.5 向检索链添加聊天记录 

- **核心概念**：`ChatPromptTemplate`。
- **逻辑类比**：大模型底层是无记忆的。它们不记得你之前说过什么，所以每次发起请求，必须把整个历史对话数组一起传给它。

```Python
from langchain_core.prompts import ChatPromptTemplate

system_prompt = "你是一个问答任务的助手。...\n\n{context}"

# 构建支持多轮对话的模板
qa_prompt = ChatPromptTemplate(
    [
        ("system", system_prompt),
        # 使用 placeholder 专门给历史聊天记录数组留出位置
        ("placeholder", "{chat_history}"),
        ("human", "{input}"),
    ]
)
```

#### 2.6 带有信息压缩的检索链

- **核心概念**：信息压缩 (Condense Question) 与 智能路由 (`RunnableBranch`)。

  在实际对话中，如果你问“请介绍下周志华”，接着问“他是谁？”。如果直接拿“他是谁”去数据库搜，系统根本不知道“他”是谁，什么都搜不到。

- **解决方案**：在检索数据库之前，加一个“预处理”管道。偷偷调用一次 LLM，让它结合历史记录，把“他是谁”压缩/转换为“周志华是谁？”，拿着这句完整的话再去查数据库。

```Python
from langchain_core.runnables import RunnableBranch

# --- 第一部分：信息压缩与路由道岔 ---
# 1. 指示大模型执行“代词消解”和“问题完善”
condense_question_prompt = ChatPromptTemplate([
        ("system", "请根据聊天记录完善用户最新的问题..."),
        ("placeholder", "{chat_history}"),
        ("human", "{input}"),
    ])

# 2. 设置道岔：根据数据结构中是否存在 chat_history 决定走向
retrieve_docs = RunnableBranch(
    # 分支 1：如果没有历史记录，直接拿着原始 input 扔给 retriever 去搜库
    (lambda x: not x.get("chat_history", False), (lambda x: x["input"]) | retriever, ),
    
    # 分支 2：如果有历史，先过改写模板 -> LLM 改写出无代词的新问题 -> 转成字符串 -> 拿新问题去搜库
    condense_question_prompt | llm | StrOutputParser() | retriever,
)

# --- 第二部分：组装终极多轮问答链 ---
def combine_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs["context"]) 

# 终极总装：带有记忆、会改写问题、能搜库、能回答的全能链条
qa_history_chain = RunnablePassthrough.assign(
    # 步骤 1：调用上面的智能路由逻辑查库，并把查到的文档列表塞入 "context" 变量中
    context = (lambda x: x) | retrieve_docs 
    ).assign(
    # 步骤 2：拿着包含 input, chat_history 和 context 的完整数据包，交给问答链生成答案，存入 "answer" 变量
    answer= (RunnablePassthrough.assign(context=combine_docs) | qa_prompt | llm | StrOutputParser() )
)
```

------

### 3 部署实战与工程化避坑指南

####  避坑 1：云端 SQLite 版本冲突 Hack

- **核心痛点**：Streamlit Cloud 底层自带的 SQLite 版本过低，会导致 Chroma 向量数据库运行崩溃。
- **解决逻辑**：在 `requirements.txt` 中添加 `pysqlite3-binary`，并在主程序**最顶端**强行替换系统模块。

```python
# 引入 pysqlite3 并替换系统的 sqlite3 模块
__import__('pysqlite3')
import sys
sys.modules['sqlite3'] = sys.modules.pop('pysqlite3')
```

####  避坑 2：密钥安全 (Secrets 隔离)与自动寻址

- **核心痛点**：将包含 API Key 的 `.env` 文件推送到 GitHub 会导致极大的密钥泄露风险。
- **解决逻辑**：本地环境使用 `dotenv` 加载；云端绝不上传 `.env`，改为在 Streamlit Cloud 的 Advanced settings -> Secrets 中手动配置环境变量。代码需兼容这两种读取方式。

```python
# 使用 dotenv 库安全加载环境变量的代码
import os
from dotenv import load_dotenv, find_dotenv

# find_dotenv() 自动寻找 .env 文件；在云端无 .env 时自动失效并默认读取系统/Secrets环境变量
_ = load_dotenv(find_dotenv())
```