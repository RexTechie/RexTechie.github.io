---
title: "MetaGPT: Meta Programming for A Multi-Agent Collaborative Framework"
date: 2025-01-07T20:39:47+08:00
draft: false
description: "一种多智能体写作代码生成的方法"
categories: ["📒论文笔记"]
tags: ["代码生成",]
---


## 基本信息

- **标题**: MetaGPT: Meta Programming for A Multi-Agent Collaborative Framework
- **作者**: Sirui Hong、Mingchen Zhuge、Chenglin Wu（通讯作者）等。
- **作者单位**: DeepWisdom、阿卜杜拉国王科技大学、厦门大学、香港中文大学、南京大学、宾夕法尼亚大学、加州大学伯克利分校、瑞士人工智能实验室
- **期刊/会议**: ICLR 2023
- **发表年份**: 2023
- **DOI**: [2308.00352](https://arxiv.org/abs/2308.00352)
- **开源地址**: [Github](https://github.com/geekan/MetaGPT)
- **关键词**: MetaGPT, Multi-Agent Collaboration, Standardized Operating Procedures (SOPs), Large Language Models (LLMs), Code Generation

---

## 研究背景 (Background)

利用大语言模型的Agent为增强和复制人类的工作流程提供了机会。但是实际应用中，现有系统将复杂问题过度简化。很多人想努力实现有效、连贯和准确的解决问题，尤其是需要协作的任务。而SOP可以有效的分解任务并且协调各个任务，明确的SOPs能够提高任务执行的一致性和准确性，确保其与定义的角色和质量标准相符。

---

## 研究问题 (Research Questions)

1. 如何使用应用SOPs与Agent协作开发。
2. 如何优化Agent协作通信能力。
3. 如何提高代码生成的质量。

---

## 核心贡献 (Key Contributions)

总结本文的主要贡献点：

1. 引入了MetaGPT，一个基于LLM的多智能体协作元编程框架。
2. 作者在MetaGPT设计中创新性地集成了SOP，减少了LLM的代理之间的无效协作。此外，还引入了一种新颖的执行反馈机制，可以在运行时调试和执行代码，从而提高了代码生成的质量（MBPP上提高了5.4%）
3. 在HumanEval和MBPP达到了SOAT

---

## MetaGPT 框架 (MetaGPT Framework)

### SOP中的Agent（Agents in Stanndard Operating Procedures）

#### 角色的特定职能

解决复杂的任务或问题通常需要具有不同技能和专业知识的智能体协作，每个智能体都针对特定问题提供专门的输出。如在一家软件公司，产品经理的任务是分析业务、软件工程师负责编程开发。因此MetaGPT定义了五个角色：产品经理（Product Manager）、架构师（Architect）、项目经理（Project Manager）、工程师（Engineer）以及QA工程师（QA Engineer）。如下面图1所示：
![Figure 1](/images/20250108143655.png)
图1：MetaGPT与真实世界人类团队之间的软件开发SOPs。

在MetaGPT中，对于每个角色制定了Agent的名字（name）、资料（profile）、目标（goal）和约束（constraints）。同时也初始化每个角色的特定的上下文（context）和技能（skill）。例如，产品经理可以使用网络搜索技能，而工程师可以执行代码。如下图2所示。所有Agent遵循[ReAct](https://arxiv.org/pdf/2210.03629)风格。

![Figure 2](/images/20250108143614.png)
图2：一个展示MetaGPT软件开发生成过程的图表，强调其对SOPs的显著依赖。

每个Agent监控环境（MetaGPT中的消息池）以发现重要的观察结果（如其他agent的消息），这些消息可以直接触发Agent的行为，也可以协助完成任务。

#### Agent之间的工作流程（Workflow across Agents）

通过定义代理的角色和技能，可以建立起基本的工作流。在MetaGPT的工作中，遵循软件开发中的SOP，使得所有工作都能按顺序执行。

具体来说，如之前图1所示，产品经理在获取用户需求后，进行全面的需求分析，制定PRD（Product requirement document），里面包含用户故事和需求池。之后将PRD传递给架构师，他会将需求转为系统设计，例如文件列表、数据结构以及接口定义等。一旦在系统设计中捕获，信息就会直接发送给项目经理以进行任务分配。工程师则继续执行指定的类和功能（如图2所示）。在之后的阶段，QA工程师制定测试用例确保代码质量。最后MetaGPT生成高质量的软件解决方案。详细的SOP工作流程如下图3所示。

![Figure 3](/images/20250108143517.png)
图3：MetaGPT软件开发生成过程。

### 通信协议（Communication Protocol）

#### 结构化通信接口（Structed Communication Interfaces）

如今大部分基于大语言模型的多智能体框架（[CAMEL](https://proceedings.neurips.cc/paper_files/paper/2023/file/a3621ee907def47c1b952ade25c67698-Paper-Conference.pdf)、[NLSOM](https://arxiv.org/pdf/2305.17066)、[CoELA](https://arxiv.org/pdf/2307.02485)、[Generative Agents](https://dl.acm.org/doi/pdf/10.1145/3586183.3606763)）都是使用无约束的自然语言作为通信接口。但是如果只用自然语言作为通信接口，会出现问题。如在[电话游戏](https://en.wikipedia.org/wiki/Telephone_game)中，在几轮传递之后，原始的信息会发生扭曲。当用纯自然语言进行通信时，也会出现这种问题。因此需要每个角色制定一个结构化的通信接口，以确保信息的准确传递。如之前的图3所示，架构师Agent产生两个输出：系统接口设计和顺序流程图。与[ChatDev](https://arxiv.org/pdf/2303.08268)不同，MetaGPT中的Agent通过文档和图输出，而不是对话。这些文档也都是包含必要信息的文档，防止出现不相关或丢失的内容。

#### 发布订阅机制（Publish-Subscribe Mechanism）

共享信息对于协作非常关键。比如：建筑师和工程师需要参考PRD。然而，如果每次都是以一对一的方式传达这些信息，可能会使通信变得复杂，效率低下。为了解决这个问题，一个可行的方案是将信息存储在一个**全局信息池**中。如图2左边所示，metagpt引入了共享消息池，允许所有agent直接访问和发布信息。这样就可以使得任何Agent可以直接从共享池中检索所需的信息，而不需要通过其他Agent传递。这样提高了通信效率。

如果与每个Agent共享所有信息可能会导致信息过载。在执行任务期间，Agent通常喜欢接受与任务相关的信息，要避免因不相关的细节而分散注意力。如图2左边所示，MetaGPT引入了订阅机制，Agent不依赖对话，而是利用角色特定的兴趣来提取相关的信息。实际实现的时候，一个Agent只会在接收到所有其先决依赖项后才会激活操作。如之前图3所示，架构师主要关注产品经理提供的PRD，而QA工程师等角色的文档可能不太关心。

### 迭代编程与执行反馈（Iterative programming with execution feedback）

在现实开发中，调试和优化非常重要。然而现有的方法通常缺少自我纠错的机制（self-correction mechanism），从而导致代码生成失败。之前也有一些人引入了不可执行代码审查（non-executable code review）和自我反思（self-reflection）:[ChatDev](https://arxiv.org/pdf/2303.08268)、[ReAct](https://arxiv.org/pdf/2210.03629)。然而它们在确保代码可执行性和运行时正确性上还是面临很大的挑战。

MetaGPT引入了一种可执行的反馈机制(executable feedback)来迭代改进代码。更具体地说，如之前图2所示，要求工程师根据产品需求和设计编写代码。这使得工程师能够使用自己的历史执行记录和调试记忆来持续的改进代码。为了获得额外的信息，工程师编写并执行相应的单元测试用例，然后接收测试结果。如果测试结果满足条件，则启动其他的开发任务，否则工程师将进行调试。这种迭代过程将会持续进行，直到测试通过或达到最大迭代次数（3次）。

---

## 实验结果 (Results)

### 实验环境（Experimental Setup）

#### 数据集（Dataset）

作者使用了两个公开的benchmarks（HumanEval、MBPP）以及一个自己设计的软件开发的benchmark

- [HumanEval](../human_eval): 出自openai，有164个手写编程任务，包括提示词（prompt）、标准答案（canonical_solution）、单元测试代码（test）、函数名称（entry_point）。
- [MBPP](https://arxiv.org/pdf/2108.07732v1): 出自google search，有427个Python任务，这些任务涵盖核心概念和标准库代码，并包括要求（text）、代码（code）、单元测试集（test_list）等信息。
- SoftwareDev：作者自己设计的Benchmark，包含70个软件开发任务的代表性示例，每个示例都有自己的任务提示。如下表所示。这些任务有游戏、算法、数据可视化等等。Software专注于工程方面。
![Table 8](/images/20250108191714.png)

#### 评估指标（Evaluation Metrics）

- pass@k: 对于HumanEval和MBPP，作者使用了[openai](../human_eval)提出的无偏估计的pass@k指标，$\text{pass@}k := \mathbb{E}_{\text{Problems}} \left[ 1 - \frac{\binom{n-c}{k}}{\binom{n}{k}} \right]$。
- 对于SoftwareDev，优先考虑实际使用，并通过人工评估（A、E）或统计分析（B、C、D）来评估性能。
  - A：可执行性，该指标对代码进行评分，从1（失败/无法运行）～4（完美）。1-非功能性、2-可运行但不完美、3-可运行但不完美、4-完美。
  - B：成本，包括运行时间、token使用量以及费用。
  - C：代码统计，包括代码文件数、每个文件代码行数以及总代码数
  - D：生产率，定义为token使用量/代码行数，即每行代码消耗的令牌。
  - E：人工修订成本，指手动修正的次数，解决诸如包导入错误、类名不正确或不完整的引用路径等问题。通常，每次修正涉及最多3行代码。

#### 基线（Baselines）

对于HumanEval和MBPP，作者使用了一些基线模型：

- [AlphaCode](https://www.science.org/doi/10.1126/science.abq1158)
- [InCoder](https://arxiv.org/pdf/2204.05999)
- [CodeGeeX](https://dl.acm.org/doi/pdf/10.1145/3580305.3599790)
- [CodeGen](https://arxiv.org/pdf/2203.13474)
- [CodeX](../human_eval)
- [CodeT](https://arxiv.org/pdf/2207.10397)
- [PaLM](https://www.jmlr.org/papers/volume24/22-1144/22-1144.pdf)
- [GPT-4](https://arxiv.org/pdf/2303.08774)

其中Dong等人已经提供了一些结果（如InCoder、CodeGeeX）。在HumanEval和MBPP中，作者稍微修改了下提示词，以适应MetaGPT的输入格式。

对于SoftwareDev上的基线

- [AutoGPT](https://github.com/Significant-Gravitas/AutoGPT)
- [LangChain](https://github.com/langchain-ai/langchain) with Python Read-Eval-Print Loop(REPL) [tool](https://en.wikipedia.org/wiki/Read–eval–print_loop)
- [AgentVerse](https://cz5waila03cyo0tux1owpyofgoryroob.itic-sci.com/9C/54/B4/9C54B47616795A320E36FCB1EA595C91.pdf)
- [ChatDev](https://arxiv.org/pdf/2303.08268)

### 主要结果（Main Result）

![Figure 4](/images/20250108195807.png)
上图表明MetaGPT在HumanEval和MBPP基准测试中达到了SOAT。

![Table 1](/images/20250108201132.png)
MetaGPT在SoftwareDev上的实验结果几乎均好于ChatDev。这些结果均凸显出SOP在多个Agent直接协作中的好处。

![Figure 5](/images/20250108201912.png)
在上图中，通过可视化示例展示了MetaGPT在SoftwareDev上的生成结果。

### 能力分析（Capability Analysis）

![Table 2](/images/20250108202212.png)
如上表所示，MetaGPT包含多种能力，可以有效地处理复杂且专业的开发业务，结合SOPs可以显著改进代码生成。

### 消融实验（Ablation Study）

#### 角色的有效性

![Table 3](/images/20250108202228.png)
为了了解不同角色对最终结果的影响，作者对角色部分做了消融实验。结果如上表所示，虽然更多的角色会稍微增加费用，但整体性能显著提高，展示了各种角色的有效性。

#### 可执行的反馈机制的有效性

在一开始的[主要结果](#主要结果main-result)展示的图中，见可执行反馈添加到MetaGPT，pass@1显著提高。此外，表1中显示出反馈机制提高了可行性并降低了人工修改成本。这些结果说明了反馈机制如何能够生成更高质量的代码。

---

## 参考文献 (References)

- Austin J, Odena A, Nye M, et al. Program synthesis with large language models[J]. arXiv preprint arXiv:2108.07732, 2021.
- Yao S, Zhao J, Yu D, et al. React: Synergizing reasoning and acting in language models[J]. arXiv preprint arXiv:2210.03629, 2022.
- Zhao X, Li M, Weber C, et al. Chat with the environment: Interactive multimodal perception using large language models[C]//2023 IEEE/RSJ International Conference on Intelligent Robots and Systems (IROS). IEEE, 2023: 3590-3596.

## 备注 (Notes)

这篇论文本身对方法的描述不足够详细，在实现Agent上有很多小技巧，需要读源码才能体会到

---

## 优点与创新点 (Strengths)

1. 将SOPs与Agent结合，更加模仿了真实世界的软件开发过程。
2. 设计了共享消息池和订阅机制，提高了通信效率。
3. 引入了可执行的反馈机制，提高了代码生成的质量。

---

## 局限性与不足 (Limitations)

1. 实际使用后，项目的可执行性还是有待提高。
2. SoftwareDev的实验评价指标略为主观

---

## 我的思考 (Personal Thoughts)

1. 本文与我研究的相关性：MetaGPT可以实现一些复杂任务的开发，在全栈项目开发中有共同之处。对于SOPs的思想、共享消息池和订阅机制以及可执行的反馈机制，可以考虑在项目中使用。
2. 是否有可以改进的地方：评价指标还有待改进。
3. 后续可能的研究方向：MetaGPT实现的任务较为广泛，考虑缩小范围，只生成Web开发相关的代码。

---
