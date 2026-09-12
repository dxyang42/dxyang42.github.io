---
title: DeepSeek-V4.1-Flash 精读（一）：百万上下文真正贵在哪里
date: 2026-09-12 20:49:00
tags: [DeepSeek, 大模型, 长上下文, KV Cache]
categories: [人工智能]
description: 从 prefill、运行时 KV、持久 KV 与数据搬运四条成本线，拆解百万上下文真正昂贵的地方。
cover: /img/deepseek-v41/cover-01.webp
top_img: /img/deepseek-v41/cover-01.webp
---

“支持 1M 上下文”，听起来就是模型能一次塞进去一百万个 token。

但塞进去只是第一步。算一次要多少钱？状态放在哪里？下一轮怎么取？取出来还要搬多远？到了工程侧，这几个问题一个都躲不过。

DeepSeek-V4.1-Flash 技术报告把长上下文当成了一笔系统账。Attention FLOPs 当然要算，但账单不止这一项。Prefill 的计算、在线推理时常驻的 global KV、为了前缀复用而长期保存的 persistent KV，还有 HBM、主机内存、SSD 和互连带宽，都得付钱。

这一篇先把账摊开。至于每一项怎么压，后面几篇再聊。

## 系列目录

1. [DeepSeek-V4.1-Flash 精读（一）：百万上下文真正贵在哪里](/2026/09/12/DeepSeek-V4.1-Flash精读一：百万上下文真正贵在哪里/)
2. [DeepSeek-V4.1-Flash 精读（二）：CED 和 CSA2 如何重做长上下文架构](/2026/09/12/DeepSeek-V4.1-Flash精读二：CED和CSA2如何重做长上下文架构/)
3. [DeepSeek-V4.1-Flash 精读（三）：把持久 KV 缓存压到八分之一](/2026/09/12/DeepSeek-V4.1-Flash精读三：把持久KV缓存压到八分之一/)
4. [DeepSeek-V4.1-Flash 精读（四）：没有新 RL 算法，Agent 为什么变强](/2026/09/12/DeepSeek-V4.1-Flash精读四：没有新RL算法，Agent为何变强/)
5. [DeepSeek-V4.1-Flash 精读（五）：成绩、推理旋钮与真实边界](/2026/09/12/DeepSeek-V4.1-Flash精读五：成绩、推理旋钮与真实边界/)

## 先分清两种 KV

报告里反复出现两个词：runtime KV 和 persistent KV。名字很像，花钱的地方却不一样。

Runtime KV 是当前请求运行时要用的缓存。DeepSeek-V4 的注意力分成两路：global attention 看完整上下文，Sliding-Window Attention（SWA）只看固定窗口。SWA 窗口固定，缓存不会跟着整段序列一直长。上下文足够长之后，占大头的是 global KV，包括 main KV 和用于索引的 indexer K。

请求还在跑，这些缓存就要随时参加计算，所以主要受 HBM 容量限制。HBM 很快，也很贵，空间更谈不上宽裕。上下文从几千拉到一百万 token，每个 token 哪怕只占一点，乘上一百万也很可观。

Persistent KV 管的是以后。为了下次不用重算某段前缀，系统会把部分 KV 长期保存下来。它不必一直待在显存里，通常放在 SSD 或主机内存；等请求命中这段前缀，再加载并迁移到计算侧。于是它既吃存储容量，也吃 I/O 和互连带宽。

可以先这么记：**runtime KV 管当前计算，persistent KV 管前缀复用。** 前者更容易卡 HBM，后者更容易卡 SSD 和主机内存，最后又都会碰到数据搬运。

所以一句“KV Cache 降了多少”，信息其实不够。显存省下来了，持久化容量未必同步下降；SSD 少存一点，也可能要靠更多 prefill 重算补回来。账只是从一个口袋挪到了另一个口袋。

## 第一笔钱：Prefill

长上下文请求开始生成之前，要先做 prefill：把整段输入读一遍，建立后续生成要用的状态。

Agent 的负载越来越偏向“大量读、少量写”。历史对话、工具返回、代码、图片、文档一路往上下文里堆，最后输出也许只有一小段。Decode 每个 token 再省，第一次吞下一百万 token 也不会免费。报告直接把 prefill 称为计算昂贵的环节。

DeepSeek-V4.1-Flash 用 Causal Encoder-Decoder（CED）来处理这笔开销：prefill 每个 token 激活 8B 参数，decode 每个 token 激活 16B 参数。

这里很容易被总参数量带跑。模型有 552B backbone 参数，另外还有 196B Engram 参数，但 MoE 不会让每个 token 跑遍全部参数。部署时更该看的，是不同阶段实际激活了多少。把输入最重的 prefill 压到 8B，正好对着长时程 Agent 的负载下手。

![DeepSeek-V4.1-Flash 的 Agent 基准成绩与历代全局 KV Cache 对比](/img/deepseek-v41/fig01-overview.png)

*来源：DeepSeek-V4.1-Flash 技术报告*

## 第二笔钱：Global KV 一直占着 HBM

Attention 做稀疏以后，会出现一个挺有意思的变化：计算压力降了，存储压力反倒更扎眼。

全局分支还要从百万 token 里找信息，就得保留足够的全局状态。它可以不对每个 token 做完整的两两 attention，但 global KV 仍然会随着序列变长。只要请求还活着，这些状态就占着高价值的 HBM。长请求一多，并发往往先撞上缓存容量，而不是别的东西。

V4.1-Flash 把 global KV 压到 **890 bytes/token**，约为 V4-Flash 的 **1/4**。按线性关系直算，一百万 token 的 global KV 大约是 890 MB。比前代小了不少，但 890 MB 仍是实打实的高速存储占用。1M 当然不是在界面上多开一个选项那么轻松。

我读到这里第一反应是，部署者可能更该盯住 890 bytes/token，而不是“支持 1M”这句宣传。窗口上限告诉你最长能放多少，每 token 缓存量才方便拿来规划容量：一块 HBM 能塞几个请求，要给权重和运行时留多少，并发还能不能继续加。

## 第三笔钱：Persistent KV 一路延伸到 SSD

长时程 Agent 很少只跑一次。系统提示可能重复，知识前缀可能固定，已经完成的长轨迹也可能在后续请求里继续用。KV 到这里不再是一次请求结束就丢的临时状态，而是要持续管理的东西。

容量压力也随之从 HBM 扩到 SSD 和主机内存。单份缓存越大，能留下的前缀越少。就算存得下，加载速度跟不上，GPU 还是得干等。难怪报告会把 persistent KV、I/O 和互连带宽放在一起谈，只数 attention 做了多少乘加，多少有点自欺欺人。

V4.1-Flash 的 persistent KV 约为 V4-Flash 的 **1/8**。它没有走“全部保存”或“全部重算”这种一刀切路线，而是只持久化更值得留下的部分。SWA 状态则采用 bounded replay：需要时只重放最近一个窗口的 token，近似重建缺失状态。多花一点 prefill 重算，换掉一大块长期存储。

这笔交换贯穿整套长上下文设计。少存一点，可能就要多算一点；压缩更激进，精度可能受影响；缓存放到更远、更便宜的介质，搬运时间又会冒出来。计算、存储、带宽，说到底是在三个账户之间调预算。

## 第四笔钱：Decode FLOPs 没暴涨，账也没消失

报告的 Figure 2 很抢眼。上下文从 4K 扩到 1M，长度增加 256 倍，V4.1-Flash 的单 token decode FLOPs 只增加约 **25%**。图里的计算量还按精度加权：BF16、FP8、FP4 分别按 1、0.5、0.25 计。

![不同上下文长度下的单 token Decode FLOPs](/img/deepseek-v41/fig02-decode-flops.png)

*来源：DeepSeek-V4.1-Flash 技术报告*

这个结果能说明稀疏和压缩设计有效，decode 没有跟着上下文长度近似线性爆炸。但“百万上下文只贵 25%”就读过头了。25% 说的是单 token decode 的加权计算量，不包括首次 prefill 吞下超长输入的全部代价，也不代表 KV 占了多少 HBM、persistent KV 占了多少 SSD，以及缓存迁移花了多久。

FLOPs 曲线回答的是“下一个 token 要多算多少”。它没有回答前面那一百万 token 的状态放哪儿、留多久、什么时候搬过来。线上跑起来，后面几个问题照样可能卡住吞吐。

## 带宽这笔账，很容易被漏掉

缓存离开计算设备，容量问题马上会变成搬运问题。Persistent KV 放在 SSD 或主机内存，确实比常驻 HBM 便宜。等某次请求重新命中前缀，数据还是要加载、迁移。报告也明确提到，I/O 和互连带宽会限制这个过程。

“已经缓存”不代表“现在就能用”。迁移追不上计算，设备就等；一批长请求同时恢复，数据通道还要互相抢。单 token 的理论 FLOPs 可以很好看，端到端延迟和吞吐却可能卡在最慢的那条数据路径上。

报告给出的处理方式很有代表性：长生命周期的 global KV 和短生命周期的 encoder SWA KV 分开管理，缺失的 SWA 状态再用 bounded replay 近似重建。我的理解是，系统在按寿命和复用价值给状态分层。哪些值得常驻，哪些适合落盘，哪些重算更划算，都不能只靠模型结构回答。

## 把四笔账放在一起

把报告里的成本项排在一起，大概是这样：

> 百万上下文成本 = Prefill 计算 + Runtime KV 的 HBM 占用 + Persistent KV 的存储占用 + KV 加载与迁移的带宽开销。

漏掉任何一项，1M 都可能停留在“单请求可以跑”，距离高并发、可复用、成本可控还有一截。

V4.1-Flash 也没有指望靠一个点解决全部问题。CED 降低 prefill 的激活量；CSA2 复用跨层 global KV 和索引；FP4 继续压 global KV；SWA Bounded Replay 用有限重算换持久化空间；推理系统再处理缓存分层、通信计算重叠和内核融合。几层一起动，才有了这张完整的账。

后续几篇会沿着这条线拆。第二篇先看 CED：为什么同一个模型在 prefill 激活 8B、decode 激活 16B，以及这种不对称为什么适合输入重的 Agent。再往后是 CSA2、FP4 和 SWA Bounded Replay，看它们分别怎么压运行时 KV 和持久 KV。

最后还得留个边界：**支持 1M，不等于所有百万 token 任务都可靠。** 技术报告说明模型具备这一上下文容量，也给出了相应的成本和系统设计。但具体任务能不能从百万 token 里稳定找对信息、完成推理、维持多步行为，仍取决于任务结构和实际评测。窗口够长，不能直接推出答案就一定够好。
