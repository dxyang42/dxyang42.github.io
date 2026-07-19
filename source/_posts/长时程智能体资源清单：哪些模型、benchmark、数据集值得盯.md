---
title: 长时程智能体（二）：模型、benchmark、数据集，一份值得收藏的清单
date: 2026-07-19 22:30:00
tags:
  - AI
  - Agent
  - 论文
categories:
  - 读书笔记
cover: /img/long-horizon-resources-cover.png
description: Towards Long-Horizon Agents 综述资源篇：五类应用领域的 benchmark、值得关注的开源模型、训练数据集与环境、框架协议，一份可以收藏慢慢用的清单。
---

这个系列的[第一篇](https://dxyang42.github.io/2026/07/19/149页综述读完，我把长时程智能体的术语挨个拆了一遍/)把《Towards Long-Horizon Agents》的术语挨个拆了一遍，这篇干点更实际的：把里面点名的、可以拿来就用的资源整理成清单。

综述引了差不多 900 篇，但真值得收藏的没那么多。我按"评测—模型—数据与环境—框架"四类整理，全部以论文实际提到的为准。

看榜单之前有句话得先放这：论文反复强调，**一个 benchmark 分数反映的是模型 + harness + 训练配方的联合贡献**。只看模型名没意义，得看它套的什么 scaffold、给了多少预算，不然分数没法横向比。

## 一、Benchmark：按五个应用领域分

### 软件工程

- **SWE-bench / SWE-bench Verified**（2024，OpenAI 人工筛的 500 题版）：仓库级 issue 修复，事实标准。
- **SWE-bench Pro**（2025.09，Scale AI）：更长程的企业级仓库任务。
- **Terminal-Bench 2.0**（2025.11）：难且真实的命令行任务，Kimi K3 那篇博客里我也提过它。
- **OctoBench**（2026）：scaffold 感知的指令遵循评测，专门把"模型 vs 脚手架"的归因拆开来，这个方向很重要。
- **DebugBench**（2024，清华）：多语言 bug 诊断。
- **NL2Repo-Bench / RepoZero**：从零生成整个仓库，测的是超长程生成。
- 训练向环境：**SWE-Gym**、**SWE-smith**、**R2E**，从真实仓库/issue/commit 重建可执行任务，拿来训练用。

### 信息检索（Deep Search / Deep Research）

- **BrowseComp**（2025，OpenAI）：网页浏览 agent 的简单但硬核 benchmark。
- **AssistantBench**（2024）：现实耗时的网页任务。
- **WideSearch**（2026，字节）：广度信息搜集，测并行调查能力。
- **DeepResearch Bench**（2026）：深度研究报告质量评测。
- **GISA**（2026）：通用信息搜集助手 benchmark。
- **MMInA**（2025）：多跳多模态网页任务。
- **DeepConsult**（2025）：咨询/商业类 deep research 评测。
- 系统侧值得关注：**Tongyi DeepResearch**（通义，开源）、**Open Deep Research**、**MiroThinker**、**WebDancer / WebSailor** 家族（既产系统也产合成数据管线）。

### Computer Use（GUI 操作）

- **WebArena**（2024）：自托管的真实网站环境，经典中的经典。
- **OSWorld**（2024）：真实桌面环境开放式任务，GUI agent 的事实标准。
- **Mind2Web**（2023）：跨域网页操作演示数据。
- **AndroidWorld / AndroidLab**：移动端动态评测环境和训练框架。
- **ScreenSpot-Pro**：高分辨率专业软件 GUI 定位。
- **Agent-SafetyBench / ST-WebAgentBench / OS-Sentinel**：GUI agent 的安全评测，做生产的要盯。
- 系统：**UI-TARS / UI-TARS-2**（字节，原生视觉 GUI 感知）、**WebVoyager**、**browser-use**（开源浏览器控制）、**Mobile-Agent-v3.5**。

### 多模态

- **Video-MME / LongVideoBench / EgoSchema / MVBench**：长视频理解的基线四件套。
- **StreamingBench**：流式视频理解。
- **OmniVideoBench**（2026）：音视频联合理解。
- **OmniGAIA**（2026，就是这篇综述团队自己的）：跨模态工具使用推理。
- **AgentVista**（2026）：多模态 agent 综合评测。
- 注意论文自己的提醒：这些视频理解 benchmark 严格说不是长时程 agent 任务，收进来是因为它们测的是底层感知能力。

### 通用与生产力

- **GAIA**（2023）：通用助手任务，被引用最多的综合榜之一。
- **τ-bench**（2025，Sierra）：有状态的工具- agent-用户交互，每次调用会改后端状态，比单次 function calling 评测真实得多。
- **AppWorld**（2024）：可控制的应用世界，状态化个人任务。
- **BFCL**：函数调用榜单，单次调用能力看这。
- **Toolathlon / MCP-Mark / MCP-Atlas**（2025-2026）：真实 MCP 工具生态下的大规模压测。
- **MLE-bench**（OpenAI）：机器学习工程任务。
- **PaperBench**（OpenAI）：端到端复现论文，极长程。
- **GDPval**：真实经济任务价值评测。
- **METR time-horizon**：不是传统榜单，是测量方法论：50% 成功率下 agent 能完成的任务时长。目前 frontier 数字：o3 约 2 小时、GPT-5.4 约 5.4 小时、Gemini 3.1 Pro 约 6.4 小时、Claude Opus 4.6 约 12 小时、Claude Mythos 16 小时以上。
- **Agents' Last Exam**（2026）：前沿专业工作流评测，名字就很狂。

## 二、模型：按用途分

**开源通用基座**（搭 agent 的底座）：

- **DeepSeek-V3 / R1**：MLA 架构（论文专门聊了 MLA 是"显式上下文和压缩状态之间的特例"），R1 证明了自验证可以靠规则奖励的 RL 涌现。
- **Qwen2.5 / Qwen3 / Qwen3-Coder-Next**：后者显式做了面向仓库级理解和 agent 交互的中训练。
- **GLM-4.5 / GLM-5.2**：GLM-5.2 上了 1M 上下文 + 长程编码场景专门训练。
- **Kimi K2 / K2.5**：大 MoE + MLA；Kimi Linear 是混合架构（线性路径 + 全局 MLA）的代表。

**架构研究向**（如果关心长上下文怎么做）：Longformer、BigBird、RetNet、RWKV、Mamba、Jamba（Transformer+Mamba+MoE 混合）。

**多模态**：Qwen2-VL、InternVL3、Cambrian-1、Seed1.5-VL/1.6（视觉接地和 GUI 控制当原生能力训）、MiMo-VL、Qwen3.5-Omni。

**安全护栏**（生产必备，论文里单列了一类）：AgentDoG / AgentDoG 1.5（风险分类 + 轨迹根因诊断）、ToolSafe 的 TS-Guard（工具调用步级护栏）、AdaptiveGuard、AGrail（终身学习的安全检查）。

**Frontier 商业系统**（论文点名的）：Claude Code、Codex、OpenHands、Devin、OpenAI/Gemini Deep Research、Perplexity、Anthropic Computer Use、OpenAI Operator、Cursor。

## 三、数据集与环境：训练用

- **轨迹数据**：**TOUCAN**（真实 MCP 环境的大规模工具轨迹）、**AgentInstruct / AgentTuning**、**Agent-FLAN**、**AgentBank**、**OpenResearcher**（深度研究轨迹）。跨数据集统一 schema 看 **Agent Data Protocol (ADP)**。
- **合成任务生成**：WebShaper、WebSailor（知识图谱游走 + 信息遮蔽）、TaskCraft、AgentSynth、SWE-smith、CLI-Gym。
- **可执行环境**（既能评测也能训练）：WebArena、OSWorld、SWE-Gym、Terminal-Bench、TheAgentCompany、AppWorld、τ-bench。
- **SFT 精选小集**：LIMO（817 条）、s1（1K），"少即是多"路线的代表。
- **预训练语料**（自己做 mid-training 的话）：DataComp-LM、FineWeb、Dolma、OLMo。

## 四、框架、协议与工具

- **协议层**：**MCP**（工具连接事实标准）、**A2A**（Google，agent 间通信）、**IBM ACP**、**AGENTS.md / Agent Skills**（便携的 harness 配置和技能文件）。
- **编排**：**LangGraph**（DAG 编排，状态可恢复可审计）、**AutoGen / Magentic-One**、**MetaGPT**。
- **Agent runtime**：**OpenHands**（持久工作区）、**SWE-agent**（agent-computer interface 的标杆）、**Aider**（repo map 的思路很值得抄）、**OpenManus**、**browser-use**。
- **记忆**：**MemGPT**（OS 式分页记忆）、**Mem0**、**HippoRAG**（图记忆联想）、**ReasoningBank**（蒸馏可复用推理策略）、**Voyager**（可执行技能库，skill 路线的起点）。
- **上下文压缩**：LLMLingua、ReSum、CompAct。
- **训练基础设施**：**Agent Lightning**（微软，把已有 agent 的执行轨迹转成 RL 可训练单元，这个对想给自己系统上 RL 的人最实用）；RL 算法开源实现一抓一大把：DAPO、Dr.GRPO、GSPO、Tree-GRPO、ARPO 等。
- **验证器**：CRITIC（工具接地的 verify-then-correct）、AgentPRM、Web-Shepherd（低成本 web 导航 PRM）。

## 五、怎么持续跟进

1. **Awesome-Long-Horizon-Agents**（github.com/RUC-NLPIR/Awesome-Long-Horizon-Agents）：论文配套仓库，按章节组织的论文清单，持续更新，还接受 PR。这是最该 star 的一个。
2. **long-horizon-agents.github.io**：项目主页，图做得比 PDF 里还清楚。
3. **metr.org/time-horizons**：时间跨度测量的源头，隔几个月看一次 frontier 数字更新。

## 我自己的用法

做应用的话，我会先盯 **τ-bench、SWE-bench Verified、OSWorld、BrowseComp** 这四个有区分度的榜。框架从 **LangGraph + MCP + 一个记忆方案**起步。看别人报 benchmark，一律先问三样：模型、harness、预算。

做研究的话，值得挖的坑论文 §7 点得挺明白。harness 可迁移性是一个，同一 harness 换个模型排名能差几十分。budget-aware agency 也算一个，现在的 agent 全是预算盲。还有环境合成的保真度，faithfulness 这东西该像成功率一样被显式测量。

清单里大部分都有开源仓库，慢慢翻。模型侧的优化细节在[第三篇](https://dxyang42.github.io/2026/07/19/长时程智能体的模型内功：能力是怎么长进权重里的/)里拆了，想让我再深挖哪个方向，留言说。

**系列导航：**
- [（一）149 页综述，先把术语挨个拆清楚](https://dxyang42.github.io/2026/07/19/149页综述读完，我把长时程智能体的术语挨个拆了一遍/)
- [（二）模型、benchmark、数据集，一份值得收藏的清单](https://dxyang42.github.io/2026/07/19/长时程智能体资源清单：哪些模型、benchmark、数据集值得盯/)
- [（三）模型内功，能力是怎么长进权重里的](https://dxyang42.github.io/2026/07/19/长时程智能体的模型内功：能力是怎么长进权重里的/)
- [（四）五大应用领域，九个前沿问题](https://dxyang42.github.io/2026/07/19/长时程智能体的应用地图和九个前沿问题/)

---

*参考来源：*
- *[Towards Long-Horizon Agents: A Survey（OpenReview）](https://openreview.net/forum?id=HyhfhlbWGh)*
- *[Awesome-Long-Horizon-Agents（论文配套仓库）](https://github.com/RUC-NLPIR/Awesome-Long-Horizon-Agents)*
- *[METR time-horizon 测量](https://metr.org/time-horizons)*
