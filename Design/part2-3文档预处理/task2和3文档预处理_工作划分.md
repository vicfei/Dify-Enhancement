这个划分非常清晰。基于我们之前的设计（解析器只负责“读”和“转”，切块与入库负责“业务逻辑”和“写”），API开发工作的归属如下：

### 结论速览

- **任务1（解析器开发）**：**不涉及任何对外API和Dify平台接口**。它只负责内部组件，输入是文件流，输出是标准化的`elements.json`缓存。它是“无状态”和“纯技术”的。
- **任务2（切块与入库）**：**承担所有业务逻辑、对外API接口、Dify平台交互以及状态管理**。它是“有状态”和“业务编排”的。

---

### 详细拆解：各任务的具体接口归属

#### 任务1：解析器开发（Parser Development）

**核心目标**：将MinerU封装成标准化的内部服务，输出`elements.json`。

| 接口类型 | 具体接口/组件 | 职责描述 | 为什么归属这里 |
| :--- | :--- | :--- | :--- |
| **内部组件** | **MinerU适配器 (ParserAdapter)** | 调用MinerU HTTP API，处理超时、重试，将MinerU的输出映射为我们的`Element`契约。 | 这是解析器的核心逻辑，只关心“把文件变成结构化元素”。 |
| **内部组件** | **对象存储写入** | 将标准化后的`elements.json`写入MinIO/OSS缓存路径（`/parsed_cache/{doc_uuid}/elements.json`）。 | 它是解析结果的持久化，属于解析环节的产出。 |
| **内部契约** | **`ParsedResult` 数据结构** | 定义解析器输出的Python数据类（包含`elements`列表、统计信息等）。 | 这是解析器的产出物定义。 |

> **重要说明**：任务1 **不开发任何HTTP RESTful API**。它对外（对调度器）暴露的是Python类方法（如`parse(ctx)`），而不是Web接口。它**不依赖Dify的任何API**（它不需要知道`source_kb`是否存在，也不需要鉴权）。

---

#### 任务2：切块与入库（Chunking & Storage）

**核心目标**：编排整个预处理流水线，管理状态，处理切块逻辑，并对外提供业务接口。

| 接口类型 | 具体接口/组件 | 职责描述 | 为什么归属这里 |
| :--- | :--- | :--- | :--- |
| **对外核心API** | **`POST /ingest`** | 接收文件，触发异步处理。调用任务1的解析器，然后执行切块和入库。 | **业务入口**，依赖切块逻辑和入库写入。 |
| **对外核心API** | **`GET /status/{doc_uuid}`** | 查询处理进度、当前阶段、重试次数。 | 依赖MySQL中的`status`和`retry_count`字段（由本任务维护）。 |
| **对外核心API** | **`GET /` (文档列表)** | 分页查询文档元数据，支撑统一门户。 | 完全依赖本任务写入的MySQL数据（`kb_documents`表）。 |
| **对外核心API** | **`GET /{doc_uuid}` (文档详情)** | 获取单篇文档的元数据、统计信息和错误堆栈。 | 读取本任务维护的文档主表和统计数据。 |
| **对外核心API** | **`POST /retry/{doc_uuid}`** | 手动重试失败文档。 | 修改本任务维护的`status`状态机。 |
| **外部依赖（Dify）** | **`GET /info`** | 验证API Key，获取租户身份（`tenant_id`）。 | 业务层鉴权，确定数据归属。 |
| **外部依赖（Dify）** | **`GET /datasets/{dataset_id}`** | 校验`source_kb`是否存在，获取`embedding_model`配置。 | 确保写入的向量与Dify知识库配置对齐（业务合规性）。 |
| **外部依赖（Dify）** | **`GET /workspaces/current/models/model-types/text-embedding`** | （可选）校验用户指定的Embedding模型是否可用。 | 业务配置校验。 |
| **内部组件** | **`VersionResolver`** | 根据`file_hash`查询MySQL，判断是`skip`、`new`还是`retry`。 | 依赖MySQL（本任务管理的数据表）。 |
| **内部组件** | **`ChunkerEngine`** | 读取`elements.json`，执行父子块拆分逻辑。 | **切块核心业务逻辑**。 |
| **内部组件** | **`TripleWriter`** | 批量写入MySQL → ES → Milvus，并更新文档状态。 | **入库核心写入逻辑**，三库写入全在本任务。 |
| **内部组件** | **`IngestionOrchestrator`** | 编排上述所有组件，管理状态机流转。 | 业务总控。 |

---

### 总结对比表

| 交付物 | 任务1（解析器） | 任务2（切块与入库） |
| :--- | :--- | :--- |
| **对外RESTful API** | ❌ 无 | ✅ `POST /ingest`, `GET /status`, `GET /` , `GET /{id}`, `POST /retry` |
| **Dify平台API交互** | ❌ 无 | ✅ `/info`, `/datasets/{id}`, `/models/...` |
| **核心数据写入** | ❌ 只写对象存储缓存 | ✅ 写入 MySQL / ES / Milvus |
| **业务状态管理** | ❌ 无状态 | ✅ 状态机（pending→processing→indexed/failed） |
| **依赖的基础组件** | MinerU服务、对象存储SDK | 任务1的解析器、MySQL/ES/Milvus SDK |
| **对外暴露的接口形态** | Python类内部接口（`parse()`） | HTTP API + Celery/异步任务队列 |

**结论**：在排期上，**任务1（解析器）** 是任务2的**前置依赖**。开发顺序应为：
1. 先完成解析器开发（并Mock好`ParsedResult`输出）。
2. 然后同步开发切块与入库的业务API（此时调用解析器的Mock或真实接口）。