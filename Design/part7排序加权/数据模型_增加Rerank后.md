你说得完全正确！我此前把 v5.1（召回+Rerank）和 v6.0（排序）作为两个独立文档呈现，这在逻辑上是对的（分层解耦），但在**整体数据模型**的视角下，它们必须融合成**一份完整的、统一的文档**。

目前最新的整体数据模型应该是 **v6.1（融合版）**，包含：
- **v5.1**：任务6的 Rerank 模型追踪字段 + Nacos Rerank 配置
- **v6.0**：任务7的 `kb_rank_logs` 表 + 个性化字段 + 排序权重配置

我现在为你生成**完整融合后的数据模型 v6.1**，包含全部 **15张表**（原14张 + 新增 `kb_rank_logs`），并将所有变更统一纳入版本历史。


# 数据模型设计（完整版 v6.1）

> **版本更新说明**：
> - **v6.0 → v6.1**：将任务6的 Rerank 模型（Cross-Encoder）追踪字段与任务7（排序加权）完整融合。`kb_search_logs` 中保留 v5.1 新增的 Rerank 追踪字段（`rerank_enabled`、`rerank_model`、`rerank_input_count`、`rerank_latency_ms`），Nacos `search_hybrid.yaml` 中保留完整 `rerank` 配置段。所有 v5.1 + v6.0 的变更现已合并为单一数据模型版本。**📌 v6.1** 标注了所有相对于 v5.0 的新增/修改内容。
> - **v5.0 → v6.0**：整合任务7（排序加权），新增 `kb_rank_logs` 排序日志表，在 `kb_sessions` 中增加 `user_role` 字段，在 `kb_user_feedback` 中增加排序环节追溯字段（`rank_helpful`、`rank_snapshot`），新增 Nacos `ranking_user_weights.yaml`，扩展 `ranking_weights` 配置段。
> - **v5.0 → v5.1**：整合任务6的 Rerank 模型追踪，在 `kb_search_logs` 中新增 Rerank 执行字段，在 Nacos `search_hybrid.yaml` 中新增 `rerank` 配置段。
> - **v4.0 → v5.0**：整合任务4的 MCP（模型上下文协议）动态元数据集成方案。
> - **v3.0 → v4.0**：针对任务6（召回策略），新增召回日志表。
> - **v2.0 → v3.0**：针对任务5（意图识别、拆解与改写），新增会话上下文表与查询处理日志表。
> - **v1.0 → v2.0**：新增路由日志表与反馈表增强字段。

数据模型设计服务于两个核心场景：**写入**（文档预处理后入库）和**读取**（检索时快速过滤和召回）。整体分为五块：

1. **MySQL**：存储元数据、切块原文、标签、版本、反馈、路由日志、会话上下文、查询日志、召回日志（含Rerank追踪）、**排序日志（v6.1）**、MCP 配置与映射等结构化数据
2. **Elasticsearch**：存储切块文本，用于关键词检索（BM25）和元数据过滤
3. **向量数据库（Milvus）** ：存储切块向量，用于语义检索
4. **配置中心（Nacos/Apollo）** ：存储全局字典、路由运行时参数（含MCP）、意图理解策略、**混合检索参数（含Rerank配置）**、**排序权重（含个性化角色权重）** 等热配置
5. **对象存储（MinIO/OSS）** ：存储原始文档文件及解析缓存


## 一、MySQL 数据模型

一共需要 **15张** 核心表（原14张 + 新增1张排序日志表）。

### 1.1 文档主表：`kb_documents`

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `doc_uuid` | varchar(64) | UNIQUE NOT NULL | 文档唯一标识，API使用 |
| `file_name` | varchar(255) | NOT NULL | 原始文件名 |
| `file_hash` | varchar(64) | INDEX | 全文SHA-256，用于版本比对 |
| `file_path` | varchar(512) | | 对象存储路径 |
| `source_kb` | varchar(32) | NOT NULL | 来源知识库：fund/credit/compliance等 |
| `doc_type` | varchar(32) | | 文档类型：研报/公告/合同等 |
| `version` | varchar(16) | DEFAULT 'v1.0' | 当前版本号 |
| `status` | tinyint | DEFAULT 0 | 0-待处理/处理中 / 1-已索引 / 2-异常 |
| `published_date` | datetime | INDEX | 文档发布日期 |
| `retry_count` | tinyint | DEFAULT 0 | 重试次数（补偿任务递增，超过3次则停止自动重试） |
| `last_error` | text | NULL | 最后一次失败堆栈或错误信息 |
| `processing_started_at` | datetime | NULL | 最近一次开始处理的时间（用于检测卡死任务） |
| `indexed_at` | datetime | NULL | 三库全部写入完成的时间 |
| `parser_engine` | varchar(64) | NULL | 解析引擎及版本，如"mineru-1.2.0" |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |
| `updated_at` | datetime | DEFAULT CURRENT_TIMESTAMP ON UPDATE | 更新时间 |

**索引**：
```sql
CREATE INDEX idx_source_kb ON kb_documents(source_kb);
CREATE INDEX idx_file_hash ON kb_documents(file_hash);
CREATE INDEX idx_status_created ON kb_documents(status, created_at);
```


### 1.2 切块表：`kb_chunks`

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `chunk_uuid` | varchar(64) | UNIQUE NOT NULL | 切块唯一ID，三库关联主键 |
| `doc_id` | bigint | FOREIGN KEY NOT NULL | 关联文档主表 |
| `parent_chunk_id` | bigint | FOREIGN KEY NULL | 父块ID（自关联） |
| `content_type` | varchar(16) | DEFAULT 'CHILD' | PARENT / CHILD |
| `content_text` | longtext | NOT NULL | 切块正文 |
| `chapter_path` | varchar(512) | | 章节路径 |
| `token_count` | int | | Token数量 |
| `source_elem_ids` | json | NULL | 引用的解析器原始元素ID列表，用于溯源高亮 |
| `page_numbers` | varchar(64) | NULL | 该块覆盖的页码范围 |
| `is_active` | tinyint | DEFAULT 1 | 1-生效 / 0-已废弃 |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |

**索引**：
```sql
CREATE INDEX idx_doc_id ON kb_chunks(doc_id);
CREATE INDEX idx_parent_chunk ON kb_chunks(parent_chunk_id);
```


### 1.3 标签表：`kb_tags`

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `tag_name` | varchar(64) | UNIQUE NOT NULL | 标签名称 |
| `tag_category` | varchar(32) | | industry / topic / risk / region / auto_extracted |
| `is_active` | tinyint | DEFAULT 1 | 是否启用 |


### 1.4 文档标签关联表：`kb_doc_tag_relation`

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `doc_id` | bigint | FOREIGN KEY NOT NULL | 文档ID |
| `tag_id` | bigint | FOREIGN KEY NOT NULL | 标签ID |

**索引**：
```sql
CREATE UNIQUE INDEX idx_unique_doc_tag ON kb_doc_tag_relation(doc_id, tag_id);
```


### 1.5 投研实体映射表：`kb_entity_mapping`

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `chunk_id` | bigint | FOREIGN KEY NOT NULL | 关联切块 |
| `entity_type` | varchar(16) | NOT NULL | bond_code / fund_code / stock_code |
| `entity_code` | varchar(64) | NOT NULL | 实体编码 |

**索引**：
```sql
CREATE INDEX idx_entity ON kb_entity_mapping(entity_type, entity_code);
```


### 1.6 版本变更记录表：`kb_version_changes`

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `doc_id` | bigint | FOREIGN KEY NOT NULL | 文档ID |
| `old_version` | varchar(16) | | 旧版本号 |
| `new_version` | varchar(16) | | 新版本号 |
| `status` | tinyint | DEFAULT 0 | 0-待确认 / 1-已合并 |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |


### 1.7 用户反馈表：`kb_user_feedback`（全链路追踪融合版，**含排序层字段 v6.1**）

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `trace_id` | varchar(64) | INDEX | 全链路追踪ID |
| `original_query` | text | NULL | 用户原始问句 |
| `query_text` | varchar(500) | | 用户问题（已废弃，保留兼容） |
| `thumbs_up` | tinyint | | 1-点赞 / 0-点踩 |
| `rewritten_query_used` | text | NULL | 当时系统使用的改写Query |
| `rewrite_helpful` | tinyint | NULL | 改写是否有帮助 |
| `comment` | varchar(500) | | 用户备注 |
| `router_snapshot` | json | NULL | 路由决策快照：含 `mcp_enhanced` 标识 |
| `selected_kb` | varchar(32) | NULL | 用户实际浏览的知识库 |
| `search_snapshot` | json | NULL | 召回快照：RRF融合/Rerank后的Top-K列表 |
| `retrieved_chunks` | json | NULL | 实际返回的 `chunk_uuid` 列表 |
| `search_helpful` | tinyint | NULL | 召回内容是否相关 |
| **`rank_helpful`** 📌 v6.0 | tinyint | NULL | **用户点踩时可选：排序是否合理？（1=相关但排太靠后，0=完全不相关/排序无效）** |
| **`rank_snapshot`** 📌 v6.0 | json | NULL | **排序决策快照**：记录最终的 `(w_rel, w_auth, w_time)` 三元组及 Top-3 块各自的因子得分 |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |

**📌 v6.0 新增 DDL**：
```sql
ALTER TABLE kb_user_feedback 
ADD COLUMN rank_helpful tinyint NULL COMMENT '排序是否合理（1=相关但排太后，0=排序无效）',
ADD COLUMN rank_snapshot json NULL COMMENT '排序权重快照及Top-3因子得分';
```


### 1.8 路由日志表：`kb_router_logs`（含 MCP 追踪）

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `trace_id` | varchar(64) | NOT NULL INDEX | 全链路追踪ID |
| `user_query` | varchar(500) | NOT NULL | 用户原始问题 |
| `context_kb` | varchar(32) | NULL | 请求时的上下文知识库 |
| `routed_kbs` | json | NOT NULL | 路由返回的候选库列表 |
| `clarity_flag` | varchar(16) | NOT NULL | clear / ambiguous / unknown |
| `confidence` | decimal(4,3) | NOT NULL | 置信度 |
| `matched_keywords` | json | NULL | 命中的关键词列表 |
| `filter_tags` | json | NULL | 提取的标签 |
| `mcp_triggered` | tinyint | DEFAULT 0 | 是否触发了MCP动态干预 |
| `mcp_signals` | json | NULL | MCP返回的原始信号快照 |
| `mcp_weight_applied` | json | NULL | MCP信号映射的权重增量 |
| `selected_kb` | varchar(32) | NULL | 用户最终选择的知识库 |
| `is_resolved` | tinyint | DEFAULT 0 | 0-待确认 / 1-已完成 |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |

**索引**：
```sql
CREATE INDEX idx_trace_id ON kb_router_logs(trace_id);
CREATE INDEX idx_mcp_triggered ON kb_router_logs(mcp_triggered);
```


### 1.9 会话上下文表：`kb_sessions`（📌 v6.0 新增 `user_role` 字段）

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `session_id` | varchar(64) | PRIMARY KEY | 会话ID |
| `user_id` | varchar(64) | NOT NULL | 用户标识 |
| **`user_role`** 📌 v6.0 | varchar(32) | NULL | **用户角色（基金经理/风控专员/信评分析师等），用于排序阶段拉取个性化权重** |
| `source_kb` | varchar(32) | NULL | 当前会话主要知识库 |
| `history_summary` | text | NULL | 历史对话摘要（JSON） |
| `last_query` | text | NULL | 上一轮用户问句 |
| `last_processed` | text | NULL | 上一轮改写Query列表 |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |
| `updated_at` | datetime | DEFAULT CURRENT_TIMESTAMP ON UPDATE | 更新时间 |

**索引**：
```sql
CREATE INDEX idx_user_session ON kb_sessions(user_id, updated_at);
CREATE INDEX idx_user_role ON kb_sessions(user_role);
```


### 1.10 查询处理日志表：`kb_query_logs`

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `log_id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `trace_id` | varchar(64) | NOT NULL INDEX | 全链路追踪ID |
| `session_id` | varchar(64) | NULL | 会话ID |
| `original_query` | text | NOT NULL | 用户原始输入 |
| `intent_type` | varchar(32) | NULL | 意图类型 |
| `confidence` | decimal(4,3) | NULL | 置信度 |
| `is_decomposed` | tinyint(1) | DEFAULT 0 | 是否触发拆解 |
| `sub_questions` | json | NULL | 拆解后的子问题列表 |
| `rewritten_queries` | json | NOT NULL | 最终改写Query列表 |
| `expanded_terms` | json | NULL | 扩展的同义词列表 |
| `llm_call_duration_ms` | int | NULL | LLM总耗时 |
| `fallback_triggered` | tinyint(1) | DEFAULT 0 | 是否触发降级 |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |


### 1.11 召回日志表：`kb_search_logs`（📌 v5.1 含 Rerank 追踪字段）

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `trace_id` | varchar(64) | NOT NULL INDEX | 全链路追踪ID |
| `session_id` | varchar(64) | NULL | 会话ID |
| `original_query` | text | NOT NULL | 用户原始输入 |
| `rewritten_query` | text | NULL | 改写后的主Query |
| `sub_questions` | json | NULL | 拆解后的子问题列表 |
| `tag_filters` | json | NULL | 标签筛选条件 |
| `entity_constraints` | json | NULL | 投研实体代码列表 |
| `source_kb` | varchar(32) | NULL | 路由锁定的知识库 |
| `semantic_query` | text | NULL | 实际送给Milvus的查询文本 |
| `keyword_query` | text | NULL | 实际送给ES的查询文本 |
| `semantic_top_k` | int | DEFAULT 50 | 语义通路召回数 |
| `keyword_top_k` | int | DEFAULT 50 | 关键词通路召回数 |
| `semantic_hits` | int | NULL | 语义通路命中数 |
| `keyword_hits` | int | NULL | 关键词通路命中数 |
| `fuzzy_hits` | int | NULL | 模糊通路命中数 |
| `overlap_count` | int | NULL | 多路召回重叠数 |
| **`rerank_enabled`** 📌 v5.1 | tinyint(1) | DEFAULT 0 | **是否启用了 Rerank 模型精排** |
| **`rerank_model`** 📌 v5.1 | varchar(64) | NULL | **Rerank 模型标识，如 `cohere/rerank-v3.5`** |
| **`rerank_input_count`** 📌 v5.1 | int | NULL | **送入 Rerank 的候选块数量** |
| **`rerank_latency_ms`** 📌 v5.1 | int | NULL | **Rerank 调用耗时（毫秒）** |
| `rrf_params` | json | NULL | RRF融合参数快照 |
| `final_result_count` | int | NULL | 最终返回数量 |
| `semantic_latency_ms` | int | NULL | Milvus检索耗时 |
| `keyword_latency_ms` | int | NULL | ES检索耗时 |
| `fusion_latency_ms` | int | NULL | 融合排序耗时 |
| `fallback_triggered` | tinyint(1) | DEFAULT 0 | 是否触发宽松召回 |
| `status` | tinyint | DEFAULT 1 | 1=成功 / 0=失败 |
| `error_msg` | text | NULL | 异常信息 |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |

**索引**：
```sql
CREATE INDEX idx_trace ON kb_search_logs(trace_id);
CREATE INDEX idx_created ON kb_search_logs(created_at);
CREATE INDEX idx_rerank ON kb_search_logs(rerank_enabled);
```


### 1.12 排序日志表：`kb_rank_logs`（📌 v6.0 新增）

> **新增原因**：排序环节成为一个需要独立可观测的“黑盒”，必须记录权重三元组、因子得分、多样性策略等，与召回日志解耦。

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `trace_id` | varchar(64) | NOT NULL INDEX | 全链路追踪ID（关联 `kb_search_logs`） |
| `session_id` | varchar(64) | NULL | 会话ID |
| `user_role` | varchar(32) | NULL | 本次请求使用的用户角色（冗余便于排查） |
| `candidate_count` | int | NOT NULL | 排序前候选集大小（通常 ≤ 50） |
| `ranked_count` | int | NOT NULL | 排序后输出数量（即送入LLM的块数） |
| `used_weights` | json | NOT NULL | 实际使用的权重三元组：`{"w_relevance": 0.45, "w_authority": 0.30, "w_timeliness": 0.25}` |
| `authority_coefficients` | json | NULL | 文档来源系数快照 |
| `time_decay_lambda` | decimal(4,3) | NULL | 本次使用的时效衰减率（如 `0.5`） |
| `diversity_mode` | varchar(16) | NULL | 多样性策略：`none` / `cap_per_doc` / `mmr` |
| `max_chunks_per_doc` | int | DEFAULT 3 | 单个文档允许进入 Top-N 的最大块数 |
| `scenario_boosts` | json | NULL | 场景加成系数应用记录 |
| `final_top_scores` | json | NULL | 最终Top-5排序块的 `chunk_uuid` 及对应 `rank_score` |
| `total_latency_ms` | int | NULL | 排序环节总耗时（毫秒） |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |

**索引**：
```sql
CREATE INDEX idx_trace ON kb_rank_logs(trace_id);
CREATE INDEX idx_created ON kb_rank_logs(created_at);
CREATE INDEX idx_user_role ON kb_rank_logs(user_role);
```


### 1.13 MCP 服务提供者配置表：`kb_mcp_providers`

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `provider_name` | varchar(64) | UNIQUE NOT NULL | 服务名称 |
| `endpoint_url` | varchar(255) | NOT NULL | MCP服务地址 |
| `auth_type` | varchar(16) | DEFAULT 'none' | 鉴权方式 |
| `auth_credential` | varchar(255) | NULL | 加密凭证 |
| `timeout_ms` | int | DEFAULT 500 | 超时（毫秒） |
| `cache_ttl_seconds` | int | DEFAULT 120 | 缓存有效期 |
| `retry_count` | tinyint | DEFAULT 1 | 最大重试次数 |
| `is_active` | tinyint | DEFAULT 1 | 是否启用 |
| `priority` | int | DEFAULT 0 | 调用优先级 |
| `description` | varchar(255) | NULL | 服务描述 |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |
| `updated_at` | datetime | DEFAULT CURRENT_TIMESTAMP ON UPDATE | 更新时间 |


### 1.14 MCP 信号→路由权重映射表：`kb_mcp_weight_mapping`

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `provider_id` | bigint | FOREIGN KEY NOT NULL | 关联 `kb_mcp_providers.id` |
| `signal_key` | varchar(64) | NOT NULL | MCP字段名 |
| `signal_value` | varchar(64) | NOT NULL | MCP具体值 |
| `target_kb` | varchar(32) | NOT NULL | 目标知识库 |
| `weight_delta` | decimal(4,2) | NOT NULL | 权重增量 |
| `expire_seconds` | int | DEFAULT 0 | 有效期（秒），0=永久 |
| `operator` | varchar(16) | DEFAULT 'eq' | 比较运算符 |
| `is_active` | tinyint | DEFAULT 1 | 是否启用 |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |
| `updated_at` | datetime | DEFAULT CURRENT_TIMESTAMP ON UPDATE | 更新时间 |


### 1.15 MCP 实时信号缓存表：`kb_mcp_cache`

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `cache_key` | varchar(128) | UNIQUE NOT NULL | 查询实体哈希 |
| `provider_id` | bigint | FOREIGN KEY NOT NULL | 关联 `kb_mcp_providers.id` |
| `request_tokens` | json | NOT NULL | 原始Token列表 |
| `response_payload` | json | NOT NULL | MCP返回快照 |
| `expires_at` | datetime | NOT NULL | 过期时间 |
| `hit_count` | int | DEFAULT 0 | 缓存命中次数 |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |


## 二、Elasticsearch 索引设计

### 索引名：`kb_chunks_v1`

```json
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "ik_max_word_analyzer": {
          "type": "custom",
          "tokenizer": "ik_max_word"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "chunk_uuid": {"type": "keyword"},
      "doc_id": {"type": "long"},
      "parent_chunk_id": {"type": "long"},
      "content_text": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart"
      },
      "chapter_path": {"type": "keyword"},
      "source_kb": {"type": "keyword"},
      "doc_type": {"type": "keyword"},
      "published_date": {"type": "date"},
      "tags": {"type": "keyword"},
      "entity_codes": {"type": "keyword"},
      "is_active": {"type": "boolean"}
    }
  }
}
```


## 三、向量数据库（Milvus）设计

### Collection 名：`kb_embeddings_v1`

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `chunk_uuid` | VARCHAR(64) | PRIMARY KEY | 与ES/MySQL保持一致 |
| `embedding` | FLOAT_VECTOR(1536) | NOT NULL | 向量 |
| `source_kb` | VARCHAR(32) | PARTITION KEY | 分区键 |
| `is_active` | BOOL | | 是否生效 |


## 四、配置中心（Nacos）存储内容

### 4.1 路由规则（静态字典）
```yaml
routing:
  - keywords: ["净值", "收益率", "夏普"]
    target_kbs: ["基金知识库"]
  - keywords: ["评级", "违约", "利差"]
    target_kbs: ["信评知识库", "固收投研知识库"]
```

### 4.2 同义词表
```yaml
synonyms:
  - ["ROE", "净资产收益率"]
  - ["久期", "持续期"]
```

### 4.3 排序权重（📌 v6.0 扩展）

```yaml
ranking_weights:
  # --- 原有字段（保留，作为基线） ---
  alpha_similarity: 0.5
  gamma_source: 0.25
  delta_time: 0.25

  source_priority:
    监管公告: 1.2
    内部研报: 1.0
    外部资讯: 0.8

  # --- 📌 v6.0 新增：排序算法超参数 ---
  normalization_method: "min_max"
  time_decay:
    lambda: 0.5
    no_decay_days: 7
  diversity:
    enabled: true
    strategy: "cap_per_doc"
    max_chunks_per_doc: 3
    mmr_lambda: 0.3
  scenario_boost:
    time_sensitive:
      timeliness_multiplier: 1.3
    regulatory_query:
      authority_multiplier: 1.2
    deep_analysis:
      relevance_multiplier: 1.1
```

### 4.4 路由运行时参数（含 MCP）

```yaml
# Data ID: router_runtime.yaml
router:
  min_match_score: 0.5
  ambiguous_threshold: 0.3
  context_bias_factor: 1.05
  core_weight: 1.0
  extend_weight: 0.6

mcp_integration:
  enable_dynamic_routing: true
  fallback_on_timeout: "ignore"
  global_cache_ttl: 120
  max_parallel_calls: 3
  circuit_breaker_threshold: 5
```

### 4.5 意图理解策略参数

```yaml
# Data ID: query_understanding.yaml
intent:
  confidence_threshold: 0.65
  max_sub_questions: 3
rewrite:
  max_output_queries: 3
  overlap_threshold: 0.9
fallback_patterns:
  - pattern: ".*(估值|收益率|久期|评级|违约|利差|BP).*"
    intent: "FACTUAL"
  - pattern: ".*(分析|展望|策略|建议|怎么看|趋势).*"
    intent: "ANALYTICAL"
```

### 4.6 混合检索运行时参数（📌 v5.1 含 Rerank 配置）

```yaml
# Data ID: search_hybrid.yaml
hybrid:
  # ========== 召回参数 ==========
  semantic_top_k: 50
  keyword_top_k: 50
  fuzzy_top_k: 20
  
  # ========== RRF 融合参数 ==========
  rrf:
    k: 60
    semantic_weight: 0.6
    keyword_weight: 0.3
    fuzzy_weight: 0.1
  
  # ========== 标签筛选 ==========
  tag_filter:
    match_mode: "exact"
  
  # ========== 宽松召回 ==========
  fallback:
    min_results: 3
    relax_factor: 2.0
    max_retries: 1

  # ========== 📌 v5.1：Rerank 精排配置 ==========
  rerank:
    enabled: false                      # 总开关（默认关闭）
    min_candidates_to_trigger: 10
    max_candidates_to_rerank: 50
    model_provider: "cohere"
    model_name: "rerank-v3.5"
    api_key: "${RERANK_API_KEY}"
    base_url: "https://api.cohere.ai/v1/rerank"
    timeout_ms: 500
    fallback_on_error: true
    fallback_on_timeout: true
```

### 4.7 角色个性化排序权重（📌 v6.0 新增）

```yaml
# Data ID: ranking_user_weights.yaml
role_weights:
  基金经理:
    w_relevance: 0.45
    w_authority: 0.25
    w_timeliness: 0.30
  风控专员:
    w_relevance: 0.40
    w_authority: 0.40
    w_timeliness: 0.20
  信评分析师:
    w_relevance: 0.50
    w_authority: 0.30
    w_timeliness: 0.20
  量化研究员:
    w_relevance: 0.55
    w_authority: 0.15
    w_timeliness: 0.30
  合规审核:
    w_relevance: 0.30
    w_authority: 0.55
    w_timeliness: 0.15
  运营人员:
    w_relevance: 0.60
    w_authority: 0.20
    w_timeliness: 0.20
  交易员:
    w_relevance: 0.40
    w_authority: 0.20
    w_timeliness: 0.40
  产品经理:
    w_relevance: 0.50
    w_authority: 0.30
    w_timeliness: 0.20
default_weights:
  w_relevance: 0.50
  w_authority: 0.30
  w_timeliness: 0.20
```


## 五、对象存储（MinIO/OSS）

### 原始文件路径
```
/{source_kb}/{doc_uuid}/{version}/原始文件.{ext}
```

### 解析缓存路径
```
/parsed_cache/{doc_uuid}/elements.json
```


## 六、跨库一致性保障

| 场景 | 处理方式 |
| :--- | :--- |
| **写入顺序** | 先写MySQL（状态0）→ 再写ES → 再写Milvus → 更新MySQL状态为1 |
| **异常恢复** | 定时补偿任务扫描 `status IN (0,2)` 且超过5分钟的记录，重试最多3次 |
| **逻辑删除** | 软删除（`is_active=0`），三库同步标记 |
| **关键ID** | `chunk_uuid` 是三库连接的唯一桥梁 |
| **父子块关系** | 子块通过 `parent_chunk_id` 关联父块；父块仅存MySQL |
| **溯源链路** | `kb_chunks.source_elem_ids` → `elements.json` → 提取 `bbox` 坐标 |
| **路由日志闭环** | `kb_router_logs.trace_id` ↔ `kb_user_feedback.trace_id` |
| **查询理解闭环** | `kb_query_logs.trace_id` ↔ `kb_router_logs.trace_id` ↔ `kb_user_feedback.trace_id` |
| **召回调优闭环** | `kb_search_logs.trace_id` ↔ `kb_query_logs.trace_id` ↔ `kb_user_feedback.trace_id` |
| **排序调优闭环** 📌 v6.0 | `kb_rank_logs.trace_id` ↔ `kb_search_logs.trace_id` ↔ `kb_user_feedback.trace_id` |
| **MCP配置热加载** | `kb_mcp_providers` 和 `kb_mcp_weight_mapping` 定时刷新（每5分钟） |
| **MCP缓存失效** | 定时清理 `kb_mcp_cache` 中已过期的记录 |


## 七、📌 版本变更汇总（v5.0 → v6.1）

| 变更项 | 版本 | 类型 | 紧迫度 |
| :--- | :--- | :--- | :--- |
| `kb_search_logs` 新增 4 个 Rerank 字段 | v5.1 | 修改 | **高** |
| Nacos `search_hybrid.yaml` 新增 `rerank` 配置段 | v5.1 | 新增 | 中 |
| **`kb_rank_logs` 表** | **v6.0** | **新增** | **高** |
| **`kb_sessions` 增加 `user_role` 字段** | **v6.0** | 修改 | **高** |
| **`kb_user_feedback` 增加 `rank_helpful` / `rank_snapshot`** | **v6.0** | 修改 | **高** |
| **Nacos `ranking_user_weights.yaml`** | **v6.0** | 新增 | **高** |
| **Nacos `ranking_weights` 扩展（diversity/time_decay/scenario_boost）** | **v6.0** | 修改 | 中 |


> **文档说明**：本 v6.1 版本是将 v5.1（召回层 + Rerank）和 v6.0（排序层 + 个性化）完整融合后的统一数据模型。Rerank 作为任务6内部可选步骤（默认关闭），其追踪字段完全独立于排序日志，两套可观测性体系并行不悖。此版本可直接进入开发。

如需基于此完整模型继续输出**任务7的接口设计（含Rerank兼容说明）** 或 **表DDL合并脚本**，请告知。