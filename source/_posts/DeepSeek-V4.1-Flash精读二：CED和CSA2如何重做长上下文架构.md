---
title: DeepSeek-V4.1-Flash 精读（二）：CED 和 CSA2 如何重做长上下文架构
date: 2026-09-12 20:48:00
tags: [DeepSeek, Transformer, 稀疏注意力, KV Cache]
categories: [人工智能]
description: 从 CED 的半程 prefill 到 CSA2 的跨层 KV 与索引复用，拆解 DeepSeek-V4.1-Flash 如何同时压低长上下文计算、显存和检索成本。
cover: /img/deepseek-v41/cover-02.webp
top_img: /img/deepseek-v41/cover-02.webp
---

长上下文到底贵在哪里？

第一反应通常是注意力计算量。但在 Agent 场景里，问题至少有三层：工具调用不断把新结果塞回上下文，缓存一旦没命中，就要重新 prefill；上下文越长，KV Cache 越占 HBM；即使 KV 已经压缩，稀疏注意力还得从很长的历史里找 Top-K，索引本身也会越来越贵。

DeepSeek-V4.1-Flash 的答案不是再加一个孤立技巧，而是把问题拆开：**CED 主要砍 prefill 计算，CSA2 主要砍跨层 KV 存储和索引计算。** 两者拼起来，才是这代长上下文架构真正值得看的地方。

## 系列目录

1. [DeepSeek-V4.1-Flash 精读（一）：百万上下文真正贵在哪里](/2026/09/12/DeepSeek-V4.1-Flash精读一：百万上下文真正贵在哪里/)
2. [DeepSeek-V4.1-Flash 精读（二）：CED 和 CSA2 如何重做长上下文架构](/2026/09/12/DeepSeek-V4.1-Flash精读二：CED和CSA2如何重做长上下文架构/)
3. [DeepSeek-V4.1-Flash 精读（三）：把持久 KV 缓存压到八分之一](/2026/09/12/DeepSeek-V4.1-Flash精读三：把持久KV缓存压到八分之一/)
4. [DeepSeek-V4.1-Flash 精读（四）：没有新 RL 算法，Agent 为什么变强](/2026/09/12/DeepSeek-V4.1-Flash精读四：没有新RL算法，Agent为何变强/)
5. [DeepSeek-V4.1-Flash 精读（五）：成绩、推理旋钮与真实边界](/2026/09/12/DeepSeek-V4.1-Flash精读五：成绩、推理旋钮与真实边界/)

## CED 为什么不能理解成传统 Encoder-Decoder？

先看总图。语言骨干仍然是 40 层 causal Transformer，只是被切成前 20 层 causal encoder 和后 20 层 decoder。

![DeepSeek-V4.1-Flash 总体架构](/img/deepseek-v41/fig03-architecture.png)

*图源：DeepSeek-V4.1-Flash Technical Report，Figure 3。*

这里最容易产生误会：既然叫 Encoder-Decoder，是不是像机器翻译那样，encoder 双向读“源句子”，decoder 再对“目标句子”做 cross-attention？不是。

CED 的 encoder 仍然是**因果的**。每个位置只能看见它左边的内容，整个模型仍在同一条序列上做自回归生成。它没有把输入和输出拆成两套语义空间，也不是传统 seq2seq。更准确的理解是：DeepSeek 把一条 40 层因果网络沿深度切成上下两段，并重新规定了全局 KV 在两段之间怎么产生。

问题来了：普通 Transformer 在 prefill 时，提示词中的每个 token 都要跑完 40 层。长上下文 Agent 每调用一次工具，新增内容又可能触发一轮大 prefill，这笔账非常重。

CED 的做法是让 decoder 的**全局 KV**不再由每个 decoder 层自己的隐藏状态生成。对第 20 层输出，也就是最终 encoder states，decoder 各层使用自己的投影参数，直接生成该层需要的 main KV，以及相应的压缩权重。于是，同一份固定的 encoder states 可以为 20 个 decoder 层提供各自不同的全局 KV。

所以在主干意义上，prompt prefill 只需要完整跑前 20 层 encoder。进入 decode 后，历史 prompt 对应的 encoder states 已经固定，decoder 用这些状态投影出的全局 KV，再结合当前生成 token 逐层计算。它省掉的不是“20 层模型”，而是**让绝大多数 prompt token 不必完整穿过后 20 层 decoder**。

论文给出的复杂度很直观。普通 prefill 近似是：

$$O(NL)$$

CED 则变成：

$$O(NL/2+n_{win}L/2)\approx O(NL/2),\quad N\gg n_{win}$$

为什么后面还有一项？因为 CED 没把所有东西都跨层冻结。

## 全局上下文可以借，局部上下文不能偷懒

V4.1-Flash 的注意力可以看成两条支路：一条是跨越长历史的 global branch，另一条是只看附近 token 的 Sliding-Window Attention，也就是 SWA。

CED 改写的是前者。decoder 的全局 KV 可以从最后一层 encoder states 投影出来。但 SWA KV 仍然是 layer-local 的：每一层都要基于该层自己的隐藏状态生成，不能拿第 20 层的结果替代第 37 层。

这也是“prefill 只跑 encoder”需要加的一句限定：**长 prompt 的主体只跑 encoder；为了恢复 decoder 各层的局部状态，还要对最后 $n_{win}$ 个 token 做 Decoder SWA Bounded Replay。** 它不是把 decoder 在 prefill 阶段彻底删除，而是把 decoder 工作量限制在固定窗口，不再随完整上下文长度一起增长。

我认为这是 CED 最聪明的取舍。全局分支负责“远处有什么”，允许复用；局部分支负责“最近几步怎么演化”，保留逐层深度。它没有为了省计算，把两种信息处理方式一起压扁。

把一次请求按时间展开，会更容易理解。假设 Agent 已经积累了一段很长的对话，又收到一份工具返回。新增和未缓存的 prompt token 先经过 20 层 causal encoder，得到第 20 层隐藏状态。各 decoder 层再用各自的投影，把这批状态变成自己需要的全局 KV；局部窗口则通过有限 replay 补齐。开始生成后，新 token 仍然依次通过完整 decoder，使用自己的 query 读取那批由 encoder states 建好的历史全局 KV，并持续更新本层 SWA KV。

因此，“decode 利用固定 encoder states”不等于生成阶段只算一次 decoder，也不等于 decoder 的隐藏状态被冻结。固定的是历史 prompt 的 encoder 表示及其所提供的全局记忆底座；每个新 token 在 decoder 中的状态、Main Q、局部 KV 和注意力输出仍然逐层变化。CED 节省的是把历史 prompt 反复送过 decoder 的成本，不是取消 decoder 对新 token 的深层推理。

这也解释了为什么它特别适合输入重、输出相对短的 Agent 请求。如果每一轮都塞进大量工具日志，却只生成少量动作，省下 prompt 的 20 层计算很划算；反过来，如果任务几乎没有输入、只做超长生成，prefill 优势在总成本中的占比自然会下降。

## CSA2 又在压什么？

CED 解决了 prompt 要跑多少层，但长上下文还要常驻大量缓存。

CSA2 把注意力所需的状态明确拆成两类：

- **Main KV**：承载压缩后的全局历史，供稀疏检索和核心注意力读取；
- **SWA KV**：每层自己的局部窗口状态，负责最近上下文。

每个 query 先由轻量 indexer 在 Main KV 中选出 Top-K，再把选中的 Main KV 与本层 SWA KV 拼起来做核心注意力。也就是说，Main KV 负责“远而少”，SWA KV 负责“近而全”。

CSA2 相比上一代 CSA 还简化了压缩器：压缩率为 $m$ 时，每 $m$ 个 token 形成一个 Main KV entry，不再用相邻压缩项之间重叠的 $2m$ 个输入，也去掉了压缩时的绝对位置编码；Indexer K 则直接从 Main KV 投影，不再另走一条从隐藏状态出发的压缩路径。

但真正关键的不是这些局部简化，而是它把“共享缓存”和“复用选择结果”拆成了两件事。

## Full、Reindex、Reuse：不要把两种复用混在一起

![CSA2 的三种工作模式](/img/deepseek-v41/fig04-csa2-modes.png)

*图源：DeepSeek-V4.1-Flash Technical Report，Figure 4。*

CSA2 的每一层会被静态指定为三种模式之一。

**Full Mode** 什么都自己做。它生成本层 Main KV，计算 Indexer Q，并由 Main KV 投影 Indexer K，然后对可见历史打分，产出新的 Top-K。它既是新的缓存起点，也是新的索引起点。

**Reindex Mode** 不生成新的 Main KV，也不生成新的 Indexer K，而是沿用前面最近一个 Full 层的那一套缓存。但它会计算自己的 Indexer Q，重新给共享的 Indexer K 打分，得到本层新的 Top-K。

**Reuse Mode** 更进一步。它复用最近可用的 Main KV，也复用针对这份 Main KV 最近一次算出的 Top-K，来源可以是 Full，也可以是 Reindex。它不计算 Indexer Q，不重新评分，直接拿这个选择做注意力。

三种模式仍然都会计算本层自己的 Main Q 和 SWA KV。

这里必须分清两个概念。

**共享 Main KV / Indexer K**，省的是缓存：多个层不再各存一份全局状态。**复用 Top-K indices**，省的是索引计算：这一层不再重新问“该看历史里的哪些位置”。Reindex 正好证明两者可以解耦——它共享缓存，却保留自己的选择；Reuse 才连选择结果一起拿来。

如果把 CSA2 只总结成“跨层共享 KV”，就漏掉了一半；如果只说“跨层复用 Top-K”，又解释不了缓存为什么会变小。

## 890 bytes/token 是三条轴一起乘出来的

论文报告，V4.1-Flash 的 global KV 常驻 HBM 开销降到 **890 bytes/token**，约为 V4-Flash 的四分之一。这个数字不该被理解成某一个神奇量化格式的功劳。

对长上下文占主导的全局缓存，可以用下面这个结构理解，而不要把它误写成三个百分比相加：

$$
\text{bytes/token}\propto
\underbrace{\frac{1}{m}}_{\text{序列压缩}}
\times
\underbrace{\frac{L}{s}}_{\text{独立 KV 组数}}
\times
\underbrace{b}_{\text{每元素字节数}}
$$

这里 $m$ 是每个 Main KV entry 汇聚的 token 数；$s$ 表示相邻 Full 缓存起点的大致共享间隔，间隔越大，需要独立保存的 Main KV / Indexer K 组越少；$b$ 是缓存精度对应的字节成本。FP4 把 main KV 从 FP8 的约 1 byte/element 降到约 0.5 byte/element，实际格式还包含分组 scale，不能粗暴宣称“任何缓存都严格再除以二”。Indexer K 在 V4 已使用 FP4，SWA KV 也仍保留 FP8，因此 890 是具体架构配置下的最终值，不是把 $1/m$、$1/s$、$1/2$ 随手一乘就能反推出来的万能公式。

我更愿意把这三条轴叫作：**少存位置、少存层、每份少占字节。** CSA2 覆盖前两条，FP4 补上第三条。

## Top-K 少算几次还不够，搜索范围也得缩

即使 Reuse 层完全不跑 indexer，Full 和 Reindex 层仍可能面对百万 token。只要每次都扫描全部因果可见位置，剩下的索引仍会线性增长。

![Hierarchical Sparse Indexer](/img/deepseek-v41/fig05-hierarchical-indexer.png)

*图源：DeepSeek-V4.1-Flash Technical Report，Figure 5。*

Hierarchical Sparse Indexer 的做法分两步。

Decoder 中第一个 Full Mode 层仍然扫描完整可见范围，选出自己最终注意力使用的 Top-512。同时，它按 block 聚合：每个 block 取其中最大的 index score，再选高分 block，组成一个比 Top-K 更大的 candidate pool。论文示例是 2,048 个 block、每块 8 个位置，也就是 16,384 个候选位置。

后续 Reindex 层不再扫描完整上下文，只在这 16,384 个候选里各自重排并选自己的 Top-512。Reuse 层照旧不索引，直接沿用最近的 Top-K。这样，共享的是候选池，各 Reindex 层的最终选择仍可以不同。

必须强调：**Hierarchical Sparse Indexer 只用于 CED 的 decoder。** 第一层 Full 依然要做一次全范围扫描，它不是让所有索引都变成常数成本；它让后续 indexer 在候选池大小固定时，单 query 的评分量不再随上下文长度增长。并且这套候选限制在 post-training 和推理中一致使用，不是部署时临时硬剪。

## 我的判断：这不是单点压缩，而是重新分配责任

CED 和 CSA2 放在一起看，逻辑其实很统一。

CED 问的是：历史 token 的深层计算，真的要在 prefill 时逐层重做吗？答案是，全局状态可以由固定 encoder states 提供，decoder 只保留生成时需要的深层处理与局部 replay。

CSA2 问的是：每一层真的都要拥有独立的全局 KV，并重新做一次稀疏选择吗？答案是，缓存可以共享，选择可以按需更新，连更新时的搜索域也可以逐级缩小。

代价也很明确：系统开始依赖“固定的 encoder states 足够支撑 decoder 全局 KV”“浅层候选池不会漏掉深层真正需要的位置”“有限 SWA replay 足够恢复局部状态”这些结构性假设。论文报告性能可与基线相当，但这些边界仍值得在超长、多轮、频繁工具调用的真实 Agent 负载里继续观察。

不过从工程方向看，我认为它比单纯追求更激进的稀疏率更有启发。长上下文的成本不是一个数字，而是 prefill、HBM、SSD、带宽和索引计算共同形成的链条。CED 与 CSA2 的价值，正是把这条链上的责任重新分配了一遍。

下一篇继续看 SWA Bounded Replay：为什么只重放一个窗口，就能把持久化 KV 再压到 V4-Flash 的约八分之一。
