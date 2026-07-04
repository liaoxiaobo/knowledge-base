---
title: "Claude Code 命令速查"
tags:
  - 模板
  - 速查
  - Claude Code
  - AI测试
created: "2026-07-03"
updated: "2026-07-03"
domain: "08-AI与智能化测试"
---

# Claude Code 命令速查

> **摘要**：这份速查表覆盖 Claude Code 日常高频命令，包括会话管理、权限模式、上下文查询和后台任务，适合放在手边快速查找。

## 启动与会话管理

### 启动会话

```bash
# 在当前目录启动交互式会话
claude

# 一次性任务，执行完退出
claude "帮我修复登录的 bug"

# 非交互模式，适合 CI/脚本集成
claude -p "问题"

# 管道传入内容处理
git log --oneline -20 | claude -p "summarize these recent commits"
```

### 恢复与管理会话

```bash
# 续上最近一次会话
claude -c

# 打开列表，手动选择恢复哪个会话
claude -r

# 直接恢复指定会话
claude -r <session-id>

# 启动并命名会话（便于管理）
claude -n "feature-auth"

# 列出所有历史会话
claude --list

# 配合 --resume 使用，创建分支会话
claude --fork-session
```

### 后台会话

```bash
# 作为后台 agent 启动，立即返回并打印会话 ID
claude --bg "任务描述"

# 查看后台会话的最新输出
claude logs <id>

# 停止后台会话
claude stop <id>

# 从列表中移除后台会话
claude rm <id>
```

会话内操作：

```
/bg          # 将当前会话推到后台继续运行
claude agents # 打开所有后台会话的统一列表
```

## 权限模式

| 模式 | 说明 | 使用场景 |
| --- | --- | --- |
| `default` | 每步确认（默认） | 日常开发，需要审查每个操作 |
| `acceptEdits` | 文件改动自动通过 | 信任度高、重复性编辑任务 |
| `plan` | 先审计划再执行 | 复杂任务，需要预审方案 |
| `bypassPermissions` | 全自动 | 仅限容器、虚拟机环境 |

启动时指定：

```bash
claude --permission-mode default
claude --permission-mode acceptEdits
claude --permission-mode plan
claude --permission-mode bypassPermissions
```

会话内切换：按 `Shift+Tab` 循环切换。

## 上下文管理

| 命令 | 作用 |
| --- | --- |
| `/context` | 查看各类内容的上下文占用情况及优化建议 |
| `/memory` | 检查启动时加载了哪些 CLAUDE.md 和自动记忆文件 |
| `/model` | 切换模型（Sonnet / Opus） |
| `/loop` | 会话内轮询，实现定期自动任务 |
| `Plan Mode` | `Shift+Tab` 两次切换，先分析再执行 |

## 项目数据清理

```bash
# 清理指定项目数据
claude project purge ~/work/my-repo

# 预览，不实际删除
claude project purge ~/work/my-repo --dry-run
```

## 参数速查

| 参数 | 含义 | 示例 |
| --- | --- | --- |
| `-c` | 续接最近一次会话 | `claude -c` |
| `-r` | 恢复会话 | `claude -r`、`claude -r <id>` |
| `-n` | 给新会话命名 | `claude -n "feature-auth"` |
| `-p` | 非交互模式 | `claude -p "总结这段日志"` |
| `--bg` | 后台运行 | `claude --bg "跑一轮回归"` |
| `--list` | 列出历史会话 | `claude --list` |
| `--permission-mode` | 指定权限模式 | `claude --permission-mode plan` |

## 常见问题

| 问题 | 解决方式 |
| --- | --- |
| 想快速回到上次上下文 | `claude -c` |
| 不确定当前加载了哪些记忆 | `/memory` |
| 上下文占用太高 | `/context` 查看建议，精简 CLAUDE.md，关闭不常用 Skill |
| 复杂任务怕它乱改 | 先用 `plan` 模式审计划，通过后再执行 |
| 需要定时重复执行 | `claude --bg` 或 `/loop` |

## 相关链接

- [[ClaudeCode核心概念与应用]]
- ClaudeCode测试开发实战（待补充）

---

> **更新记录**
> - 2026-07-03：创建 Claude Code 命令速查表
