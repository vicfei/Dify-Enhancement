好的，基于您已有的任务体系（任务4～9的API设计风格）以及项目文件存储的决策，我将为**任务10（召回评测）**设计一套完整的RESTful API接口。这套接口将用于管理黄金问答集、触发评测流水线、查询评测状态和获取评测报告，所有数据均以文件形式存储，不依赖数据库扩展。

---

# 任务10：召回评测接口设计文档（v1.0）

## 一、概述

### 1.1 背景
任务10构建了独立的**召回评测流水线**，用于持续追踪检索（任务6）和排序（任务7）的质量。评测数据（黄金问答对、运行记录、指标结果）全部以文件形式存储于`./evaluation/`目录下，不修改现有数据库模型。

### 1.2 服务定位
- **接口前缀**：`/api/v1/evaluation`
- **内容类型**：`application/json`（文件上传除外）
- **认证方式**：沿用系统现有的API Key Bearer Token（与任务6/7一致）

### 1.3 存储路径约定（仅供参考，接口内部实现）
```
evaluation/
├── datasets/
│   └── v1/
│       ├── golden_fund.json
│       └── ...
├── runs/
│   └── {run_id}/
│       ├── config.yaml
│       ├── raw_results.jsonl
│       ├── metrics.json
│       └── report.md
├── baselines/
│   ├── baseline_20260801.json
│   └── current_baseline.json
```

---

## 二、接口列表

| 方法 | 路径 | 功能 |
| :--- | :--- | :--- |
| POST | `/api/v1/evaluation/datasets` | 上传或更新黄金问答集 |
| GET | `/api/v1/evaluation/datasets` | 查询当前黄金问答集信息 |
| POST | `/api/v1/evaluation/runs` | 触发一次评测运行 |
| GET | `/api/v1/evaluation/runs/{run_id}` | 获取某次评测的状态和结果 |
| GET | `/api/v1/evaluation/runs` | 列出所有评测运行历史 |
| GET | `/api/v1/evaluation/runs/{run_id}/report` | 下载评测报告（Markdown/HTML） |
| POST | `/api/v1/evaluation/baseline` | 将某次运行设为基线 |
| GET | `/api/v1/evaluation/baseline` | 获取当前基线信息 |

---

## 三、详细接口定义

### 3.1 上传黄金问答集
**`POST /api/v1/evaluation/datasets`**

#### 请求体
支持两种方式：

- **直接提交JSON数组**（适合小规模）：
```json
[
  {
    "id": "fund_001",
    "question": "XX债券的最新信用评级是多少？",
    "ground_truth": "XX债券最新主体评级为AA+，展望稳定。",
    "expected_chunk_ids": ["chunk_abc123", "chunk_def456"],
    "expected_doc_ids": ["doc_789"],
    "source_kb": "credit",
    "difficulty": "easy",
    "question_form": "MATCH",
    "risk_level": "high"
  }
]
```

- **上传文件**（multipart/form-data）：
  - 字段名：`file`
  - 文件格式：JSON（结构与上述数组一致）或 CSV（需符合预定义schema）

#### 响应
```json
{
  "status": "success",
  "dataset_version": "v1.0",
  "record_count": 45,
  "updated_at": "2026-08-07T10:30:00Z"
}
```

#### 错误码
| 状态码 | code | 说明 |
| :--- | :--- | :--- |
| 400 | `invalid_dataset_format` | JSON/CSV格式错误 |
| 400 | `missing_required_fields` | 缺少必填字段 |
| 409 | `dataset_version_conflict` | 版本冲突 |

---

### 3.2 查询黄金问答集信息
**`GET /api/v1/evaluation/datasets`**

#### 查询参数（可选）
| 参数 | 类型 | 说明 |
| :--- | :--- | :--- |
| `version` | string | 指定版本，默认最新 |

#### 响应
```json
{
  "dataset_version": "v1.0",
  "total_records": 45,
  "by_kb": {
    "credit": 15,
    "fund": 12,
    "equity": 18
  },
  "by_difficulty": {
    "easy": 20,
    "medium": 15,
    "hard": 10
  },
  "created_at": "2026-08-01T00:00:00Z",
  "updated_at": "2026-08-07T10:30:00Z"
}
```

---

### 3.3 触发评测运行
**`POST /api/v1/evaluation/runs`**

#### 请求体
```json
{
  "name": "Rerank开启对比测试",          // 可选，自定义运行名称
  "config": {
    "search_api": "http://localhost:8080/api/v1/search",
    "rank_api": "http://localhost:8080/api/v1/rank",  // 可选，不填则仅评测召回
    "k": 10,
    "rerank_enabled": false,
    "source_kbs": ["credit", "fund"],    // 为空则评测所有知识库
    "dataset_version": "v1.0"            // 可选，默认使用最新
  }
}
```

#### 响应（异步启动）
```json
{
  "run_id": "run_20260807_103045_abc123",
  "status": "queued",
  "estimated_time_seconds": 120,
  "created_at": "2026-08-07T10:30:45Z"
}
```

---

### 3.4 获取评测详情
**`GET /api/v1/evaluation/runs/{run_id}`**

#### 响应
```json
{
  "run_id": "run_20260807_103045_abc123",
  "name": "Rerank开启对比测试",
  "status": "completed",          // queued | running | completed | failed
  "config": { ... },               // 完整配置快照
  "progress": {
    "total_queries": 45,
    "processed": 45,
    "failed": 0
  },
  "metrics": {
    "overall": {
      "recall@5": 0.72,
      "recall@10": 0.86,
      "mrr": 0.68,
      "ndcg@10": 0.71
    },
    "by_kb": {
      "credit": { "recall@5": 0.80, "recall@10": 0.90, "mrr": 0.75 },
      "fund": { "recall@5": 0.65, "recall@10": 0.82, "mrr": 0.60 }
    },
    "vs_baseline": {
      "recall@5": "+0.05",
      "recall@10": "+0.02",
      "mrr": "-0.01"
    }
  },
  "started_at": "2026-08-07T10:30:50Z",
  "completed_at": "2026-08-07T10:32:10Z"
}
```

#### 错误码
| 状态码 | code | 说明 |
| :--- | :--- | :--- |
| 404 | `run_not_found` | 指定的run_id不存在 |

---

### 3.5 列出评测历史
**`GET /api/v1/evaluation/runs`**

#### 查询参数
| 参数 | 类型 | 说明 |
| :--- | :--- | :--- |
| `limit` | int | 返回条数，默认20 |
| `offset` | int | 偏移，默认0 |
| `status` | string | 按状态筛选（queued/running/completed/failed） |
| `from` | date | 起始日期 |
| `to` | date | 结束日期 |

#### 响应
```json
{
  "total": 12,
  "limit": 20,
  "offset": 0,
  "items": [
    {
      "run_id": "run_20260807_103045_abc123",
      "name": "Rerank开启对比测试",
      "status": "completed",
      "created_at": "2026-08-07T10:30:45Z",
      "metrics": {
        "recall@10": 0.86,
        "mrr": 0.68
      }
    }
  ]
}
```

---

### 3.6 下载评测报告
**`GET /api/v1/evaluation/runs/{run_id}/report`**

#### 查询参数
| 参数 | 类型 | 说明 |
| :--- | :--- | :--- |
| `format` | string | `md`（默认）或 `html` |

#### 响应
返回文件流，`Content-Type` 为 `text/markdown` 或 `text/html`，`Content-Disposition: attachment`。

---

### 3.7 设置基线
**`POST /api/v1/evaluation/baseline`**

#### 请求体
```json
{
  "run_id": "run_20260807_103045_abc123"
}
```

#### 响应
```json
{
  "status": "success",
  "baseline_run_id": "run_20260807_103045_abc123",
  "updated_at": "2026-08-07T10:35:00Z"
}
```

#### 错误码
| 状态码 | code | 说明 |
| :--- | :--- | :--- |
| 400 | `run_not_completed` | 该运行尚未完成 |
| 404 | `run_not_found` | 不存在 |

---

### 3.8 获取当前基线
**`GET /api/v1/evaluation/baseline`**

#### 响应
```json
{
  "baseline_run_id": "run_20260801_090000_xyz789",
  "metrics": {
    "recall@5": 0.70,
    "recall@10": 0.84,
    "mrr": 0.67
  },
  "set_at": "2026-08-01T09:00:00Z"
}
```
若未设置基线，返回 `{"baseline_run_id": null}`。

---

## 四、错误码统一规范

所有接口遵循以下错误响应格式：

```json
{
  "status": 400,
  "code": "invalid_param",
  "message": "参数`k`必须为正整数",
  "details": null
}
```

通用错误码：

| code | 说明 |
| :--- | :--- |
| `invalid_param` | 请求参数非法 |
| `internal_error` | 服务器内部错误（如文件读写失败） |
| `service_unavailable` | 依赖服务（搜索/排序API）不可用 |
| `rate_limited` | 请求频率超限 |

---

## 五、与现有任务6/7的集成

评测运行器内部调用现有接口：

- **任务6召回**：`POST /api/v1/search`（返回候选块列表及`recall_score`）
- **任务7排序**（可选）：`POST /api/v1/rank`（返回排序后列表）

评测接口本身不修改这两个接口的任何行为，仅作为消费者。

---

## 六、文件存储实现说明（内部）

接口后端实现遵循以下存储约定：

- **黄金问答集**：存于`evaluation/datasets/{version}/`，以JSON文件按知识库拆分。
- **运行记录**：每个`run_id`对应一个目录，包含：
  - `config.yaml`：请求体中的配置
  - `raw_results.jsonl`：每行一个问题的召回结果明细
  - `metrics.json`：聚合指标（与接口响应中的`metrics`结构一致）
  - `report.md`：生成的Markdown报告
- **基线**：`evaluation/baselines/current_baseline.json` 软链接或指向某个运行目录。

---

## 七、后续扩展

- **对比分析**：可增加`POST /api/v1/evaluation/compare`接口，对比两个运行的结果差异。
- **趋势图数据**：增加`GET /api/v1/evaluation/trend`，返回历史指标时间序列。
- **CI/CD集成**：支持通过Webhook触发评测，返回结果用于阻塞合并请求。

---

> 文档版本：v1.0  
> 编写日期：2026-08-07  
> 状态：待评审