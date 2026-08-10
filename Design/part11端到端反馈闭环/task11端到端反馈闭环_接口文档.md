好的，结合您提供的Dify API参考、任务11架构设计以及`数据模型_v6.4`，我为您输出**任务11：端到端反馈闭环**的完整接口设计文档。

这份设计充分遵循了“人机协同半自动闭环”的原则，并参考Dify现有的反馈API模式，以确保与前端和Dify编排层的无缝衔接。

---

# 任务11：端到端反馈闭环接口设计文档（v1.0）

> **版本**：v1.0
> **更新日期**：2026-08-09
> **状态**：✅ 已定稿，待进入实现阶段
> **关联数据模型**：`数据模型_v6.4`（扩展`kb_user_feedback` + 新增`kb_feedback_analysis_logs` + 新增`kb_feedback_stats`）

---

## 1. 概述

### 1.1 设计目标
建立 **“反馈采集 → 问题归类 → 根因分析 → 策略优化”** 的闭环机制。接口设计遵循 **“漏斗式分层处理”**：
- **采集层**：复用Dify内置反馈能力，汇聚至自研服务。
- **分析层**：支持自动（置信度>0.85）、辅助（0.6~0.84）、人工（<0.6）三种分析模式。
- **优化层**：系统生成候选建议，强制人工审批后方可执行变更。

### 1.2 接口前缀
所有接口路径前缀为：`/api/v1/feedback`

### 1.3 与 Dify 的集成方式
- **采集**：Dify WebApp/API 通过 `POST /messages/{message_id}/feedbacks` 提交点赞/点踩，自研服务提供 **统一汇聚接口** 接收（或通过定时任务拉取Dify的 `GET /app/feedbacks`）。
- **追踪**：通过 `trace_id` 关联自研服务全链路日志（`kb_router_logs`、`kb_search_logs`、`kb_rank_logs`）。

---

## 2. 反馈采集接口

### 2.1 接收/汇聚反馈

**接口描述**：接收来自 Dify 或其他渠道（如 WebApp 直连）的反馈数据，写入 `kb_user_feedback` 表并触发异步根因分析。

- **URL**：`POST /api/v1/feedback`
- **请求体（JSON）**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :--- | :--- |
| `trace_id` | string | 是 | 全链路追踪ID |
| `session_id` | string | 否 | 会话ID |
| `message_id` | string | 否 | Dify消息ID（用于从Dify拉取补全信息） |
| `rating` | string | 是 | `thumbs_up` / `thumbs_down` |
| `comment` | string | 否 | 用户备注 |
| `source` | string | 否 | `dify_web` / `dify_api` / `custom`，默认`dify_web` |
| `user_id` | string | 否 | 用户标识 |
| `metadata` | json | 否 | 扩展元数据（如当前知识库、原始查询等） |

- **响应**：
```json
{
  "code": 0,
  "data": {
    "feedback_id": 12345,
    "status": "pending"
  }
}
```

- **内部逻辑**：
  1. 持久化反馈，`analysis_status` 初始为 `pending`。
  2. 发送 MQ 消息异步触发根因分析（非阻塞）。
  3. 若 `source` 为 `dify_web` 且未传入 `trace_id`，则调用 Dify API `GET /messages/{message_id}` 反向补全上下文。

---

## 3. 根因分析接口

### 3.1 触发单条分析（手动重试）

- **URL**：`POST /api/v1/feedback/{feedback_id}/analyze`
- **请求体**：无（或可选 `force` 布尔值强制重新分析）
- **响应**：
```json
{
  "code": 0,
  "data": {
    "analysis_id": 789,
    "status": "queued"
  }
}
```

### 3.2 获取分析结果

- **URL**：`GET /api/v1/feedback/{feedback_id}/analysis`
- **响应**（含完整分析明细）：
```json
{
  "code": 0,
  "data": {
    "feedback_id": 12345,
    "category": "retrieval_failure",
    "category_desc": "召回失败：相关文档未被检索到",
    "root_cause": "查询改写后关键词偏移，导致ES未能匹配到目标文档",
    "analysis_confidence": 0.87,
    "analysis_status": "completed",
    "evidence": {
      "trace_id": "trace_xxx",
      "search_logs": { "recall_count": 3, "top_score": 0.42 },
      "rank_logs": { "ranked_chunks": [...] }
    },
    "suggestion": {
      "type": "config_tuning",
      "detail": "建议调整 query_rewriter 的同义词扩展策略，增加'XX'相关同义词",
      "priority": "high"
    },
    "analyzed_at": "2026-08-09T10:30:00Z"
  }
}
```

### 3.3 批量触发分析（运营看板用）

- **URL**：`POST /api/v1/feedback/analyze/batch`
- **请求体**：
```json
{
  "source_kb": "fund_kb",
  "time_range": { "start": "2026-08-01", "end": "2026-08-07" },
  "category_filter": ["retrieval_failure", "knowledge_gap"],
  "limit": 100
}
```
- **响应**：返回任务ID，异步执行。

---

## 4. 统计与看板接口

### 4.1 趋势统计（用于Grafana/前端看板）

- **URL**：`GET /api/v1/feedback/stats/trend`
- **Query参数**：`source_kb`、`start_date`、`end_date`、`granularity`（`day`/`week`/`month`）
- **响应**：
```json
{
  "code": 0,
  "data": {
    "total": 156,
    "positive": 92,
    "negative": 64,
    "trend": [
      { "date": "2026-08-01", "positive": 10, "negative": 5 },
      { "date": "2026-08-02", "positive": 12, "negative": 8 }
    ]
  }
}
```

### 4.2 问题类型分布

- **URL**：`GET /api/v1/feedback/stats/categories`
- **响应**：
```json
{
  "code": 0,
  "data": {
    "distribution": {
      "retrieval_failure": 45,
      "knowledge_gap": 32,
      "rank_bias": 28,
      "generation_error": 21,
      "route_error": 18,
      "intent_error": 12
    }
  }
}
```

### 4.3 Top N 失败查询

- **URL**：`GET /api/v1/feedback/top-failing-queries`
- **Query参数**：`limit`（默认20）、`source_kb`
- **响应**：返回点踩率最高的原始查询词列表。

---

## 5. 优化建议管理接口（人工审批流）

### 5.1 获取待处理优化建议列表

- **URL**：`GET /api/v1/feedback/optimizations/pending`
- **响应**：
```json
{
  "code": 0,
  "data": [
    {
      "feedback_id": 12345,
      "suggestion": { "type": "config_tuning", "detail": "...", "priority": "high" },
      "optimization_status": "pending",
      "created_at": "2026-08-09T10:00:00Z"
    }
  ]
}
```

### 5.2 更新优化状态（人工批准/驳回/实施）

- **URL**：`PUT /api/v1/feedback/{feedback_id}/optimization`
- **请求体**：
```json
{
  "status": "approved",  // pending | approved | rejected | implemented | verified
  "comment": "已批准，将在下个迭代中实施",
  "reviewer_id": "admin_001"
}
```
- **响应**：返回更新后的状态。

### 5.3 触发优化验证（对接任务10）

- **URL**：`POST /api/v1/feedback/{feedback_id}/verify`
- **说明**：当`optimization_status`为`implemented`时调用，触发任务10的评测流水线进行回归测试。
- **响应**：
```json
{
  "code": 0,
  "data": {
    "evaluation_run_id": "run_20260809_001",
    "status": "triggered"
  }
}
```

---

## 6. 系统与运维接口

### 6.1 手动同步 Dify 反馈

用于定时任务或手动补全Dify侧的孤立反馈数据。

- **URL**：`POST /api/v1/feedback/sync/dify`
- **请求体**：
```json
{
  "app_id": "dify_app_xxx",
  "start_time": "2026-08-01T00:00:00Z",
  "end_time": "2026-08-08T23:59:59Z"
}
```

### 6.2 健康检查

- **URL**：`GET /api/v1/feedback/health`
- **响应**：`{"status": "ok", "mq_connected": true, "db_connected": true}`

---

## 7. 通用数据结构与错误码

### 7.1 分析状态机
- `pending` → `analyzing` → `completed` / `failed` / `manual_review`

### 7.2 优化状态机
- `pending` → `approved` / `rejected` → `implemented` → `verified`

### 7.3 标准错误响应（参照Dify规范）

```json
{
  "code": "invalid_param",
  "message": "feedback_id is required",
  "status": 400
}
```

### 7.4 主要错误码清单

| code | HTTP Status | 描述 |
| :--- | :--- | :--- |
| `feedback_not_found` | 404 | 反馈记录不存在 |
| `analysis_in_progress` | 409 | 分析任务正在进行中 |
| `optimization_invalid_status` | 400 | 当前状态不允许执行该操作 |
| `dify_sync_failed` | 502 | 同步Dify数据失败 |
| `evaluation_trigger_failed` | 500 | 触发任务10评测失败 |

---

## 8. 接口调用流程示例

```
1. 用户点踩 → Dify触发Webhook → POST /api/v1/feedback (自研汇聚)
2. 异步任务消费MQ → 自研执行根因分析 → 更新kb_user_feedback
3. 运营查看看板 → GET /api/v1/feedback/stats/categories
4. 运营审批建议 → PUT /api/v1/feedback/{id}/optimization (status=approved)
5. 开发实施变更 → PUT .../optimization (status=implemented)
6. 自动验证 → POST .../{id}/verify → 任务10跑回归测试
7. 测试通过 → PUT .../optimization (status=verified)
```

---

此接口设计文档与您提供的 Dify API 参考（`repomix-output.xml`）保持语义一致性，特别是在反馈提交、错误处理和状态码设计上复用了Dify的成熟规范，便于前端和Dify编排层无缝对接。

如需进入下一步（如编写 Python Flask/FastAPI 代码骨架、实现 MQ 消费者、或编写 LLM 分析 Prompt），请随时告知。