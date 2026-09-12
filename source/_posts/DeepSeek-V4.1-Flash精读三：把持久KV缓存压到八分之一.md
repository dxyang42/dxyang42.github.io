---
title: DeepSeek-V4.1-Flash 精读（三）：把持久 KV 缓存压到八分之一
date: 2026-09-12 20:47:00
tags:
  - DeepSeek
  - 推理系统
  - KV Cache
  - FP4
categories:
  - 人工智能
description: DeepSeek-V4.1-Flash 如何用 FP4、跨层复用和 SWA Bounded Replay，把 global KV 压到 V4-Flash 的约四分之一，并把持久 KV 进一步压到约八分之一。
cover: /img/deepseek-v41/cover-03.webp
top_img: /img/deepseek-v41/cover-03.webp
---

看到“KV 缓存减少 8 倍”，很容易以为 DeepSeek 只是把原来的 KV 全换成低精度，然后文件大小直接除以八。

这笔账没这么简单。

DeepSeek-V4.1-Flash 先改模型结构和数值精度，把 global KV 压到 V4-Flash 的约 1/4。到了部署环节，它又把短命的 SWA KV 移出持久缓存。两步叠加，persistent KV 才降到约 1/8。

1/4 和 1/8 不是同一个口径。前者主要关系到运行时常驻 HBM 的全局缓存，后者说的是为了前缀复用，长期放在 SSD 或主机内存里的持久缓存。不能把它们合并成一句“FP4 带来 8 倍压缩”。FP4 没有这么大的功劳。

## 系列目录

1. [DeepSeek-V4.1-Flash 精读（一）：百万上下文真正贵在哪里](/2026/09/12/DeepSeek-V4.1-Flash精读一：百万上下文真正贵在哪里/)
2. [DeepSeek-V4.1-Flash 精读（二）：CED 和 CSA2 如何重做长上下文架构](/2026/09/12/DeepSeek-V4.1-Flash精读二：CED和CSA2如何重做长上下文架构/)
3. [DeepSeek-V4.1-Flash 精读（三）：把持久 KV 缓存压到八分之一](/2026/09/12/DeepSeek-V4.1-Flash精读三：把持久KV缓存压到八分之一/)
4. [DeepSeek-V4.1-Flash 精读（四）：没有新 RL 算法，Agent 为什么变强](/2026/09/12/DeepSeek-V4.1-Flash精读四：没有新RL算法，Agent为何变强/)
5. [DeepSeek-V4.1-Flash 精读（五）：成绩、推理旋钮与真实边界](/2026/09/12/DeepSeek-V4.1-Flash精读五：成绩、推理旋钮与真实边界/)

## 先把两类缓存分开

V4 的注意力要维护两类状态。

global KV 负责在整个上下文里做全局检索。上下文越长，它占的空间越大，也是 HBM、SSD 和传输带宽的主要压力。

另一类是每层自己的 Sliding-Window Attention，也就是 SWA KV。它只看最近一个窗口，运行时不会跟着上下文无限增长。不过为了前缀复用和多轮会话，V4 还是会在提示末尾、输出末尾等位置持久化 SWA KV。报告说，这部分接近 V4 持久 KV 容量的一半。

问题就出在保存时间上。

global KV 有长尾复用价值，72 小时后仍有可能命中。SWA KV 不一样。它通常只在活跃会话的几分钟内有用，一轮对话结束，或者用户不再回来，很快就成了死数据。

拿长期存储去放分钟级状态，怎么看都不划算。V4.1 的做法很直接：global KV 继续做小，SWA KV 则不再长期保存。

## FP4 省的是每份 KV 的大小

V4.1-Flash 把量化感知训练扩展到了 main KV cache。写入缓存时用约四比特格式，做注意力计算前再反量化。存储和传输按 FP4 的体积走，矩阵计算不必等所有硬件原生支持某种 FP4 指令。

格式也不是随便选的。Main KV 使用 E2M1，每 16 个通道共享一个 E4M3 scale，而且没有第二级 global scale。报告为此算过动态范围：RMSNorm 后，512 维 KV latent 的范数有界；训练中观测到的最大幅值也离格式上限很远。

主 KV 在 post-training 阶段做 QAT，并放在 RoPE 之后量化，避免解码时多出一条额外路径。缓存布局和融合 kernel 也要跟着调整，否则 scale 元数据、反量化和额外的 kernel launch，很容易吃掉低比特省下来的带宽。

所以单看 main KV，FP4 相比 V4 的 FP8 接近减半，不是八分之一。global KV 能做到约 1/4，还要算上 CSA2 的跨层复用：main KV 和 indexer K 不用每层各存一份，部分层还会复用 Top-K 选择。

换成算账的说法，就是每份状态更小了，状态的份数也少了。两个因素乘起来，global KV 才到了约 1/4。

![DeepSeek-V4.1-Flash 单 token 解码 FLOPs 随上下文长度变化](/img/deepseek-v41/fig02-decode-flops.png)

*图：不同代 DeepSeek 模型的单 token Decode FLOPs。来源：DeepSeek-V4.1-Flash Technical Report，Figure 2。*

图里的单 token 解码 FLOPs 随上下文增长几乎不变。缓存压缩省的也不只是硬盘。KV 变小后，HBM 容量、跨机搬运和长上下文 attention 的数据读取都会轻一些。

## 从 1/4 到 1/8，靠的是不存 SWA

V4 的 persistent KV 里，global KV 和 SWA KV 各占接近一半。V4.1 继续把 global KV 保留至少 72 小时，但把 SWA KV 放进一个分布式内存池。这个池子由每台机器约 10% 的主机 DRAM 组成，TTL 只有几分钟。

TTL 短，池子就能快速周转，大部分活跃会话仍然可以直接命中。麻烦发生在 SWA KV 已经被淘汰、global KV 却还在的时候。系统有全局状态，但缺少接着计算需要的局部状态。

可以重算，只是精确重算太贵。

假设模型有 L 层，窗口大小是 n_win。SWA 的依赖会逐层累积。要精确恢复所有层的 SWA KV，不能只跑最后 n_win 个 token，而要付出 L × n_win 个 token 的等价工作量。V4 报告设想过 Zero SWA Caching，生产环境扛不住这笔完整重算。

V4.1 的 Bounded Replay 只回放最近 n_win 个 token，并把 SWA 的可见范围限制在这段 replay 里。重算量从 L × n_win 降到 n_win，而且有明确上界。一次偶发 miss 不会突然放大成按层累积的重算，SWA KV 才能放心从 SSD 持久层移走。

现在再算一次总账。原来的持久缓存里，接近一半是 SWA，移走后只剩 global KV；global KV 又缩到了 V4 的约 1/4。两者相乘，persistent KV 就是约 1/8。

这不是某一种格式压出来的数字，而是部署策略和模型改造叠加后的结果。

少存的也不只是 SSD 容量。缓存越大，节点之间迁移前缀越慢，热点请求越难调度，故障恢复时要搬的数据也越多。对百万 token 的 Agent，容量、带宽和调度弹性都在同一张账单上。相比把某个 kernel 再提速几个百分点，少存八分之七可能更值钱。

当然，DRAM 池也不免费。它占了每台机器约 10% 的主机内存，只是靠分钟级回收提高周转率。这套方案依赖几个条件：线上会话的活跃期够短，SWA miss 足够少，replay 的开销又有硬上界。

报告给了生产经验上的结论，但没有公开完整的命中率分布和容量敏感性曲线。约 1/8 应该看成特定工作负载下的系统结果，不是换个集群也一定成立的常数。

## Encoder 和 Decoder 怎么补回状态

Encoder 侧如果命中 global KV、缺失 SWA KV，系统会回放缓存前缀末尾的 n_win 个 token，再和未缓存的 suffix 一起处理。回放段只重建 SWA KV。已经命中的 global KV 直接复用，既不重算，也不覆盖。suffix 才同时生成两类 KV。

Decoder 侧也只让提示末尾 n_win 个 token 经过 decoder layers，利用 encoder 输出重建解码起步所需的 SWA KV。

这套做法和 CED 的 encoder、prefill、decoder 分离配套。视觉编码、prefill 和 decode 可以独立扩缩容，缓存预取和加载尽量放到后台，与计算重叠。否则 replay 虽短，整条流水线还是可能被 I/O 堵住。

这里要留意一个边界：Bounded Replay 得到的是 approximate state，也就是近似状态，不是数学等价的状态。

Encoder 回放后，suffix 状态会受缓存命中位置影响。Decoder 重建出的 SWA KV，也不等于完整 decoder forward 的结果。报告明确承认了这一点，并在 post-training 中模拟同样的 replay，让模型提前适应部署时会遇到的截断状态。

同样的思路也用在低比特缓存上。QAT 让权重提前见过量化误差；RoPE、反量化和 attention 等步骤再由推理系统尽量融合。模型允许这种近似，系统负责把节省变成吞吐，缺一边都不行。

Encoder–Prefill–Decode 分离也不是架构图上的装饰。三段独立扩缩容后，长提示预处理、视觉编码和逐 token 解码可以按各自的瓶颈配机器。可预知的 embedding 和缓存读取放到后台，尽量与计算重叠。论文里的压缩率能不能落到线上单价，最后还得看这些工程细节。

## 这笔近似值不值

报告称，Encoder 和 Decoder 的 bounded replay 对响应质量影响都“可忽略”。这个结果足以解释为什么它敢上线，但证据没有覆盖所有极端情况。

我会关心几类边界：连续很多个短 turn，关键信息正好落在窗口边缘，不同的 cache-hit 位置，超长工具输出后马上追问，以及多次 miss 和 replay 累积后，答案会不会出现系统性漂移。报告没有给这些条件下独立、细粒度的消融。

能确认的是，DeepSeek 用训练适配和自己的线上工作负载证明了损失很小。不能确认的是 bounded replay 数学等价，或者所有长尾场景都没有风险。这条证据边界要写清楚。我也没有亲自测试这些极端条件。

从成本账看，我认为这笔交易合理。先用结构减少重复，再降低每份状态的字节数，最后把低复用、短寿命的数据赶出持久层。miss 发生时，也不追求完整恢复，而是把最坏重算量锁在一个窗口里。

V4.1-Flash 的约 1/8，是这几步一起算出来的。比起单独再做一次 KV 量化，这套取舍更值得看。
