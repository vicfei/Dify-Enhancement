基于 Dify 官方 API 文档，以下是对预处理服务所需交互的 Dify 接口的详细拆解。

### 认证方式

所有对 Dify API 的请求都必须通过在 HTTP Header 中携带 API Key 来完成认证。

```http
Authorization: Bearer {api_key}
```

**关键说明**：
- **API Key 类型**：知识库 API 密钥具有访问该账户下所有可见知识库的权限，需妥善保管。
- **安全性**：API Key 必须存储在服务端，绝不能嵌入客户端代码或公开暴露。

### 接口1：验证 API Key 并获取租户信息

**接口名称**：`GET /info`

**用途**：
1.  **验证 API Key 有效性**：确认调用方提供的 `api_key` 是有效的。
2.  **获取租户/应用上下文**：解析响应中的 `mode` 和 `author_name` 等字段，用于确定请求来源的租户身份，实现数据隔离。

**请求示例**：

```bash
curl --request GET \
  --url https://{api_base_url}/info \
  --header 'Authorization: Bearer {api_key}'
```

**成功响应示例** (`200 OK`)：

```json
{
  "name": "My Chat App",
  "description": "一个有用的客服聊天机器人。",
  "tags": ["customer-service", "chatbot"],
  "mode": "chat",
  "author_name": "Dify Team"
}
```
*响应字段说明*：`mode` 表示应用类型（如 `chat`, `workflow` 等），`author_name` 可关联到工作空间或租户。

**失败响应** (`401 Unauthorized`)：
当 API Key 缺失或无效时，会返回 `401` 状态码。

### 接口2：校验知识库 (source_kb) 是否存在

**接口名称**：`GET /datasets/{dataset_id}`

**用途**：
1.  **验证 `source_kb` 有效性**：确认用户请求中传入的 `source_kb` (即 `dataset_id`) 在 Dify 平台中真实存在。
2.  **获取知识库配置**：响应中的 `embedding_model` 和 `embedding_model_provider` 等信息，是后续写入 Milvus 时保证向量维度一致的关键。

**请求示例**：

```bash
curl --request GET \
  --url https://{api_base_url}/datasets/{dataset_id} \
  --header 'Authorization: Bearer {api_key}'
```

**成功响应示例** (`200 OK`)：

```json
{
  "id": "c42e2a6e-40b3-4330-96f8-f1e4d768e8c9",
  "name": "Product Documentation",
  "description": "产品 API 技术文档",
  "provider": "vendor",
  "permission": "only_me",
  "indexing_technique": "high_quality",
  "embedding_model": "text-embedding-3-small",
  "embedding_model_provider": "langgenius/openai/openai",
  // ... 其他字段
}
```

### 接口3：获取可用的 Embedding 模型列表

**接口名称**：`GET /workspaces/current/models/model-types/text-embedding`

**用途**：
1.  **校验 Embedding 模型配置**：当用户指定或系统配置了某个 Embedding 模型时，调用此接口验证该模型是否在当前工作空间中可用且已配置。
2.  **获取模型详细信息**：响应中会包含模型的 `context_size` 等属性，可用于优化向量化策略。

**请求示例**：

```bash
curl --request GET \
  --url https://{api_base_url}/workspaces/current/models/model-types/text-embedding \
  --header 'Authorization: Bearer {api_key}'
```

**成功响应示例** (`200 OK`)：

```json
{
  "data": [
    {
      "provider": "langgenius/openai/openai",
      "label": {
        "en_US": "OpenAI",
        "zh_Hans": "OpenAI"
      },
      "status": "active",
      "models": [
        {
          "model": "text-embedding-3-small",
          "label": {
            "en_US": "text-embedding-3-small",
            "zh_Hans": "text-embedding-3-small"
          },
          "model_type": "text-embedding",
          "model_properties": {
            "context_size": 8191
          },
          "status": "active",
          "deprecated": false
        }
        // ... 其他模型
      ]
    }
    // ... 其他供应商
  ]
}
```

### （可选）接口4：检查文档索引状态

> **说明**：此接口主要用于在**未完全采用自研管道**的场景下，作为对 Dify 原生索引流程的补充。在你的设计中，由于完全替换了 Dify 的 ETL 链路，**此接口可能不是必需的**，但了解其用途有助于理解 Dify 的原生行为。

**接口名称**：`GET /datasets/{dataset_id}/documents/{batch}/indexing-status`

**用途**：
当通过 Dify 原生 API（如 `/create-by-file`）创建文档时，用于轮询其异步索引任务的进度。

**请求示例**：

```bash
curl --request GET \
  --url https://{api_base_url}/datasets/{dataset_id}/documents/{batch}/indexing-status \
  --header 'Authorization: Bearer {api_key}'
```

**成功响应示例** (`200 OK`)：

```json
{
  "data": [
    {
      "id": "a8e0e5b5-78c6-4130-a5ce-25feb0e0b4ac",
      "indexing_status": "completed",
      "processing_started_at": 1741267200,
      "parsing_completed_at": 1741267200,
      "cleaning_completed_at": 1741267200,
      "splitting_completed_at": 1741267200,
      "completed_at": 1741267200,
      "error": null,
      "completed_segments": 5,
      "total_segments": 5
    }
  ]
}
```

### 总结：接口交互矩阵

| Dify 接口 | 调用时机 | 获取的关键信息 | 在预处理服务中的用途 |
| :--- | :--- | :--- | :--- |
| `GET /info` | 服务启动或首次请求时 | `mode`, `author_name` | **鉴权与租户识别**：验证 API Key，绑定租户身份 |
| `GET /datasets/{dataset_id}` | 接收到 `source_kb` 后 | `embedding_model`, `embedding_model_provider` | **获取知识库配置**：确保向量写入与 Dify 配置一致 |
| `GET /workspaces/current/models/model-types/text-embedding` | （可选）模型校验时 | 可用 Embedding 模型列表及属性 | **模型有效性校验**：验证用户指定的模型是否可用 |
| `GET /.../indexing-status` | （非必需）跟踪原生任务时 | 文档索引进度和状态 | **任务进度跟踪**：仅在依赖 Dify 原生索引时使用 |