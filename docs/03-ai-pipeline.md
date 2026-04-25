# 03 · AI 内容生产流水线

本文档描述「长文 → 短视频」的端到端 AI 流水线（DAG），以及每一步的 Prompt 设计、模型选择和质量控制策略。

## 3.1 流水线总图（DAG）

```
                ┌───────────────────────────┐
                │  0. ingest 素材采集与清洗 │
                └─────────────┬─────────────┘
                              │
                ┌─────────────▼─────────────┐
                │  1. understand 长文理解   │
                │   摘要 / 关键观点 / 实体  │
                └─────────────┬─────────────┘
                              │
                ┌─────────────▼─────────────┐
                │  2. ideate 选题与角度     │
                │   平台调性 / 热点融合     │
                └─────────────┬─────────────┘
                              │
                ┌─────────────▼─────────────┐
                │  3. script 短视频文案     │
                │   钩子 / 口播稿 / Hashtag │
                └─────────────┬─────────────┘
                              │
                ┌─────────────▼─────────────┐
                │  4. storyboard 分镜脚本   │
                │   场次 / 画面 / 时长      │
                └─────────────┬─────────────┘
                              │
        ┌─────────────┬───────┴────────┬──────────────┐
        ▼             ▼                ▼              ▼
 5a. image gen   5b. video gen   5c. tts 配音    5d. bgm 选乐
 (文/图生图)    (文/图生视频)   (口播/角色音)   (音乐 / 音效)
        │             │                │              │
        └─────────────┴────────┬───────┴──────────────┘
                               ▼
                ┌─────────────────────────────┐
                │  6. compose 合成 (FFmpeg)   │
                │  字幕 / 转场 / 封面 / 水印  │
                └─────────────┬───────────────┘
                              │
                ┌─────────────▼───────────────┐
                │  7. moderate 审核           │
                │  文本 + 图像 + 视频         │
                └─────────────┬───────────────┘
                              │
                ┌─────────────▼───────────────┐
                │  8. review 人工微调         │
                └─────────────┬───────────────┘
                              │
                ┌─────────────▼───────────────┐
                │  9. publish 多平台发布      │
                └─────────────────────────────┘
```

## 3.2 阶段详解

### 阶段 0 · ingest（采集与清洗）

- **输入**：URL / PDF / DOCX / MD / 纯文本 / RSS 条目。
- **处理**：
  - URL：Playwright 渲染 + Mozilla Readability 抽正文。
  - PDF / DOCX：`unstructured` / `pdfplumber` / `python-docx` 解析为结构化段落。
  - 清洗：去广告、去版权声明、保留小标题层级。
  - 分段（Chunking）：按 800~1200 token 切，保留章节信息。
  - 向量化：`bge-m3` / `text-embedding-3-large` 入库。
- **产物**：`material_id`, `chunks[]`, `outline`（章节树）。

### 阶段 1 · understand（长文理解）

**目标**：把长文压缩成结构化"知识卡"，为后续选题和文案提供事实底座。

**输出 JSON Schema**：

```json
{
  "summary": "150 字以内整体摘要",
  "key_points": [
    {"point": "...", "evidence_chunk_ids": ["c1","c5"]}
  ],
  "entities": {
    "person": ["..."], "org": ["..."], "place": ["..."],
    "product": ["..."], "concept": ["..."]
  },
  "audience": {
    "level": "入门|进阶|专业",
    "interest_tags": ["AI","创业","健康"]
  },
  "tone": "专业|轻松|犀利|励志|科普",
  "hot_potential": 0.0,
  "controversy_risk": 0.0
}
```

**Prompt 要点**（系统提示节选）：

> 你是资深内容编辑。请阅读以下长文，输出严格符合给定 JSON Schema 的结构化结果。
> 规则：
> 1. 不要捏造事实，所有 key_points 必须能在原文 chunk_ids 中找到证据。
> 2. summary 控制在 150 字以内，使用客观陈述。
> 3. 评估 hot_potential（0~1，越高越具备成为热点的潜质）和 controversy_risk（0~1）。

**模型选择**：默认 GPT-4o-mini / Claude Haiku / Qwen-Plus（性价比层）；长文（>30k token）走 Gemini 1.5 Pro / Qwen-Long。

### 阶段 2 · ideate（选题与角度）

**目标**：基于"知识卡 + 平台调性 + 热点库"生成 N 个候选选题。

**输入**：
- 阶段 1 的 JSON
- 目标平台（抖音 / 小红书 / 视频号 / B 站 / YouTube Shorts）
- 用户选择的「风格预设」（口播 / 图文 / 影视感 / 卡通 ...）
- 当日热搜（爬取 + 缓存）

**输出**：

```json
{
  "topics": [
    {
      "title": "为什么 90% 的人都误解了这件事？",
      "angle": "反直觉切入",
      "hook": "第一句钩子，3 秒抓住注意力",
      "platform_fit": {"douyin": 0.9, "xhs": 0.6, "bili": 0.7},
      "estimated_ctr": 0.08
    }
  ]
}
```

**Prompt 要点**：注入「平台调性手册」（小红书更感性、B 站更专业长内容、抖音强钩子、视频号偏中老年友好），以及「爆款标题模式库」（数字 / 反差 / 提问 / 痛点 / 揭秘 / 对比 / 故事）。

### 阶段 3 · script（短视频文案）

**目标**：把选定 topic 写成 30~90 秒的逐字口播稿 + 字幕分句 + Hashtag。

**输出**：

```json
{
  "title": "...",
  "hook": "前 3 秒：...",
  "body": [
    {"sentence": "...", "duration_ms": 3200, "emotion": "好奇"},
    {"sentence": "...", "duration_ms": 2800, "emotion": "强调"}
  ],
  "cta": "结尾引导：关注我看下集",
  "hashtags": ["#AI", "#创业", "#干货"],
  "platform_variants": {
    "xhs": {"caption": "小红书风格的图文文案，emoji 多一点 ✨"},
    "douyin": {"caption": "抖音口播稿同上"}
  }
}
```

**质量控制**：
- 平均句长不超过 18 字（适配字幕）。
- 每 7 秒必须有"信息钩子"（数字 / 反问 / 转折）。
- 结尾必须有 CTA。
- 自动跑"可读性分数"（中文字数 / 标点密度）。

### 阶段 4 · storyboard（分镜脚本）

**目标**：把口播稿切成 N 个镜头，每个镜头给出画面 prompt。

**输出**：

```json
{
  "scenes": [
    {
      "index": 0,
      "duration_ms": 3000,
      "subtitle": "...",
      "voice_text": "...",
      "visual_prompt": "电影感俯拍，城市夜景，霓虹灯，赛博朋克风，8k",
      "visual_negative_prompt": "low quality, watermark, text",
      "camera": "俯拍 / 推镜 / 拉镜",
      "transition_in": "fade",
      "transition_out": "cut",
      "asset_strategy": "video|image|stock|user_upload"
    }
  ]
}
```

**关键策略**：
- **风格一致性**：整条视频复用同一套 style preset（关键词 + LoRA + seed）。
- **人脸一致性**：若涉及主角，使用 IP-Adapter / Reference Image 锁定。
- **素材策略**：短/快镜头优先文生图 + Ken Burns 推拉；强调动态用文生视频；常识镜头走免版权素材库（Pexels / Pixabay）。

### 阶段 5a · image gen（文生图 / 图生图）

- **模型路由**：
  - 高质量封面 → Midjourney v6 / Flux.1 Pro
  - 批量分镜图 → SDXL / Flux Dev / 通义万相
  - 写实人像 → Flux + IP-Adapter
  - 卡通 / 国风 → 自训 LoRA
- **输出规格**：竖版 1080×1920（短视频）/ 横版 1920×1080（B 站、YouTube）/ 方版 1080×1080（视频号封面）。
- **质量门**：自动 NSFW 检测、清晰度 / 美学打分（aesthetic score < 5 重生成）。

### 阶段 5b · video gen（文生视频 / 图生视频）

- **模型**：可灵 Kling、即梦、Runway Gen-3、Pika、Vidu、Sora（如可用）。
- **策略**：
  - 优先「图生视频」：先生成 keyframe 再驱动，可控性更强。
  - 单镜头时长 ≤ 5 秒，避免模型穿帮。
  - 失败回退：视频生成失败 → 退化为图片 + Ken Burns 动效。

### 阶段 5c · tts（配音）

- **模型**：ElevenLabs、MiniMax 海螺、火山 TTS、Azure TTS、CosyVoice。
- **能力**：
  - 多语言、情感控制、语速控制。
  - 角色音色克隆（用户上传 30 秒样本）。
  - 输出对齐时间戳（用于字幕烧录）。

### 阶段 5d · bgm / sfx

- 免版权曲库（Epidemic Sound / Artlist / 自建）+ AI 配乐（Suno / Udio）。
- 根据 `tone` 自动选风格；根据时长自动剪辑。

### 阶段 6 · compose（合成）

- **引擎**：FFmpeg（命令行编排）+ Remotion（声明式 React 视频）+ MoviePy（Python 灵活拼接）。
- **能力**：
  - 字幕烧录（多样式：综艺花字 / 极简 / 电影感）。
  - 转场、变速、画中画、品牌 logo / 水印。
  - 自动封面（首帧 + 标题排版）。
  - 输出多规格：竖版 1080×1920、横版 1920×1080、方版。

### 阶段 7 · moderate（审核）

- 文本：敏感词字典 + LLM 二次判定（涉政 / 涉黄 / 医疗金融 / 版权）。
- 图片 / 视频：NSFW 模型、明星人脸识别、Logo 识别。
- AIGC 标识：按各平台规则烧录"AI 生成"标识。

### 阶段 8 · review（人工微调）

- Web 时间线编辑器：替换镜头、改字幕、改配音、调时长。
- 一键"重新生成本镜头"。
- 协作评论。

### 阶段 9 · publish（多平台发布）

- 平台 Adapter 统一接口：`publish(account, asset, meta) -> {video_id, url}`。
- 定时调度（cron + 时区）。
- 失败重试 + 风控（频率、IP、设备指纹）。

## 3.3 Prompt 模板中心

Prompt 不写死在代码里，独立成"模板"对象，统一管理：

```yaml
id: script.douyin.v3
locale: zh-CN
model_pref: ["gpt-4o", "claude-3.5-sonnet", "qwen-max"]
temperature: 0.7
inputs:
  - name: knowledge_card  # 阶段1输出
  - name: topic           # 阶段2选定topic
  - name: target_duration # 60
system: |
  你是抖音爆款短视频文案专家。请按以下规则改写...
user: |
  【知识卡】{{knowledge_card}}
  【选题】{{topic}}
  【目标时长】{{target_duration}} 秒
  请输出严格 JSON：{...schema...}
output_schema_ref: schemas/script.v3.json
guardrails:
  - max_sentence_chars: 18
  - require_cta: true
  - banned_keywords_ref: dict/banned.zh.txt
```

模板支持版本化、AB 实验、按 `org_id` 私有化覆盖。

## 3.4 模型路由策略

```
Router(input) -> Model
  if input.size > 30k token  -> long_context_pool (Gemini 1.5 Pro / Qwen-Long)
  elif task in (script, storyboard) and quality=high
                             -> top_pool (GPT-4o, Claude 3.5 Sonnet)
  elif task=ideation         -> creative_pool (Claude, GPT-4o)
  elif task=summarize        -> cheap_pool (gpt-4o-mini, qwen-plus)
  region=CN & compliance=strict
                             -> cn_pool (通义/文心/豆包/智谱)
```

实现成「策略 YAML + 运行时打分」，可灰度。

## 3.5 质量与评测

- **离线评测集**：100 篇标注长文 + 人工挑选 ground truth 选题与文案。
- **指标**：
  - 摘要忠实度（FactScore）
  - 选题相关性（人工 1~5 分）
  - 文案钩子强度（人工 + LLM-as-judge）
  - 出图美学分（aesthetic predictor）
  - 端到端时长 / 成本
- **在线评测**：发布后 7 日 CTR / 完播 / 点赞回流，作为 reward 信号。

## 3.6 失败与降级

| 故障点 | 降级策略 |
| --- | --- |
| 主 LLM 超时 | 自动切换备用模型（同池 fallback） |
| 文生视频失败 | 退化为图片 + Ken Burns 动效 |
| TTS 失败 | 切换备用 TTS；最差用系统 TTS |
| 渲染失败 | 重试 3 次；保留中间产物供人工合成 |
| 平台发布失败 | 指数退避重试；失败 3 次告警，进入人工队列 |
