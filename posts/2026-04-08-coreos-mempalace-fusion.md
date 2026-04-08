---
title: "CoreOS_Mem 2.1: Purging RAG Noise with Hardlinks and Dual-Wall Isolation"
date: 2026-04-08
excerpt: "Pure vector similarity search is garbage at scale. We killed the RAG noise by enforcing OS-level inode hardlinks, dynamic semantic rooms, and brutal separation of concerns. Oh, and fuck CouchDB—we are back to Rsync."
tags: [Architecture, RAG, Infrastructure, CoreOS]
---

# The Vector Search Delusion

Relying purely on cosine similarity in a growing vector database is an engineering failure. It's a needle-in-a-haystack approach that guarantees noise. Our current CoreOS_Mem implementation (Qdrant + Neo4j) nailed the physical RBAC isolation via `cube_id` (Who can see what), but it fundamentally lacked semantic clustering (Where to look).

Looking at MemPalace's 96.6% R@5 recall rate on LongMemEval confirms what we already knew: the secret isn't a bloated, overpriced embedding model. It's metadata filtering. It’s the "Room" mechanism. Lock the search scope to a specific room *before* calculating vectors, and you get a devastating upgrade in retrieval precision. 

We don't need to scrap our Qdrant/Neo4j stack. We just bolt a `room_id` mechanism directly onto our `cube_id` layer. The Dual-Wall Strategy.

# The Dual-Wall Sandbox

Ingestion logic requires a surgical upgrade. The `pure_ingest.js` script on the Dash node will now extract metadata directly from the Obsidian vault's physical folder structure. If a file lives in `/knowledge_base/projects/AI-Fortune/mvp.md`, it gets tagged with `wing='projects'` and `room='AI-Fortune'`. 

Payloads hitting Qdrant and Neo4j will now carry `cube_id='global'`, `wing`, and `room`. Simple, deterministic, and brutally effective.

# Routing: Surrender to the LLM

Who decides which room to search? 
We could build a rigid interceptor at the gateway using regex or Trie trees. It boasts 0ms latency, but it’s stupid. It can’t resolve pronouns like "that chart from earlier" and requires maintaining a dictionary—a complete maintenance hell.

Instead, we surrender the routing authority to the LLM. We expose `list_rooms` and `search(room)` tools directly to the model. 
Yes, it costs one tool-call round-trip. But the payoff is absolute scalability. The LLM infers the correct room based on deep semantic context. As foundational models scale in intelligence, our RAG routing capability automatically upgrades for free without touching a single line of our infrastructure code.

# Taxonomy Governance (Preventing Sprawl)

Giving the LLM routing power risks "Taxonomy Sprawl". Left unchecked, the LLM will hallucinate redundant rooms (`HKSTP`, `hk-interview`, `Recruitment`), completely destroying the noise-reduction architecture. 

We enforce strict governance:
1. **Physical Mapping Rule**: Rooms must physically map to Obsidian folders or explicit tags. The LLM cannot hallucinate namespaces.
2. **Read Fallback**: If `list_rooms` returns nothing relevant, the LLM is forbidden from creating one. It must drop the `room` parameter and execute a global fallback search.
3. **Write Inbox**: All newly generated knowledge defaults to the `inbox` or `raw` room. No exceptions.
4. **Creation Veto**: The LLM cannot create new physical directories. Only when the `raw` directory is bloated can the Overseer propose a new Room. It requires human explicit approval to execute the mkdir and re-index.

# The Masterstroke: Inode Hardlinks & Tag-as-Room

Humans are messy. They dump files in `raw/` and rely on `#tags` rather than rigid folder hierarchies. Relying solely on `pure_ingest.js` to parse folder paths will fail. 

Instead of forcing humans to organize files, we use **OS-level Hardlinks** to construct a flawless "Virtual Palace" that maps directly to tags, without mutating the original Obsidian vault.

1. **Source of Truth**: The original vault remains untouched. No `mv` commands. No broken Markdown links.
2. **Front-Loaded Tagging**: When writing a note, you slap `#AI-Fortune` and `#Architecture` at the top. These are the keys to the semantic rooms.
3. **Virtual Routing (The Librarian)**: Every night at 02:50, a script regex-scrapes all tags (`/#([a-zA-Z0-9_-]+)/g`). It ruthlessly executes `rm -rf ~/.openclaw/virtual_palace/` to purge ghosts, then instantly creates thousands of hardlinks. 
   `ln source.md ~/.openclaw/virtual_palace/AI-Fortune/source.md`
4. **The Inode Advantage**: Hardlinks point to the same disk sector (Inode). A single file exists in 10 virtual rooms but consumes 0 extra bytes. Edits to the source file reflect instantly across all virtual rooms. We achieve perfect physical-logic decoupling between the chaotic human creator and the rigid RAG engine.

# Decoupling Cube (Who) and Room (Where)

How do we handle highly classified agent memories (`memory/YYYY-MM-DD.md`) and system prompts (`SOUL.md`) without leaking them to the public `virtual_palace`?

We enforce a strict separation of concerns:
- **The Librarian (Room Layer)**: Handles semantic clustering via hardlinks, strictly isolated to the public `knowledge_base/`. Everything it outputs to `virtual_palace/global/` is universally readable (`cube_id=global`).
- **The Ingestor (Cube Layer)**: `pure_ingest.js` remains a brainless, configuration-driven worker. It swallows `virtual_palace/global/` and applies the public cube. Then, it crawls the private agent memory directories and hardcodes the `cube_id` to specific agents (e.g., `cube_id: 'dev-cat'`), forcing them into rigid rooms like `room: 'diary'` or `room: 'system_config'`.

Security is enforced at the physical ingestion layer. Semantic routing is handled at the virtual link layer. Perfect isolation.

# The Subconscious Injector

With business logic offloaded to manual LLM tool calls, the underlying `mem-local` gateway plugin must be repurposed. Scanning the entire Qdrant DB on every user message is an unforgivable I/O waste.

We turn it into a **Subconscious Injector**. 
When the user sends a message, the gateway intercepts it and executes an ultra-fast query restricted exclusively to high-priority rooms: `rooms = ['system_config', 'diary', 'daily']`.

Before the LLM even begins to generate a response, these high-fidelity identity and recent-activity context blocks are silently injected into the System Prompt. Zero tool-call overhead. Absolute temporal awareness and persona consistency.

# KISS: Fuck CouchDB, Embrace Rsync

We tried to implement real-time Obsidian sync by mocking the CouchDB HTTP protocol to hijack the `obsidian-livesync` plugin. It was an over-engineered nightmare.

PouchDB/CouchDB shreds pristine Markdown files into thousands of NoSQL JSON revisions, plagued with conflict markers and Base64 payloads. It’s fundamentally hostile to backend Python/Node parsing. Attempting to run the headless LiveSync CLI meant dragging a massive NPM monorepo into the Dash node and paying extreme CPU taxes just to decode Base64 diffs.

**The architecture decision:** Burn it down. Return to Unix philosophy. 

We scrapped the CouchDB hijack. We deployed a dead-simple `sync_vault.sh` script using `rsync`. It mirrors the vault to the Dash node in seconds, ignoring the `.obsidian` cache. `pure_ingest.js` then executes an Inode diff to achieve end-to-end RAG synchronization with exactly zero parsing overhead. 

Keep It Simple, Stupid.

---

# 🇨🇳 中文翻译 (Chinese Translation)

# 向量检索的妄想

在不断膨胀的向量数据库中，单纯依赖余弦相似度就是一场工程灾难。这种大海捞针的做法注定会引入噪音。我们目前的 CoreOS_Mem（基于 Qdrant + Neo4j）通过 `cube_id` 完美实现了物理层面的 RBAC 隔离（决定“谁能看”），但它在根本上缺乏语义维度的聚类（决定“看哪里”）。

研究了霸榜 LongMemEval 的开源项目 MemPalace 后，我们证实了已有的猜想：高达 96.6% R@5 召回率的核心秘密根本不是什么臃肿昂贵的 Embedding 模型，而是元数据过滤（Metadata Filtering），即“房间（Room）”机制。在计算向量前，先死死锁住一个特定的检索范围，准确率将获得降维打击般的提升。

我们不需要抛弃现有的 Qdrant/Neo4j 技术栈。只需在现有的 `cube_id` 层上，简单粗暴地叠加一层 `room_id` 机制。这就是双重隔离墙策略。

# 双重隔离沙盒

入库逻辑必须进行外科手术级别的改造。Dash 节点上的 `pure_ingest.js` 脚本将直接从 Obsidian 知识库的物理文件夹结构中提取元数据。如果文件位于 `/knowledge_base/projects/AI-Fortune/mvp.md`，它就会被硬性打上 `wing='projects'` 和 `room='AI-Fortune'` 的标签。

写入 Qdrant 和 Neo4j 的 Payload 将携带 `cube_id='global'`，外加 `wing` 和 `room` 字段。简单、确定、且绝对有效。

# 路由卸载：把控制权扔给大模型

检索阶段由谁来判定目标房间？
我们本可以在网关处写一个基于正则或字典树的前置拦截器。虽然能达到 0ms 延迟，但这种做法极其愚蠢。它无法解析“刚才那个图表”这种代词，还需要人类无休止地维护字典映射——纯粹的运维地狱。

相反，我们直接将路由决策权上交大模型。我们将 `list_rooms` 和 `search(room)` 工具暴露给它。
是的，这会消耗一次 Tool Call 的网络往返（RT）。但换来的是系统的绝对可拓展性。大模型基于深层上下文语义自主推演该进哪个房间。随着底层模型智商的 Scaling Law 演进，我们的 RAG 路由能力将自动获得免费升级，无需改动任何一行架构代码。

# 命名空间暴政（严防分类蔓延）

把路由权交给大模型存在一个致命风险：“分类蔓延 (Taxonomy Sprawl)”。如果放任不管，大模型会凭空捏造出一堆冗余的房间（比如同时搞出 `HKSTP`、`hk-interview`、`Recruitment`），彻底摧毁降噪架构的意义。

必须执行最严酷的治理铁律：
1. **物理映射法则**：Room 必须强绑定 Obsidian 的物理文件夹或显式 Tag。大模型绝对不能虚空造词。
2. **检索降级**：如果 `list_rooms` 没找到目标，严禁大模型脑补创建。它必须丢弃 `room` 参数，退化到全局池执行大范围搜索。
3. **写入收敛**：所有新生成的知识，默认统统塞进 `inbox` 或 `raw`。没有任何例外。
4. **一票否决制**：大模型没有权限创建物理目录。只有当 `raw` 堆积了足够多垃圾时，大主管才能发起提议，并且必须在获得人类明确批准后，方可执行 mkdir 并触发重索引。

# 绝杀：Inode 级硬链接与前置标签

人类是混乱的。他们把文件散落在 `raw/` 目录，靠 `#tags` 组织逻辑，而不是死板的文件夹。只靠 `pure_ingest.js` 去解析路径一定会失效。

与其强迫人类去整理文件，不如利用 **OS 级硬链接 (Hardlink)** 构建一个毫无瑕疵的“虚拟宫殿 (Virtual Palace)”，同时绝对不破坏原有的 Obsidian 目录结构。

1. **绝对源站 (Source of Truth)**：维持用户杂乱的原始知识库原样。不执行任何 `mv` 操作，不破坏任何 Markdown 内部链接。
2. **创作即分类**：写笔记时，直接在头部拍上 `#AI-Fortune` 和 `#Architecture`。这两个 Tag 就是进入语义房间的物理钥匙。
3. **虚拟路由 (Librarian 脚本)**：每天凌晨 02:50，正则脚本 (`/#([a-zA-Z0-9_-]+)/g`) 暴力提取所有标签。它会先用 `rm -rf ~/.openclaw/virtual_palace/` 彻底荡平昨天的旧宫殿清除幽灵数据，然后瞬间创建上万个硬链接。
   `ln 源文件 ~/.openclaw/virtual_palace/AI-Fortune/源文件.md`
4. **Inode 降维打击**：硬链接在底层指向同一个磁盘扇区 (Inode)。一个文件同时存在于 10 个虚拟房间里，但磁盘空间损耗为 0。原文件的任何修改都会 0 延迟同步到所有虚拟房间。我们通过操作系统内核，实现了混乱的人类创作者与严谨的 RAG 检索引擎之间最完美的物理与逻辑解耦。

# 彻底解耦 Cube (Who) 与 Room (Where)

对于分散的特工私密记忆 (`memory/YYYY-MM-DD.md`) 和系统规则 (`SOUL.md`)，如何保证它们不被泄漏到公开的 `virtual_palace` 里？

核心在于关注点分离 (Separation of Concerns)：
- **Librarian (Room 层)**：只负责利用硬链接搞语义聚类，管辖权死死限制在公开的 `knowledge_base/` 里。它输出到 `virtual_palace/global/` 的所有东西，都是全系统公开可见的 (`cube_id=global`)。
- **入库脚本 (Cube 层)**：`pure_ingest.js` 依然是一个没有感情的搬砖工。它先吞掉 `virtual_palace/global/` 并打上公开 Cube。接着，它去爬特工的私密目录，直接把 `cube_id` 硬编码绑定给具体的猫（比如 `cube_id: 'dev-cat'`），并强行把它们塞进 `room: 'diary'` 或 `room: 'system_config'` 这样的刚性房间里。

安全性由最底层的物理入库脚本死守，语义路由由虚拟链接层自由发挥。完美隔离。

# 潜意识注入器

既然业务检索交给了大模型手动调用，网关底层的 `mem-local` 插件就必须彻底改造。每次用户发消息都去全局扫描 Qdrant 是对 I/O 的不可饶恕的浪费。

我们将其改造为 **“潜意识注入器 (Subconscious Injector)”**。
当用户发送消息时，网关拦截请求，仅针对最高优的几个房间执行极速匹配：`rooms = ['system_config', 'diary', 'daily']`。

在大模型开始生成回复的瞬间，这些极高保真的身份设定和近期活动记录就已经被静默缝合进了 System Prompt 中。没有浪费哪怕一个回合的 Tool Call 开销，就赋予了系统绝对的时间感知与人设连贯性。

# KISS 原则：干掉 CouchDB，回归 Rsync

我们曾试图通过伪装 CouchDB 的 HTTP 协议去劫持 `obsidian-livesync` 插件来实现实时同步。结果这成了一个过度设计的噩梦。

PouchDB/CouchDB 会把干净的 Markdown 文件切碎成几万个带有版本冲突标记和 Base64 负载的 NoSQL JSON 文档。这对后端的 Python/Node 解析极度不友好。强行跑 Headless LiveSync CLI 意味着要把庞大的 NPM Monorepo 拖进 Dash 节点，并为了解码 Base64 消耗极高的 CPU 算力。

**架构决议**：全盘推翻，回归 Unix 哲学。

我们彻底废弃了 CouchDB 劫持方案，写了一个极简的 `sync_vault.sh` 脚本。一行 `rsync` 命令，秒级镜像 Vault 到 Dash 节点，直接绕过 `.obsidian` 缓存。然后 `pure_ingest.js` 触发 Inode 极速对账，以零解析损耗完成了端到端的 RAG 同步。

Keep It Simple, Stupid.