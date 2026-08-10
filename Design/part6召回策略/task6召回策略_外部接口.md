好的，基于我们确认的 **“Rerank 模型作为任务6内部可选步骤”** 的架构决策，我为您修正任务6的接口设计文档。

修正的核心原则是：**对外接口完全不变，对内流程增加 Rerank 可选步骤**。调用方（Dify 编排层、任务5）**完全无感知**，任务7的输入契约也保持不变。


# 任务6 召回策略 — 接口设计文档（v1.1）

> **版本**：v1.1  
> **状态**：已定稿  
> **适用对象**：Dify 编排层开发者、后端开发人员  
> **变更说明**：v1.1 在 v1.0 基础上，将 Rerank 模型（Cross-Encoder）作为任务6内部可选步骤纳入架构，对外接口保持不变。Rerank 默认关闭，通过 Nacos 配置热开启。


## 一、概述

### 1.1 定位

任务6（召回策略）是知识库查询链路中的核心环节。本服务接收上游（任务5）处理后的查询信息，执行多路召回（语义检索、关键词检索、模糊匹配），**可选择性地通过 Rerank 模型（Cross-Encoder）进行深度语义精排**，最终输出候选 Chunk 列表供下游任务7排序。

### 1.2 任务6内部完整流程（含 Rerank）

```
用户 Query
    ↓
┌─────────────────────────────────────────────────────────────────┐
│                      任务6 召回策略（内部流程）                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 【多路并行召回】                                             │
│     ├── 语义检索（Milvus）  → Top 100                           │
│     ├── 关键词检索（ES）    → Top 100                           │
│     └── 模糊匹配（ES）      → Top 20                            │
│                                                                 │
│  2. 【结果合并与去重】→ 候选池（去重后约 120 条）                  │
│                                                                 │
│  3. 【RRF 粗筛】→ 将多路排名融合，截断至 Top N（默认 50）          │
│                                                                 │
│  4. 【可选：Rerank 模型精排】← Nacos 开关控制，默认关闭            │
│     ├── 调用 Cohere / BGE-reranker 等 Cross-Encoder 模型        │
│     ├── 输入：(Query, 每个候选块内容)                           │
│     └── 输出：每个块获得统一的 rerank_score（0~1）               │
│          ⚠️ 启用时：recall_score = rerank_score                 │
│          ⚠️ 关闭时：recall_score = rrf_score                    │
│                                                                 │
│  5. 【标签与投研实体过滤】→ 最终候选集（Top 50）                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
    ↓
输出给 任务7（排序加权）
```

### 1.3 上下游关系

```
┌─────────┐    ┌──────────────┐    ┌─────────────┐    ┌─────────────┐
│  Dify   │───▶│  任务5:意图   │───▶│ 任务6:召回   │───▶│ 任务7:排序   │
│ 编排层  │    │  理解/改写    │    │  (本服务)    │    │             │
└─────────┘    └──────────────┘    └─────────────┘    └─────────────┘
                                       │
                                       │ 内部可选 Rerank（Cross-Encoder）
                                       ▼
                                  ┌─────────────┐
                                  │ Rerank模型   │
                                  │ (Nacos开关)  │
                                  └─────────────┘
```


## 二、API 接口规范

### 2.1 基础信息

| 项目 | 说明 |
| :--- | :--- |
| **Base URL** | `http://{your-search-service}/api/v1` |
| **认证方式** | API Key（Bearer Token），通过环境变量 `SEARCH_API_KEY` 配置 |
| **超时** | 建议 5s（含内部 Rerank 调用，如启用） |
| **响应格式** | JSON |

### 2.2 核心接口：`POST /search`

> **接口不变性声明**：本接口的请求/响应结构与 v1.0 完全一致。Rerank 模型的引入是**内部实现细节**，调用方无需感知。

#### 请求头

```
Authorization: Bearer {search_api_key}
Content-Type: application/json
```

#### 请求体（Request Body）

| 字段 | 类型 | 必填 | 描述 |
| :--- | :--- | :--- | :--- |
| `request_id` | string | 是 | 全链路追踪 ID |
| `session_id` | string | 否 | 会话 ID |
| `query` | string | 是 | 用户原始查询（或改写后的主查询） |
| `query_embedding` | array[float] | 是 | 查询向量（1536 维） |
| `rewritten_query` | string | 否 | 改写后的查询 |
| `sub_questions` | array[string] | 否 | 拆解后的子问题列表 |
| `intent` | object | 否 | 意图分类结果 |
| `source_kb` | string | 否 | 路由指定的知识库名称 |
| `tag_filters` | array[string] | 否 | 标签筛选列表 |
| `tag_match_mode` | string | 否 | 标签匹配模式：`exact`/`any`/`all`（默认 `exact`） |
| `entity_codes` | array[string] | 否 | 投研实体代码列表 |
| `entity_expand` | boolean | 否 | 是否展开关联实体（默认 `false`） |
| `top_k` | integer | 否 | 最终返回结果数（默认 20，最大 100） |
| `semantic_top_k` | integer | 否 | 语义通路召回数量（默认 50） |
| `keyword_top_k` | integer | 否 | 关键词通路召回数量（默认 50） |
| `debug` | boolean | 否 | 是否返回调试信息（默认 `false`） |

> **注**：`tag_filters` 与 `entity_codes` 字段为标准化后的过滤条件。其值可来源于上游路由模块（任务4）通过 MCP（模型上下文协议）动态生成的增强信号，也可来源于人工配置的静态字典。召回服务（任务6）仅消费标准化后的最终值，不关注其生成逻辑。

#### 响应体（Response Body）

**成功（HTTP 200）**：

| 字段 | 类型 | 描述 |
| :--- | :--- | :--- |
| `code` | integer | 0 表示成功 |
| `message` | string | 提示信息 |
| `data` | object | 结果数据 |
| `data.request_id` | string | 回传请求 ID |
| `data.results` | array[object] | 召回结果列表 |
| `data.results[].chunk_uuid` | string | Chunk 唯一标识 |
| `data.results[].doc_id` | integer | 文档 ID |
| `data.results[].content_text` | string | Chunk 文本内容 |
| `data.results[].parent_chunk_id` | string | 父块 ID |
| `data.results[].source_kb` | string | 来源知识库 |
| `data.results[].tags` | array[string] | 标签列表 |
| **`data.results[].recall_score`** | float | **语义相关性分数（0~1）**：Rerank 启用时为 `rerank_score`，关闭时为 `rrf_score` |
| `data.results[].scores` | object | 各通路原始得分（调试用） |
| `data.results[].found_in` | array[string] | 命中的通路名称列表 |
| `data.results[].metadata` | object | 文档元数据 |
| `data.total_candidates` | integer | 融合前候选总数 |
| `data.fusion_stats` | object | 融合统计信息 |
| **`data.rerank_info`** | object | **Rerank 执行信息（新增）** |
| `data.rerank_info.enabled` | boolean | 本次请求是否启用 Rerank |
| `data.rerank_info.model` | string | 使用的 Rerank 模型名称（如启用） |
| `data.rerank_info.input_count` | integer | 送入 Rerank 的候选数量（如启用） |
| `data.rerank_info.latency_ms` | integer | Rerank 调用耗时（如启用） |
| `data.latency_ms` | integer | 服务总耗时（毫秒） |
| `data.fallback_triggered` | boolean | 是否触发宽松召回 |

> **字段语义说明**：
> - `recall_score` 是任务7的输入字段，**无论 Rerank 是否启用，字段名不变**，保证下游无感知。
> - `rerank_info` 是新增的**可观测性字段**，供调试和监控使用，不影响任务7逻辑。


## 三、错误码

| 错误码 | 说明 | 处理建议 |
| :--- | :--- | :--- |
| 0 | 成功 | - |
| 4001 | 请求参数错误（缺少必填字段、embedding 维度不匹配等） | 检查请求体 |
| 4002 | 标签不存在 | 校验标签名称 |
| 4003 | 实体代码不存在 | 校验实体代码 |
| 5001 | 语义检索服务异常（Milvus 不可用） | 重试或降级 |
| 5002 | 关键词检索服务异常（ES 不可用） | 重试或降级 |
| 5003 | 融合排序异常 | 检查日志 |
| 5004 | 超时（>3s，含 Rerank） | 缩小 `top_k`、关闭 Rerank 或重试 |
| **5005** | **Rerank 模型调用异常（已自动降级为 RRF）** | **检查 Rerank 服务配置，不影响业务** |


## 四、Rerank 配置说明（Nacos）

Rerank 相关配置位于 Nacos 的 `search_hybrid.yaml` 中，支持运行时热更新：

```yaml
hybrid:
  # ... 原有配置 ...
  
  rerank:
    enabled: false                      # 总开关，默认关闭
    min_candidates_to_trigger: 10       # 候选数少于该值时跳过Rerank
    max_candidates_to_rerank: 50        # 对RRF粗筛后的前N条进行精排
    model_provider: "cohere"            # cohere / jina / custom
    model_name: "rerank-v3.5"
    api_key: "${RERANK_API_KEY}"        # 从环境变量读取
    base_url: "https://api.cohere.ai/v1/rerank"
    timeout_ms: 500
    fallback_on_error: true             # 失败时自动降级为 RRF
    fallback_on_timeout: true
```


## 五、Dify 集成示例

### 5.1 HTTP 请求节点配置

**方法**：`POST`  
**URL**：`{{env.SEARCH_SERVICE_URL}}/api/v1/search`  
**Headers**：
- `Authorization: Bearer {{env.SEARCH_API_KEY}}`
- `Content-Type: application/json`

**Body JSON 模板**：

```json
{
  "request_id": "{{sys.workflow_run_id}}",
  "session_id": "{{session_id}}",
  "query": "{{intent_understanding.rewritten_query}}",
  "query_embedding": {{embedding_node.embedding}},
  "source_kb": "{{router_node.source_kb}}",
  "tag_filters": {{router_node.filter_tags}},
  "entity_codes": {{router_node.entity_codes}},
  "top_k": 20
}
```

### 5.2 完整工作流示意

```
[用户输入] → [任务5:意图理解] → [Embedding模型] → [HTTP请求(任务6)] → [任务7:排序] → [输出]
                                    │
                                    └─ 生成 query_embedding
```


## 六、版本历史

| 版本 | 日期 | 变更说明 |
| :--- | :--- | :--- |
| v1.0 | 2026-08-05 | 初始版本，基于任务6详细设计 |
| **v1.1** | **2026-08-07** | **将 Rerank 模型纳入任务6内部，作为可选步骤；响应体新增 `rerank_info` 可观测字段；接口契约完全不变** |


## 七、设计原则总结

| 原则 | 说明 |
| :--- | :--- |
| **对外接口不变** | 调用方（Dify、任务5、任务7）无需任何改动 |
| **Rerank 是可插拔的** | 通过 Nacos 一键开启/关闭，支持 A/B 测试 |
| **下游无感知** | 无论 Rerank 是否启用，`recall_score` 字段名不变 |
| **可观测性完备** | `kb_search_logs` 记录 Rerank 执行详情，`rerank_info` 返回给调用方 |
| **优雅降级** | Rerank 失败时自动降级为 RRF，不影响业务 |


**下一步建议**：任务7（排序加权）将消费本接口返回的 `results` 列表中的 `recall_score` 作为语义相关性基线分，叠加业务权重（来源、时效、个性化）后输出最终排序结果。任务7无需关心 `recall_score` 来自 RRF 还是 Rerank 模型。