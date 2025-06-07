# API 信息总览

本文件汇总了 `lpm_kernel` 项目中所有注册到 Flask app 的 API Blueprint，包括用途、主要路由前缀、典型接口和简要说明。

---

## 1. health_bp
- **用途**：健康检查和 favicon 提供。
- **主要前缀**：`/health`, `/favicon.ico`
- **典型接口**：
  - `GET /health`：服务健康检查。
  - `GET /favicon.ico`：网站图标。

---

## 2. document_bp
- **用途**：文档管理与分析。
- **主要前缀**：`/api/documents`
- **典型接口**：
  - `GET /documents/list`：列出所有文档。
  - `POST /documents/scan`：扫描目录下文档并入库。
  - `POST /documents/analyze`：分析所有未分析文档。
    
    **详细说明：**
    - **用途**：批量分析所有未分析的文档，为每个文档生成洞察（insight）、摘要（summary）等结构化信息，并写入数据库。适用于知识归纳、自动摘要、内容结构化等场景。
    - **请求方式**：`POST /api/documents/analyze`，无需请求体参数。
    - **处理流程**：
      1. 查询所有未分析的文档。
      2. 对每个文档调用分析方法，生成主题、关键词、摘要等。
      3. 更新数据库分析结果。
      4. 返回所有已分析文档的结构化信息。
    - **返回示例**：
      ```json
      {
        "code": 0,
        "message": "success",
        "data": {
          "total": 3,
          "documents": [
            {
              "id": 1,
              "name": "example.txt",
              "insight": { "topics": ["AI", "NLP"], "keywords": ["人工智能", "文本处理"] },
              "summary": "这是一篇关于AI和NLP的文档……",
              "analyze_status": "success"
            }
          ]
        }
      }
      ```
    - **字段说明**：
      - `id`：文档主键
      - `name`：文件名
      - `insight`：结构化洞察（如主题、关键词、标签等）
      - `summary`：自动生成的文档摘要
      - `analyze_status`：分析状态（success/failed）
    - **典型应用场景**：
      - 知识库自动归纳、批量生成主题/标签/摘要
      - 内容推荐、知识图谱构建等
  - `GET /documents/<document_id>/l0`：获取文档 L0 数据。
  - `GET /documents/<document_id>/chunks`：获取文档分块。

    - **实现与核心代码分析：**

      ```python
      # 路由入口
      @document_bp.route("/documents/analyze", methods=["POST"])
      def analyze_documents():
          try:
              analyzed_doc_dtos = document_service.analyze_all_documents()
              return jsonify(
                  APIResponse.success(
                      data={
                          "total": len(analyzed_doc_dtos),
                          "documents": [doc.dict() for doc in analyzed_doc_dtos],
                      }
                  )
              )
          except Exception as e:
              logger.error(f"Error analyzing documents: {str(e)}", exc_info=True)
              return jsonify(
                  APIResponse.error(message=f"Error analyzing documents: {str(e)}")
              )
      ```

      - **核心逻辑**：调用 `document_service.analyze_all_documents()`，分析所有未分析文档，并返回结构化结果。

      **核心服务实现**

      ```python
      def analyze_all_documents(self) -> List[DocumentDTO]:
          try:
              unanalyzed_docs = self._repository.find_unanalyzed()
              analyzed_docs = []
              for doc in unanalyzed_docs:
                  try:
                      analyzed_doc = self._analyze_document(doc)
                      analyzed_docs.append(analyzed_doc)
                  except Exception as e:
                      logger.error(f"Document {doc.id} processing failed: {str(e)}")
                      continue
              return analyzed_docs
          except Exception as e:
              logger.error(f"Error occurred during batch analysis: {str(e)}", exc_info=True)
              raise
      ```

      - 先查找所有未分析文档，逐个调用 `_analyze_document` 进行分析。

      **单文档分析流程**

      ```python
      def _analyze_document(self, doc: DocumentDTO) -> DocumentDTO:
          try:
              insight_result = self._insight_kernel.analyze(doc)
              summary_result = self._summary_kernel.analyze(doc, insight_result["insight"])
              updated_doc = self._repository.update_document_analysis(
                  doc.id, insight_result, summary_result
              )
              return updated_doc
          except Exception as e:
              logger.error(f"Document {doc.id} analysis failed: {str(e)}", exc_info=True)
              self._update_analyze_status_failed(doc.id)
              raise
      ```

      - 先生成洞察（insight），再生成摘要（summary），最后写入数据库。

      **洞察与摘要生成**

      ```python
      # 洞察生成
      def analyze(self, doc: DocumentDTO) -> Dict:
          insight_result = self.generator.insighter(insighter_input)
          return {
              "title": insight_result.get("title"),
              "insight": insight_result.get("insight"),
          }

      # 摘要生成
      def analyze(self, doc: DocumentDTO, insight: str = "") -> Dict:
          summary_result = self.generator.summarizer(summarizer_input)
          return {
              "title": summary_result.get("title"),
              "summary": summary_result.get("summary"),
              "keywords": summary_result.get("keywords", []),
          }
      ```

      - 均通过 LLM/算法自动生成。

      **数据库更新**

      ```python
      def update_document_analysis(self, doc_id: int, insight: Dict, summary: Dict) -> Optional[DocumentDTO]:
          with self._db.session() as session:
              document = session.get(self.model, doc_id)
              if document:
                  document.insight = insight
                  document.summary = summary
                  document.analyze_status = ProcessStatus.SUCCESS
                  session.commit()
                  return Document.to_dto(document)
              return None
      ```

      - 更新文档的 `insight`、`summary` 字段，并将 `analyze_status` 设为 `SUCCESS`。

      **整体流程总结**：

      1. 路由接收 POST 请求，调用 `analyze_all_documents`。
      2. 查询所有未分析文档，逐个调用 `_analyze_document`。
      3. 每个文档先生成洞察，再生成摘要，最后写入数据库。
      4. 返回所有已分析文档的结构化信息。

      > 洞察和摘要均通过 LLM/算法自动生成，支持批量处理，异常文档自动跳过并记录失败状态，数据库状态实时更新，便于后续流程衔接。

    - **文档分块（Chunking）机制：**

      文档处理过程中，系统会自动将原始文档内容切分为若干语义片段（分块/Chunk），以便后续的嵌入生成、聚类、检索和知识归纳。

      **核心流程**：
      1. 读取文档原文（raw_content）。
      2. 通过 `DocumentChunker.split(content)` 切分为若干分块（Chunk）。
      3. 每个分块内容、所属文档ID等信息被保存到数据库的 `chunk` 表。
      4. 分块后会进一步生成每个分块的嵌入（embedding），用于后续检索和聚类。

      **主要实现**：
      - 分块逻辑由 `lpm_kernel/file_data/chunker.py` 的 `DocumentChunker` 类实现。
      - 默认采用 LangChain 的 `RecursiveCharacterTextSplitter`，支持多种分隔符（如段落、句号、标点等），优先保证语义完整。
      - 支持 `chunk_size`（如1000字符）和 `overlap`（如200字符）参数，保证分块长度和上下文连续性。

      **示例代码**：
      ```python
      chunker = DocumentChunker(chunk_size=1000, overlap=200)
      chunks = chunker.split(doc.raw_content)
      for chunk in chunks:
          chunk.document_id = doc.id
          chunk_service.save_chunk(chunk)
      ```

      **分块算法细节**：
      - 优先按段落、句子、标点等分隔符切分，内容过长时强制按字符数/Token数切分。
      - 支持分块重叠（overlap），提升上下文连贯性。
      - 底层还可用 `TokenTextSplitter`、`TokenParagraphSplitter` 等做更细粒度的Token级切分。

      **相关数据结构**：
      - `Chunk`：每个分块对象，包含 `id`, `document_id`, `content`, `embedding`, `tags`, `topic` 等字段。
      - `chunk` 表：数据库中存储所有分块内容及其元数据。

      > 文档分块是知识自动整理和智能检索的基础环节，保证了后续聚类、归纳、上下文增强等流程的高效与准确。

---

## 3. kernel_bp
- **用途**：L1 层数据生成与管理（如人物传记、聚类、主题等）。
- **主要前缀**：`/api/kernel`
- **典型接口**：
  - `POST /l1/global/generate`：从 L0 生成 L1 数据。
  - `GET /l1/global/versions`：获取所有 L1 版本。
  - `GET /l1/global/version/<version>`：获取指定版本 L1 数据。
  - `POST /l1/status_bio/generate`：生成状态传记。
  - `GET /l1/status_bio/get`：获取状态传记。

### 详细接口说明：/api/kernel/l1/global/generate

- **用途**：
  - 根据已有的底层知识（L0数据，如文档分块、嵌入等），自动生成结构化的高层知识（L1数据），如人物传记、主题聚类、多视角特征等，并存入数据库。
  - 适用于知识归纳、知识图谱、AI角色画像等场景。

- **请求方式**：
  - `POST /api/kernel/l1/global/generate`
  - **请求体**：无需参数，直接 POST 即可。

- **返回结果**：
  - 成功时：
    ```json
    {
      "code": 0,
      "message": "success",
      "data": {
        "bio": {
          "content": "张三是一位AI专家，专注于自然语言处理和知识图谱研究。",
          "content_third_view": "在同事眼中，张三是技术大牛，乐于分享。",
          "summary": "AI专家，NLP方向，知识图谱。",
          "summary_third_view": "团队核心成员，推动多项创新。",
          "create_time": "2024-06-01T12:34:56",
          "shades": [
            {
              "name": "创新能力",
              "aspect": "技术",
              "icon": "💡",
              "desc_third_view": "善于提出新想法",
              "content_third_view": "多次主导创新项目",
              "desc_second_view": "技术创新",
              "content_second_view": "推动团队技术进步"
            }
          ]
        },
        "clusters": {
          "clusterList": [
            {
              "clusterId": "1",
              "memory_ids": ["101", "102"],
              "topic": "自然语言处理",
              "cluster_center": { "embedding": [0.1, 0.2] }
            },
            {
              "clusterId": "2",
              "memory_ids": ["103"],
              "topic": "知识图谱",
              "cluster_center": { "embedding": [0.3, 0.4] }
            }
          ]
        },
        "chunk_topics": {
          "chunk_101": {
            "topic": "NLP",
            "tags": ["文本处理", "分词"]
          },
          "chunk_102": {
            "topic": "实体识别",
            "tags": ["NER", "信息抽取"]
          }
        },
        "generate_time": "2024-06-01T12:34:56"
      }
    }
    ```
  - 失败时：
    ```json
    {
      "code": 1,
      "message": "No valid L1 data generated"
    }
    ```

- **字段说明**：
  - `bio`：全局人物传记/知识总结（多视角）
    - `content`：主视角传记内容
    - `content_third_view`：第三视角描述
    - `summary`：主视角摘要
    - `summary_third_view`：第三视角摘要
    - `create_time`：生成时间
    - `shades`：多视角特征列表
  - `clusters`：主题聚类结果
    - `clusterList`：聚类列表，每个聚类包含 `clusterId`、`memory_ids`、`topic`、`cluster_center`
  - `chunk_topics`：每个分块的主题标签
    - 以分块ID为 key，每个 value 包含 `topic`、`tags`
  - `generate_time`：本次 L1 生成的时间戳

- **示例应用场景**：
  - 知识库自动归纳总结，生成结构化知识画像
  - AI 角色自动生成背景故事
  - 数据分析与主题聚类

---

## 4. kernel2_bp
- **用途**：L2 层及模型管理、推理、训练等高级功能。
- **主要前缀**：`/api/kernel2`
- **典型接口**：
  - `GET /health`：L2 服务健康检查。
  - `POST /model/download`：下载基础模型。
  - `POST /data/prepare`：准备训练数据。
  - `POST /train2`：L2 训练。
    
    **详细说明：**
    - **用途**：启动 L2 层的个性化模型训练任务。该接口会根据用户指定的模型名称和训练参数，后台异步调用训练脚本，进行模型微调或个性化训练。
    - **请求方式**：`POST /api/kernel2/train2`，请求体为 JSON。
    - **请求参数**：
      - `model_name`（必填）：基础模型名称（如 "Qwen/Qwen2.5-0.5B-Instruct"）。
      - `learning_rate`（选填，默认 2e-4）：学习率。
      - `number_of_epochs`（选填，默认 3）：训练轮数。
      - `concurrency_threads`（选填，默认 2）：并发线程数。
      - `data_synthesis_mode`（选填，默认 "low"）：数据合成模式。
      - `use_cuda`（选填，默认 false）：是否使用 CUDA。
    - **处理流程**：
      1. 校验参数，检查模型是否已下载。
      2. 检查是否已有训练任务在运行。
      3. 设置环境变量，准备日志和输出目录。
      4. 后台异步启动训练脚本（如 `train_for_user.sh`），并将日志写入 `logs/train.log`。
      5. 立即返回任务已启动的响应。
    - **返回示例**：
      ```json
      {
        "code": 0,
        "message": "Training task started successfully",
        "data": {
          "status": "training_started",
          "model_name": "Qwen/Qwen2.5-0.5B-Instruct",
          "log_path": "/path/to/logs/train.log"
        }
      }
      ```
      失败时：
      ```json
      {
        "code": 1,
        "message": "Model 'xxx' does not exist, please download first"
      }
      ```
    - **字段说明**：
      - `status`：训练任务状态（training_started）。
      - `model_name`：本次训练的模型名称。
      - `log_path`：训练日志文件路径。
    - **典型应用场景**：
      - 用户发起个性化微调、增量训练。
      - 后台异步训练，前端可通过日志接口实时查看训练进度。
  - `POST /llama/start|stop`、`GET /llama/status`：llama-server 启停与状态。
  - `POST /chat`：对话推理。

---

## 5. loads_bp
- **用途**：用户身份/实例注册与管理。
- **主要前缀**：`/api/loads`
- **典型接口**：
  - `POST /`：创建新 load 记录。
  - `GET /current`：获取当前 load 信息。
  - `PUT /current`：更新当前 load 信息。
  - `DELETE /<load_name>`：删除指定 load。
  - `POST|GET /<load_name>/avatar`：上传/获取头像。

---

## 6. memories_bp
- **用途**：记忆文件上传与管理。
- **主要前缀**：`/api/memories`
- **典型接口**：
  - `POST /file`：上传记忆文件（txt/pdf/md）。
  - `DELETE /file/<filename>`：删除记忆文件。

---

## 7. role_bp
- **用途**：角色管理（L2 层）。
- **主要前缀**：`/api/kernel2/roles`
- **典型接口**：
  - `POST /`：创建角色。
  - `GET /`：获取所有角色。
  - `GET|PUT|DELETE /<uuid>`：获取/更新/删除指定角色。
  - `POST /share`：角色共享到注册中心。

---

## 8. trainprocess_bp
- **用途**：训练流程管理。
- **主要前缀**：`/api/trainprocess`
- **典型接口**：
  - `POST /start`：启动训练流程。
  - `GET /logs`：实时获取训练日志。
  - `GET /progress/<model_name>`：获取训练进度。
  - `POST /progress/reset`：重置进度。
  - `POST /stop`：停止训练。
  - `GET /training_params`：获取训练参数。
  - `POST /retrain`：重新训练。

---

## 9. upload_bp
- **用途**：上传实例注册、连接、状态管理。
- **主要前缀**：`/api/upload`
- **典型接口**：
  - `POST /register`：注册上传实例。
  - `POST /connect`：建立 WebSocket 连接。
  - `GET /status`：获取上传实例状态。
  - `DELETE /`：注销上传实例。
  - `GET /`：列出所有上传实例。
  - `GET /count`：统计上传实例数量。
  - `PUT /`：更新上传实例信息。

---

## 10. space_bp
- **用途**：Space（空间/讨论组）管理。
- **主要前缀**：`/api/space`
- **典型接口**：
  - `POST /create`：创建 Space。
  - `GET /<space_id>`：获取 Space 信息。
  - `GET /all`：获取所有 Space。
  - `DELETE /<space_id>`：删除 Space。
  - `POST /<space_id>/start`：开始讨论。
  - `GET /<space_id>/status`：获取讨论状态。
  - `POST /<space_id>/share`：分享 Space。

---

## 11. talk_bp
- **用途**：高级对话与多轮会话。
- **主要前缀**：`/api/talk`
- **典型接口**：
  - `POST /chat`：流式对话接口。
  - `POST /chat_json`：JSON 格式对话接口。
  - `POST /advanced_chat`：多阶段高级对话。

---

## 12. user_llm_config_bp
- **用途**：用户 LLM（大模型）配置管理。
- **主要前缀**：`/api/user-llm-configs`
- **典型接口**：
  - `GET /`：获取 LLM 配置。
  - `PUT /`：更新 LLM 配置。
  - `PUT /thinking`：更新思维模型配置。
  - `DELETE /key`：删除 API key。 