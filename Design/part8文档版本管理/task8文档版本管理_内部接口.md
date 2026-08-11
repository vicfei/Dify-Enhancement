明白了！您需要的是**任务8“文档版本管理与整体联调”的内部接口契约**，而非上一轮给出的外部RESTful API。

这套接口是自研服务内部的**模块边界定义**，用于指导开发团队在任务8阶段实现“版本检测→审批流转→差量索引→全链路联调”的核心逻辑。设计沿用了你们项目已有的 `I` 前缀和 Pydantic 风格，确保与任务4-7的代码骨架风格统一。

---

# 任务8 内部接口契约设计（v1.0）

> **设计目标**：为文档版本管理与整体联调提供清晰的模块边界，实现“变更自动发现 → 人工审批闸门 → 三库差量更新 → 全链路可观测”的闭环。

---

## 一、模块架构概览

| 模块 | 职责 | 依赖 |
| :--- | :--- | :--- |
| **`IVersionDetector`** | 检测文档变更（file_hash比对），决定是否需要进入审批流程 | `kb_documents` 表 |
| **`IApprovalWorkflow`** | 管理审批状态流转（pending_approval → approved/rejected），记录审批人/时间/原因 | `kb_documents` 表、`kb_version_changes` 表 |
| **`IDeltaIndexer`** | 执行三库（MySQL/ES/Milvus）差量更新：旧块逻辑删除，新块插入 | 任务2的 `ITripleWriter` 接口 |
| **`INotificationSender`** | 发送变更提醒（邮件/后台待办），记录通知日志 | `kb_notification_logs` 表、邮件服务 |
| **`ITraceAggregator`** | 根据 trace_id 聚合全链路日志（路由→意图→召回→排序→反馈） | `kb_router_logs`、`kb_query_logs`、`kb_search_logs`、`kb_rank_logs`、`kb_user_feedback` |
| **`ISystemHealthChecker`** | 检查所有依赖组件（MySQL/ES/Milvus/Redis/Dify API）的健康状态 | 各组件客户端 |

---

## 二、详细接口定义（Python Abstract Base Classes）

### 1. 版本检测器接口 `IVersionDetector`

> 负责文档上传时的变更检测，决定走“首次发布”还是“版本更新审批”流程。

```python
from abc import ABC, abstractmethod
from pydantic import BaseModel
from typing import Optional, Dict, Any
from enum import Enum

class ChangeType(str, Enum):
    NEW_UPLOAD = "new_upload"          # 全新文档
    VERSION_UPGRADE = "version_upgrade" # 已存在文档的新版本
    DUPLICATE = "duplicate"            # 相同hash，无需处理

class DetectedChange(BaseModel):
    """检测结果"""
    doc_uuid: str
    file_hash: str
    change_type: ChangeType
    existing_doc_id: Optional[str] = None  # 仅当 change_type = version_upgrade 时存在
    old_version: Optional[str] = None
    new_version: str                       # 自动生成的版本号，如 "v2.0"
    change_summary: Optional[str] = None   # 变更摘要（可选，可由LLM生成或人工填写）

class IVersionDetector(ABC):
    @abstractmethod
    def detect_change(
        self, 
        file_hash: str, 
        source_kb: str, 
        file_name: str
    ) -> DetectedChange:
        """
        根据 file_hash 和 source_kb 查询已存在的文档记录，判断变更类型。
        - 若完全不存在 → NEW_UPLOAD
        - 若存在但 file_hash 不同 → VERSION_UPGRADE
        - 若存在且 file_hash 相同 → DUPLICATE
        """
        pass

    @abstractmethod
    def generate_next_version(self, doc_id: str) -> str:
        """根据当前版本号递增，如 v1.0 → v2.0"""
        pass
```

---

### 2. 审批工作流接口 `IApprovalWorkflow`

> 管理文档状态机的流转，记录审批操作。

```python
class ApprovalAction(str, Enum):
    APPROVE = "approve"
    REJECT = "reject"

class ApprovalRequest(BaseModel):
    doc_id: str
    action: ApprovalAction
    approver_id: str
    reject_reason: Optional[str] = None

class ApprovalResult(BaseModel):
    doc_id: str
    new_lifecycle_status: str  # published | rejected
    previous_status: str       # 回退时的原始状态
    approved_at: str
    message: str

class IApprovalWorkflow(ABC):
    @abstractmethod
    def submit_for_approval(self, doc_id: str, detected_change: DetectedChange) -> None:
        """
        将文档置为 pending_approval 状态，记录 previous_lifecycle_status 和 pending_version。
        同时插入 kb_version_changes 记录。
        """
        pass

    @abstractmethod
    def process_approval(self, request: ApprovalRequest) -> ApprovalResult:
        """
        处理人工确认/驳回：
        - APPROVE：状态改为 published，调用 IDeltaIndexer 执行差量索引。
        - REJECT：状态回退到 previous_lifecycle_status，不触发索引。
        - 无论结果，记录 approved_by、approved_at、reject_reason。
        """
        pass

    @abstractmethod
    def list_pending_changes(
        self, 
        source_kb: Optional[str] = None, 
        page: int = 1, 
        limit: int = 20
    ) -> Dict[str, Any]:
        """查询待审批列表（供管理员后台使用）"""
        pass
```

---

### 3. 差量索引器接口 `IDeltaIndexer`

> 核心！在审批通过后执行“增量更新”而非全量重建。需要复用任务2的 `ITripleWriter` 底层写入能力。

```python
class DeltaIndexRequest(BaseModel):
    doc_id: str
    old_version: str
    new_version: str

class DeltaIndexResult(BaseModel):
    success: bool
    affected_chunks: int          # 更新的子块数量
    new_chunks_inserted: int
    old_chunks_disabled: int
    error_msg: Optional[str] = None
    duration_ms: int

class IDeltaIndexer(ABC):
    @abstractmethod
    def execute_delta_index(self, request: DeltaIndexRequest) -> DeltaIndexResult:
        """
        执行三库差量更新：
        1. 解析新文档，生成新的子块（chunk_uuid 重新生成）。
        2. 找出需要保留的子块（内容未变化的，通过 content_hash 比对）。
        3. MySQL：新块插入（is_active=1），旧块标记 is_active=0。
        4. ES：批量更新 is_active 字段，插入新文档。
        5. Milvus：新向量插入，旧向量标记删除（或 partition 过滤）。
        6. 更新 kb_documents 的 version 字段和 indexed_at。
        """
        pass

    @abstractmethod
    def rollback_index(self, doc_id: str) -> None:
        """当索引失败时，回滚到旧版本（恢复 is_active 状态）"""
        pass
```

---

### 4. 通知发送器接口 `INotificationSender`

> 满足“变更时自动提醒”的硬性要求，支持多渠道，并记录发送日志。

```python
class NotificationChannel(str, Enum):
    EMAIL = "email"
    ADMIN_PANEL = "admin_panel"
    WEBHOOK = "webhook"  # 预留

class NotificationRequest(BaseModel):
    doc_id: str
    channel: NotificationChannel
    recipient: str          # 邮箱地址或用户名
    subject: str
    content: str            # 支持 Markdown 或纯文本

class NotificationResult(BaseModel):
    notification_id: str
    status: str  # sent | failed
    error_msg: Optional[str] = None

class INotificationSender(ABC):
    @abstractmethod
    def send(self, request: NotificationRequest) -> NotificationResult:
        """
        发送通知，并将记录写入 kb_notification_logs。
        - 邮件渠道：调用 SMTP 服务。
        - 后台待办：在 admin_pending_changes 列表中增加红点标记（通过缓存）。
        """
        pass

    @abstractmethod
    def notify_pending_approval(self, doc_id: str, change_summary: str) -> None:
        """
        便捷方法：当文档进入 pending_approval 时，自动触发通知。
        收件人从配置中读取（如管理员邮箱列表）。
        """
        pass
```

---

### 5. 全链路追踪聚合器接口 `ITraceAggregator`

> 联调阶段的核心观测工具，根据 trace_id 串联所有日志表。

```python
class TraceDetail(BaseModel):
    trace_id: str
    router_log: Optional[Dict[str, Any]] = None
    query_log: Optional[Dict[str, Any]] = None
    search_log: Optional[Dict[str, Any]] = None
    rank_log: Optional[Dict[str, Any]] = None
    feedback: Optional[Dict[str, Any]] = None
    timeline: List[Dict[str, Any]]  # 按时间排序的事件列表

class ITraceAggregator(ABC):
    @abstractmethod
    def get_full_trace(self, trace_id: str) -> Optional[TraceDetail]:
        """
        根据 trace_id 联合查询 5 张日志表，按 created_at 排序后聚合返回。
        若 trace_id 不存在，返回 None。
        """
        pass

    @abstractmethod
    def get_recent_traces(
        self, 
        limit: int = 100, 
        status: Optional[str] = None  # success | failed
    ) -> List[TraceDetail]:
        """获取最近的 trace 列表，用于联调监控面板"""
        pass
```

---

### 6. 系统健康检查接口 `ISystemHealthChecker`

> 确保服务启动和运行时各依赖组件可达。

```python
class ComponentHealth(BaseModel):
    name: str
    status: str  # ok | degraded | down
    latency_ms: Optional[int] = None
    error_msg: Optional[str] = None

class SystemHealth(BaseModel):
    overall_status: str  # healthy | degraded | unhealthy
    components: List[ComponentHealth]

class ISystemHealthChecker(ABC):
    @abstractmethod
    def check_all(self) -> SystemHealth:
        """
        并发检查所有依赖：
        - MySQL（执行 SELECT 1）
        - Elasticsearch（/_cluster/health）
        - Milvus（/healthz）
        - Redis（PING）
        - Dify API（/info，使用内部 API Key）
        """
        pass

    @abstractmethod
    def check_component(self, name: str) -> ComponentHealth:
        """检查单个组件"""
        pass
```

---

## 三、关键数据流转示意（内部调用链）

```text
[用户上传文档]
    │
    ▼
IVersionDetector.detect_change()
    │
    ├── NEW_UPLOAD → 直接写入三库（复用任务2的 TripleWriter）
    │
    └── VERSION_UPGRADE → IApprovalWorkflow.submit_for_approval()
            │
            ├── 更新 kb_documents.lifecycle_status = 'pending_approval'
            ├── 插入 kb_version_changes
            └── INotificationSender.notify_pending_approval() → 写入 kb_notification_logs

[管理员后台]
    │
    ▼
IApprovalWorkflow.list_pending_changes()  ← 展示待审批列表

[管理员点击确认/驳回]
    │
    ▼
IApprovalWorkflow.process_approval()
    │
    ├── APPROVE → IDeltaIndexer.execute_delta_index()
    │       │
    │       ├── MySQL: 旧块 is_active=0, 新块插入
    │       ├── ES: 批量 upsert
    │       ├── Milvus: 新向量插入, 旧向量逻辑删除
    │       └── 更新 kb_documents.indexing_status=1, version=新版本
    │
    └── REJECT → 回退 lifecycle_status = previous, 不触发索引

[联调观测]
    │
    ▼
ISystemHealthChecker.check_all()  ← 每日/每次发布前运行
ITraceAggregator.get_full_trace()  ← 问题排查时按 trace_id 查询
```

---

## 四、与现有任务2（入库）的复用约定

| 现有接口 | 复用方式 |
| :--- | :--- |
| `ITripleWriter.write_chunks()` | `IDeltaIndexer` 内部调用此方法插入新块（仅针对变更部分）。 |
| `ITripleWriter.soft_delete_chunks()` | `IDeltaIndexer` 内部调用此方法批量标记旧块 `is_active=0`。 |
| `IOrchestrator.get_parsed_result()` | `IVersionDetector` 可调用此方法预解析文档，获取 `elements.json` 用于差量比对。 |

---

## 五、错误处理与事务边界约定

- **审批操作**：`process_approval` 必须在一个**数据库事务**中完成状态更新 + 版本变更记录插入。
- **差量索引**：三库写入采用**最终一致性**策略（先 MySQL → 再 ES → 再 Milvus）。若某一步失败：
  - 记录 `indexing_status = 2`（异常），触发补偿任务重试。
  - 若补偿任务仍失败，调用 `IDeltaIndexer.rollback_index()` 恢复 `is_active` 状态。
- **通知发送**：通知失败不影响审批流程主事务，仅记录错误日志，由定时任务重试（`kb_notification_logs.status = 0`）。

---

## 六、后续开发建议（供新会话参考）

1. **先实现 `IDeltaIndexer`**（最核心，依赖已有 `ITripleWriter`）。
2. **再实现 `IApprovalWorkflow`**（状态机逻辑，需与前端管理员后台联调）。
3. **最后补齐 `INotificationSender` 和 `ITraceAggregator`**（可独立并行开发）。
4. **单元测试重点**：
   - 状态流转覆盖（draft → pending_approval → published/rejected）。
   - 差量索引数据一致性（三库最终状态校验）。
   - 通知重试机制。

---

以上为任务8的完整内部接口契约设计。如需我继续输出**代码骨架（Python类实现）** 或**状态机的完整流转表**，新会话中直接告知即可。