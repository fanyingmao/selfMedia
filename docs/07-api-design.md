# 07 · API 设计

对外采用 **REST + OpenAPI 3.1**（便于第三方与 SDK 自动生成），Web 前端内部走 **tRPC**（类型安全 + 减少胶水代码），实时进度走 **WebSocket / SSE**。

## 7.1 通用约定

- BaseURL：`https://api.selfmedia.app/v1`
- 鉴权：`Authorization: Bearer <jwt>`，机器人 / OpenAPI 用 `X-API-Key`。
- 多租户：所有请求强制携带 `X-Org-Id`（或 token 内嵌）。
- 错误格式（RFC 7807 Problem Details）：

  ```json
  {
    "type": "https://errors.selfmedia.app/quota_exceeded",
    "title": "Quota exceeded",
    "status": 402,
    "detail": "monthly video quota 100 reached",
    "code": "BILLING_QUOTA_EXCEEDED",
    "request_id": "req_01HXXX..."
  }
  ```

- 分页：`?cursor=...&limit=20`，返回 `{items, next_cursor}`。
- 幂等：写操作支持 `Idempotency-Key` 头。
- 速率限制：`X-RateLimit-Remaining`、`Retry-After`。

## 7.2 资源与端点（核心子集）

### Auth
- `POST /auth/register`
- `POST /auth/login`
- `POST /auth/oauth/{provider}`
- `POST /auth/refresh`

### Org / Users / Members
- `GET /orgs/me`
- `POST /orgs`
- `GET /orgs/{id}/members`
- `POST /orgs/{id}/members` `{user_email, role}`

### Materials（素材）
- `POST /materials/url` `{url}` → 异步抓取，返回 `material_id`
- `POST /materials/upload` `multipart/form-data`
- `GET /materials?cursor=&q=`
- `GET /materials/{id}`
- `DELETE /materials/{id}`
- `POST /materials/{id}/reprocess`（重新清洗 + 向量化）
- `POST /materials/search` `{query, top_k}` → 向量混合检索

### Style Presets / Prompts
- `GET /styles`
- `POST /styles` (org 私有风格)
- `GET /prompts/templates?key=script.douyin`
- `POST /prompts/templates`（org 私有覆盖）

### Projects / Drafts
- `POST /projects` `{name, target_platforms, style_preset_id}`
- `GET /projects?cursor=`
- `GET /projects/{id}`
- `POST /projects/{id}/drafts` `{material_id, options}` → 启动整条 pipeline
- `GET /drafts/{id}` → 含 status、cost、阶段产物
- `PATCH /drafts/{id}` → 局部更新（标题、文案、镜头等）
- `POST /drafts/{id}/regenerate` `{stage: "script"|"storyboard"|"scene", scene_idx?}`
- `POST /drafts/{id}/render` → 触发合成
- `GET /drafts/{id}/revisions`
- `POST /drafts/{id}/revisions/{n}/restore`
- `DELETE /drafts/{id}`

### Scenes（分镜级编辑）
- `PATCH /drafts/{id}/scenes/{idx}` → 改字幕 / prompt / 时长
- `POST /drafts/{id}/scenes/{idx}/regenerate-image`
- `POST /drafts/{id}/scenes/{idx}/regenerate-video`
- `POST /drafts/{id}/scenes/{idx}/regenerate-tts`
- `POST /drafts/{id}/scenes/{idx}/upload-asset` → 用户自定义素材

### Assets
- `GET /assets/{id}` → 元数据
- `GET /assets/{id}/download` → 签名 URL

### Social Accounts
- `POST /social-accounts/{platform}/oauth-start`
- `GET /social-accounts/{platform}/oauth-callback`
- `GET /social-accounts`
- `DELETE /social-accounts/{id}`

### Publish
- `POST /publish` `{draft_id, account_ids[], scheduled_at?, caption_overrides?}`
- `GET /publish/tasks?status=&cursor=`
- `GET /publish/tasks/{id}`
- `POST /publish/tasks/{id}/retry`

### Insights
- `GET /insights/summary?range=7d`
- `GET /insights/by-account/{account_id}`
- `GET /insights/by-draft/{draft_id}`

### Billing
- `GET /billing/subscription`
- `POST /billing/subscribe` `{plan}`
- `GET /billing/usage?from=&to=`

### Webhooks（出站）
- `draft.status_changed`
- `publish.published`
- `publish.failed`
- `insights.daily`

## 7.3 Pipeline 端点示例（核心）

### 启动整条流水线

`POST /projects/{project_id}/drafts`

请求：

```json
{
  "material_id": "mat_01HXY...",
  "options": {
    "target_platforms": ["douyin", "xhs"],
    "duration_sec": 60,
    "style_preset_id": "style_cinematic_zh",
    "language": "zh-CN",
    "auto_publish": false
  }
}
```

响应：

```json
{
  "draft_id": "drf_01HXZ...",
  "status": "ANALYZING",
  "stages": [
    {"name": "understand", "status": "running"},
    {"name": "ideate", "status": "queued"},
    {"name": "script", "status": "queued"},
    {"name": "storyboard", "status": "queued"},
    {"name": "image", "status": "queued"},
    {"name": "video", "status": "queued"},
    {"name": "tts", "status": "queued"},
    {"name": "render", "status": "queued"}
  ]
}
```

### 实时进度（WebSocket）

`wss://api.selfmedia.app/v1/drafts/{draft_id}/events`

事件示例：

```json
{"type":"stage_started","stage":"script","at":"2026-04-25T03:01:00Z"}
{"type":"stage_progress","stage":"image","progress":0.6,"detail":"3/5 scenes"}
{"type":"stage_finished","stage":"render","asset_id":"ast_..."}
{"type":"draft_status","status":"READY"}
```

### 单镜头重生成

`POST /drafts/{id}/scenes/3/regenerate-image`

```json
{
  "model": "flux-pro",
  "prompt_override": "更强的电影感、暖色调、广角",
  "seed": 42
}
```

返回新 `asset_id` 与 `job_id`。

## 7.4 OpenAPI 与 SDK

- `services/*` 自动产出 OpenAPI 3.1 文档，统一聚合到 `apps/web` 的 `/api-docs`。
- 由 OpenAPI 生成 TypeScript SDK（`packages/ts/api-client`）和 Python SDK（`packages/py/sdk`）。
- 所有响应使用 zod / pydantic 双侧 schema 校验，避免前后端漂移。

## 7.5 错误码规范（节选）

| 区段 | 含义 |
| --- | --- |
| `AUTH_*` | 鉴权 / 权限 |
| `MATERIAL_*` | 素材采集与解析 |
| `PIPELINE_*` | AI 流水线运行时 |
| `MODEL_*` | 模型调用（超时 / 拒绝 / 限频） |
| `RENDER_*` | 媒资合成 |
| `PUBLISH_*` | 平台发布 |
| `MODERATION_*` | 内容审核拦截 |
| `BILLING_*` | 计费 / 配额 |

每个错误码在文档中需配备：含义、HTTP 状态、是否可重试、用户提示文案。
