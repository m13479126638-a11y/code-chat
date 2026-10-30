# Code Chat

本地优先的 AI 对话与编排系统。单文件前端 + 单文件桥接服务，浏览器打开即用。

> **定位：仅供可信环境中的个人开发者使用。** 设计假设为单用户本机场景——进程级隔离而非容器、令牌体系防外不防内。在共享机器/公网暴露/不可信插件环境下使用，超出安全承诺范围。

## 特性

- **四种会话模式**：Chat（纯对话）/ Agent（任务清单驱动）/ Agent-2（单轮 agentic loop，一轮内最多 12 次工具调用，结果不裁剪回传）/ 群聊
- **本地能力桥**：Python/CMD/PowerShell 终端、网页抓取、文件读写、屏幕截图、键鼠控制、执行历史检索——全部经 SecurityGate + AI 审查双层安全（fail-closed）
- **多 AI 成员**：每成员独立服务商/模型/persona/思考模式，随会话保存
- **上下文工程**：窗口轮数保留、自动摘要、工具结果暂存、长会话桥接分片按需加载、导出前自动拉齐
- **EPIP 插件生态**：本地执行插件（Python 沙箱）+ 服务插件（HTTP 端口）。内置 [ai_ork AI 编排端口]——对 Agent 隐藏的服务插件，外部 Web 前端可凭 token 创建/控制会话，为游戏/多 AI 研讨类应用提供基础设施
- **安全模型**：自动生成令牌、审查通道一次性 key、插件独立进程+资源限制、结果带外完整性校验、停止按钮真正终止进程

## 快速开始

```text
1. 运行 local_bridge_server-v26.11.py（控制台显示 token）
2. 浏览器打开 Code-Chat-v26.11.html → 填 127.0.0.1:8765 + token → 握手
3. 左侧「API 设置」配置 LLM（key 只存浏览器）
4. 开聊
```

## 组件

| 文件 | 说明 |
|---|---|
| `Code-Chat-v26.11.html` | 前端单文件（四种模式/编排/渲染/插件菜单/AI 编排面板） |
| `local_bridge_server-v26.11.py` | 桥接单文件（HTTP 服务/终端/安全引擎/插件运行时/存储） |
| `ai_ork.epip` | AI 编排端口服务插件（可选安装） |

## 文档

- [EPIP标准.md](docs/EPIP标准.md) — 插件构建/接入/服务插件规范（v1.1 正式版）
- [安全设置说明.md](docs/安全设置说明.md) — 令牌体系/SecurityGate/审查通道/部署加固清单
- [项目介绍.md](docs/项目介绍.md) — 定位/功能与代码实现详解/发布流程

## 版本与校验

版本号规则：年月式（v26.11 = 2026 年 11 月线），前后端同版本强制配对。

| 发布物 | SHA-256 |
|---|---|
| Code-Chat-v26.11.html | `ccf7b2d7b0ec889eee945de30123bb820a3eae05353342d511c17b6b51104534` |
| local_bridge_server-v26.11.py | `de04b6498fca4646b478698e0b03ab23441a409b4b21c416bdf3a1cf90ce636a` |
| ai_ork.epip | `a881269f08abd42db41f30d6268b950511e3dd45e02be916e41a6f5ae2193d95` |

```powershell
Get-FileHash .\local_bridge_server-v26.11.py -Algorithm SHA256
```

## 许可

[MIT](LICENSE) — Copyright (c) 2026 Code Chat contributors
