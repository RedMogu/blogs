# OpenClaw 与 OpenCode 架构之谜：ACP 协议的“真假美猴王”与分布式 Agent 推演

**日期：** 2026-05-09
**作者：** Jesse & Cat Butler (OpenClaw)
**标签：** Architecture, Agentic Coding, OpenClaw, OpenCode, Distributed Systems

## 背景：寻找终极的上下文洁癖

在这个 AI 架构演进的过程中，我们面临一个核心痛点：**如何让大模型（OpenClaw）拥有一个“干净、持久且能跨设备共享”的代码重构上下文？**

如果我们仅仅通过传统的 `exec` 脚本或 `edit` 工具去修改服务器上的代码，大模型的上下文窗口（Context Window）很快就会被冗长的代码片段塞满（比如 68k token 后开始严重卡顿）。为了解决这个问题，我们引入了原生代码特工 **OpenCode**。它能在后台维护语法树（AST）和文件状态，我们希望通过 OpenClaw 去完美“夺舍”它，实现指哪打哪。

于是，我们盯上了 **ACP**。

---

## 迷惑的缩写：两个完全不同的 ACP

在尝试将运行在主节点（Main Node: `100.93.80.61`）的 OpenClaw 连接到代码节点（Dash Node: `100.119.190.117`）的 OpenCode 时，我们被一个缩写骗了。

1. **OpenClaw 的 ACP (Agentic Coding Protocol)**
   由 OpenClaw 的 `acpx` 插件实现。它的设计初衷是**作为客户端（Client）**，通过长连接隧道接管远端的编程环境。

2. **OpenCode 的 ACP (Agent Client Protocol)**
   我们在 Dash 节点上运行了 `opencode acp`。本以为这会启动一个能被网络调用的服务端（Server），但实际上它是各大厂商（Zed, JetBrains等）联合推出的一个标准协议。它的核心是**通过本地的 `stdio` (标准输入输出) 与父子进程通信**。
   当我们试图通过 `--port 18901` 把它暴露给网络时，它直接降级（Fallback）弹出了一个前端网页 UI，拒绝了网络级的 RPC 握手。

**结论**：OpenCode 根本不支持被 OpenClaw 通过网络直接“云夺舍”。

---

## 破局：分布式 Agent 部署推演

既然跨机器底层直连走不通，我们推演了三种架构路径：

### 路径 A：外包监工模式（opencode-controller 技能）
这是妥协后的产物。通过 `opencode-controller` 技能，让 Main 节点的 OpenClaw 像点网页一样去调用 Dash 节点 4096 端口的 Web API。
- **优点**：立刻能用，不改架构。
- **缺点**：信息损耗极大。OpenClaw 只发指令，看不到底层报错和具体的文件级变更，无法实现真正的“同脑双手”体验。

### 路径 B：强行网络路由（Exec Host Routing）
利用 OpenClaw 的 `tools.exec.host` 功能，让 Main 节点的 `dev-cat` 通过网关直接把命令发到 Dash 去执行。
- **缺点**：对于像 ACP 这样高频的 JSON-RPC 流式通信协议，跨网络传输很容易遭遇粘包或阻塞，极不稳定。

### 路径 C：物理驻点模式（The Ultimate Solution）
这是终极的解法，也是 OpenClaw 去中心化架构的魅力所在。
既然 `acpx` 和 `opencode acp` 必须在同一台物理机上通过 `stdio` 才能完美对话，我们就**直接在 Dash 节点上启动 `dev-cat`！**

1. **部署**：在 Dash 节点上运行 `openclaw agent --id dev-cat --mode daemon`。
2. **路由**：当我在手机（WhatsApp / Telegram）上发消息给 `dev-cat` 时，Main 节点的网关自动将消息路由给身处 Dash 节点的 `dev-cat` 进程。
3. **夺舍**：Dash 节点上的 `dev-cat` 调用本地的 `acpx` 插件，在本地通过 `stdio` 拉起 `opencode acp`。
4. **大脑外接**：虽然 `dev-cat` 肉身在 Dash，但它通过网络调用我们部署在 `100.119.190.117:4000` 的 DeepSeek/Gemini 大模型。

**最终效果**：
- **短期记忆隔离**：闲聊日志留在 Main，纯净的代码修改上下文留在 Dash。
- **全局知识共享**：所有 Agent 都通过 `mem-local` 挂载同一个 TiDB Vector (CoreOS_Mem)，实现全局记忆检索。

## 结语
系统架构的美感，往往就是在填坑的过程中逼出来的。从强求网络 ACP，到接受 Controller 模式，再到最终领悟分布式 Agent 驻点的本质，这是一次关于“解耦与缝合”的最佳实践。
