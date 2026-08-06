# 数据模型设计（完整版 v5.0）

> **版本更新说明**：
> - **v4.0 → v5.0**：整合任务4的 MCP（模型上下文协议）动态元数据集成方案与任务6的召回日志方案。将路由模块从“静态关键词匹配”升级为“实时上下文感知路由”，同时完善召回环节的可观测性。新增MCP服务配置表、信号权重映射表及缓存表，扩展路由日志与用户反馈表以记录MCP干预痕迹，并在Nacos中新增MCP动态路由配置段。所有新增/修改内容已用 **📌 v5.0** 标注。
> - **v3.0 → v4.0**：针对任务6（召回策略：混合检索 + 标签筛选 + 投研融合），新增召回日志表以记录多路召回细节（语义/关键词/模糊各通路命中数、RRF参数、延迟、宽松触发），增强用户反馈表对召回环节的追溯能力，并新增Nacos配置组用于混合检索运行时参数热更新。
> - **v2.0 → v3.0**：针对任务5（意图识别、拆解与改写），新增会话上下文表与查询处理日志表，强化用户反馈表对改写环节的追溯能力，并新增Nacos配置组用于意图理解策略热更新。
> - **v1.0 → v2.0**：新增路由日志表与反馈表增强字段，用于支撑全局字典与路由模块的可观测性及端到端闭环。

数据模型设计服务于两个核心场景：**写入**（文档预处理后入库）和**读取**（检索时快速过滤和召回）。整体分为五块：

1. **MySQL**：存储元数据、切块原文、标签、版本、反馈、路由日志、会话上下文、查询日志、**召回日志（v4.0）**、**MCP配置与映射（v5.0）** 等结构化数据
2. **Elasticsearch**：存储切块文本，用于关键词检索（BM25）和元数据过滤
3. **向量数据库（Milvus）** ：存储切块向量，用于语义检索
4. **配置中心（Nacos/Apollo）** ：存储全局字典、排序权重、路由运行时参数（含 **MCP动态路由配置 v5.0**）、意图理解策略、混合检索参数等热配置
5. **对象存储（MinIO/OSS）** ：存储原始文档文件及解析缓存


## 一、MySQL 数据模型

一共需要 **14张** 核心表（原10张 + 新增1张召回日志 + 新增3张MCP相关表）。

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

> **状态机说明**：
> - **0（待处理/处理中）**：文档已录入但尚未完成入库，或正在执行解析/切块/写入管道。补偿任务会扫描该状态且`created_at`超过5分钟的记录进行重试。
> - **1（已索引）**：三库（MySQL+ES+Milvus）全部写入成功，可被检索。
> - **2（异常）**：管道处理失败，或重试超过3次仍失败，需人工介入排查`last_error`字段。

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
| `source_elem_ids` | json | NULL | 引用的解析器原始元素ID列表，如["elem_0001","elem_0002"]，用于溯源高亮 |
| `page_numbers` | varchar(64) | NULL | 该块覆盖的页码范围（如"3-5"或"3"），仅用于排序展示，不用于精确坐标定位 |
| `is_active` | tinyint | DEFAULT 1 | 1-生效 / 0-已废弃 |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |

**索引**：
```sql
CREATE INDEX idx_doc_id ON kb_chunks(doc_id);
CREATE INDEX idx_parent_chunk ON kb_chunks(parent_chunk_id);
```

> **父子块与溯源设计**：
> - **父块（PARENT）**：作为返回给用户的完整上下文单元，`parent_chunk_id`为`NULL`。
> - **子块（CHILD）**：用于向量检索的最小单元（约256 tokens），通过`parent_chunk_id`指向父块。
> - **`source_elem_ids`**：存储该切块引用了解析器输出的哪些原始元素（`elem_id`）。当用户需要高亮原文时，后端根据`doc_uuid`从对象存储加载`elements.json`，提取对应`elem_id`的物理坐标（`page_num` + `bbox`）返回前端。

### 1.3 标签表：`kb_tags`

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `tag_name` | varchar(64) | UNIQUE NOT NULL | 标签名称，如"新能源" |
| `tag_category` | varchar(32) | | industry / topic / risk / region / auto_extracted |
| `is_active` | tinyint | DEFAULT 1 | 是否启用 |

> **说明**：`tag_category` 中的 `auto_extracted` 用于标注路由模块或MCP自动提取并补齐的标签。

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

### 1.7 用户反馈表：`kb_user_feedback`（📌 v2.0 + v3.0 + v4.0 + v5.0 融合）

> **设计说明**：本表融合了路由层（v2.0）、理解层（v3.0）、召回层（v4.0）和MCP动态干预（v5.0）的全链路反馈字段，实现端到端问题定位。

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `trace_id` | varchar(64) | INDEX | 全链路追踪ID |
| **`original_query`** 📌 v3.0 | text | NULL | **用户原始问句（冗余，便于直接查看）** |
| `query_text` | varchar(500) | | 用户问题（已废弃，保留兼容） |
| `thumbs_up` | tinyint | | 1-点赞 / 0-点踩 |
| **`rewritten_query_used`** 📌 v3.0 | text | NULL | **当时系统使用的改写Query（取列表第一条，即实际用于检索的那条）** |
| **`rewrite_helpful`** 📌 v3.0 | tinyint | NULL | **用户点踩时可选：改写是否有帮助（1=有帮助，0=改写误导）** |
| `comment` | varchar(500) | | 用户备注 |
| **`router_snapshot`** 📌 v2.0 + 📌 v5.0 | json | NULL | **路由决策快照**：`{routed_kbs, clarity_flag, confidence, mcp_enhanced}`，其中 `mcp_enhanced` 标识本次路由是否受MCP动态信号干预 |
| **`selected_kb`** 📌 v2.0 | varchar(32) | NULL | **用户实际浏览的知识库（歧义引导场景下用户点击选择的结果）** |
| **`search_snapshot`** 📌 v4.0 | json | NULL | **召回快照**：记录RRF融合后的Top-K `chunk_uuid`列表及各通路得分，用于回溯"召回是否遗漏" |
| **`retrieved_chunks`** 📌 v4.0 | json | NULL | **实际返回给用户的 `chunk_uuid` 列表（前N条）** |
| **`search_helpful`** 📌 v4.0 | tinyint | NULL | **用户点踩时可选：召回的内容是否相关？（1=相关，0=完全不相关）** |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |

> **各版本修改原因**：
> - **v2.0**（路由层）：用户反馈必须能追溯到“当时的路由决策”，才能区分是“路由错了”还是“检索召回了差文档”还是“排序不好”。
> - **v3.0**（理解层）：用户点踩可能因为“改写错了”（如“收益”被改成了“到期收益率”）。必须记录当时的改写结果，否则无法区分“检索没召回”和“改写把语义带偏了”。
> - **v4.0**（召回层）：用户点踩还可能因为“语义没召回相关段落”或“关键词没匹配上”。新增召回快照字段可将反馈精细定位到“召回”这一层。
> - **v5.0**（MCP层）：用户点踩可能因为路由受MCP信号误导（如错误的风险信号导致路由到错误知识库）。在 `router_snapshot` 中增加 `mcp_enhanced` 标识，便于回溯MCP影响。

**DDL 示例（v4.0 + v5.0 新增字段）**：
```sql
ALTER TABLE kb_user_feedback 
ADD COLUMN search_snapshot json NULL COMMENT '召回快照（RRF融合后的Top-K列表及得分）',
ADD COLUMN retrieved_chunks json NULL COMMENT '返回的chunk_uuid列表',
ADD COLUMN search_helpful tinyint NULL COMMENT '召回内容是否相关';
-- router_snapshot 字段本身已存在，其 JSON 结构扩展支持 mcp_enhanced 字段，无需 DDL 变更
```

### 1.8 路由日志表：`kb_router_logs`（📌 v2.0 新增 + 📌 v5.0 MCP 追踪扩展）

> **新增原因（v2.0）**：路由模块仅在内存中处理，不落库。但第11周的“召回测评”和“端到端反馈闭环”必须依赖历史路由数据来分析：关键词命中率、歧义率、零命中率（评估MD字典覆盖度），以及用户最终选择是否与路由推荐一致。**紧迫度：高**。
>
> **扩展原因（v5.0）**：MCP动态信号介入路由后，必须记录干预痕迹，否则无法评估“动态路由”是否有效，也无法解释“为何路由做出了与静态字典预期不符的选择”。**紧迫度：高**。

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `trace_id` | varchar(64) | NOT NULL INDEX | 全链路追踪ID，与`kb_user_feedback.trace_id`打通 |
| `user_query` | varchar(500) | NOT NULL | 用户原始问题 |
| `context_kb` | varchar(32) | NULL | 请求时的上下文知识库（若有） |
| `routed_kbs` | json | NOT NULL | 路由返回的候选库列表，如["固收投研","信评"] |
| `clarity_flag` | varchar(16) | NOT NULL | clear / ambiguous / unknown |
| `confidence` | decimal(4,3) | NOT NULL | 置信度（0~1） |
| `matched_keywords` | json | NULL | 命中的关键词列表，如["城投债","利差"] |
| `filter_tags` | json | NULL | 提取的标签，如["城投债"] |
| **`mcp_triggered`** 📌 v5.0 | tinyint | DEFAULT 0 | **本次路由是否触发了 MCP 动态干预** |
| **`mcp_signals`** 📌 v5.0 | json | NULL | **MCP 返回的原始信号快照**，如`{"risk_level":"high","bond_code":"123456","regulator_warning":true}` |
| **`mcp_weight_applied`** 📌 v5.0 | json | NULL | **MCP 信号映射为权重的详细记录**，如`{"风控知识库": "+0.9", "信评知识库": "+0.7"}` |
| `selected_kb` | varchar(32) | NULL | 用户最终点击/选择的KB（歧义场景由前端回传，通过后续trace关联更新） |
| `is_resolved` | tinyint | DEFAULT 0 | 0-待确认 / 1-用户完成检索（通过后续trace关联判定） |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |

**索引**：
```sql
CREATE INDEX idx_trace_id ON kb_router_logs(trace_id);
CREATE INDEX idx_created_at ON kb_router_logs(created_at);
CREATE INDEX idx_clarity ON kb_router_logs(clarity_flag);
CREATE INDEX idx_mcp_triggered ON kb_router_logs(mcp_triggered);
```

### 1.9 会话上下文表：`kb_sessions`（📌 v3.0新增）

> **新增原因**：任务5需要处理多轮对话中的指代消解（如“它现在的估值呢？”中的“它”）。Dify自身虽维护对话历史，但为了在自研核心中实现轻量级上下文感知改写（避免每次调用LLM时传递完整历史，节省Token），需要一张轻量的会话摘要表。**紧迫度：高**。

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `session_id` | varchar(64) | PRIMARY KEY | 会话ID（由Dify传入，对应conversation_id） |
| `user_id` | varchar(64) | NOT NULL | 用户标识 |
| `source_kb` | varchar(32) | NULL | 当前会话主要所在知识库（来自路由） |
| `history_summary` | text | NULL | 历史对话摘要（JSON格式，保留最近3轮QA对，用于指代消解） |
| `last_query` | text | NULL | 上一轮用户原始问句 |
| `last_processed` | text | NULL | 上一轮系统处理后的改写Query列表（JSON） |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |
| `updated_at` | datetime | DEFAULT CURRENT_TIMESTAMP ON UPDATE | 更新时间 |

**索引**：
```sql
CREATE INDEX idx_user_session ON kb_sessions(user_id, updated_at);
CREATE INDEX idx_source_kb ON kb_sessions(source_kb);
```

### 1.10 查询处理日志表：`kb_query_logs`（📌 v3.0新增）

> **新增原因**：现有`kb_router_logs`记录路由决策，但任务5的意图分类、拆解、改写过程需要单独记录。原因有二：一是路由发生在理解之前（先定`source_kb`，再做意图理解）；二是为了后续分析“LLM改写是否有效”与“意图分类是否准确”，必须留存原始输入与最终输出的映射关系，方便做Bad Case分析。**紧迫度：高**。

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `log_id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `trace_id` | varchar(64) | NOT NULL INDEX | 全链路追踪ID（与`kb_router_logs.trace_id`关联） |
| `session_id` | varchar(64) | NULL | 会话ID |
| `original_query` | text | NOT NULL | 用户原始输入 |
| `intent_type` | varchar(32) | NULL | 事实查询(FACTUAL)/分析推理(ANALYTICAL)/数据计算(CALCULATION)/闲聊(CHITCHAT)/无效(INVALID) |
| `confidence` | decimal(4,3) | NULL | 意图分类置信度 |
| `is_decomposed` | tinyint(1) | DEFAULT 0 | 是否触发拆解（仅ANALYTICAL会拆） |
| `sub_questions` | json | NULL | 拆解后的子问题列表，如["信用风险分析","配置建议"] |
| `rewritten_queries` | json | NOT NULL | 最终输出的改写Query列表（供检索使用） |
| `expanded_terms` | json | NULL | 扩展的同义词/相关词列表 |
| `llm_call_duration_ms` | int | NULL | 本次处理LLM总耗时（毫秒） |
| `fallback_triggered` | tinyint(1) | DEFAULT 0 | 是否触发了降级兜底（规则匹配） |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |

**索引**：
```sql
CREATE INDEX idx_trace ON kb_query_logs(trace_id);
CREATE INDEX idx_session ON kb_query_logs(session_id);
CREATE INDEX idx_intent ON kb_query_logs(intent_type);
CREATE INDEX idx_created ON kb_query_logs(created_at);
```

### 1.11 召回日志表：`kb_search_logs`（📌 v4.0新增）

> **📌 新增原因**：任务6（混合检索）涉及多路并行召回（语义/关键词/模糊）、RRF融合排序以及宽松召回（Fallback）策略。若缺失此表，后续做“召回测评”时将无法分析：语义通路 vs 关键词通路的命中覆盖率与重叠率；RRF融合参数（`k`值、各通路权重）是否合理；宽松召回触发频率是否过高；各通路延迟是否在可接受范围内。**紧迫度：高**。

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `trace_id` | varchar(64) | NOT NULL INDEX | 全链路追踪ID（关联 `kb_query_logs` 与 `kb_router_logs`） |
| `session_id` | varchar(64) | NULL | 会话ID |
| `original_query` | text | NOT NULL | 用户原始输入（冗余便于排查） |
| `rewritten_query` | text | NULL | 改写后用于检索的主Query |
| `sub_questions` | json | NULL | 拆解后的子问题列表 |
| `tag_filters` | json | NULL | 请求传入的标签筛选条件，如`["财务分析"]` |
| `entity_constraints` | json | NULL | 请求传入的投研实体代码列表，如`["600519"]` |
| `source_kb` | varchar(32) | NULL | 路由锁定的知识库 |
| `semantic_query` | text | NULL | 实际送给Milvus的查询文本（可能经过向量化） |
| `keyword_query` | text | NULL | 实际送给ES的查询文本（可能是改写后的） |
| `semantic_top_k` | int | DEFAULT 50 | 语义通路召回数配置 |
| `keyword_top_k` | int | DEFAULT 50 | 关键词通路召回数配置 |
| `semantic_hits` | int | NULL | 语义通路实际命中的候选数量 |
| `keyword_hits` | int | NULL | 关键词通路实际命中的候选数量 |
| `fuzzy_hits` | int | NULL | 模糊匹配通路实际命中数（若启用） |
| `overlap_count` | int | NULL | 多路召回之间的重叠文档数 |
| `rrf_params` | json | NULL | 融合参数快照，如`{"k":60, "semantic_weight":0.6, "keyword_weight":0.3, "fuzzy_weight":0.1}` |
| `final_result_count` | int | NULL | 最终返回给上游的Top-K数量 |
| `semantic_latency_ms` | int | NULL | Milvus检索耗时（毫秒） |
| `keyword_latency_ms` | int | NULL | ES检索耗时（毫秒） |
| `fusion_latency_ms` | int | NULL | RRF融合排序耗时（毫秒） |
| `fallback_triggered` | tinyint(1) | DEFAULT 0 | 是否因结果不足触发了宽松召回（如top_k乘2） |
| `status` | tinyint | DEFAULT 1 | 1=成功 / 0=失败 |
| `error_msg` | text | NULL | 异常信息（如有） |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |

**索引**：
```sql
CREATE INDEX idx_trace ON kb_search_logs(trace_id);
CREATE INDEX idx_session ON kb_search_logs(session_id);
CREATE INDEX idx_created ON kb_search_logs(created_at);
CREATE INDEX idx_status ON kb_search_logs(status);
```

### 1.12 MCP 服务提供者配置表：`kb_mcp_providers`（📌 v5.0 新增）

> **📌 新增原因**：路由模块需要动态拉取实时元数据（如风险评级、监管动态、舆情标签），必须知道 MCP 服务的地址、鉴权方式和超时配置。此表是 MCP 动态路由的基础设施。**紧迫度：高**。

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `provider_name` | varchar(64) | UNIQUE NOT NULL | 服务名称，如 "RiskMetadataService" |
| `endpoint_url` | varchar(255) | NOT NULL | MCP 服务地址，如 `http://mcp-internal/query` |
| `auth_type` | varchar(16) | DEFAULT 'none' | 鉴权方式：none / bearer / api_key |
| `auth_credential` | varchar(255) | NULL | 加密存储的凭证（AES-256） |
| `timeout_ms` | int | DEFAULT 500 | 单次调用超时（毫秒） |
| `cache_ttl_seconds` | int | DEFAULT 120 | 该服务返回数据的缓存有效期（秒） |
| `retry_count` | tinyint | DEFAULT 1 | 失败时最大重试次数 |
| `is_active` | tinyint | DEFAULT 1 | 是否启用 |
| `priority` | int | DEFAULT 0 | 调用优先级（数值越大越优先，当多个MCP来源冲突时） |
| `description` | varchar(255) | NULL | 服务描述 |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |
| `updated_at` | datetime | DEFAULT CURRENT_TIMESTAMP ON UPDATE | 更新时间 |

**索引**：
```sql
CREATE INDEX idx_is_active ON kb_mcp_providers(is_active);
```

### 1.13 MCP 信号→路由权重映射表：`kb_mcp_weight_mapping`（📌 v5.0 新增）

> **📌 新增原因**：MCP 返回的实时信号（如 `risk_level: high`）必须能映射为具体知识库的加减分权重，否则无法驱动路由决策。此表即“动态字典”的核心，是领导所说“将元数据构成MCP纳入全局字典”的具体落地。**紧迫度：高**。

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `provider_id` | bigint | FOREIGN KEY NOT NULL | 关联 `kb_mcp_providers.id` |
| `signal_key` | varchar(64) | NOT NULL | MCP 返回的字段名，如 `risk_level` |
| `signal_value` | varchar(64) | NOT NULL | MCP 返回的具体值，如 `high` |
| `target_kb` | varchar(32) | NOT NULL | 目标知识库名称，如 "风控知识库" |
| `weight_delta` | decimal(4,2) | NOT NULL | 动态加减分权重（正数加分，负数减分），如 +0.9 |
| `expire_seconds` | int | DEFAULT 0 | 该映射规则的有效期（秒），0 表示永久有效 |
| `operator` | varchar(16) | DEFAULT 'eq' | 比较运算符：eq / ne / contains / gt / lt |
| `is_active` | tinyint | DEFAULT 1 | 是否启用 |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |
| `updated_at` | datetime | DEFAULT CURRENT_TIMESTAMP ON UPDATE | 更新时间 |

**索引**：
```sql
CREATE INDEX idx_provider_signal ON kb_mcp_weight_mapping(provider_id, signal_key, signal_value);
CREATE INDEX idx_target_kb ON kb_mcp_weight_mapping(target_kb);
```

### 1.14 MCP 实时信号缓存表：`kb_mcp_cache`（📌 v5.0 新增）

> **📌 新增原因**：路由模块高频调用 MCP 服务存在网络开销和延迟风险。对同一实体（如“贵州城投债”）的重复查询应使用缓存，避免打爆 MCP 服务或拖垮路由响应时间。**紧迫度：中**。

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `cache_key` | varchar(128) | UNIQUE NOT NULL | 查询实体哈希（如 `md5("贵州_城投债")`） |
| `provider_id` | bigint | FOREIGN KEY NOT NULL | 关联 `kb_mcp_providers.id` |
| `request_tokens` | json | NOT NULL | 请求时的原始 Token 列表 |
| `response_payload` | json | NOT NULL | MCP 返回的完整元数据快照 |
| `expires_at` | datetime | NOT NULL | 缓存过期时间 |
| `hit_count` | int | DEFAULT 0 | 缓存命中次数（用于后续分析缓存价值） |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |

**索引**：
```sql
CREATE INDEX idx_cache_key ON kb_mcp_cache(cache_key);
CREATE INDEX idx_expires_at ON kb_mcp_cache(expires_at);
```


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

> **说明**：ES索引**只存储子块（CHILD）**，不存储父块（PARENT），以控制索引体积。检索时命中子块后，通过MySQL的`parent_chunk_id`拉取父块完整内容。`tags` 和 `entity_codes` 字段用于任务6的标签筛选与投研实体过滤。


## 三、向量数据库（Milvus）设计

### Collection 名：`kb_embeddings_v1`

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `chunk_uuid` | VARCHAR(64) | PRIMARY KEY | 与ES/MySQL保持一致 |
| `embedding` | FLOAT_VECTOR(1536) | NOT NULL | 向量 |
| `source_kb` | VARCHAR(32) | PARTITION KEY | 分区键 |
| `is_active` | BOOL | | 是否生效 |

**分区设计**：每个知识库独立分区（如`fund`、`credit`、`compliance`），查询时只扫描相关分区。


## 四、配置中心（Nacos）存储内容

全局字典、排序权重、路由运行时参数（含MCP）、意图理解策略及混合检索参数放在配置中心，支持运行时修改无需重启。

### 4.1 路由规则（静态字典）
```yaml
routing:
  - keywords: ["净值", "收益率", "夏普"]
    target_kbs: ["基金知识库"]
  - keywords: ["评级", "违约", "利差"]
    target_kbs: ["信评知识库", "固收投研知识库"]
  - keywords: ["合规", "监管", "适当性"]
    target_kbs: ["合规知识库"]
```
> **说明**：此部分现已迁移至 `global_router.md` 文件（人工维护更友好），Nacos中保留仅作为备选存储源。代码优先读取MD文件。

### 4.2 同义词表
```yaml
synonyms:
  - ["ROE", "净资产收益率"]
  - ["久期", "持续期"]
```

### 4.3 排序权重（检索排序用）
```yaml
ranking_weights:
  alpha_similarity: 0.5
  gamma_source: 0.25
  delta_time: 0.25

source_priority:
  监管公告: 1.2
  内部研报: 1.0
  外部资讯: 0.8
```

### 4.4 路由运行时参数（📌 v2.0 新增 + 📌 v5.0 MCP 扩展）

> **v2.0 新增原因**：将业务字典（MD文件）与运行时阈值参数分离，二者变更频率不同。字典每周由业务方调整，阈值仅在调试/压测时修改，分开存储更清晰。**紧迫度：中**。
>
> **📌 v5.0 扩展原因**：MCP动态路由需要独立配置段，包含全局开关、熔断降级、超时策略等运行时控制参数。

```yaml
# Data ID: router_runtime.yaml
router:
  min_match_score: 0.5          # 最低命中分数阈值，低于此值视为零命中
  ambiguous_threshold: 0.3      # 前两名分数差小于此值，且命中歧义词表时触发引导
  context_bias_factor: 1.05     # 用户当前所在知识库的偏置系数
  core_weight: 1.0              # (核心) 关键词权重
  extend_weight: 0.6            # (扩展) 关键词权重

# 📌 v5.0 新增：MCP 动态路由全局配置
mcp_integration:
  enable_dynamic_routing: true          # 总开关，false时完全走静态字典
  fallback_on_timeout: "ignore"         # ignore / use_last_cached / fail_open
  global_cache_ttl: 120                 # 全局缓存 TTL（秒），覆盖各服务独立配置
  max_parallel_calls: 3                 # 同时调用的最大 MCP 服务数
  circuit_breaker_threshold: 5          # 连续失败次数触发熔断
  circuit_breaker_timeout_seconds: 30   # 熔断后尝试恢复的时间窗口
```

### 4.5 意图理解策略参数（📌 v3.0新增）

> **新增原因**：任务5中的Prompt模板、降级正则、拆解数量上限需要支持热更新。尤其是意图分类的正则兜底规则，业务刚上线时会频繁调整，放在Nacos可避免频繁发版。**紧迫度：中**。

```yaml
# Data ID: query_understanding.yaml
intent:
  confidence_threshold: 0.65        # LLM分类置信度低于此值自动降级为规则匹配
  max_sub_questions: 3              # 最大拆解子问题数

rewrite:
  max_output_queries: 3             # 最终输出检索Query的最大数量
  overlap_threshold: 0.9            # 去重相似度阈值（余弦相似度）

# 降级正则（当LLM超时或报错时，靠正则兜底）
fallback_patterns:
  - pattern: ".*(估值|收益率|久期|评级|违约|利差|BP).*"
    intent: "FACTUAL"
  - pattern: ".*(分析|展望|策略|建议|怎么看|趋势).*"
    intent: "ANALYTICAL"
  - pattern: ".*(计算|加总|平均|统计|汇总).*"
    intent: "CALCULATION"
  - pattern: "^(你好|谢谢|在吗|hi|hello).*"
    intent: "CHITCHAT"
```

### 4.6 混合检索运行时参数（📌 v4.0新增）

> **📌 新增原因**：任务6中RRF（倒数排名融合）的常数`k`、各通路权重（语义/关键词/模糊）、标签匹配模式（exact/any/all）以及宽松召回（Fallback）的阈值，在开发初期需要频繁调参。硬编码会导致发版成本高，放在Nacos可支持运维动态调整，无需重启服务。**紧迫度：中**。

```yaml
# Data ID: search_hybrid.yaml
hybrid:
  # 各通路召回数量
  semantic_top_k: 50
  keyword_top_k: 50
  fuzzy_top_k: 20          # ES 模糊匹配（如 match_phrase_prefix）单独通路
  
  # RRF（倒数排名融合）参数
  rrf:
    k: 60                  # 常数，通常设为 60
    semantic_weight: 0.6   # 语义通路权重
    keyword_weight: 0.3    # 关键词通路权重
    fuzzy_weight: 0.1      # 模糊通路权重
  
  # 标签筛选模式
  tag_filter:
    match_mode: "exact"    # exact / any / all
    # exact: 文档必须包含所有指定标签（精确匹配）
    # any: 文档包含任意一个指定标签即可
    # all: 文档必须同时包含全部指定标签
  
  # 宽松召回（Fallback）策略
  fallback:
    min_results: 3         # 若 RRF 融合后结果少于 3 条，触发宽松召回
    relax_factor: 2.0      # 将 semantic_top_k 和 keyword_top_k 乘以该系数重试
    max_retries: 1         # 宽松召回最多重试 1 次，避免性能雪崩
```


## 五、对象存储（MinIO/OSS）

存储原始文档文件及解析缓存，路径规范如下：

### 5.1 原始文件路径
```
/{source_kb}/{doc_uuid}/{version}/原始文件.{ext}
```

### 5.2 解析缓存路径
```
/parsed_cache/{doc_uuid}/elements.json
```

> **缓存说明**：`elements.json` 是适配器（Adapter）将MinerU原始输出标准化后的产物，包含完整的结构化元素列表（含`elem_id`、`page_num`、`bbox`物理坐标、`chapter_path`、`content_text`等）。切块器直接读取该文件进行语义切分，溯源高亮时也依赖该文件提取物理坐标。建议缓存文件长期保留（至少与文档生命周期一致），避免重复解析。


## 六、跨库一致性保障

| 场景 | 处理方式 |
| :--- | :--- |
| **写入顺序** | 先写MySQL（状态置为0） → 再写ES → 再写Milvus → 最后更新MySQL状态为1（已索引），并填充`indexed_at` |
| **异常恢复** | 定时补偿任务扫描MySQL中`status IN (0,2)`且`created_at`超过5分钟的记录，重新触发索引流程；`retry_count`超过3次则停止重试，等待人工介入 |
| **逻辑删除** | 软删除（`is_active=0`），三库同步标记，不物理删除 |
| **关键ID** | `chunk_uuid`是连接三库的唯一桥梁，必须完全一致 |
| **父子块关系** | 子块通过`parent_chunk_id`关联父块；父块不写入ES和Milvus，仅存MySQL |
| **溯源链路** | `kb_chunks.source_elem_ids` → 对象存储 `elements.json` → 提取 `bbox` 坐标 → 前端高亮 |
| **路由日志闭环** 📌 v2.0 | `kb_router_logs.trace_id` ↔ `kb_user_feedback.trace_id`，实现“路由决策 → 检索召回 → 用户反馈”全链路可追溯 |
| **查询理解闭环** 📌 v3.0 | `kb_query_logs.trace_id` ↔ `kb_router_logs.trace_id` ↔ `kb_user_feedback.trace_id`，实现“路由 → 意图理解 → 检索 → 反馈”完整链路可追溯 |
| **召回调优闭环** 📌 v4.0 | `kb_search_logs.trace_id` ↔ `kb_query_logs.trace_id` ↔ `kb_user_feedback.trace_id`，实现“路由 → 意图理解 → 多路召回 → RRF融合 → 反馈”全链路可追溯。`kb_search_logs` 中的 `rrf_params`、各通路 `_hits`、`overlap_count`、`fallback_triggered` 字段专门服务于第11周的召回测评。 |
| **MCP 配置热加载** 📌 v5.0 | `kb_mcp_providers` 和 `kb_mcp_weight_mapping` 变更后，路由模块通过 `/reload` 或定时任务（每5分钟）刷新内存缓存，无需重启服务。MCP动态路由能力因此具备“纳入全局字典”的实时性。 |
| **MCP 缓存失效** 📌 v5.0 | 定时清理 `kb_mcp_cache` 中 `expires_at` 已过期的记录；缓存命中率低于 10% 时自动告警，提示可能需调整 TTL 或检查 MCP 服务可用性。 |


## 七、📌 版本变更汇总（v1.0 → v5.0）

| 变更项 | 版本 | 类型 | 紧迫度 | 核心原因 |
| :--- | :--- | :--- | :--- | :--- |
| `kb_router_logs` 表 | v2.0 | 新增 | **高** | 路由效果评估的数据基石，上线即需采集 |
| `kb_user_feedback` 增加 `router_snapshot` / `selected_kb` | v2.0 | 修改 | **高** | 让用户反馈可追溯到路由决策，区分问题归属 |
| Nacos `router_runtime` 配置组 | v2.0 | 新增 | 中 | 将运行时阈值与业务字典分离，便于运维调优 |
| `kb_tags.tag_category` 增加 `auto_extracted` | v2.0 | 修改 | 低（增强） | 支持路由自动补齐标签，提升过滤准确率 |
| `kb_sessions` 表 | v3.0 | 新增 | **高** | 支撑多轮指代消解，降低LLM重复传入历史的Token消耗 |
| `kb_query_logs` 表 | v3.0 | 新增 | **高** | 记录“意图-拆解-改写”全过程，是Bad Case分析和检索评测的数据基石 |
| `kb_user_feedback` 增加 3 个字段（理解层） | v3.0 | 修改 | **高** | 必须区分“改写环节”是否出错，否则端到端闭环无法定位根因 |
| Nacos `query_understanding.yaml` | v3.0 | 新增 | 中 | 将降级规则与Prompt策略解耦，支持运维随时调整兜底逻辑 |
| `kb_search_logs` 表 | v4.0 | 新增 | **高** | 记录多路召回细节（各通路命中数、RRF参数、延迟、宽松触发），是召回调优与评测的数据基石 |
| `kb_user_feedback` 增加 3 个字段（召回层） | v4.0 | 修改 | **高** | 让用户反馈可追溯到“召回”环节，区分“没召回”和“召回了但排序差”，定位根因 |
| Nacos `search_hybrid.yaml` 配置组 | v4.0 | 新增 | 中 | RRF权重、top_k、Fallback阈值需要运行时动态调参，避免频繁发版 |
| **`kb_mcp_providers` 表** | **v5.0** | **新增** | **高** | 管理MCP元数据服务的连接配置，是MCP动态路由的基础设施 |
| **`kb_mcp_weight_mapping` 表** | **v5.0** | **新增** | **高** | 将MCP实时信号映射为路由权重，是“将元数据构成MCP纳入全局字典”的核心落地 |
| **`kb_mcp_cache` 表** | **v5.0** | **新增** | 中 | 缓存MCP查询结果，保证路由模块响应延迟可控 |
| **`kb_router_logs` 增加 MCP 追踪字段** | **v5.0** | 修改 | **高** | 记录MCP干预痕迹（`mcp_triggered`、`mcp_signals`、`mcp_weight_applied`），用于召回测评与效果评估 |
| **`kb_user_feedback.router_snapshot` 增加 `mcp_enhanced`** | **v5.0** | 修改 | 中 | 用户反馈可追溯到是否受MCP影响，完善端到端定位能力 |
| **Nacos `mcp_integration` 配置段** | **v5.0** | 新增 | 中 | MCP动态路由的全局开关、熔断降级与缓存策略，支持运维热调整 |


> **文档说明**：本 v5.0 版本已将任务4（MCP动态路由）、任务5（意图理解与改写）、任务6（混合检索与召回）的数据模型完整融合，覆盖从“路由前动态信号注入”到“召回后可观测性”的全链路。如需继续生成融合后的接口设计文档，请告知。