好的，以下是整合了 **MCP 动态元数据集成** 修正方案后的完整数据模型文档。

---

# 数据模型设计（完整版 v3.0）

> **版本更新说明**：本次新增 MCP（模型上下文协议）动态元数据集成能力，将实时业务信号纳入路由决策，实现从“静态关键词匹配”到“实时上下文感知路由”的升级。所有新增/修改内容已用 **📌** 标注。


## 一、MySQL 数据模型

一共需要 **11张** 核心表（原8张 + 新增3张 MCP 相关表）。

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
| `source_elem_ids` | json | NULL | 引用的解析器原始元素ID列表，如["elem_0001","elem_0002"]，用于溯源高亮 |
| `page_numbers` | varchar(64) | NULL | 该块覆盖的页码范围（如"3-5"或"3"），仅用于排序展示 |
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


### 1.7 用户反馈表：`kb_user_feedback`

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `trace_id` | varchar(64) | INDEX | 全链路追踪ID |
| `query_text` | varchar(500) | | 用户问题 |
| `thumbs_up` | tinyint | | 1-点赞 / 0-点踩 |
| `comment` | varchar(500) | | 用户备注 |
| **`router_snapshot`** 📌 | json | NULL | **路由决策快照**：{routed_kbs, clarity_flag, confidence, mcp_enhanced} |
| **`selected_kb`** 📌 | varchar(32) | NULL | **用户实际浏览的知识库**（歧义引导场景下用户点击选择的结果） |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |


### 1.8 路由日志表：`kb_router_logs`（📌 已扩展 MCP 追踪字段）

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
| **`mcp_triggered`** 📌 | tinyint | DEFAULT 0 | **本次路由是否触发了 MCP 动态干预** |
| **`mcp_signals`** 📌 | json | NULL | **MCP 返回的原始信号快照**，如{"risk_level":"high","bond_code":"123456"} |
| **`mcp_weight_applied`** 📌 | json | NULL | **MCP 信号映射为权重的详细记录**，如{"风控知识库": "+0.9"} |
| `selected_kb` | varchar(32) | NULL | 用户最终点击/选择的KB（歧义场景由前端回传） |
| `is_resolved` | tinyint | DEFAULT 0 | 0-待确认 / 1-用户完成检索 |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |

**索引**：
```sql
CREATE INDEX idx_trace_id ON kb_router_logs(trace_id);
CREATE INDEX idx_created_at ON kb_router_logs(created_at);
CREATE INDEX idx_clarity ON kb_router_logs(clarity_flag);
CREATE INDEX idx_mcp_triggered ON kb_router_logs(mcp_triggered);
```


### 1.9 MCP 服务提供者配置表：`kb_mcp_providers`（📌 新增）

> **📌 新增原因**：路由模块需要动态拉取实时元数据（如风险评级、监管动态、舆情标签），必须知道 MCP 服务的地址、鉴权方式和超时配置。  
> **紧迫度：高** —— 若不加，MCP 动态路由无法落地。

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


### 1.10 MCP 信号→路由权重映射表：`kb_mcp_weight_mapping`（📌 新增）

> **📌 新增原因**：MCP 返回的实时信号（如 `risk_level: high`）必须能映射为具体知识库的加减分权重，否则无法驱动路由决策。此表即“动态字典”的核心。  
> **紧迫度：高** —— 路由引擎依赖此表将信号转化为可计算的权重。

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


### 1.11 MCP 实时信号缓存表：`kb_mcp_cache`（📌 新增）

> **📌 新增原因**：路由模块高频调用 MCP 服务存在网络开销和延迟风险。对同一实体（如“贵州城投债”）的重复查询应使用缓存，避免打爆 MCP 服务或拖垮路由响应时间。  
> **紧迫度：中** —— 初期可禁用缓存调试，但生产环境建议开启。

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

### 索引名：`kb_chunks_v1`（无变化）

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

### Collection 名：`kb_embeddings_v1`（无变化）

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `chunk_uuid` | VARCHAR(64) | PRIMARY KEY | 与ES/MySQL保持一致 |
| `embedding` | FLOAT_VECTOR(1536) | NOT NULL | 向量 |
| `source_kb` | VARCHAR(32) | PARTITION KEY | 分区键 |
| `is_active` | BOOL | | 是否生效 |


## 四、配置中心（Nacos）存储内容

### 4.1 路由规则（静态字典，保留）

```yaml
routing:
  - keywords: ["净值", "收益率", "夏普"]
    target_kbs: ["基金知识库"]
  - keywords: ["评级", "违约", "利差"]
    target_kbs: ["信评知识库", "固收投研知识库"]
```

### 4.2 同义词表（保留）

```yaml
synonyms:
  - ["ROE", "净资产收益率"]
  - ["久期", "持续期"]
```

### 4.3 排序权重（保留）

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

### 4.4 路由运行时参数（📌 已扩展 MCP 配置）

```yaml
# Data ID: router_runtime.yaml
router:
  min_match_score: 0.5
  ambiguous_threshold: 0.3
  context_bias_factor: 1.05
  core_weight: 1.0
  extend_weight: 0.6

# 📌 新增：MCP 动态路由全局配置
mcp_integration:
  enable_dynamic_routing: true
  fallback_on_timeout: "ignore"      # ignore / use_last_cached / fail_open
  global_cache_ttl: 120              # 全局缓存 TTL（秒）
  max_parallel_calls: 3              # 同时调用的最大 MCP 服务数
  circuit_breaker_threshold: 5       # 连续失败次数触发熔断
```


## 五、对象存储（MinIO/OSS）

### 5.1 原始文件路径（无变化）
```
/{source_kb}/{doc_uuid}/{version}/原始文件.{ext}
```

### 5.2 解析缓存路径（无变化）
```
/parsed_cache/{doc_uuid}/elements.json
```


## 六、跨库一致性保障（📌 新增 MCP 相关）

| 场景 | 处理方式 |
| :--- | :--- |
| **写入顺序** | 先写MySQL（状态置为0） → 再写ES → 再写Milvus → 最后更新MySQL状态为1（已索引），并填充`indexed_at` |
| **异常恢复** | 定时补偿任务扫描MySQL中`status IN (0,2)`且`created_at`超过5分钟的记录，重新触发索引流程；`retry_count`超过3次则停止重试，等待人工介入 |
| **逻辑删除** | 软删除（`is_active=0`），三库同步标记，不物理删除 |
| **关键ID** | `chunk_uuid`是连接三库的唯一桥梁，必须完全一致 |
| **父子块关系** | 子块通过`parent_chunk_id`关联父块；父块不写入ES和Milvus，仅存MySQL |
| **溯源链路** | `kb_chunks.source_elem_ids` → 对象存储 `elements.json` → 提取 `bbox` 坐标 → 前端高亮 |
| **路由日志闭环** | `kb_router_logs.trace_id` ↔ `kb_user_feedback.trace_id`，实现“路由决策 → 检索召回 → 用户反馈”全链路可追溯 |
| **📌 MCP 配置热加载** | `kb_mcp_providers` 和 `kb_mcp_weight_mapping` 变更后，路由模块通过 `/reload` 或定时任务（每5分钟）刷新内存缓存，无需重启服务 |
| **📌 MCP 缓存失效** | 定时清理 `kb_mcp_cache` 中 `expires_at` 已过期的记录；缓存命中率低于 10% 时自动告警 |


## 七、📌 新增内容汇总（供领导快速审阅）

| 变更项 | 类型 | 紧迫度 | 核心原因 |
| :--- | :--- | :--- | :--- |
| `kb_mcp_providers` 表 | 新增 | **高** | 管理 MCP 元数据服务的连接配置，动态路由的基础 |
| `kb_mcp_weight_mapping` 表 | 新增 | **高** | 将 MCP 实时信号映射为路由权重，实现“纳入字典” |
| `kb_mcp_cache` 表 | 新增 | 中 | 缓存 MCP 查询结果，保证路由模块响应延迟可控 |
| `kb_router_logs` 增加 MCP 追踪字段 | 修改 | **高** | 记录 MCP 干预痕迹，用于召回测评与效果评估 |
| `kb_user_feedback.router_snapshot` 增加 mcp_enhanced | 修改 | 中 | 用户反馈可追溯到是否受 MCP 影响 |
| Nacos `mcp_integration` 配置组 | 新增 | 中 | 动态路由的全局开关与熔断降级策略 |


以上即为整合了 MCP 动态元数据集成方案的完整数据模型文档。你可以直接使用或根据业务场景微调字段长度/类型。需要我继续生成修正后的 **《接口设计文档》** 吗？