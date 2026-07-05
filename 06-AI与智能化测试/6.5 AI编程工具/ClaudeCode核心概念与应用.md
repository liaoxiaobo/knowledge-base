---
title: "Claude Code 核心概念与应用"
tags:
  - AI测试
  - Claude Code
  - 实战
created: "2026-07-03"
updated: "2026-07-03"
---

# Claude Code 核心概念与应用

## 简介

Claude Code 是 Anthropic 推出的终端智能编程助手。它通过 **Agentic Loop** 持续收集上下文、执行操作、验证结果，帮助开发者完成代码理解、编辑、运行测试和调试，并可通过 Skills、MCP、Hooks、Subagents 扩展能力。

## 核心功能 / 机制

### 1. Agentic Loop

Claude Code 接收任务后，在三个阶段中循环推进，用户可随时打断并调整方向：

- **收集上下文（Gather Context）**：搜索文件、理解代码结构。
- **执行操作（Take Action）**：编辑文件、运行命令。
- **验证结果（Verify Results）**：运行测试、确认更改。

多个工具调用会被串联成链，Claude 能在过程中自我纠偏。

### 2. 模型与可访问内容

- **模型选择**：Sonnet 适合大多数编程任务，Opus 适合复杂架构决策，可通过 `/model` 切换。
- **可访问内容**：
  - 当前目录及子目录中的项目文件
  - 终端命令行工具
  - Git 状态（分支、未提交更改、近期提交）
  - `CLAUDE.md` 项目级持久化指令
  - 自动记忆（Auto Memory）
  - 扩展配置：MCP 服务器、Skills、Subagents、Chrome 插件

### 3. 扩展机制

在 Agentic Loop 之上，Claude Code 提供四层扩展：

- **Skills**：扩展 Claude 知道的内容，可通过 `/name` 调用。
- **MCP**：连接到外部服务和工具。
- **Hooks**：在工具调用前后自动化工作流。
- **Subagents**：将任务卸载给独立子智能体。

不需要一次性配齐，应按需逐步添加。

### 4. `.claude` 目录结构

Claude Code 从项目目录的 `.claude/` 和用户主目录的 `~/.claude/` 读取配置：

| 文件/目录 | 作用 | 范围 |
| --- | --- | --- |
| `CLAUDE.md` | 每次会话自动加载的指令 | 项目级/全局 |
| `rules/*.md` | 主题范围指令，可按文件路径触发 | 项目级/全局 |
| `settings.json` | 权限、Hooks、环境变量、模型默认值 | 项目级/全局 |
| `skills/` | 可复用技能 | 项目级/全局 |
| `agents/*.md` | 子代理定义 | 项目级/全局 |
| `.mcp.json` | MCP 服务器配置 | 仅项目级 |
| `~/.claude.json` | 应用状态、OAuth、UI 开关、个人 MCP | 仅全局 |

### 5. 上下文窗口管理

会话开始前自动加载：`CLAUDE.md`、自动记忆、MCP 工具名称、Skills 描述。工作过程中每次文件读取、路径范围规则、Hook 触发都会增加上下文。

管理建议：

- `CLAUDE.md` 建议控制在 200 行以内。
- 参考资料移入 Skills 按需加载。
- 不常用的 Skill 设置 `disable-model-invocation: true`。
- 使用 Subagent 处理会产生大量中间输出的任务。

## 适用场景

- 快速理解新项目代码库
- 辅助编写、重构和调试代码
- 运行测试并分析失败原因
- 接入外部工具（Jira、浏览器、数据库等）
- 在终端中完成一次性或持续性的开发任务

## 快速开始 / 常用操作

```bash
# 启动交互式会话
claude

# 一次性任务
claude "帮我修复登录的 bug"

# 续接最近一次会话
claude -c

# 恢复指定会话
claude -r <session-id>

# 非交互模式，适合 CI/脚本
claude -p "summarize these recent commits"

# 查看上下文占用
/context

# 检查加载的记忆文件
/memory

# 切换模型
/model
```

权限模式切换：

```bash
claude --permission-mode default           # 每步确认（默认）
claude --permission-mode acceptEdits       # 文件改动自动通过
claude --permission-mode plan              # 先审计划再执行
claude --permission-mode bypassPermissions # 全自动，仅限容器/虚拟机
```

会话内也可按 `Shift+Tab` 循环切换。

## 配置说明

| 配置项 | 作用 | 示例 |
| --- | --- | --- |
| `CLAUDE.md` | 项目级持久化指令 | 项目规范、命名约定 |
| `settings.json` | 权限、Hooks、环境变量 | `permissions.defaultMode` |
| `.mcp.json` | MCP 服务器配置 | 连接外部工具 |
| `skills/` | 可复用技能 | `/code-review` |

## 注意事项 / 常见误区

- 对话记录和历史以明文存储在 `~/.claude/`，OS 文件权限是唯一保护；避免让 Claude 读取包含凭据的 `.env` 文件。
- `CLAUDE.md` 不要写太长，200 行以内为宜，否则会增加每次会话的上下文负担。
- 不常用的 Skill 应关闭自动触发，避免占用上下文。
- 复杂任务先用 Plan Mode 分析再执行，比直接写代码效果更好。
- 不要一次性配置所有扩展，按需添加并关注上下文占用。

## 相关链接

- [[ClaudeCode命令速查]]

## 参考来源

- [Claude Code 官方文档](https://code.claude.com/docs)
