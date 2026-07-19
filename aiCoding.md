# aiCoding面试问题梳理

## 1、Spec 文档如何设计，才能让 AI 快速定位？

### 问题

Spec 是否可以按 `specs/场景/功能说明.md` 组织，并像 Skill 一样写清楚 `description`，让 AI 知道执行任务时应该查看和修改哪个文档？

### 回答

可以。推荐采用“目录分层 + 元数据路由”的设计：

- 按 `specs/<场景>/<功能>.md` 组织文档。
- 每份 Spec 通过 YAML Front Matter 声明 `description`、`triggers`、`owners`、`related` 等信息；`description` 重点说明“哪些任务需要查阅本文档”。
- 在 `specs/README.md` 建立统一索引，并在 `AGENTS.md` 中要求 AI 开始任务前匹配 Spec，业务行为发生变化后同步更新。
- 每项功能只保留一份权威 Spec，避免规则重复或分散。

```yaml
---
description: 涉及退款申请、审核、状态或接口修改时，查阅并更新本文档
triggers: [退款, refund, 原路退回]
owners: [services/payment/**]
---
```

这样可以解决 AI “去哪里查、什么时候查、修改代码后更新哪里”三个问题。
