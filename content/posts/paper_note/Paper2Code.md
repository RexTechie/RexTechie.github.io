---
title: "Paper2Code"
date: 2025-05-12T13:41:31+08:00
draft: false
description: "一种复现论文代码的方法"
categories: ["📒论文笔记"]
tags: ["代码生成", "Agent"]
---

## 基本信息

- **标题**: Paper2Code: Automating Code Generation from Scientific Papers in Machine Learning
- **作者**: Minju Seo, Jinheon Baek, Seongyun Lee, Sung Ju Hwang等。
- **作者单位**: KAIST, DeepAuto.ai
- **期刊/会议**: ArXiv
- **发表年份**: 2025
- **DOI**: [2504.17192v2](https://arxiv.org/abs/2504.17192v2)
- **开源地址**: [Github](https://github.com/going-doer/Paper2Code)
- **关键词**: Large Language Models (LLMs), Code Generation（Machine Learning）

---

## 研究背景 (Background)

尽管机器学习研究的迅速增长，但相应的代码实施通常无法实现，这使研究人员重现结果并在先前的工作基础上进行劳动密集型。同时，最近的大型语言模型（LLMS）在了解科学文档并生成高质量代码方面表现出色。

---

## 研究问题 (Research Questions)

用agent复现机器学习相关的论文方法与实验

---

## 核心贡献 (Key Contributions)

- 提出从科研论文自动生成代码的框架: PaperCoder
- ​​三阶段工作流程​​：PaperCoder框架将代码生成任务分解为三个结构化的阶段，规划、分析、编码。
- 构建基准数据集（包含90篇来自ICML、NeurIPS和ICLR 2024的论文）
- 在PaperBench基准测试中的优异表现：在新发布的PaperBench基准测试中，PaperCoder表现出色，显著优于其他强基线方法。

---

## 方法（Methods）

作者引入了PaperCoder，是一种新颖的框架，用于实现研究仓库（论文复现出来的代码仓库）的生成。作者将工作过程建模为M(R) = C，其中M是模型，R是论文，C是代码。受软件开发方法的启发，作者采用了一种结构化的方法，该方法反映了良好的经验验证的软件工程原则，即：规划-分析-实现的工作流程。为了实现这一目标，作者将过程分解为三个阶段：1）规划（Planing）. 2）分析（Analyzing）. 3）编码（Coding）。每个阶段都利用多智能体方案。更加规范化的定义是C = M (P) = M_code(R, P, A)，其中R是论文，P是规划，A是分析。每个部分的生成遵循：P = M_plan(R), A = M_analysis(R, P) and C = M_code(R, P, A)。完整的流程如下图所示：

![Paper2Code Overview](/images/20250512142016.png)

### 规划（Planing）

直接将论文作为输入，生成代码是不现实的，因为论文只要是为了记录一些发现和说服读者自己的方法，而不是作为软件开发的结构化输入。因此论文通常包含补充信息，尽管有些核心概念至关重要，但是与实现无关。这些噪音，都会使得代码生成非常的困难。为了解决这个问题，需要**将论文分解为结构化的多个角度的计划**。作者用四个步骤来描述论文的分解计划：总体规划、架构设计、逻辑设计和配置文件生成。用公式表示即：Mplan(R) = P = {o, d, l, g}，o表示总体规划的产物，d表示架构设计的产物，l表示逻辑设计的产物，g表示配置文件的产物。每个输出都是在相应的阶段生成的，此阶段遵循一个顺序结构，每个阶段的输出作为下一个输入。

#### 总体规划（Over Plan）

规划阶段的第一步就是总体规划，涉及概括和组织从大局观（原文使用的是从高级别的角度）的角度看到的实现论文代码所具备的核心要素。提取出来的摘要信息提供了基本的概念框架，可以清楚地指导后续步骤。对应的公式表示是M_plan(R) = o, R表示论文，o表示总体的规划。

#### 架构设计（Architect Design）

第二阶段是通过上一阶段的产物和论文，来架构软件。设计良好的结构是必不可少的，特别是对于软件系统必须无缝交互的软件系统。**这个阶段着重于确定必要的组成部分并定义其关系**，以确保井井有条且符合功能的代码仓库。为了实现这个目标，作者要求创建定义软件体系结构的关键artifact。

- File list: 文件列表是存储库所需的文件的结构化集合，概述了软件的模块化结构。
- Class diagram：类图提供了系统数据结构和接口的静态表示，使用UML类图进行描述。
- Sequence diagram：时序图以动态表示程序的调用流程和对象直接的交互，同样适用UML表示。

这种结构化方法可确保对软件体系结构的清晰有组织的表示。通过构建这些artifact，可以直观地展示论文中描述的基本组成部分，从而使得代码生成方法更加结构化和系统化。这个过程有助于更好地分析依赖关系，从而确保生成的代码仓库与论文的核心思想保持一致。用公式表示：M_plan(R, o) = d，其中r是论文，o是总体规划，d是架构设计的产物。

#### 逻辑设计（Logic Design）

在软件开发过程中，各个文件很少独立发挥作用。取而代之的是，它们通常通过导入包和模块交互。比如，如果在utils.py中定义了一个函数a，然后将函数a导入evaluation.py，为了保持正确的依赖结构，utils.py必须在evaluation.py之前实现。为了解决这些依赖性，这个阶段将论文、总体规划的产物和架构设计的产物一起输入。然后分析每个文件及其组件的逻辑，确定必要的依赖项和实现的最佳执行顺序。最终的输出，它会生成一个**有序的文件列表，详细说明每个文件作用，以及考虑到依赖关系和仓库内依赖项应实现哪些文件**。这个方法可确保存储库的生成不仅考虑单个文件结构，还考虑相互通信，从而促进了组织良好且逻辑上连贯的实现。用公式表示：Mplan(R, o, d) = l，R是论文，o是总体的规划，d是架构设计的产物，l是逻辑设计的产物。

#### 配置文件生成（Configuration File Generation）

最后，配置文件生成的阶段合成了之前所有的产物，生成**包含模型训练所需的超参数和配置文件（config.yaml）**。在这个阶段，用户可以查看和修改config.yaml文件，以识别和纠正丢失或错误指定的详细信息。例如用户可能需要指定huggingface数据集的路径或定义checkpoint存储目录。这个步骤有助于减少生成过程中的幻觉，例如产生不存在的数据集或引用错误的文件路径。用公式表示：M_plan(R, o, d, l) = g，R是论文，o是总体的规划，d是架构设计的产物，l是逻辑设计的产物，g是配置文件的产物。

### 分析（Analyzing）

规划阶段主要关注设计整体代码的架构并概括大致的路线图，分析阶段**主要深入设计每个文件的实现**。在这个阶段，对仓库中每个文件的详细目的和必要的考虑进行了彻底的分析。具体而言，这个阶段的输入包括论文以及先前输出的产物，输出包括文件级别的分析准确实现细节，以后将为代码生成过程提供信息。用公式表示：$(\{0, d, l, g\}, f_{i}) = a_{i}^{n}$，其中R是论文，d是架构设计的产物，l是逻辑设计的产物，g是配置文件的产物，f是文件，a是分析，n是文件的个数。这个过程对每个文件单独应用，以生产一个文件级的实现计划。

### 编码（Coding）

最后的阶段是编码阶段，它会产生构成研究仓库的代码。每个文件的生成都以之前的输出为指导。由于仓库文件直接通常存在导入依赖，因此要求严格遵循规划阶段简历的有序文件列表，确保顺序一致性。用公式表示：$\{ M_{\text{coder}}(R, P = \{0, d, l, g\}, f_i, a_i) = c_i \}_{i = 1}^{n(\text{file})}$，其中R是论文，d是架构设计的产物，l是逻辑设计的产物，g是配置文件的产物，f是文件，a是分析，n是文件的个数，c是文件级代码。最初生成的代码可能需要调试或完善，以确保正确性和完整功能。*在这项工作中，综合调试策略和详细的错误纠正工作流程超出了本文的范围。*

## 实验环境（Experimental Setup）

### 数据（Data）

- Paper2Code Benchmark: 数据源于ICML 2024、NeuraIPS 2024 and ICLR 2024。通过OpenReview API过滤出具有开源代码的论文。在其中选择出总代码库少于70000个标记的仓库。为了确保质量，作者评估了基于模型的仓库，并在每个会议选择了得分最高的30篇论文。最终有90篇论文作为实验的基准。对于人类评估，我们选择评估参与者撰写的13篇论文(大多来自于KAIST，不全是顶会)。
- PaperBench Code-Dev: 作者进一步使用PaperBench Code-Dev基准验证我们的框架，该基准包含来自ICML 2024的20篇精选论文集。这个数据集来源于OpenAI 2025.04.02发表的PaperBench论文

### 基线和我们的方法（Baselines and Our Model）

- Paper2Code Benchmark: 由于没有直接的方法可以实现端到端将论文转为代码，因此作者将几种相关的方法作为基准，尤其是那些用自然语言输入的软件开发设计的方法。
  - ChatDev
  - MetaGPT
  - Abstract: 一个简单基准，其中只向语言模型提供论文的摘要，要求它根据最少的信息实现代码库。
  - Paper: 另一个简单的基准，其中将完整论文作为输入，并提示模型生成相应的代码仓库。
  - PaperCoders（作者的方法）
  - Oracle: 该论文作者发布的官方代码仓库。
- PaperBench Code-Dev: 作者将PaperCoders的方法与PaperBench论文中提到的两个智能体进行比较。
  - Basic Agent：基本智能体遵循ReAct风格的方案，并配备了一套预定义的工具，包括bash shell命令执行器、Python代码运行器、网页浏览器以及用于处理长文档的分页文件阅读器。
  - Iterative Agent：迭代智能体通过利用全部可用计算时间并融入鼓励通过子目标分解逐步推理的定制提示策略，扩展了基本智能体

### 评估指标（Evaluation Setup）

#### Paper2Code Benchmark

评估代码生成通常依赖于单元测试或验证特定的实现组件。然而，由于缺乏黄金标准库，因此这类评估不可行。许多论文不发布官方代码，即使发布了，测试脚本也常常不可用。此外，手动标注每篇论文的ground-truth实现细节非常耗时，使得大模型评估不切实际。

- Model-Based Evaluation 为了解决这种挑战，作者提出一种基于模型的评估方法，该方法在上使用或不使用黄金标准库的情况下评估生成仓库的质量。考虑两个变量：1）reference-based，同时使用论文和作者提供的源码。2）reference-free，仅使用论文评估生成的仓库。对于这两种变体，使用语言模型对所需的实现组件进行评价并评估其正确性。它将严重程度级别（高、中、低）分配给任何缺失或存在缺陷的元素，并在1到5的范围上生成一个整体正确性的粉。为确保稳定性，作者使用n-way采样报告多次生成的平均得分。在这个实验中，作者设置n=8并使用o3-mini-high作为一个评估模型。
- Reference-Based Evaluation 当存在官方存储库时，将本文中描述的方法视为可能实现的方案。虽然可能存在多个有效的实现，但作者发布的版本视为最准确的版本。在这个实验设置中，作者提供论文和黄金标准仓库作为输入，并且让评估模型来识别和评价所需的组件。然后，模型将预测的库与这些组件进行比较，并从1～5分配一个正确性分数，反映组件覆盖率和任何错误的严重程度。
- Reference-Free Evaluation 在许多情况下，作者发布的版本不可用。为了处理此类情况,作者提出一种只根据论文，评价生成的存储库的策略。该模型通过提示词用评估模型评价生成的各个组件，并评估它们是否在生成的仓库中得到充分实现。然后根据这些组件的质量，分配一个1～5的正确性分数。
- Human Evaluation 尽管有了模型评估的方式，还是需要人类评估的方法。由于任务复杂，需要理解论文并判断方案的可行性，作者招募了计算机科学专业的MS和PhD学生，要求有至少撰写一篇同行评审论文的经验。评估过程如下
  1. 首先，标注着根据论文定义关键事实标准，涵盖数据处理、方法和评估部分。
  2. 他们随后对生成的代码库进行审查和排名，分为以下三个比较组: Group1: Model Variants of Our Method 作者的系统使用不同的主干模型（例如，o3-mini vs 其他三个开源方式）； Group2: Navie Baselines. 仅使用论文或摘要作为输入生成的仓库； Group3: Related Works. 由MetaGPT和ChatDev等现有软件开发框架生成的仓库。在每一组中，标注员分配一个相对排名。排名被转换为1-5分的评分。在有三名候选人的小组中，排名第一的仓库获得5分，第二获得3分，第三获得1分。在完成所有组级别的排名之后，注释者被要求选择总体上最好的存储库，并提供一个简短的理由。他们还被问及，排名第一的仓库是否会比从头开始更容易重现论文的方法和实验。如果不是，他们需要解释原因。
  3. 最后，标注者重新访问作者的方法生成的仓库，评估之前定义的每个实现标准是否完全（o）、部分（△）或未满足（×）。

#### PaperBench Code-Dev Evaluation

为了评估作者的系统在PaperBench Code-Dev基准测试上的表现，作者采用了PaperBench官方的评估协议，该协议测量了在精心挑选的ICML 2024论文集中的复制准确性。具体来说，我们遵循原始论文作者撰写的评分标准，该标准定义了一套层次化的实现要求。评估模型根据提交的代码是否正确实现了任何指定要求来评估每个生成的代码库。重要的是，评估仅关注代码开发节点，即候选代码库是否包含某些要求的正确实现。

## 实验结果与分析 (Experimental Results & Analysis)

### Main Results

![Main Results](/images/20250512192046.png)

### Correlation between Reference-based and Reference-free

![Correlation between Reference-based and Reference-free](/images/20250512192358.png)

作者希望通过这个实验来验证**是否可以使用reference-free的评估方法来替代reference-based**。实验方法即通过对生成的代码仓库分别使用reference-free和reference-based评分方式进行评分，然后看这两种评分直接是否相关（即是否一致）。

- r（皮尔逊相关系数）：衡量两个变量之间的线性相关程度。范围（-1～+1）
- p（显著性水平）：p值表示观察到当前数据的结果在“零假设”（即认为两者无相关性）成立的情况下出现的概率。范围（0～1）

### PaperBench Code-Dev Results

![PaperBench Code-Dev Results](/images/20250512192417.png)

作者想要**与OpenAI提出的PaperBench进行比较**，分别比较了其论文中提到的BasicAgent和IterativeAgent

### Human Evaluation Results

![Human Evaluation Results](/images/20250512193110.png)

为了评估生成代码的实际效用和主观质量，作者进行了全面的**人工评估**。

### Detailed Analysis on Generated Repository

作者进行**更精细的人类评估**。具体就是让作者自己从baseline和消融实验变体的仓库中选择一个比较好的一个。77%（13人位作者中的10位）选择了由作者方法生成的仓库，10人里有3人选择了消融变体生成的代码。

![Qualitative analysis of top-ranked repositories](/images/20250513103530.png)

为了进一步评估实用性，询问参与者从头开始相比，排名最高的仓库是否会使复现论文的方法和实验更简单。85%表示了认可。
对于每个论文的每个部分（Data Processing, Method, Evaluation），要求作者评估是否完全实现了每个标准。结果如下所示：

![Detailed Analysis on Generated Repository](/images/20250512192643.png)

### Analysis on Human Alignment for Evaluation

![Analysis on Human Alignment for Evaluation](/images/20250512192721.png)

这个实验是为了评估模型评估的可信度，上表报告了在reference-based和reference-free的配置下，**人类评分与模型评分之间的秩相关**。由实验结果表明，人类评分与模型评分的相关性是较高的，因此基于模型的评估方式可以作为参考。

### Analysis on Different LLMs

![Analysis on Different LLMs](/images/20250512192437.png)

作者分别应用DeepSeek-Coder-V2-Lite-Instruct、Qwen2.5-Coder-7B-Instruct、DeepSeek-R1-Distill Qwen-14B、o3-mini-high在框架中。作者的实验结果中，o3-mini-high的模型性能最高。

### Ablation Studies

![Ablation Studies](/images/20250512193418.png)

### Analysis on Executability

![Analysis on Executability](/images/20250512193457.png)

为了验证生成的代码不仅结构合理，而且能够在最小干预下执行，作者对五篇代表性论文进行了手动调试分析。对于每种情况，作者都尝试执行生成的存储库，并记录为实现成功运行而修改的行数。

## 参考文献 (References)

- PaperBench: Evaluating AI's Ability to Replicate AI Research

## 备注 (Notes)

本文实验内容丰富

---

## 优点与创新点 (Strengths)

- 实验中大量使用了统计学相关的指标来判断模型：皮尔逊系数、显著性水平、秩相关系数
- 三阶段的设计中侧重于规划阶段的设计，规划阶段中包含绘制类图、时序图，使得编码时更加规范
- 规划阶段，确定好生成的文件，并文件的功能，分析阶段具体分析各个文件的详细功能。

---

## 局限性与不足 (Limitations)

- 方法缺乏迭代优化
- 方法中各智能体无法自动实现协作，需要手动一个个执行文件。

---

## 我的思考 (Personal Thoughts)

本文由于实验直接执行困难，只能通过模型评估和人类评估的方式进行实验，因此还需要进行实验验证评估方式的有效性。这种实验方式可作为后续实验的参考。

---
