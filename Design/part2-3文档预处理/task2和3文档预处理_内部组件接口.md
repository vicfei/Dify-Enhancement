好的。既然你要求“深入详细拆解内部组件接口”，我们就不再停留在流程层面，而是直接深入到**代码级别的接口契约**。

在自研预处理服务中，我按照**领域驱动设计（DDD）**的原则，将内部划分为 **5 个核心组件**，它们通过**严格的接口契约（Python Protocol / ABC）**进行协作。这样做的好处是：未来如果要替换切块算法或更换向量库，只需实现新的类，而不影响整个管道。

---

### 一、整体组件关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                    IngestionOrchestrator (调度器)               │
│  (状态机: 版本判定 → 解析 → 切块 → 写入 → 状态更新)            │
└───────────────┬─────────────────┬─────────────────┬─────────────┘
                │                 │                 │
                ▼                 ▼                 ▼
┌───────────────────┐ ┌───────────────────┐ ┌─────────────────────┐
│  VersionResolver  │ │  ParserAdapter    │ │  ChunkerEngine      │
│  (版本/去重)       │ │  (MinerU适配器)   │ │  (父子块逻辑)        │
└───────────────────┘ └───────────────────┘ └─────────────────────┘
                                                    │
                                                    ▼
                                          ┌─────────────────────┐
                                          │  TripleWriter       │
                                          │  (MySQL/ES/Milvus)  │
                                          └─────────────────────┘
```

所有组件共享一个**上下文对象（IngestionContext）**，避免参数爆炸：

```python
@dataclass
class IngestionContext:
    doc_uuid: str
    source_kb: str
    file_bytes: bytes
    file_name: str
    file_hash: str
    tenant_id: str           # 从Dify /info 获取
    embedding_model: str     # 从Dify /datasets 获取
    embedding_provider: str
    chunking_config: dict    # 例如 {"max_tokens": 256, "overlap": 20}
```

---

### 二、组件 1：版本解析器（VersionResolver）

**职责**：仅负责判断“这个文件是新文档、已有文档还是需要重试”，**不做任何解析**。

#### 接口定义（Python ABC）

```python
from abc import ABC, abstractmethod

class VersionResolver(ABC):
    @abstractmethod
    def resolve(self, ctx: IngestionContext) -> VersionDecision:
        """
        根据 file_hash 查询 MySQL kb_documents 表。
        """
```

#### 输入/输出契约

| 输入 | 类型 | 说明 |
| :--- | :--- | :--- |
| `ctx.file_hash` | `str` | 全文SHA-256 |
| `ctx.source_kb` | `str` | 用于限定查询范围（防止跨库Hash碰撞） |

**输出结构：`VersionDecision`（枚举 + 数据）**

```python
@dataclass
class VersionDecision:
    action: Literal["skip", "new", "retry"]
    existing_doc_uuid: Optional[str] = None
    existing_version: Optional[str] = None
    retry_count: Optional[int] = None

    # 示例：
    # VersionDecision(action="skip", existing_doc_uuid="doc_abc")  # 直接返回，不处理
    # VersionDecision(action="new")  # 插入新记录，status=0
    # VersionDecision(action="retry", retry_count=1)  # status=2的文档重试
```

#### 内部实现逻辑（伪代码）

```python
def resolve(self, ctx):
    record = db.query("SELECT doc_uuid, version, status, retry_count FROM kb_documents WHERE file_hash = %s AND source_kb = %s", ...)
    if not record:
        return VersionDecision(action="new")
    if record.status == 1:
        return VersionDecision(action="skip", existing_doc_uuid=record.doc_uuid)
    if record.status == 2 and record.retry_count < 3:
        return VersionDecision(action="retry", retry_count=record.retry_count)
    # 超过重试次数，抛出异常，人工介入
    raise MaxRetryExceededError()
```

---

### 三、组件 2：MinerU 适配器（ParserAdapter）

**职责**：调用MinerU服务，将原始文件转化为标准化的`elements.json`，并存入对象存储缓存。

#### 接口定义

```python
class ParserAdapter(ABC):
    @abstractmethod
    def parse(self, ctx: IngestionContext) -> ParsedResult:
        """
        调用 MinerU HTTP API，返回标准化的元素结构。
        超时时间 120s，超时则抛出 TimeoutError。
        """
```

#### 输入/输出契约

**输入**：`ctx.file_bytes` + `ctx.file_name`（MinerU根据扩展名自动选择解析器）

**输出结构：`ParsedResult`**

```python
@dataclass
class ParsedResult:
    doc_uuid: str
    elements: List[Element]          # 核心输出
    full_merged_text: str            # 全文拼接，用于ES兜底
    statistics: dict                 # 页数、字数、表格数量等
    cache_path: str                  # 对象存储路径: /parsed_cache/{doc_uuid}/elements.json
    parse_warnings: List[dict]       # 非致命警告，如“第5页无文本层”

@dataclass
class Element:
    elem_id: str                     # "elem_0001"
    type: Literal["heading", "paragraph", "table", "image_placeholder", "equation"]
    level: Optional[int]             # 仅 heading 有
    chapter_path: str                # "第一章 > 宏观 > 数据"
    page_num: int
    bbox: List[float]                # [x0, y0, x1, y1]
    content_text: str                # 纯文本 / Markdown表格 / LaTeX公式
    extra: dict                      # MinerU原始信息透传
```

**契约约束**：
- `elem_id` 在文档内必须全局唯一。
- `chapter_path` 即使推断不准，也必须有值（兜底为`"第{page_num}页"`），否则切块器报错。
- 适配器**不写MySQL**，只写入对象存储缓存。

---

### 四、组件 3：切块引擎（ChunkerEngine）

**职责**：读取`elements.json`，执行“父块组装”和“子块拆分”，生成最终的`Chunk`列表。

#### 接口定义

```python
class ChunkerEngine(ABC):
    @abstractmethod
    def chunk(self, parsed: ParsedResult, ctx: IngestionContext) -> ChunkResult:
        """
        输入标准化元素，输出父子块列表（含 source_elem_ids 映射）。
        """
```

#### 输入/输出契约

**输入**：`ParsedResult`（来自适配器） + `ctx.chunking_config`（如 `max_tokens=256, overlap=20`）

**输出结构：`ChunkResult`**

```python
@dataclass
class ChunkResult:
    doc_id: int                      # MySQL自增ID，写入后回填
    parent_chunks: List[ParentChunkRecord]
    child_chunks: List[ChildChunkRecord]

@dataclass
class BaseChunkRecord:
    chunk_uuid: str                  # "p_xxx" 或 "c_xxx"
    content_type: Literal["PARENT", "CHILD"]
    content_text: str
    chapter_path: str
    token_count: int
    page_numbers: str                # "3-5" 或 "3"
    source_elem_ids: List[str]       # 核心溯源字段

@dataclass
class ParentChunkRecord(BaseChunkRecord):
    parent_chunk_id: None = None     # 父块该字段为 None
    children: List[str] = field(default_factory=list)  # 子块的 chunk_uuid 列表（仅供内存使用，不写库）

@dataclass
class ChildChunkRecord(BaseChunkRecord):
    parent_chunk_id: int             # 指向父块的 MySQL 自增 id（写入时回填）
```

**关键契约**：
- **父块的 `source_elem_ids`** = 它所包含的所有 `elem_id` 的并集（去重）。
- **子块的 `source_elem_ids`** = 继承自它所在的原始段落/表格的 `elem_id`（即使该段落被拆成2个子块，两个子块都指向同一个 `elem_id`）。
- 切块器**不写库**，只返回内存对象列表。

---

### 五、组件 4：三库写入器（TripleWriter）

**职责**：按顺序写入 MySQL → ES → Milvus，保证最终一致性。

#### 接口定义

```python
class TripleWriter(ABC):
    @abstractmethod
    def write(self, chunk_result: ChunkResult, ctx: IngestionContext) -> WriteStatus:
        """
        1. 批量插入 MySQL（获取父块ID）
        2. 批量写入 ES（仅子块）
        3. 批量写入 Milvus（仅子块）
        4. 更新 kb_documents.status = 1
        """
```

#### 输入/输出契约

**输入**：`ChunkResult` + `ctx`

**输出结构：`WriteStatus`**

```python
@dataclass
class WriteStatus:
    success: bool
    mysql_inserted: int
    es_indexed: int
    milvus_inserted: int
    error: Optional[str] = None
    retryable: bool = False          # True 表示可以补偿重试（如ES超时）
```

#### 内部写入细节（按顺序）

```python
def write(self, chunk_result, ctx):
    # Step 1: 批量插入 MySQL
    parent_ids = []
    with db.transaction():
        for p in chunk_result.parent_chunks:
            pid = db.insert("kb_chunks", {...content_type="PARENT", parent_chunk_id=None...})
            parent_ids.append(pid)
        for c in chunk_result.child_chunks:
            c.parent_chunk_id = parent_ids[c.parent_index]  # 回填
            db.insert("kb_chunks", {...content_type="CHILD", parent_chunk_id=c.parent_chunk_id...})
    
    # Step 2: 写入 ES (Bulk API, 每批500条，仅子块)
    es.bulk(index="kb_chunks_v1", actions=[{...} for c in chunk_result.child_chunks])
    
    # Step 3: 写入 Milvus (Bulk Insert, 每批1000条，仅子块)
    milvus.insert(collection="kb_embeddings_v1", data=[...])
    
    # Step 4: 更新文档主表状态
    db.update("kb_documents", {"status": 1, "indexed_at": now()}, where={"doc_uuid": ctx.doc_uuid})
```

**重要契约**：
- **父块不写入 ES / Milvus**，只存 MySQL。
- 如果 Step 2 失败，**不阻断 Step 3**，但最终 `status` 置为 2，补偿任务会补写。
- `chunk_uuid` 作为三库关联主键，必须**严格一致**（不能 MySQL 用 `p_xxx`，Milvus 用另一个ID）。

---

### 六、组件 5：入库调度器（IngestionOrchestrator）

**职责**：编排上述四个组件，实现状态机流转。这是**唯一对外暴露入口的组件**。

#### 接口定义（对外的内部入口）

```python
class IngestionOrchestrator:
    def __init__(self, resolver, parser, chunker, writer):
        self.resolver = resolver
        self.parser = parser
        self.chunker = chunker
        self.writer = writer

    def run(self, ctx: IngestionContext) -> IngestionResult:
        # 1. 版本决策
        decision = self.resolver.resolve(ctx)
        if decision.action == "skip":
            return IngestionResult(action="skip", doc_uuid=decision.existing_doc_uuid)
        
        # 2. 如果是 retry，清理旧数据（软删除 is_active=0）
        if decision.action == "retry":
            self._clean_old_chunks(decision.existing_doc_uuid)
            ctx.doc_uuid = decision.existing_doc_uuid  # 复用旧UUID
        
        # 3. 执行解析
        parsed = self.parser.parse(ctx)
        
        # 4. 执行切块
        chunks = self.chunker.chunk(parsed, ctx)
        
        # 5. 执行三库写入
        status = self.writer.write(chunks, ctx)
        
        # 6. 返回最终结果
        return IngestionResult(action="indexed", doc_uuid=ctx.doc_uuid, chunk_count=len(chunks.child_chunks))
```

---

### 七、异常处理与重试契约

内部组件间定义了统一的异常类型，供上层捕获决策：

| 异常类 | 触发组件 | 调度器处理策略 |
| :--- | :--- | :--- |
| `MaxRetryExceededError` | VersionResolver | 直接置 `status=2`，发告警，不触发后续 |
| `MinerUTimeoutError` | ParserAdapter | 置 `status=2`，`retry_count+1`，等待补偿 |
| `ChunkingValidationError` | ChunkerEngine | 置 `status=2`，记录 `last_error`（数据格式异常，不重试） |
| `ESWriteTimeoutError` | TripleWriter | 置 `status=2`，但 `retryable=True`，补偿任务只重写 ES |
| `MilvusConnectionError` | TripleWriter | 同上，仅重写 Milvus |

---

### 八、总结：组件接口的“硬边界”清单

| 组件 | 输入 | 输出 | 副作用（写操作） | 是否幂等 |
| :--- | :--- | :--- | :--- | :--- |
| **VersionResolver** | `file_hash` | `VersionDecision` | 无（只读） | 是 |
| **ParserAdapter** | `file_bytes` | `ParsedResult` | 写对象存储缓存 | 是（重复解析覆盖缓存） |
| **ChunkerEngine** | `ParsedResult` | `ChunkResult` | 无（纯内存计算） | 是 |
| **TripleWriter** | `ChunkResult` | `WriteStatus` | 写 MySQL/ES/Milvus | 部分（MySQL用`ON DUPLICATE KEY`，ES用`doc_as_upsert`，Milvus用`upsert`） |
| **Orchestrator** | `IngestionContext` | `IngestionResult` | 更新 `kb_documents.status` | 否（由VersionResolver控制重入） |

这套接口契约设计确保了每个组件可以独立演进（例如从MinerU切换到其他解析引擎只需重写`ParserAdapter`），且测试时可以轻松Mock。你可以以此作为开发团队的详细设计文档。