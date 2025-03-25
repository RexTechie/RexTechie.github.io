---
title: "PlanGEN: A Multi-Agent Framework for Generating Planning and Reasoning Trajectories for Complex Problem Solving"
date: 2025-03-25T11:52:08+08:00
draft: false
description: "Agent生成框架"
categories: ["📒论文笔记"]
tags: ["Agent",]
---


## 基本信息

- **标题**: PlanGEN: A Multi-Agent Framework for Generating Planning and Reasoning Trajectories for Complex Problem Solving
- **作者**: Mihir Parmar、Xin Liu等
- **作者单位**: Google、Arizona State University
- **期刊/会议**: ArXiv
- **发表年份**: 2025
- **DOI**: [2502.16111](https://arxiv.org/abs/2502.16111)
- **开源地址**: 🈚️
- **关键词**: Agent

---

## 研究背景与问题 (Background & Questions)

有效的计划是设计在解决复杂现实世界问题的系统的关键组件。传统的用模板化的方式缺乏通用性。 LLM 可以较好的做一些规划任务，比如用 LLM 在自然语言中做规划可以解决一些代码问题。因此作者考虑增强 LLM 生成计划的能力，并证明其在科学和金融领域下游任务的实用性。

---

## 核心贡献 (Key Contributions)

总结本文的主要贡献点：

1. PlanGEN，一种新颖的、模型无关的、可扩展的多智能体框架，用于增强LLM的自然规划。
2. 在一些复杂规划和推理的 Benchmarks 上达到了 SOTA 级别。
3. 一种基于约束的验证和基于实例级复杂性的推理算法选择的新方法。

## PlanGEN

### LLM Agents

![PlanGEN Overview](/images/20250325144915.png)
PlanGEN 包括三个 大模型 Agent：Constraint Agent、 Verification Agent 和一个 Selection Agent。每个Agent使用现成的 LLM（如：Gemini, GPT）。

### Contstraint Agent

“约束”定义为验证问题解决方案所必须的标准。这些标准是具体例子特定的，比如：在 NATURE PLAN 的日历安排中，包括“个人计划”、“可用性”、“偏好”；在 GPQA 的科学推理问题中，约束可能是“使用的概念”、“计算正确性”和“公式选择”。作者认为，提取实例的约束对于成功验证至关重要。Constraint Agent是框架中的预处理组件，负责从问题描述中提取约束。通过分析输入问题，该Agent可以确定生成Plan验证所需的所有可能的关键约束。提取的约束为验证提高 Planning 过程整体相关性和质量的计划提供了基础。Constraint Agent 使用的 Prompt 使其能够通过要求 LLM 关注问题描述的特定方面，系统地识别约束。这样可以确保没有忽略关键信息，并且最终的约束是全面的。

### Verification Agent

Verification Agent在框架中发挥着关键作用，通过评估由Constraint Agent生成的约束条件所生成的计划的质量。这个 Agent 用来确保计划和任务目标保持一致，遵守约束，并在逻辑上朝着正确的解决方案进行进展。Verification Agent有两个关键组件：(i) Feedback Generation，以及 (ii) Numerical Reward Generation。

**Feedback Generation** 当验证约束与生成的计划，Verification Agent 会生成有关计划质量评估的详细推理。作者把这一过程视为“反馈”，提供了可解释性和可行的下一步改进。

**Numerical Reward Generation** 受到Chain-of-experts的启发，作者让 Agent 评估生成的计划和约束，并给出-100 ～ 100的奖励分数。评分机制要求按照严格的质量标准，其阈值表示为高质量的计划（如95）。

### Selection Agent

Selection Agent 根据任务的复杂性动态的选择最合适的推理算法。它利用历史表现、多样性和恢复分数以及来自大模型的指导，自适应地选择当前任务的最佳算法。为了创建 Selection Agent，作者使用了一个修改后的置信区间上界算法（UCB，Upper Confidence Bound）策略。这个策略结合了多个因素：归一化奖励、探索奖金、多样性调整和恢复分数。此外，该 Agent 整合了LLM的先验知识，根据问题描述、任务要求和先前计划（如有）提供算法适用性评分。这些先验条件使 Agent 能够将其选择的算法与输入实例的复杂性和相应的约束对齐，提高所选算法的相关性。

**Modified UCB Policy** 方程结合了多个术语以平衡在给定任务实例中选择最佳算法时的利用和探索。

$UCB(a) = \frac{R(a)}{N(a)} + \sqrt{\frac{2 \log(T+1)}{N(a)}} + \lambda_{\text{prior}} \cdot \text{Prior}(a) + \frac{\alpha_{\text{diversity}}}{N(a)+1} + \alpha_{\text{recovery}} \cdot S_{\text{recovery}}(a)$

第一项$\frac{R(a)}{N(a)}$表示的是算法 a 的平均奖励得分，其中 $R(a)$ 表示算法 a 的累积的总奖励，$N(a)$ 表示算法 a 被选择的次数。这一项确保优先根据历史经验选择。

第二项$\sqrt{\frac{2 \log(T+1)}{N(a)}}$ 是探索部分，鼓励选择尝试次数较少的算法。

第三项$\lambda_{\text{prior}} \cdot \text{Prior}(a)$ Prior(a) 是基于LLM（大语言模型）的先验评分，用于根据实例复杂性调整算法选择。在这里，λprior是一个动态衰减的权重，定义为λprior = 1+T，其中T代表试验的总数。

第四项$\frac{\alpha_{\text{diversity}}}{N(a)+1}$ 这一项惩罚被过度使用的算法，确保所有选项的平衡探索。

第五项$\alpha_{\text{recovery}} \cdot S_{\text{recovery}}(a)$ S_recovery(a) 是算法 a 的恢复评分，奖励能够有效从失败中恢复的算法。

**Selection Process** 选择过程首先要初始化算法的各个变量值，然后，智能体结合LLM引导的先验知识，根据问题陈述和任何提供的反馈为算法生成适宜度分数。这些先验知识来自LLM，并作为调整UCB值的初始估计。

### Framework

#### PlanGen(Best of N)

受 RePrompt 启发，作者改编了 Best of N 算法，并使用他们自己的 Constraint 和 Verification Agent 对其进行修改。这个框架生成 N 个候选Plan（Plan 1, Plan 2, ...Plan n），每个 Plan 由 Verification Agent 根据一组约束进行评估。然后，Verification Agent 分配相应的Reward（Reward 1, Reward 2, ...Reward n）。最后选择 Reward最大的方案，确保得到一个最优解，该解最好地满足问题约束。

其过程如图所示：
![PlanGen(Best of N)](/images/20250325152212.png)

#### PlanGen(ToT)

受 Tree of Thoght 启发，作者用 Constraint 和 Verification 修改了 ToT 算法。该方法首先初始化一个代表问题的根节点，并生成多个潜在的下一步步骤，创建一个树状结构。生成的步骤通过Verification Agent 进行验证，该Agent根据一组约束条件分配奖励分数。遍历过程包括评估给定深度的所有可能步骤，根据奖励分数选择最有希望的路径，并通过生成新步骤进一步扩展它。

其过程如图所示：
![PlanGen(ToT)](/images/20250325153216.png)

#### PlanGen(REBASE)

该框架集成了动态选择和扩展策略，以迭代优化解决方案。在树的每个深度，候选节点根据其分配的奖励分数（使用Verification Agent获得）进行排名，确保首先探索最有希望的候选者。即使奖励较低的步骤也被考虑，但随着子节点数量的减少，意味着它们的探索深度有限。这种层次化剪枝有助于保持效率，从而减少对较弱节点的无谓探索。这个过程会持续进行，直到找到有效、完整的解决方案或达到预定义的深度或宽度限制。此外，还有一个类似于ToT的完成检查，它可以识别代表完整解决方案的节点，一旦确定了令人满意的结果，REBASE就可以提前终止。

其过程如图所示：
![PlanGen(REBASE)](/images/20250325153944.png)

#### PlanGen(Mixture of Algorithms)

算法混合框架（图1）引入了一个Selection Agent，该 Agent 根据实例级复杂度动态选择上述章节中提出的inference-time algorithm。该框架以模块化和迭代的方式运行，确保在有效解决不同复杂度的规划和推理问题时具有适应性。

**Orchestration** 这个过程开始于用 LLM 根据任务和问题描述生成一个初步的计划。与此同时，Constraint Agent来生成特定任务和问题的约束集。根据约束条件，Verification Agent 评估初步计划的质量并提供一个奖励分数。如果初步计划达到所需的阈值，则可作为最终计划接受，否则，开始Iterative Refinement过程。

**Iterative Refinement** 这个优化循环由一系列推理算法驱动。在这次Iterative Refinement过程中，Selection Agent 根据实例特定的复杂性和历史的 UCB 值确定最合适的算法。所选算法生成一个更新后的计划，然后由Verification Agent 重新评估。为确保持续改进，框架纳入了由提供指导的Verification Agent 生成的反馈，这个反馈循环使系统能够逐步优化计划。

---

## 实验与结果（Experiments and Results）

### 实验安排（Experiment Setup）

**Datasets** 为了展示自然规划能力的提升，我们使用了NATURAL PLAN。在改进计划后，我们表明这显著增强了LLMs在两个基准上的推理能力：GPQA和OlympiadBench（仅文本）。此外，我们表明PlanGEN在特定领域数据集DocFinQA上提高了性能。

**Baselines and Our Frameworks** 我们为与我们的框架进行比较开发了两个baselines：(i) Zero-shot CoT 和 (ii) a Vanilla Multi-Agent Baseline。在 Zero-shot COT，作者为框架提供了输入提示，该模型以`<Cot reasoning>`的形式生成输出。对于 Multi-Agent Baseline， 相同的模型在多次迭代中反复调用。系统通过反馈循环反复优化其输出，其中反馈是基于一个旨在提高推理能力的自我反思提示词生成的。我们在所有基准上评估所有提出的框架。对于推理任务，我们采用两阶段方法：(1) 使用我们的框架生成一个优化方案，(2) 执行产生最终答案的计划。

### 主要结果（Main Results）

下图比较了各种单智能体和多智能体baselines（baselines间有所不同——GPQA的一些单智能体基线来自[gpqa-eval](https://klu.ai/glossary/gpqa-eval)的多智能体框架的性能。从结果来看，多智能体框架始终优于baselines。

![Main Results](/images/20250325162737.png)

**Performance on NATURAL PLAN** 对于图 a 可以看出，PlanGen(Best of N)在所有任务中取得了最高的 EM 分数：60.7(Calendar), 43.8(Metting) 和 41.63(Trip)。在所有四个框架都比最强的 Baseline(多智能体) 高出约 10%。对于会议和旅游计划，除了 ToT 之外，所有框架都比最好的 Baseline(Gemini-1.5-Pro) 分别高出约 6% 和约 7%。PlanGEN(Mixture of Algo.) 在会议和旅游计划中达到了第二高的表现，而在日程计划中仍然具有竞争力。这些结果证明了我们的框架在处理多样化的自然语言规划任务以及建立 NATURAL PLAN 三个类别 SOTA的有效性。

**Performance on OlympiadBench** 从图b来看，PlanGEN(Mixture of Algo.) 在数学分类上达到了最高的准确性(55.94%), 表现优于最强的Baseline(Multi-Agent, 50.68%) 大约 5%。值得注意的是，PlanGEN（Mixture of Algo.）在MATH中的卓越性能突显了其在复杂数学推理中的有效性，为MATH达到了SOTA。在PHY类别中，所有多智能体框架都超越了Gemini-1.5-Flash（最强基线），其中PlanGEN（Best of N）实现了最高的准确率（31.78%），在PHY达到了SOTA。

**Performance on GPQA** 从图c可以看出，PlanGEN（Mixture of Algo.）实现了最高的准确率（59.6%）。单个推理时间算法的性能较低，表明选择是有用的。所有提出的框架都大幅优于Gemini-1.5-Pro（46.2%）、GPT模型（∼48%）和Claude-3-Opus（50.4%）。而Claude-3.5-Sonnet和Multi-Agent Baseline与PlanGEN（Mixture of Algo.）相比，表现具有竞争力（∼59%）。

**Performance on DocFinQA** 从图d可以看出，我们的框架在DocFinQA上的性能显著提升，PlanGEN（Best of N）实现了最高的准确率（31.16%）和F1分数（29.45%），为该任务设定了SOTA。所有我们的框架都大幅超越了Gemini-1.5-Pro（最强Baseline）（∼7%）。这些结果突出了多智能体框架在金融文档理解以及在其上进行推理的有效性。

**Performance of our frameworks w.r.t. different complexity** 如图所示，我们对从NATURAL PLAN到日程调度任务的案例进行了研究，以分析不同复杂度级别对不同框架性能的影响。对于日程调度，我们发现PlanGEN（ToT）在简单问题上的表现最佳，而PlanGEN（Best of N）在中级问题上更为有效。随着复杂度的增加，PlanGEN（混合算法）被证明是最有效的方法。我们进一步对附录E中NATURAL PLAN提出的会议和行程规划进行了类似的分析。

![Performance of our frameworks w.r.t. different complexity](/images/20250325204556.png)

**Main Findings** 与单代理系统相比，多代理框架在生成优化规划轨迹方面表现始终如一（图5）。此外，多代理（Baseline）并不总是最强的基准，因为自我校正可能带来挑战，如黄等人在2024年所示。因此，系统内的不同代理需要不同的处理策略，类似于我们的PlanGEN。此外，即使在PlanGEN的多代理框架中，依赖inference-time algorithm对于更复杂的问题来说也证明是不够的（图6）。PlanGEN（Mixture of Algo.）方法在解决复杂规划问题方面具有显著优势，突出了根据实例特定复杂性选择算法的重要性（图1）。鉴于我们的框架是多代理的，我们在下一节进一步讨论了LLM调用次数与它们性能之间的关系。

---

## 参考文献 (References)

- Os-copilot: Towards generalist computer agents with self-improvement.
- AFlow: Automating agentic workflow generation

---

## 备注 (Notes)



---

## 优点与创新点 (Strengths)

1. 提供了一个选择inference-time algorithm的算法，这个算法基于数学理论算法扩展得到，不是简单通过让 LLM 选择。

---

## 局限性与不足 (Limitations)

1. 作者没有提供开源代码，无法复现实验结果。

---

## 我的思考 (Personal Thoughts)

1. 综述与我研究的相关性：能否让智能体自己规划编写项目的计划，然其他智能体进行实现与验证？
2. 是否有可以改进的地方：增加框架 
3. 后续可能的研究方向：将项目规划能力融入项目级代码生成中

---
