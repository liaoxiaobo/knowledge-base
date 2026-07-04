---
title: "MCP协议与MCPServer测试"
tags:
  - 理论
  - AI测试
created: "2026-07-03"
updated: "2026-07-04"
domain: "08-AI与智能化测试"
---

# MCP协议与MCPServer测试

## 一句话本质

MCP（Model Context Protocol）是 Anthropic 提出的开放协议，让 AI 客户端能标准化地发现和调用 MCP Server 提供的工具、资源和提示词。

## 核心内容

1. **协议分层**：客户端、Server、传输层（stdio/sse）。
2. **Server 能力**：Tools、Resources、Prompts 三类能力声明。
3. **发现机制**：客户端通过 initialize 和 tools/list 获取 Server 能力。
4. **测试重点**：协议兼容性、工具调用参数 schema、错误码、传输稳定性。
5. **安全注意**：Server 权限边界、敏感资源访问控制。

## 适用场景

- 
- 
- 

## 典型反模式

- 
- 
- 

## 最小可复现实践

1. 
2. 
3. 

## 行动建议

- **本周可以尝试**：...
- **本月可以输出**：...
- **本季度可以落地**：...

## 关联项目 / 相关文章


## 参考来源

- 
