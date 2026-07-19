---
title: CZ Biohub 在 Kaggle 发了个比赛：用 AI 追踪发育中的每一个细胞
date: 2026-07-18 17:20:00
tags:
  - AI
  - 生物技术
  - Kaggle
  - 细胞追踪
categories:
  - 技术
description: CZ Biohub 联合 Kaggle 发起细胞追踪比赛——3D 显微视频里检测、追踪斑马鱼胚胎细胞，还要认出分裂事件。数据集、评分、官方 baseline 一次说清。
cover: /img/kaggle-cell-tracking-cover.png
top_img: /img/kaggle-cell-tracking-cover.png
---

搞过细胞成像的朋友应该都有心理阴影：显微视频一开，几百个细胞在里头挤来挤去，然后你得一帧一帧手动标它们谁是谁、谁跑哪去了、谁刚分裂了。这活儿在发育生物学实验室里属于祖传体力活，一般落在运气不好的研究生头上。

现在 Chan Zuckerberg Biohub 把这事搬上了 Kaggle，比赛名叫 **Biohub: Cell Tracking During Development**。意思很直接：这苦差以后让算法干。

## 具体要干什么

给你一段斑马鱼胚胎的 3D 荧光显微视频（带时间轴的那种），你要做三件事：把每个细胞找出来，把相邻帧的同一个细胞连起来，再把细胞分裂的时刻认出来。

输出就是一张图。节点是细胞，坐标 (t, z, y, x)；边是跨帧的链接；分裂长成一个节点分出两条边。做到位了，一个受精卵怎么变成一坨有模有样的组织，谱系树就全出来了。

## 数据是真家伙

不是合成的玩具数据，是光片显微镜实拍的斑马鱼胚胎：

- 每个样本 `(100, 64, 256, 256)`，100 帧，uint16，Zarr v3 格式
- 物理分辨率 z=1.625 µm，xy=0.40625 µm
- 标注是稀疏的，只有一部分细胞有真值。训练的时候你得习惯“老师只批改了一部分作业”
- 训练集和测试集按胚胎严格分开。想靠背下某个胚胎的长相混分？人家防着呢

稀疏标注加密集细胞加成像噪声，难度基本就藏在这几个词里。

## 评分

```
score = adjusted_edge_jaccard + 0.1 × division_jaccard
```

边占大头，分裂检测占小头。边的判定挺严格：预测的细胞先跟真值做二分图匹配（质心距离 7 µm 以内才算同一个），一条边两头都对了才算对。细胞报多了还会挨罚。另外因为标注是稀疏的，分数有补偿机制，理论上是可能超过 1.0 的，第一次看到分数破百别慌。

指标代码开源在 [royerlab/kaggle-cell-tracking-competition](https://github.com/royerlab/kaggle-cell-tracking-competition)，可以自己先扒。

## 官方 baseline 挺有诚意

Royer Lab 自己放了个 baseline，不是糊弄人的那种：

检测用 3D U-Net 加时间注意力，出逐体素特征和检测热图，局部极大值抑制取细胞中心。链接这块，把检测到的中心特征喂给 cross-attention transformer，给所有相邻帧的细胞对打分。检测和链接端到端一起训，稀疏标注的处理也干脆，没标注的边直接不参与反向传播。

官方还特地说了句“没训到收敛”。翻译一下：上分空间留好了，各位请。

## 这比赛哪来的

2024 年 CZ Biohub SF 在 *Cell* 上发了 Zebrahub，一个斑马鱼发育的细胞图谱：受精后头 24 小时里几乎每个细胞的 3D 时序影像，外加 10 个时间点、12 万个细胞的单细胞表达数据。

为了拍这些片子，他们专门造了台叫 DaXi 的光片显微镜，视野大到能装下整个活胚胎；为了追踪，又写了 Ultrack 来自动识别细胞核。这次比赛说白了就是把实验室内部最疼的那个问题开放给全世界。追踪算法要是真被卷出来了，他们那个“发育生物学版 Google Earth”的拼图就算补上了关键一块。

## 参赛信息

Research Code 赛，提交 notebook 的那种。没现金奖，给积分和奖牌。截稿还剩两个月左右，具体日子看 Kaggle 页面。

上手路线很顺：下数据，clone 官方 repo，`uv run python scripts/train_unet_transformer.py` 先跑通，然后开始魔改。

## 最后

细胞追踪这事，往小了说是图像算法题，往大了说是“一个细胞怎么变成一个生命体”这条路上最基础的数据基建。谁能在 dense 细胞群里把谱系重建做准，发育生物学、免疫、再生的实验室都会排着队感谢你。

反正不要报名费。搞视觉的、搞生信的，去会会这群斑马鱼呗。

---

*参考链接：*

- *比赛：[kaggle.com/competitions/biohub-cell-tracking-during-development](https://www.kaggle.com/competitions/biohub-cell-tracking-during-development)*
- *baseline：[github.com/royerlab/kaggle-cell-tracking-competition](https://github.com/royerlab/kaggle-cell-tracking-competition)*
- *Zebrahub：Lange et al., Cell, 2024*
