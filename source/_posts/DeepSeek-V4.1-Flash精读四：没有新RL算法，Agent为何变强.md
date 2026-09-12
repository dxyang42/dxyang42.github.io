---
title: DeepSeek-V4.1-Flash 精读（四）：没有新 RL 算法，Agent 为什么变强
date: 2026-09-12 20:46:00
tags: [DeepSeek, Agent, 强化学习, DSec]
categories: [人工智能]
description: DeepSeek-V4.1-Flash 没有发明新的后训练算法，却靠任务合成、可验证环境、跨 scaffold rollout、checkpoint merging、DSec 与异步后训练把 Agent 能力推了上去。
cover: /img/deepseek-v41/cover-04.webp
top_img: /img/deepseek-v41/cover-04.webp
---

这是 DeepSeek-V4.1-Flash 技术报告精读的第四篇。

## 系列目录

1. [DeepSeek-V4.1-Flash 精读（一）：百万上下文真正贵在哪里](/2026/09/12/DeepSeek-V4.1-Flash精读一：百万上下文真正贵在哪里/)
2. [DeepSeek-V4.1-Flash 精读（二）：CED 和 CSA2 如何重做长上下文架构](/2026/09/12/DeepSeek-V4.1-Flash精读二：CED和CSA2如何重做长上下文架构/)
3. [DeepSeek-V4.1-Flash 精读（三）：把持久 KV 缓存压到八分之一](/2026/09/12/DeepSeek-V4.1-Flash精读三：把持久KV缓存压到八分之一/)
4. [DeepSeek-V4.1-Flash 精读（四）：没有新 RL 算法，Agent 为什么变强](/2026/09/12/DeepSeek-V4.1-Flash精读四：没有新RL算法，Agent为何变强/)
5. [DeepSeek-V4.1-Flash 精读（五）：成绩、推理旋钮与真实边界](/2026/09/12/DeepSeek-V4.1-Flash精读五：成绩、推理旋钮与真实边界/)

报告讲到后训练时，开头先交代了一件不太像发布会宣传的话：这次没有引入新的后训练算法。

配方还是 SFT、RL 和 on-policy distillation，优化方法也没有跳出已有做法。报告给出的解释相当直接：在固定且常规的优化流程下，观测到的提升基本来自合成数据和环境，尤其是规模、多样性与可验证性的改善。

这并不是说 RL 不重要。只是这一次，稀缺的不是另一条更复杂的 loss，而是大量够难、能执行、还能自动判分的任务，以及支撑百万级交互跑起来的系统。样本本身太单薄，再精巧的目标函数也没有多少东西可学。

Agent 数据也不等于多收集一些问答。一次训练样本往往是一段执行过程：模型采取动作，环境发生变化，工具返回观察，验证器再判断结果。任务种类和 scaffold 一变，模型见到的状态—动作分布也跟着变。数据规模到了这里，已经离不开可执行系统的规模。

所以报告会花不少篇幅讲容器、调度和中断恢复。短问答里的基础设施通常影响训练快慢；长时程 Agent 还要靠它保证轨迹能完整采集、失败可以复现、奖励值得相信。它不只是后勤。

## 一条样本，背后是三个部分

报告把任务生产拆成一个三元组：task specification、environment、reference solution。

Task specification 规定目标、初始状态和限制条件。Environment 让模型实际调用工具、修改文件、执行命令并接收反馈。Reference solution 用来帮助生成检查点和奖励信号。到了质量审计阶段，报告又把它写成 problem、environment、verification system。参考解答只是手段，最后仍要落到一个能自动判断任务是否完成的验证系统上。

普通指令数据大多只要处理“问题—答案”。Agent 任务麻烦得多：依赖是否能安装，初始仓库是否处于正确状态，测试有没有覆盖题面，旧功能会不会被破坏，答案是否泄漏，模型又能不能绕过测试拿到奖励。

任何一部分出问题，RL 信号都会受影响。题目含糊，动作就没有明确目标；环境不可复现，同样的操作可能得到不同结果；reference solution 或 verifier 留了洞，模型很可能去钻洞。DeepSeek 会同时按难度和正确性筛选任务，也会拿新的 rollout 回头复审环境。训练中新出现的轨迹，正好能暴露先前没有发现的问题。

## 两条任务管线

第一条从真实查询和失败案例出发。

通用 Agent 侧会收集用户自愿返回的日常工作交互、负反馈和失败，再根据其中出现的接口去 mock SaaS、企业应用与专用后台。要复刻的不只是一句 prompt，还包括工具输入输出、API schema、行为约束、交互方式和失败条件。这样得到的任务更接近实际分布，线上弱点也能被放进沙箱反复训练。

编码 Agent 的来源类似。团队会从内部员工和外部伙伴的会话中筛选复杂任务，或者专门挑模型做得不好的任务，再按照轨迹去重。它们提供的不是一份想象出来的能力清单，而是模型具体会在哪些地方摔跤。

第二条从零生成任务。

系统先从达到 star 门槛的公开 GitHub 仓库里选项目。多个专门 Agent 一起工作：判断项目能否在容器内完整运行、结果能否自动验证；选择一个 turn 或 commit 作为起点；设计实现方向以及 fail-to-pass、pass-to-pass 检查点；再配置依赖、工作目录、测试和题面。完成自测并清掉答案痕迹后，环境才会被打包成新的镜像层。

到这里还不能直接入库。多个求解 Agent 会实际做题，独立的质检 Agent 同时检查环境和轨迹，寻找事实错误、题面与评分点错位、环境故障以及被 hack 的可能。发现问题后，修复 Agent 会修改环境或调整难度，然后再跑一轮验证。

我比较认同其中一个取舍：把真实工作流转成训练沙箱时，优先保留任务原意。接口可以 mock，外部依赖可以装进容器，敏感信息也应该清除，但用户原本要完成的目标、约束和失败条件不能一起被洗掉。不然 verifier 也许很稳定，模型练的却是另一件事。

## Scaffold 也会改变训练分布

Agent 最后怎么行动，由模型和 scaffold 共同决定。提示词、工具集合、上下文管理、压缩策略或动作协议稍有变化，rollout 分布都可能变化。如果始终只在一个 harness 中训练，模型对特定脚手架的适应，很容易被当成通用能力。

报告沿着两个方向扩大 RL：累计更多 RL steps，同时增加 scaffold 数量。训练既覆盖多个 Claude Code 版本，也横跨 OpenCode、Pi，以及 DeepSeek Harness 的 Standard 和 PTC 模式。我之前写[《DeepSeek Harness 和 Pi Agent》](/2026/08/13/DeepSeek-Harness和Pi-Agent/)时也讨论过，这些运行时不只是给模型套一层界面。

工程上，rollout 被分成 agent sandbox 和 worker container。Agent sandbox 负责运行 scaffold 与工具，worker container 则提供独立于 scaffold 的控制层，把不同交互整理成统一轨迹，再与 trainer 通信。接入新 scaffold 时，底层 RL 算法不需要跟着修改。

![随累计 RL steps 增加，多个代码 Agent 基准的表现变化](/img/deepseek-v41/fig07-rl-scaling.png)

*图 1：单一 DeepSeek Harness Minimal 模式下的 RL scaling。来源：DeepSeek-V4.1 技术报告 Figure 7。*

Figure 7 里，Pass@1 随累计训练步数整体上升，1M 上下文中的超长任务还在继续受益。Figure 8 展示的多版本 Claude Code 与异构 scaffold 联合训练，也有相近的走势。

![多版本 Claude Code 与异构 scaffold 的联合 RL](/img/deepseek-v41/fig08-multi-scaffold-rl.png)

*图 2：多版本 Claude Code 及 OpenCode、Pi、DeepSeek Harness Standard/PTC 的联合训练。来源：DeepSeek-V4.1 技术报告 Figure 8。*

不过，这些曲线不是严格的因果消融。它们能说明指标随着整套训练流程推进而上升，但报告没有在算力、数据、初始化和评测条件完全相同的情况下，只切换“多 scaffold”这一项。结果支持这条工程路线，不能据此把全部增益归因给某一个组件。

曲线中不连续的部分还涉及 checkpoint merging。不同 scaffold、不同配置的 RL run 会走上不同的优化路径，DeepSeek 将这些 checkpoint 合并，再用它初始化下一轮 RL。几条并行路径学到的能力因此可以重新汇合，然后继续训练。这种做法延长了有效 RL compute，也改善了 token 效率，更像训练编排上的接力，而不是一套新 RL 算法。

## DSec 先解决百万级环境

并发沙箱达到百万级后，问题不只在 PPO 或 GRPO 公式。数据中心还得调度、隔离并清理大量会主动操作系统的 Agent。DSec，也就是 DeepSeek Elastic Compute，承担了这部分工作。

它把计算节点划成多个 shard，也叫 scale unit，用来隔离实验并缩小故障范围。调度没有采用代价很高的全局强一致方案。多个 placement engine 副本各自根据近期观测做出足够好的放置决定，系统接受最终一致性。

但一致性放松后，安全底线仍留在节点本地。每个节点都有一道硬约束：新任务一旦会让资源超过本地告警阈值，就直接拒绝。中央调度可以近似，节点负责兜底，避免全局协调在百万容器规模上成为瓶颈。

单节点密度也被推得很高。DSec 使用硬件支持的 sub-NUMA 分区，把 worker VM 绑定到独立 NUMA domain，并把容器的 CPU 和内存限制在对应的本地 NUMA 资源内。在可比负载下，每台物理节点承载的并发存活容器从约 1000 个提高到 2500 个以上，之后才出现可以测出的端到端退化。

密度太高会干扰时间敏感的评测。为此，系统区分 latency-sensitive 执行类型，降低非敏感任务的调度优先级，也约束同一超线程兄弟核上的任务类型。Agent 可能删除系统文件、尝试漏洞，甚至从镜像中偷答案，DSec 用 AppArmor 和 eBPF 网络策略做隔离。环境若被破坏，系统会记录失败轨迹，再把后果信号送回 RL。

## 异步 RL 也有代价

同步 rollout 很容易被长尾拖住。大部分样本已经结束，整组计算还在等待最慢的那一个。V4.1 的后训练基础设施把几乎所有 RL 和 OPD 任务改为异步生成，用高并发盖住长尾。

调度粒度落到 sample。每当新完成的样本数达到下一个 prompt 所需的 GRPO group size，系统就继续发任务，不必等某个旧 group 全部结束。吞吐提高了，但随之出现两类统计问题。

一类是 length bias。短序列更早结束，训练初期的 batch 容易被短样本占满。系统会按数据集限制并发，通过来源配比间接控制长度分布；也允许丢弃过早返回的短样本，让训练逐渐进入稳态，减少对“越短越先被训练”这一偏差的拟合。

另一类是 off-policy。部分 token 是旧 checkpoint 生成的，不再严格来自当前策略。系统通过调度与等待条件限制最大的 off-policy 比例，并对陈旧度过高的 token 做 loss masking，不让它们贡献梯度。遇到跨 checkpoint 样本，则保留各段 rollout 当时的专家路由并进行拼接，而不是用新模型重新计算。

系统还支持 token 级中断。训练拿到足够样本后，进行中的生成可以很快停止；KV cache 和专家路由会按 token 保存，切换 checkpoint 后再从原位置恢复。省下来的不只是生成时间，也包括已经发生过的环境交互。

## 算法没变，训练对象变大了

读完这一节，我的感觉是，Agent 后训练正在从“设计一个更好的 loss”，逐渐变成一项任务生产和系统工程。

真实失败要能变成任务，公开仓库也要能批量造题；环境既要保留原意，又要可复现、可判分并且抗作弊；rollout 要跨 scaffold；不同训练路径通过 checkpoint merging 接起来；百万级沙箱交给 DSec 承载；异步训练拿到吞吐后，还要处理 length bias 和 off-policy 留下的偏差。

因此，“没有新 RL 算法”并不等于没有工程创新。这次变化主要落在 RL 能看到哪些数据、数据在哪里执行，以及漫长的交互如何稳定持续。训练对象从几千道静态题，扩展成一套会自动造题、搭环境、验证答案、运行轨迹和检查作弊的生产系统。Agent 的提升，也就不必只从算法名字里寻找解释了。
