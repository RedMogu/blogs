---
title: CoreOS_Mem 架构升级：融合 MemPalace 的动态房间机制
date: 2026-04-08
tags: [Architecture, RAG, Engineering]
---

# CoreOS_Mem 架构升级草案：融合 MemPalace 的“动态房间”机制
**日期**: 2026-04-08
**作者**: 大主管 (Main Cat) & 主人 (Shengren)
**状态**: 架构共识已定稿，包含终极 Tag-as-Room 与 Hardlink 机制

## 1. 现状痛点与启发
目前 CoreOS_Mem（基于 Qdrant + Neo4j）实现了底层的双轨制记忆，且通过 `cube_id` 实现了完美的 **“权限物理隔离”**。
但是，我们缺乏 **“语义维度的聚类隔离”**。

在查阅了霸榜 LongMemEval 的开源项目 `mempalace` 后，我们发现其 96.6% R@5 召回率的核心秘密并非复杂的 Embedding 模型，而是前置的 **Metadata Filtering（元数据过滤）**——也就是“记忆宫殿（Wing -> Room）”隐喻。

- **咱们的 Cube**：解决了“谁能看”（Who）。
- **MemPalace 的 Room**：解决了“看哪里”（Where）。

当向量数据库里的知识越来越多时，仅靠 Cosine Similarity（余弦相似度）去大海捞针必定会产生噪音。如果能在检索前，先死死锁住一个“Room”，准确率将获得降维打击般的提升。

## 2. 融合方案：双重隔离墙 (The Dual-Wall Strategy)

我们不需要抛弃现有的 Qdrant/Neo4j 栈，只需在现有的 `cube_id` 机制上，叠加一层 `room_id` 机制。

### 阶段一：入库改造 (Ingestion Upgrades)
修改 Dash 节点上的 `pure_ingest.js`：
1. **自动标签化**：利用 Obsidian 知识库天然的文件夹结构。
   - 规则示例：如果文件路径为 `/knowledge_base/projects/AI-Fortune/mvp.md`，则提取其层级，打上元数据 `wing='projects'`, `room='AI-Fortune'`。
2. **Payload 结构升级**：
   在写入 Qdrant 和 Neo4j 时，除了 `cube_id='global'`，新增 `wing` 和 `room` 字段。

## 3. “手动挡”与“自动挡”的博弈（路由决策）
在检索阶段，系统如何判定目标房间（Room）？
- **自动挡 (网关前置拦截)**：利用正则或字典树进行关键词命中。优点是 0ms 延迟，不消耗 Token；缺点是极度僵化，无法理解“刚才那个图表”这种代词指代，也无法理解深层语义。
- **手动挡 (大模型自主工具调用，即 MemPalace 模式)**：将 `list_rooms` 和 `search(room)` 工具暴露给大模型。
  - **优点**：“大模型越聪明，做得越好”。模型可以结合上下文语义自主决定该进哪个房间找资料，且逻辑推演方式与代码执行保持一致，未来模型升级后系统能力自动跃升。
  - **权衡**：需要消耗一次工具调用的 Round-Trip (RT)。

**决议：上交路由权**。在当前模型智商已足够高且延迟可接受的前提下，**将房间的路由选择权交给大模型** 是更具拓展性的架构选择。这不仅避免了硬编码的字典维护地狱，也能让系统的 RAG 表现直接享受到模型自身 Scaling Law 带来的红利。

## 4. 房间的治理与防扩散机制 (Taxonomy Governance)
将选择权交给大模型后，必须严防“分类蔓延 (Taxonomy Sprawl)”。大模型绝对不能在找不到房间时“随手瞎建”一个（例如同时存在 `HKSTP`、`hk-interview`、`Recruitment` 三个意思相同的房间，这会彻底毁掉聚类降噪的意义）。

**治理铁律：**
1. **物理映射法则**：在我们的系统中，Room 必须强绑定 Obsidian 的物理文件夹或显式 Tag。大模型不能虚空造词，建房间等同于建文件夹或打标签。
2. **检索降级 (Read Fallback)**：当大模型 `list_rooms` 发现没有对应房间时，**严禁脑补创建**。它应该主动摘掉 `room` 参数，退化为在 `global` 池子里进行大范围全局搜索。
3. **写入收敛 (Write Inbox)**：如果有新领域的知识产出，默认统统塞进 `inbox` 或 `raw` 房间。
4. **建房审批制 (Creation Veto)**：只有当某个话题在 `raw` 里堆积了足够多的文件，或者主人明确下令时，大主管才能提议：“关于 XXX 的文件变多了，我们需要正式辟出一个新的物理 Room 吗？” 获得批准后，方可创建新目录并触发重索引。

## 5. The Masterstroke：硬链接与前置标签 (Hardlink & Tag-as-Room)
如果用户的原始文档结构非常混乱（比如大量散落在 `raw` 目录，且没有打正确的标签），纯靠 `pure_ingest.js` 的自动路径提取法就会失效。

与其强迫用户手动整理知识库，不如利用 **人类显式 Tag + OS 硬链接 (Hardlink)** 的特性，虚拟重构一个完美的“记忆宫殿”，而绝对不破坏原始的目录结构。

### 核心机制：Virtual Palace via Hardlinks
1. **源站隔离 (Source of Truth)**：维持用户原本凌乱的知识库原样。不对它们进行任何移动（`mv`）操作，防止打断用户在 Obsidian 里的既有链接。
2. **创作即分类 (Front-Loaded Tagging)**：当用户（或 Agent）在 Obsidian 中创建一篇 Markdown 笔记时，直接在文本开头写上 `#AI-Fortune` `#Architecture`。这 2 个 Tag，就是这篇文档在记忆宫殿里的 2 把房间钥匙。
3. **极速解析与虚拟分身 (Virtual Routing)**：
   - 每天凌晨 2:50，`mem-librarian`（图书管理员脚本）启动。它用正则 `/#([a-zA-Z0-9_-]+)/g` 瞬间抓出所有标签和父文件夹名。
   - 脚本首先强制 `rm -rf ~/.openclaw/virtual_palace/` 清空昨天的旧宫殿（彻底防止幽灵残留）。
   - 然后，脚本根据抓取到的标签，使用 Unix `ln` 指令，瞬间创建一万个跨房间的**硬链接分身**。
     - `ln 原始文件 ~/.openclaw/virtual_palace/AI-Fortune/原文件.md`
     - `ln 原始文件 ~/.openclaw/virtual_palace/Architecture/原文件.md`
4. **降维打击优势**：
   - 真正的“分身”：硬链接在操作系统的底层，指向的其实是同一个磁盘扇区（同一个 Inode）。一个文件可以同时存在于 10 个不同的虚拟房间里，但占据的磁盘空间依然只有 1 份。
   - 天然的同步更新：由于底层是同一个 Inode，原文件修改 0 延迟同步。
   - 解耦的最高境界：人类（无序的创造者）与底层 RAG 引擎（有序的检索者）通过操作系统的 Inode 和硬链接机制达成了完美的物理与逻辑解耦。纯净的 `pure_ingest.js` 只需要在凌晨 3:00 扫描这个已经被完美梳理好的 `virtual_palace` 即可。

## 14. 权限隔离 (Cube) 与聚类检索 (Room) 的大一统
在引入了 Virtual Palace 机制后，系统面临一个核心挑战：**大模型整理完公开知识库的 Room 后，如何处理分散在不同物理路径下的特工私密记忆（`memory/YYYY-MM-DD.md`）和系统规则（`SOUL.md`）？**

这本质上是 **Cube (Who)** 与 **Room (Where)** 维度的碰撞。

### 方案对比 (The Cube Conundrum)
**方案 A (保持纯粹入库逻辑)**：
让入库脚本 (`pure_ingest.js`) 彻底失去寻找文件的能力。它不再像一个到处乱爬的蜘蛛，而是只盯着大模型整理好的 `virtual_palace/` 目录。
- **做法**：图书管理员脚本 (`mem-librarian`) 升级，不仅扫描 `knowledge_base`, 还扫描每一个猫猫的 `memory` 目录。在虚拟宫殿里创建像 `~/.openclaw/virtual_palace/dev-cat/diary/` 这样的硬链接。入库脚本只管闭着眼睛吃。
- **缺点**：这会引入极其复杂的权限管理代码。Librarian Cat 必须拥有系统最高权限去读各个 Sandbox 的私有内存，还要把 `cube_id` 编码到虚拟目录的文件夹名字上。一旦出错，极易导致某只猫的数据错误被打上 `cube_id: global` 标签，导致隐私泄露。

**方案 B (关注点分离 Separation of Concerns) -【最优解】**
我们保持现有的**混合供血模式**。
- **Librarian Cat (大模型/正则脚本)** 只负责做最脏最累的语义聚类活儿。它的管辖范围严格限定在公共的 `knowledge_base/`。它生成的 `virtual_palace/global/` 里的所有内容，默认都是全系统可见的（`cube_id=global`）。
- **入库脚本 (`pure_ingest.js`)** 作为一个没有感情的搬运工，它依然保留它的 `SOURCES` 数组配置。
  1. 它去吃 `virtual_palace/global/`，把它们打上 `cube_id: global` 以及通过文件夹名提取的各色 `room` 标签入库。
  2. 然后它依次去爬每只猫猫的物理 `memory` 目录和 `SOUL.md`。对于这些特工私密文件，它直接硬编码打上 `cube_id: [猫的名字]`，并强行将其归类为 `room: 'diary'` 或 `room: 'system_config'`。

**结论**：方案 B 完美解耦。
- **权限安全 (Security/Cube)**：由呆板的、不可篡改的配置脚本 `pure_ingest.js` 在底层物理把关，绝对保证不会有数据越权串台。

- **知识聚类 (Semantic/Room)**：由灵活的、擅长整理文件的大模型（或标签正则）在 `virtual_palace/` 这一层尽情发挥。1
## 15. The Silent Guardian: 改造网关的前置拦截插件 (mem-local)
在确立了由大模型自主调用 `mem-search` 进行业务检索（手动挡）的基调后，网关底层的 `mem-local` 插件（自动挡）必须重新定位。

**痛点反思**：
- `mem-local` 之前会在每次用户发消息时，对整个 Qdrant 数据库进行全局扫描。虽然提高了阈值（0.75）避免了杂音注入，但这依然是对 I/O 的巨大浪费。
- 业务检索交给了大模型，那底层的自动拦截器该干什么？它应该作为 **“潜意识注入器 (Subconscious Injector)”**。

**改造方向：限定房间的精准打击 (Targeted Injection)**
我们不应该让它全局乱搜，而是让它在特定的高优房间内进行极速匹配。
在入库阶段（`pure_ingest.js`），我们已经为特工的私密记忆打上了极其精确的语义房间标签：
- 系统人设（SOUL.md, MEMORY.md 等） -> `room: 'system_config'`
- 每日记录（2026-04-08.md 等） -> `room: 'diary'`
- 业务日常（Obsidian 中的 /daily/ 目录） -> `room: 'daily'`

**新的过滤逻辑：**
当 `mem-local` 向后端的 `/query_graph` 发起检索时，必须同时携带这三个至关重要的房间名：
`rooms = ['system_config', 'diary', 'daily']`。

**为什么这样设计？**
这保证了当用户问及“我昨天跟你说了什么”或者“我的习惯是什么”时，大模型在生成回复**之前**的瞬间，系统就已经把这三类**高保真的身份与近期活动记忆**静默地缝合进了 System Prompt 中。大模型甚至不需要浪费一次 Tool Call 回合，就能表现出极其连贯的时间感知和身份认同。

## 结语：超越 MemPalace 的“零冗余物理态智能路由”
至此，CoreOS_Mem 2.1 架构升级完成。我们吸取了 MemPalace 优秀的“空间降维检索 (Wing/Room)”思想，但彻底抛弃了它在工程落地上的妥协（如：依赖 Hash ID 导致重名黑洞、依赖 LLM 查重导致性能爆炸、强行扭曲用户目录习惯）。

我们通过：
1. **OS Hardlink + Regex Tags** 实现了零空间损耗的一文多房间（Librarian Cat）。
2. **Inode 极速对账** 实现了真正的毫秒级增删改移全量同步（pure_ingest.js）。
3. **Cube + Room 双过滤墙** 实现了权限隔离与语义聚类的完美统一（server.js）。
4. **Agentic Tool Call + Subconscious Injector** 实现了业务主动探索与潜意识人设注入的关注点分离（mem-search & mem-local）。

这套架构将真正成为支撑 Limina Labs 与 OpenClaw 生态的终极知识底座。

## 16. Obsidian 实时同步架构 (Self-hosted LiveSync 替代轮询)
**痛点**：目前我们依靠 cron 每晚执行 `pure_ingest.js` 或 Inode 极速对账来保持知识库同步。但这仍然属于“准实时 (Near Real-time)”范畴。用户在 Mac 的 Obsidian 刚写完一段话，系统无法做到零延迟入库。

**方案：劫持 Self-hosted LiveSync**
用户已经在 Obsidian 中安装了开源插件 `obsidian-livesync` (Self-hosted LiveSync)，该插件天然具备将 Markdown 增量修改实时推送到远端 CouchDB 的能力。

**架构设计：**
1. **协议伪装 (Protocol Spoofing)**：无需在 Dash 节点真正部署一个庞大的 Apache CouchDB。我们可以用 Node.js 写一个极其轻量的 API 层，伪装成 CouchDB 的接收端 (mocking CouchDB HTTP protocol)。
2. **实时管道 (Real-time Pipeline)**：
   - 用户的 Obsidian 插件一按保存，`obsidian-livesync` 就会将这段文字修改以 PouchDB 增量文档 (Revision) 的格式 `POST` 到咱们伪装的 Node.js 接口。
   - 我们的伪装接口立刻解析 JSON，抽出正文，触发 Gemini Embedding。
   - 将向量实时写入 Qdrant，同时通知 Neo4j 更新图谱。
3. **彻底抛弃硬盘扫描**：结合前面设计的 `Tag-as-Room` 与硬链接，系统完全从 **“基于硬盘 I/O 的轮询”** 跃升为 **“基于 HTTP 增量事件的微秒级同步”**。用户的每一个键盘敲击，都会被转化成记忆宫殿里瞬间成型的一块新砖。

## 附录：LiveSync 插件的底层协议溯源 (CouchDB/PouchDB)
**PouchDB** 是一个开源的 JavaScript 数据库，其设计初衷是让浏览器在离线状态下也能像操作数据库一样存储数据，并能在恢复网络时自动与后端的 **Apache CouchDB** 进行双向同步（Replication）。

Obsidian 的  插件正是重度依赖了这套协议：
1. 它在您 Mac 本地的 Obsidian (Electron 浏览器环境) 里运行着一个本地的 **PouchDB**。
2. 您的每一次键盘敲击和 Markdown 修改，首先被存入本地 PouchDB。
3. 插件利用 PouchDB 原生的  方法，自动将增量数据（Revision）通过 HTTP API 打包发送给 Dash 节点上的服务端 **CouchDB**。

这也是为什么我们无法在服务器后端通过简单的 Node 脚本直接读取这些 Markdown 文件的原因：它们在 CouchDB 中并非以直观的  文本文件存在，而是被切碎成了数以万计的带有修订号（Revision ）、冲突标记和 Base64 编码附件的 NoSQL JSON 文档。

## 附录：LiveSync 插件的底层协议溯源 (CouchDB/PouchDB)
PouchDB 是一个开源的 JavaScript 数据库，其设计初衷是让浏览器在离线状态下也能像操作数据库一样存储数据，并能在恢复网络时自动与后端的 Apache CouchDB 进行双向同步（Replication）。

Obsidian 的 obsidian-livesync 插件正是重度依赖了这套协议：
1. 它在您 Mac 本地的 Obsidian (Electron 浏览器环境) 里运行着一个本地的 PouchDB。
2. 您的每一次键盘敲击和 Markdown 修改，首先被存入本地 PouchDB。
3. 插件利用 PouchDB 原生的 replicate.to() 方法，自动将增量数据（Revision）通过 HTTP API 打包发送给 Dash 节点上的服务端 CouchDB。

这也是为什么我们无法在服务器后端通过简单的 Node 脚本直接读取这些 Markdown 文件的原因：它们在 CouchDB 中并非以直观的 .md 文本文件存在，而是被切碎成了数以万计的带有修订号（Revision _rev）、冲突标记和 Base64 编码附件的 NoSQL JSON 文档。

## 附录 2：放弃 LiveSync CLI，回归 Rsync 大道至简
在尝试部署 obsidian-livesync 官方提供的 Headless CLI 版本（在 Dash 上作为 CouchDB 客户端使用）后，我们发现：
1. CLI 处于 Monorepo 深处，直接脱离宿主环境进行 NPM 编译的依赖成本极高。
2. 即便使用官方 Docker 版，由于其底层双向同步机制，每一次 Pull 都伴随着大量的 PouchDB JSON 冲突处理和 Base64 解码开销。

**KISS 架构决策 (Keep It Simple, Stupid)：**
既然本地的 Obsidian Vault 已经是 100% 完好的物理文件群，我们完全没有必要绕路通过 CouchDB 这个黑盒去进行解包和还原。

最终方案：
我们废弃了对 CouchDB 的反向劫持，回归最纯粹的 Unix 哲学。
在用户的 Mac 或 Gateway 节点上，配置一段基于 rsync 的 sync_vault.sh 脚本。
该脚本通过物理级拷贝，直接将 Markdown 文件秒级镜像到 Dash 节点的 /data/claw/private-wiki/，并完美规避 .obsidian 缓存。随后触发 pure_ingest.js 的 Inode 极速对账，实现真正意义上 0 解析损耗的端到端 RAG 同步！
## 17. 动态房间的生命周期：随生随灭 (Ephemeral Architecture)
在 Tag-as-Room 与 Hardlink 的组合架构下，房间的生命周期实现了真正的 **“所见即所得”**。

**传统的困境：**
一旦在系统中通过 LLM 或者代码生成了一个分类（Category / Room），即使里面所有的文档都被删除了，这个分类本身往往还会作为一个空的“幽灵文件夹”或“元数据标签”残留在数据库中。这种“元数据蔓延”会导致大模型的选项越来越多，噪音越来越大。

**我们的破局方案：**
1. **每日“核平”重构 (The Nightly Nuke)**：
   每天凌晨 2:50，`mem-librarian` 脚本的第一行代码就是 `rm -rf ~/.openclaw/virtual_palace/`。它无情地抹除昨日建立的所有虚拟宫殿和硬链接，不留任何历史包袱。
   然后，它仅基于**当前时刻**的 Markdown 文件内的 `#Tags` 和父文件夹名，重新使用 `ln` 指令铺设虚拟房间。
   **推论：** 如果您今天把所有日记里的 `#Kami-Vision` 标签都删除了，今晚重构时，虚拟宫殿里就绝对不会再创建出 `Kami-Vision` 这个硬链接子目录。

2. **极速对账的超度 (The Inode Reconciliation)**：
   每天凌晨 3:00，`pure_ingest.js` 进行扫描。当它发现由于昨天某个标签（硬链接）消失了，导致它记录在 `coreos_sync_state.json` 里的某个路径在今天“不见了”，它会立刻触发服务端的 `/delete` 接口，顺着文件路径把旧的 Vector（向量）和 Neo4j 图谱节点**连根拔起，当场超度**。

3. **所见即所得的房间清单 (List-Rooms)**：
   `mem-list-rooms` 工具并不是读取配置文件，而是直接调用 Qdrant 原生接口对前 10,000 个向量做了一次 `rooms` 字段聚合 (`Set` 去重)。
   当最后一条挂着 `#Kami-Vision` 的向量被 `/delete` 超度后，这个 Room 在数据库里就随之灰飞烟灭了。当大模型再次调用 `list_rooms` 时，它就不可能再看到这个空房间。

**总结：标签即房间。**
有内容，房间拔地而起；没内容，房间烟消云散。
这是一种绝对弹性的认知架构，它保障了无论用户的兴趣如何迁移，系统的导航地图永远精确地反映当前脑海里最活跃的领域。
