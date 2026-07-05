# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 提供在本仓库中工作的指导。

## 仓库类型

本仓库是一个 **Markdown 知识库**（Obsidian Vault），聚焦于测试开发、质量保障与 AI 辅助测试领域。内容包括结构化笔记、项目实战、模板与参考资料。它不是软件项目，因此不存在构建、测试、Lint 或依赖管理命令。

## 目录结构

知识库采用 **7 大领域 + 元文件夹** 结构：

- `00-inbox/` — 临时收集箱，存放待整理的碎片素材。
- `01-测试基础与方法论` — 测试理论、方法论与质量观。
- `02-测试开发基本技能` — Python、Linux、网络、Git、数据库等通用技能。
- `03-自动化测试工程` — 自动化框架、UI/接口自动化、CI/CD。
- `04-云原生与基础设施` — Kubernetes、容器、中间件、环境管理。
- `05-专项测试技术` — 性能、稳定性、混沌工程、安全、兼容性测试。
- `06-AI与智能化测试` — AI 基础、AI 辅助测试、AI 应用测试、AI 安全测试。
- `07-项目案例/` — 按实际项目名称组织的项目交付物。
- `_templates/` — 新建笔记的 Markdown 模板。
- `assets/` — 图片、截图、PDF 等附件。

## 命名规范

- **顶层目录**：`[编号]-主题名`，如 `03-自动化测试工程`、`06-AI与智能化测试`。
- **项目案例子目录**（位于 `07-项目案例/` 下）：全部小写，将空格和 `/` 替换为 `-`，去掉其余特殊符号。例如 `sugoncloud/playwright-sugon` → `sugoncloud-playwright-sugon`。
- **文件名**：`主题名.md`，不嵌入编号。例如 `AI智能失败归因 Skill 实战.md`。
- **页面标题**：与文件名保持一致，正文使用 `# 主题名`。
- **图片命名**：`领域-主题-序号.png`，例如 `2.2.4-nfs-01.png`。

## Frontmatter

每篇文档必须包含 `title`、`tags`、`created`、`updated`：

```markdown
---
title: "文章标题"
tags:
  - 文档类型标签
  - 技术栈或场景标签
created: "YYYY-MM-DD"
updated: "YYYY-MM-DD"
---
```

- `tags` 建议 3–5 个：1 个文档类型标签（`理论`、`实战`、`项目`、`复盘`、`规范`、`速查`），1–2 个主题/技术栈标签，以及可选的 1 个场景标签。
- `created` 与 `updated` 使用 `YYYY-MM-DD` 格式。

## 模板选择

根据文档类型从 `_templates/` 中选择模板：

- `通用知识笔记模板.md` — 方法论、原则、最佳实践、能力模型、通用技术知识。
- `项目实战文档模板.md` — 具体项目测试实践或简历素材，必须回答简历四问。
- `工具使用文档模板.md` — 工具安装、配置与使用。
- `技术方案设计文档模板.md` — 架构设计或技术方案类文档。

## 新增与编辑内容

- 当用户想把零散素材整理成结构化知识库文章时，使用自定义 Claude Code Skill `.claude/skills/knowledge-builder/SKILL.md`。
- 通用可复用知识放入 `01-05` 或 `06`；绑定具体项目的内容放入 `07-项目案例/<project-dir>/`。
- 使用 `[[文件名]]` 建立双向链接时，只链接真实存在的文件，禁止编造不存在的链接。
- 附件统一存放于 `assets/`。

## Git 约定

- 不要自动提交，提交时机由用户掌控。
- 近期提交前缀示例：`feat(08):`、`refactor(02):`、`docs:`，后接简短中文描述。括号中的域编号对应顶层目录编号（历史上 `08` 对应 `06-AI与智能化测试` 域）。
- `.gitignore` 已排除 Obsidian 个人状态文件（`.obsidian/workspace.json`、`.obsidian/graph.json`）、`.smart-env/`、`.DS_Store` 与 `.idea/`。

## 工具链

- 知识库主要在 **Obsidian** 中编辑，启用了 **Smart Connections** 与 **Copilot** 社区插件。
- `.obsidian/app.json` 配置：`attachmentFolderPath` 为 `assets`，`templates-folder` 为 `_templates`，`alwaysUpdateLinks` 为 `true`。
- `.claude/settings.local.json` 包含针对常见重构操作的权限白名单。
