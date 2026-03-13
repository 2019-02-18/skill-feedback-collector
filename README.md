# Skill Feedback Collector

基于 MCP 协议的人类反馈收集器，通过 WebSocket 连接前端 UI 实现 AI 线程挂起，防止对话过早结束。

专为**腾讯云 / 阿里云 Coding Plan**（按轮次计费）设计，最大化每一轮对话的使用价值。

## 核心原理

```
┌─────────────┐     stdio      ┌──────────────────┐    WebSocket     ┌──────────────┐
│  AI Agent   │◄──────────────►│  MCP Server      │◄───────────────►│  浏览器 UI    │
│  (OpenClaw) │  MCP Protocol  │  (Node.js)       │   Port 18061    │  (index.html) │
└─────────────┘                └──────────────────┘                  └──────────────┘
```

1. AI 完成任务后调用 `ask_human_feedback` 工具，传入工作摘要
2. MCP Server 通过 WebSocket 将问题推送到浏览器前端
3. AI 线程被 **挂起**（Promise pending）—— 等待期间不消耗 Token
4. 用户在浏览器中阅读问题并输入反馈
5. 反馈通过 WebSocket 返回 → MCP Server 释放 Promise → AI 继续工作

## 快速开始

### 安装

```bash
git clone git@github.com:2019-02-18/skill-feedback-collector.git
cd skill-feedback-collector
npm install
npm run build
```

### 配置 MCP

在你的 AI 客户端（Cursor / OpenClaw 等）的 MCP 配置中添加：

```json
{
  "mcpServers": {
    "feedback-collector": {
      "command": "node",
      "args": ["build/index.js"],
      "cwd": "/path/to/skill-feedback-collector"
    }
  }
}
```

### 访问前端

服务启动后，在浏览器中打开：

```
http://你的服务器IP:18061
```

## 功能特性

| 特性 | 说明 |
|------|------|
| MCP 线程挂起 | 通过 Promise 挂起 AI 线程，等待期间零 Token 消耗 |
| WebSocket 实时通信 | 问题推送与反馈回传全部通过 WebSocket 实时完成 |
| 反馈模式开关 | 支持通过 UI 或 MCP 工具随时开关反馈确认模式 |
| 对话历史持久化 | 自动保存到 `feedback-history.json`，最多 500 条 |
| 快捷回复 | 内置"继续"、"好的"、"重做"、"结束"快捷按钮 |
| 自动重连 | 断线后 3 秒自动重连，不会丢失待处理的问题 |
| 服务器部署 | 绑定 `0.0.0.0`，支持外网浏览器访问 |
| 历史记录 API | `/api/history` 返回最近 100 条对话记录 |
| 健康检查 | `/health` 返回服务状态、连接数、待处理请求 |

## 环境变量

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `FEEDBACK_PORT` | `18061` | HTTP + WebSocket 服务端口 |

```bash
# 自定义端口
FEEDBACK_PORT=9090 node build/index.js
```

## 项目结构

```
skill-feedback-collector/
├── SKILL.md              # OpenClaw/ClawHub 技能定义文件
├── package.json           # 依赖管理
├── tsconfig.json          # TypeScript 配置
├── src/
│   └── index.ts           # MCP Server 核心逻辑
├── client/
│   └── index.html         # 前端交互面板
└── feedback-history.json  # 对话历史（运行后自动生成，已 gitignore）
```

## MCP 工具

### `ask_human_feedback`

挂起 AI 线程，等待人类输入。反馈模式关闭时直接返回不挂起。

**参数：**
- `reason`（string，必需）：工作摘要和需要用户确认的内容

**返回：** 用户输入的文本（或关闭模式下的 bypass 消息）

**使用场景：**

```
✅ 任务完成后 → 询问用户是否继续
❓ 遇到不确定参数 → 请用户决定
🔧 修复错误后 → 请用户验证
🚫 即将结束对话 → 确认用户同意
```

### `set_feedback_mode`

开关反馈确认模式。

**参数：**
- `enabled`（boolean，必需）：`true` 开启，`false` 关闭

**使用场景：**

```
用户说"自由模式" → 调用 set_feedback_mode(enabled: false)
用户说"确认模式" → 调用 set_feedback_mode(enabled: true)
也可以直接在浏览器 UI 上切换开关
```

## 对话流程示例

```
AI: [完成任务] → 调用 ask_human_feedback("✅ 登录 API 已完成，需要继续吗？")
    ⏸️ AI 线程挂起，不消耗 Token
用户: [在浏览器中输入] → "继续，帮我加上注册接口"
AI: [收到反馈] → 开始编写注册接口
AI: [完成任务] → 调用 ask_human_feedback("✅ 注册接口完成，还需要什么？")
    ⏸️ 再次挂起
用户: "没有了，结束吧"
AI: [收到反馈] → 结束对话
```

## 适用场景

- **Coding Plan** — 按轮次计费，一轮对话内完成更多任务,避免不必要的对话轮次消耗
- **任何需要人工确认的 AI 工作流** — 防止 AI 猜测导致返工
- **OpenClaw Skill 生态** — 符合 ClawHub 标准，可发布分享

## 技术栈

- **TypeScript** + **Node.js**
- **@modelcontextprotocol/sdk** — MCP 协议 SDK
- **ws** — WebSocket 实现
- **原生 HTTP** — 静态文件服务 + REST API

## 许可证

MIT
