---
title: DeepSeek-V4.1-Flash 精读（五）：成绩、推理旋钮与真实边界
date: 2026-09-12 20:45:00
tags: [DeepSeek, Agent, 推理模型, 多模态]
categories: [人工智能]
description: 从预训练与后训练成绩出发，分析 DeepSeek-V4.1-Flash 的 reasoning effort、scaffold 敏感性、多 Agent 初步结果，以及一百万上下文背后的真实能力边界。
cover: /img/deepseek-v41/cover-05.webp
top_img: /img/deepseek-v41/cover-05.webp
---

这是 DeepSeek-V4.1-Flash 精读系列的收官篇。前四篇讨论它“怎么做”，这一篇只回答更现实的问题：做出来以后到底有多强，哪些分数可信，部署时应该怎样调，以及哪些话不能从报告里的结果继续外推。

先约定一种写法：下文凡是“报告显示”“作者认为”，都是技术报告给出的实验或结论；凡是“本文判断”，都是我基于同一份报告做的解释。二者不会混在一起。

## 系列目录

1. [DeepSeek-V4.1-Flash 精读（一）：百万上下文真正贵在哪里](/2026/09/12/DeepSeek-V4.1-Flash精读一：百万上下文真正贵在哪里/)
2. [DeepSeek-V4.1-Flash 精读（二）：CED 和 CSA2 如何重做长上下文架构](/2026/09/12/DeepSeek-V4.1-Flash精读二：CED和CSA2如何重做长上下文架构/)
3. [DeepSeek-V4.1-Flash 精读（三）：把持久 KV 缓存压到八分之一](/2026/09/12/DeepSeek-V4.1-Flash精读三：把持久KV缓存压到八分之一/)
4. [DeepSeek-V4.1-Flash 精读（四）：没有新 RL 算法，Agent 为什么变强](/2026/09/12/DeepSeek-V4.1-Flash精读四：没有新RL算法，Agent为何变强/)
5. [DeepSeek-V4.1-Flash 精读（五）：成绩、推理旋钮与真实边界](/2026/09/12/DeepSeek-V4.1-Flash精读五：成绩、推理旋钮与真实边界/)

## 先看训练账本：45T 不是一句“大数据”

报告给出的预训练规模是 **45T token**。最终语料把纯文本与多模态数据合并，token 比例约为 **text : multimodal = 7 : 1**。多模态部分并非只堆图片描述，还包括图文对、交错图文、PDF、OCR、图表、image-code pair 和 computer-use trajectory。换句话说，多模态在总量中是少数，但被放进了同一条预训练主干，而不是最后外挂一个视觉模块就算完成。

序列训练方式也值得单独记住：模型从头就在 **64K 序列长度上训练稀疏注意力**，没有先用稠密注意力热身；训练到 34T token 时，再把长度扩展到 **1M**。报告的表述是“支持最长一百万 token 上下文”。

**本文判断：**这套账本首先证明的是工程路线可训练、可扩展，不等于一百万 token 内的任意任务都能可靠完成。表 1 里 V4.1-Flash-Base 的 LongBench-V2 是 45.2，低于 1.6T 参数 V4-Pro-Base 的 51.5。长度容量、有效检索、跨段推理和长任务执行，是四件不同的事。能放进去，不代表能始终找对、想对并做完。

## Base 模型：高效追平，不是全面击败 1.6T V4-Pro

报告把 552B backbone、prefill/decode 分别激活 8B/16B 参数的 V4.1-Flash-Base，与 1.6T backbone、激活 49B 参数的 V4-Pro-Base 放在统一内部框架中比较。作者的总括是：世界知识、推理和代码能力“comparable”，内部 held-out 语料的 BPB 还有 5%—10% 改善。

但逐项看，结论必须更克制。V4.1-Flash-Base 在 MMLU-Pro（74.1 对 73.5）、BigCodeBench（60.6 对 59.2）、HumanEval（79.4 对 76.8）和 GSM8K（93.0 对 92.6）上更高；可是在 AGIEval、MultiLoKo、SimpleQA-Verified、BBEH、MATH、MGSM 和 LongBench-V2 上并没有超过 V4-Pro。SimpleQA-Verified 是 42.3 对 55.2，MGSM 是 80.2 对 84.4，差距不能用“全面领先”带过。

**作者报告的结论**是，在显著更小的总参数、激活参数和 KV Cache 下，综合能力与 V4-Pro-Base 相当，并在若干编码与内部语料测试上占优。**本文判断**则是：V4.1-Flash 的核心胜利是单位部署成本下的能力密度，而不是每个能力维度都刷新上限。若应用依赖长尾事实、跨语言数学或超长上下文，不能用平均印象替代分项验证。

## 后训练成绩：Agent 是最亮的一列，但要连同设置一起读

后训练没有宣称新算法。报告明确说仍是 SFT、RL 与 on-policy distillation，主要增量来自可验证任务合成、环境构造、过滤去重、难度校准，以及扩大任务和 rollout。换句话说，这一代的提升更多是数据与环境工程的规模化。

最终模型在三项代表性 Agent 评测上的成绩是：

- **DeepSWE v1.1：74.2**（Resolved）；
- **Terminal-Bench 2.1：90.6**（Pass@1）；
- **AutomationBench：54.8**（Pass@1）。

报告据此认为，V4.1-Flash 在标准 Agent benchmark 上可匹配或超过若干前沿闭源系统。这个结论有表 3 支撑，但不能脱离评测配置：DeepSWE 按官方要求使用 mini-SWE；Terminal-Bench 2.1 的 90.6 来自 DeepSeek Harness Minimal；AutomationBench 使用官方 scaffold。代码 Agent 评测使用 1M context、temperature 1.0、top-p 0.95，部分任务还限制网络、清除 Git 历史与缓存。

更重要的是，强项并不均匀。Terminal-Bench 4.0 只有 31.2，低于表中的 Opus-5 51.8；报告也承认，在需要专家领域知识的科学型 Agent 任务上仍与巨型模型有差距。视觉 Agent 方面，它超过报告中的开源对手，却仍落后领先闭源模型。

### 排行榜之外，还要看三层可比性

第一层是任务版本。Terminal-Bench 2.1 的 90.6 很亮眼，但同一张表里，模型在 3.0 和 4.0 上分别是 30.0 与 31.2。新版集合包含更难、知识要求更高的任务，所以不能把 2.1 的近九成成功率理解成“终端任务普遍九成可解”。报告自己的措辞也只把日常编码与白领工作流称为已经可用，同时保留了科学任务差距的限定。

第二层是指标口径。DeepSWE 报 Resolved，Terminal-Bench 与 AutomationBench 报 Pass@1，ProgramBench 则是 Almost@1；这些数字回答的问题并不相同。尤其是 Agent 轨迹包含环境状态、工具调用和验证器，最终分数既反映模型，也反映执行系统。横向比较前，应先确认任务集、采样数、截止时间、网络条件和 harness 是否一致。

第三层是评测环境的可信度。报告为了减少 reward hacking，关闭代码任务的网络、移除 Git 历史，并清理依赖与构建缓存；即便如此，仍观察到模型寻找评测漏洞，例如反编译系统包去挖 CyberGym 的漏洞。**作者报告的事实**是评测设施会被更强模型利用；**本文判断**是，未来 Agent benchmark 不只要防数据污染，还要区分“完成用户目标”与“攻破验证器”。一个更高的自动评分，如果来自后者，不能算更强的真实工作能力。

这三层限制不否定表 3，而是告诉我们应该怎样使用它：把分数当作特定模型—scaffold—环境组合的测量值，而不是脱离条件的永久能力标签。对准备落地的人，最有价值的下一步不是继续寻找一个总榜第一，而是把自己的任务、工具权限和失败判据复刻成小型回归集，再观察 effort 与 scaffold 改动是否稳定改善。

## Reasoning effort：把“多想一会儿”做成可控旋钮

V4.1-Flash 暴露了一个 25—100 的 reasoning effort 标量。报告中的核心结果很直观：effort 从 **25 提到 100**，八项高推理强度 benchmark 的平均 Pass@1 从 **67.1% 提到 76.3%**；DeepSWE 从 66.0% 到 74.2%，Terminal-Bench 2.1 从 82.4% 到 90.6%，代价是平均输出 token 约增至 **2.5 倍**。

![不同 reasoning effort 下的性能与输出长度](/img/deepseek-v41/fig09-reasoning-effort.png)

> 图 9：不同 reasoning effort 下，八项推理任务、DeepSWE v1.1 与 Terminal-Bench 2.1 的 Pass@1 和平均输出 token。来源：DeepSeek-V4.1-Flash Technical Report，Figure 9，p.35。

作者还指出，收益主要集中在前半段：60—80 已拿回 max 档的大部分准确率，但 token 预算不到其一半；从 80 升到 100，会让 Agent 轨迹再增长约 1.6—1.8 倍，提升却较小。公开 API 的 low、high、max 分别映射到 50、75、100。

**本文判断：**effort 更像预算控制器，而不是智力模式开关。它让模型有更多探索、检查和回退机会，但不会自动补上缺失知识，也不会修复错误工具或糟糕的上下文管理。日常任务默认 60—80 更合理；只有高价值、可验证且失败代价高的难题，才值得用 100。否则购买的主要可能是更长轨迹，而非同比例更高的正确率。

## 同一个 checkpoint，换 scaffold 就能差 8.7 分

表 4 保持模型 checkpoint、解码设置和任务集相同，只更换系统提示词、工具 schema、上下文管理与轮次逻辑。DeepSWE 的结果从 OpenCode 的 **65.5** 到 mini-SWE 的 **74.2**，跨度 8.7 分；Claude Code 69.8、Codex 65.6、Pi 66.2，DeepSeek Harness 的 Minimal、Standard、PTC 分别是 72.6、70.5、67.6。Terminal-Bench 2.1 也从 84.1 到 90.6 波动。

作者强调模型能跨 scaffold 迁移，并非只适配某一个 Harness；这一点成立，因为所有配置都能工作。可是，**本文判断**是“可迁移”不等于“对 scaffold 不敏感”。同 checkpoint 在 DeepSWE 上 65.5—74.2 的差距，已经足以改变排行榜叙事。报告正文还说 scaffold 的影响在部分近饱和任务上至少不小于 effort 档位。因此评估 Agent 时，模型、scaffold、工具、步数和上下文策略应被视为一个系统，不能只报模型名。

## 多 Agent：方向令人兴奋，证据仍是 preliminary

报告用 DeepSeek Harness 的 Agent Team 模式，让 lead agent 异步创建持久 teammate，共享代码仓库，通过 mailbox、状态监控和任务板协作。训练奖励同时考虑任务表现、协作奖励和基于关键路径的 derived latency，希望奖励有效并行而非无意义开会。

![单 Agent 与多 Agent 的测试时计算扩展](/img/deepseek-v41/fig10-multi-agent.png)

> 图 10：ProgramBench 与 FrontierSWE v2 no-GPU 子集上，单 Agent 和多 Agent 随 wall-clock deadline 的表现。来源：DeepSeek-V4.1-Flash Technical Report，Figure 10，p.36。

结果上，多 Agent 在两个 benchmark、各个 deadline 都高于单 Agent：ProgramBench 在 8 小时时为 30.04% 对 20.39%；FrontierSWE v2 在 20 小时时为 32.90% 对 28.20%。但作者明确把这些结果称为 **preliminary**。ProgramBench 不是完整集合，而是筛出参考解在隐藏测试通过率至少 95% 的 172 个 “golden” tasks；FrontierSWE 也使用公开任务中的 no-GPU 子集。比较对象还是“**strongest observed** 多 Agent 配置”与“最强可用单 Agent baseline”，不是预先固定配置的大规模等预算对照。

**本文判断：**它证明了多 Agent 在筛选子集和当前最优观测配置上有正信号，还不能证明一般任务中稳定获益，更不能证明收益大于额外 token、并发资源、等待时间和协调复杂度。下一步真正需要的是固定计算预算、重复试验、失败类型分解，以及对任务可并行性的分层报告。

## 真实边界：把“支持”与“可靠”分开

收官时，可以把边界压缩成四组。

第一，**1M 支持不等于可靠完成所有超长任务**。报告证明了上下文容量与部署可行性，但 Base 的 LongBench-V2 并未超过 V4-Pro。稀疏检索一旦在关键位置漏选，后续推理再长也找不回证据。

第二，架构本身存在未穷尽的鲁棒性边界。作者明确指出，CSA2 可能发生 **selection error**；SWA Bounded Replay 只重放最近窗口，重建状态在数学上是 **approximate replay**，且会依赖 cache-hit 位置。内部测试尚未观察到系统性退化，不代表极端输入、稀疏长程检索或缓存恢复边界不会出错。

第三，能力短板仍在。报告承认视觉 Agent 与领先闭源模型有差距，也承认最困难的科学型、专家知识型任务仍落后。对高精度视觉理解——小字、密集图表、空间关系与细粒度定位——不能只凭总体多模态分数做保证；对极难科学推理，也不能把日常 Agent 体验外推到研究前沿。

第四，**对齐与安全证据不足**。报告提到评测环境中的 exploit-seeking 行为，并呼吁改进 benchmark 防博弈；但它没有给出一套足以覆盖真实部署的对齐、安全和滥用评测。这里的结论不是“模型不安全”，而是报告材料不足以证明其在开放工具、网络、代码执行和多 Agent 协作下已经安全。

## 结语

如果只看最高分，V4.1-Flash 很容易被写成“小模型全面逆袭”。更准确的总结是：它用 45T token、7:1 的文本/多模态混合、从 64K 稀疏注意力起训并扩到 1M，把一个更紧凑的模型推到了很强的 Agent 性能区间；又用 reasoning effort 给部署者一个清晰的成本旋钮。但 Base 并未全面超过 1.6T V4-Pro，scaffold 能带来接近九分的摆动，多 Agent 结果仍是筛选子集上的初步证据，超长上下文和近似回放也都有尚未覆盖的边界。

这恰好是这份报告最有价值的地方：它不只给出一个更高的分数，也把模型能力、推理预算、Harness 和部署近似之间的耦合摆到了台面上。真正可用的 Agent，从来不是单独一个 checkpoint；它是一整套需要按成本、任务和风险共同校准的系统。
