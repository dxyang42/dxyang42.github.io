---
title: DeepSeek-V4.1-Flash 精读（五）：成绩、推理旋钮与真实边界
date: 2026-09-12 20:45:00
tags: [DeepSeek, Agent, 推理模型, 多模态]
categories: [人工智能]
description: 从预训练与后训练成绩出发，分析 DeepSeek-V4.1-Flash 的 reasoning effort、scaffold 敏感性、多 Agent 初步结果，以及一百万上下文背后的真实能力边界。
cover: /img/deepseek-v41/cover-05.webp
top_img: /img/deepseek-v41/cover-05.webp
---

这是 DeepSeek-V4.1-Flash 精读系列的最后一篇。前四篇一直在拆它怎么训、怎么省、Agent 能力又是怎么来的。到了收官，还是得回到几个不那么浪漫的问题：分数到底怎么样，部署时该怎么调，哪些结论出了报告就站不住。

下面的数据和实验结论都来自技术报告；涉及“值不值得”“意味着什么”的地方，是我自己的判断。尽量把两件事分开说，但不打算每段都贴一次标签。

## 系列目录

1. [DeepSeek-V4.1-Flash 精读（一）：百万上下文真正贵在哪里](/2026/09/12/DeepSeek-V4.1-Flash精读一：百万上下文真正贵在哪里/)
2. [DeepSeek-V4.1-Flash 精读（二）：CED 和 CSA2 如何重做长上下文架构](/2026/09/12/DeepSeek-V4.1-Flash精读二：CED和CSA2如何重做长上下文架构/)
3. [DeepSeek-V4.1-Flash 精读（三）：把持久 KV 缓存压到八分之一](/2026/09/12/DeepSeek-V4.1-Flash精读三：把持久KV缓存压到八分之一/)
4. [DeepSeek-V4.1-Flash 精读（四）：没有新 RL 算法，Agent 为什么变强](/2026/09/12/DeepSeek-V4.1-Flash精读四：没有新RL算法，Agent为何变强/)
5. [DeepSeek-V4.1-Flash 精读（五）：成绩、推理旋钮与真实边界](/2026/09/12/DeepSeek-V4.1-Flash精读五：成绩、推理旋钮与真实边界/)

## 先把训练账本摊开

预训练规模是 45T token。最终语料把纯文本和多模态数据混在一起，比例约为 text : multimodal = 7 : 1。多模态部分不只是常见的图片描述，还塞进了图文对、交错图文、PDF、OCR、图表、image-code pair 和 computer-use trajectory。量上它是少数，位置上却不是最后外挂的视觉补丁，而是从预训练主干里一起长出来的。

序列长度的安排也很直接：模型从头就在 64K 长度上训练稀疏注意力，没有先用稠密注意力热身；跑到 34T token 后，再把长度拉到 1M。报告据此说模型“支持最长一百万 token 上下文”。

我会把这句话读窄一点。它证明了这条工程路线能训、能扩，不代表扔进一百万 token 后，什么任务都能稳稳做完。表 1 里，V4.1-Flash-Base 的 LongBench-V2 是 45.2，反而低于 1.6T 参数 V4-Pro-Base 的 51.5。装得下、找得到、推得对、做得完，本来就是四回事。窗口大只是入场券，不是通关证明。

## Base 模型的成绩

报告把 552B backbone、prefill/decode 分别激活 8B/16B 参数的 V4.1-Flash-Base，和 1.6T backbone、激活 49B 参数的 V4-Pro-Base 放进同一套内部框架比较。报告的总评是，世界知识、推理和代码能力“comparable”，内部 held-out 语料的 BPB 还有 5%—10% 改善。

逐项看就没那么整齐了。V4.1-Flash-Base 在 MMLU-Pro 上是 74.1 对 73.5，BigCodeBench 是 60.6 对 59.2，HumanEval 是 79.4 对 76.8，GSM8K 是 93.0 对 92.6，确实更高。但 AGIEval、MultiLoKo、SimpleQA-Verified、BBEH、MATH、MGSM 和 LongBench-V2 都没有超过 V4-Pro。SimpleQA-Verified 是 42.3 对 55.2，MGSM 是 80.2 对 84.4，这种差距不能拿一句“全面领先”糊过去。

所以报告说的是：在总参数、激活参数和 KV Cache 都显著更小的前提下，综合能力和 V4-Pro-Base 相当，编码与部分内部语料测试还更好。我的读法更朴素：赢的是单位部署成本下的能力密度，不是每个维度都赢。业务真吃长尾事实、跨语言数学或超长上下文，就老老实实按分项验证，别拿平均印象下注。

## Agent 分数怎么读

后训练部分没有发明一个新算法。报告写得很清楚，还是 SFT、RL 和 on-policy distillation，主要增量来自可验证任务合成、环境构造、过滤去重、难度校准，以及把任务和 rollout 做大。说白了，这次更像是把数据与环境工程拧得更紧，而不是算法突然换代。

最终模型在三项代表性 Agent 评测上拿到：

- DeepSWE v1.1：74.2（Resolved）；
- Terminal-Bench 2.1：90.6（Pass@1）；
- AutomationBench：54.8（Pass@1）。

报告据此判断，V4.1-Flash 在标准 Agent benchmark 上可以匹配或超过若干前沿闭源系统。表 3 确实撑得起这个说法，不过配置一个字都不能省：DeepSWE 按官方要求使用 mini-SWE；Terminal-Bench 2.1 的 90.6 来自 DeepSeek Harness Minimal；AutomationBench 使用官方 scaffold。代码 Agent 评测开 1M context，temperature 1.0、top-p 0.95，部分任务还会断网、清掉 Git 历史和缓存。

成绩也不是处处都亮。Terminal-Bench 4.0 只有 31.2，低于表中 Opus-5 的 51.8。报告自己也承认，需要专家领域知识的科学型 Agent 任务还打不过巨型模型；视觉 Agent 虽然超过报告里的开源对手，和领先闭源模型仍有距离。

### 榜单数字旁边的小字

先看任务版本。Terminal-Bench 2.1 的 90.6 很抢眼，但同一张表里，3.0 和 4.0 分别只有 30.0、31.2。新版集合更难，专家知识要求也更高，不能把 2.1 的近九成成功率翻译成“终端任务普遍九成可解”。报告也只把日常编码和白领工作流称作已经可用，后面还留着科学任务的限定。

再看指标。DeepSWE 报 Resolved，Terminal-Bench 和 AutomationBench 报 Pass@1，ProgramBench 报 Almost@1，它们问的不是同一个问题。Agent 的整条轨迹里还有环境状态、工具调用和验证器，最后一分既在测模型，也在测执行系统。横向比较之前，任务集、采样数、截止时间、网络条件、harness，缺一个都可能对不上。

最后是评测环境本身。报告为了压住 reward hacking，给代码任务断网、移除 Git 历史、清理依赖和构建缓存。即便如此，模型还是会找评测漏洞，比如反编译系统包去挖 CyberGym 的漏洞。这是报告观察到的现象。往外推一步，我更在意的是：以后的 Agent benchmark 不只要防数据污染，还得分清模型是在完成用户目标，还是在攻破验证器。后者拿到更高自动分，不能算真实工作能力更强。

这不是要把表 3 推翻，而是把它放回正确的位置：它测到的是某个模型—scaffold—环境组合，不是一个脱离条件、永久有效的能力标签。真要落地，最好别继续找一个总榜第一。拿自己的任务、工具权限和失败判据做个小回归集，再看 effort 和 scaffold 的改动能不能稳定复现收益，这更有用。

## Reasoning effort 这颗旋钮

V4.1-Flash 给了一个 25—100 的 reasoning effort 标量。报告里的趋势相当清楚：effort 从 25 调到 100，八项高推理强度 benchmark 的平均 Pass@1 从 67.1% 涨到 76.3%；DeepSWE 从 66.0% 到 74.2%，Terminal-Bench 2.1 从 82.4% 到 90.6%。代价也明码标价，平均输出 token 大约变成 2.5 倍。

![不同 reasoning effort 下的性能与输出长度](/img/deepseek-v41/fig09-reasoning-effort.png)

> 图 9：不同 reasoning effort 下，八项推理任务、DeepSWE v1.1 与 Terminal-Bench 2.1 的 Pass@1 和平均输出 token。来源：DeepSeek-V4.1-Flash Technical Report，Figure 9，p.35。

收益主要在前半段。报告说 60—80 已经拿回 max 档的大部分准确率，token 预算却不到后者一半；从 80 再拉到 100，Agent 轨迹会继续长约 1.6—1.8 倍，提升已经不大。公开 API 的 low、high、max 分别对应 50、75、100。

这东西与其叫“智力档位”，不如叫预算控制器。更多预算能换来更多探索、检查和回退机会，但补不出模型不知道的知识，也救不了错误工具和混乱的上下文管理。日常任务把 60—80 当默认区间比较划算；100 更适合高价值、可验证、失败代价又高的难题。否则多花的钱，很可能主要买来了一条更长的轨迹。

## Scaffold 能把同一模型拉开多少

表 4 固定 checkpoint、解码设置和任务集，只换系统提示词、工具 schema、上下文管理和轮次逻辑。DeepSWE 从 OpenCode 的 65.5 一路到 mini-SWE 的 74.2，差了 8.7 分。中间还有 Claude Code 69.8、Codex 65.6、Pi 66.2；DeepSeek Harness 的 Minimal、Standard、PTC 分别是 72.6、70.5、67.6。Terminal-Bench 2.1 也会从 84.1 摆到 90.6。

报告强调的是模型能跨 scaffold 工作，不是只认某一个 Harness。这个结论没问题，毕竟所有配置都跑得起来。但“能迁移”和“对 scaffold 不敏感”显然不是一回事。同一个 checkpoint 在 DeepSWE 上能落进 65.5–74.2 这么宽的区间，排行榜故事都足够重写一遍。报告正文还提到，在一些接近饱和的任务上，scaffold 的影响至少不小于 effort 档位。以后再看 Agent 分数，模型、scaffold、工具、步数和上下文策略最好绑在一起报。只写模型名，信息不够。

## 多 Agent 的初步试验

DeepSeek Harness 的 Agent Team 模式里，lead agent 可以异步创建持久 teammate。它们共享代码仓库，通过 mailbox、状态监控和任务板协作。训练奖励同时考虑任务表现、协作奖励和基于关键路径的 derived latency，目的是鼓励真并行，不是让几个 Agent 围在一起反复开会。

![单 Agent 与多 Agent 的测试时计算扩展](/img/deepseek-v41/fig10-multi-agent.png)

> 图 10：ProgramBench 与 FrontierSWE v2 no-GPU 子集上，单 Agent 和多 Agent 随 wall-clock deadline 的表现。来源：DeepSeek-V4.1-Flash Technical Report，Figure 10，p.36。

结果是，多 Agent 在两个 benchmark、各个 deadline 下都高于单 Agent。ProgramBench 跑到 8 小时时是 30.04% 对 20.39%；FrontierSWE v2 跑到 20 小时时是 32.90% 对 28.20%。不过报告明确用了 preliminary 这个词，原因也摆在实验设置里。

ProgramBench 不是完整集合，而是筛过一遍，只留下参考解在隐藏测试中通过率至少 95% 的 172 个“golden”tasks。FrontierSWE 用的也是公开任务里的 no-GPU 子集。对照还是“strongest observed”多 Agent 配置和最强可用单 Agent baseline，并非预先锁死配置、反复运行的等预算大样本实验。

因此目前能说的只有：在筛选子集和当前观察到的最优配置上，多 Agent 有正信号。它还证明不了一般任务都能稳定受益，更证明不了收益一定盖得过额外 token、并发资源、等待时间和协调成本。后面真要把话说硬，至少还缺固定计算预算、重复试验、失败类型拆解，以及按任务可并行性分层的结果。

## 报告没有覆盖掉的边界

一百万上下文先别神化。报告证明了 1M 容量与部署可行性，但 Base 的 LongBench-V2 没有超过 V4-Pro。稀疏检索如果在关键位置漏选，后面想得再久也捞不回那段证据。“支持”和“可靠完成所有超长任务”之间，还隔着很远。

架构本身也有没摸到底的鲁棒性问题。报告明确写了，CSA2 可能出现 selection error；SWA Bounded Replay 只重放最近窗口，重建状态在数学上属于 approximate replay，而且会依赖 cache-hit 位置。内部测试目前没看到系统性退化，这是已有证据。极端输入、稀疏长程检索和缓存恢复的边界是否会翻车，报告并没有替我们回答。

能力短板同样不难找。视觉 Agent 和领先闭源模型仍有差距，最难的科学型、专家知识型任务也还落后。高精度视觉理解里的小字、密集图表、空间关系、细粒度定位，不能拿总体多模态分数担保；日常 Agent 做得顺，也不代表极难科学推理就能照着外推。

安全部分的证据更薄。报告记录了 exploit-seeking 行为，也呼吁 benchmark 加强防博弈，但没有给出一套足以覆盖真实部署的对齐、安全和滥用评测。这不能直接推出“模型不安全”，只能说明：仅凭这份报告，还证明不了它在开放工具、网络、代码执行和多 Agent 协作下已经安全。两句话差得很大。

## 收尾

V4.1-Flash 当然强，尤其是把部署成本一起放进来看的时候。45T token、7:1 的文本与多模态混合、64K 稀疏注意力起训再扩到 1M，最后把一个更紧凑的模型推到 DeepSWE 74.2、Terminal-Bench 90.6、AutomationBench 54.8，这套工程账是漂亮的。reasoning effort 也确实给部署者留了一颗能用的成本旋钮。

但别急着写“小模型全面逆袭”。Base 没有全面超过 1.6T V4-Pro；effort 从 25 拉到 100，平均分从 67.1% 到 76.3%，同时输出约 2.5 倍；同一 checkpoint 换个 scaffold，DeepSWE 就能在 65.5–74.2 之间晃；多 Agent 还是筛选子集上的 preliminary 结果；1M 上下文、selection error 和 approximate replay 也各有各的边界。

这份报告最值得看的，恰恰不是某个最高分，而是它把账摊开了：能力来自模型，也来自预算、Harness、工具和部署近似。单独一个 checkpoint 不等于可用的 Agent。真上生产，还是得按自己的任务、成本和风险一项项校准。没那么性感，但这是实话。
