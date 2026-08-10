好的，基于您提供的完整背景以及 Dify 官方 API 参考，我为您设计**任务8：文档版本管理与整体联调**所需的接口规范。

> 本设计聚焦于自研知识库服务的对外接口，同时指明与 Dify API 的集成点，确保全链路贯通。

---

# 任务8 接口设计文档（v1.0）

**版本**：v1.0  
**日期**：2026-08-07  
**作者**：项目负责人  
**状态**：草稿，待评审  

---

## 1. 概述

本接口文档覆盖任务8（文档版本管理与整体联调）所需的自研服务 API。主要功能分为两大块：

- **文档版本管理与审批流程**：实现新文档自动检测、变更提醒、人工确认、重新索引的闭环。
- **全链路联调与运维支持**：提供健康检查、端到端测试、日志追踪等接口，确保服务可运行。

接口设计遵循 RESTful 风格，统一使用 JSON 格式，认证方式沿用现有的 API Key（Bearer Token）。所有接口返回标准错误结构（参考 Dify 错误格式）。

---

## 2. 接口列表概览

| 模块 | 方法 | 路径 | 描述 |
|------|------|------|------|
| **版本管理** | POST | `/api/v1/documents` | 上传新文档（触发变更检测） |
| | GET | `/api/v1/admin/pending-changes` | 获取待人工确认的变更列表（管理员） |
| | POST | `/api/v1/admin/approve-index` | 人工确认或驳回变更（管理员） |
| | GET | `/api/v1/documents/{doc_id}/versions` | 获取文档版本历史 |
| | POST | `/api/v1/documents/{doc_id}/reindex` | 手动触发重新索引（管理员） |
| **联调与运维** | GET | `/api/v1/health` | 健康检查 |
| | POST | `/api/v1/test/end-to-end` | 端到端全链路测试（模拟用户提问） |
| | GET | `/api/v1/trace/{trace_id}` | 根据 trace_id 查询全链路日志 |

---

## 3. 详细接口定义

### 3.1 上传新文档（或更新文档）

> 此接口复用已有的文档上传流程，但需**增加版本检测逻辑**，并在检测到变更时进入 `pending_approval` 状态。

**路径**：`POST /api/v1/documents`

**请求体**（multipart/form-data 或 JSON）：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `file` | binary | 是 | 文档文件（支持 PDF、DOCX 等） |
| `source_kb` | string | 是 | 所属知识库标识（如 `fund`） |
| `doc_type` | string | 否 | 文档类型 |
| `metadata` | object | 否 | 元数据（键值对） |

**响应**（201 Created）：

```json
{
  "doc_id": "uuid",
  "doc_uuid": "uuid",
  "file_hash": "sha256",
  "status": "pending_approval",   // 若检测到变更则为此状态，否则为 "published"
  "message": "文档已上传，若检测到变更将进入审批流程"
}
```

**处理逻辑**：
1. 计算文件 hash，查询 `kb_documents` 是否已存在相同 `file_hash` 且 `lifecycle_status='published'`。
2. 若存在且 hash 不同，将旧文档 `lifecycle_status` 置为 `pending_approval`，新文档插入状态为 `pending_approval`，旧版本继续服务。
3. 触发通知（邮件/后台待办）。
4. 若 hash 相同且文档已存在，视为重复上传，直接返回成功（跳过索引）。

---

### 3.2 获取待确认变更列表（管理员）

**路径**：`GET /api/v1/admin/pending-changes`

**查询参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `page` | int | 否 | 页码，默认 1 |
| `limit` | int | 否 | 每页条数，默认 20 |
| `source_kb` | string | 否 | 按知识库筛选 |

**响应**（200 OK）：

```json
{
  "data": [
    {
      "doc_id": "uuid",
      "file_name": "合规制度V2.0.pdf",
      "source_kb": "compliance",
      "old_version": "v2.0",
      "new_version": "v3.0",
      "detected_at": "2026-08-07T10:00:00Z",
      "change_summary": "第三章数据更新，影响5个子块",
      "status": "pending_approval"
    }
  ],
  "pagination": { "page": 1, "limit": 20, "total": 5 }
}
```

---

### 3.3 人工确认或驳回变更（管理员）

**路径**：`POST /api/v1/admin/approve-index`

**请求体**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `doc_id` | string | 是 | 文档 ID |
| `action` | string | 是 | `approve` 或 `reject` |
| `reject_reason` | string | 否 | 驳回理由（action=reject 时建议提供） |

**响应**（200 OK）：

```json
{
  "doc_id": "uuid",
  "new_status": "published",   // 或 "rejected"
  "message": "变更已批准，正在重新索引..."
}
```

**处理逻辑**：
- 若 `approve`：将 `lifecycle_status` 改为 `published`，并触发三库差量更新（旧块标记 `is_active=0`，新块写入）。
- 若 `reject`：将状态改为 `rejected`，并回退到 `previous_lifecycle_status`（即原来的 `published`），不更新索引。
- 无论结果如何，记录审批人、时间和原因。

---

### 3.4 获取文档版本历史

**路径**：`GET /api/v1/documents/{doc_id}/versions`

**响应**（200 OK）：

```json
{
  "doc_id": "uuid",
  "versions": [
    {
      "version": "v1.0",
      "file_hash": "abc...",
      "indexed_at": "2026-01-01T00:00:00Z",
      "status": "archived"
    },
    {
      "version": "v2.0",
      "file_hash": "def...",
      "indexed_at": "2026-06-01T00:00:00Z",
      "status": "published"
    }
  ]
}
```

---

### 3.5 手动触发重新索引（管理员）

> 用于异常修复或强制重新索引。

**路径**：`POST /api/v1/documents/{doc_id}/reindex`

**响应**（202 Accepted）：

```json
{
  "message": "重新索引任务已提交",
  "task_id": "task-xxx"
}
```

---

### 3.6 健康检查

**路径**：`GET /api/v1/health`

**响应**（200 OK）：

```json
{
  "status": "healthy",
  "components": {
    "mysql": "ok",
    "elasticsearch": "ok",
    "milvus": "ok",
    "redis": "ok",
    "dify_api": "ok"
  }
}
```

---

### 3.7 端到端测试接口

> 用于联调时模拟完整查询流程，返回各阶段耗时和结果，便于定位瓶颈。

**路径**：`POST /api/v1/test/end-to-end`

**请求体**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `query` | string | 是 | 用户问题 |
| `source_kb` | string | 否 | 指定知识库，不填则走路由 |
| `user_role` | string | 否 | 用户角色（用于个性化排序） |

**响应**（200 OK）：

```json
{
  "query": "...",
  "trace_id": "trace-xxx",
  "stages": {
    "router": { "duration_ms": 12, "result": { "source_kb": "fund" } },
    "intent": { "duration_ms": 230, "result": { "intent": "factual", "rewritten_query": "..." } },
    "retrieve": { "duration_ms": 450, "rerank_enabled": false, "candidate_count": 20 },
    "rank": { "duration_ms": 35, "final_count": 5 }
  },
  "final_answer": "根据最新规定..."
}
```

---

### 3.8 根据 trace_id 查询全链路日志

**路径**：`GET /api/v1/trace/{trace_id}`

**响应**：返回该 trace_id 在各日志表中的聚合记录（简化示例）：

```json
{
  "trace_id": "trace-xxx",
  "router_log": { ... },
  "query_log": { ... },
  "search_log": { ... },
  "rank_log": { ... },
  "feedback": { ... }
}
```

---

## 4. 错误码规范

沿用 Dify 风格，统一错误结构：

```json
{
  "code": "error_code",
  "message": "可读的错误描述",
  "status": 400
}
```

常见错误码：

| 错误码 | HTTP 状态 | 含义 |
|--------|-----------|------|
| `invalid_doc_id` | 404 | 文档不存在 |
| `invalid_action` | 400 | action 仅允许 approve/reject |
| `already_processed` | 409 | 该变更已被处理 |
| `permission_denied` | 403 | 无权限（非管理员） |
| `indexing_failed` | 500 | 重新索引失败 |

---

## 5. 与 Dify API 的集成点

任务8中的“重新索引”需要调用 Dify 的知识库文档更新接口。我们已在数据模型中记录了 `doc_id` 与 Dify 的 `document_id` 映射。当管理员批准变更时，自研服务应调用以下 Dify API：

- **更新文档**：`PATCH /datasets/{dataset_id}/documents/{document_id}`（上传新文件并重新索引）
- **获取索引状态**：`GET /datasets/{dataset_id}/documents/{batch}/indexing-status`（轮询直至完成）

同时，联调过程中需验证 Dify 编排节点（路由、意图、检索、排序）是否与自研服务正确对接，可通过上述 `/test/end-to-end` 接口模拟完整流程。

---

## 6. 附录：联调检查清单

| 检查项 | 接口/方式 |
|--------|-----------|
| 路由模块是否返回正确 `source_kb` | 调用 `/test/end-to-end` 查看 `router` 阶段 |
| 意图识别是否正常（含降级） | 同上 |
| 召回是否包含 Rerank 开关及降级 | 查看 `search_logs` 中的 `rerank_enabled` 字段 |
| 排序权重是否按角色动态加载 | 检查 `rank_logs` 中的 `used_weights` |
| 版本变更后旧版本是否继续服务 | 上传新文档后，查询旧版本仍返回内容 |
| 人工确认接口是否触发三库更新 | 批准后检查 `kb_chunks.is_active` 和 ES/Milvus |

---

**文档结束**