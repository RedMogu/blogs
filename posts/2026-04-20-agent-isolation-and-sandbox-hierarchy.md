---
title: "Zero-Trust AI Subsidiaries: Rigid Hierarchy for Agent Fleets"
date: 2026-04-20
excerpt: "Capability without containment is liability. We enforce mathematical isolation and disposable sandboxes to run subsidiaries with untrusted agent swarms."
tags: ["Architecture", "Zero-Trust", "LLMs", "Distributed Systems"]
---

[🇬🇧 English](#english) | [🇨🇳 中文](#chinese)

**Date:** 2026-04-20
**Author:** Limina Engineering Team

---

<h2 id="english">English</h2>

The industry fixates on single-agent brilliance. For enterprises running subsidiaries, branch offices, and partner networks, capability is irrelevant without isolation. Scale breaks trust.

Deploying an agent to interface with partners hands a black box access to your corporate nervous system. Grant it cloud keys and a "be helpful" directive and you built a liability.

We stress-tested OpenClaw—our multi-node, geographically distributed AI gateway. The goal wasn't task completion. It was validating hard hierarchy for managing fleets of specialized operators across subsidiaries. Result: enterprise AI demands mathematically enforced boundaries, not softer prompts.

## "Smart" ≠ Trusted

Standard enterprise AI: drop a frontier model into Slack/Feishu, wire APIs, let it handle partner requests. Reality: agents in semi-public channels operate in zero-trust environments. Poorly phrased requests from partners or confused subsidiary employees trigger execution. Shell or filesystem access turns misunderstanding into data exfiltration or production damage.

Prompt engineering fails under context decay. Security must be enforced at the architectural layer.

## Dual-Track Hierarchy

We implemented a two-tier authority model: **Orchestrators** vs. **Operators**.

### Orchestrators (Privileged Internal Nodes)
Locked to internal VPNs and specific organizational units.
- **Role**: Resource allocation, high-level task decomposition, sensitive database access (HR, proprietary IP).
- **Access**: Private, tunneled channels (SSH, HQ-locked DM). Never touches partner traffic.
- **Capability**: Non-sandboxed host permissions. DevOps, deployment, resource management. Zero external communication.

### Operators (Public & Partner Operations)
Partner-facing.
- **Role**: Subsidiary interfaces. Feishu/Slack, partner channels, report drafting, inquiry handling.
- **Access**: Untrusted. Container sandbox only. No host visibility.

## Rich Sandboxes, Not Empty Ones

Standard sandboxing neuters AI by stripping runtime tools. Usefulness isn't a model quality issue; it's an environment issue. We replace empty sandboxes with rich-context images.

Custom Docker layers: Python 3.12, Node.js, data analysis libraries, secure API connectors. When an Operator needs to parse a financial report or clean a dataset, the gateway spins up the container, executes, returns output, and kills the instance. Full dev-environment capability. Zero persistence. No visibility into SSH keys or network config.

## Workspace Binding

Sandbox escape or config overwrites are still possible. We add filesystem middleware.

Before any read/write/edit, path resolution verifies the operation is bound to `/workspace`. Attempts to break out are intercepted at the middleware layer before they ever reach the execution environment.

## Conclusion

Subsidiary management via AI is distributed systems engineering. Autonomy requires absolute boundaries.

If your rollout prioritizes "smarter agents" over "containable agents," you are scaling risk. Provide sharp tools. Enforce steel walls. That is the only way to run an AI-driven enterprise.

---

<h2 id="chinese">中文</h2>

行业痴迷于单个智能体的“聪明”。对企业运行子公司、分支机构和合作伙伴网络而言，能力若无隔离则毫无意义。规模会击穿信任。

部署一个智能体来与合作伙伴对接，相当于把进入你公司神经系统的黑盒权限交了出去。给它云密钥和一个“保持帮助”的指令，你就制造了一个负债。

我们压力测试了 OpenClaw——我们的多节点、地理分布的 AI 网关。目标不是完成任务，而是验证用于管理跨子公司专业作业员机队的硬性层级结构。结论：企业级 AI 需要数学手段强制执行的边界，而非更温和的提示词。

## “聪明”并不等同于“可信”

标准的初创企业 AI：把一个前沿模型丢进 Slack/飞书，连接 API，让它处理合作伙伴请求。现实：在半公开频道中的智能体运行在零信任环境中。合作伙伴糟糕的措辞或子公司员工的困惑都会触发执行。Shell 或文件系统权限会让误解变成数据泄露或生产破坏。

提示词工程在上下文衰减下会失效。安全必须在架构层强制执行。

## 双轨层级结构

我们实现了一套双层权限模型：**编排员（Orchestrators）** vs **作业员（Operators）**。

### 编排员（特权内部节点）
锁定在内部 VPN 和特定的组织单元中。
- **角色**：资源分配、高级任务分解、敏感数据库访问（HR、专利 IP）。
- **访问权限**：私有、隧道化通道（SSH、总部锁定 DM）。绝不接触合作伙伴流量。
- **能力**：非沙箱主机权限。DevOps、部署、资源管理。零外部通信。

### 作业员（公众与合作伙伴运营）
面向公众与合作伙伴。
- **角色**：子公司对外接口。飞书/钉钉、合作伙伴频道、报告起草、问询处理。
- **访问权限**：按已失陷处理。仅容器沙箱。无主机可见性。

## 富沙箱，而非废沙箱

传统沙箱通过剥离运行时工具让 AI 失效。代理无用并非模型质量问题，而是环境缺失。我们采用富沙箱镜像替代。

定制 Docker 层：Python 3.12、Node.js、数据分析库、安全 API 连接器。当作业员需要解析财务报告或清洗数据集时，网关启动容器、执行、返回输出、销毁实例。完整开发环境能力。零持久化。SSH 密钥与网络配置不可见。

## 工作区束缚

沙箱逃逸与配置文件覆盖仍可能发生。我们加入文件系统中间件。

任何读写编辑操作前，路径解析验证是否被限制在 `/workspace` 内。尝试越界访问在到达执行环境前即被中间件拦截。

## 结论

通过 AI 管理子公司是分布式系统工程。自主性需要绝对边界。

若你的落地优先级是“更聪明的代理”而非“可被控制的代理”，你正在放大风险。提供锋利工具。强制钢墙。这是运行 AI 驱动企业的唯一方式。
