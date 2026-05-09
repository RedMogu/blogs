---
title: "Understanding the Agent Client Protocol (ACP) in Distributed OpenClaw Deployments"
date: "2026-05-09"
author: "Jesse"
tags: ["Architecture", "Agent Client Protocol", "OpenClaw", "OpenCode", "Distributed Systems"]
status: "draft"
---

# Understanding the Agent Client Protocol (ACP) in Distributed OpenClaw Deployments

When building distributed AI agent architectures, achieving seamless file-system context and strict memory isolation becomes paramount. In this post, we analyze a common architectural pitfall involving the **Agent Client Protocol (ACP)** and detail how we resolved it by deploying a distributed sub-agent in OpenClaw.

## The Problem: The Need for Seamless Context

In our architecture, the **Main Gateway** (where the orchestrator agent resides) is decoupled from the **Execution Node** (where the code repository and `OpenCode` runner live). 

We wanted the orchestrator to "take over" the remote OpenCode environment to maintain a clean AST/file-system context without blowing up the primary LLM's context window. We attempted to bind OpenClaw's `acpx` plugin (acting as an ACP Client) to a remote OpenCode process over a TCP port (`--port 18901`).

## The Protocol Mismatch: Stdio vs. Network Sockets

The connection immediately failed, returning an HTML Web UI instead of a JSON-RPC upgrade. The root cause lay in a fundamental misunderstanding of the ACP specification.

The **Agent Client Protocol (ACP)** is an open standard designed to decouple IDEs (like Zed, JetBrains, or Neovim) from AI backend runners (like OpenCode or Claude Code). 

By design, the ACP specification relies on **Standard Input/Output (`stdio`)** to transmit JSON-RPC messages. It is built to facilitate parent-child process communication on a local machine, not to be routed across public or private networks via TCP sockets. When we forced the OpenCode ACP server to bind to a network port, it fell back to serving its default Web UI.

- **The Client**: OpenClaw's `acpx` plugin.
- **The Server**: OpenCode (`opencode acp`).
- **The Constraint**: Both must exist on the exact same physical node to establish the `stdio` pipeline.

## The Solution: Distributed Agent Deployment

Because native ACP requires a local process boundary, "remote control" via ACP network tunnels is an anti-pattern. We rejected the compromise of using brittle HTTP "controller" skills (which lose critical execution output) and instead leaned into OpenClaw's distributed node capabilities.

The correct architectural pattern is **Physical Agent Deployment on the Execution Node**:

1. **Deploy the Agent Locally**: We spin up the `dev-cat` daemon directly on the execution node (`100.119.190.117`).
2. **Network Routing**: The Main Gateway (`100.93.80.61`) simply routes the user's chat messages across the Tailscale network to the `dev-cat` process.
3. **Local ACP Hook**: The `dev-cat` agent, now physically residing alongside the code, uses the `acpx` plugin to spawn `opencode acp` as a local child process. The `stdio` JSON-RPC connection succeeds instantly.
4. **LLM Invocation**: The local agent process reaches out to the centralized LLM inference server (e.g., DeepSeek/Gemini via LiteLLM) to perform the reasoning.

### Architectural Benefits

This distributed approach yields three critical advantages:

1. **Short-Term Memory Isolation**: The `dev-cat` agent maintains its own session files (`.jsonl`) locally on the execution node. The developer's idle chatter on the Main Gateway never pollutes the coding agent's context.
2. **Perfect ACP Compliance**: By respecting the `stdio` constraint of the Agent Client Protocol, we avoid network unreliability and brittle Web API wrappers.
3. **Global Knowledge Access**: Both the Main Gateway and the remote `dev-cat` agent share the same TiDB Vector database (`mem-local` plugin) for long-term memory retrieval, ensuring the decentralized agents remain strategically aligned.

## Conclusion

When integrating ACP-compliant tools like OpenCode or Claude Code into an orchestrator framework, respect the protocol's physical boundaries. Do not attempt to pipe `stdio` RPC commands over the network. Instead, push the agent process to the edge where the code lives, and let your central gateway handle the message routing.
