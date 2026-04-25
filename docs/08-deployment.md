# 08 · 部署与运维

## 8.1 环境分级

| 环境 | 用途 | 数据 | 模型 |
| --- | --- | --- | --- |
| local | 个人开发 | docker-compose 起 PG/Redis/MinIO/Kafka | 真实 API（少量额度）+ Mock 适配器 |
| dev | 团队联调 | 共享小集群 | 真实 API（共享 Key） |
| staging | 上线前演练 | 与生产同结构、脱敏数据 | 与生产同模型 |
| prod | 生产 | 多 AZ 高可用 | 多模型路由 + 容灾 |

## 8.2 本地一键启动

`deploy/compose/docker-compose.yml` 启动：

- postgres 16（带 pgvector 扩展）
- redis 7
- minio + minio-console
- kafka + schema-registry
- meilisearch
- temporal + temporal-ui
- jaeger + prometheus + grafana + loki

`Makefile` 入口：

```
make up          # 起依赖
make dev-web     # pnpm dev
make dev-svc     # 启动主要服务（FastAPI/NestJS hot reload）
make dev-worker  # 起 worker（CPU 池）
make seed        # 灌测试数据
make e2e         # Playwright 跑端到端
```

## 8.3 容器化与 K8s

每个服务一份 Dockerfile（多阶段构建），镜像产物推到内部 Registry。

K8s 编排：

- **Namespace**：`selfmedia-{env}`
- **Workload**：业务服务用 `Deployment`，Worker 用 `Deployment` + `KEDA` 自动伸缩（按 Kafka lag / Redis queue 长度）。
- **GPU 节点池**：节点打 `accelerator=nvidia` 标签，`worker-image / worker-video / render-svc` 的 Pod 用 `nodeSelector` 调度。
- **存储**：DB 走云托管 PG（RDS / 阿里云 PolarDB / 腾讯云 TDSQL）；对象用 S3 / OSS / COS。
- **配置**：ConfigMap + External Secrets Operator → Vault。
- **入站**：APISIX Ingress；TLS 由 cert-manager + Let's Encrypt。
- **服务网格**：默认不上；规模化后再引入 Linkerd。

Helm Chart：每个服务一份；`infra/helm/` 集中。`umbrella-chart` 用 ArgoCD 部署。

## 8.4 GitOps（CI / CD）

```
开发者 push -> GitHub Actions
   ├── lint / test / build
   ├── 构建镜像 + push 到 Registry
   ├── 更新 infra/argocd/values 中的 image tag
   └── ArgoCD 检测到 Git 变更 -> 自动同步到 K8s
```

- 主干策略：`main` = staging 自动；`release/*` 分支 = prod 手动 promote。
- Canary：APISIX + Argo Rollouts 流量灰度（5% → 25% → 100%）。
- 数据库变更：Atlas / Sqitch 管理迁移，CI 强制 `migration plan` 评审。

## 8.5 可观测性（O11y）

- **Tracing**：OpenTelemetry SDK 统一接入；每个 pipeline job 一条 trace，跨服务串联（Kafka 头注入 traceparent）。
- **Metrics**：核心 SLI
  - `pipeline_duration_seconds{stage}`
  - `model_call_seconds{provider,model,task}`
  - `model_cost_cents{provider,model}`
  - `render_seconds`
  - `publish_success_ratio{platform}`
  - `queue_lag{queue}`
- **Logs**：JSON 结构化；Loki + Grafana。
- **告警**：Prometheus Alertmanager → 飞书 / Slack / PagerDuty。
  - 关键告警：模型调用错误率 > 5%、渲染失败率 > 2%、发布失败率 > 5%、GPU 节点不健康、队列堆积。
- **业务看板**：Grafana 面板 + ClickHouse / DuckDB 聚合 `usage_events` 做实时计费看板。

## 8.6 容量与成本

- **GPU 弹性**：用 Spot / 抢占式实例 + 队列优先级；峰值用按需，平时用 Spot。
- **批处理优惠**：模型 API 走"批量模式"（OpenAI Batch / Anthropic Batch）做次日交付的低优任务降本。
- **缓存**：相同 prompt + 相同 seed 的图片 / 视频做内容寻址缓存（CAS，按 hash）。
- **预算告警**：按 `org_id` 每日成本 > 阈值 自动触发降级（切换 cheap_pool）+ 通知。

## 8.7 备份与灾备

- **PG**：每日全备 + 流式 WAL 归档；7 日内可 PITR。
- **对象存储**：跨区域复制（成片 / 封面），中间产物按生命周期清理。
- **配置 / 密钥**：Vault 周期快照，加密备份。
- **灾备演练**：每季度一次"机房切换"演练。

## 8.8 安全基线

- 强制 HTTPS / HSTS；内部服务默认 mTLS（如启用网格）。
- 密钥分级：模型 API Key、平台账号 Token、用户密码 hash 都进 Vault；只在运行时注入内存。
- 审计日志：所有"管理类"动作（账号绑定 / 角色修改 / 计费变更 / 发布动作）落审计表，180 天留存。
- 漏洞扫描：Trivy 扫镜像；Snyk / Dependabot 扫依赖；GitHub Code Scanning。
- WAF：海外 Cloudflare；国内阿里云 WAF。
- DDoS：云厂高防 + 限流。

## 8.9 SLO 建议

| 指标 | 目标 |
| --- | --- |
| API 可用性 | 99.9% |
| 创建草稿 → READY 时延 P95 | ≤ 5 min |
| 单镜头重生成 P95 | ≤ 60 s |
| 发布任务成功率 | ≥ 98% |
| 数据回采延迟 | ≤ 1 h |
