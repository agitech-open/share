# 知识增强与大模型调用流程说明

---

## 一、知识增强（Knowledge Augmentation）流程

知识增强的目标是：**在用户提问时，自动检索与问题相关的知识（如用户文档分块L0、聚类L1、传记等），将其拼接进Prompt，提升大模型的上下文理解和个性化能力。**

### 1. 触发条件
- 用户请求的 `metadata` 中包含 `enable_l0_retrieval` 或 `enable_l1_retrieval` 等标志。
- 例如：
  ```json
  {
    "messages": [...],
    "metadata": {
      "enable_l0_retrieval": true,
      "enable_l1_retrieval": false
    }
  }
  ```

### 2. 检索相关知识
- 检索L0：如分块后的文档内容、嵌入、摘要等。
- 检索L1：如主题聚类、人物传记、特征等。
- 检索方式通常为向量检索（embedding相似度）、关键词匹配等。

**关键代码片段**（见 `api/domains/kernel2/services/prompt_builder.py`）：

```python
class KnowledgeEnhancedStrategy(SystemPromptStrategy):
    def build_prompt(self, request, context=None):
        base_prompt = self.base_strategy.build_prompt(request, context)
        # 检查metadata
        enable_l0_retrieval = request.metadata.get('enable_l0_retrieval', False)
        enable_l1_retrieval = request.metadata.get('enable_l1_retrieval', False)
        knowledge_sections = []
        if enable_l0_retrieval:
            l0_knowledge = retrieve_l0_knowledge(request)
            knowledge_sections.append(l0_knowledge)
        if enable_l1_retrieval:
            l1_knowledge = retrieve_l1_knowledge(request)
            knowledge_sections.append(l1_knowledge)
        # 拼接知识到prompt
        return base_prompt + "\n".join(knowledge_sections)
```

- `retrieve_l0_knowledge`/`retrieve_l1_knowledge` 通常会调用向量数据库或本地索引，返回与当前问题最相关的内容片段。

### 3. 拼接到Prompt
- 检索到的知识以 `system` 或 `user` 角色插入到 `messages` 列表中，作为大模型的上下文输入。
- 例如：

```python
messages = [
    {"role": "system", "content": "你是我的个性化AI助手..."},
    {"role": "system", "content": "相关知识片段：..."},
    {"role": "user", "content": "我最近在学Python，有什么高效的学习建议吗？"}
]
```

---

## 二、调用大模型（LLM调用）流程

### 1. 构建标准消息格式
- 按照OpenAI等主流大模型API的格式，构建 `messages` 列表。
- 结构示例：

```python
messages = [
    {"role": "system", "content": "你是我的个性化AI助手..."},
    {"role": "system", "content": "相关知识片段..."},
    {"role": "user", "content": "用户问题..."}
]
```

### 2. 调用大模型API
- 通过统一的LLM客户端（如 `common/llm.py`、`L0/l0_generator.py`），将 `messages` 发送给大模型服务。
- 支持OpenAI、Qwen、DeepSeek等主流API。

**关键代码片段**（见 `common/llm.py`）：

```python
response = self.client.chat.completions.create(
    model=self.model_name,
    messages=messages,
    max_tokens=4096,
    temperature=0.7,
)
reply = response.choices[0].message.content
```

- `self.client` 是大模型API的客户端实例（如OpenAI、Qwen等）。
- `model_name` 指定具体模型。
- `messages` 为拼接好的上下文消息。

### 3. 后处理与返回
- 对模型返回的内容做格式化、结构化、过滤等处理。
- 最终通过API返回给前端/用户。

---

## 三、流程示意图

```
用户提问
   ↓
API路由接收
   ↓
知识增强（检索L0/L1相关内容）
   ↓
拼接Prompt（messages）
   ↓
调用大模型API
   ↓
获取回复
   ↓
后处理
   ↓
返回用户
```

---

## 四、示例

**请求示例：**

```json
{
  "messages": [
    {"role": "system", "content": "你是我的个性化AI助手..."},
    {"role": "user", "content": "我最近在学Python，有什么高效的学习建议吗？"}
  ],
  "metadata": {
    "enable_l0_retrieval": true
  }
}
```

**拼接后Prompt：**

```python
messages = [
    {"role": "system", "content": "你是我的个性化AI助手..."},
    {"role": "system", "content": "相关知识片段：你最近阅读了《Python入门》..."},
    {"role": "user", "content": "我最近在学Python，有什么高效的学习建议吗？"}
]
```

**大模型回复：**

```json
{
  "reply": "学习Python建议多做项目实践，结合官方文档和社区资源，遇到问题及时查阅Stack Overflow..."
}
```

---

## 五、总结

- **知识增强**：自动检索与问题相关的知识，拼接进Prompt，提升个性化和准确性。
- **大模型调用**：标准OpenAI格式，支持多种大模型，统一接口调用。
- **关键代码**：见 `KnowledgeEnhancedStrategy`、`llm.py`、`l0_generator.py` 等。

如需具体检索算法、Prompt模板或API调用细节的代码，可进一步展开！

---

## 六、向量数据的存储字段及含义

在知识增强检索中，向量数据（embedding及其相关信息）主要存储在 ChromaDB 的 collection 中。每一条向量数据包含以下主要字段：

| 字段名         | 含义与用途                                               |
|----------------|--------------------------------------------------------|
| id             | 唯一标识，主键（如分块ID、文档ID）                     |
| documents/text | 原始文本内容（如分块内容、文档内容）                   |
| embeddings     | 向量表示（float数组），用于相似度检索                   |
| document_id    | 所属文档ID，便于分块与文档关联                         |
| title          | 标题                                                   |
| mime_type      | 文档类型                                               |
| create_time    | 创建时间                                               |
| document_size  | 文档大小                                               |
| url            | 文档存储路径/下载链接                                  |
| topic          | 主题标签                                               |
| tags           | 关键词、标签                                           |
| ...            | 其他自定义元数据字段                                   |

### 字段说明
- **id**：唯一标识该向量数据的ID（如分块ID、文档ID），用于检索、更新、删除等操作的主键。
- **documents/text**：原始文本内容，便于后续展示、调试、人工校验等。
- **embeddings**：文本内容对应的向量（embedding），通常为float数组（如1536维），用于向量相似度检索。
- **metadatas**：与该向量数据相关的元信息（字典结构），如document_id、title、mime_type、create_time、document_size、url、topic、tags等，便于多维度检索、过滤、业务扩展。

### 代码示例

```python
self.document_collection.add(
    documents=[document.raw_content],
    ids=[str(document.id)],
    embeddings=[embedding.tolist()],
    metadatas=[{
        "title": document.title or document.name,
        "mime_type": document.mime_type,
        "create_time": document.create_time.isoformat() if document.create_time else None,
        "document_size": document.document_size,
        "url": document.url,
    }]
)

self.chunk_collection.add(
    documents=[c.content for c in unprocessed_chunks],
    ids=[str(c.id) for c in unprocessed_chunks],
    embeddings=embeddings.tolist(),
    metadatas=[
        {
            "document_id": str(c.document_id),
            "topic": c.topic or "",
            "tags": ",".join(c.tags) if c.tags else "",
        }
        for c in unprocessed_chunks
    ],
)
```

如需查看ChromaDB实际存储结构或某一字段的具体用法，可进一步展开！

---

## 七、embeddings 字段详解

### 1. 字段内容
- `embeddings` 字段存储的是一组浮点数（float数组），即文本内容的向量化表示（embedding）。
- 每个 embedding 是一个一维向量（如 `[0.123, -0.456, ...]`），用于表示文本的语义特征。
- 维度通常为 768、1024、1536、3072 等，取决于所用 embedding 模型。

### 2. 生成方式
1. **文本内容准备**：提取每个文档、分块或知识片段的原始文本。
2. **调用大模型/Embedding服务**：
   - 使用大模型（如 OpenAI、Qwen、DeepSeek 等）的 embedding API，将文本转化为向量。
   - 相关代码见 `lpm_kernel/common/llm.py`、`file_data/embedding_service.py`。
   - 例如：
     ```python
     embeddings = self.llm_client.get_embedding([document.raw_content])
     # embeddings 结果为 [[0.123, -0.456, ...]]
     ```
3. **存入向量数据库**：
   - 生成的 embedding 向量与原始文本、元数据一起，存入 ChromaDB 的 collection。
   - 见 `self.document_collection.add(..., embeddings=[embedding.tolist()], ...)`

### 3. 用途
- 用于向量相似度检索（如余弦相似度、欧氏距离等），以便查找与用户问题最相关的知识片段。
- 是知识增强检索的核心数据。

### 4. 生成流程示意
1. 原始文本：如“Python 是一种流行的编程语言。”
2. 调用 embedding API：如 OpenAI 的 `/v1/embeddings`，返回 1536 维向量。
3. 存储：将该向量与文本、ID、元数据一起存入 ChromaDB。

### 5. 代码示例
```python
# 生成 embedding
embeddings = self.llm_client.get_embedding([chunk.content])
# 存储到 ChromaDB
self.chunk_collection.add(
    documents=[chunk.content],
    ids=[str(chunk.id)],
    embeddings=[embeddings[0].tolist()],
    metadatas=[...]
)
```

### 6. 相关配置
- embedding 维度、模型名称等可通过环境变量或配置文件设置（如 `embedding_model_name`）。
- 支持多种 embedding 服务，自动适配当前用户配置。

### 7. 小结
- `embeddings` 字段存储的是文本的高维向量表示，是通过大模型的 embedding API 生成的。
- 该向量用于高效的相似度检索，是知识增强的基础。

如需具体 embedding 生成的 API 调用细节或支持的模型列表，可进一步展开！
