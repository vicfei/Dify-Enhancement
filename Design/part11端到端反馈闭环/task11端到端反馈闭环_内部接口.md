收到。在完成了**对外接口（RESTful API）**的设计后，本次我们深入到系统内部，设计**任务11的内部接口**。

内部接口主要面向**后端服务模块**、**异步任务（Celery）**、**数据访问层（Repository）**以及**第三方服务适配器（Dify Client）**。它们共同构成了反馈闭环的“引擎”和“骨架”。

以下为 **《任务11：端到端反馈闭环内部接口设计文档（v1.0）》**。

---

# 任务11：端到端反馈闭环内部接口设计文档（v1.0）

> **版本**：v1.0 | **关联外部API**：`/api/v1/feedback/*` | **关联数据模型**：v6.4

## 1. 总体架构与模块划分

内部系统采用 **分层六边形架构**，核心模块划分如下：

| 模块 | 层级 | 核心职责 |
| :--- | :--- | :--- |
| **`interface/api/`** | 适配层 | 已定义的 REST Controller，负责参数校验与响应序列化。 |
| **`application/`** | 应用服务层 | 编排业务用例（Service），发布/消费事件，事务管理。 |
| **`domain/`** | 领域层 | 核心业务逻辑（根因分析引擎、规则引擎、置信度计算）。 |
| **`infrastructure/`** | 基础设施层 | 数据持久化（Repository）、外部服务调用（Dify Client）、消息队列（MQ）。 |

---

## 2. 应用服务层接口（Application Services）

### 2.1 `FeedbackCollector`（反馈采集服务）
**职责**：接收反馈、去重、存储、触发异步分析任务。

```python
class FeedbackCollector:
    def collect(self, dto: FeedbackCollectDTO) -> FeedbackEntity:
        """
        1. 校验并持久化反馈（analysis_status=pending）
        2. 若 metadata 缺失，调用 DifyClient 补全上下文
        3. 发布 FeedbackCreatedEvent 到消息队列
        """
        pass

    def sync_from_dify(self, app_id: str, start: datetime, end: datetime) -> int:
        """
        定时任务/手动触发：从 Dify API 拉取历史反馈并批量写入
        """
        pass
```

### 2.2 `AnalysisOrchestrator`（根因分析编排服务）
**职责**：编排完整的“证据收集→规则初筛→LLM分析→置信度评估→结果持久化”流程。

```python
class AnalysisOrchestrator:
    def analyze(self, feedback_id: int, force: bool = False) -> AnalysisResult:
        """
        1. 查询反馈记录
        2. 初始化 EvidenceContext（证据上下文）
        3. 调用 Domain 层的 RootCauseAnalyzer
        4. 持久化分析结果至 kb_user_feedback 和 kb_feedback_analysis_logs
        5. 若 confidence < 0.6，触发人工复核通知
        """
        pass

    def batch_analyze(self, filters: AnalysisFilter) -> str:
        """生成批量分析任务，返回 batch_task_id 供轮询进度"""
        pass
```

### 2.3 `OptimizationManager`（优化建议管理服务）
**职责**：管理优化建议的状态机流转（待审批 → 已批准 → 已实施 → 已验证）。

```python
class OptimizationManager:
    def get_pending_list(self, page: int, size: int) -> list[OptimizationSuggestion]:
        """获取待人工审批的建议列表"""
        pass

    def transition_status(self, feedback_id: int, event: OptimizationEvent, operator: str):
        """
        状态机驱动：
        - APPROVE：生成实施工单（若类型为config_tuning，生成Nacos草稿）
        - IMPLEMENT：调用 Infrastructure 层执行变更
        - VERIFY：触发任务10的评测流水线（回调 VerificationService）
        """
        pass
```

---

## 3. 领域层核心接口（Domain Layer）

### 3.1 `EvidenceCollector`（证据收集器）⭐战略设计模式
**职责**：根据 `trace_id` 聚合全链路日志（路由、意图、召回、排序），构建标准化的 `EvidenceContext`。

**设计原则**：面向接口编程，未来接入 Wiki 和知识图谱时，只需新增 `WikiCollector` 和 `KGCollector` 实现，不影响核心分析逻辑。

```python
class EvidenceContext:
    trace_id: str
    query: str
    rewritten_query: str
    route_decision: str
    search_logs: list[SearchLogEntry]   # 含召回分数、Rerank信息
    rank_logs: list[RankLogEntry]       # 含最终排序TopK
    # 未来扩展字段（预留）
    kg_evidence: dict | None = None     
    wiki_context: str | None = None

class IEvidenceCollector(ABC):
    @abstractmethod
    def collect(self, trace_id: str) -> EvidenceContext:
        """从三库（MySQL/ES/Milvus）及日志表中拉取数据，组装上下文"""
        pass

# 现有实现：基于 v6.3 日志表的收集器
class SQLAlchemyEvidenceCollector(IEvidenceCollector):
    def collect(self, trace_id: str) -> EvidenceContext:
        # 关联查询 kb_query_logs, kb_search_logs, kb_rank_logs
        pass
```

### 3.2 `RootCauseAnalyzer`（根因分析器）⭐核心算法
**职责**：运行“漏斗式分析Pipeline”。

```python
class RootCauseAnalyzer:
    def __init__(self, rule_engine: RuleEngine, llm_analyzer: LLMAnalyzer):
        self.rule_engine = rule_engine
        self.llm_analyzer = llm_analyzer

    def analyze(self, context: EvidenceContext) -> AnalysisResult:
        # 1. 规则初筛（低成本快速归类）
        rule_result = self.rule_engine.match(context)
        
        # 2. 若规则置信度足够高（>0.9）且匹配明确，提前返回
        if rule_result.confidence > 0.9:
            return rule_result
        
        # 3. 否则，调用 LLM 进行深度归因
        llm_result = self.llm_analyzer.invoke(context, rule_result.hints)
        
        # 4. 融合规则与LLM结果，计算最终置信度
        return self._merge_results(rule_result, llm_result)
```

### 3.3 `RuleEngine`（规则引擎）
**职责**：基于预定义规则快速命中常见问题类型（避免每次消耗LLM Token）。

```python
class RuleEngine:
    RULES = [
        Rule(condition=lambda c: c.search_logs.recall_count == 0, 
             category="retrieval_failure", confidence=0.95),
        Rule(condition=lambda c: c.route_decision.confidence < 0.5, 
             category="route_error", confidence=0.90),
        Rule(condition=lambda c: c.rank_logs.top1_score < 0.3, 
             category="rank_bias", confidence=0.80),
    ]
    
    def match(self, context: EvidenceContext) -> Optional[RuleResult]:
        for rule in self.RULES:
            if rule.condition(context):
                return RuleResult(category=rule.category, confidence=rule.confidence, hints=...)
        return None
```

### 3.4 `LLMAnalyzer`（LLM 深度分析器）
**职责**：封装 Prompt 工程，调用自研 `session.model.llm` 接口。

```python
class LLMAnalyzer:
    def invoke(self, context: EvidenceContext, hints: list[str]) -> LLMAnalysisResult:
        """
        1. 根据 hints 构建 System Prompt（参考之前的Prompt模板）
        2. 调用 self.session.model.llm.invoke()
        3. 解析结构化 JSON 输出，返回分类、根因、优化建议
        """
        prompt = self._build_prompt(context, hints)
        response = self.llm_client.chat(prompt, response_format=AnalysisSchema)
        return LLMAnalysisResult(
            category=response.category,
            root_cause=response.root_cause,
            suggestion=response.suggestion,
            confidence=response.confidence
        )
```

---

## 4. 基础设施层接口（Infrastructure Layer）

### 4.1 `FeedbackRepository`（数据持久化）
```python
class FeedbackRepository:
    def save(self, entity: FeedbackEntity) -> int: ...
    def get_by_id(self, feedback_id: int) -> FeedbackEntity: ...
    def update_analysis(self, feedback_id: int, result: AnalysisResult): ...
    def update_optimization_status(self, feedback_id: int, status: str, reviewer: str): ...
    def find_pending_analyses(self, limit: int) -> list[FeedbackEntity]: ...
    def find_by_filters(self, filters: dict) -> PaginatedResult: ...
```

### 4.2 `DifyClient`（反向调用适配器）
**职责**：与 Dify Service API 交互，拉取缺失的上下文或历史反馈。

> 基于 `repomix-output.xml` 中的 Dify 接口定义，封装如下方法：

```python
class DifyClient:
    def get_message_detail(self, message_id: str) -> DifyMessage:
        """调用 GET /messages 获取单条消息的完整上下文"""
        # 对应 Dify API: GET /messages?conversation_id=... （需先获取conversation_id）
        pass

    def list_app_feedbacks(self, app_id: str, start: datetime, end: datetime) -> list[DifyFeedback]:
        """调用 GET /app/feedbacks 批量拉取反馈"""
        # 对应 repomix 中的 /app/feedbacks 接口
        pass

    def get_conversation_history(self, conversation_id: str) -> list[Message]:
        """用于当 trace_id 缺失时，通过 conversation_id 还原上下文"""
        pass
```

### 4.3 `ChangeExecutor`（变更执行器）
**职责**：执行经过审批的优化建议（对接任务4/8或Nacos）。

```python
class ChangeExecutor:
    def apply_config_tuning(self, change_detail: dict) -> str:
        """调用 Nacos API 修改 runtime 配置（如 search_hybrid.yaml 的权重）"""
        pass

    def trigger_doc_reindex(self, doc_id: str) -> str:
        """触发任务8的文档重新索引（知识缺失场景）"""
        pass

    def update_router_rule(self, rule_change: dict) -> str:
        """触发任务4的 `POST /api/v1/reload` 热加载路由规则"""
        pass
```

### 4.4 `VerificationClient`（验证客户端 - 对接任务10）
```python
class VerificationClient:
    def trigger_evaluation(self, run_config: dict) -> str:
        """异步调用任务10的评测 Runner，返回 evaluation_run_id"""
        # 调用任务10外部接口 POST /api/v1/evaluation/runs
        pass

    def get_evaluation_status(self, run_id: str) -> dict:
        """轮询评测状态"""
        pass
```

---

## 5. 异步消息契约（MQ Events）

采用 **RabbitMQ/Pulsar** 解耦核心流程，内部接口通过事件驱动。

### 5.1 `FeedbackCreatedEvent`（生产者：FeedbackCollector；消费者：AnalysisOrchestrator）
```json
{
  "event_type": "feedback.created",
  "timestamp": "2026-08-09T10:00:00Z",
  "payload": {
    "feedback_id": 12345,
    "trace_id": "trace_xxx",
    "rating": "thumbs_down",
    "source": "dify_web"
  }
}
```

### 5.2 `AnalysisCompletedEvent`（生产者：AnalysisOrchestrator；消费者：StatsService / NotificationService）
```json
{
  "event_type": "analysis.completed",
  "payload": {
    "feedback_id": 12345,
    "category": "retrieval_failure",
    "confidence": 0.87,
    "suggestion_generated": true
  }
}
```

### 5.3 `OptimizationApprovedEvent`（生产者：OptimizationManager；消费者：ChangeExecutor）
```json
{
  "event_type": "optimization.approved",
  "payload": {
    "feedback_id": 12345,
    "suggestion_type": "config_tuning",
    "detail": { "config_key": "search.rrf_k", "new_value": 60 },
    "operator": "admin_001"
  }
}
```

---

## 6. 关键数据流转时序图（内部调用）

```
[前端/Dify] → [REST Controller] 
    → (1) FeedbackCollector.collect() 
        → (2) FeedbackRepository.save() 
        → (3) MQ.publish(FeedbackCreatedEvent)
        
[MQ Consumer] → (4) AnalysisOrchestrator.analyze()
    → (5) EvidenceCollector.collect(trace_id) 
        → (6) SQLAlchemyEvidenceCollector 查询三库日志
    → (7) RootCauseAnalyzer.analyze(context)
        → (8) RuleEngine.match()  [规则初筛]
        → (9) LLMAnalyzer.invoke() [若需深度分析]
    → (10) FeedbackRepository.update_analysis()
    → (11) MQ.publish(AnalysisCompletedEvent)
    
[人工审批看板] → (12) OptimizationManager.transition_status(APPROVE)
    → (13) ChangeExecutor.apply_config_tuning() [仅当类型为config_tuning]
    
[实施完成后] → (14) OptimizationManager.transition_status(IMPLEMENT)
    → (15) VerificationClient.trigger_evaluation() [对接任务10]
```

---

## 7. 异常处理与重试策略（内部规范）

| 组件 | 异常类型 | 重试策略 |
| :--- | :--- | :--- |
| `EvidenceCollector` | 数据库超时/连接中断 | 指数退避重试3次，降级返回部分上下文并标记 `is_incomplete=true` |
| `LLMAnalyzer` | LLM 限流（429）或超时 | 自动降级为规则结果；若规则无法匹配，标记 `manual_review` |
| `DifyClient` | Dify API 不可用 | 记录日志，不阻塞主流程；等待定时任务补偿同步 |
| `ChangeExecutor` | Nacos 写入失败 | 回滚状态至 `pending`，发送告警邮件至运维组 |

---

此内部接口文档为开发团队提供了清晰的**模块边界**、**类方法契约**以及**数据流转路径**。下一步可据此编写 Python 代码骨架（遵循 DDD 分层结构），并实现 MQ 消费者与 Celery 任务绑定。

如需我生成具体的 Python 抽象类（ABC）代码或 Celery Task 定义，请随时告知。