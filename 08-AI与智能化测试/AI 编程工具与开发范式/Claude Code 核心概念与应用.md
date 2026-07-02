---
title: Claude Code 核心概念与应用
tags:
  - AI测试
  - 理论
  - 实战
created: 2026-07-03
updated: 2026-07-03
domain: 08-AI与智能化测试
---

# Claude Code 核心概念与应用

## 一句话本质

Claude Code = 运行在终端、基于 **Agentic Loop（收集上下文 → 执行操作 → 验证结果）** 的智能编程助手，通过自然语言对话完成代码理解、编辑、测试和调试，并可通过 Skills、MCP、Hooks、Subagents 扩展能力边界。

## 核心内容

### 1. Claude Code 基于 Agentic Loop 工作

Claude Code 接收任务后，在三个相互交织的阶段中循环推进，用户可随时打断并调整方向：

- **收集上下文（Gather Context）**：搜索文件、理解代码结构、读取相关资源。
- **执行操作（Take Action）**：编辑文件、运行命令、调用工具。
- **验证结果（Verify Results）**：运行测试、检查输出、确认更改。

多个工具调用会被串联成链，Claude 能在过程中自我纠偏。

### 2. 模型、工具与可访问范围

- **模型选择**：Sonnet 适合大多数编程任务，Opus 适合复杂架构决策，可通过 `/model` 切换。
- **可访问内容**：
  - 当前目录及子目录中的项目文件
  - 终端和命令行工具
  - Git 状态（分支、未提交更改、近期提交）
  - `CLAUDE.md` 项目级持久化指令
  - 自动记忆（Auto Memory）
  - 扩展配置：MCP 服务器、Skills、Subagents、Chrome 插件

### 3. 高效使用 Claude Code 的关键技巧

- **像对话一样工作**：不需要完美的提示词，直接表达意图，不对就迭代修正。
- **前期说清楚**：初始提示越精确，后续修正越少。引用具体文件、说明约束条件、给出示例模式。
- **给可验证的目标**：提供测试用例、截图或预期输出，Claude 能自我验证时效果最好。
- **委托而非指挥**：给出背景和方向，信任 Claude 决定细节，不需要指定读哪个文件或运行什么命令。
- **复杂任务先探索再实现**：使用 Plan Mode（Shift+Tab 两次切换）先分析代码库、制定计划，审阅通过后再执行。

### 4. 扩展机制让 Claude Code 适应更多场景

在 Agentic Loop 之上，Claude Code 提供四层扩展：

| 扩展 | 作用 |
| --- | --- |
| **Skills** | 扩展 Claude 知道的内容，可通过 `/name` 调用 |
| **MCP** | 连接外部服务和工具 |
| **Hooks** | 在工具调用前后自动化工作流 |
| **Subagents** | 将任务卸载给独立子智能体 |

不需要一次性配齐所有扩展，应按需逐步添加，并关注上下文占用成本。

### 5. `.claude` 目录是项目级配置的载体

Claude Code 从项目目录的 `.claude/` 和用户主目录的 `~/.claude/` 读取配置：

| 文件 | 作用 | 范围 |
| --- | --- | --- |
| `CLAUDE.md` | 每次会话自动加载的指令 | 项目级/全局 |
| `rules/*.md` | 按文件路径触发的主题范围指令 | 项目级/全局 |
| `settings.json` | 权限、Hooks、环境变量、模型默认值 | 项目级/全局 |
| `skills/` | 可复用技能 | 项目级/全局 |
| `agents/*.md` | 子代理定义 | 项目级/全局 |
| `.mcp.json` | MCP 服务器配置 | 仅项目级 |
| `~/.claude.json` | 应用状态、OAuth、UI 开关 | 仅全局 |

**安全提示**：Claude Code 运行时会自动写入对话记录等数据到 `~/.claude/`，以明文存储，OS 文件权限是唯一保护。若读取了 `.env` 或命令输出凭据，相关信息会被记录。可用 `claude project purge` 清理指定项目数据。

### 6. 上下文窗口需要主动管理

Claude Code 的上下文窗口按时间序列加载：

- **会话开始前自动加载**：`CLAUDE.md`、自动记忆、MCP 工具名称、Skills 描述。
- **工作过程中增加**：每次文件读取、路径范围规则、Hook 触发输出。

管理建议：

- `CLAUDE.md` 建议控制在 200 行以内，参考资料移入 Skills 按需加载。
- 运行 `/context` 查看上下文占用及优化建议。
- 运行 `/memory` 检查启动时加载了哪些文件。
- 对不希望自动触发的 Skill，设置 `disable-model-invocation: true`。
- 使用 Subagent 处理会产生大量中间输出的任务，只让摘要返回主上下文。

## 适用场景

- 系统学习 Claude Code 的基本概念和使用方式
- 在团队内推广 Claude Code，建立共同认知
- 设计项目级 Claude Code 使用规范和 CLAUDE.md
- 优化个人或团队的 AI 辅助开发效率
- 面试准备（AI 编程工具方向）

## 典型反模式

- 把 Claude Code 当搜索引擎用，只问不执行
- 给模糊目标后频繁纠偏，浪费上下文和 Token
- 不信任 Claude，过度 micromanage，指定每一步操作
- 忽视 `CLAUDE.md`，把项目规范全部放在对话里
- 让单次会话无限增长，不关注上下文上限
- 在不受控环境使用 `bypassPermissions` 全自动模式
- 一次性配置所有扩展，导致上下文占用过高

## 最小可复现实践

1. **创建项目级 `CLAUDE.md`**：写入项目定位、命名规范、常用命令和需要避免的操作。
2. **用 Plan Mode 处理复杂任务**：先让 Claude 分析并输出计划，审阅后再执行。
3. **定期用 `/context` 和 `/memory` 检查上下文**：发现并清理不必要的自动加载内容。

## 行动建议

- **本周可以尝试**：参考 [[Claude Code 命令速查]]，整理一份个人常用的 Prompt 模式清单。
- **本月可以输出**：为所在项目写一份 Claude Code 使用规范或 `CLAUDE.md` 初稿。
- **本季度可以落地**：在测试项目中引入 Claude Code 辅助自动化用例开发或失败分析的具体流程。

## 关联项目 / 相关文章

- [[Claude Code 命令速查]]
- [[Claude Code 在测试开发中的应用]]（待补充）
- [[测试工作的核心原则]]

## 参考来源

- [Claude Code 官方文档](https://code.claude.com/docs)
