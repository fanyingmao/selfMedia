# 02 · 系统架构设计

## 2.1 总体架构（分层）

```
┌─────────────────────────────────────────────────────────────────┐
│                        接入层  (Edge)                          │
│  Web (Next.js)  ·  小程序  ·  桌面客户端  ·  Open API / SDK     │
└──────────────────────────┬──────────────────────────────────────┘
                           │ HTTPS / WSS
┌──────────────────────────▼──────────────────────────────────────┐
│                    API 网关 (Gateway)                          │
│  鉴权 · 限流 · 路由 · 灰度 · 审计  (Kong / APISIX / Nginx)      │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                      业务服务层 (BFF + Domain)                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │
│  │ 用户中心 │ │ 素材中心 │ │ 内容工作 │ │ 发布中心 │           │
│  │ Account  │ │ Material │ │ 台 Studio│ │ Publish  │           │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘           │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │
│  │ 数据中心 │ │ 计费中心 │ │ 通知中心 │ │ 审核中心 │           │
│  │ Insight  │ │ Billing  │ │ Notify   │ │ Moderate │           │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘           │
└──────────────────────────┬──────────────────────────────────────┘
                           │ gRPC / MQ
┌──────────────────────────▼──────────────────────────────────────┐
│                  AI 编排层 (AI Orchestrator)                   │
│  Workflow Engine · Prompt 模板中心 · Agent 路由 · 任务队列       │
│  (LangGraph / Dify-like / 自研 DAG)                             │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                    AI 能力层 (Model Adapters)                  │
│  LLM   │ 文生图 │ 文生视频 │ 图生视频 │  TTS  │  ASR  │ 翻译    │
│ OpenAI │ SDXL  │  Sora    │ Runway   │ Azure │Whisper│ DeepL    │
│ Claude │ MJ    │ 可灵 Kling│ Pika    │ ElevenLabs                │
│ 通义/文 │ Flux  │ 即梦     │ Vidu     │ MiniMax / 火山 TTS       │
│ 心/豆包 │       │          │          │                         │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                    基础设施层 (Infra)                          │
│ PostgreSQL · Redis · MinIO/S3 · Milvus/PGVector · Kafka         │
│ K8s · Prometheus · Grafana · Loki · OpenTelemetry · ArgoCD      │
└─────────────────────────────────────────────────────────────────┘
```

## 2.2 核心子系统

### 2.2.1 用户中心 (account-svc)
- 注册 / 登录 / OAuth / SSO
- 组织 / 团队 / 角色 / 权限（RBAC）
- 配额、套餐、Token 余额

### 2.2.2 素材中心 (material-svc)
- 多源接入：URL 抓取（Playwright + Readability）、文件解析（PDF / DOCX / MD / EPUB）、RSS、剪贴板
- 文本清洗、分段（Chunking）、向量化（Embedding）入库
- 素材标签、分组、检索（关键词 + 向量混合检索）
- 知识库版本管理

### 2.2.3 内容工作台 (studio-svc)
- 项目（Project）、内容稿（Draft）、版本（Revision）
- 编辑器后端：分镜 / 字幕 / 时间线数据结构（类似 Premiere XML / DaVinci 项目结构简化版）
- 协作锁、操作日志（OT / CRDT）

### 2.2.4 AI 编排层 (ai-orchestrator)
- DAG / 状态机定义流水线（采集 → 理解 → 选题 → 文案 → 分镜 → 图 → 视频 → 合成）
- 节点失败重试、断点续跑、人工审批节点
- Prompt 模板中心 + 变量注入 + 多语言
- 模型路由策略（按价格 / 速度 / 质量 / 区域 / 合规）
- 任务持久化（任务表 + 状态机日志）

### 2.2.5 AI 能力层 (model-adapter)
- 统一 SDK 抽象：`generate_text` / `generate_image` / `generate_video` / `tts` / `asr`
- 各厂商 Adapter：OpenAI、Anthropic、Google、阿里通义、百度文心、字节豆包、MiniMax、Kling、即梦、Runway、Pika、Vidu、ElevenLabs、火山 TTS …
- 统一输入输出 Schema、统一错误码、统一计量

### 2.2.6 媒资合成 (render-svc)
- FFmpeg / MoviePy / Remotion 渲染
- 字幕烧录、转场、变速、画中画、封面合成
- 渲染队列（GPU / CPU 节点池）

### 2.2.7 发布中心 (publish-svc)
- 平台适配器：Douyin / Xiaohongshu / WeChat Channels / Bilibili / YouTube / X / TikTok
- 账号托管（Cookie / OAuth Token / 浏览器指纹隔离）
- 定时任务、失败重试、防风控（频率、IP 池、设备指纹）
- 发布日志、平台返回的链接 / ID 回写

### 2.2.8 数据中心 (insight-svc)
- 平台数据回采（官方 API + 爬虫兜底）
- 维度：曝光 / 播放 / 完播 / 点赞 / 评论 / 涨粉 / 转化
- 报表 + 选题反向训练数据集

### 2.2.9 审核中心 (moderate-svc)
- 文本敏感词、合规规则（涉政 / 涉黄 / 涉暴 / 医疗金融特殊行业）
- 图片 / 视频内容审核（接 NSFW、涉政模型）
- AIGC 标识：水印、标签、平台合规接入

### 2.2.10 计费中心 (billing-svc)
- 套餐订阅、按量计费、Token 余额
- 模型成本归因（每个任务记录调用了哪些模型 / 多少 Token / 多少图 / 多少秒）

### 2.2.11 通知中心 (notify-svc)
- 站内信、邮件、短信、Webhook、IM Bot（飞书 / 钉钉 / 企业微信 / Slack / Telegram）
- 任务进度推送（WebSocket / SSE）

## 2.3 关键数据流

### 2.3.1 内容生产数据流（核心）

```
用户提交「项目」(含长文素材ID + 风格预设 + 平台目标)
        │
        ▼
studio-svc 创建 Project + Draft，发起 Pipeline 任务
        │
        ▼
ai-orchestrator 拉起 DAG，写入任务表 (jobs)，向 Kafka 发任务事件
        │
   ┌────┴────────────────────────────────────────────┐
   ▼                                                 ▼
 worker-llm (理解+文案+分镜)                worker-media (图/视频/TTS/合成)
   │                                                 │
   ▼                                                 ▼
 调 model-adapter -> 各 LLM 厂商                调 model-adapter -> SD/MJ/Kling/Runway/TTS
   │                                                 │
   ▼                                                 ▼
 结果写回 Draft (JSONB: outline/script/scenes)  素材落 MinIO/S3，元数据写 DB
        │                                                │
        └────────────────►  render-svc (FFmpeg/Remotion) ◄┘
                                   │
                                   ▼
                           成片 mp4 + 封面 jpg + 字幕 srt
                                   │
                                   ▼
                           Draft 状态变 Ready，推 WebSocket 通知前端
                                   │
                                   ▼
                           用户预览 / 二次编辑 / 提交发布
                                   │
                                   ▼
                       publish-svc 调度 → 各平台 Adapter
                                   │
                                   ▼
                       平台返回 video_id + url，写回数据库
                                   │
                                   ▼
                  insight-svc 定时拉取播放/互动数据
```

### 2.3.2 状态机 (Draft 状态)

```
DRAFT_INIT
   │ 创建项目
   ▼
ANALYZING       ── 长文理解 / 实体抽取
   │
   ▼
SCRIPTING       ── 文案 + 分镜
   │
   ▼
GENERATING_MEDIA ── 并发生成图 / 视频 / TTS
   │
   ▼
RENDERING       ── 合成 mp4
   │
   ▼
READY           ── 等待用户编辑 / 审核
   │
   ▼
APPROVED        ── 用户确认
   │
   ▼
SCHEDULED       ── 等待定时
   │
   ▼
PUBLISHING      ── 调用平台 API
   │
   ▼
PUBLISHED / FAILED
```

任意节点失败 → `FAILED`，可一键 retry 单节点（断点续跑）。

## 2.4 部署拓扑

```
                   ┌─────────────┐
                   │   CDN/WAF   │
                   └──────┬──────┘
                          │
                ┌─────────▼──────────┐
                │   Ingress (NGINX)  │
                └─────────┬──────────┘
                          │
   ┌──────────────────────┼──────────────────────┐
   ▼                      ▼                      ▼
 Web (Next.js SSR)   API Gateway          Static (S3+CDN)
                          │
   ┌──────────────────────┼──────────────────────┐
   ▼                      ▼                      ▼
 account-svc        studio-svc            publish-svc
 material-svc       insight-svc           moderate-svc
 billing-svc        notify-svc            ai-orchestrator
                          │
                          ▼
              ┌─────────────────────┐
              │   Kafka / Redis     │  ← 任务总线
              └──────────┬──────────┘
                         │
   ┌─────────────────────┼─────────────────────┐
   ▼                     ▼                     ▼
 worker-llm        worker-image         worker-video
 (CPU pool)        (GPU pool A)         (GPU pool B)
                         │
                         ▼
                    render-svc (GPU + ffmpeg)

      ┌─────────────┐  ┌──────────┐  ┌────────┐  ┌──────────┐
      │ PostgreSQL  │  │ MinIO/S3 │  │ Milvus │  │  Redis   │
      └─────────────┘  └──────────┘  └────────┘  └──────────┘
```

- **GPU 节点** 仅给 `worker-image` / `worker-video` / `render-svc` 用，按需弹性扩缩。
- **业务服务** 全部无状态，水平扩展。
- **任务总线** Kafka 解耦业务和 AI 算力。

## 2.5 横切关注点

- **可观测性**：OpenTelemetry → Tempo（trace）+ Loki（log）+ Prometheus（metric）+ Grafana。
- **安全**：所有上传素材服务端加密；模型 API Key 统一进 Vault；行级数据隔离（org_id 强约束）。
- **多租户**：所有业务表带 `org_id`；Postgres RLS（行级安全）+ 应用层双校验。
- **国际化**：UI（i18next）、Prompt（按 locale 选模板）、平台适配（按地域选发布通道）。
- **灰度与 AB**：模型路由灰度、Prompt 模板 AB、UI 灰度（Feature Flag，Unleash / Flagsmith）。
