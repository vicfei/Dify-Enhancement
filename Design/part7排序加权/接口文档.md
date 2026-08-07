好的，基于提供的Dify API参考风格和任务7的定位，我为你编写了完整的**任务7（排序加权）接口设计文档**。这份文档遵循了你们项目一贯的规范风格，包含对外API接口、内部模块接口契约、错误码和示例。

---

# 排序加权模块接口设计（任务7）

> **版本**：v1.0  
> **状态**：详细设计定稿  
> **对应数据模型版本**：v6.0（已整合 `kb_rank_logs` 表）

---

## 一、接口定位

### 1.1 模块职责

排序加权模块对召回阶段返回的候选块列表进行**多维度综合重排序**，综合考虑：

- **相关性**（原始检索分归一化）
- **来源权威性**（文档类型/知识库等级）
- **时效性**（文档发布日期衰减）
- **多样性**（同文档块数限制 / MMR）
- **用户角色个性化**（权重微调）
- **场景加成**（如 time_sensitive 临时提高时效权重）

最终输出排序后的有序块列表，供 LLM 生成答案。

### 1.2 在整体流程中的位置

```
用户查询 → 路由 → 意图理解 → 召回（任务6） → 【排序加权（任务7）】 → 父块组装 → LLM生成
```

排序模块接收任务6输出的候选块列表（含 `recall_score`、元数据等），返回排序后的列表。

---

## 二、对外 API 接口

### 2.1 概述

为便于调试、独立测试或未来可能的服务拆分，排序模块提供独立的 RESTful API。

**接口地址**：`/api/v1/rank`

**方法**：`POST`

**Content-Type**：`application/json`

**鉴权**：Bearer Token（与现有 Dify 自研服务统一鉴权）

### 2.2 请求体

| 字段 | 类型 | 必填 | 描述 |
| :--- | :--- | :--- | :--- |
| `candidates` | array[object] | 是 | 待排序的候选块列表（来自召回阶段） |
| `user_context` | object | 是 | 用户上下文，用于个性化权重 |
| `options` | object | 否 | 排序参数覆盖（不传则使用服务端默认配置） |

#### `candidates` 中每个对象的字段

| 字段 | 类型 | 必填 | 描述 |
| :--- | :--- | :--- | :--- |
| `chunk_uuid` | string | 是 | 块唯一标识 |
| `doc_id` | integer | 是 | 文档ID |
| `parent_chunk_id` | integer | 否 | 父块ID（用于去重） |
| `content_text` | string | 是 | 块内容 |
| `source_kb` | string | 是 | 来源知识库 |
| `recall_score` | number | 是 | 原始检索分（来自ES或Milvus，未归一化） |
| `recall_source` | string | 否 | 召回通路（`vector` / `keyword` / `hybrid`） |
| `metadata` | object | 是 | 文档元数据，必须包含 `doc_type`、`published_date`（ISO 8601） |

#### `user_context` 字段

| 字段 | 类型 | 必填 | 描述 |
| :--- | :--- | :--- | :--- |
| `user_id` | string | 是 | 用户标识 |
| `role` | string | 是 | 用户角色（`基金经理`、`风控专员`、`信评分析师`等） |
| `source_kb` | string | 是 | 当前所在知识库 |
| `session_id` | string | 否 | 会话ID（用于日志关联） |
| `time_sensitive` | boolean | 否 | 是否对时效性敏感（来自意图识别） |
| `regulatory_query` | boolean | 否 | 是否为监管类查询 |

#### `options` 字段（覆盖默认配置）

| 字段 | 类型 | 描述 |
| :--- | :--- | :--- |
| `weights` | object | 自定义权重三元组 `{"relevance": 0.5, "authority": 0.3, "timeliness": 0.2}` |
| `authority_coefficients` | object | 自定义来源系数，如 `{"监管公告": 1.3}` |
| `time_decay_lambda` | number | 自定义时效衰减率 |
| `diversity` | object | 多样性策略 `{"mode": "cap_per_doc", "max_chunks_per_doc": 3}` |

### 2.3 响应体

成功时返回 `200`，响应体为：

| 字段 | 类型 | 描述 |
| :--- | :--- | :--- |
| `ranked_chunks` | array[object] | 排序后的块列表（按 `rank_score` 降序） |
| `used_weights` | object | 实际使用的权重三元组 |
| `used_authority_coefficients` | object | 实际使用的来源系数 |
| `used_time_decay_lambda` | number | 实际使用的衰减率 |
| `diversity_mode` | string | 实际使用的多样性策略 |

`ranked_chunks` 中每个对象包含：

| 字段 | 类型 | 描述 |
| :--- | :--- | :--- |
| `chunk_uuid` | string | 块ID |
| `doc_id` | integer | 文档ID |
| `content_text` | string | 块内容 |
| `rank_score` | number | 最终综合得分 |
| `factor_relevance` | number | 归一化后的相关性因子 |
| `factor_authority` | number | 权威性因子 |
| `factor_timeliness` | number | 时效性因子 |
| `rank_position` | integer | 排序位置（从1开始） |
| `metadata` | object | 原始元数据 |

**示例响应**：

```json
{
  "ranked_chunks": [
    {
      "chunk_uuid": "abc-123",
      "doc_id": 456,
      "content_text": "...",
      "rank_score": 0.89,
      "factor_relevance": 0.85,
      "factor_authority": 1.2,
      "factor_timeliness": 0.95,
      "rank_position": 1,
      "metadata": { "doc_type": "内部研报", "published_date": "2026-06-15T00:00:00Z" }
    }
  ],
  "used_weights": {
    "relevance": 0.45,
    "authority": 0.30,
    "timeliness": 0.25
  },
  "used_authority_coefficients": {
    "内部研报": 1.0,
    "监管公告": 1.3
  },
  "used_time_decay_lambda": 0.5,
  "diversity_mode": "cap_per_doc"
}
```

### 2.4 错误码

| HTTP状态 | `code` | 描述 |
| :--- | :--- | :--- |
| 400 | `invalid_param` | 请求参数缺失或格式错误（如 `candidates` 为空、`user_context.role` 缺失） |
| 400 | `unsupported_role` | 用户角色未在配置表中定义 |
| 404 | `not_found` | 相关配置（如权重配置）未找到 |
| 500 | `internal_server_error` | 排序计算过程中发生内部错误 |

**错误响应示例**：

```json
{
  "status": 400,
  "code": "invalid_param",
  "message": "candidates cannot be empty"
}
```

---

## 三、内部模块接口契约（面向开发）

排序模块作为自研核心的一部分，通过依赖注入提供内部接口，供其他模块（如查询处理流程）调用。

### 3.1 核心接口 `IRanker`

```python
from abc import ABC, abstractmethod
from typing import List, Optional
from pydantic import BaseModel

class RankCandidate(BaseModel):
    chunk_uuid: str
    doc_id: int
    parent_chunk_id: Optional[int] = None
    content_text: str
    source_kb: str
    recall_score: float
    recall_source: Optional[str] = None
    metadata: dict  # 必须包含 doc_type, published_date

class UserContext(BaseModel):
    user_id: str
    role: str
    source_kb: str
    session_id: Optional[str] = None
    time_sensitive: bool = False
    regulatory_query: bool = False

class RankOptions(BaseModel):
    weights: Optional[dict] = None          # {"relevance": 0.5, "authority": 0.3, "timeliness": 0.2}
    authority_coefficients: Optional[dict] = None
    time_decay_lambda: Optional[float] = None
    diversity: Optional[dict] = None        # {"mode": "cap_per_doc", "max_chunks_per_doc": 3}

class RankResult(BaseModel):
    ranked_chunks: List[RankCandidate]      # 保持原有结构，附加计算字段
    used_weights: dict
    used_authority_coefficients: dict
    used_time_decay_lambda: float
    diversity_mode: str

class IRanker(ABC):
    @abstractmethod
    def rank(
        self,
        candidates: List[RankCandidate],
        user_context: UserContext,
        options: Optional[RankOptions] = None
    ) -> RankResult:
        """
        对候选块列表进行排序，返回排序结果。
        实现必须：
        1. 从配置中心加载默认权重、系数、衰减率。
        2. 根据用户角色微调权重（从 Nacos ranking_user_weights.yaml 读取）。
        3. 根据 options 覆盖默认配置。
        4. 计算每个候选块的 rank_score。
        5. 应用多样性策略（如 cap_per_doc）。
        6. 记录排序日志到 kb_rank_logs（异步）。
        """
        pass
```

### 3.2 辅助接口（配置加载）

```python
class IRankConfigProvider(ABC):
    @abstractmethod
    def get_default_weights(self) -> dict:
        """返回默认权重三元组"""
        pass

    @abstractmethod
    def get_role_weights(self, role: str) -> dict:
        """根据角色返回个性化权重，若未配置则返回默认"""
        pass

    @abstractmethod
    def get_authority_coefficients(self) -> dict:
        """返回来源权威系数表"""
        pass

    @abstractmethod
    def get_time_decay_lambda(self) -> float:
        """返回默认衰减率"""
        pass

    @abstractmethod
    def get_diversity_config(self) -> dict:
        """返回多样性策略配置"""
        pass
```

### 3.3 依赖注入与使用示例

在查询处理流程中调用：

```python
from app.domain.ranking import IRanker, RankCandidate, UserContext

def process_query(query, user_context, candidates):
    ranker = container.resolve(IRanker)
    result = ranker.rank(
        candidates=candidates,
        user_context=UserContext(**user_context),
        options=None   # 使用默认配置
    )
    # 将 result.ranked_chunks 送入后续父块组装
    return result.ranked_chunks
```

---

## 四、配置数据来源（Nacos）

排序模块依赖以下 Nacos 配置组，支持热更新：

- **`ranking_user_weights.yaml`**：按角色配置权重三元组（已在 v6.0 数据模型中定义）
- **`ranking_weights`**（扩展）：包含归一化方法、衰减参数、多样性策略、场景加成系数（已在 v6.0 中扩展）

排序模块在启动时加载这些配置，并监听变更（通过 Nacos 长轮询或定时刷新）。

---

## 五、日志记录（`kb_rank_logs`）

排序模块在每次排序操作完成后，**异步**将关键信息写入 `kb_rank_logs` 表（已在 v6.0 数据模型中定义），用于后续测评和问题定位。

记录的字段包括：
- `trace_id`、`session_id`
- `user_role`
- `candidate_count`、`ranked_count`
- `used_weights`、`authority_coefficients`、`time_decay_lambda`
- `diversity_mode`、`max_chunks_per_doc`
- `final_top_scores`（Top-5 块的得分快照）
- `total_latency_ms`

日志写入通过消息队列或异步任务完成，不阻塞主流程。

---

## 六、接口版本与兼容性

- 当前版本：`v1`
- 后续如有破坏性变更，将通过版本号（如 `/api/v2/rank`）或新增字段（保持向后兼容）的方式演进。

---

## 七、参考实现（代码骨架）

```python
# app/application/ranking/service.py

class RankingService(IRanker):
    def __init__(self, config_provider: IRankConfigProvider, logger: IRankLogger):
        self.config_provider = config_provider
        self.logger = logger

    def rank(self, candidates, user_context, options=None):
        start_time = time.time()
        # 1. 加载配置
        weights = self._resolve_weights(user_context.role, options)
        authority = self._resolve_authority(options)
        lambda_decay = self._resolve_lambda(options)
        diversity = self._resolve_diversity(options)

        # 2. 计算每个候选块因子
        for c in candidates:
            c.factor_relevance = self._normalize(c.recall_score, candidates)
            c.factor_authority = self._get_authority_factor(c.metadata.get('doc_type'), authority)
            c.factor_timeliness = self._get_time_factor(c.metadata.get('published_date'), lambda_decay)
            c.rank_score = (
                weights['relevance'] * c.factor_relevance +
                weights['authority'] * c.factor_authority +
                weights['timeliness'] * c.factor_timeliness
            )

        # 3. 排序
        ranked = sorted(candidates, key=lambda x: x.rank_score, reverse=True)

        # 4. 多样性裁剪
        ranked = self._apply_diversity(ranked, diversity)

        # 5. 日志（异步）
        self.logger.log_async(
            trace_id=user_context.session_id,
            user_role=user_context.role,
            candidate_count=len(candidates),
            ranked_count=len(ranked),
            used_weights=weights,
            authority_coefficients=authority,
            time_decay_lambda=lambda_decay,
            diversity_mode=diversity.get('mode'),
            final_top_scores=[(c.chunk_uuid, c.rank_score) for c in ranked[:5]],
            latency_ms=(time.time() - start_time) * 1000
        )

        return RankResult(
            ranked_chunks=ranked,
            used_weights=weights,
            used_authority_coefficients=authority,
            used_time_decay_lambda=lambda_decay,
            diversity_mode=diversity.get('mode')
        )
```

---

## 八、与Dify编排的集成方式

在Dify工作流中，通过 **HTTP请求节点** 调用自研服务的 `/api/v1/rank` 接口，或直接通过自研服务内部的 `IRanker` 接口调用（推荐后者以减少网络开销）。

若采用API方式，Dify的知识检索节点输出候选列表后，经HTTP节点调用排序服务，再将结果传给LLM节点。

---

## 九、附录：权重与系数默认值（示例）

| 配置项 | 默认值 |
| :--- | :--- |
| `w_relevance` | 0.5 |
| `w_authority` | 0.3 |
| `w_timeliness` | 0.2 |
| 监管公告权威系数 | 1.3 |
| 内部研报权威系数 | 1.0 |
| 外部资讯权威系数 | 0.8 |
| 时效衰减率 `λ` | 0.5 |
| 近7天无衰减 | `no_decay_days: 7` |
| 多样性策略 | `cap_per_doc`, `max_chunks_per_doc: 3` |

---

以上即为任务7的完整接口设计文档。如需进一步补充**API的OpenAPI规范（YAML/JSON）**或**单元测试用例**，请告知。