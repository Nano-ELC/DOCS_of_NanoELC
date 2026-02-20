---
title: LLM Universe 2
---
# 第三章：搭建向量知识库

### 1. 向量及向量知识库

#### 1.1 词向量与向量

* **词向量 (Word Embedding)**：将单词转化为实数向量的技术，使计算机能理解语义。
  * **核心逻辑**：语义相似的对象在向量空间中距离更近（如 "King" 与 "Queen"）。
* **RAG 中的应用**：使用**通用文本向量 (Universal Text Embedding)**，将输入单位从单词变为文本片段，以捕捉更丰富的上下文语义。
* **向量的优势**：
1. **检索能力**：相比关键词匹配（词法搜索），向量通过计算语义相似度（如余弦距离），能找到字面不同但意思相近的内容。
2. **跨模态能力**：能将文字、图像、声音映射到统一的向量形式，实现多模态关联。

#### 1.2 向量数据库

* **概念**：专门用于存储和检索向量数据（Embedding）的数据库系统。
* **逻辑关系**：向量数据库使用专门的索引和查询算法（不同于传统关系型数据库），旨在处理海量向量数据时，高效计算目标向量与库中向量的相似度（如余弦距离、点积）。
* **常见工具**：Chroma（轻量级，适合入门）、Weaviate（支持混合搜索）、Qdrant（高效率）。

### 2. 使用 Embedding API (以智谱 AI 为例)

#### 2.1 调用逻辑

调用 Embedding API 的过程通常包括：配置 API Key -> 初始化客户端 -> 指定模型与输入文本 -> 获取返回结果（包含向量数据、Token 消耗等信息）。

#### 2.2 代码解析

```python
from zhipuai import ZhipuAI

def zhipu_embedding(text: str):
    # 1. 获取环境变量中的 API Key，这是调用服务的凭证
    api_key = os.environ['ZHIPUAI_API_KEY']
    
    # 2. 初始化 ZhipuAI 客户端实例
    client = ZhipuAI(api_key=api_key)
    
    # 3. 调用 embeddings.create 方法发起请求
    response = client.embeddings.create(
        model="embedding-3",  # 4. 指定使用的 Embedding 模型版本
        input=text,           # 5. 传入需要向量化的文本内容
    )
    
    # 6. 返回 API 的响应对象，其中包含了生成的向量数据
    return response

# 示例调用
text = '要生成 embedding 的输入文本，字符串形式。'
response = zhipu_embedding(text=text)

```

### 3. 数据处理

构建知识库前，必须对原始文档进行读取、清洗和分割，这是决定检索效果下限的关键步骤。

#### 3.1 数据读取

* **PDF 文档**：使用 LangChain 的 `PyMuPDFLoader`。它速度快，能提取详细元数据，每页作为一个 Document 对象。
* **Markdown 文档**：使用 `UnstructuredMarkdownLoader`。
* **读取结果结构**：返回一个列表，元素包含 `page_content`（内容）和 `metadata`（元数据）。

**[练习] 代码实现：读取 PDF 和 Markdown 文件**

```python
# 导入 Loader，加载文件
from langchain_community.document_loaders import PyMuPDFLoader
loader = PyMuPDFLoader("文件路径")
pdf_pages = loader.load()
# 查看加载后的数据结构
print(f"载入后的变量类型为:{type(pdf_pages)},",f"该PDF一共包含{len(pdf_pages)}页")
# 载入后的变量类型为:<class list>,该PDF一共包含X页
# pdf_pages作为一个列表，其中的每一个元素，都有以下特性
pdf_page = pdf_pages[1]
print(f"类型:{type(pdf_page)}",f"描述性数据:{pdf_page.metadata}",f"具体内容{pdf_page.page_content}")

```

#### 3.2 数据清洗

* **目的**：去除影响语义理解的噪声，如多余的换行符 `\n`、特殊的列表符号（如 `•`）、无意义的空格等。
* **方法**：通常使用正则表达式 (`re` 模块) 或字符串的 `replace` 方法。

**[练习] 代码实现：使用正则或 replace 清洗文本中的换行符和特殊符号**

```python
# 编写清洗规则，去除 \n 和干扰符号
import re
pattern = re.compile(r'[^\u4e00-\u9fff](\n)[^\u4e00-\u9fff]',re_DOTALL)
pdf_page.page_content = re.sub(pattern, lambda match: match.group(0).replace('\n',''),pdf_page.page_content)
```

#### 3.3 文档分割 (Chunking)

* **必要性**：单个文档通常超过 LLM 的上下文窗口限制，且过长的文本包含杂乱信息，不利于精准检索。
* **核心参数**：
* `chunk_size`：每个切片包含的字符或 Token 数。
* `chunk_overlap`：切片之间的重叠长度，用于保持上下文连贯性，防止语义被切断。


* **常用工具**：`RecursiveCharacterTextSplitter`（递归字符分割器），优先按段落、句子分隔符切割，尽可能保持语义完整。

**[练习] 代码实现：配置分割器并对文档进行切分**

```python
# 导入分割器，设置 chunk_size 和 chunk_overlap，并执行分割
from langchain_text_splitters import RecursiveCharacterTextSplitter
# 知识库中单段文本长度
CHUNK_SIZE = 500

# 知识库中相邻文本重合长度
OVERLAP_SIZE = 50

text_splitter = RecursiveCharacterTextSplitter(
	chunk_size=CHUNK_SIZE,
	chunk_overlap=OVERLAP_SIZE
)
text_splitter.split_text(pdf_page.page_content[0:1000])

split_docs = text_splitter.split_documents(pdf_pages)
print(f"切分后的文件数量：{len(split_docs)}")
print(f"切分后的字符数（可以用来大致评估 token 数）：{sum([len(doc.page_content) for doc in split_docs])}")

```

------

### 4. 搭建并使用向量数据库

在完成数据的读取、清洗和分割后，接下来的核心步骤是将这些文本切片（Chunks）通过 Embedding 模型转化为向量，并存储到专门的向量数据库中，以供随时检索调用。

#### 4.1 构建 Chroma 向量库

- **核心逻辑**：借助 LangChain 框架的集成能力，我们可以将分割好的文档切片集合、选定的 Embedding 模型（如智谱 AI、OpenAI 等）与轻量级的内存级数据库 Chroma 绑定，并设定目录将其持久化保存到本地磁盘中。

**[练习] 代码实现：初始化 Embedding 模型并构建本地 Chroma 数据库**

在这个练习中，核心目标是定义 Embedding 模型，设置数据库保存的路径，并将之前切分好的文档数据持久化保存到磁盘上。这里我们默认使用教程推荐的智谱 Embedding 模型。

```python
# 1. 导入必要的库
from zhipuai_embedding import ZhipuAIEmbeddings
from langchain_community.vectorstores import Chroma

# 2. 定义 Embeddings
embedding = ZhipuAIEmbeddings()

# 3. 定义向量数据库的持久化路径
persist_directory = '../../data_base/vector_db/chroma'

# 4. 构建并持久化 Chroma 向量库
# 注意：这里假设在前面的代码中，我们已经成功将文档切分并赋值给了 split_docs 变量
vectordb = Chroma.from_documents(
    documents=split_docs,
    embedding=embedding,
    persist_directory=persist_directory  # 允许我们将 persist_directory 目录保存到磁盘上
)

# 5. 验证是否构建成功 (打印存储的 chunk 数量)
print(f"向量库中存储的数量：{vectordb._collection.count()}")
```

#### 4.2 向量检索 (Vector Retrieval)

当知识库搭建完毕后，面对用户的提问，系统会同样将问题转化为词向量，在数据库中寻找匹配的文档片段。常用的检索策略主要有两种：

- **相似度搜索 (Similarity Search)**：

  - **逻辑**：这是最基础的检索方式，系统会计算问题向量与数据库中各文本段向量的相似度，返回最匹配的 Top-k 个结果。Chroma 默认使用余弦距离来衡量相似度。   

     **余弦距离公式**：

    $$
    similarity = cos(A, B) = \frac{A \cdot B}{||A|| || B ||} = \frac{\sum_1^n a_i b_i}{\sqrt{\sum_1^n a_i^2}\sqrt{\sum_1^n b_i^2}}
    $$

- **最大边际相关性搜索 (MMR, Maximum Marginal Relevance)**：

  - **逻辑**：单纯按相似度检索容易导致找出的多段文本内容高度重复单一、丢失重要信息。MMR 的核心思想是在选出一个相关性高的文档后，下一个选择的文档要在“与问题相关”和“与已选文档不同（增加信息丰富度）”之间寻找平衡点。

    **[练习] 代码实现：使用不同策略进行知识库检索**

```python
# 设定一个 question 变量，分别调用 vectordb.similarity_search() 和 vectordb.max_marginal_relevance_search()，观察并对比返回结果的多样性差异
# 1. 设定一个测试问题
question = "什么是大语言模型"

# 策略 A：相似度搜索 (Similarity Search)
print("【策略 A：相似度搜索结果】")
# k=3 表示返回 3 个最相关的文档片段
sim_docs = vectordb.similarity_search(question, k=3)
print(f"检索到的内容数：{len(sim_docs)}\n")

for i, sim_doc in enumerate(sim_docs):
    print(f"检索到的第{i}个内容: \n{sim_doc.page_content[:200]}", end="\n--------------\n")


# 策略 B：最大边际相关性搜索 (MMR)

print("\n【策略 B：最大边际相关性 (MMR) 搜索结果】")
# 同样返回 3 个结果，但使用 MMR 策略
mmr_docs = vectordb.max_marginal_relevance_search(question, k=3)

for i, mmr_doc in enumerate(mmr_docs):
    print(f"MMR 检索到的第{i}个内容: \n{mmr_doc.page_content[:200]}", end="\n--------------\n")
```
