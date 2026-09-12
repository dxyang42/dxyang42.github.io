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

很多人先想到注意力计算量。放到 Agent 场景里，账还要再算细一点。工具调用会不断把新结果塞回上下文，缓存没命中，就得重新 prefill。上下文越长，KV Cache 占掉的 HBM 越多。就算 KV 已经压缩，稀疏注意力还要从漫长的历史里找 Top-K，索引也会跟着变贵。

DeepSeek-V4.1-Flash 没指望一个技巧包办这些问题。CED 处理 prefill 计算，CSA2 处理跨层 KV 存储和索引计算。两套设计接在一起，才构成这次长上下文架构的主要变化。

## 系列目录

1. [DeepSeek-V4.1-Flash 精读（一）：百万上下文真正贵在哪里](/2026/09/12/DeepSeek-V4.1-Flash精读一：百万上下文真正贵在哪里/)
2. [DeepSeek-V4.1-Flash 精读（二）：CED 和 CSA2 如何重做长上下文架构](/2026/09/12/DeepSeek-V4.1-Flash精读二：CED和CSA2如何重做长上下文架构/)
3. [DeepSeek-V4.1-Flash 精读（三）：把持久 KV 缓存压到八分之一](/2026/09/12/DeepSeek-V4.1-Flash精读三：把持久KV缓存压到八分之一/)
4. [DeepSeek-V4.1-Flash 精读（四）：没有新 RL 算法，Agent 为什么变强](/2026/09/12/DeepSeek-V4.1-Flash精读四：没有新RL算法，Agent为何变强/)
5. [DeepSeek-V4.1-Flash 精读（五）：成绩、推理旋钮与真实边界](/2026/09/12/DeepSeek-V4.1-Flash精读五：成绩、推理旋钮与真实边界/)

## CED 不是传统的 Encoder-Decoder

先看总图。语言骨干还是 40 层 causal Transformer，只是从中间切开：前 20 层是 causal encoder，后 20 层是 decoder，也就是 20+20。

![DeepSeek-V4.1-Flash 总体架构](/img/deepseek-v41/fig03-architecture.png)

*图源：DeepSeek-V4.1-Flash Technical Report，Figure 3。*

Encoder-Decoder 这个名字很容易让人想到机器翻译：encoder 双向读取“源句子”，decoder 再对“目标句子”做 cross-attention。CED 不是这套结构。它的 encoder 仍然遵守因果掩码，每个位置只能看左边，模型也始终在同一条序列上自回归生成。

可以把它理解成一条 40 层因果网络沿深度切成两段，然后重新安排全局 KV 的生产方式。普通 Transformer 做 prefill 时，prompt 里的每个 token 都要走完 40 层。长上下文 Agent 调一次工具，新增内容又可能带来一轮大 prefill，这部分计算很重。

CED 让 decoder 的全局 KV 不再从各层自己的隐藏状态生成。第 20 层的输出，也就是最终 encoder states，会被交给后面的 decoder。每个 decoder 层都有自己的投影参数，可以从同一份 encoder states 生成本层需要的 Main KV 和对应的压缩权重。输入虽然相同，20 个 decoder 层拿到的全局 KV 仍然可以不同。

这样一来，prompt prefill 的主体只用完整跑过前 20 层 encoder。进入 decode 后，历史 prompt 的 encoder states 已经固定。decoder 从这些状态投影全局 KV，再结合当前生成 token 逐层计算。

这里省下的不是后半个模型。准确地说，是大部分 prompt token 不用再完整穿过后 20 层 decoder。报告给出的计算口径是 prefill 约 8B、decode 约 16B；两者对应的执行阶段不同，不能把它理解成整个模型始终只激活 8B 参数。

论文给出的复杂度也很直观。普通 prefill 近似为：

$$O(NL)$$

CED 变成：

$$O(NL/2+n_{win}L/2)\approx O(NL/2),\quad N\gg n_{win}$$

后面为什么还留着一项？因为 decoder 里还有一部分状态不能直接从 encoder 借来。

## 全局上下文能复用，局部窗口还得逐层算

V4.1-Flash 的注意力有两条支路。一条是跨越长历史的 global branch，另一条只看附近 token，也就是 Sliding-Window Attention（SWA）。

CED 改的是全局分支。decoder 的全局 KV 可以从最后一层 encoder states 投影出来，SWA KV 仍然属于各层。第 37 层的局部状态，要基于第 37 层自己的隐藏状态生成，不能拿第 20 层的结果直接顶上。

所以“prefill 只跑 encoder”要带一个限定：长 prompt 的主体只跑 encoder，但最后 $n_{win}$ 个 token 还要做 Decoder SWA Bounded Replay，用来补回 decoder 各层的局部状态。decoder 没有从 prefill 阶段消失，它的工作量被压在一个固定窗口里，不再跟完整上下文一起增长。

这个取舍挺干净。全局分支回答“远处有哪些内容”，适合复用；局部分支保留最近几步的逐层变化。省计算的同时，没有把两种信息处理方式一起压平。

按一次 Agent 请求顺下来会更好懂。假设对话已经很长，又收到一份工具返回。新增且没有缓存的 prompt token 先走 20 层 causal encoder，得到第 20 层隐藏状态。各 decoder 层用自己的投影，把这批状态变成本层的全局 KV。局部窗口则靠有限 replay 补齐。

开始生成后，每个新 token 仍然依次经过完整 decoder。它用自己的 query 读取从 encoder states 建好的历史全局 KV，同时持续更新本层 SWA KV。

“decode 使用固定 encoder states”只描述历史底座。固定的是历史 prompt 的 encoder 表示，以及它提供的全局记忆。新 token 在 decoder 中的隐藏状态、Main Q、局部 KV 和注意力输出，依旧会一层层变化。CED 少算的是历史 prompt 反复穿过 decoder 的成本，decoder 对新 token 的深层处理还在。

这也说明了它更适合输入重、输出相对短的 Agent 请求。每轮塞进大量工具日志，只生成少量动作，prompt 少走 20 层就很划算。任务如果几乎没有输入，只做很长的连续生成，prefill 优势在总成本里的占比自然会降低。

## CSA2 接着处理缓存

CED 缩短了 prompt 要走的路径，长上下文仍有大量状态需要常驻。

CSA2 把注意力状态分成两类：

- Main KV：压缩后的全局历史，供稀疏检索和主注意力读取；
- SWA KV：每层自己的局部窗口状态，负责最近上下文。

每个 query 先由轻量 indexer 在 Main KV 中挑出 Top-K，再把选中的 Main KV 和本层 SWA KV 拼在一起做注意力。简单说，Main KV 管“远而少”，SWA KV 管“近而全”。

和上一代 CSA 相比，CSA2 还把压缩器做得更简单。压缩率为 $m$ 时，每 $m$ 个 token 形成一个 Main KV entry。它不再用相邻压缩项之间重叠的 $2m$ 个输入，也去掉了压缩阶段的绝对位置编码。Indexer K 直接从 Main KV 投影，不再单独走一条从隐藏状态开始的压缩路径。

这些局部变化之外，CSA2 还拆开了两种容易混淆的复用：缓存可以跨层共享，Top-K 的选择结果也可以跨层复用。它们不是同一件事。

## Full、Reindex、Reuse 各自省什么

![CSA2 的三种工作模式](/img/deepseek-v41/fig04-csa2-modes.png)

*图源：DeepSeek-V4.1-Flash Technical Report，Figure 4。*

CSA2 会把每一层静态指定为 Full、Reindex、Reuse 三种模式之一。

Full Mode 所有步骤都自己完成。它生成本层 Main KV，计算 Indexer Q，再从 Main KV 投影 Indexer K，对可见历史打分并产出新的 Top-K。后面的层可以从这里开始共享缓存，也可以复用这次索引。

Reindex Mode 沿用前面最近一个 Full 层的 Main KV 和 Indexer K，不再生成一套新的缓存。不过它有自己的 Indexer Q，会重新给共享的 Indexer K 打分，因此可以得到本层自己的 Top-K。

Reuse Mode 连这次重排也省了。它复用最近可用的 Main KV，也复用针对这份 Main KV 最近算出的 Top-K。这个 Top-K 可以来自 Full，也可以来自 Reindex。Reuse 不计算 Indexer Q，不重新评分，直接用已有选择做注意力。

三种模式都会计算本层自己的 Main Q 和 SWA KV，这部分没有跨层拿走。

把账分开看就清楚了。共享 Main KV / Indexer K，减少的是缓存，多层不用各存一份全局状态。复用 Top-K indices，减少的是索引计算，这一层不用再判断该看历史中的哪些位置。

Reindex 正好卡在两者中间：它共享缓存，但保留本层的选择。到了 Reuse，缓存和选择结果才一起沿用。只用“跨层共享 KV”概括 CSA2，会漏掉索引复用；只提“跨层复用 Top-K”，又解释不了缓存为什么缩小。

## 890 bytes/token 怎么来的

论文报告，V4.1-Flash 的 global KV 常驻 HBM 开销降到 890 bytes/token，约为 V4-Flash 的四分之一。这个结果来自几条轴一起缩减，不能归到某一种量化格式头上。

对长上下文占主导的全局缓存，可以先用下面的结构理解：

$$
\text{bytes/token}\propto
\underbrace{\frac{1}{m}}_{\text{序列压缩}}
\times
\underbrace{\frac{L}{s}}_{\text{独立 KV 组数}}
\times
\underbrace{b}_{\text{每元素字节数}}
$$

$m$ 是每个 Main KV entry 汇聚的 token 数。$s$ 表示相邻 Full 缓存起点的大致共享间隔；间隔越大，要独立保存的 Main KV / Indexer K 组就越少。$b$ 是缓存精度对应的字节成本。

FP4 把 Main KV 从 FP8 的约 1 byte/element 降到约 0.5 byte/element。不过实际格式还带有分组 scale，不能直接说“任何缓存都严格减半”。Indexer K 在 V4 里已经使用 FP4，SWA KV 也仍然保留 FP8。因此，890 bytes/token 是具体架构配置算出的最终值，不能拿 $1/m$、$1/s$、$1/2$ 随手相乘，再当成通用公式。

这三条轴可以记成一句话：少存位置，少存层，每份少占字节。CSA2 覆盖前两条，FP4 补上第三条。

## Top-K 少算几层，搜索范围还是太大

Reuse 层可以完全不跑 indexer，但 Full 和 Reindex 层面对的仍可能是百万 token。只要每次都扫过全部因果可见位置，留下来的索引计算还是会随上下文线性增长。

![Hierarchical Sparse Indexer](/img/deepseek-v41/fig05-hierarchical-indexer.png)

*图源：DeepSeek-V4.1-Flash Technical Report，Figure 5。*

Hierarchical Sparse Indexer 用了两步。

decoder 里的第一个 Full Mode 层仍然扫描完整可见范围，选出最终用于注意力的 Top-512。同时，它会按 block 聚合：每个 block 取最大的 index score，再挑出高分 block，组成一个比 Top-K 更大的 candidate pool。论文示例用了 2,048 个 block，每块 8 个位置，一共 16,384 个候选位置。

后面的 Reindex 层不再扫描完整上下文。它们只在这 16,384 个候选里各自重排，再选出自己的 Top-512。Reuse 层照旧不做索引，直接沿用最近一次 Top-K。大家共享的是候选池，各个 Reindex 层最后看到的位置仍然可以不同。

这里有三条限制要记住。第一，Hierarchical Sparse Indexer 只用在 CED 的 decoder。第二，第一个 Full 层依然要完成一次全范围扫描，所以并非所有索引都变成常数成本。它做到的是：候选池大小固定后，后续 indexer 对单个 query 的评分量不再随上下文长度增长。第三，这套候选限制在 post-training 和推理阶段保持一致，不是部署时临时硬剪出来的。

## 把几笔账放回一张图里

CED 和 CSA2 处理的是同一条成本链上的不同位置。

CED 先看历史 token 的深层计算。全局状态由固定的 encoder states 提供，decoder 留下生成阶段的深层处理，以及固定窗口内的局部 replay。这样，长 prompt 的主体从 40 层缩到前 20 层。

CSA2 再看每层的全局状态和稀疏选择。Main KV 可以跨层共享，Top-K 可以按层重新选择，也可以直接复用。需要重选时，后续层还可以只在固定候选池里搜索。

代价同样清楚。这套设计依赖几个结构性假设：固定 encoder states 足以支撑 decoder 的全局 KV；浅层给出的候选池不会漏掉深层真正需要的位置；有限的 SWA replay 足以恢复局部状态。论文报告的性能与基线相当，但在超长、多轮、频繁工具调用的真实 Agent 负载里，这些边界仍值得继续观察。

从工程角度看，这比单独追求更高稀疏率更有意思。长上下文成本分散在 prefill、HBM、SSD、带宽和索引计算上。CED 与 CSA2 做的事情，是重新安排每一段由谁计算、保存和复用。

下一篇继续看 SWA Bounded Replay：为什么只重放一个窗口，就能把持久化 KV 再压到 V4-Flash 的约八分之一。
