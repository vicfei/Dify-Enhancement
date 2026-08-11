收到，现在为你输出**任务7（排序加权）完整的内部接口设计文档**。这份文档定位为**供开发团队实现的契约**，聚焦于类职责划分、接口契约和核心数据流，不包含对外REST API（已在之前单独交付）。


# 任务7：排序加权 —— 内部接口设计文档

> **版本**：v1.0  
> **状态**：详细设计定稿  
> **对应数据模型**：v6.1（`kb_rank_logs`、`kb_sessions.user_role`、`ranking_user_weights.yaml`）  
> **上游依赖**：任务6（召回策略）—— 提供 `candidates` 候选块列表  
> **下游输出**：排序后的 `ranked_chunks` → 父块组装 → LLM 生成


## 一、模块定位与核心职责

任务7是自研核心中的**业务排序层（Business Re-rank Layer）**，它不对语义相关性做任何计算（那是任务6的职责），而是专注于**业务效用（Utility）** 的评估与加权。

| 职责 | 描述 |
| :--- | :--- |
| **相关性因子归一化** | 将任务6传入的 `recall_score`（可能来自RRF或Rerank）归一化到统一量纲 |
| **权威性因子计算** | 根据文档类型/来源知识库，查表赋予权威系数（监管公告 > 内部研报 > 外部资讯） |
| **时效性因子计算** | 根据 `published_date`，使用指数衰减模型计算时效性得分（近7天满分，越旧衰减越快） |
| **个性化权重合成** | 根据用户角色（`user_role`）从 Nacos 拉取对应的 `(w_rel, w_auth, w_time)` 三元组 |
| **场景加成** | 根据意图识别传递的信号（如 `time_sensitive=true`）动态调整权重乘数 |
| **多样性压制** | 对同一文档的多个块进行截断（`cap_per_doc`），保证 LLM 输入的信息广度 |
| **可观测性** | 异步写入 `kb_rank_logs`，记录权重快照和 Top-5 得分，支撑后续排序调优 |


## 二、枚举与实体定义（共用数据契约）

```python
from enum import Enum
from typing import Optional, List, Dict, Any
from pydantic import BaseModel, Field
from datetime import datetime

# ========== 枚举定义 ==========

class DiversityMode(str, Enum):
    NONE = "none"
    CAP_PER_DOC = "cap_per_doc"
    MMR = "mmr"

class NormalizationMethod(str, Enum):
    MIN_MAX = "min_max"
    Z_SCORE = "z_score"
    SIGMOID = "sigmoid"

# ========== 输入实体 ==========

class RankCandidate(BaseModel):
    """来自任务6的候选块（含召回分数和元数据）"""
    chunk_uuid: str
    doc_id: int
    parent_chunk_id: Optional[int] = None
    content_text: str
    source_kb: str
    recall_score: float                  # 原始语义分（RRF 或 Rerank 输出）
    recall_source: Optional[str] = None  # "vector" / "keyword" / "hybrid" / "rerank"
    metadata: Dict[str, Any] = Field(default_factory=dict)
    # metadata 中必须包含:
    #   - doc_type: str          (如 "内部研报", "监管公告", "外部资讯")
    #   - published_date: str    (ISO 8601 格式, 如 "2026-06-15T00:00:00Z")
    #   - tags: list[str]        (可选)
    #   - entity_codes: list[str](可选)

class UserContext(BaseModel):
    """用户上下文，用于个性化权重"""
    user_id: str
    role: str                    # 基金经理 / 风控专员 / 信评分析师 / 量化研究员 / 合规审核 / 运营人员 / 交易员 / 产品经理
    source_kb: str               # 当前所在知识库
    session_id: Optional[str] = None
    time_sensitive: bool = False # 来自任务5的意图识别结果
    regulatory_query: bool = False
    deep_analysis: bool = False

class RankOptions(BaseModel):
    """可选运行时覆盖参数（用于调试或A/B测试）"""
    weights: Optional[Dict[str, float]] = None           # {"relevance": 0.5, "authority": 0.3, "timeliness": 0.2}
    authority_coefficients: Optional[Dict[str, float]] = None
    time_decay_lambda: Optional[float] = None
    diversity: Optional[Dict[str, Any]] = None           # {"mode": "cap_per_doc", "max_chunks_per_doc": 3}
    normalization: Optional[NormalizationMethod] = None

# ========== 内部计算中间体 ==========

class FactorScores(BaseModel):
    """单个候选块的三因子得分（计算过程中的中间产物）"""
    relevance_raw: float        # 归一化前的 recall_score
    relevance_norm: float       # 归一化后的相关性因子 (0~1)
    authority: float            # 权威性因子 (通常 0.7~1.3)
    timeliness: float           # 时效性因子 (0~1)
    rank_score: float           # 最终综合得分

# ========== 输出实体 ==========

class RankedChunk(RankCandidate):
    """排序后的候选块（附加计算字段）"""
    factor_relevance: float
    factor_authority: float
    factor_timeliness: float
    rank_score: float
    rank_position: int

class RankResult(BaseModel):
    """排序结果"""
    ranked_chunks: List[RankedChunk]
    used_weights: Dict[str, float]
    used_authority_coefficients: Dict[str, float]
    used_time_decay_lambda: float
    diversity_mode: str
    total_latency_ms: float
```


## 三、核心接口 —— `IRanker`

```python
from abc import ABC, abstractmethod

class IRanker(ABC):
    """排序加权核心接口"""

    @abstractmethod
    def rank(
        self,
        candidates: List[RankCandidate],
        user_context: UserContext,
        options: Optional[RankOptions] = None
    ) -> RankResult:
        """
        对候选块列表进行多维度综合排序。

        执行流程（严格顺序）：
        1. 加载配置：从 IRankConfigProvider 获取默认权重、权威系数、衰减率。
        2. 个性化权重：根据 user_context.role 覆盖权重（从 ranking_user_weights.yaml 读取）。
        3. 覆盖选项：若 options 不为空，以 options 中的值覆盖上述配置。
        4. 因子计算：
           a. 相关性：对 candidates 中的 recall_score 做归一化（归一化方法由配置决定）。
           b. 权威性：根据 metadata.doc_type 查表。
           c. 时效性：根据 published_date 计算指数衰减值。
        5. 综合得分：rank_score = w_rel * rel_norm + w_auth * authority + w_time * timeliness。
        6. 场景加成：若 user_context.time_sensitive == True，将 w_time 临时 ×1.3。
        7. 排序：按 rank_score 降序排列。
        8. 多样性过滤：根据配置的 diversity 策略（如 cap_per_doc）对结果进行裁剪。
        9. 日志：异步将关键信息写入 kb_rank_logs（通过 IRankLogger）。

        Args:
            candidates: 任务6输出的候选块列表
            user_context: 用户上下文（含角色、敏感标记等）
            options: 运行时参数覆盖（可选）

        Returns:
            RankResult: 排序后的列表 + 使用的配置快照
        """
        pass
```


## 四、辅助接口（依赖注入）

### 4.1 配置提供者 —— `IRankConfigProvider`

```python
class IRankConfigProvider(ABC):
    """从 Nacos 读取排序相关配置"""

    @abstractmethod
    def get_default_weights(self) -> Dict[str, float]:
        """返回默认权重三元组（来自 ranking_weights）"""
        pass

    @abstractmethod
    def get_role_weights(self, role: str) -> Dict[str, float]:
        """
        根据角色返回个性化权重（来自 ranking_user_weights.yaml）。
        若角色未配置，返回 default_weights。
        """
        pass

    @abstractmethod
    def get_authority_coefficients(self) -> Dict[str, float]:
        """返回来源权威系数表（来自 ranking_weights.source_priority）"""
        pass

    @abstractmethod
    def get_time_decay_lambda(self) -> float:
        """返回默认时效衰减率"""
        pass

    @abstractmethod
    def get_time_decay_no_decay_days(self) -> int:
        """返回无衰减保护天数（如近7天不衰减）"""
        pass

    @abstractmethod
    def get_diversity_config(self) -> Dict[str, Any]:
        """返回多样性策略配置"""
        pass

    @abstractmethod
    def get_normalization_method(self) -> NormalizationMethod:
        """返回归一化方法"""
        pass

    @abstractmethod
    def get_scenario_boosts(self) -> Dict[str, Dict[str, float]]:
        """返回场景加成系数表（如 time_sensitive -> timeliness_multiplier: 1.3）"""
        pass
```

### 4.2 归一化器 —— `INormalizer`

```python
class INormalizer(ABC):
    """将原始 recall_score 归一化到统一量纲"""

    @abstractmethod
    def normalize(
        self,
        scores: List[float],
        method: NormalizationMethod
    ) -> List[float]:
        """
        对一组分数进行归一化。

        实现细节：
        - MIN_MAX: (x - min) / (max - min)，处理除零情况（全部相等则返回 0.5）。
        - Z_SCORE: (x - mean) / std，截断到 [-3, 3] 后映射到 [0, 1]。
        - SIGMOID: 1 / (1 + exp(-(x - mean) / std))。
        """
        pass
```

### 4.3 多样性过滤器 —— `IDiversityFilter`

```python
class IDiversityFilter(ABC):
    """对排序后的列表进行多样性压制，防止同一文档霸屏"""

    @abstractmethod
    def apply(
        self,
        ranked: List[RankedChunk],
        config: Dict[str, Any]
    ) -> List[RankedChunk]:
        """
        应用多样性策略。

        支持的策略：
        - "none": 原样返回
        - "cap_per_doc": 限制同一 doc_id 的最大入选块数（config.max_chunks_per_doc）
        - "mmr": 最大边际相关性（在相关性得分基础上增加多样性惩罚）
        """
        pass
```

### 4.4 排序日志记录器 —— `IRankLogger`

```python
class IRankLogger(ABC):
    """异步写入 kb_rank_logs（通过消息队列或后台线程）"""

    @abstractmethod
    def log_async(self, log_data: Dict[str, Any]) -> None:
        """
        异步记录排序日志，不阻塞主流程。

        log_data 应包含：
        - trace_id, session_id, user_role
        - candidate_count, ranked_count
        - used_weights, authority_coefficients, time_decay_lambda
        - diversity_mode, max_chunks_per_doc
        - final_top_scores: list of (chunk_uuid, rank_score) for top 5
        - total_latency_ms
        """
        pass
```


## 五、核心实现流程（伪代码 / 时序）

```python
class RankingService(IRanker):
    def __init__(
        self,
        config_provider: IRankConfigProvider,
        normalizer: INormalizer,
        diversity_filter: IDiversityFilter,
        logger: IRankLogger
    ):
        self.config = config_provider
        self.normalizer = normalizer
        self.diversity_filter = diversity_filter
        self.logger = logger

    def rank(self, candidates, user_context, options=None):
        start_time = time.time()

        # 1. 加载配置
        weights = self._resolve_weights(user_context, options)
        authority = self._resolve_authority(options)
        lambda_decay = self._resolve_lambda(options)
        diversity_config = self._resolve_diversity(options)
        norm_method = self._resolve_norm_method(options)

        # 2. 归一化 recall_score
        raw_scores = [c.recall_score for c in candidates]
        norm_scores = self.normalizer.normalize(raw_scores, norm_method)

        # 3. 逐块计算因子 + 综合得分
        ranked_candidates = []
        for idx, c in enumerate(candidates):
            # 3a. 权威性因子
            doc_type = c.metadata.get("doc_type", "内部研报")  # 默认内部研报
            auth_factor = authority.get(doc_type, 1.0)

            # 3b. 时效性因子
            pub_date = c.metadata.get("published_date")
            time_factor = self._calc_time_factor(pub_date, lambda_decay)

            # 3c. 综合得分
            rank_score = (
                weights["relevance"] * norm_scores[idx] +
                weights["authority"] * auth_factor +
                weights["timeliness"] * time_factor
            )

            # 3d. 场景加成（直接在权重上应用乘数）
            rank_score = self._apply_scenario_boosts(rank_score, user_context, weights, norm_scores[idx], auth_factor, time_factor)

            ranked_candidates.append(
                RankedChunk(
                    **c.dict(),
                    factor_relevance=norm_scores[idx],
                    factor_authority=auth_factor,
                    factor_timeliness=time_factor,
                    rank_score=rank_score,
                    rank_position=0  # 稍后填充
                )
            )

        # 4. 排序
        ranked_candidates.sort(key=lambda x: x.rank_score, reverse=True)

        # 5. 填充位置
        for pos, item in enumerate(ranked_candidates, start=1):
            item.rank_position = pos

        # 6. 多样性过滤
        filtered = self.diversity_filter.apply(ranked_candidates, diversity_config)

        # 7. 异步日志
        self.logger.log_async({
            "trace_id": user_context.session_id,
            "session_id": user_context.session_id,
            "user_role": user_context.role,
            "candidate_count": len(candidates),
            "ranked_count": len(filtered),
            "used_weights": weights,
            "authority_coefficients": authority,
            "time_decay_lambda": lambda_decay,
            "diversity_mode": diversity_config.get("mode"),
            "max_chunks_per_doc": diversity_config.get("max_chunks_per_doc"),
            "final_top_scores": [(c.chunk_uuid, c.rank_score) for c in filtered[:5]],
            "total_latency_ms": (time.time() - start_time) * 1000
        })

        return RankResult(
            ranked_chunks=filtered,
            used_weights=weights,
            used_authority_coefficients=authority,
            used_time_decay_lambda=lambda_decay,
            diversity_mode=diversity_config.get("mode", "none"),
            total_latency_ms=(time.time() - start_time) * 1000
        )
```


## 六、与 Dify 编排的集成方式（调用路径）

```text
Dify 工作流
    │
    ├── 知识检索节点（任务6）
    │   └── 输出 candidates（含 recall_score）
    │
    ├── HTTP 请求节点 或 自研核心方法
    │   └── 调用 RankingService.rank()
    │       ├── 传入 candidates
    │       ├── 传入 user_context（从会话中提取 role）
    │       └── 输出 ranked_chunks
    │
    └── LLM 节点
        └── 将 ranked_chunks（经父块组装后）作为上下文
```

> **关键说明**：由于 Rerank 完全封装在任务6内部，任务7对 `recall_score` 的来源（RRF 或 Rerank）完全透明，因此**排序服务的代码、接口、配置均不需要为 Rerank 做特殊适配**。这体现了我们架构中“语义层”与“业务层”的完美解耦。


## 七、错误处理与异常码

| 异常场景 | 错误码 | 处理策略 |
| :--- | :--- | :--- |
| `candidates` 为空列表 | `EMPTY_CANDIDATES` | 直接返回空结果，不记录日志 |
| `user_context.role` 未配置 | `UNSUPPORTED_ROLE` | 降级使用 `default_weights`，记录 WARNING 日志 |
| 归一化时全部分数相等 | `NORMALIZATION_NO_VARIANCE` | 所有 `relevance_norm = 0.5`，继续执行 |
| `published_date` 缺失或格式错误 | `MISSING_PUBLISHED_DATE` | 将该块的时效性因子设为 `1.0`（不惩罚），记录 WARNING |
| 配置中心（Nacos）不可用 | `CONFIG_LOAD_FAILED` | **熔断兜底**：使用代码内置的硬编码默认值，排序服务继续可用 |
| 日志写入失败 | `LOG_WRITE_FAILED` | **静默失败**：仅打印 ERROR 日志，不影响主流程 |


## 八、开发人员注意事项

1. **性能预算**：排序模块是纯内存计算，**必须保证 < 50ms**（不含日志异步写入）。避免在循环内做任何IO操作（如查数据库）。
2. **配置热更新**：`IRankConfigProvider` 应监听 Nacos 变更，但**权重切换不需要重启服务**，每次 `rank()` 调用时实时读取。
3. **单元测试必测场景**：
   - 只有 1 个候选块（边界）。
   - 所有候选块来自同一文档（多样性压制）。
   - 用户角色为“合规审核”（权威性权重最高）。
   - `time_sensitive=true` 时，近7天文档明显排更前。
4. **日志采样**：生产环境可配置 `log_sample_rate=0.1`，避免 `kb_rank_logs` 表写入压力过大。


## 九、后续扩展点（预留）

| 扩展点 | 说明 |
| :--- | :--- |
| **Learning to Rank (LTR)** | 后续若引入点击反馈模型，可在 `IRanker` 中增加 `apply_ltr_boost()` 方法，叠加 LTR 分数 |
| **业务规则引擎** | 若排序逻辑变得极其复杂（如“必须置顶3天内监管公告”），可引入 Drools 等规则引擎，但当前简单的加权模型已满足需求 |
| **实时反馈微调** | 若用户点击了排序靠后的块，可将该信号回传至 `kb_user_feedback.rank_helpful`，用于后续离线调参 |

---

以上即为任务7完整的内部接口设计文档。若需要我基于此设计生成**单元测试代码骨架**或**Nacos配置文件的完整示例**，请随时告知。