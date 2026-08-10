# 任务9：多步规划与自主评估 —— 接口设计文档（v1.0）

> **版本说明**：本文档基于任务9（ReAct模式 + 复杂任务自主评估）的详细设计方案，定义对外服务API及内部交互契约。接口设计遵循Dify服务API规范，支持流式响应，与已交付的任务4（路由）、任务5（意图理解）、任务6（召回）、任务7（排序）无缝协同。

---

## 一、设计目标与适用范围

任务9对外提供**智能多步推理与自主决策**能力，适用于需要复杂问题拆解、多轮检索、结果评估与动态调整的场景。

核心能力包括：

- **动态拆解**：将用户问题自动分解为多步检索/推理子任务，每步根据前一步结果调整策略（ReAct循环）。
- **自主评估**：对每一步检索结果的置信度进行综合评估，并据此决策：
  - 信息充足 → 生成最终答案
  - 信息不足 → 主动反问用户补充
  - 结果矛盾 → 说明冲突并提供依据
  - 严重不足 → 拒绝回答
- **与现有模块协同**：内部调用路由（任务4）、意图理解（任务5）、召回（任务6）、排序（任务7）作为工具，通过评估模块（新增）驱动循环。

**适用应用类型**：Chatflow、Workflow、Agent，均可通过API集成。

**接口风格**：遵循Dify Service API规范，支持流式（SSE）与阻塞两种响应模式。

---

## 二、接口概览

| 接口 | 方法 | 路径 | 说明 |
| :--- | :--- | :--- | :--- |
| 发起多步规划 | POST | `/api/v1/agent/run` | 启动一个多步规划任务，返回流式或阻塞响应 |
| 停止规划任务 | POST | `/api/v1/agent/stop/{task_id}` | 中断正在运行的任务 |
| 提交反问回复 | POST | `/api/v1/agent/ask/submit` | 用户对反问的回复，用于恢复暂停的规划 |
| 获取任务状态 | GET | `/api/v1/agent/status/{task_id}` | 查询任务当前状态（可选，流式已包含） |

**认证方式**：所有接口均需在请求头中携带 `Authorization: Bearer {API_KEY}`，使用工作空间API Key。

---

## 三、接口详细定义

### 3.1 发起多步规划

```
POST /api/v1/agent/run
```

#### 请求体

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| `query` | string | 是 | 用户原始问题 |
| `inputs` | object | 否 | 应用输入变量（如用户角色、上下文等），键值对 |
| `user` | string | 是 | 终端用户标识，用于会话隔离和日志追踪 |
| `conversation_id` | string | 否 | 会话ID，用于多轮对话继承上下文；不传则新建会话 |
| `response_mode` | string | 否 | `streaming`（流式，默认）或 `blocking`（阻塞） |
| `max_iterations` | integer | 否 | 最大规划循环轮次，默认3，上限5 |
| `retrieval_top_k` | integer | 否 | 每轮召回的候选数量，默认10 |
| `confidence_threshold` | float | 否 | 置信度阈值（0~1），达到此值即停止检索并回答，默认0.85 |
| `agent_config` | object | 否 | Agent级别配置，可覆盖默认设置，结构见下文 |

**`agent_config` 结构**：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| `model` | object | 指定推理模型（provider/model），若不指定则使用默认模型 |
| `tools` | array | 允许调用的工具列表（路由/召回/评估等），若不指定则全部可用 |
| `instructions` | string | 自定义系统指令，覆盖默认的Agent提示词 |

**请求示例**（流式）：

```json
{
  "query": "请对比分析2024年Q3和Q4的基金业绩，并说明原因。",
  "inputs": {
    "user_role": "基金经理",
    "preferred_language": "中文"
  },
  "user": "user-123",
  "conversation_id": "conv-abc",
  "response_mode": "streaming",
  "max_iterations": 4,
  "confidence_threshold": 0.80
}
```

#### 响应

- **阻塞模式**：返回 JSON，包含最终答案及元数据。
- **流式模式**：返回 `text/event-stream`，事件类型如下：

| 事件名称 | 触发时机 | 数据字段 |
| :--- | :--- | :--- |
| `agent_start` | 规划开始 | `task_id`, `conversation_id`, `iteration_count` |
| `agent_thought` | 每轮推理步骤 | `round`, `thought`, `tool_calls`（数组，含工具名称、参数） |
| `agent_tool_result` | 工具调用完成 | `tool_name`, `result_summary`, `confidence`（各维度得分） |
| `agent_evaluation` | 评估节点完成 | `overall_confidence`, `missing_info`, `contradictions`, `decision`（`continue`/`ask`/`reject`/`answer`） |
| `agent_ask` | 需要反问用户 | `question`, `options`（可选）, `form_token` |
| `agent_answer` | 最终答案（片段） | `text`（增量），最终会有一个 `is_final: true` |
| `agent_end` | 规划结束 | `final_answer`, `total_rounds`, `usage`（token消耗） |
| `error` | 发生错误 | `status`, `code`, `message` |
| `ping` | 保活（每10秒） | 无数据 |

**事件数据示例（`agent_evaluation`）**：

```json
{
  "event": "agent_evaluation",
  "data": {
    "round": 2,
    "overall_confidence": 0.62,
    "dimensions": {
      "relevance": 0.70,
      "coverage": 0.55,
      "authority": 0.80,
      "consistency": 0.45
    },
    "missing_info": ["2024年Q4具体收益数据", "基金规模变化"],
    "contradictions": [
      {
        "field": "收益率",
        "value_a": "5.2%（来源：A基金季报）",
        "value_b": "4.8%（来源：B基金季报）"
      }
    ],
    "decision": "ask"
  }
}
```

**错误码**（与Dify规范一致）：

| 错误码 | HTTP状态 | 说明 |
| :--- | :--- | :--- |
| `invalid_param` | 400 | 请求参数缺失或格式错误 |
| `agent_not_available` | 400 | 当前无可用Agent（如未配置模型） |
| `too_many_requests` | 429 | 并发请求超限 |
| `internal_server_error` | 500 | 服务内部错误 |

---

### 3.2 停止规划任务

```
POST /api/v1/agent/stop/{task_id}
```

**请求体**：

```json
{
  "user": "user-123"
}
```

**响应**：成功返回 `{"result": "success"}`。

**错误码**：`task_not_found`（404），`user_mismatch`（403，用户不匹配）。

---

### 3.3 提交反问回复

当流式事件中收到 `agent_ask` 时，客户端需展示反问内容并收集用户回复，然后调用此接口恢复规划。

```
POST /api/v1/agent/ask/submit
```

**请求体**：

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| `form_token` | string | 是 | 来自 `agent_ask` 事件中的 `form_token` |
| `user` | string | 是 | 终端用户标识，需与发起任务时一致 |
| `inputs` | object | 否 | 用户填写的回答内容，键为表单字段名 |
| `action` | string | 否 | 用户选择的按钮操作（如果提供） |

**响应**：返回与 `POST /api/v1/agent/run` 相同的流式/阻塞响应，继续执行后续规划。

**错误码**：`form_token_invalid`（400），`form_expired`（400），`user_mismatch`（403）。

---

### 3.4 获取任务状态（可选）

```
GET /api/v1/agent/status/{task_id}?user={user}
```

**响应**：

```json
{
  "task_id": "task-123",
  "status": "running" | "paused" | "completed" | "rejected" | "failed",
  "current_round": 2,
  "confidence": 0.62,
  "missing_info": ["..."],
  "created_at": "2026-08-07T10:00:00Z",
  "updated_at": "2026-08-07T10:05:00Z"
}
```

---

## 四、内部组件交互契约

任务9的自研服务内部包含以下核心模块，它们之间的接口如下（供开发参考）：

### 4.1 Agent Orchestrator（编排器）

对外暴露 `run()` 方法，负责整个ReAct循环的调度。

**输入**：`AgentRequest`（包含query, inputs, user, conversation_id, config等）  
**输出**：`Generator[AgentEvent]`（流式事件流）

### 4.2 Tool Invoker（工具调用器）

封装对任务4/5/6/7及评估模块的调用。

**接口**：

- `route(query, context) -> RoutingResult`
- `understand(query, session) -> IntentResult`
- `retrieve(query, filters, top_k) -> SearchResult`
- `rank(candidates, user_context) -> RankedResult`
- `evaluate(query, candidates, history) -> EvaluationResult`（新增）

所有工具调用均支持超时和降级策略。

### 4.3 State Manager（状态管理器）

维护每个会话的Agent状态（当前轮次、已积累知识、置信度、缺失信息等），持久化到MySQL（表 `kb_sessions.agent_state`）。

**接口**：

- `get_state(conversation_id) -> AgentState`
- `update_state(conversation_id, state) -> void`
- `reset_state(conversation_id) -> void`

### 4.4 Evaluator（评估模块）

新增核心模块，实现置信度计算和决策建议。

**接口**：`evaluate(query, candidates, history) -> EvaluationResult`  
**内部计算逻辑**：综合召回分数、文档权威性、覆盖度、多轮一致性，加权得出综合置信度，并识别缺失信息与矛盾。

---

## 五、与Dify API的集成点

任务9服务本身可作为Dify的一个**工具**或**端点插件**集成，也可作为独立微服务由Dify通过HTTP节点调用。

推荐集成方式：

1. **作为Dify的自定义工具**：将 `/api/v1/agent/run` 封装为Dify工具，供Workflow/Agent节点调用，实现复杂任务的编排。

2. **作为Dify的扩展端点**：通过插件端点暴露，由外部系统直接调用。

3. **直接调用**：在Dify的HTTP节点中构造请求，调用任务9服务。

**与Dify应用API的异同**：

- 任务9的流式事件与Dify Agent应用的 `agent_thought` 和 `agent_message` 类似，但新增了 `agent_evaluation` 和 `agent_ask` 事件。
- 反问/恢复机制与Dify的 `human_input` 节点流程一致，可复用 `form_token` 模式。

---

## 六、错误处理与重试

- **临时性错误**（如召回超时、模型限流）：内部自动重试（最多3次），重试失败后以 `error` 事件通知。
- **用户输入错误**（如参数缺失）：返回 `invalid_param` 错误码。
- **服务不可用**：返回 `internal_server_error`，建议客户端退避重试。

---

## 七、性能与配额

- 单任务最大迭代次数：5（可配置）。
- 单任务最大执行时间：120秒（超时自动终止）。
- 并发任务数：默认每用户5个，可通过环境变量调整。

---

## 八、示例（流式调用）

**请求**：

```bash
curl -X POST https://api.example.com/api/v1/agent/run \
  -H "Authorization: Bearer {API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "请分析最近一周的科技股走势，并给出投资建议。",
    "user": "user-456",
    "response_mode": "streaming"
  }'
```

**响应流（SSE）**：

```
event: agent_start
data: {"task_id": "task-789", "conversation_id": "conv-xyz", "iteration_count": 0}

event: agent_thought
data: {"round": 1, "thought": "需要先获取科技股最近一周的行情数据，再分析走势。", "tool_calls": [{"tool": "route", "params": {"query": "科技股 近期走势"}}]}

event: agent_tool_result
data: {"tool_name": "route", "result_summary": "路由到股票知识库", "confidence": 0.95}

event: agent_thought
data: {"round": 1, "thought": "现在调用召回获取具体数据。", "tool_calls": [{"tool": "retrieve", "params": {"query": "科技股 一周 涨跌幅"}}]}

event: agent_tool_result
data: {"tool_name": "retrieve", "result_summary": "找到15条相关片段", "confidence": 0.72}

event: agent_evaluation
data: {"overall_confidence": 0.68, "missing_info": ["具体个股涨跌幅排序"], "decision": "continue"}

event: agent_thought
data: {"round": 2, "thought": "信息不够具体，需要细化查询个股。", "tool_calls": [{"tool": "retrieve", "params": {"query": "科技股 个股 涨幅 排名"}}]}

event: agent_tool_result
data: {"tool_name": "retrieve", "result_summary": "获取到个股涨跌排名", "confidence": 0.85}

event: agent_evaluation
data: {"overall_confidence": 0.88, "decision": "answer"}

event: agent_answer
data: {"text": "根据最近一周数据，科技股整体上涨2.3%，其中", "is_final": false}

event: agent_answer
data: {"text": "半导体板块表现突出，龙头股涨幅超5%。建议关注...", "is_final": true}

event: agent_end
data: {"final_answer": "根据最近一周数据，科技股整体上涨2.3%，其中半导体板块表现突出...", "total_rounds": 2, "usage": {"prompt_tokens": 300, "completion_tokens": 150}}
```

---

## 九、版本与兼容性

- 当前版本：v1.0
- 接口设计兼容Dify v1.0+ API规范。
- 后续迭代将保持向后兼容，新增字段不影响已有客户端。

---

**文档结束**