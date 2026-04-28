# 11 · AI 协作开发指南（Cursor 实战版）

> 本指南面向"用 **Cursor + AI** 把这个仓库从设计阶段（M0）写到能跑（M1 MVP）→ 可商用（M2 V1）"的开发者。
>
> 核心思想：**人不写代码，但人是"产品经理 + 验收 + 仲裁者"**。AI 写代码，但必须按本指南的"任务卡 → 自验证 → 提交"三步走，不允许自由发挥。

---

## 11.1 为什么需要这份指南？

直接对 AI 说"帮我把 SelfMedia 写出来"会失败，原因有 4 个：

1. **范围太大**：1 个仓库 = 10+ 微服务 + 5 个 Worker + 1 个前端 + 1 个编辑器，AI 一次写不完，也记不住。
2. **缺乏约束**：AI 会自由命名、自由建目录、自由换技术栈，破坏 Monorepo 一致性。
3. **缺乏验证**：AI 写完声称"完成了"，但没启动、没测试、没接通，下一轮接力时全是坑。
4. **跨语言协作**：仓库横跨 TS / Python / SQL / YAML / Helm，AI 容易在错误语言里写错误代码。

本指南通过 **3 件事** 解决：

| 工具 | 作用 | 文件 |
| --- | --- | --- |
| 任务卡（Task Card） | 把"模糊大目标"变成"AI 一次能吃下的小任务"，每张卡 ≤ 4 小时人审 | 本文 §11.5 |
| Cursor 项目规则（.mdc） | 在仓库里钉死"AI 必须遵守的硬约束" | `.cursor/rules/*.mdc` |
| AI 自验证 SOP | AI 写完必须自己跑：lint → 单测 → 启服务 → curl → 截图 → 写 PR | 本文 §11.4 |

---

## 11.2 角色与边界

| 角色 | 谁来做 | 做什么 |
| --- | --- | --- |
| 产品 / 需求 | **你** | 拆任务、决定优先级、最终验收 |
| 设计 / 架构 | **你 + AI**（已完成 docs/01~10） | 任何架构变更必须先改文档 |
| 编码 / 测试 / 文档 | **AI（Cursor Agent）** | 写代码、写单测、跑通自验证、写 PR 描述 |
| 验收 / 仲裁 | **你** | code review、决定合并、决定回滚 |
| 运维 / 上线 | **你 + AI** | AI 写 Helm/CI，你执行集群操作 |

**红线**：

- 任何"修改 Schema、修改 API 契约、修改目录结构、引入新技术栈"的需求，AI **必须先改 `docs/`，等你确认后再写代码**。
- 任何外部副作用（真实的支付、真实的发布到平台、真实地花钱调用昂贵模型）默认 **mock**，AI 不允许擅自接真实凭据。

---

## 11.3 一次完整的 AI 协作循环（推荐流程）

```
你：从 §11.5 任务卡里挑一张 → 复制到 Cursor Chat → 选 Agent 模式
        │
        ▼
AI：阅读相关 docs/、相关已存在代码、相关 schemas/
        │
        ▼
AI：（必要时）先改文档 / Schema → 给你确认（在 chat 里展示 diff）
        │
        ▼
你：确认 / 修正
        │
        ▼
AI：在指定目录下写代码 + 写单测 + 写示例
        │
        ▼
AI：执行 §11.4 的"自验证清单"（命令 + 输出贴回 chat）
        │
        ▼
AI：git add / commit / push / 开 Draft PR，PR 描述按 §11.6 模板
        │
        ▼
你：review，要求 AI 修改 / 合并
```

**口诀**：**1 张卡 = 1 个分支 = 1 个 PR = 1 次自验证日志**。不要在一张卡里同时改 5 个服务。

---

## 11.4 AI 自验证清单（每张任务卡都必须跑）

> AI 在 Cursor 里"自称写完了"是不算数的。必须按以下顺序执行命令，并把命令 + 输出贴到 PR 描述里。

### A. 静态自验证（无外部依赖）

1. **依赖装得上**
   - TS：`pnpm install`（或 `pnpm --filter <pkg> install`）
   - Python：`uv sync` / `poetry install`（在对应 service / worker 目录）
2. **类型 / Lint 通过**
   - TS：`pnpm --filter <pkg> typecheck && pnpm --filter <pkg> lint`
   - Python：`ruff check .` + `mypy .`（按 `pyproject.toml` 配置）
3. **格式化干净**
   - `pnpm format` / `ruff format .`，`git diff` 应该是空的（除了本任务的改动）
4. **单元测试通过**
   - TS：`pnpm --filter <pkg> test`
   - Python：`pytest -q`，覆盖率新增代码 ≥ 70%（关键服务 ≥ 85%）
5. **Schema 校验**（涉及 pipeline 输出时）
   - 用 `schemas/*.json` 对模型输出做 JSON Schema 校验，必须 100% 通过

### B. 动态自验证（启服务）

只要任务涉及"对外接口"（HTTP / gRPC / WebSocket / CLI），AI 必须：

6. **能启动**：起对应服务（`make dev-svc <name>` / `uvicorn ... --reload` / `pnpm --filter <pkg> dev`），日志里没有 ERROR。
7. **健康检查**：`curl /health` 返回 200。
8. **核心 happy path**：用 `curl` 或 `httpie` 跑 1 个端到端例子，输出 200 + 期望 JSON，**贴日志**。
9. **失败路径**：构造 1 个非法请求，确认返回正确的错误码（参见 `docs/07-api-design.md §7.5`）。

### C. 流水线集成验证（涉及 ai-orchestrator / worker 时）

10. **mock 模型**跑通 1 条端到端 pipeline（understand → ideate → script → storyboard → image → tts → render），输出最终 mp4 / json，并把产物路径贴回 PR。
11. **断点续跑**：人为把某个 stage 标 failed，触发 retry，确认幂等。

### D. 提交前最后一关

12. **commit 之前**重新跑一遍 A1~A5；任何步骤失败 = 任务未完成，禁止提交。
13. **PR 描述里贴出 §11.6 模板的"自验证日志"段落**，没有这段一律不合并。

> 如果 AI 跑不动某一步（缺凭据、缺 GPU），必须**显式**在 PR 里说明"⚠️ 跳过 X，原因 Y，建议 Z"，不允许默默跳过。

---

## 11.5 任务卡总目录（按里程碑组织）

下面把 `docs/09-roadmap.md` 的 **M1 MVP** 与 **M2 V1** 拆成 AI 可执行的具体卡片。每张卡都按统一格式给出。

### 任务卡通用格式

```
ID:        TASK-<里程碑>-<编号>
标题:      <一句话>
依赖:      [前置 TASK-ID]
触达目录:  [<只允许动这些目录>]
禁止动:    [<尤其不能动的目录 / 文件>]
输入:      <已存在的代码 / 文档 / Schema>
产出:      <代码、文件、迁移、测试>
AI 约束:   <硬性规则，写在 prompt 里>
自验证:    <对应 §11.4 哪些步骤 + 额外命令>
人审重点: <你 review 时最该看什么>
```

> 把"任务卡通用格式"这一段直接复制到 Cursor，就是给 AI 的 prompt 骨架。

---

### M1 · MVP 阶段（"1 篇长文 → 1 条 60 秒 YouTube Shorts"端到端跑通）

> M1 的目标：**一条主链路活着**。所有支线（多平台、矩阵、计费、协作）一律放后面。
>
> M1 全部用 **Mock Adapter** 起步，跑通主链路再逐步替换为真实模型。

#### TASK-M1-00 · 仓库脚手架（Monorepo 基线）

- **依赖**：无
- **触达目录**：根目录（`package.json` / `pnpm-workspace.yaml` / `turbo.json` / `pyproject.toml` / `Makefile` / `.editorconfig` / `.gitignore` / `.nvmrc` / `.python-version`）
- **禁止动**：`docs/`（除非加新章节）、`schemas/`（这一卡先不动）
- **输入**：`docs/05-directory-structure.md`
- **产出**：
  - 严格按文档建好空目录骨架（用 `.gitkeep` 占位）
  - pnpm workspace + Turborepo 配置
  - Python uv / poetry workspace 配置
  - `Makefile` 至少包含 `up / down / dev-web / dev-svc / dev-worker / lint / test`
  - `.editorconfig` / `.gitignore` / `.nvmrc`（Node 20）/ `.python-version`（3.11）
- **AI 约束**：
  - 不允许引入未在 `docs/04-tech-stack.md` 里出现的依赖
  - 任何包都必须先有 `package.json` / `pyproject.toml`，不能裸建空目录
- **自验证**：A1 / A2 / A3；额外 `make lint` 必须返回 0
- **人审重点**：目录树是否与 `docs/05-directory-structure.md` 一致

#### TASK-M1-01 · 本地 docker-compose 依赖栈

- **依赖**：TASK-M1-00
- **触达目录**：`deploy/compose/`、`deploy/envs/`
- **产出**：
  - `docker-compose.yml` 起 Postgres 16 (含 pgvector)、Redis 7、MinIO、Kafka（KRaft 模式即可）、Temporal、Jaeger
  - `.env.example`：列出所有需要的环境变量（DB URL、S3、API Keys 留空占位）
  - `make up` / `make down` 验证可起停
- **AI 约束**：必须用 `pgvector/pgvector:pg16` 而不是 `postgres:16`
- **自验证**：
  - `make up && docker ps`，所有容器 healthy
  - `psql` 能连上 PG，`create extension vector;` 成功
  - `mc alias` 能访问 MinIO
- **人审重点**：环境变量命名是否 12-Factor，端口是否冲突

#### TASK-M1-02 · 共享包：`packages/py/common`（日志 / 配置 / 错误码）

- **依赖**：TASK-M1-00
- **触达目录**：`packages/py/common/`
- **产出**：
  - 配置：`pydantic-settings`，从环境变量加载
  - 日志：`structlog` JSON 格式，自动注入 `request_id` / `trace_id`
  - 错误码体系（按 `docs/07-api-design.md §7.5`）：`SelfMediaError(code, http_status, detail)`
  - OTel 初始化样板
  - `pytest` 覆盖以上所有
- **AI 约束**：
  - 错误码字符串必须从 `Enum` 取，不能写裸字符串
  - 所有公共函数必须有类型注解 + docstring
- **自验证**：A1~A4
- **人审重点**：错误码命名空间是否与 API 文档一致

#### TASK-M1-03 · 共享包：`packages/py/storage_kit`（S3 / MinIO 适配）

- **依赖**：TASK-M1-02
- **触达目录**：`packages/py/storage_kit/`
- **产出**：统一接口 `Storage.put / get / presigned_url / delete`，两个实现 `S3Storage` / `MinioStorage`
- **AI 约束**：
  - Key 必须遵守 `docs/06-data-model.md §6.4` 的模板
  - `presigned_url` 默认 15 分钟过期，必须可参数化
- **自验证**：A1~A4 + 用 docker-compose 起的 MinIO 跑集成测试
- **人审重点**：Key 模板是否一致

#### TASK-M1-04 · 共享包：`packages/py/model_adapter`（含 Mock 实现）

- **依赖**：TASK-M1-02
- **触达目录**：`packages/py/model_adapter/`
- **产出**：
  - 接口：`LLMClient` / `ImageClient` / `VideoClient` / `TTSClient`（按 `docs/04-tech-stack.md §4.2.2`）
  - 至少 2 个实现：`MockLLM` / `MockImage` / `MockTTS` / `MockVideo`（确定性输出，便于测试）+ 1 个真实实现 `OpenAILLM`（仅打通，不强求生产可用）
  - 统一返回 `usage`：`{tokens_in, tokens_out, images, seconds, cost_cents}`
  - 模型路由器 `ModelRouter`（读 `model_routes` 表 / 配置文件）
- **AI 约束**：
  - **不允许在 unit test 中调真实 API**；CI 全部走 Mock
  - 真实 Adapter 的 API Key 只能从环境变量读，禁止硬编码
- **自验证**：A1~A5；额外：`pytest -m integration` 在有 `OPENAI_API_KEY` 时跑一条 happy path
- **人审重点**：Mock 输出是否覆盖了下游 Schema 所需的所有字段

#### TASK-M1-05 · `schemas/` 全套 JSON Schema

- **依赖**：TASK-M1-00
- **触达目录**：`schemas/`
- **产出**：
  - `knowledge_card.v1.json` / `topic.v1.json` / `script.v3.json` / `storyboard.v2.json`
  - 字段必须与 `docs/03-ai-pipeline.md` 描述完全一致
  - 提供 Python `pydantic` 与 TS `zod` 两份生成器（或手写类型）
- **AI 约束**：
  - Schema 一旦定下，**任何字段变更必须新增版本号**（v3 → v3.1 / v4），禁止破坏性修改
- **自验证**：写 5 个 fixture（valid + invalid），用 `jsonschema` 校验
- **人审重点**：版本号策略

#### TASK-M1-06 · `prompts/` 模板中心（YAML + 加载器）

- **依赖**：TASK-M1-02 / TASK-M1-05
- **触达目录**：`prompts/`、`packages/py/prompt_center/`
- **产出**：
  - 按 `docs/03-ai-pipeline.md §3.3` 的 YAML 格式起步
  - 至少 4 个模板：`understand.zh.v1` / `ideate.douyin.v1` / `script.douyin.v3` / `storyboard.v2`
  - 加载器：变量注入（Jinja2）、版本选择、locale 选择、guardrails 校验
- **AI 约束**：
  - Prompt **不允许写死在 `.py` 文件里**，一律走 `prompt_center.render(key, version, vars)`
- **自验证**：A1~A4
- **人审重点**：prompt 是否会因为 jinja 注入引发提示词注入风险（保留原文要 escape）

#### TASK-M1-07 · `services/material-svc`（FastAPI · 素材采集与解析）

- **依赖**：TASK-M1-02 / TASK-M1-03
- **触达目录**：`services/material-svc/`、`infra/docker/material-svc.Dockerfile`
- **产出**：
  - REST 端点：`POST /materials/url` / `POST /materials/upload` / `GET /materials/{id}` / `POST /materials/search`（向量检索）
  - URL 抓取：`playwright` + `readability-lxml`
  - PDF / DOCX：`unstructured` 或 `pdfplumber` + `python-docx`
  - 分块：800~1200 token，保留章节
  - 写入 `materials / chunks / embeddings` 表（用 `pgvector`）
- **AI 约束**：
  - 一切外部网络请求必须有超时 + 重试上限（≤ 3 次）
  - URL 抓取**禁止**绕过 `robots.txt`
- **自验证**：A1~A9 + 用一个真实公众号文章 URL 跑通，贴回 chunks 数与第 1 段文本
- **人审重点**：分块边界是否合理（是否撕了句子）

#### TASK-M1-08 · `services/account-svc`（FastAPI · 用户与组织）

- **依赖**：TASK-M1-02
- **触达目录**：`services/account-svc/`
- **产出**：
  - 邮箱注册 / 登录 / JWT 颁发与刷新
  - 单组织模型（自动给每个用户建一个 personal org）
  - RBAC 中间件 + RLS context（`set local app.current_org_id = ...`）
- **AI 约束**：
  - 密码 `argon2`，禁用 `bcrypt < 2024`
  - JWT secret 只能从环境变量读
- **自验证**：A1~A9，覆盖"未授权访问其他 org 的资源"必须 403
- **人审重点**：JWT payload 是否泄漏多余字段

#### TASK-M1-09 · `services/ai-orchestrator`（FastAPI + LangGraph 单进程版）

- **依赖**：TASK-M1-04 / TASK-M1-05 / TASK-M1-06
- **触达目录**：`services/ai-orchestrator/`
- **产出**：
  - DAG：`understand → ideate → script → storyboard → image → tts → render`
  - 任务入口：`POST /pipelines/run` `{material_id, options}` → `draft_id`
  - 状态机持久化到 `jobs` / `drafts` 表
  - WebSocket：`/drafts/{id}/events`，按 `docs/07-api-design.md §7.3` 推送事件
  - 单节点失败 → 进入 `failed`；提供 `POST /jobs/{id}/retry`
- **AI 约束**：
  - 默认全部走 Mock Adapter；真实模型通过 env `MODEL_PROFILE=real` 切换
  - 节点之间禁止互相直接 import；只能通过 DAG 的 `state` 对象传递
- **自验证**：
  - A1~A9
  - C10 / C11（重点）：用 1 篇预置 markdown 跑完整 DAG，输出 `final.mp4`（mock 渲染就是占位 mp4 也可），WS 事件序列与文档一致
- **人审重点**：失败重试是否幂等

#### TASK-M1-10 · `workers/worker-llm`（Celery / Redis Streams）

- **依赖**：TASK-M1-04 / TASK-M1-09
- **触达目录**：`workers/worker-llm/`
- **产出**：
  - 消费队列 `q.llm`，调用 `model_adapter`，把结果写回 jobs 表
  - 健康检查、优雅退出、并发度可配
- **AI 约束**：
  - Worker 必须无状态、可水平扩展
  - **禁止**直接连业务 DB 修改业务表，只能写 `jobs.output`，由 orchestrator 收口
- **自验证**：A1~A8
- **人审重点**：背压策略

#### TASK-M1-11 · `services/render-svc`（FastAPI + FFmpeg）

- **依赖**：TASK-M1-03
- **触达目录**：`services/render-svc/`
- **产出**：
  - REST：`POST /render` `{storyboard_json, assets[], style_preset}` → `final_asset_id`
  - 用 `ffmpeg` 把：图片序列 + TTS 音频 + 字幕 srt → mp4（竖版 1080x1920）
  - 字幕烧录、片头片尾、AIGC 水印
- **AI 约束**：
  - 全部用 `ffmpeg` 子进程，**禁止**调用任何商业渲染 SaaS
  - 临时目录必须清理，禁止留垃圾
- **自验证**：A1~A9，最终产物 mp4 必须能用 `ffprobe` 校验时长 / 分辨率 / 编码
- **人审重点**：水印位置 + 字幕字体可读性

#### TASK-M1-12 · `services/publish-svc`（NestJS · 仅 YouTube Shorts）

- **依赖**：TASK-M1-08 / TASK-M1-11
- **触达目录**：`services/publish-svc/`
- **产出**：
  - OAuth：YouTube Data API v3
  - 上传：`/publish` 接口，按 `docs/07-api-design.md §7.2` 实现
  - 失败重试：指数退避 + 最多 3 次
- **AI 约束**：
  - YouTube 凭据**全部走 Mock**，提供 `--dry-run` 模式做 e2e 验证
  - 真实上传只在你手动给 OAuth Token 后才允许
- **自验证**：A1~A9 dry-run；提供"如何 OAuth"文档
- **人审重点**：失败回写 `publish_tasks` 是否正确

#### TASK-M1-13 · `apps/web` 最小可用前端

- **依赖**：TASK-M1-07 / TASK-M1-09 / TASK-M1-12
- **触达目录**：`apps/web/`
- **产出**：
  - 页面：登录 / 注册 / 素材上传 / 项目列表 / 项目详情（实时进度 WS）/ 草稿预览 / 发布按钮
  - 仅 5 个页面，UI 用 shadcn 默认主题即可
  - 自动从 OpenAPI 生成 TS Client（`packages/ts/api-client`）
- **AI 约束**：
  - 不允许直接 `fetch`，必须走 `api-client`
  - 任何页面级状态先用 TanStack Query，不要塞 Zustand
- **自验证**：A1~A4 + Playwright 跑 1 条 e2e（注册 → 上传 markdown → 等待 READY → 点发布 dry-run → 看到成功）
- **人审重点**：错误提示是否友好

#### TASK-M1-14 · M1 联调收口

- **依赖**：M1-00 ~ M1-13 全部完成
- **触达目录**：`tests/e2e/` / `tools/seed-data/` / `docs/09-roadmap.md`
- **产出**：
  - 一键脚本 `make e2e-mvp`：`make up && make seed && pytest tests/e2e/test_mvp.py`
  - 50 篇种子 markdown
  - 文档更新：`docs/09-roadmap.md` M1 章节加"实际完成情况"小节
- **AI 约束**：禁止改 §M1 的"完成判据"
- **自验证**：50 条种子全部跑通；P95 时长贴日志
- **人审重点**：实际成本 / 时长是否落在判据内

---

### M2 · V1 阶段（多平台 + 编辑器 + 计费）

> M2 的目标：**真正能被创作者用**。每张任务卡仍按上面格式拆，规模一致。

#### M2 任务卡清单（精简版，详细按 M1 风格补全）

| ID | 标题 | 主要触达目录 |
| --- | --- | --- |
| TASK-M2-01 | 抖音 / 小红书 / 视频号 Adapter（先 Playwright 兜底） | `services/publish-svc/adapters/` |
| TASK-M2-02 | 时间线编辑器（拖拽 / 替换素材 / 调时长） | `apps/web/`、`packages/ts/editor/` |
| TASK-M2-03 | 风格预设系统（10 套预设 + Remotion 模板） | `packages/ts/editor/`、`prompts/styles/` |
| TASK-M2-04 | 文生视频 Adapter（可灵 / 即梦 / Runway 选 1） | `packages/py/model_adapter/` |
| TASK-M2-05 | 内容审核服务 `services/moderate-svc`（敏感词 + NSFW） | `services/moderate-svc/` |
| TASK-M2-06 | 计费服务 `services/billing-svc`（套餐 + 按量 + Stripe） | `services/billing-svc/` |
| TASK-M2-07 | 通知服务 `services/notify-svc`（站内 + 邮件 + Webhook） | `services/notify-svc/` |
| TASK-M2-08 | Temporal 接管长事务（视频生成等待 / 定时发布） | `services/ai-orchestrator/` |
| TASK-M2-09 | OTel 全链路 + Grafana 看板（看 §08 SLO） | `infra/`、各服务 |
| TASK-M2-10 | V1 灰度发布 + 1000 用户 Beta 接入 | `infra/argocd/`、`apps/landing/` |

> 你在准备做 M2 时，**复制 M1 的任务卡格式**，把每张 M2 卡补完整 8 字段，再交给 AI。

---

### M3+ 阶段

略。等 M2 跑稳后再按同样方法拆，避免提前优化。

---

## 11.6 PR 描述模板（AI 必须用这个写）

```markdown
## 任务卡
TASK-M1-09 · ai-orchestrator 单进程版

## 改动范围
- services/ai-orchestrator/**
- packages/py/pipeline_kit/**

## 是否破坏性
- [ ] Schema 变更（如有，列出版本号迁移）
- [ ] API 契约变更
- [ ] 数据库迁移（附 SQL）
- [x] 不破坏

## 自验证日志（§11.4）
A1 pnpm install / uv sync ✅
A2 typecheck / ruff / mypy ✅
A3 format ✅
A4 pytest -q  → 42 passed (cov 81%)
A5 jsonschema → 5/5 fixtures ok
B6 uvicorn 启动 ✅，无 ERROR
B7 curl /health → 200
B8 happy path → 贴 curl 响应
B9 invalid → 422 + PIPELINE_INVALID_INPUT
C10 mock 全链路 → final.mp4 12.3s
C11 retry → 幂等通过

## 跳过项
⚠️ 真实 OpenAI 调用未跑（CI 无 OPENAI_API_KEY），仅 MockLLM 验证。

## 截图 / 产物
- 附 final.mp4 链接
- 附 WS 事件序列截图

## 回滚方案
revert 本 commit；DB 无迁移。
```

---

## 11.7 Cursor 使用建议

### 模式选择
- **Ask 模式**：你想让 AI 解释架构 / 找 bug / 评估方案，不允许改代码。
- **Agent 模式**：你已经把任务卡贴进去，让 AI 自己写 + 自己跑命令 + 自己提交。
- **Plan 模式**：跨多个文件 / 多个服务的复杂卡，让 AI 先出执行计划再开干。

### 上下文管理
- 给 AI 的 prompt 末尾**始终加**：
  > 请严格遵循 `.cursor/rules/project.mdc` 与 `docs/11-ai-collaboration-guide.md`。完成后按 §11.4 自验证清单跑一遍并把日志贴回来。
- 把当前任务卡 + 相关 docs 章节 + 相关已存在文件 用 `@` 引用进上下文，不要让 AI "盲猜"。
- 一次最多打开 1 张任务卡，不要 PR 串多张。

### 模型选择
- 大段架构 / 难点 → 用顶级模型（Claude / GPT-4 同档）。
- 大量样板 / 翻译 / 重复 → 用经济模型，节约预算。
- 单测生成 → 用顶级模型 + 一次只生成一个文件。

### 反模式（看到 AI 这么做就立即喊停）
- 凭空建了文档没提到的目录 / 文件
- 不写测试，或者测试只测 happy path
- 把密钥 / token 写进代码或 `.env.example`
- 跳过 §11.4 自验证就 commit
- 一个 PR 改 5 个不相关服务
- 引入未在 `docs/04-tech-stack.md` 的依赖
- "改完了"但既没启动也没 curl

---

## 11.8 总结：用一段话给 AI

> 你正在 SelfMedia 仓库工作。所有架构、Schema、API、目录结构都已经在 `docs/01~10` 里固化，**严禁擅自变更**。
> 你每次只接一张 §11.5 的任务卡，按卡片"触达目录 / 禁止动 / AI 约束"严格执行。
> 写完后，**必须**按 §11.4 自验证清单跑一遍并把日志贴在 PR 描述里。
> 任何破坏性变更（Schema / API / DB / 技术栈）必须先改 `docs/` 并等我确认，然后再写代码。
> 真实外部依赖（付费模型、平台账号）默认 mock，不要擅自调真实凭据。

把这段话粘到 Cursor 系统提示里，再开干。
