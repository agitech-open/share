# lpm_kernel 项目主要数据结构梳理

---

## 1. 文档与分块相关

### Document / DocumentDTO
- **Document**（ORM模型，`file_data/document.py`、`file_data/models.py`）：
  - 字段：id、name、title、mime_type、user_description、url、document_size、raw_content、insight、summary、extract_status、embedding_status、analyze_status、create_time 等。
  - 作用：数据库中的文档对象，记录原始文件及其结构化信息。
- **DocumentDTO**（Pydantic模型，`file_data/document_dto.py`）：
  - 字段与Document类似，便于API传输和序列化。

### Chunk / ChunkDTO
- **Chunk**（`L1/bio.py`）：
  - 字段：id、document_id、content、embedding（向量）、tags、topic。
  - 作用：文档分块，支持内容分割与向量化。
- **ChunkDTO**（`file_data/dto/chunk_dto.py`）：
  - 字段：id、document_id、content、embedding、has_embedding、tags、topic、length。
  - 作用：API返回用的分块结构。

---

## 2. 嵌入与记忆

### Memory
- **Memory**（`models/memory.py`）：
  - 字段：id、name、size、type、path、meta_data、document_id、created_at、updated_at、status。
  - 作用：记录文件/分块的元数据和嵌入信息。

---

## 3. L0 层数据结构

- **FileInfo**（`L0/models.py`）：
  - 字段：data_type、filename、content、file_content。
  - 作用：文件相关信息，供L0分析使用。
- **BioInfo**（`L0/models.py`）：
  - 字段：global_bio、status_bio、about_me。
  - 作用：用户生平信息。
- **InsighterInput/SummarizerInput**（`L0/models.py`）：
  - 字段：file_info、bio_info/insight。
  - 作用：L0 insight/summary 生成的输入结构。

---

## 4. L1 层数据结构

### Note/Memory/Cluster
- **Note**（`L1/bio.py`）：
  - 字段：id、content、create_time、memory_type、embedding、chunks、title、summary、insight、tags、topic。
  - 作用：笔记，承载分块后的内容及其结构化信息。
- **Memory**（`L1/bio.py`）：
  - 字段：memoryId、embedding。
  - 作用：L1层的记忆单元。
- **Cluster**（`L1/bio.py`）：
  - 字段：cluster_id、memory_list、cluster_center、is_new、size。
  - 作用：主题聚类，支持知识归纳。

### 传记与特征
- **Bio**（`L1/bio.py`）：
  - 字段：content、contentThirdView、summary、summaryThirdView、attributeList、shadesList。
  - 作用：人物传记，多视角知识总结。
- **ShadeInfo/AttributeInfo**（`L1/bio.py`）：
  - 字段：名称、描述、置信度、时间线等。
  - 作用：多视角特征、属性信息。

### L1GenerationResult
- **L1GenerationResult**（`models/l1.py`）：
  - 字段：bio（传记）、clusters（聚类）、chunk_topics（分块主题）、generate_time。
  - 作用：L1生成结果的结构化封装。

---

## 5. L2 层与训练数据

- **DPOData**（`L2/dpo/dpo_data.py`）：
  - 字段：prompt、chosen、rejected。
  - 作用：DPO训练数据结构。
- **L2DataProcessor**（`L2/data.py`）：
  - 作用：L2训练数据处理器，负责数据格式转换、信息提取等。

---

## 6. 空间/讨论组相关

- **Space**（`models/space.py`）：
  - 字段：id、title、objective、participants、host、create_time、status、conclusion。
  - 作用：讨论组/空间。
- **SpaceMessage**（`models/space.py`）：
  - 字段：id、space_id、sender_endpoint、content、message_type、round、create_time、role。
  - 作用：空间内消息。

---

## 7. 状态传记

- **StatusBiography**（`models/status_biography.py`）：
  - 字段：id、content、content_third_view、summary、summary_third_view、create_time、update_time。
  - 作用：状态传记，支持多视角知识总结。

---

## 8. 其它典型结构

- **ProcessStatus**（`file_data/process_status.py`）：文档处理状态（INITIALIZED、FAILED等）。
- **各种DTO**（`dto.py`）：如UserLLMConfigDTO、RoleDTO、ChatDTO等，均为API传输用的结构体。

---

## 结构化举例

- 文档（Document）→ 分块（Chunk/ChunkDTO）→ 嵌入（embedding）→ 记忆（Memory）→ 聚类（Cluster）→ 传记/特征（Bio/ShadeInfo/AttributeInfo）→ 训练数据（DPOData等）
- 空间（Space）→ 消息（SpaceMessage）
- L0/L1/L2各层有专属的数据结构，支持知识分层、聚类、特征提取、训练与推理。

---

如需某一具体结构的详细字段或代码示例，请查阅对应模块源码或联系开发者！ 