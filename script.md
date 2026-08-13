# AI Harness Design — 演讲脚本 / Speaker Script

Deck: `AI_Harness_Design_v7.pptx` (23 slides) · Target: 40 min + 10 min Q&A
每页格式：**EN** 关键词 → **中文** 关键词 → 数据/引用 → 例子

> 用法：不要照读。每页先说 ✦ 那一句，其余是备用弹药。

---

## ⏱ 时间分配 / Timing

| Slides | Section | Min |
|---|---|---|
| 1–2 | Opening | 2 |
| 3 | Vocabulary | 2 |
| 4–6 | Theory | 9 |
| 7 | The loop | 3 |
| 8 | Design contents | 0.5 |
| 9–20 | Design | 23 |
| 21–22 | Where the field is going | 3.5 |
| 23 | Close | 0.5 |

**卡点提示**：讲到 slide 12 时应该已过 20 分钟。超时先砍 21–22（where the field is going）。

---

# 1 · Title

**EN**
- ✦ "Nothing today is about making the model smarter. All of it is the code around the model."
- Shape: 10 min foundations → the rest is design decisions
- Ask them to hold questions to section breaks

**中文**
- ✦ 「今天讲的没有一句是关于怎么让模型更聪明。全部都是模型外面那一圈代码。」
- 结构：前 10 分钟打底 → 剩下全是设计决策
- 问题请留到每节之间

---

# 2 · We are not building an agent

**EN**
- ✦ Agency comes from training, not from our orchestration code
- Model = driver · harness = vehicle
- Anti-pattern: node graphs + hardcoded if/else + LLM as a text-completion node
- Caps out at the quality of your decision tree — gets *relatively worse* as models improve

**中文**
- ✦ 智能来自训练，不来自我们的编排代码
- 模型是司机，harness 是车
- 反面模式：拖拉拽流程图 + 写死的 if/else + 把 LLM 当成一个补全节点
- 上限 = 你那棵决策树的水平；模型越强，这种做法相对越差

**数据 / Evidence（被质疑时才用）**
- DQN/Atari 2013 · OpenAI Five 2019 · AlphaStar 2019 · LLM coding agents 2024–26
- 每一次都是同一个架构：训练好的模型 + 环境 + 动作空间

**例子 / Example**
- "if the query mentions VaR, call `compute_var()`" → 这是在赌模型不行，这个赌注一定输
- 中文：写死「问到 VaR 就调 compute_var()」，等于赌模型永远不会自己判断 —— 这个赌必输

---

# 3 · Vocabulary

**EN**
- ✦ A tool is capability · a skill is knowledge · a hook is what always runs
- Read down the right-hand column ("where it lives") — fastest way to make it concrete
- Model row is load-bearing: stateless, no memory → all of theory follows from it

**中文**
- ✦ tool 是「能力」，skill 是「知识」，hook 是「无论模型怎么决定都会跑的代码」
- 顺着最右边一列「住在哪里」念一遍，最快建立体感
- Model 那一行最关键：无状态、没有记忆 —— 后面理论部分全部由此推出

**如果被问 MCP / If asked about MCP**
- MCP 不是一种 tool，是「拿到别人写的 tool」的传输+发现标准
- 到了模型眼里，MCP tool 和自研 tool 完全一样：都只是 tools 数组里的 name + schema

---

# 4 · One request, two very different phases

**EN**
- ✦ Output costs 5× input — that's hardware, not pricing
- Token ≈ 4 chars English ≈ 0.75 word
- **Prefill (read)**: whole prompt in one parallel pass · big matmuls · compute-bound · GPU busy
- **Decode (write)**: one token at a time · each reloads full weights from HBM for tiny arithmetic · bandwidth-bound · GPU util ~20–40%
- Consequence: reading cheap, writing expensive
- TTFT = prefill (scales with prompt); everything after = decode (scales with answer)

**中文**
- ✦ 输出比输入贵 5 倍 —— 这是硬件决定的，不是定价策略
- 一个 token ≈ 4 个英文字符 ≈ 0.75 个词
- **Prefill（读）**：整个 prompt 一次并行过完 · 大矩阵乘 · 算力瓶颈 · GPU 是真的在忙
- **Decode（写）**：一次吐一个 token · 每个 token 都要把整个模型权重从显存搬一遍，只为做一点点算术 · 带宽瓶颈 · GPU 利用率只有 20–40%
- 结论：读便宜，写贵
- 首字延迟 = prefill，跟 prompt 长度走；之后全是 decode，跟答案长度走

**数据 / Numbers**
- Opus 5 $5 / $25 · Sonnet 5 $2 / $10 · Haiku 4.5 $1 / $5（每百万 token，输入/输出）
- 全线都是 5×

**例子 / Example**
- EN: pull a 50-page doc in, ask for a 3-line answer = right way round. Ask the model to *reproduce* the doc = wrong way round
- 中文：把 50 页文档塞进去、要 3 行结论 → 用对了；让模型把文档「复述」一遍 → 用反了
- 有人说 agent 慢 → 先问是「等第一个字」还是「吐字过程慢」，两者的优化方向完全不同

---

# 5 · The KV cache

**EN**
- ✦ Attention = every token looks at every earlier token → naive = quadratic work for linear output
- Fix: compute each token's key/value **once**, keep them; new token only computes its own
- 3 consequences:
  1. this is why long context works at all
  2. grows with the conversation, lives in GPU memory → decode slows as you go
  3. **dies when the request ends** — per-request, ephemeral
- ✦ Point 3 is the entire reason the next slide exists

**中文**
- ✦ Attention 要求每个 token 看它前面所有 token → 朴素做法是「线性的输出，平方的计算量」
- 解法：每个 token 的 key/value 只算一次并缓存；新 token 只算自己的，其余去缓存里取
- 三个后果：
  1. 长上下文之所以可行，就是靠它
  2. 缓存随对话线性增长、占显存 → 对话越长 decode 越慢
  3. **请求一结束就没了** —— 单次请求内有效，用完即弃
- ✦ 第 3 点就是下一页存在的全部理由

**如果被问 / If asked**
- Q: 为什么不能让服务器一直帮我存着？
- A: 那正是 prompt cache，默认存 5 分钟；也正因为是「原样复用」，才只能精确前缀匹配

---

# 6 · The prompt cache

**EN**
- ✦ Prefix stability is architecture, not a tuning detail
- Same KV state, kept between requests. Nothing more exotic
- Exact prefix match, order **tools → system → messages**; change any level → that level + everything after is invalidated
- Read 0.1× · write 1.25× (5 min) · write 2× (1 h) · 5-min cache refreshed free on every hit
- Fine print: 4 breakpoints · 20-block lookback · min 512–4,096 tokens (below that, silently no cache)
- Observability: `cache_read_input_tokens` / `cache_creation_input_tokens` / `input_tokens` — both cache fields 0 = not caching

**中文**
- ✦ 前缀稳定性是架构问题，不是调参细节
- 就是把上一页那份 KV 状态在请求之间留着，没有别的玄机
- 精确前缀匹配，顺序固定 **tools → system → messages**；任何一层变了，这一层及其之后全部失效
- 读 0.1× · 写 1.25×（5 分钟）· 写 2×（1 小时）· 5 分钟缓存每次命中免费续期
- 细则：最多 4 个断点 · 回溯 20 个 block · 最少 512–4,096 token，低于此静默不缓存
- 可观测：三个字段都要看；两个 cache 字段都是 0 = 根本没缓存上

**两个必须点名的坑 / Two failure modes to name**
1. EN: timestamp or live mark near the top of system prompt = 0% hit rate forever **and** 1.25× on every call
   中文：把时间戳、实时报价放在 system prompt 靠前 → 永远 0 命中率，而且每次都付 1.25×
2. EN: tool definitions sit at the very front → change one description, cache dies for *every live session* at once
   中文：tool 定义在最前面 → 改一句描述，所有在跑的会话缓存同时失效。**tool 定义是发布物，不是配置**

---

# 7 · The loop

**EN**
- ✦ The loop never changes. Everything else in this talk is what lives inside steps 1 and 4
- Steps 1 + 4 ≈ 90% of the code; step 2 (the "AI part") is the smallest
- Exit is a **typed enum** of ten terminal reasons — not a thrown exception
- Loop is UI-agnostic: all I/O injected as callbacks → one loop serves CLI, batch, SDK
- If a proposed feature requires editing the loop → probably in the wrong place

**中文**
- ✦ 这个循环本身永远不变。今天剩下的内容，全都住在第 1 步和第 4 步里
- 第 1 步 + 第 4 步大约占 90% 的代码；第 2 步（真正「AI」的那步）反而最小
- 退出是**带类型的枚举**，十种终止原因 —— 不是抛异常
- 循环与 UI 无关：所有 I/O 以回调注入 → 同一个循环同时服务 CLI、批处理、SDK
- 如果一个需求要求你改这个循环，多半是放错地方了

---

# 8 · Design — contents

**EN**
- ✦ Don't read the list. Point and say where the emphasis is
- Context first: it's the one every harness gets wrong, and it bleeds money quietly instead of failing loudly
- Exception handling + dead loops sit together: same problem from two sides — a step fails vs. recovery itself fails
- Governance deliberately in the middle, not at the end. Retrofitting it is how these projects die
- If short on time, cut build-vs-buy — it's the only part about someone else's code

**中文**
- ✦ 不要念清单。用手指一下，说明重点在哪
- Context 放第一：这是每个 harness 都会做错的地方，而且它不是「报错」，是「安静地烧钱」
- 异常处理和死循环放在一起：同一个问题的两面 —— 一步失败了，和「恢复本身也失败了」
- 治理故意放中间，不放最后。补票式治理是这类项目的死法
- 时间不够先砍 build vs buy —— 只有这一节讲的是别人的代码

---

# 9 · Context

**EN**
- ✦ Every fact belongs to **exactly one** tier — and the tiers differ in review, audit and rollback
- Assembled fresh **every turn** from 4 sources: system prompt (concatenated sections, not a hardcoded string) · user context (project file + today's date, memoised per conversation so it doesn't churn the cache prefix) · system context (git status) · attachments
- Walk the **Mechanism** column — that's what makes the tiers concrete
- Project tier = a file in a repo: diffable, reviewable, revertable, shows up in a PR. Session tier = none of that; nobody code-reviews a transcript
- Long memory = 3 problems: **selection** (worth remembering?) · **extraction** (cleanly out of a transcript) · **consolidation** (merge without unbounded growth). Most teams build extraction and skip the other two
- Persona: no separate personality config — behaviour is prompt sections + project file → changing behaviour is a *reviewed code change*

**中文**
- ✦ 每一条信息只属于**一个** tier —— 而不同 tier 的区别在于：能不能评审、能不能审计、能不能回滚
- **每一轮**都重新拼装，来自 4 个来源：system prompt（分段拼接，不是硬编码字符串）· user context（项目文件 + 当天日期，按会话记忆化，避免打乱缓存前缀）· system context（git 状态）· attachments
- 重点讲 **Mechanism** 那一列 —— 有了它，四个 tier 才具体
- Project tier = 仓库里的一个文件：可 diff、可评审、可回滚、会出现在 PR 里。Session tier 一个都不占：没人会去 code review 一份对话记录
- 长期记忆是三个问题：**选择**（值不值得记）·**抽取**（怎么干净地从对话里拿出来）·**归并**（合并进已有内容而不无限膨胀）。多数团队只做了抽取
- 人设：没有独立的「性格配置」。行为 = prompt 分段 + 项目文件 → 改行为 = 改一次要过评审的代码

**For us / 对我们**
- EN: methodology + desk conventions → project tier. A client's positions or today's marks → **none of them**, fetch via tool every time
- 中文：方法论、台面惯例 → project tier。客户持仓、当日估值 → **哪个 tier 都不进**，每次用 tool 现取
- 一句话：memory 文件里的过期财务数据 = 一起合规事故

**指着截图说 / Point at the screenshot**
- 246.1k / 1m 已用 · system prompt 20k · messages 214.6k · free space 75.4%
- 「还没干活，20k 就没了」

---

# 10 · Context management

**EN**
- ✦ Four jobs we'd have hand-built a year ago are now platform primitives. The line between **server** and **client** is where your bugs and compliance exposure live
- Prompt caching — server, GA. We own prefix stability only
- Context editing — server. `clear_tool_uses` / `clear_thinking`, beta header. Clears oldest tool results past your threshold, leaves a placeholder, **preserves block pairing**, and you can exempt specific tools
- Compaction — server. Summarises whole conversation, returns a compaction block, drops everything before it. Trigger min 50k, default 150k
- **Memory tool — CLIENT.** GA, no beta header, `memory_20250818`
  - Claude only *requests* file ops (`view` / `create` / `str_replace` / `insert` / `delete` / `rename`) against `/memories`
  - Your code executes every one, against storage you choose
  - API injects its own system prompt: "check memory first, assume the context window may reset at any moment"
- ✦ So the security is entirely ours — docs are explicit

**中文**
- ✦ 一年前要自己写的四件事，现在是平台原语。**server 还是 client** 这条线，就是 bug 和合规风险所在
- Prompt caching —— 服务端，已 GA。我们只负责前缀稳定
- Context editing —— 服务端。`clear_tool_uses` / `clear_thinking`，走 beta header。超过阈值后清掉最老的 tool 结果，留一个占位符，**保持 block 配对**，还能指定某些 tool 豁免
- Compaction —— 服务端。把整段对话总结成一个 compaction block，之前的全丢掉。触发阈值最低 5 万，默认 15 万
- **Memory tool —— 客户端。** 已 GA，不需要 beta header，类型 `memory_20250818`
  - Claude 只是「请求」文件操作（六个命令），路径前缀 `/memories`
  - 每一个都由你的代码执行，存到你自己选的地方
  - API 会自动注入一段系统提示：先看 memory 目录；假设上下文随时可能被清空
- ✦ 所以安全责任完全在我们这边 —— 官方文档写得很直白

**必须点名的安全项 / Security items to name**
- 路径穿越校验 —— `/memories/../../secrets.env` 是真实攻击面
- 文件大小上限 · 写入前脱敏 · 过期清理
- 中文一句话：这是个五行代码就能打开、但会引入「模型输出驱动的文件写入」的功能 → 是安全评审项，不是配置项

**配合使用 / Pairing**
- compaction 管「压缩」，memory 管「压缩后必须还活着的东西」
- 长任务模式：初始化会话写 progress log + feature list → 之后每次会话先读、只做一件事、结束前更新 → **端到端验证过才算完成**

---

# 11 · Compaction

**EN**
- ✦ Five strategies, cheapest first — a heavier stage no-ops if a lighter one already freed enough. Only the last costs a model call
- 1 tool-result budget: cap aggregate output, offload to disk, keep a reference
- 2 snip: drop whole stale messages
- 3 microcompact: blank the **content** of old tool results, keep the structure
- 4 context collapse: read-time projection, summaries in a side store
- 5 autocompact: summarise and **replace** ← the only model call
- Need **proactive AND reactive**: proactive on an estimate before the call; reactive after a real 413. Your estimate will be wrong
- Circuit breaker: stop at 3. It does **not** back off — it stops

**中文**
- ✦ 五种策略，从便宜到贵 —— 前面的省够了，后面的自动变成空操作。只有最后一个要花一次模型调用
- 1 tool 结果预算：限制总输出体积，超出的落盘，只留引用
- 2 snip：整条整条丢掉陈旧消息
- 3 microcompact：把老的 tool 结果**内容**清空，结构保留
- 4 context collapse：读取时投影，摘要存在旁边的库里
- 5 autocompact：总结并**替换** ← 唯一要调模型的
- 必须**主动 + 被动**都有：主动是调用前按估算触发，被动是真的收到 413 之后。你的估算一定会错
- 熔断：连续失败 3 次就停。注意不是退避重试，是**停**

**数据 / Numbers**
- 1,279 个会话连续失败 50+ 次，单会话最高 3,272 次
- 全球每天浪费约 25 万次 API 调用
- 中文金句：输入本身就无法恢复时，退避只是让同一个失败发生得更慢

**For us / 对我们**
- 绝不把 5 万行结果塞进 context → 落盘，返回 schema + 前几行 + 一个句柄

---

# 12 · A tool must never throw into the loop

**EN**
- ✦ Errors become **values**. An exception at the tool boundary becomes a `tool_result` with `is_error: true`, and is yielded like any other result
- Why: API requires every `tool_use` to have a matching `tool_result`. An uncaught throw leaves an orphan → the **next** request 400s → crash surfaces one turn later, far from its cause
- Defended in 4 places: the tool wrapper · a generator that synthesises results for dangling blocks (on fallback / error / abort) · synthetic errors for cancelled siblings · a drain that must be consumed even on abort
- ✦ General rule: every unit of concurrent work needs a **guaranteed terminal message** — happy path, error path, AND cancellation path
- Layering: transient (429/529/ECONNRESET) retried **below** the loop · semantic (413, max-output-tokens) become **messages** · only bugs throw
- Withhold ladder: withhold → recover cheaply → escalate → surface & stop
- Exhaustion rule (quote it): "Surface the withheld error and exit. Do NOT fall through to stop hooks — that creates a death spiral"

**中文**
- ✦ 错误变成**值**。tool 边界上的异常，变成一个 `is_error: true` 的 `tool_result`，像普通结果一样 yield 出去
- 为什么：API 要求每个 `tool_use` 必须有配对的 `tool_result`。异常没接住就留下孤儿 → **下一次**请求 400 → 崩溃在一轮之后才暴露，离病因很远
- 四个地方守这条不变量：tool 包装层 · 为悬空 block 合成结果的 generator（fallback / 出错 / 中断时）· 给被取消的兄弟 tool 合成错误 · 即使中断也必须消费完的 drain
- ✦ 通用规则：每一个并发工作单元都要有**保证送达的终止消息** —— 正常路径、错误路径、**以及取消路径**
- 分层：瞬时错误（429/529/断连）在循环**下面**重试 · 语义错误（413、输出截断）变成**消息**交给循环判断 · 只有代码 bug 才真的抛
- 阶梯：withhold → 便宜的恢复 → 升级 → 暴露并停止
- 「耗尽规则」原文照念：恢复失败就把错误抛出来直接退出，**不要**掉进 stop hook —— 那是死亡螺旋

**如果有时间 / If time allows**
- 重试策略带来源标记：只有用户正在等的调用才重试 529。摘要、起标题、分类器立即放弃 —— 容量雪崩时每次重试都是 3–10 倍网关压力
- 关闭有硬超时：harness 里「退出时卡住」会被当成「崩溃」上报

---

# 13 · Dead loops

**EN**
- ✦ Name the state, then cap it
- Six causes: ① error → recovery → same error ② unbounded retry ③ re-entrancy / self-recursion ④ missing hard caps ⑤ reactive feedback loops ⑥ real deadlock
- **Latch + explicit reset policy** — the subtle one. A one-shot flag in loop state; you must decide *at each continue-site* whether it resets. Stop-hook branch preserves it; a fresh turn resets it. **Defaulting to reset is how you get the spiral**
- **Provenance tags, not a recursion flag** — tag every model call with its source; subsystems refuse to fire on their own provenance. A generic "am I recursing?" flag can't express the nasty case: a forked agent whose compaction resets state shared with its parent
- Caps checked only on the happy path aren't caps — max-turns is checked on the abort path too

**中文**
- ✦ 先给状态起名字，再给它设上限
- 六种成因：① 出错 → 恢复 → 同样的错 ② 无上限重试 ③ 重入 / 自递归 ④ 没有硬上限 ⑤ 响应式系统里的反馈回路 ⑥ 真正的死锁
- **闩锁 + 明确的重置策略** —— 最微妙的一条。一次性标志位放在循环状态里；**每一个 continue 的地方**都要显式决定它重不重置。stop hook 那条分支故意保留，新一轮才重置。**默认重置就是螺旋的来源**
- **来源标记，而不是「我是不是在递归」的开关** —— 每次模型调用都打上来源；子系统拒绝在自己的来源上触发。通用递归标志表达不了那个恶心的场景：一个 fork 出来的 agent，它的压缩会重置和父进程共享的状态
- 只在正常路径上检查的上限不算上限 —— max-turns 在中断路径上也要检查

**数据 / Quote**
- 「compact → 还是太长 → 报错 → stop hook 拦截 → 再 compact → …」烧掉几千次 API 调用
- ✦ 这一页是讲义。如果他们只拍一张照片，会是右边这张清单

---

# 14 · Concurrency

**EN**
- ✦ There are no threads. Zero `worker_threads`, zero `new Worker()` in the whole codebase — a decision, not an omission
- Three layers, kept distinct — that separation is what makes the error model tractable:
  - **async** — event loop + async generators — isolation = try/except inside the generator
  - **sub-process** — spawn (shell, search, hooks) — isolation = exit code + abort signal
  - **separate process** — files + locks, no shared memory — isolation = the OS
- Why: workload is IO-bound (model latency, subprocess latency) → async gives you all the concurrency there is. Need true isolation → jump straight to **processes**: forces serializable communication, makes crash isolation free
- Cancellation is a **tree**, not a flag: query → sibling batch → per tool. Asymmetric by design — aborting a child never touches the parent → kill sibling tools without ending the turn. One escape hatch: permission rejection must end the turn, so that abort re-fires upward

**中文**
- ✦ 没有线程。整个代码库里 `worker_threads` 和 `new Worker()` 都是零 —— 这是设计决策，不是遗漏
- 三层并发，刻意分开 —— 正是这个分离让错误模型可控：
  - **async** —— 事件循环 + 异步生成器 —— 隔离靠 generator 内部的 try/except
  - **子进程** —— spawn（shell、搜索、hook）—— 隔离靠退出码 + abort 信号
  - **独立进程** —— 文件 + 锁，无共享内存 —— 隔离靠操作系统
- 理由：负载是 IO 密集（模型延迟、子进程延迟）→ async 已经给了你全部的并发。需要真隔离时，直接跳到**进程**：强制通信可序列化，崩溃隔离白送
- 取消是一棵**树**，不是一个开关：query → 同批 → 单个 tool。方向刻意不对称 —— 取消子节点不影响父节点 → 可以杀掉兄弟 tool 而不结束这一轮。唯一的例外：权限拒绝必须结束这一轮，所以那个 abort 会向上冒泡

**Python 对照 / For our backend**
- `asyncio` + async generators —— 循环、流式、tool 并发
- `asyncio.create_subprocess_exec` —— 每个要 shell 的 tool
- `anyio.CancelScope` / `asyncio.TaskGroup` —— 免费拿到嵌套；向上冒泡那条要自己写
- **IO 密集永远不要用线程**

**细节（有时间才讲）/ Details if time**
- listener 要用弱引用：父子直接注册会每个子节点漏一个 handler，几千次 tool 调用后就是稳定和 OOM 的区别
- abort reason 是带类型的协议不是字符串：`interrupt` 只取消声明可取消的 tool —— 正在写文件的不会被打断
- 多进程协调用「带锁的文件邮箱」：**异步锁 + 重试**，绝不用同步锁 —— 同步锁会阻塞事件循环，把流式、UI、其他所有 tool 一起卡住

---

# 15 · Multi-agent

**EN**
- ✦ Four **distinct** concurrency models, not variations of one thing
- **L1 tool parallel** — shared context, flat batch — free, start here. Defaults fail closed: an unmarked tool running in parallel is a data race; unmarked and serial is just slow. Safety is per **input**, not per tool type
- **L2 subagent** — own context, parent waits — when **context** is the bottleneck. Parent sees only the final report, not the 40 tool calls. Subagents are themselves concurrency-safe → N in one message fan out in parallel
- **L3 background agent** — own context, detached — when **latency** is the bottleneck. Promotion pattern: register foreground, race a background signal, offer to background at 2 s. Abort tree deliberately **severed** so it survives the turn
- **L4 teammate** — own process, graph of peers — long-lived collaborators only. Real cost: file locks, permission sync, reconnection, negotiated shutdown
- ✦ Whatever the level: propagate a **chain id + depth on every event**. Without it you have concurrent logs and no causality

**中文**
- ✦ 是四种**不同**的并发模型，不是一件事的四个版本
- **L1 tool 并行** —— 共享上下文，扁平批次 —— 免费，从这里开始。默认失败关闭：没标记的 tool 并行跑 = 数据竞争；没标记但串行 = 只是慢。安全性看**输入**，不看 tool 类型
- **L2 子 agent** —— 独立上下文，父节点等待 —— 当瓶颈是**上下文**时用。父节点只看到最终报告，不看那 40 次 tool 调用。子 agent 本身是并发安全的 → 一条消息里开 N 个会并行
- **L3 后台 agent** —— 独立上下文，脱离 —— 当瓶颈是**延迟**、且结果不影响继续时用。晋升模式：先注册成前台，和后台信号赛跑，2 秒后提示「要不要转后台」—— 把决定推迟到有依据的那一刻。abort 树在这里**故意剪断**，让它活过这一轮
- **L4 队友** —— 独立进程，对等图 —— 只给真正长期协作用。代价是实打实的：文件锁、权限同步、断线重连、协商关闭
- ✦ 不管哪一层：**每个事件都带上 chain id 和 depth**。没有它，你只有并发的日志和零因果关系

**For us / 对我们**
- 子 agent 的隔离不只是上下文边界，更是**治理边界**
- 一个只给只读行情 tool、完全没有写 tool 的 research 子 agent —— 不管 prompt 怎么写都出不了事故
- 中文金句：这是代码里能强制执行的，指令做不到

---

# 16 · Permissions

**EN**
- ✦ A control problem before an engineering one. What's new isn't the controls, it's **where they attach**
- Gate returns allow / deny / ask — precedence: **deny beats ask beats allow**
- Modes (default / plan / acceptEdits / bypass / auto) transform the outcome **globally** → treat a mode change as a controlled operation
- Batch needs an explicit policy for what "ask" means: auto-deny, or route to a classifier. This is where accidents happen
- Hooks are the deterministic layer: PreToolUse · PostToolUse · Stop · SessionStart/End · PreCompact · UserPromptSubmit → **compliance logic lives there, not in a prompt**
- Read ≠ write ≠ send outbound — three separate boundaries
- Point at the screenshot: the model asked, a human decided. That's the shape we want in production

**中文**
- ✦ 这首先是控制问题，其次才是工程问题。新的不是「有控制」，而是控制**挂在哪里**
- 门返回 allow / deny / ask —— 优先级：**deny > ask > allow**
- 模式（default / plan / acceptEdits / bypass / auto）会**全局**改变结果 → 改模式要当成一次受控操作
- 批处理必须明确规定「ask」是什么意思：自动拒绝，还是走分类器。事故都出在这里
- Hook 是确定性那一层：PreToolUse、PostToolUse、Stop、SessionStart/End、PreCompact、UserPromptSubmit → **合规逻辑住在这里，不住在 prompt 里**
- 读 ≠ 写 ≠ 对外发送 —— 三条独立边界
- 指着截图：模型问了，人决定了。这就是我们要的生产形态

**不可谈判 / Non-negotiable**
- 每一次 tool 调用的不可变审计日志：入参、出参、权限决定
- 涉及 book of record 的写操作，硬性拒绝清单
- 任何要送到客户面前的模型输出，默认必须有人在环

**互动 / Interaction**
- 表格是故意留空的 —— 现场一起填，或提前填好

---

# 17 · Skills explosion

**EN**
- ✦ Description text is **rent**. Every definition sits in the prompt on every request → pay for a **name**, not a definition
- 100 MCP tools ≈ tens of thousands of tokens of permanent overhead, competing with actual work
- **Tools → deferred loading**: model sees name + one line; full definitions expand server-side only when it searches
  - auto-enable by **measured cost** (past ~10% of the window), **not** tool count — one verbose tool can cost more than twenty terse ones
  - discovery must survive compaction, or the model re-searches for tools it already has
  - announce pool changes as a **delta** — MCP servers connect/disconnect mid-session; a full re-listing destroys the prompt cache
- **Skills → progressive disclosure + hard budget**: 1% of context for the listing, 250 chars per description, full content only on invoke
- **`paths:` scoping** is the headline: conditional skills are **completely absent** from the model's view until a file operation touches a matching path
- Escape valve — semantic retrieval, with 2 conditions: trigger on typed signals (always-on found nothing 97% of the time), and prefetch under existing latency (250–570 ms hidden inside a 2–30 s turn)

**中文**
- ✦ 描述文字是**房租**。每一条定义每次请求都在 prompt 里 → 只为**名字**付费，不为定义付费
- 100 个 MCP tool ≈ 几万 token 的永久开销，和真正的工作抢地方
- **Tools → 延迟加载**：模型只看到名字 + 一行；完整定义在它搜索时才由服务端展开
  - 按**实测成本**开启（超过窗口约 10%），**不是**按 tool 数量 —— 一个啰嗦的 tool 可能比二十个简洁的还贵
  - 发现结果必须扛过压缩，否则模型会重新搜一遍它已经有的 tool
  - 变更要按**增量**广播 —— MCP server 会中途连上/断开；整份重发会摧毁 prompt 缓存
- **Skills → 渐进披露 + 硬预算**：清单占上下文 1%，每条描述 250 字符，完整内容只在调用时加载
- **`paths:` 作用域**是重点：带条件的 skill 在文件操作命中路径之前，对模型**完全不存在**
- 兜底方案 —— 语义检索，两个前提：按明确信号触发（一直开着的版本在生产里 97% 的时候什么都没找到），以及把延迟藏在已有工作下面（250–570 毫秒藏进 2–30 秒的一轮里）

**例子 / Example**
- EN: equity-research and credit-research shouldn't both be visible. Scope them → mutually invisible → no prompt cost **and** no chance of picking the wrong one
- 中文：股票研究和信用研究两个 skill 不该同时可见。用路径圈定 → 互相不可见 → 既省 prompt，又根本没有选错的机会

**梯子 / The ladder**
- 预算清单 → 按路径圈定 → 长尾延迟加载 → 前三步不够才上语义检索 → **量化命中率**

**收尾一句 / Hand-off line**
- 「路径圈定只解决了三种冲突里的一种。」→ 翻页

---

# 18 · When scoping is not enough

**EN**
- ✦ Path scoping answers **one of three** conflict shapes. The other two have no file event to trigger on
- **① File-triggered — solved.** Model opens `portfolios/equities/…` → skill activates. Unambiguous signal
- **② Question-triggered — no signal at all.** "What's our exposure to rising rates?" is legitimately a rates question *and* a credit question. Nothing on disk touched → `paths:` never fires → both sit in the listing, competing. The model picks one and you never learn it was a coin flip
- **③ History-inherited — the dangerous one.** Activation is **one-way**: conditional → active, never back. A skill that fired 40 turns ago is still visible. After compaction the *reason* it fired is summarised away — the skill is not. A session that opened in equities silently biases a credit question an hour later
- Three fixes: `conflicts_with:` (mutual exclusion — `paths:` expresses relevance, not "never both") · an **expiry** on activation (TTL, drop after N turns with no signal) · **re-derive after compaction** (rebuild the active set from what survived, never from a flag that outlived its reason)

**中文**
- ✦ 路径圈定只回答了三种冲突里的**一种**。另外两种根本没有文件事件可以触发
- **① 文件触发 —— 已解决。** 模型打开 `portfolios/equities/…` → skill 激活。信号明确无歧义
- **② 问题触发 —— 完全没有信号。** 「我们对利率上行的敞口有多大？」这**同时**是利率问题和信用问题。硬盘上什么都没动 → `paths:` 永远不会触发 → 两个 skill 一起挂在清单里抢同一个位置。模型选了一个，而你永远不知道那是抛硬币
- **③ 历史继承 —— 最危险的一种。** 激活是**单向**的：conditional → active，永远回不去。40 轮之前激活的 skill 现在还在。压缩之后，它当初为什么激活这件事被总结掉了 —— 但 skill 还在。一个从股票开场的会话，一小时后会悄悄污染一个信用问题
- 三个补丁：`conflicts_with:`（互斥 —— `paths:` 表达的是「相关」，表达不了「绝不能同时出现」）· 给激活加**过期**（TTL，连续 N 轮没有匹配信号就摘掉）· **压缩后重新推导**（用幸存下来的上下文重建激活集合，绝不用一个活得比理由还久的标志位）

**为什么第三条最容易漏 / Why fix ③ is the one people miss**
- 压缩正是「理由消失」的那一刻。标志位活下来了，依据没有 —— 这时候它就是一个你无法辩护的状态
- 中文金句：一个你说不出理由的激活状态，就是一个 bug 在等时间

**For us / 对我们**
- 多资产台面本来就会在一个会话里换资产类别 —— 这不是边缘情况，是正常工作流
- ✦ 「为切换而设计，不要只为开场那个问题设计」

---

# 19 · Long-running tasks

**EN**
- ✦ Output goes to disk from byte zero. The model reads deltas by byte offset → memory is O(1) however long the job runs
- Hard cap + in-band truncation marker → a runaway loop fills a file, not the heap
- Push completion, poll slowly as a backstop — and **exactly one owner** per notification (two owners = dual delivery)
- Priority queue: user input `next`, task notifications `later`; drained **mid-turn**; scoped by agent id so subagents never see the user prompt stream
- Notifications need an explicit **terminal flag** — the *absence* of a status field is load-bearing: an unknown value falls through to "completed" and falsely closes the task
- Completion must be **idempotent and atomic** — natural exit, user kill, tool-driven stop can all finish a task; exactly one notification fires
- Give the model its own task tools: create / list / get / output / update / stop
- ✦ **The stall watchdog** — the hang-with-no-error is the failure you'll actually hit

**中文**
- ✦ 输出从第一个字节就写盘。模型按字节偏移读增量 → 不管任务跑多久，内存都是 O(1)
- 硬上限 + 带内截断标记 → 失控的循环撑爆的是一个文件，不是堆
- 完成用推送，慢轮询只做兜底 —— 并且每条通知**有且只有一个负责人**（两个 = 重复投递）
- 优先级队列：用户输入 `next`，任务通知 `later`；**在一轮中间**排空；按 agent id 隔离，子 agent 永远看不到用户输入流
- 通知必须有明确的**终止标记** —— 「没有 status 字段」这件事本身是有含义的：未知值会 fall through 成 "completed"，把任务错误地关掉
- 完成必须**幂等且原子** —— 自然退出、用户杀掉、工具停止都可能完成一个任务，但只能发一条通知
- 给模型自己的任务工具：create / list / get / output / update / stop

**Watchdog 四步 / Four steps**
1. 盯输出文件有没有增长
2. 45 秒没长？把尾部读出来
3. 最后一行看起来像不像交互提示？
4. 像的话，告诉用户**怎么修**（改用管道输入 / 加非交互参数），不是只说「卡住了」

**For us / 对我们**
- 回测、风险跑批、隔夜任务正是这个形状
- 别把回测结果灌进 context → 落盘，给模型一个 task id，让它自己轮询
- 中文金句：你真正会遇到的故障不是崩溃，是一个安静地永远跑不完的任务

---

# 20 · Build vs buy

**EN**
- ✦ The classification is the point, not the vendor list. Ask: **what layer does each thing operate at?**
- **A · product harnesses** — Claude Code, GitHub Copilot, M365 Copilot — you configure them, you don't embed them
- **B · runtime SDKs** — Claude Agent SDK (Py + TS), GitHub Copilot SDK (GA Jun 2026, 6 languages), OpenAI Agents SDK — someone else's loop, driven from your code
- **C · orchestration frameworks** — LangGraph (+DeepAgents), MS Agent Framework 1.0 (GA Apr 2026, SK + AutoGen merged), Pydantic AI, CrewAI — you assemble the loop
- **D · reference implementations** — OpenClaw (steal the heartbeat + cron pattern, not the codebase — published security analyses), learn-claude-code
- Our two constraints do the filtering: **Python backend** + **banking governance**

**中文**
- ✦ 重点是这个分类，不是厂商清单。要问的是：**这东西工作在哪一层？**
- **A · 产品型 harness** —— Claude Code、GitHub Copilot、M365 Copilot —— 你配置它，不是把它嵌进来
- **B · 运行时 SDK** —— Claude Agent SDK（Python + TS）、GitHub Copilot SDK（2026 年 6 月 GA，六种语言）、OpenAI Agents SDK —— 别人的循环，由你的代码驱动
- **C · 编排框架** —— LangGraph（+DeepAgents）、微软 Agent Framework 1.0（2026 年 4 月 GA，SK 和 AutoGen 合并）、Pydantic AI、CrewAI —— 循环要你自己拼
- **D · 参考实现** —— OpenClaw（值得偷的是心跳 + cron 的设计，不是代码库 —— 已有公开的安全分析）、learn-claude-code
- 我们的两个约束自动完成筛选：**Python 后端** + **银行治理要求**

**两个可选底座 / Two credible bases**
- **Claude Agent SDK** —— 今天讲的设计几乎都已经暴露出来了：`allowed_tools`、`permission_mode`、`can_use_tool`、`HookMatcher`、`agents=`、`setting_sources`、`max_turns`、`@tool` + `create_sdk_mcp_server`。压缩和子 agent 隔离白送。代价：形态偏 Anthropic，底下跑一个 Node 运行时 —— 要过我们的部署评审
- **LangGraph** —— 如果持久状态、检查点、人在环必须是一等公民且不绑定厂商。代价：压缩阶梯、权限门、子 agent 隔离全要自己写，也就是今天设计部分的大半

**无论选哪个都复用 / Reuse either way**
- `mcp`（别的团队的 tool 免对接）· `pydantic`（tool schema 直接就是发给模型的 JSON Schema）· `anthropic`（流式、cache_control、用量核算）· Agent Skills 文件格式（方法论文档，只是约定，不是代码）

**React 那一句 / On React**
- 它们都不带 UI。都是流式吐**结构化消息事件** —— 那就是我们前端要对接的契约，也是循环必须与 UI 无关的原因

---

# 21 · The harness is moving into the API

**EN**
- ✦ Capabilities we were about to hand-build are becoming platform primitives
- Server-side compaction — replaces roughly stage 5 of our ladder; stages 1–4 still ours, and they're the cheap ones
- ⚠ Billing change: usage arrives as an `iterations` array; `usage.input_tokens` **no longer includes** the compaction pass → anyone building cost tracking must know
- Advanced tool use — Tool Search Tool (~72K → ~8.7K tokens; MCP eval 79.5% → 88.1%; doesn't break prompt caching) · **Programmatic Tool Calling** (43,588 → 27,297 avg tokens, −37%) · Tool Use Examples (72% → 90%)
- ✦ PTC is the headline for us: our tools return large tabular data, our workflows are "fetch N things, aggregate, compare against a limit"
- Packaging — MCP → Linux Foundation (Dec 2025) · Agent Plugins 1.0.0 (Aug 2026). Fine print: **no permission model, no sandboxing, no signature verification, no secrets mechanism**

**中文**
- ✦ 我们本来准备自己造的能力，正在变成平台原语
- 服务端压缩 —— 大致替代我们阶梯的第 5 级；1–4 级还是我们的，而且是便宜的那几级
- ⚠ 计费变了：用量以 `iterations` 数组返回，`usage.input_tokens` **不再包含**压缩那一趟 → 做成本核算的人必须知道
- 高级 tool 使用 —— Tool Search Tool（约 7.2 万 → 8,700 token；MCP 评测 79.5% → 88.1%；不破坏 prompt 缓存）·**程序化 tool 调用**（平均 43,588 → 27,297 token，降 37%）· Tool Use Examples（72% → 90%）
- ✦ 对我们来说重点是程序化 tool 调用：我们的 tool 返回大表格，我们的流程就是「取 N 个东西、聚合、和限额比较」—— 正是它为之设计的形状
- 打包标准化 —— MCP 捐给 Linux 基金会（2025 年 12 月）· Agent Plugins 1.0.0（2026 年 8 月）。细则要念：**没有权限模型、没有沙箱、没有签名校验、没有密钥机制**
- 中文一句话：打包约定可以采纳，信任边界得我们自己建

---

# 22 · Harness engineering became a named discipline

**EN**
- ✦ **6×** performance spread on the same benchmark from harness changes alone — model held constant
- Arc: prompt engineering (2022–24) → context engineering (2025) → harness engineering (2026)
- Meta-Harness (Stanford / MIT et al., arXiv 2603.28052, Mar 2026): an outer loop that searches over harness *code*
  - +7.7 points over SOTA context management, using 4× fewer context tokens
  - discovered harnesses beat the best hand-engineered baselines on TerminalBench-2
  - search completes in hours, produces readable transferable strategies
- ✦ Two implications: build the **eval harness alongside** the agent harness from day one; and don't over-invest in hand-tuned orchestration — the transferable asset is a clean tool surface + good evals

**中文**
- ✦ 同一个基准、同一个模型，只改 harness，性能差距 **6 倍**
- 脉络：prompt engineering（2022–24）→ context engineering（2025）→ harness engineering（2026）
- Meta-Harness（斯坦福 / MIT 等，arXiv 2603.28052，2026 年 3 月）：一个在 harness **代码**上做搜索的外层循环
  - 比当前最好的上下文管理方案高 7.7 分，同时少用 4 倍上下文 token
  - 搜出来的 harness 在 TerminalBench-2 上打过所有手工调优的基线
  - 几小时跑完，产出的是可读、可迁移的策略
- ✦ 两个推论：**评测 harness 要和 agent harness 一起建**，从第一天开始；不要在手工编排上过度投入 —— 真正能迁移的资产是干净的 tool 界面 + 好的评测

---

# 23 · Close

**EN**
- ✦ **Name the state, then cap it**
- Errors → typed messages. Continuation → a named transition. Tool safety → a declared predicate. Skill relevance → a path. Task completion → a protocol field
- And every one of them gets a bound
- The bounds, if you want them: 3 recovery attempts · 3 compaction failures · 1% of context for skills · 10% before deferring tools · 45 s to stall · 5 s to shut down

**中文**
- ✦ **先给状态起名字，再给它设上限**
- 错误 → 有类型的消息。继续 → 有名字的状态转移。tool 安全 → 声明式谓词。skill 相关性 → 一条路径。任务完成 → 协议里的一个字段
- 而且每一项都有上限
- 上限清单（需要就念）：恢复 3 次 · 压缩失败 3 次 · skill 清单占 1% 上下文 · tool 定义超 10% 就延迟加载 · 45 秒判定卡死 · 5 秒强制关闭

---

# 常见问题 / Likely questions

| 问题 | 一句话答案 |
|---|---|
| 成本会失控吗？ | 三个杠杆：prompt 缓存命中率、压缩阶梯、别把大结果塞进 context。都可量化 |
| Cost blowup? | Three levers: cache hit rate, the compaction ladder, keeping big results out of context. All measurable |
| tool 执行怎么沙箱？ | 子进程 + 权限门 + hook 审计；写操作走独立边界；book of record 硬拒绝 |
| Sandboxing? | Sub-process + permission gate + hook audit; writes are a separate boundary; hard deny on the book of record |
| 直接用现成框架不行吗？ | 可以，而且推荐 —— 但要清楚哪一层是你的。见 build vs buy 那页 |
| Why not just use a framework? | You should — but be clear which layer stays yours. See build vs buy |
| 怎么评测？ | 和 harness 一起建评测；Meta-Harness 那个 6× 就是理由 |
| How do we evaluate? | Build evals alongside. The 6× spread is the argument |
| 银行合规怎么过？ | 不可变审计日志 + 硬拒绝清单 + 人在环；memory tool 是安全评审项不是配置项 |
| Compliance? | Immutable audit log + deny-list + human in the loop; the memory tool is a security review item |
