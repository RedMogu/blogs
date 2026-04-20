# OpenLLM 动态降级与意图感知路由策略 (Dynamic Routing Strategy)

**Date**: 2026-04-20
**Tags**: Architecture, OpenLLM, LiteLLM, Cost-Optimization, LLM-Routing
**Status**: Draft (Raw Idea for Review)

## 1. 背景与动机 (Background & Motivation)
在 CoreOS 多区域部署（如 `dash` 与 `kami` 节点）及高并发使用场景中，主节点单一的高端模型（如 `gemini-3.1-pro-preview` 或 `gpt-4o`）极易触发 429 Rate Limit 或产生高昂的 Token 费用。

传统的路由策略“一刀切”地将所有请求发给高端模型，造成了巨大的算力浪费。例如，“1+1等于几”与“重构分布式锁代码”消耗的是同等单价的算力。

**核心顿悟**：我们需要在 OpenLLM (LiteLLM) 层实现**“语义路由 (Semantic Routing)”**或**“上下文感知降级 (Context-Aware Downgrade)”**。

## 2. 为什么“基于上下文结构的启发式降级”最合理？
相较于使用一个独立的小模型作为“分拣员”（会增加至少 0.2s 延迟），利用上下文的结构特征进行启发式拦截更为优雅：

1. **破冰成本高，追问成本低**：新话题冷启动需要大模型（Pro）建立世界观。但在多轮对话后，针对细节的简短追问（如“第二行改一下”），在已有大段上下文的加持下，小模型（Flash）完全可以胜任。
2. **结合 Mem-local 机制**：`mem-local` 在系统层外挂注入了海量背景知识（如 `MEMORY.md`）。到达 LLM 层的 Prompt 已经高度结构化，这极大降低了对模型原生推理能力的硬性要求，使降级变得更加安全可靠。

## 3. 架构设计与伪代码 (Implementation Design)
我们通过 LiteLLM 提供的 `custom_callbacks.py` 钩子 (`async_pre_call_hook`) 实现该层。这是一种“非侵入式扩展”，在不修改底层路由源码的前提下注入 CoreOS 的“灵魂”。

### 拦截判断规则：
*   **规则 A (强制越权)**：若用户输入包含特定触发词（如 `!pro`、`深入`、`仔细`），保持大模型不变。
*   **规则 B (热追问降级)**：若对话轮次 >= 4 且本次输入极其简短（< 50字），判定为已有充分上下文的追问，无条件降级到小模型（如 `gemini-3-flash-preview`）。
*   **规则 C (纯确认词汇)**：若是“好的”、“继续”、“测试”等，直接降级。

### 伪代码实现参考：
```python
import litellm
from litellm.integrations.custom_logger import CustomLogger

class ContextAwareRouter(CustomLogger):
    async def async_pre_call_hook(self, user_api_key, dict_kwargs, **kwargs):
        model = dict_kwargs.get("model", "")
        messages = dict_kwargs.get("messages", [])
        
        # 仅针对昂贵的 Pro 模型考虑降级
        if "pro" not in model.lower():
            return dict_kwargs

        user_message = next((m["content"] for m in reversed(messages) if m["role"] == "user"), "")
        turn_count = len([m for m in messages if m["role"] in ["user", "assistant"]])
        
        should_downgrade = False

        if "!pro" in user_message or "仔细" in user_message:
            should_downgrade = False
        elif turn_count >= 4 and len(user_message) < 50:
            should_downgrade = True
        elif user_message.strip() in ["好的", "继续", "嗯", "可以", "谢谢", "没问题", "测试"]:
            should_downgrade = True

        if should_downgrade:
            dict_kwargs["model"] = "google/gemini-3-flash-preview"
            
        return dict_kwargs
```

## 4. 下一步计划 (Next Steps)
1. Review 此伪代码逻辑的边缘场景（Corner Cases）。
2. 将此脚本挂载到 `dash` 节点或新部署的 `kami` 节点 (AWS Tokyo) 的 LiteLLM 实例中进行灰度测试。
3. 结合 Cloudflare 或多端点配置，完成多区域（Multi-Region）的宏观灾备 (Failover) 部署。