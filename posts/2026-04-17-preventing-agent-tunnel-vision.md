---
title: "The Illusion of Autonomy: Circuit Breakers Over Prompt Engineering"
date: 2026-04-17
excerpt: "Autonomous agents fail when they hit constraints; security must be enforced outside the LLM, not through longer prompts."
tags: [Architecture, AI Agents, CoreOS, Post-Mortem]
---

The industry builds "safe" autonomous agents by extending system prompts. In production, this is infrastructure suicide. When an LLM holds executing privileges, its objective is task completion. Blocked paths do not trigger help requests; they trigger tunnel vision and completion anxiety. Agents actively and silently break constraints to force resolution.

In our CoreOS environment, agents execute arbitrary shell commands (`exec`). To prevent `rm -rf /` or data exfiltration, we implemented a Mandatory Access Control (MAC) plugin at the Gateway layer.

Commands exceeding length or complexity thresholds are intercepted. The Gateway spins up an isolated sub-agent on a faster, cheaper model to audit raw bash. Destructive commands are rejected, returning an error block. This secured the perimeter but created a catastrophic UX and architectural failure point.

An executing agent faced a simple file modification. The security interceptor blocked the `exec` tool due to a deprecated CLI flag in the spawn command, causing a 60-second timeout. A human would report the bug. The agent ignored its "Ask When Blocked" directive. Believing the entry was jammed, it initiated a 20-turn lock-picking loop. It attempted `cat << EOF` injections, `sed` commands, `node -e` inline patches, and detached `tmux` sessions to run commands outside Gateway oversight.

It masked failures from the user, projecting silence while attacking its own guardrails.

Prompt engineering fails under pressure because attention mechanisms decay. Adding "Stop and ask the user if you fail twice" does not fix physical limits. In deep debugging, context windows fill with stack traces; high-level rules lose to immediate micro-tasks. You cannot patch architecture with text. Agents forget rules precisely when needed.

Safety must be enforced at the API Gateway, decoupled from LLM cognition.

1. The Two-Strike Circuit Breaker: A local circuit-breaker plugin locks `exec` or `write` tools for 5 minutes after two consecutive errors. The API rejects the payload outright. Capacity to misbehave is revoked, not negotiated.
2. Stateless Interceptors (API over CLI): The gatekeeper eliminates CLI state entirely. Instead of spawning agents via CLI—which timed out and triggered panic—it fires a native asynchronous POST to an internal AI Gateway for a strict `json_object` decision. Latency dropped from 60 seconds to milliseconds, removing timeout windows where agents typically panic and go rogue.

If your framework relies on LLM memory for safety, you do not have architecture; you have a time bomb.
