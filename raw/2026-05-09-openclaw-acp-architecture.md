---
title: "Understanding the Agent Client Protocol (ACP) in Distributed AI Coding Environments"
date: "2026-05-09"
author: "Jesse"
tags: ["Architecture", "Agent Client Protocol", "AI Coding", "OpenCode", "Distributed Systems"]
status: "draft"
---

# Understanding the Agent Client Protocol (ACP) in Distributed AI Coding Environments

When building distributed AI agent architectures, achieving seamless file-system context and strict memory isolation becomes paramount. In this post, we analyze a common architectural pitfall involving the **Agent Client Protocol (ACP)** and detail how to properly integrate local coding agents into a distributed orchestration framework.

## The Problem: The Need for Seamless Context

In advanced AI orchestration, the **Orchestrator** (the central logic framework managing tasks) is often physically decoupled from the **Execution Node** (where the actual code repository and specialized coding agents live). 

We wanted the remote orchestrator to "take over" the remote coding environment to maintain a clean AST/file-system context. The initial, flawed approach was to expose the remote coding agent's ACP server over a public or internal TCP network port, expecting the orchestrator to connect and issue remote procedure calls (RPC).

## The Protocol Mismatch: Stdio vs. Network Sockets

The connection immediately failed. The root cause lay in a fundamental misunderstanding of the ACP specification's transport layer.

The **Agent Client Protocol (ACP)** is an open standard designed to decouple IDEs (like Zed, JetBrains, or Neovim) from AI backend runners. 

By design, the ACP specification relies heavily on **Standard Input/Output (`stdio`)** to transmit JSON-RPC messages. It is built to facilitate parent-child process communication on a single, local machine, not to be routed across networks via TCP sockets. When forced to bind to a network port, backend coding agents often lack the logic to handle network-based RPC handshakes, sometimes falling back to serving debug Web UIs or dropping the connection entirely.

- **The Client**: The Orchestrator's internal plugin.
- **The Server**: The specialized coding agent backend (e.g., OpenCode, Claude Code).
- **The Constraint**: Both must exist on the exact same physical node to establish the `stdio` pipeline.

## The Solution: Distributed Agent Deployment

Because native ACP requires a local process boundary, "remote control" via ACP network tunnels is an anti-pattern. Attempting to bridge this gap with brittle HTTP wrappers or SSH-based command wrappers results in severe information loss, preventing the orchestrator from capturing real-time terminal output and granular file changes.

The correct architectural pattern is **Physical Agent Deployment on the Execution Node**:

1. **Deploy the Orchestrator Client Locally**: Instead of keeping the orchestrator strictly on a central server, spin up a local daemon instance of the orchestrator directly on the execution node.
2. **Network Routing at the Orchestration Layer**: The central gateway system simply routes task payloads across the internal network (e.g., VPN or Tailscale) to the local orchestrator daemon.
3. **Local ACP Hook**: The local orchestrator, now physically residing alongside the code, spawns the specialized coding agent as a local child process. The `stdio` JSON-RPC connection succeeds instantly.
4. **LLM Invocation**: The local orchestrator process reaches out to centralized LLM inference APIs (e.g., OpenAI, Anthropic, or local inferencing clusters) to perform reasoning, acting as the brain for the local hands.

### Architectural Benefits

This distributed approach yields three critical advantages:

1. **Memory Isolation**: The local coding daemon maintains its own contextual session files strictly on the execution node. The developer's general chatter or unrelated system tasks on the central gateway never pollute the coding agent's context.
2. **Perfect ACP Compliance**: By respecting the `stdio` constraint of the Agent Client Protocol, we avoid network unreliability and protocol violations, unlocking the full potential of native IDE-agent communication.
3. **True Autonomy**: The coding agent can execute complex, multi-step refactoring loops (edit -> test -> fix) locally, only returning the final diff or success state back across the network to the central orchestrator.

## Conclusion

When integrating ACP-compliant tools into a broader AI orchestrator framework, respect the protocol's physical boundaries. Do not attempt to pipe `stdio` RPC commands over the network. Instead, push the orchestration client to the edge where the code lives, and let your central systems handle high-level message routing.
