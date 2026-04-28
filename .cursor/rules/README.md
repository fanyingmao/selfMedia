# Cursor 项目规则

本目录存放本仓库给 Cursor 使用的"项目规则"（Project Rules）。Cursor Agent 在工作时会按 frontmatter 自动加载相应规则。

| 文件 | 作用 | 加载时机 |
| --- | --- | --- |
| `project.mdc` | 全局硬约束（架构、目录、依赖方向、安全、错误码、自验证） | `alwaysApply: true`，每次都加载 |
| `ai-task-template.mdc` | 单个任务卡的执行 SOP（Step 0~6） | 当编辑 `services / workers / packages / apps / tests / schemas / prompts` 下的文件时自动加载 |

## 怎么用？

在 Cursor 里打开本仓库 → 切到 Agent 模式 → 把 `docs/11-ai-collaboration-guide.md §11.5` 的某张任务卡贴进 chat → AI 会自动按 `ai-task-template.mdc` 的 SOP 执行，并受 `project.mdc` 约束。

## 想新增规则？

- 需要"全仓所有人都遵守" → 写到 `project.mdc`。
- 仅在某个目录 / 某种文件类型生效 → 新建 `*.mdc`，frontmatter 用 `globs: ["..."]` 限定范围。
- 仅在 chat 显式 `@` 引用时加载 → frontmatter 不写 `alwaysApply` 与 `globs`，让用户主动调用。

## 参考

- 设计文档总览：[`docs/README.md`](../../docs/README.md)
- AI 协作开发指南：[`docs/11-ai-collaboration-guide.md`](../../docs/11-ai-collaboration-guide.md)
