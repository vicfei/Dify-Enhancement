# 任务5 接口设计文档：意图识别、拆解与改写（Query Understanding API）

> **版本**：v1.0  
> **更新日期**：2026年8月  
> **适用范围**：知识库深度优化项目，用于统一知识库门户的查询预处理模块。  
> **相关任务**：任务5（计划第6-7周，9.7-9.20）

---

## 概述

本接口模块负责对用户的自然语言查询进行**意图识别、问题拆解与查询改写**，为下游混合检索（任务6）提供标准化、多路扩展的检索Query列表。

模块采用“轻量分类器 + LLM深度处理”的分层策略，支持事实查询、分析推理、数据计算等意图分类，并能将复杂多步问题拆解为子问题，同时进行指代消解、术语标准化和同义词扩展，显著提升召回率。

**核心流程**：

1. 接收原始Query及会话上下文（`session_id`）、路由结果（`source_kb`）等。
2. 进行意图分类（L1快速通路或LLM深度分类）。
3. 若为`ANALYTICAL`类型，则拆解为子问题列表。
4. 进行查询改写（补全省略、替换口语、标准化术语、扩展同义词），输出1~3个检索Query。
5. 返回处理结果，同时记录日志（`kb_query_logs`）并更新会话上下文（`kb_sessions`）。

---

## 认证与鉴权

所有接口均需通过 **Bearer Token** 认证，使用与Dify服务API相同的API Key（在`Authorization`头中传递）。  
示例：

```http
Authorization: Bearer {your_api_key}
```

---

## 接口列表

| 接口 | 方法 | 用途 |
| :--- | :--- | :--- |
| `/api/v1/query/process` | POST | 核心查询处理接口，返回改写后的Query列表及元数据 |
| `/api/v1/query/health`   | GET  | 健康检查（可选） |

---

### 1. 查询处理接口 `POST /api/v1/query/process`

#### 功能描述

对用户原始查询进行意图识别、拆解（如需）和改写，输出可直接用于混合检索的标准化Query列表。

#### 请求体 (JSON)

| 字段 | 类型 | 必填 | 描述 |
| :--- | :--- | :--- | :--- |
| `query` | string | 是 | 用户原始查询文本 |
| `session_id` | string | 否 | 会话ID，用于多轮指代消解。若提供，系统将加载历史上下文 |
| `source_kb` | string | 否 | 路由模块输出的知识库标识（如`信评知识库`），用于领域偏置 |
| `user_id` | string | 否 | 用户标识，用于日志记录 |
| `trace_id` | string | 否 | 全链路追踪ID，便于关联路由日志与反馈 |

**示例**：

```json
{
  "query": "那个AAA城投的信用风险怎么样，现在能配吗？",
  "session_id": "sess_20260907_001",
  "source_kb": "信评知识库",
  "user_id": "zhangsan",
  "trace_id": "req_20260907_xxx"
}
```

#### 响应体 (JSON)

成功时返回 `200 OK`，响应体结构：

| 字段 | 类型 | 描述 |
| :--- | :--- | :--- |
| `code` | integer | 业务状态码，`0`表示成功 |
| `data` | object | 处理结果 |
| `data.intent` | string | 意图类型：`FACTUAL`/`ANALYTICAL`/`CALCULATION`/`CHITCHAT`/`INVALID` |
| `data.confidence` | float | 意图置信度（0~1） |
| `data.sub_questions` | array[string] | 拆解后的子问题列表（仅`ANALYTICAL`时非空） |
| `data.rewritten_queries` | array[string] | 改写后的检索Query列表，供任务6使用（1~3条） |
| `data.expanded_terms` | array[string] | 扩展的同义词/相关术语列表 |
| `data.processing_time_ms` | integer | 处理耗时（毫秒） |
| `trace_id` | string | 原样返回请求中的`trace_id`，便于链路追踪 |

**示例**：

```json
{
  "code": 0,
  "data": {
    "intent": "ANALYTICAL",
    "confidence": 0.92,
    "sub_questions": [
      "AAA城投主体信用风险评估",
      "当前城投债配置建议"
    ],
    "rewritten_queries": [
      "AAA评级城投公司信用风险分析报告",
      "城投债投资策略与配置建议 2026",
      "城投平台信用资质评价"
    ],
    "expanded_terms": ["城投平台", "地方融资平台", "信用资质"],
    "processing_time_ms": 450
  },
  "trace_id": "req_20260907_xxx"
}
```

**错误响应**（HTTP 4xx/5xx）：

遵循Dify统一错误格式：

```json
{
  "status": 400,
  "code": "invalid_param",
  "message": "query is required"
}
```

#### 错误码说明

| HTTP状态 | `code` | 含义 | 处理方式 |
| :--- | :--- | :--- | :--- |
| 400 | `invalid_param` | 请求参数缺失或格式错误（如`query`为空） | 检查请求体 |
| 400 | `Q001` | LLM分类服务超时 | 已自动切换降级规则，返回基础改写结果 |
| 400 | `Q002` | 改写结果为空（无法生成有效Query） | 返回原始Query，记录告警 |
| 400 | `bad_request` | 请求体JSON解析失败 | 修正格式 |
| 401 | `unauthorized` | API Key无效或缺失 | 检查认证头 |
| 429 | `too_many_requests` | 并发请求过多 | 退避重试 |
| 500 | `internal_server_error` | 服务内部错误 | 联系管理员 |

---

### 2. 健康检查接口 `GET /api/v1/query/health`

用于Kubernetes探针或运维监控，返回服务状态。

**响应**：

```json
{
  "status": "ok",
  "timestamp": "2026-09-07T10:00:00Z"
}
```

---

## 依赖的外部接口

本模块在实现中会依赖以下Dify服务接口（由自研核心调用）：

| 外部接口 | 用途 | 备注 |
| :--- | :--- | :--- |
| `GET /datasets` | 获取知识库列表，校验`source_kb`有效性 | 定时缓存，避免频繁调用 |
| `GET /datasets/{id}` | 获取知识库详情，用于歧义引导时的描述展示 | 仅在歧义场景调用 |
| Dify LLM节点（或直接调用模型供应商API） | 执行意图分类、问题拆解、查询改写 | 通过`session.model.llm`调用 |

---

## 限流与超时

- **单请求超时**：`5s`（含LLM调用），超时后自动切换至规则兜底。
- **并发限制**：同一`user_id`每秒最多10个请求（可配置）。
- **LLM调用熔断**：若LLM连续失败3次，自动降级至正则匹配模式，持续5分钟。

---

## 数据持久化

本接口会自动记录以下信息到数据库，用于后续评测与优化：

- 写入 `kb_query_logs`：完整记录原始Query、意图、拆解、改写结果、耗时、是否降级等。
- 更新 `kb_sessions`：存储当前轮次的改写结果和实体摘要，供下一轮指代消解使用。

详见[数据模型设计 v3.0](/zh/data-model)中的对应表结构。

---

## 变更历史

| 版本 | 日期 | 变更内容 | 作者 |
| :--- | :--- | :--- | :--- |
| v1.0 | 2026-08-05 | 初始版本，基于任务5详细设计生成 | 项目组 |

---

**附录：降级兜底规则（Nacos配置 `query_understanding.yaml`）**

```yaml
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

当LLM不可用时，采用上述正则匹配兜底，改写仅做“去除语气词+原样返回”。