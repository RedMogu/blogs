[🇨🇳 中文版本](#🇨🇳-中文版本-limina-labs-工程博客) | [🇬🇧 English Version](#🇬🇧-english-version-limina-labs-engineering-blogs)

---

# 🇬🇧 English Version: Limina Labs Engineering Blogs

Welcome to the **Limina Labs (RedMogu Engineering)** blog repository. 

This is where we cut through the AI industry's hype and share hardcore, battle-tested engineering architectures. We believe in **Production Reality over Academic Elegance**.

## 🧠 Our Philosophy

The current AI landscape is flooded with "toy" paradigms born from academic researchers or indie hackers lacking enterprise production experience. 

Here, we debunk the hype and establish true engineering thought leadership:
- **LangGraph & SOPs > Autonomous LLM Reasoning**: We don't trust LLMs to "figure out" workflows on the fly. We use deterministic DAGs and strict Standard Operating Procedures (SOPs) to lock down the execution backbone, using LLMs only for unstructured data extraction.
- **JIT Context & Async Dual-Core > O(N²) LLM-Wikis**: We reject the naive, token-burning idea of letting LLMs autonomously rewrite global knowledge bases. We advocate for Just-In-Time (JIT) retrieval for cold data, and Async Dual-Core pipelines (cheap models for fetching, expensive models for on-demand synthesis) for knowledge curation.
- **Identity Gateways > CLI Wrappers**: We expose the "CLI Agent Revolution" for what it is—a single-player toy. True enterprise automation requires multi-tenant, dynamic identity mapping, and robust Approval Hooks at the gateway level.

## 📂 Repository Structure

Our publishing process is governed by a strict workflow to ensure content remains authentic and devoid of generic "AI buzzwords".

- `/raw/`: Initial brain-dumps, architectural outlines, and AI-assisted drafts. Content here is raw material.
- `/posts/`: (Coming soon) Finalized articles. Before moving here, `raw` drafts must pass through our **"De-AI" (去AI化)** LangGraph workflow to strip out formulaic LLM writing styles, ensuring the content reads like it was written by an authentic, highly opinionated engineering leader.

## ✍️ Authors
**Limina Engineering Team**
*Debunking hype-driven AI trends, one architecture at a time.*

---

# 🇨🇳 中文版本: Limina Labs 工程博客

欢迎来到 **Limina Labs (RedMogu Engineering)** 博客仓库。

在这里，我们拨开 AI 行业的各种炒作迷雾，分享硬核、经过实战检验的工程架构。我们坚信 **“生产现实大于学术优雅” (Production Reality over Academic Elegance)**。

## 🧠 我们的工程哲学

目前的 AI 圈充斥着各种缺乏企业级生产经验的学术研究者或独立开发者所创造的“玩具”范式。

在这里，我们致力于打破这些炒作，建立真正的工程思想领导力：
- **LangGraph & SOPs > LLM 自由发散**: 我们从不迷信让大模型在运行中“自主涌现”出工作流。我们使用确定性的 DAG（有向无环图）和严格的 SOP（标准作业程序）来锁死执行骨架，仅在局部依靠 LLM 提取非结构化数据。
- **JIT 检索 & 异步双核 > O(N²) 的全自动 LLM-Wiki**: 我们极度排斥让大模型全自动全局重写知识库这种极其消耗 Token 且容易导致级联错误（Cascading Failures）的伪需求。我们主张对冷数据进行 JIT（Just-In-Time）即时检索，利用异步双核流（廉价模型做脏活抓取，昂贵模型做按需整合）来进行知识管理。
- **多租户身份网关 > CLI 单机玩具**: 我们撕下所谓“CLI Agent 革命”的伪装——这不过是个单机玩具。真正的企业级自动化，需要的是在网关层实现多租户隔离、动态身份映射以及坚如磐石的审批门禁（Approval Hooks）。

## 📂 仓库结构

我们的发布流程由严格的 Workflow 驱动，以确保内容纯粹、硬核，且绝无大模型生成的“废话（AI buzzwords）”。

- `/raw/`: 原始大脑倾印（Brain-dumps）、架构草案和 AI 辅助的初稿。这里的只是未经雕琢的“原石”。
- `/posts/`: （即将上线）定稿发布区。在进入此目录前，`raw` 目录下的草稿必须经过我们特有的 **"De-AI" (去AI化)** LangGraph 节点清洗，以剥离刻板的大模型行文风格，确保最终输出读起来就像是一位极其资深、且极具观点态度的工程负责人的亲笔手书。

## ✍️ 作者
**Limina Engineering Team**
*一次一个架构，戳破 AI 炒作的泡沫。*