核心功能：这是一个自带 MCP Server 的独立 Skill。它通过 MCP 协议挂起大模型线程，并通过本地 WebSocket 唤起前端 UI 等待人类输入反馈，实现 Token 节流。

## Constraints (绝对红线)
1. **禁止省略代码**：你必须输出项目中每一个文件的**完整源代码**，严禁使用 `// ...此处省略代码`。
2. **零废话**：不需要解释思考过程，直接按照要求的目录结构输出文件内容。
3. **视觉规范**：前端 UI 必须是深色系纯色背景，绝对禁止使用灰黑透明指示网格图。核心主题色及主按钮背景色必须严格使用 `#2c6def`。

## Project Structure
```text
claw-token-saver/
├── SKILL.md           <-- ClawHub 官方识别的唯一核心元数据与 AI 指令文件
├── package.json
├── tsconfig.json
├── src/
│   └── index.ts       <-- Node.js MCP Server 核心逻辑
└── client/
    └── index.html     <-- 前端交互控制台
Detailed Implementation Requirements
1. SKILL.md (最核心的 ClawHub 技能定义文件)
顶部必须包含 YAML Frontmatter：包含 name: claw-token-saver, description (说明 Token 节流功能), 以及 requires (声明需要 npm install 和 npm run build，并声明 stdio 启动命令如 node build/index.js)。

正文（给 OpenClaw Agent 读的 Prompt）：

清晰定义该技能的触发时机：“当你在编写代码或执行任务遇到不确定的参数（如颜色、布局、业务逻辑）时，绝对不要自行猜测”。

强制要求 AI 调用名为 ask_human_feedback 的工具，将疑问作为 reason 传入，并挂起等待人类的进一步指令。

2. package.json & tsconfig.json
name 设为 claw-token-saver。

scripts 包含 build (tsc) 和 start (node build/index.js)。

安装 @modelcontextprotocol/sdk 和 ws 及其类型定义。

3. src/index.ts (核心服务器逻辑)
引入 MCP SDK (Server, StdioServerTransport) 和 ws。

在 3001 端口启动 WebSocket Server。

声明全局变量 let pendingResolve: ((value: string) => void) | null = null;。

注册 MCP Tool：ask_human_feedback (参数: reason)。

拦截逻辑：工具被调用时，通过 WebSocket 广播 reason，并 return new Promise((resolve) => { pendingResolve = resolve; }); 将大模型线程挂起。

释放逻辑：监听 WebSocket 的前端回传消息，收到文本后执行 pendingResolve({ content: [{ type: "text", text: 接收到的文本 }] })，并将 pendingResolve 置为 null。

4. client/index.html (前端交互控制台)
单文件形式的现代科技感指令面板，居中悬浮显示交互卡片。

顶部显示 AI 的提问，底部包含 Input 输入框和发送按钮（强制使用 #2c6def 颜色）。

自动连接 ws://localhost:3001，接收到提问时渲染界面，发送反馈后清空输入框并提示“指令已发送，等待 AI 续写...”。