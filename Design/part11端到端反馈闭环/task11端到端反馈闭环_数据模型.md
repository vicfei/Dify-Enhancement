基于任务11（端到端反馈闭环）的架构设计，我们在 `数据模型_v6.3` 的基础上进行增量修改，形成 **`数据模型_v6.4`**。

以下是修改后的完整数据模型文档。**本次修改遵循“零侵入”原则**：完全不触碰任务1-9的任何现有表结构，仅对 `kb_user_feedback` 进行**扩展**（新增字段），并**新增** 2 张辅助表（分析日志 + 聚合统计）。

---

# 数据模型设计（完整版 v6.4 — 含任务11反馈闭环）

> **版本更新说明**：
> - **v6.3 → v6.4（任务11增强版）**：
>   1. **扩展 `kb_user_feedback`**：新增 **问题分类**、**根因分析结果（JSON）**、**分析状态机**、**优化建议（JSON）**、**优化状态机** 及 **关联变更ID**，实现从“原始反馈”到“优化落地”的全生命周期管理。
>   2. **新增 `kb_feedback_analysis_logs`**：记录每次根因分析的详细过程（LLM输入输出、证据链快照、置信度），用于审计追溯与 Prompt 迭代调优。
>   3. **新增 `kb_feedback_stats`**：按天/知识库维度聚合反馈指标，支撑趋势看板与问题热点识别，避免每次查询都全表扫描大表。
> - **v6.2 → v6.3**：任务9（多步规划）ReAct 全轨迹、评估明细、反问队列。
> - **v6.1 → v6.2**：文档生命周期双态（`indexing_status` + `lifecycle_status`）、审批流、通知审计。
> - **v6.0 → v6.1**：融合任务6（Rerank）与任务7（排序日志）。

（*以下为 v6.4 完整版，未提及的表（1.1-1.6、1.8-1.19）与 v6.3 完全一致，此处仅标注章节索引，详情继承 v6.3，不再重复赘述。*）

---

## 一、MySQL 数据模型（共 21 张表）

> **表 1.1 ~ 1.6**（`kb_documents`、`kb_chunks`、`kb_tags`、`kb_doc_tag_relation`、`kb_entity_mapping`、`kb_version_changes`）继承 v6.3，**零改动**。
>
> **表 1.8 ~ 1.19**（`kb_router_logs`、`kb_sessions`、`kb_query_logs`、`kb_search_logs`、`kb_rank_logs`、`kb_mcp_providers`、`kb_mcp_weight_mapping`、`kb_mcp_cache`、`kb_notification_logs`、`kb_agent_loop_traces`、`kb_evaluation_results`、`kb_pending_questions`）继承 v6.3，**零改动**。

---

### 1.7 用户反馈表：`kb_user_feedback`（📌 v6.4 重大扩展）

> **v6.4 变更说明**：在保留原有全链路追踪字段（路由/召回/排序快照）的基础上，新增 **“分析-优化”双状态机**及根因/建议结构化存储，将反馈从“原始记录”升级为“可驱动的优化工单”。

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `trace_id` | varchar(64) | INDEX | 全链路追踪ID |
| `session_id` | varchar(64) | NULL | 会话ID（冗余，便于会话维度聚合） |
| `original_query` | text | NULL | 用户原始问句 |
| `thumbs_up` | tinyint | | 1-点赞 / 0-点踩 |
| `comment` | varchar(500) | | 用户备注文本 |
| **⬇️ 以下为 v6.4 新增字段（反馈分类与根因分析）** |
| `feedback_category` | varchar(32) | NULL | **问题分类**（分析后填充）：`route_error` / `intent_error` / `retrieval_failure` / `rank_bias` / `knowledge_gap` / `generation_error` / `context_truncation` / `other` |
| `root_cause_analysis` | json | NULL | **根因分析详细结果**：含 `summary`(根因简述)、`evidence_chain`(证据链数组)、`related_components`(定位到的环节，如`["task6_rerank","task7_ranking"]`) |
| `analysis_status` | varchar(16) | DEFAULT 'pending' | **分析状态机**：`pending`(待分析) / `analyzing`(分析中) / `completed`(已完成) / `failed`(失败) / `manual_review`(待人工复核，置信度低时转入) |
| `analyzed_at` | datetime | NULL | 根因分析完成时间 |
| `analysis_confidence` | decimal(3,2) | NULL | **本次分析的置信度**（0-1），用于自动/人工分流 |
| `optimization_suggestion` | json | NULL | **优化建议**：含 `suggestion_type`(`config_tuning`/`knowledge_update`/`prompt_optimization`/`model_switch`/`route_adjust`)、`detail`(具体内容)、`priority`(`high`/`medium`/`low`) |
| `optimization_status` | varchar(16) | DEFAULT 'pending' | **优化状态机**：`pending`(待审核) / `approved`(已批准) / `rejected`(已驳回) / `implemented`(已实施) / `verified`(已验证，指通过任务10回归测试) |
| `related_change_id` | varchar(64) | NULL | 关联的变更任务ID（如任务8的审批单号或Nacos变更单号） |
| `reviewer_id` | varchar(64) | NULL | 人工复核/批准人ID |
| `reviewed_at` | datetime | NULL | 人工复核时间 |
| **⬇️ 以下为 v6.2 及之前已有字段（保留，向前兼容）** |
| `rewritten_query_used` | text | NULL | 当时系统使用的改写Query |
| `rewrite_helpful` | tinyint | NULL | 改写是否有帮助（1-是/0-否） |
| `router_snapshot` | json | NULL | 路由决策快照 |
| `selected_kb` | varchar(32) | NULL | 用户实际浏览的知识库 |
| `search_snapshot` | json | NULL | 召回快照 |
| `retrieved_chunks` | json | NULL | 实际返回的 `chunk_uuid` 列表 |
| `search_helpful` | tinyint | NULL | 召回内容是否相关 |
| `rank_helpful` | tinyint | NULL | 排序是否合理 |
| `rank_snapshot` | json | NULL | 排序决策快照 |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |
| `updated_at` | datetime | DEFAULT CURRENT_TIMESTAMP ON UPDATE | 更新时间（v6.4新增，用于跟踪状态变更） |

**索引**：
```sql
CREATE INDEX idx_trace_id ON kb_user_feedback(trace_id);
CREATE INDEX idx_analysis_status ON kb_user_feedback(analysis_status);
CREATE INDEX idx_optimization_status ON kb_user_feedback(optimization_status);
CREATE INDEX idx_category_created ON kb_user_feedback(feedback_category, created_at);
CREATE INDEX idx_session_id ON kb_user_feedback(session_id);
```

---

### 1.20 反馈根因分析日志表：`kb_feedback_analysis_logs`（📌 v6.4 新增）

> **用途**：专门存储每一次根因分析任务的“原始输入-推理过程-输出结果”，解决黑盒问题。当人工复核发现分析错误时，可回溯此表调优 Prompt 或规则。

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `feedback_id` | bigint | FOREIGN KEY NOT NULL | 关联 `kb_user_feedback.id` |
| `trace_id` | varchar(64) | INDEX | 全链路追踪ID（冗余，便于快速关联） |
| `analysis_trigger` | varchar(16) | DEFAULT 'auto' | 触发方式：`auto`(系统自动) / `manual`(人工手动触发重分析) |
| `rule_hit` | varchar(64) | NULL | **规则初筛命中**：如 `recall_zero` / `low_confidence_route` / `empty_rank`，用于快速归因 |
| `retrieval_logs_snapshot` | json | NULL | 关联的 `kb_search_logs` 关键字段快照（召回数、Top分数、Rerank开关） |
| `rank_logs_snapshot` | json | NULL | 关联的 `kb_rank_logs` 关键字段快照（Top-5得分、多样性模式） |
| `agent_traces_snapshot` | json | NULL | **若为多步规划场景**，关联 `kb_agent_loop_traces` 的摘要（轮数、最终决策） |
| `llm_analysis_input` | text | NULL | 喂给 LLM 的完整 Prompt（含系统指令 + 脱敏后的业务上下文） |
| `llm_analysis_output` | text | NULL | LLM 原始输出（JSON 字符串） |
| `llm_model_used` | varchar(64) | NULL | 本次分析使用的模型（便于成本分摊） |
| `llm_prompt_tokens` | int | NULL | 本次分析的 Prompt Token 消耗 |
| `llm_completion_tokens` | int | NULL | 本次分析的生成 Token 消耗 |
| `final_category` | varchar(32) | NULL | 最终确认的分类（可能与 LLM 输出一致，也可能被规则覆盖） |
| `final_root_cause` | text | NULL | 最终确认的根因描述 |
| `confidence_score` | decimal(3,2) | NULL | 最终置信度 |
| `analysis_duration_ms` | int | NULL | 分析总耗时（规则+LLM） |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |

**索引**：
```sql
CREATE INDEX idx_feedback_id ON kb_feedback_analysis_logs(feedback_id);
CREATE INDEX idx_trace_id ON kb_feedback_analysis_logs(trace_id);
CREATE INDEX idx_created ON kb_feedback_analysis_logs(created_at);
```

---

### 1.21 反馈聚合统计表：`kb_feedback_stats`（📌 v6.4 新增）

> **用途**：定时（如每小时）从 `kb_user_feedback` 聚合数据写入此表，避免看板查询时对大表进行实时 `GROUP BY`，保障查询性能。同时存储趋势数据，便于对比周/月变化。

| 字段名 | 类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | bigint | PRIMARY KEY AUTO_INCREMENT | 自增主键 |
| `stat_date` | date | NOT NULL | 统计日期 |
| `source_kb` | varchar(32) | NOT NULL | 知识库名称 |
| `total_feedback` | int | DEFAULT 0 | 当日总反馈数 |
| `positive_count` | int | DEFAULT 0 | 点赞数 |
| `negative_count` | int | DEFAULT 0 | 点踩数 |
| **⬇️ 按问题类型细分（仅统计点踩）** |
| `category_route_error` | int | DEFAULT 0 | 路由错误导致点踩数 |
| `category_intent_error` | int | DEFAULT 0 | 意图理解错误导致点踩数 |
| `category_retrieval_failure` | int | DEFAULT 0 | 召回失败导致点踩数 |
| `category_rank_bias` | int | DEFAULT 0 | 排序偏差导致点踩数 |
| `category_knowledge_gap` | int | DEFAULT 0 | 知识缺失导致点踩数 |
| `category_generation_error` | int | DEFAULT 0 | 生成错误导致点踩数 |
| `category_context_truncation` | int | DEFAULT 0 | 上下文截断导致点踩数 |
| `category_other` | int | DEFAULT 0 | 其他/未分类点踩数 |
| **⬇️ 优化效率指标** |
| `avg_analysis_duration_ms` | int | NULL | 当日平均分析耗时 |
| `auto_analysis_rate` | decimal(3,2) | NULL | 自动分析完成率（`analysis_status='completed'` 占比） |
| `avg_confidence` | decimal(3,2) | NULL | 当日平均分析置信度 |
| `created_at` | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 |
| `updated_at` | datetime | DEFAULT CURRENT_TIMESTAMP ON UPDATE | 更新时间 |

**索引**：
```sql
CREATE UNIQUE INDEX uk_date_kb ON kb_feedback_stats(stat_date, source_kb);
CREATE INDEX idx_source_kb ON kb_feedback_stats(source_kb);
CREATE INDEX idx_stat_date ON kb_feedback_stats(stat_date);
```

---

## 二、Elasticsearch / Milvus / 配置中心 / 对象存储

**全部无变动**，完全继承 v6.3 的定义（ES索引、Milvus Collection、Nacos配置、对象存储路径均不受任务11影响）。

---

## 三、跨库一致性保障（📌 v6.4 新增反馈闭环规则）

在 v6.3 的基础上，新增以下针对反馈闭环的一致性规则：

| 场景 | 处理方式 |
| :--- | :--- |
| **反馈产生触发分析** | `kb_user_feedback` 写入时 `analysis_status='pending'`，通过 **消息队列（如 RabbitMQ）** 异步触发根因分析任务，不阻塞前端响应。 |
| **分析结果回写** | 分析完成后，更新 `kb_user_feedback` 的 `feedback_category`、`root_cause_analysis`、`analysis_status='completed'`、`analysis_confidence`；同时**写入** `kb_feedback_analysis_logs` 完整审计记录。 |
| **低置信度人工介入** | 若 `analysis_confidence < 0.60`，自动将 `analysis_status` 置为 `manual_review`，并触发内部运营工单（通过 `kb_notification_logs` 发送提醒）。 |
| **优化建议审批流** | 当 `optimization_status` 由 `pending` 变更为 `approved` 时，若建议类型为 `knowledge_update`，自动调用任务8的文档上传接口；若为 `config_tuning`，生成 Nacos 变更草稿供人工提交。 |
| **优化效果验证** | 当 `optimization_status` 变更为 `implemented` 后，自动触发任务10的评测流水线（运行回归测试），测试通过后自动扭转状态为 `verified`。 |
| **统计表刷新** | 定时任务（每小时）从 `kb_user_feedback` 按 `date(created_at)` + `source_kb` 聚合写入 `kb_feedback_stats`，并计算 `avg_analysis_duration_ms` 等衍生指标。 |
| **数据脱敏** | `kb_feedback_analysis_logs.llm_analysis_input` 中若包含用户敏感信息（如姓名、手机号），需在写入前进行脱敏处理（使用公司统一脱敏组件）。 |

---

## 四、📌 版本变更汇总（v6.3 → v6.4）

| 变更项 | 版本 | 类型 | 紧迫度 | 影响范围 |
| :--- | :--- | :--- | :--- | :--- |
| **`kb_user_feedback` 扩展 13 个字段（分类/分析/优化双状态机）** | **v6.4** | **修改** | **高** | 任务11（反馈闭环） |
| **`kb_feedback_analysis_logs` 新增表** | **v6.4** | **新增** | **高** | 任务11（审计与调试） |
| **`kb_feedback_stats` 新增表** | **v6.4** | **新增** | **中** | 任务11（看板性能优化） |
| v6.3 全部变更（任务9） | v6.3 | 新增/修改 | 高 | 任务9 |
| v6.2 全部变更（任务8） | v6.2 | 重构/新增 | 🔴阻塞级 | 任务8 |

---

> **文档结论**：
> - **任务1-9**：由 v6.3 完整覆盖，**本次零改动**。
> - **任务10（召回评测）**：**零改动**，按独立文件存储方案执行。
> - **任务11（端到端反馈闭环）**：由 **v6.4 增量（3处改动）** 完整覆盖，完全向后兼容，不影响任何历史表结构和已有业务逻辑。

此版本可直接进入开发。**实施顺序建议**：
1. 先执行 `kb_user_feedback` 的 ALTER 语句（新增字段，均为 NULLABLE，无阻塞风险）。
2. 创建 `kb_feedback_analysis_logs` 和 `kb_feedback_stats` 两张新表。
3. 部署异步分析消费者（消费 MQ 中的反馈 ID）。
4. 部署定时统计任务（每小时刷新 `kb_feedback_stats`）。

如需我生成具体的 DDL 迁移脚本（`.sql` 文件）或 Python ORM 模型代码（SQLAlchemy），请随时告知。