---
title: "ATLAS: Agentic Harness for Whole-Body Intelligence from Human Video"
paper_url: "https://atlas-humanoid.github.io/paper/atlas_tech_report.pdf"
authors: "Junli Ren, Kaixiang Yao, Junfeng Long et al."
venue: "UC Berkeley 技术报告"
published: "2026"
read_date: "2026-11-26"
read_at: "2026-11-26T15:56:32+08:00"
status: "已读"
tags: ["Agentic Systems", "Training & Optimization", "Evaluation"]
search_terms: ["人形机器人", "System 2", "System 1", "System 0", "ScaleBFM", "测试时学习", "末端补偿", "Qwen3-VL", "Isaac Lab"]
one_liner: "ATLAS 从一段人类示范提取全身动作，把手的接触点对齐到眼前物体，再用机器人的实际尝试修正伸手偏差和视觉判断。"
---

## 研究问题

人能示范怎么开柜门，机器人也能模仿人的姿势，但换一个柜子，把手的位置就变了。即使相机找对了把手，机器人也可能每次都差几厘米。ATLAS 问的是，能否只给一段相关视频和一句指令，让已有的全身控制器在新场景完成接触任务，并从失败中改进？[项目演示](https://atlas-humanoid.github.io/#method)

## 核心方法

**System 2 说明什么时候碰什么。** 视觉语言模型读示范和指令，写出任务阶段、接触帧、要找的物体或部件、末端如何交互，以及什么可见结果算成功。GVHMR 从视频恢复人体运动，GMR 将其映射到 G1。头部 RGB-D 相机配合 SAM 3，给出当前场景的三维接触点。[论文图 3、§3.2](https://atlas-humanoid.github.io/paper/atlas_tech_report.pdf#page=4)

**用编辑后的参考轨迹连接 System 2 和 System 0。** 以开门为例，保留示范里的下蹲和伸手姿态，把最后一段伸手逐步移向实际把手。接触帧为 $\tau$，$c_t^{\mathrm{ref}}$ 是摆放到当前站位后的参考末端接触点，$p^\star$ 是相机定位的目标。$\alpha_t$ 随最后一段伸手走过的距离从 0 增至 1，因此接触时 $n_\tau=p^\star$。[论文式 1](https://atlas-humanoid.github.io/paper/atlas_tech_report.pdf#page=4)

$$
n_t=c_t^{\mathrm{ref}}+\alpha_t\bigl(p^\star-c_\tau^{\mathrm{ref}}\bigr).
$$

**快系统纠正执行偏差。** 冻结的 ScaleBFM 以 50 Hz 跟踪参考。快系统记录发出的腕部指令和从关节状态算出的实际末端位置，在多次尝试间学习局部运动响应，再训练一个输出小幅补偿 $u_t$ 的策略。执行指令可写成 $n_t+u_t$，误差仍相对 $n_t$ 计算。空抓留下的运动轨迹也能用于更新。[论文图 4、§3.4](https://atlas-humanoid.github.io/paper/atlas_tech_report.pdf#page=5)

**慢系统纠正感知与判定。** 如果分割器总找错物体，或机器人把空抓判成成功，Agent 回看头部相机画面，改写找物描述或成功条件。先在留出的记录上比较新旧规则，通过后才在后续尝试启用。[论文 §3.5](https://atlas-humanoid.github.io/paper/atlas_tech_report.pdf#page=6)

## 关键发现

- **快系统能把手伸得更准。** 图 9a 固定苹果位置，用头部相机定位。在最后一组仿真尝试中，合爪前保持阶段的末端误差中位数为无补偿 **4.6 cm**、固定偏置 **2.1 cm**、递归最小二乘 **2.5 cm**、ATLAS **1.3 cm**。固定偏置已能纠正相当一部分误差，是复现时应保留的基线。[图 9a、§4.6](https://atlas-humanoid.github.io/paper/atlas_tech_report.pdf#page=9)
- **真机抓取也改善，但仍经常失败。** G1 抓起物体并握住站直才算成功。无补偿与在线补偿分别成功 **15/60、38/60** 次。分物体看，香蕉 **3→11**、罐子 **7→16**、苹果 **5→11**，每项各 20 次。学习组先做 5 次热身，各组独立运行，苹果学习组额外做了相机校正。[表 3](https://atlas-humanoid.github.io/paper/atlas_tech_report.pdf#page=9)
- **慢系统有收益，也有代价。** 在四段原描述失效的真机记录中，180 张目标可见的留出帧上，找物召回率从 **18.9% 升到 78.3%**。另一组 103 次真机尝试回放中，空抓误报成功从 **30/72 降到 10/72**，同时真实抓住却判失败从 **9/31 增到 14/31**。[表 8、表 9](https://atlas-humanoid.github.io/paper/atlas_tech_report.pdf#page=18)
- **一段示范能覆盖相近任务。** 在 SIMPLE 的三种取箱任务上，同一段人类视频得到 **29/30、28/30、30/30**。用每任务 100 条机器人示范微调的基线，各任务最高分别为 **30/30、29/30、30/30**。这里验证的是同类取箱动作在不同条件下的复用。[表 1](https://atlas-humanoid.github.io/paper/atlas_tech_report.pdf#page=7)

## 我的提问

### Q1：相对 System 2 → 1 → 0，真正新增了什么？

OKAMI 已能从单段人类视频按物体调整动作，KINO 已把 VLM 给出的关键姿态接到全身控制器。ATLAS 的增量在两层之间：用示范轨迹、接触几何和在线补偿承担通常交给 System 1 的动作生成工作，省去为新任务训练中间 VLA 的步骤。它也给两种失败留出了不同的改法：看错把手时重写视觉描述，找对把手但够偏时修正腕部指令。底层复用已预训练的 ScaleBFM，能力仍受示范和跟踪器约束。[相关工作、§3](https://atlas-humanoid.github.io/paper/atlas_tech_report.pdf#page=3)

### Q2：System 2 要微调吗？开源 Qwen 能替代到什么程度？

作者直接使用已有 VLM，默认是 Claude Opus 5.5。表 7 让每个候选模型对同一任务只写一次规格。下表的“可执行”指通过格式、词汇与帧检查；举箱、开门是通过检查后在 Isaac Lab 实际执行的结果，各 20 次，另有 5 次热身。[表 7](https://atlas-humanoid.github.io/paper/atlas_tech_report.pdf#page=18)

| 任务规格模型 | 可执行任务数 | 举箱 | 开柜门 | API 输入 / 输出价¹ |
| --- | --- | --- | --- | --- |
| Claude Opus 5.5 | 7/7 | 75% | 65% | 4 / 20 |
| Claude Haiku 5.5 | 7/7 | 80% | 0%，用错手 | 0.10 / 0.50 起 |
| Claude Sonnet 5.5 | 5/7 | 0% | 规格未通过 | 2 / 10 |
| GPT-6.2 Astra | 5/7 | 60% | 100% | 8 / 40 |
| Qwen3-VL-32B | 6/7 | 20% | 拒绝生成规格 | 开放权重，自部署计硬件成本 |

¹ 2026-11-26 查询的标准短输入价格，美元／百万 token。论文未披露各次调用的 token 用量，无法从表里算出每个成功任务的花费。Qwen 32B 可在任务开始前生成规格并保存；运行时看头部相机、辅助判断是否抓住的是另一个模型 **Qwen3-VL-4B**。两个模型都无需按 50 Hz 调用。[Claude 定价](https://platform.claude.com/docs/en/about-claude/pricing) · [Astra 定价](https://developers.openai.com/api/docs/models/gpt-6.2-astra) · [Qwen 官方权重](https://huggingface.co/Qwen/Qwen3-VL-32B-Instruct) · [执行判定](https://atlas-humanoid.github.io/paper/atlas_tech_report.pdf#page=20)

### Q3：从什么实验开始复现？需要真机、渲染或 5090 吗？

先固定抓取动作和任务规格，直接读仿真里的苹果位置与机器人位姿，比较无补偿、固定偏置和学习补偿。论文的随机摆放实验也使用真值目标。这个阶段只需要物理状态，可以省去相机渲染和 32B 服务。要对照图 9a 的绝对误差，再接入头部 RGB-D、分割和机器人自身定位；这一阶段可以离屏渲染。要对照表 3 的握持成功率，才需要 G1 真机。[附录 H](https://atlas-humanoid.github.io/paper/atlas_tech_report.pdf#page=22)

工程上先跑通冻结跟踪器，再接坐标变换、腕部补偿和按次记录，最后接入 VLM 生成规格与复核。ScaleBFM 已有公开代码和权重；截至阅读日，ATLAS 官方仓库列出的是项目网站，任务执行与在线更新逻辑仍要按报告实现。ScaleTrack 的公开安装说明固定 Isaac Sim 5.1.0；该版 x86 工作站最低配置为 **RTX 4080、16 GB 显存、32 GB 内存**，5090 并非必需。这是仿真器配置，同时加载大模型要另算显存。[ScaleTrack](https://github.com/zengweishuai/ScaleBFM/blob/main/ScaleTrack/README.md) · [跟踪器权重](https://huggingface.co/WeishuaiZeng/ScaleBFM) · [ATLAS 官方仓库](https://github.com/orgs/atlas-humanoid/repositories) · [Isaac Sim 硬件要求](https://docs.isaacsim.omniverse.nvidia.com/5.1.0/installation/requirements.html)

## 局限与疑问

示范要包含与任务相关的动作，跟踪器也要具备相应能力；楼梯演示用的控制器事先在通用楼梯数据上微调过。每个任务步骤的快系统都从当前会话的尝试中学习，首试成功与多次重试后的成功应分开看。真机抓取的标定差异，也限制了对补偿收益大小的归因。[§3.3](https://atlas-humanoid.github.io/paper/atlas_tech_report.pdf#page=5) · [图 9、表 3](https://atlas-humanoid.github.io/paper/atlas_tech_report.pdf#page=9)

## 我的判断

最值得复现的是一个能单独验证的问题：当物体找对了、全身控制器也冻结了，学习“这只手实际怎样响应指令”，比记住一个固定偏置能多纠正多少误差？图 9a 给出了这个对照。实际做时应同时记录末端位置误差和抓住后站直的成功率；前者改善几厘米，能否换来后者，要在具体任务上检验。

## 下次只看这些

1. **图 3、式 1**：理解人类示范的接触点怎样对齐到眼前物体。[作者方法原图](https://atlas-humanoid.github.io/media/fig_method_paper.jpg)
2. **图 9a、表 3**：回看固定偏置与学习补偿的差距，以及这项改进在真机上对应多少次成功。[实验](https://atlas-humanoid.github.io/paper/atlas_tech_report.pdf#page=9)
3. **表 7–9**：区分规格能解析、机器人真能完成、相机能正确判断成功这三件事。[模型与判定对照](https://atlas-humanoid.github.io/paper/atlas_tech_report.pdf#page=18)
