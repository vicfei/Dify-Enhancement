明白了，非常抱歉我上一条回复偏向了召回测评。现在回到正题：**任务9多步规划（ReAct + 自主评估）的内部接口文档**。

> **与外部API的区别**：
> - **外部API**（`/api/v1/agent/run`）面向**调用方**（Dify应用、前端），定义的是请求/响应格式。
> - **内部接口**（本文档）面向**自研服务内部各模块**，定义的是Python类之间的契约，供开发人员实现。


# 任务9：多步规划与自主评估 —— 内部接口设计文档（v1.0）

> **版本说明**：本文档定义自研服务内部各模块（Orchestrator、Evaluator、StateManager、ToolInvoker、MemoryManager）的编程接口，与已有的任务4（路由）、任务5（意图理解）、任务6（召回）、任务7（排序）协同工作。


## 一、整体架构与模块划分

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        对外接口层（API Layer）                             │
│              POST /api/v1/agent/run  →  AgentRouter                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      核心编排器（Agent Orchestrator）                       │
│                   实现 ReAct 循环：Think → Act → Observe                   │
└───────────────┬─────────────────┬─────────────────┬───────────────────────┘
                │                 │                 │
                ▼                 ▼                 ▼
┌───────────────────┐ ┌───────────────────┐ ┌───────────────────────────────┐
│  ToolInvoker      │ │  StateManager     │ │  Evaluator                    │
│  (调用任务4/6/7)  │ │  (状态持久化)     │ │  (置信度/矛盾/缺失检测)       │
└───────────────────┘ └───────────────────┘ └───────────────────────────────┘
                │                 │                 │
                └─────────────────┼─────────────────┘
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        数据访问层（DAO）                                    │
│        kb_sessions / kb_agent_loop_traces / kb_evaluation_results          │
└─────────────────────────────────────────────────────────────────────────────┘
```

**模块职责**：

| 模块 | 职责 |
| :--- | :--- |
| **Agent Orchestrator** | 运行ReAct主循环，协调各子模块，控制迭代终止 |
| **ToolInvoker** | 统一封装对任务4/5/6/7的调用，处理超时与降级 |
| **Evaluator** | 计算综合置信度，识别缺失信息与矛盾，输出决策建议 |
| **StateManager** | 管理会话状态（累积知识、轮次、中间结果）的读写与恢复 |
| **MemoryManager** | 管理对话历史摘要与压缩（Token限制） |
| **AskHandler** | 处理“反问用户”的暂停与恢复逻辑 |


## 二、核心数据实体定义（Pydantic）

### 2.1 AgentState（核心状态对象）

```python
from typing import List, Optional, Dict, Any
from pydantic import BaseModel, Field
from datetime import datetime
from enum import Enum

class AgentDecision(str, Enum):
    CONTINUE = "continue"      # 继续下一轮检索
    ASK = "ask"                # 反问用户
    ANSWER = "answer"          # 直接回答
    REJECT = "reject"          # 拒绝回答
    FINISH = "finish"          # 正常结束

class AgentState(BaseModel):
    # 基础信息
    conversation_id: str
    query: str                           # 原始用户问题
    current_round: int = 0               # 当前已执行轮次
    max_rounds: int = 3                  # 最大轮次限制

    # 累积知识
    accumulated_knowledge: str = ""      # 已累积的上下文摘要（压缩后）
    retrieved_docs: List[Dict[str, Any]] = []   # 已检索的文档列表
    tool_call_history: List[Dict] = []   # 工具调用历史

    # 置信度与评估
    confidence_score: float = 0.0        # 综合置信度（0~1）
    missing_info: List[str] = []         # 缺失信息清单
    contradictions: List[Dict] = []      # 检测到的矛盾

    # 决策状态
    decision: AgentDecision = AgentDecision.CONTINUE
    final_answer: Optional[str] = None

    # 元数据
    started_at: datetime = Field(default_factory=datetime.now)
    updated_at: datetime = Field(default_factory=datetime.now)
    usage: Dict[str, int] = {"prompt_tokens": 0, "completion_tokens": 0}
```

### 2.2 EvaluationResult（评估节点输出）

```python
class EvaluationResult(BaseModel):
    overall_confidence: float            # 综合置信度
    dimension_relevance: float           # 相关性得分（α）
    dimension_coverage: float            # 覆盖度得分（β）
    dimension_authority: float           # 权威性得分（γ）
    dimension_consistency: float         # 一致性得分（δ）
    missing_info: List[str]              # 缺失信息
    contradictions: List[Dict[str, str]] # 矛盾详情
    decision: AgentDecision              # 建议决策
    reasoning: str                       # 评估推理过程（可解释性）
```

### 2.3 ToolCallResult（工具调用返回）

```python
class ToolCallResult(BaseModel):
    tool_name: str                       # "route" / "retrieve" / "rank" / "understand"
    success: bool
    result: Any                          # 各工具返回的具体数据
    latency_ms: int
    error: Optional[str] = None
```


## 三、模块接口定义

### 3.1 Agent Orchestrator（核心编排器）

负责执行ReAct主循环，是任务9的“大脑”。

**接口**：

```python
from abc import ABC, abstractmethod
from typing import Generator, Optional

class IAgentOrchestrator(ABC):
    @abstractmethod
    def run(
        self,
        query: str,
        conversation_id: Optional[str] = None,
        user: Optional[str] = None,
        config: Optional[AgentConfig] = None
    ) -> Generator[AgentEvent, None, None]:
        """
        执行多步规划任务，返回事件流（Generator）

        Args:
            query: 用户原始问题
            conversation_id: 会话ID，为空则新建
            user: 用户标识
            config: 运行时配置（覆盖默认）

        Yields:
            AgentEvent: 流式事件（见外部接口文档）
        """
        pass

    @abstractmethod
    def resume_from_ask(
        self,
        form_token: str,
        user_response: Dict[str, Any],
        user: str
    ) -> Generator[AgentEvent, None, None]:
        """
        从“反问”状态恢复执行

        Args:
            form_token: 反问时返回的token
            user_response: 用户填写的回复
            user: 用户标识
        """
        pass

    @abstractmethod
    def stop(self, task_id: str, user: str) -> bool:
        """停止正在运行的任务"""
        pass
```

**内部核心方法**（供`run()`调用）：

```python
    def _think(self, state: AgentState) -> ThoughtResult:
        """思考：根据当前状态决定下一步要调用哪个工具"""
        pass

    def _act(self, thought: ThoughtResult, state: AgentState) -> ToolCallResult:
        """行动：调用工具（路由/召回/排序）"""
        pass

    def _observe(self, result: ToolCallResult, state: AgentState) -> EvaluationResult:
        """观察：评估工具返回结果，更新状态"""
        pass

    def _should_terminate(self, state: AgentState, eval_result: EvaluationResult) -> bool:
        """判断是否终止循环（达到轮次上限/置信度达标/决策为answer或reject）"""
        pass
```

### 3.2 ToolInvoker（工具调用器）

统一封装对任务4/5/6/7的调用，处理超时、重试与降级。

**接口**：

```python
from typing import Dict, Any, Optional

class IToolInvoker(ABC):
    @abstractmethod
    def call_route(self, query: str, context: Optional[Dict] = None) -> Dict[str, Any]:
        """
        调用任务4（路由）
        返回：{"target_kb": "fund", "confidence": 0.92, "ambiguous": False}
        """
        pass

    @abstractmethod
    def call_understand(self, query: str, session_id: Optional[str] = None) -> Dict[str, Any]:
        """
        调用任务5（意图理解）
        返回：{"intent_type": "analysis", "sub_questions": [...], "rewritten_query": "..."}
        """
        pass

    @abstractmethod
    def call_retrieve(
        self,
        query: str,
        source_kb: str,
        top_k: int = 10,
        filters: Optional[Dict] = None,
        rerank_enabled: bool = True
    ) -> Dict[str, Any]:
        """
        调用任务6（召回）
        返回：{"candidates": [...], "total_hits": 50, "rerank_used": True}
        """
        pass

    @abstractmethod
    def call_rank(
        self,
        candidates: List[Dict],
        user_context: Optional[Dict] = None,
        weights: Optional[Dict] = None
    ) -> Dict[str, Any]:
        """
        调用任务7（排序）
        返回：{"ranked_chunks": [...], "weights_used": {...}}
        """
        pass

    @abstractmethod
    def call_evaluate(
        self,
        query: str,
        candidates: List[Dict],
        history: List[Dict]
    ) -> EvaluationResult:
        """
        调用新增的评估模块（Evaluator）
        返回：EvaluationResult
        """
        pass
```

**降级策略**：

- 路由超时 → 使用默认知识库（`source_kb="default"`）
- 召回超时 → 使用上一轮结果或返回空列表，触发`missing_info`
- 排序超时 → 按召回分数简单排序
- Rerank不可用 → 自动降级为RRF融合（已在任务6设计中定义）

### 3.3 Evaluator（评估模块）

这是任务9的**核心新增模块**，负责置信度计算与决策建议。

**接口**：

```python
class IEvaluator(ABC):
    @abstractmethod
    def evaluate(
        self,
        query: str,
        candidates: List[Dict[str, Any]],
        history: List[Dict[str, Any]],
        context: Optional[Dict] = None
    ) -> EvaluationResult:
        """
        综合评估检索结果

        Args:
            query: 当前子问题/原始问题
            candidates: 任务7返回的排序后候选列表（含分数、来源等）
            history: 历史轮次的检索结果（用于一致性检测）
            context: 额外上下文（用户角色、偏好等）

        Returns:
            EvaluationResult: 含置信度、缺失信息、矛盾、决策建议
        """
        pass

    @abstractmethod
    def _calc_relevance(self, candidates: List[Dict]) -> float:
        """基于召回分数计算相关性得分"""
        pass

    @abstractmethod
    def _calc_coverage(self, query: str, candidates: List[Dict]) -> float:
        """评估候选结果是否覆盖了问题的所有关键维度（使用LLM辅助）"""
        pass

    @abstractmethod
    def _calc_authority(self, candidates: List[Dict]) -> float:
        """基于文档来源权重计算权威性得分（复用任务7的authority系数）"""
        pass

    @abstractmethod
    def _calc_consistency(self, history: List[Dict], candidates: List[Dict]) -> float:
        """检测多轮结果之间是否存在矛盾（数值/事实）"""
        pass

    @abstractmethod
    def _detect_missing_info(self, query: str, candidates: List[Dict]) -> List[str]:
        """识别缺失的关键信息（使用LLM辅助）"""
        pass

    @abstractmethod
    def _detect_contradictions(self, history: List[Dict], candidates: List[Dict]) -> List[Dict]:
        """检测矛盾（同一指标不同值、相反结论等）"""
        pass
```

**置信度计算公式**（内部实现）：

```
confidence = α × relevance + β × coverage + γ × authority + δ × consistency

默认权重：α=0.35, β=0.25, γ=0.20, δ=0.20
（可通过Nacos动态调整）
```

### 3.4 StateManager（状态管理器）

负责会话状态的持久化、恢复与更新，对应MySQL表`kb_sessions.agent_state`。

**接口**：

```python
class IStateManager(ABC):
    @abstractmethod
    def get_state(self, conversation_id: str) -> Optional[AgentState]:
        """根据conversation_id加载状态，不存在返回None"""
        pass

    @abstractmethod
    def create_state(self, conversation_id: str, query: str, config: AgentConfig) -> AgentState:
        """创建新状态，写入数据库"""
        pass

    @abstractmethod
    def update_state(self, state: AgentState) -> None:
        """更新状态（每次循环后调用）"""
        pass

    @abstractmethod
    def mark_paused(self, conversation_id: str, form_token: str) -> None:
        """标记会话为“等待反问回复”状态"""
        pass

    @abstractmethod
    def mark_completed(self, conversation_id: str, final_answer: str) -> None:
        """标记会话为“已完成”"""
        pass

    @abstractmethod
    def get_by_form_token(self, form_token: str) -> Optional[str]:
        """根据form_token查找conversation_id（用于恢复反问）"""
        pass
```

### 3.5 MemoryManager（记忆管理器）

管理对话历史，在Token限制内压缩上下文。

**接口**：

```python
class IMemoryManager(ABC):
    @abstractmethod
    def compress_history(
        self,
        history: List[Dict[str, Any]],
        max_tokens: int = 2000
    ) -> str:
        """
        将历史对话压缩为摘要文本，不超过max_tokens

        策略：保留最新2轮完整对话，更早的用LLM生成摘要
        """
        pass

    @abstractmethod
    def build_prompt(
        self,
        state: AgentState,
        memory_summary: str,
        system_instruction: str
    ) -> List[Dict[str, str]]:
        """
        构建发送给LLM的Prompt Messages
        包含：系统指令 + 历史摘要 + 当前状态 + 问题
        """
        pass
```

### 3.6 AskHandler（反问处理器）

处理“反问用户”的暂停与恢复逻辑，对应外部API的`/agent/ask/submit`。

**接口**：

```python
class IAskHandler(ABC):
    @abstractmethod
    def create_form(
        self,
        conversation_id: str,
        question: str,
        options: Optional[List[str]] = None,
        expires_in_seconds: int = 86400
    ) -> str:
        """
        生成反问表单，返回form_token（写入kb_pending_questions表）
        """
        pass

    @abstractmethod
    def submit_response(
        self,
        form_token: str,
        user_response: Dict[str, Any],
        user: str
    ) -> Optional[AgentState]:
        """
        用户提交回复后，恢复状态并返回
        若form_token无效或已过期，返回None
        """
        pass

    @abstractmethod
    def cleanup_expired(self) -> int:
        """定时清理过期的反问表单（expires_at < now），释放资源"""
        pass
```


## 四、依赖关系与现有模块集成

| 任务9模块 | 依赖的现有模块 | 依赖方式 |
| :--- | :--- | :--- |
| **ToolInvoker** | 任务4（路由） | HTTP调用 `/api/v1/route` |
| | 任务5（意图理解） | HTTP调用 `/api/v1/query/process` |
| | 任务6（召回） | HTTP调用 `/api/v1/search` 或内部Python调用 |
| | 任务7（排序） | HTTP调用 `/api/v1/rank` |
| **Evaluator** | 任务7的权威性系数 | 读取Nacos配置 `ranking_user_weights.yaml` |
| | 任务6的召回分数 | 直接从`candidates`中读取`recall_score` |
| **StateManager** | MySQL表 `kb_sessions` | 读写`agent_state` JSON字段 |
| **MemoryManager** | LLM模型 | 调用工作空间默认LLM生成摘要 |
| **AskHandler** | MySQL表 `kb_pending_questions` | 读写反问队列 |
| **AgentOrchestrator** | 全部上述模块 | 组合调用 |


## 五、序列图：一次完整的ReAct循环

```
用户       API层       Orchestrator    ToolInvoker    Evaluator    StateManager
 │           │              │              │             │              │
 │─POST /run─▶              │              │             │              │
 │           │─创建状态──────▶              │             │              │
 │           │              │───load_state──│             │              │
 │           │              │◀──────state───│             │              │
 │           │              │              │             │              │
 │           │              │─think()──────│             │              │
 │           │              │ (调用LLM)     │             │              │
 │           │              │              │             │              │
 │           │              │─act()────────▶             │              │
 │           │              │              │─call_route──│              │
 │           │              │              │─call_retrieve│             │
 │           │              │              │─call_rank───│              │
 │           │              │◀─────result───│             │              │
 │           │              │              │             │              │
 │           │              │─observe()─────────────────▶              │
 │           │              │              │             │─evaluate────│
 │           │              │              │             │◀────eval_res│
 │           │              │              │             │              │
 │           │              │─should_terminate?──────────              │
 │           │              │ (评估决策)   │             │              │
 │           │              │              │             │              │
 │           │◀─────event: agent_evaluation───────────────              │
 │           │◀─────event: agent_thought───────────────────────────────│
 │           │              │              │             │              │
 │           │   (若decision=continue，重复上述循环)                    │
 │           │   (若decision=answer，生成最终答案)                      │
 │           │   (若decision=ask，创建反问表单)                         │
 │           │              │              │             │              │
 │           │◀─────event: agent_answer / agent_ask───────────────────│
 │           │◀─────event: agent_end───────────────────────────────────│
 │           │              │              │             │              │
 │           │              │─save_state──▶│             │              │
 │           │              │              │             │              │
◀─────────────响应流结束─────│              │             │              │
```


## 六、异常处理与重试策略

| 异常类型 | 处理方式 | 对调用方表现 |
| :--- | :--- | :--- |
| **工具调用超时**（任务6/7超时） | 内部重试3次，仍失败则跳过该工具，`missing_info`中记录 | 继续下一轮或触发`ask` |
| **LLM调用失败**（think/evaluate时） | 使用规则降级（如固定返回`continue`） | 增加`warning`日志，不影响流程 |
| **StateManager写失败** | 记录错误日志，但继续执行（状态保存在内存） | 不影响当前请求，但断线恢复会丢失 |
| **AskHandler过期** | 超时后自动转为`reject` | 返回`agent_end`事件，状态为rejected |
| **达到最大轮次** | 强制终止，当前`confidence`即为最终值 | 触发`answer`（即使置信度低）或`reject` |


## 七、可观测性扩展点

### 7.1 日志接口（与任务9已设计的 `kb_agent_loop_traces` 表对齐）

```python
class ILogger(ABC):
    @abstractmethod
    def log_thought(self, trace_id: str, round: int, thought: str) -> None:
        """记录思考步骤（写入 kb_agent_loop_traces）"""
        pass

    @abstractmethod
    def log_action(self, trace_id: str, round: int, tool_name: str, input: dict, output: Any) -> None:
        """记录工具调用（写入 kb_agent_loop_traces）"""
        pass

    @abstractmethod
    def log_evaluation(self, trace_id: str, round: int, eval_result: EvaluationResult) -> None:
        """记录评估结果（写入 kb_evaluation_results）"""
        pass
```

### 7.2 指标埋点

在Orchestrator中集成OpenTelemetry：
- `agent.round.duration`：每轮耗时
- `agent.toolcall.latency`：各工具调用延迟
- `agent.confidence.score`：最终置信度分布
- `agent.decision.count`：各决策类型统计


## 八、总结与开发顺序建议

| 开发顺序 | 模块 | 预估工时 | 依赖 |
| :--- | :--- | :--- | :--- |
| 1 | **StateManager**（含DAO） | 2天 | MySQL表已设计（v6.3） |
| 2 | **ToolInvoker**（封装现有任务4/6/7） | 2天 | 各模块接口已定稿 |
| 3 | **Evaluator**（核心新增） | 3天 | 无外部依赖，需LLM辅助 |
| 4 | **MemoryManager** | 1.5天 | 需LLM模型 |
| 5 | **AskHandler** | 1.5天 | 需StateManager支持 |
| 6 | **AgentOrchestrator**（主循环） | 3天 | 依赖以上全部 |
| 7 | **集成测试 & 联调** | 2天 | 全部完成 |
| **合计** | | **15天（3周）** | |

**可并行推进**：第1~4项可由不同开发人员并行开发，第6项在2~3天后即可启动骨架开发。