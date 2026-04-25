# 05 · 目录结构（Monorepo）

采用 **pnpm workspace + Turborepo（Node 部分）+ uv / poetry workspace（Python 部分）** 的多语言 Monorepo。

```
selfmedia/
├── README.md
├── docs/                              # 设计文档（即本目录）
│
├── apps/                              # 可独立部署的应用
│   ├── web/                           # Next.js 14 前端 + BFF
│   │   ├── app/
│   │   ├── components/
│   │   ├── lib/
│   │   ├── server/                    # tRPC routers / server actions
│   │   └── package.json
│   │
│   ├── admin/                         # 运营后台（Next.js）
│   │
│   └── landing/                       # 官网营销站（Next.js / Astro）
│
├── services/                          # 后端微服务（Python / Node / Go）
│   ├── account-svc/                   # FastAPI
│   ├── material-svc/                  # FastAPI（采集、解析、向量化）
│   ├── studio-svc/                    # NestJS（项目 / 草稿 / 时间线）
│   ├── ai-orchestrator/               # FastAPI + LangGraph + Temporal worker
│   ├── render-svc/                    # FastAPI + FFmpeg/Remotion runner
│   ├── publish-svc/                   # NestJS（多平台发布适配）
│   ├── insight-svc/                   # FastAPI（数据回采 + 报表）
│   ├── moderate-svc/                  # FastAPI（审核）
│   ├── billing-svc/                   # NestJS（订阅、Token 计费）
│   ├── notify-svc/                    # NestJS（通知）
│   └── gateway/                       # APISIX 配置 + 自定义插件
│
├── workers/                           # 计算密集型 Worker（按池区分）
│   ├── worker-llm/                    # Python，调用 LLM
│   ├── worker-image/                  # Python + GPU（SDXL/Flux/ComfyUI）
│   ├── worker-video/                  # Python + GPU（文生/图生视频）
│   ├── worker-tts/                    # Python（TTS 适配）
│   └── worker-asr/                    # Python（Whisper）
│
├── packages/                          # 跨服务共享（TS / Python 各一）
│   ├── ts/
│   │   ├── ui/                        # shadcn 封装组件库
│   │   ├── config/                    # eslint / tsconfig / tailwind preset
│   │   ├── shared-types/              # 共享 TS 类型（zod schema）
│   │   ├── api-client/                # 自动生成的 OpenAPI / tRPC 客户端
│   │   └── editor/                    # 视频编辑器核心（Remotion + 时间线）
│   │
│   └── py/
│       ├── model_adapter/             # LLM/Image/Video/TTS 统一 SDK
│       ├── prompt_center/             # Prompt 模板加载与渲染
│       ├── pipeline_kit/              # DAG / 节点 / 重试 工具集
│       ├── media_kit/                 # FFmpeg / MoviePy 封装
│       ├── storage_kit/               # S3 / MinIO / OSS 适配
│       ├── moderation_kit/            # 审核统一接口
│       └── common/                    # 日志 / 配置 / OTel / 错误码
│
├── prompts/                           # Prompt 模板（YAML），独立版本管理
│   ├── understand/
│   ├── ideate/
│   ├── script/
│   ├── storyboard/
│   └── styles/                        # 风格预设
│
├── schemas/                           # JSON Schema（管道各阶段输出）
│   ├── knowledge_card.v1.json
│   ├── topic.v1.json
│   ├── script.v3.json
│   └── storyboard.v2.json
│
├── infra/                             # 基础设施代码
│   ├── terraform/                     # 多云 IaC
│   ├── helm/                          # 各服务 Helm Chart
│   ├── kustomize/
│   ├── docker/                        # Dockerfile 集中
│   └── argocd/                        # GitOps 应用清单
│
├── deploy/                            # 部署相关脚本
│   ├── compose/                       # 本地 docker-compose
│   ├── scripts/
│   └── envs/                          # 环境变量样例
│
├── tools/                             # 一次性脚本与开发工具
│   ├── seed-data/
│   ├── prompt-eval/                   # 离线评测套件
│   └── benchmarks/
│
├── tests/                             # 端到端测试
│   ├── e2e/                           # Playwright
│   └── load/                          # k6 / Locust
│
├── .github/
│   └── workflows/                     # CI/CD
│
├── package.json                       # pnpm root
├── pnpm-workspace.yaml
├── turbo.json
├── pyproject.toml                     # uv / poetry workspace root
└── Makefile                           # 一键 up / down / lint / test
```

## 5.1 关键约定

### 命名
- 服务全部以 `-svc` 结尾。
- Worker 全部以 `worker-` 开头。
- 共享库放 `packages/`，按语言分 `ts/` 和 `py/`。

### 依赖方向
- `apps/*` 只能依赖 `packages/ts/*` 和 `services/*` 暴露的 OpenAPI / tRPC。
- `services/*` 只能依赖 `packages/py/*`、`packages/ts/*`、`schemas/`、`prompts/`。
- `workers/*` 只能依赖 `packages/py/*` 与 `schemas/`。
- 服务间不直接调用，必须通过 Gateway / 消息总线 / gRPC 接口契约。

### 配置
- 12-Factor，配置全部走环境变量。
- 本地开发：`deploy/envs/.env.example` → 拷贝为 `.env.local`。
- 线上：通过 K8s Secret + Vault 注入。

### 提交规范
- Conventional Commits（feat/fix/docs/chore/refactor/perf/test）。
- 每个 PR 必须带：变更说明、影响范围、回滚方案、是否影响计量与计费。
