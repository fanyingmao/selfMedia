# 04 · 技术选型

选型原则：**主流 + 社区活跃 + 云原生友好 + 国内外双部署友好 + AI 生态成熟**。

## 4.1 总览速查表

| 层 | 选型 | 备选 | 理由 |
| --- | --- | --- | --- |
| 前端 Web | Next.js 14（App Router）+ React 18 + TypeScript | Nuxt 3 / SvelteKit | SSR / RSC、生态最完整 |
| UI 组件 | shadcn/ui + Tailwind CSS + Radix | Ant Design / Arco | 设计灵活、易主题化 |
| 视频编辑器 UI | Remotion（声明式）+ 自研时间线 | WebCodecs + Konva | 复用 React，生成与编辑一致 |
| 状态管理 | Zustand + TanStack Query | Redux Toolkit | 轻量 + 服务端状态分离 |
| 移动端 | React Native（后期）/ 微信小程序（uni-app） | Flutter | 复用前端栈 |
| BFF / API | Node.js（NestJS）+ tRPC（内部）+ REST/OpenAPI（对外） | Go (Gin) | 与前端共享类型 |
| 业务微服务 | Python（FastAPI）+ Go（gRPC，关键性能路径） | Java Spring | Python 与 AI 生态贴合，Go 处理高并发 |
| AI 编排 | Python（自研 DAG 基于 LangGraph + Celery）+ Temporal（长事务） | Airflow / Prefect | 长事务与人工审批节点都需要 |
| 任务队列 | Celery + Redis（短任务）；Temporal（长事务、有状态工作流） | RabbitMQ / Sidekiq | 与 Python 自然 |
| 消息总线 | Kafka（事件流）+ Redis Streams（轻量） | Pulsar / NATS | 数据回流 + 流处理需要 |
| 数据库 | PostgreSQL 16 + pgvector | MySQL 8 | JSONB + 向量统一栈 |
| 向量库 | pgvector（小规模）/ Milvus（大规模） | Qdrant / Weaviate | 平滑升级 |
| 缓存 | Redis 7 | KeyDB | 标准 |
| 对象存储 | S3（线上）/ MinIO（自托管 / 开发） | OSS / COS | 双云兼容 |
| 搜索 | Meilisearch / OpenSearch | Elastic | 中文友好 |
| 网关 | APISIX | Kong / Traefik | 国内云友好、插件丰富 |
| 鉴权 | Keycloak / 自研 + JWT + OAuth2 | Auth0 | 私有部署需要 |
| 审核 | 自建（敏感词 + LLM）+ 阿里云内容安全 / 腾讯天御 | AWS Rekognition | 国内合规 |
| 渲染 | FFmpeg + Remotion + MoviePy | DaVinci CLI | 灵活组合 |
| 模型托管 | 优先调用云厂 API；自托管模型用 vLLM / Triton / ComfyUI | TGI | 灵活 |
| LLM | OpenAI / Anthropic / Google / 通义 / 文心 / 豆包 / Kimi / 智谱 | DeepSeek / MiniMax | 多模型路由 |
| 文生图 | Midjourney（封面） / Flux / SDXL / 通义万相 | DALL·E 3 | 风格丰富 |
| 文生视频 | 可灵 Kling / 即梦 / Runway Gen-3 / Pika / Vidu | Sora（待开放） | 多模型容灾 |
| TTS | ElevenLabs / MiniMax 海螺 / 火山 TTS / Azure TTS / CosyVoice | 阿里 TTS | 中英文兼顾 |
| ASR | Whisper-large-v3 / 火山 ASR | 通义听悟 | 字幕回采 |
| 容器 | Docker + Kubernetes（K3s 自托管 / EKS-ACK-GKE 云） | Nomad | 行业标准 |
| 服务网格 | Istio（可选）/ Linkerd | - | 大规模才上 |
| CI/CD | GitHub Actions + ArgoCD | GitLab CI / Jenkins | GitOps |
| IaC | Terraform + Helm + Kustomize | Pulumi | 多云友好 |
| 可观测 | OpenTelemetry → Prometheus + Loki + Tempo + Grafana | DataDog | 开源全家桶 |
| 错误追踪 | Sentry | Bugsnag | 标准 |
| Feature Flag | Unleash / Flagsmith | LaunchDarkly | 自托管 |
| 密钥管理 | Vault / SOPS | AWS Secrets Manager | 模型 Key 强管理 |
| 浏览器自动化 | Playwright（采集 + 部分平台发布兜底） | Puppeteer | 跨浏览器 |

## 4.2 重点说明

### 4.2.1 为什么 Python + Node.js + Go 三栈？

- **Node.js（NestJS）**：BFF、Web SSR、tRPC，与前端类型共享，开发体验最优。
- **Python（FastAPI / Celery）**：AI 编排、模型调用、媒资处理（FFmpeg 绑定、PIL、moviepy），生态最贴合。
- **Go**：发布服务、网关附加、数据回采等高并发 IO 路径，按需引入；MVP 不强求。

> MVP 阶段可先 **只用 Python + Node.js 双栈**，Go 在 V2 性能阶段再引入。

### 4.2.2 LLM 调用层抽象

统一封装为 `model-adapter`（Python package），对外暴露：

```python
class LLMClient(Protocol):
    async def chat(self, messages, *, model: str, temperature: float, response_format=None) -> ChatResponse: ...
    async def embed(self, texts: list[str], *, model: str) -> list[list[float]]: ...

class ImageClient(Protocol):
    async def text_to_image(self, prompt, *, model, size, n, style=None) -> list[ImageAsset]: ...
    async def image_to_image(self, image, prompt, *, model, strength) -> list[ImageAsset]: ...

class VideoClient(Protocol):
    async def text_to_video(self, prompt, *, model, duration, ratio) -> VideoAsset: ...
    async def image_to_video(self, image, prompt, *, model, duration, ratio) -> VideoAsset: ...

class TTSClient(Protocol):
    async def synthesize(self, text, *, voice, model, speed=1.0, emotion=None) -> AudioAsset: ...
```

每个厂商实现一份 Adapter，统一计量（token / image / second）回写 billing。

### 4.2.3 工作流：LangGraph + Temporal 双栈

- **LangGraph**：定义 AI DAG（理解 → 选题 → 文案 → 分镜 → 媒资），适合短链路、易迭代。
- **Temporal**：跨小时 / 跨天的长事务（视频生成等待 GPU 队列、人工审批、定时发布、数据回流），需要持久化、重试、可视化。

二者通过事件桥接：LangGraph 节点产物落库 → Temporal Activity 调度后续阶段。

### 4.2.4 媒资合成：FFmpeg + Remotion 组合

- **Remotion**：用 React 写"视频模板"（封面、字幕样式、转场、品牌包装），声明式、可复用、利于 AB。
- **FFmpeg**：底层硬合成、拼接、转码、烧字幕、加水印。
- **MoviePy**：Python 端灵活粘合（少量复杂剪辑）。

模板分两层：
1. **风格模板**（Remotion 组件）：决定字幕样式 / 转场 / 包装。
2. **内容模板**（JSON）：决定每个镜头的素材、时长、字幕。

### 4.2.5 多平台发布

| 平台 | 接入方式 | 备注 |
| --- | --- | --- |
| YouTube | 官方 Data API v3 | OAuth2，最稳定 |
| TikTok | 官方 Content Posting API | 申请门槛较高 |
| 抖音 | 抖音开放平台 / 千川 API | 需企业资质；个人号走浏览器自动化兜底 |
| 视频号 | 微信公众平台 API（受限）| 复杂场景走助手 |
| 小红书 | 蒲公英开放平台 / 浏览器自动化 | 部分能力受限 |
| B 站 | B 站开放平台 | 上传 + 投稿 |
| X / Twitter | v2 API | 标准 |

> 浏览器自动化（Playwright）作为兜底，必须按平台 ToS 评估合规风险，并提供"仅本地运行"模式。

### 4.2.6 国内 / 海外双部署

- **海外**：AWS / GCP，模型用 OpenAI / Anthropic / Google / Runway / ElevenLabs。
- **国内**：阿里云 / 腾讯云，模型用通义 / 文心 / 豆包 / Kimi / 智谱 / Kling / 即梦 / 火山 TTS / 阿里内容安全。
- **统一抽象**：通过 `model-adapter` + `storage-adapter` + `moderation-adapter` 切换，业务代码无感。
