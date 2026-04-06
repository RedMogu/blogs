[🇨🇳 中文版本](#🇨🇳-中文版本-别拿单机玩具忽悠企业agent-的cli-革命是个伪命题) | [🇬🇧 English Version](#🇬🇧-english-version-stop-peddling-single-player-toys-to-enterprises-the-agent-cli-revolution-is-a-joke)

---

# 🇬🇧 English Version: Stop Peddling Single-Player Toys to Enterprises: The Agent "CLI Revolution" is a Joke

**Date**: 2026-04-06
**Author**: Limina Labs (Limina Engineering)

An article hyping "four major platforms opening CLI backdoors for Agents" recently circulated, claiming CLIs from Feishu and DingTalk mark an "execution layer revolution." The premise: point an LLM at a CLI and enterprise automation is solved.

This is marketing garbage. To anyone who has survived a production environment, this design is a toy.

Throwing a native CLI executable at an LLM in a B2B multi-tenant environment does not solve problems. It creates data breaches. Here is why.

## 1. Token Waste and Cascading Failures

Demos show Agents gracefully parsing CLI schemas and executing commands via ReAct. Reality is different: 90% of tokens and latency are wasted on the LLM guessing how to use the CLI. 

Throwing a generalized CLI at an LLM guarantees hallucinated parameters and malformed syntax. This triggers endless `HTTP 400 Bad Request` loops or unintended destructive executions. 

Cascading failures follow. Once an LLM writes hallucinated data back to the system, corruption amplifies across the automation chain. Relying on an open CLI for complex business logic is engineering negligence.

**The Fix:** Production workflows require hardcoded DAGs via LangGraph or rigid SOPs. The LLM only handles unstructured data extraction within isolated nodes. It gets zero control over orchestration.

## 2. Multi-Tenant State Pollution

CLI tools are built for single-player use. Executables like `lark-cli` rely on global configs (`~/.lark-cli-config`) or global environment variables. 

Drop this into a 50-person enterprise chat. User A and User B query separate confidential projects simultaneously. Whose identity does the Agent use?

Using a global Admin Token causes immediate data breaches. Dynamically rewriting environment variables during high concurrency causes race conditions and state pollution. Any Agent toolchain lacking dynamic, per-request multi-tenant auth isolation is a toy.

## 3. Native Tools & Identity Gateways

At Limina Labs, the CoreOS/OpenClaw architecture discards the "LLM calling CLI" anti-pattern. We use Native OAPI Tools behind an Identity & Policy Gateway.

1. **Decoupled Token Management**: Tools are hardcoded at the gateway level. The LLM passes business parameters and never touches API tokens.
2. **Context-Based Routing**: When an LLM triggers an action, the daemon intercepts the `SenderId` from the context. The Auth Store injects the specific user's OAuth Token milliseconds before the HTTP request. Contexts remain physically isolated.
3. **Physical Approval Hooks**: High-risk writes are suspended at the base layer. The gateway pushes a strict Approve/Deny prompt to the user's device, mechanically blocking LLM hallucinations.

## Conclusion

Letting LLMs call CLIs is single-player roleplay. Abandon the obsession with autonomous LLM reasoning. Without hardcoded DAGs and strict multi-tenant auth gateways, execution layer revolutions are illusions. Real enterprise AI relies on strict permission isolation, concurrency control, and rigid state machines.

---

# 🇨🇳 中文版本: 别拿单机玩具忽悠企业：Agent 的“CLI 革命”是个伪命题

**Date**: 2026-04-06
**Author**: Limina Labs (Limina Engineering)

最近炒作《四家平台同时给Agent开了CLI‘后门’》的营销文纯属垃圾。文章吹捧飞书、钉钉的 CLI 是“执行层革命”，以为让大模型敲命令行就能解决企业自动化。

任何在生产环境待过的架构师看这种设计，评价只有一个：玩具。

把原生 CLI 直接扔给大模型，在 ToB 多租户场景下解决不了业务痛点，只会是一颗随时引爆的数据越权定时炸弹。

## 1. Token 浪费与错误复利

Demo 里大模型靠 ReAct 推理玩转 CLI。工程现实是：90% 的 Token 和时间全浪费在让模型现场摸索工具上。

把高度泛化的 CLI 丢给模型，必然遇到参数缺失或语法错误。结果就是无尽的 HTTP 400 死循环，甚至执行破坏性命令。

错误复利更致命。大模型基于幻觉写入错误数据，迅速污染整条自动化链条。指望大模型在开放 CLI 上跑通复杂业务流，是极端的工程渎职。

**现实解法**：生产级工作流必须用 LangGraph 或固定 SOP 锁死为硬编码 DAG。大模型只能在隔离节点做非结构化数据提取，剥夺其底层编排控制权。

## 2. 多租户状态污染

CLI 工具从基因上就是单机版。可执行文件（如 `lark-cli`）依赖全局配置（`~/.lark-cli-config`）或环境变量。

把它扔进 50 人企业群，员工 A 和 B 同时查询涉密项目，Agent 用谁的身份？

用全局 Admin Token，权限瞬间崩溃，直接数据泄露。高并发下动态改写环境变量切 Token，必然引发多线程状态污染（Race Conditions）。没有多租户动态鉴权隔离的 Agent 工具链，全都是玩具。

## 3. 原生工具与鉴权网关

在 CoreOS/OpenClaw 架构中，我们彻底干掉“大模型调 CLI”的业余做法，采用 Native OAPI Tools + 身份与策略网关。

1. **剥离 Token**：操作工具写死在底层网关。大模型只传业务参数，绝对不碰 API Token。
2. **动态身份路由**：网关拦截消息上下文中的 `SenderId`。通过 Auth Store，在发 HTTP 请求前一毫秒动态注入对应用户的 OAuth Token。物理隔离上下文。
3. **物理审批门禁**：所有高危写操作在底层强制挂起，推送到用户的物理设备进行 Approve/Deny 确认。从机制上物理掐断大模型暴走。

## 结语

“大模型调 CLI”只是基础设施残缺时的单机过家家。抛弃对大模型自主推理的迷信。没有硬编码的 DAG 锁死流程，没有多租户动态鉴权网关，执行层革命就是扯淡。企业 AI 的核心是枯燥的权限隔离、并发控制和严密的状态机管理。