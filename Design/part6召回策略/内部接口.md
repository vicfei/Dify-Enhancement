好的，基于我们确认的 **“Rerank 模型作为任务6内部可选步骤”** 的架构决策，以及已修正的对外接口文档和数据模型，现在为您补充**任务6的内部接口设计**。

内部接口设计定义了任务6内部各组件之间的契约，用于指导团队并行开发。


# 任务6 召回策略 — 内部接口设计

> **版本**：v1.1  
> **状态**：已定稿  
> **适用对象**：后端开发人员  
> **变更说明**：v1.1 新增 `IRerankClient` 接口，将 Rerank 模型集成纳入任务6内部。


## 一、模块划分

任务6内部按职责划分为 **5 个核心模块**：

```
┌─────────────────────────────────────────────────────────────────────┐
│                      任务6 内部模块架构                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    ISearchOrchestrator                      │   │
│  │                    （召回编排器）                           │   │
│  │                  - 流程编排                                 │   │
│  │                  - 并行调度                                 │   │
│  │                  - 配置驱动                                 │   │
│  └───────────────────────────┬─────────────────────────────────┘   │
│                              │                                       │
│          ┌───────────────────┼───────────────────┐                  │
│          ▼                   ▼                   ▼                  │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐         │
│  │ IHybridSearcher│  │  ITagFilter   │  │ IEntityMapper │         │
│  │ （多路召回器） │  │ （标签筛选器） │  │ （实体映射器） │         │
│  └───────┬───────┘  └───────────────┘  └───────────────┘         │
│          │                                                          │
│          ▼                                                          │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐         │
│  │  IRerankClient │  │  IRRFEngine   │  │ ISearchLogger │         │
│  │ （Rerank客户端）│  │ （RRF融合引擎）│  │ （日志记录器） │         │
│  │   📌 v5.1新增 │  └───────────────┘  └───────────────┘         │
│  └───────────────┘                                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```


## 二、数据契约（DTO 定义）

### 2.1 `SearchRequest`（检索请求）

```python
from dataclasses import dataclass, field
from typing import Optional, List, Dict, Any

@dataclass
class SearchRequest:
    """检索请求（内部使用）"""
    # 追踪信息
    request_id: str
    session_id: Optional[str] = None
    
    # 查询信息
    query: str
    query_embedding: List[float]  # 1536 维
    rewritten_query: Optional[str] = None
    sub_questions: Optional[List[str]] = None
    intent: Optional[Dict[str, Any]] = None
    
    # 过滤条件
    source_kb: Optional[str] = None
    tag_filters: Optional[List[str]] = None
    tag_match_mode: str = "exact"  # exact / any / all
    entity_codes: Optional[List[str]] = None
    entity_expand: bool = False
    
    # 召回参数
    top_k: int = 20
    semantic_top_k: int = 50
    keyword_top_k: int = 50
    
    # 运行时配置（由编排器注入）
    rerank_config: Optional[Dict[str, Any]] = None
```

### 2.2 `SearchResult`（单条召回结果）

```python
@dataclass
class SearchResult:
    """单条召回结果"""
    # 标识信息
    chunk_uuid: str
    doc_id: int
    parent_chunk_id: Optional[str] = None
    
    # 内容
    content_text: str
    source_kb: str
    tags: List[str] = field(default_factory=list)
    
    # 得分（核心字段）
    recall_score: float = 0.0      # 统一语义分：rrf_score 或 rerank_score
    rrf_score: Optional[float] = None      # RRF 融合分（未启用Rerank时与recall_score相同）
    rerank_score: Optional[float] = None   # Rerank 精排分（启用时使用）
    
    # 各通路原始得分（调试用）
    semantic_score: Optional[float] = None
    keyword_score: Optional[float] = None
    fuzzy_score: Optional[float] = None
    
    # 来源标识
    found_in: List[str] = field(default_factory=list)  # semantic / keyword / fuzzy
    
    # 元数据
    metadata: Dict[str, Any] = field(default_factory=dict)
```

### 2.3 `SearchResponse`（检索响应）

```python
@dataclass
class SearchResponse:
    """检索响应"""
    request_id: str
    results: List[SearchResult]
    total_candidates: int
    fusion_stats: Dict[str, Any]
    rerank_info: Dict[str, Any]  # {enabled, model, input_count, latency_ms}
    latency_ms: int
    fallback_triggered: bool
```

### 2.4 `RerankRequest` / `RerankResult`

```python
@dataclass
class RerankRequest:
    """Rerank 请求"""
    query: str
    documents: List[str]  # 候选块文本列表
    top_n: int = 50
    model: Optional[str] = None

@dataclass
class RerankResult:
    """Rerank 结果"""
    scores: List[float]  # 与输入顺序对应的精排分（0~1）
    model: str
    latency_ms: int
```


## 三、核心接口定义（Python ABC）

### 3.1 `ISearchOrchestrator` — 召回编排器（入口）

```python
from abc import ABC, abstractmethod
from typing import List, Optional

class ISearchOrchestrator(ABC):
    """召回编排器——任务6的主入口"""
    
    @abstractmethod
    def execute(self, request: SearchRequest) -> SearchResponse:
        """
        执行完整召回流程
        
        流程编排：
        1. 标签/实体前置校验与过滤条件构建
        2. 并行多路召回（semantic + keyword + fuzzy）
        3. 结果合并去重
        4. RRF 粗筛（截断至 max_candidates_to_rerank）
        5. 可选：Rerank 精排（由 Nacos 开关控制）
        6. 标签后置过滤
        7. 记录日志并返回
        
        Args:
            request: 检索请求
            
        Returns:
            SearchResponse: 检索响应
            
        Raises:
            SearchServiceException: 检索服务异常
        """
        pass
    
    @abstractmethod
    def execute_batch(self, requests: List[SearchRequest]) -> List[SearchResponse]:
        """批量执行（用于子问题并行检索）"""
        pass
```

### 3.2 `IHybridSearcher` — 多路召回器

```python
class IHybridSearcher(ABC):
    """多路召回器——执行并行召回"""
    
    @abstractmethod
    def multi_recall(self, request: SearchRequest) -> List[SearchResult]:
        """
        执行多路并行召回
        
        Returns:
            List[SearchResult]: 去重后的候选列表（未融合排序）
        """
        pass
    
    @abstractmethod
    def semantic_search(self, query_embedding: List[float], 
                        filters: Dict[str, Any], 
                        top_k: int) -> List[SearchResult]:
        """纯语义检索（Milvus）"""
        pass
    
    @abstractmethod
    def keyword_search(self, query: str, 
                       filters: Dict[str, Any], 
                       top_k: int) -> List[SearchResult]:
        """关键词检索（ES BM25）"""
        pass
    
    @abstractmethod
    def fuzzy_search(self, query: str, 
                     filters: Dict[str, Any], 
                     top_k: int) -> List[SearchResult]:
        """模糊检索（ES fuzziness）"""
        pass
```

### 3.3 `IRerankClient` — Rerank 客户端（📌 v5.1 新增）

```python
class IRerankClient(ABC):
    """Rerank 模型客户端（可插拔）"""
    
    @abstractmethod
    def rerank(self, request: RerankRequest) -> RerankResult:
        """
        调用 Rerank 模型对候选块进行精排
        
        执行步骤：
        1. 检查配置开关
        2. 若启用：调用外部 Rerank 服务（Cohere/BGE/自定义）
        3. 若禁用/失败：返回空结果，由编排器降级为 RRF
        
        Args:
            request: Rerank 请求（含 query 和候选文本列表）
            
        Returns:
            RerankResult: 精排分数列表
            
        Raises:
            RerankServiceUnavailableError: 服务不可用（触发降级）
            RerankTimeoutError: 超时（触发降级）
        """
        pass
    
    @abstractmethod
    def is_available(self) -> bool:
        """检查 Rerank 服务是否可用（熔断器状态）"""
        pass
    
    @abstractmethod
    def get_model_info(self) -> Dict[str, str]:
        """返回当前使用的模型信息（用于日志）"""
        pass
```

### 3.4 `IRRFEngine` — RRF 融合引擎

```python
class IRRFEngine(ABC):
    """RRF 融合引擎"""
    
    @abstractmethod
    def fuse(self, 
             semantic_results: List[SearchResult],
             keyword_results: List[SearchResult],
             fuzzy_results: Optional[List[SearchResult]] = None,
             k: int = 60,
             weights: Optional[Dict[str, float]] = None) -> List[SearchResult]:
        """
        执行 RRF 融合
        
        Args:
            semantic_results: 语义检索结果（已按相似度降序排列）
            keyword_results: 关键词检索结果（已按 BM25 降序排列）
            fuzzy_results: 模糊检索结果（可选）
            k: RRF 常数（默认 60）
            weights: 各通路权重 {semantic: 0.6, keyword: 0.3, fuzzy: 0.1}
            
        Returns:
            List[SearchResult]: RRF 融合后按分数降序排列的列表
        """
        pass
    
    @abstractmethod
    def truncate(self, results: List[SearchResult], top_n: int) -> List[SearchResult]:
        """截断候选列表至指定数量"""
        pass
```

### 3.5 `ITagFilter` — 标签筛选器

```python
class ITagFilter(ABC):
    """标签筛选器"""
    
    @abstractmethod
    def validate_tags(self, tags: List[str]) -> bool:
        """验证标签是否存在于 kb_tags 表中"""
        pass
    
    @abstractmethod
    def build_es_filter(self, tags: List[str], mode: str = "exact") -> Dict:
        """构建 ES 过滤条件"""
        pass
    
    @abstractmethod
    def build_milvus_filter(self, tags: List[str], mode: str = "exact") -> str:
        """构建 Milvus 标量过滤表达式"""
        pass
    
    @abstractmethod
    def apply_boost(self, results: List[SearchResult], 
                    tags: List[str]) -> List[SearchResult]:
        """标签后置加权（匹配标签的块 × 1.2）"""
        pass
```

### 3.6 `IEntityMapper` — 投研实体映射器

```python
@dataclass
class EntityInfo:
    entity_code: str
    entity_type: str  # stock / bond / fund
    entity_name: str
    aliases: List[str]

class IEntityMapper(ABC):
    """投研实体映射器"""
    
    @abstractmethod
    def extract_entities(self, query: str) -> List[EntityInfo]:
        """从查询中提取投研实体"""
        pass
    
    @abstractmethod
    def expand_entities(self, entity_codes: List[str]) -> List[str]:
        """展开关联实体（股票→债券）"""
        pass
    
    @abstractmethod
    def build_filter(self, entity_codes: List[str]) -> Dict:
        """构建 ES/Milvus 过滤条件"""
        pass
```

### 3.7 `ISearchLogger` — 日志记录器

```python
class ISearchLogger(ABC):
    """召回日志记录器"""
    
    @abstractmethod
    def log(self, request: SearchRequest, 
            results: List[SearchResult],
            stats: Dict[str, Any],
            rerank_info: Dict[str, Any]) -> None:
        """
        写入 kb_search_logs 表
        
        包含：
        - 请求参数（query, filters, top_k 等）
        - 各通路命中数（semantic_hits, keyword_hits, fuzzy_hits）
        - Rerank 执行信息（enabled, model, latency_ms）
        - RRF 参数快照
        - 最终结果数量
        """
        pass
```


## 四、Rerank 客户端实现策略

### 4.1 接口适配层设计

为支持多供应商切换，`IRerankClient` 的实现类应采用**策略模式**：

```python
class CohereRerankClient(IRerankClient):
    """Cohere Rerank 实现"""
    def rerank(self, request: RerankRequest) -> RerankResult:
        # 调用 Cohere API
        pass

class BGERRankClient(IRerankClient):
    """BAAI BGE Reranker 实现（本地部署）"""
    def rerank(self, request: RerankRequest) -> RerankResult:
        # 调用本地部署的 BGE 模型
        pass

class CompositeRerankClient(IRerankClient):
    """组合客户端，支持故障转移"""
    def rerank(self, request: RerankRequest) -> RerankResult:
        # 主备切换逻辑
        pass
```

### 4.2 工厂模式创建客户端

```python
class RerankClientFactory:
    @staticmethod
    def create(config: Dict[str, Any]) -> IRerankClient:
        provider = config.get("model_provider", "cohere")
        if provider == "cohere":
            return CohereRerankClient(config)
        elif provider == "jina":
            return JinaRerankClient(config)
        elif provider == "custom":
            return CustomRerankClient(config)
        else:
            raise ValueError(f"Unsupported rerank provider: {provider}")
```


## 五、接口依赖关系图

```
┌─────────────────────────────────────────────────────────────────────┐
│                     ISearchOrchestrator                            │
│                          （编排器）                                 │
└───────┬─────────────────┬─────────────────┬─────────────────────────┘
        │                 │                 │
        ▼                 ▼                 ▼
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│ IHybridSearcher│ │  IRRFEngine   │ │ IRerankClient │
│  (多路召回)   │ │  (RRF融合)    │ │  (Rerank)     │
└───────┬───────┘ └───────────────┘ └───────┬───────┘
        │                                     │
        ▼                                     ▼
┌───────────────┐                     ┌───────────────┐
│  MilvusClient │                     │  Cohere API   │
│  ESClient     │                     │  BGE API      │
└───────────────┘                     └───────────────┘
```


## 六、使用示例（编排器实现片段）

```python
class SearchOrchestrator(ISearchOrchestrator):
    def __init__(self,
                 searcher: IHybridSearcher,
                 rrf_engine: IRRFEngine,
                 rerank_client: Optional[IRerankClient],
                 tag_filter: ITagFilter,
                 entity_mapper: IEntityMapper,
                 logger: ISearchLogger,
                 config: Dict[str, Any]):
        self.searcher = searcher
        self.rrf_engine = rrf_engine
        self.rerank_client = rerank_client
        self.tag_filter = tag_filter
        self.entity_mapper = entity_mapper
        self.logger = logger
        self.config = config
    
    def execute(self, request: SearchRequest) -> SearchResponse:
        start_time = time.time()
        rerank_info = {"enabled": False}
        
        # 1. 多路召回
        candidates = self.searcher.multi_recall(request)
        
        # 2. RRF 粗筛
        rerank_cfg = self.config.get("rerank", {})
        max_input = rerank_cfg.get("max_candidates_to_rerank", 50)
        candidates = self.rrf_engine.truncate(candidates, max_input)
        
        # 3. 可选 Rerank
        if self._should_apply_rerank(rerank_cfg, candidates):
            rerank_result = self.rerank_client.rerank(
                RerankRequest(
                    query=request.query,
                    documents=[r.content_text for r in candidates],
                    top_n=max_input
                )
            )
            # 替换 recall_score
            for i, score in enumerate(rerank_result.scores):
                candidates[i].rerank_score = score
                candidates[i].recall_score = score
            rerank_info = {
                "enabled": True,
                "model": rerank_result.model,
                "input_count": len(candidates),
                "latency_ms": rerank_result.latency_ms
            }
        else:
            # 未启用 Rerank：使用 RRF 融合
            candidates = self.rrf_engine.fuse(candidates, ...)
            for r in candidates:
                r.recall_score = r.rrf_score
        
        # 4. 标签/实体过滤
        candidates = self._apply_filters(candidates, request)
        
        # 5. 截断至 top_k
        final_results = candidates[:request.top_k]
        
        # 6. 日志
        self.logger.log(...)
        
        return SearchResponse(
            request_id=request.request_id,
            results=final_results,
            rerank_info=rerank_info,
            latency_ms=int((time.time() - start_time) * 1000),
            ...
        )
    
    def _should_apply_rerank(self, cfg: Dict, candidates: List) -> bool:
        if not cfg.get("enabled", False):
            return False
        min_trigger = cfg.get("min_candidates_to_trigger", 10)
        return len(candidates) >= min_trigger
```


## 七、版本历史

| 版本 | 日期 | 变更说明 |
| :--- | :--- | :--- |
| v1.0 | 2026-08-05 | 初始版本（IHybridSearcher、ITagFilter、IEntityMapper、IRRFEngine、ISearchLogger、ISearchOrchestrator） |
| **v1.1** | **2026-08-07** | **新增 `IRerankClient` 接口，完善 `SearchResult` 数据契约（`rerank_score`、`recall_score` 语义）** |


## 八、开发分工建议

| 接口 | 负责模块 | 依赖外部服务 | 优先级 |
| :--- | :--- | :--- | :--- |
| `IHybridSearcher` | 召回模块 | ES Client、Milvus Client | P0 |
| `IRRFEngine` | 融合模块 | 无 | P0 |
| `ITagFilter` | 过滤模块 | MySQL `kb_tags` | P0 |
| `IEntityMapper` | 实体模块 | MySQL `kb_entity_mapping` | P0 |
| `ISearchLogger` | 日志模块 | MySQL `kb_search_logs` | P0 |
| `ISearchOrchestrator` | 编排模块 | 上述所有模块 | P0 |
| **`IRerankClient`** | **Rerank 模块** | **Cohere/BGE API（可选）** | **P1（可并行开发）** |


**总结**：内部接口设计将 Rerank 模型作为 `IRerankClient` 注入到编排器中，与 `IHybridSearcher`、`IRRFEngine` 平级。这使得 Rerank 成为一个**可插拔的高级特性**，不影响其他模块的开发进度。MVP 阶段可先实现 `IRerankClient` 的空实现（直接返回空列表），待系统稳定后再接入真实模型。