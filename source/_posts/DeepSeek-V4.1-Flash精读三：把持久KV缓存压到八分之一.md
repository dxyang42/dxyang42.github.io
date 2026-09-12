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

很多模型报告写“缓存减少 8 倍”，读者很容易理解成：把原来的 KV 全部换成更低精度，文件自然变成八分之一。

DeepSeek-V4.1-Flash 不是这么做的。

它先在模型结构和数值精度上，把**global KV**压到 V4-Flash 的约 **1/4**；再在部署层重新划分“什么值得长期保存”，把短命的 SWA KV 从持久缓存里拿掉，最终让**persistent KV**降到约 **1/8**。

这两个数字说的是不同口径。一个主要对应运行时、常驻 HBM 的全局缓存；另一个对应为了前缀复用而长期放在 SSD 或主机内存里的持久缓存。把它们混成一句“FP4 带来 8 倍压缩”，既高估 FP4，也错过了这套系统最有价值的判断。

## 系列目录

1. [DeepSeek-V4.1-Flash 精读（一）：百万上下文真正贵在哪里](/2026/09/12/DeepSeek-V4.1-Flash精读一：百万上下文真正贵在哪里/)
2. [DeepSeek-V4.1-Flash 精读（二）：CED 和 CSA2 如何重做长上下文架构](/2026/09/12/DeepSeek-V4.1-Flash精读二：CED和CSA2如何重做长上下文架构/)
3. [DeepSeek-V4.1-Flash 精读（三）：把持久 KV 缓存压到八分之一](/2026/09/12/DeepSeek-V4.1-Flash精读三：把持久KV缓存压到八分之一/)
4. [DeepSeek-V4.1-Flash 精读（四）：没有新 RL 算法，Agent 为什么变强](/2026/09/12/DeepSeek-V4.1-Flash精读四：没有新RL算法，Agent为何变强/)
5. [DeepSeek-V4.1-Flash 精读（五）：成绩、推理旋钮与真实边界](/2026/09/12/DeepSeek-V4.1-Flash精读五：成绩、推理旋钮与真实边界/)

## 先分清两本账

V4 的注意力同时维护两类状态。

一类是 global KV，服务跨越整个上下文的全局检索。上下文越长，它越大，也是 HBM、SSD 和传输带宽的主要压力来源。

另一类是每层自己的 Sliding-Window Attention，也就是 SWA KV。它只覆盖最近一个窗口，单次运行时的体积不会无限随上下文增长。但为了前缀复用和多轮会话，V4 仍会在提示末尾、输出末尾等位置把它持久化。报告说，这部分接近 V4 持久 KV 容量的一半。

所以 V4.1 的两步账很好理解：先把 global KV 做小，再问一句——SWA KV 真的值得在 SSD 上保留 72 小时吗？

答案是否定的。global KV 有长尾复用，几天后仍可能命中；SWA KV 的价值通常只存在于活跃会话的几分钟里，下一轮结束或会话消失后就迅速变成死数据。用长期仓库存分钟级状态，不是压缩做得不够，而是分层策略错了。

## FP4 解决的是“每份状态多大”

V4.1-Flash 把量化感知训练扩展到 main KV cache。缓存写入时使用约四比特格式，注意力计算前再反量化。这样做的好处很现实：存储按 FP4 计费，矩阵计算却不必等待所有硬件原生支持某一种 FP4 指令。

报告选择 E2M1 数据表示，并为每 16 个通道配置一个 E4M3 scale；同时取消第二级 global scale。报告给的理由不是“经验上应该没事”，而是做了动态范围核算：RMSNorm 后 512 维 KV latent 的范数有界，训练中观测到的最大幅值也远低于该格式上限。主 KV 在 post-training 中做 QAT，RoPE 后再量化，以避免解码时增加额外路径。

如果从系统实现看，可以把这里概括成一套低比特 scale hierarchy：Main KV 使用 E2M1，每 16 个通道共享一个 E4M3 scale，并取消第二级 global scale；缓存布局和融合 kernel 则负责避免元数据与反量化开销吃掉 FP4 省下来的带宽。细节很多，但结论不用堆名词：**低比特只有进入训练、缓存格式和算子路径，才会变成真实成本下降。**

单看 main KV，FP4 相比 V4 的 FP8 接近减半，不是八分之一。global KV 能到约 1/4，还依赖 CSA2 在层维度复用 main KV 和 indexer K，并让部分层复用 Top-K 选择。也就是说，1/4 是“每项更小”乘上“少存几份”的结果。

![DeepSeek-V4.1-Flash 单 token 解码 FLOPs 随上下文长度变化](/img/deepseek-v41/fig02-decode-flops.png)

*图：不同代 DeepSeek 模型的单 token Decode FLOPs。来源：DeepSeek-V4.1-Flash Technical Report，Figure 2。*

图里解码 FLOPs 随上下文增长几乎不变，也说明缓存压缩不能只看硬盘占用。KV 更小，意味着 HBM 容量、跨机搬运和长上下文 attention 的数据读取一起受益。

## 真正决定 1/8 的，是不再长期保存 SWA

V4 的 persistent KV 里，global KV 和 SWA KV 各占接近一半。V4.1 保留前者至少 72 小时，却把后者移到每台机器拿出约 10% 主机 DRAM 组成的分布式内存池，TTL 只有几分钟。

短 TTL 让小池子也能高速周转，覆盖大多数活跃会话。可一旦 SWA KV 被淘汰，而 global KV 仍然命中，系统就缺了一块继续计算所需的局部状态。

最直觉的办法是重算。问题在于，SWA 的依赖会逐层累积。假设模型有 L 层、窗口大小为 n_win，要精确恢复所有层的 SWA KV，不能只把最后 n_win 个 token 跑一遍，而要回放 **L × n_win** 个 token 的等价工作量。V4 报告设想过 Zero SWA Caching，但这笔完整重算在生产环境里太贵。

V4.1 的 Bounded Replay 换了一条路：只回放最近 **n_win** 个 token，并把 SWA 的可见范围截断在这段 replay 内。

成本从随层数放大的 L × n_win，变成有明确上界的 n_win。一次偶发 miss 不再触发灾难性重算，于是系统才敢把 SWA KV 从 SSD 持久层彻底移走。

这一步和前面的 1/4 相乘：原持久缓存中接近一半的 SWA 被移除，留下的 global KV 又约为 V4 的 1/4，于是总 persistent KV 落到约 **1/8**。这是部署口径下的组合结果，不是某一种数据格式单独创造的魔法数字。

这里还有一个容易忽略的成本口径。persistent KV 省的不只是 SSD 采购。缓存越大，节点之间迁移前缀的时间越长，热点请求更难重新调度，故障恢复时也要搬更多数据。对百万 token 的 Agent 来说，容量、带宽和调度弹性是一笔联动账。少存八分之七，往往比把某个 kernel 再提速几个百分点更有部署价值。

反过来，DRAM 池也不是“免费缓存”。它拿走每台机器约 10% 的主机内存，只是用分钟级回收换来了高周转。这个设计成立的前提，是线上会话的活跃期确实短、SWA miss 确实少，而且 replay 成本有硬上界。报告给出了生产经验上的判断，却没有公开完整的命中率分布和容量敏感性曲线。因此，1/8 更适合被理解为给定工作负载下的系统结果，而不是任何部署都能照抄的常数。

## Encoder 和 Decoder 各自怎么回放

Encoder 侧命中 global KV、缺失 SWA KV 时，系统回放缓存前缀末尾的 n_win 个 token，并与未缓存的 suffix 一起处理。回放段只重建 SWA KV，已经命中的 global KV 直接复用，不重算也不覆盖；suffix 才同时生成两类 KV。

Decoder 侧也只让提示末尾 n_win 个 token 通过 decoder layers，利用 encoder 输出重建解码起步所需的 SWA KV。这和 CED 的 encoder/prefill/decoder 分离是配套设计：视觉编码、prefill、decode 可以独立扩缩容，后台预取和加载尽量与计算重叠，短 replay 才不会重新堵住整条流水线。

这里最重要的限定是：**Bounded Replay 得到的是近似状态，不是数学等价的状态。**

Encoder 回放后的 suffix 状态会依赖缓存命中位置；Decoder 重建出的 SWA KV 也不等于完整 decoder forward 的结果。报告明确承认这一点，并在 post-training 中模拟同样的 replay，让模型适应部署时会见到的截断状态。

这是一种训练—推理协同，而不是推理团队上线后偷偷改计算图。候选限制、低比特缓存、replay 截断都尽量进入训练分布；推理侧再用 kernel fusion、编码与解码分离、后台加载，把理论节省兑现成吞吐。

为什么这一点重要？因为只在推理时把 FP8 粗暴改成 FP4，误差会直接落到模型从未见过的状态上；只在服务端截断 replay，cache-hit 位置又会变成新的分布偏移。V4.1 把 QAT 放进 post-training，也模拟 Decoder replay，让权重提前见过将来部署时的误差。然后推理系统再负责把反量化、RoPE 和 attention 等步骤尽量融合，避免省了存储却多出一串 kernel launch 和中间张量。

同样，Encoder–Prefill–Decode 分离不是架构图上的装饰。三段独立扩缩容后，长提示的预处理、视觉编码与逐 token 解码可以按各自瓶颈配机器；可预知的 embedding 和缓存读取则放到后台，与计算重叠。模型负责允许复用和近似，系统负责安排数据何时到位，两边缺一边，论文里的压缩率都不一定能变成线上单价。

## 我的判断：这不是无损压缩，是值得下注的系统近似

报告称 Encoder 和 Decoder 的 bounded replay 对响应质量影响都“可忽略”。这个结论足以支持生产设计，但还不足以证明所有场景都安全。

我最想看的独立消融，是极端条件：连续很多个短 turn、关键信息刚好卡在窗口边界、不同 cache-hit 位置、超长工具输出之后立刻追问，以及多次 miss 和 replay 累积时，答案是否发生系统性漂移。报告没有给出这类独立、细粒度的极端条件结果。

因此更准确的说法是：DeepSeek 用训练适配和线上工作负载证据，证明这项近似在其测试里损失很小；它没有证明 bounded replay 在数学上等价，也没有封死所有长尾风险。

但从成本角度，我认可这笔交易。昂贵系统很少靠一个神奇格式降本，更多时候靠三件事一起发生：结构上减少重复，数值上降低每份状态的字节数，部署上拒绝长期保存低复用数据。V4.1-Flash 的 1/8，真正值得学的正是这套顺序。

**先分辨状态的寿命，再决定存不存；miss 之后不追求绝对恢复，而把最坏重算成本锁进一个窗口。**

这比“又把 KV 量化了一次”，更像一次成熟的系统设计。
