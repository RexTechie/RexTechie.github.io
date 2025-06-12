---
title: "DocAgent: A Multi-Agent System for Automated Code Documentation Generation"
date: 2025-06-03T13:11:17+08:00
description: "代码文档生成"
categories: ["📒论文笔记"]
tags: ["Agent"]
---


## 基本信息

- **标题**: DocAgent: A Multi-Agent System for Automated Code Documentation Generation
- **作者**: Dayu Yang（通讯作者）、Antoine Simoulin、Xin Qian等。
- **作者单位**: Meta AI
- **期刊/会议**: ArXiv
- **发表年份**: 2025.04.11
- **DOI**: [2504.08725](https://arxiv.org/abs/2504.08725)
- **开源地址**: [Github](https://github.com/facebookresearch/DocAgent)
- **关键词**: Code Documentation, Multi-Agent System, Large Language Models

---

## 研究问题 (Research Questions)

如何自动生成代码文档，尤其是处理复杂的依赖关系和上下文信息？

---

## 研究背景 (Background)

---

像现在的方法FIM（Fill in the Middle）和 chat agent可以实现自动化生成代码文档，但还是有一些局限性：

1. 省略了必要的信息（例如，参数或返回值描述）。
2. 它们通常提供最少量的上下文或理由，限制了生成文档的有用性。
3. 由于LLM的幻觉，会虚构不存在的组件，尤其是规模比较大的项目中。

---

## 核心贡献 (Key Contributions)

![DocAgent](/images/20250612102640.png)

- DocAgent，一个多智能体、拓扑结构化的上下文感知文档生成系统。
- 一个衡量代码文档完整性、实用性和事实一致性的稳健评估框架。
- 全面实验表明，DocAgent在多个数据集中持续优于最先进的基线。

---

## 方法 (Methodology)

DocAgent分两个阶段来处理复杂的依赖关系并确保上下文的相关性。

1. 用一个Navigator确定一个最佳的依赖感知的处理顺序
2. Multi-Agent System系统增量地生成文档，利用专门的Agent进行代码分析、信息检索、编写文档和验证

![Architecture of DocAgent](/images/20250609093041.png)

### Navigator: 感知依赖顺序

生成代码级别的文档需要理解依赖代码之间的关系，然而如果给出所有内容，很容易超出上下文窗口的限制。Navigator模块就是为了解决这个问题，Navigator建立了一个处理顺序，确保处理依赖关系之后才对组件进行文档化，从而实现增量上下文构建。

#### 构建依赖图

首先对整个存储库进行静态分析，解析源文件的抽象语法树（AST，Abstract Syntax Tree），以识别代码组件（函数、方法、类）及其相互关系（函数调用、继承、属性方法和模块导入），用一个有向无环图（DAG， Directed Acyclic Graph）来表示，节点是组件，边是依赖关系。为了实现拓扑排序，用Tarjan算法检测图中的循环，并将其浓缩单个超级节点。

#### Topological Traversal for Hierarchical Generation

使用有向无环图，Navigator模块使用DAG执行拓扑排序，以确定文档生成顺序。遍历遵循“依赖优先”原则：只有在所有直接依赖的组件都已被文档化后，才会处理组件。这种方式确保多智能体系统为特定组件生成文档时，已经生成该组件的所有依赖性描述。因此，每个代码文档只需要其单跳依赖项的信息，消除了需要拉入不断增长的背景信息链的需求。

### Multi-Agent Documentation Generation

多智能体系统由一个编排器协调四个专业智能体（Reader、Searcher、Writer、Verifier），为每个组件生成文档。

#### Reader

对于ReaderAgent来说，他的任务是确定对生成全面的且对代码文档有帮助的信息。他通过评估组件的复杂性、可见性和实现细节来进行判断，1）是否需要外部信息，简单的、自包含的组件可能不需要外部信息。2）需要什么信息：这涉及到识别特定的内部依赖（用到的函数、类）、使用的上下文（组件调用的地方，以及其目标）或隐式/显式的外部概念（算法、库、领域知识）。

这个Agent最终输出的是一个格式化的XML，1）关于相关代码组件的内部信息、2）针对特定算法或技术的外部知识。

对于内部信息，需要包含依赖和引用。依赖意味着目标组件调用里其他组件，Reader Agent会判断这些依赖是否是必须的信息。引用意味着目标组件在代码库的某个地方被调用，揭示了目标组件的目的。

外部请求针对的信息并非直接来自代码库本身或可从代码库推断，例如特定领域的知识或第三方库功能。

#### Searcher

Searcher Agent 负责使用工具满足Reader Agent的信息请求。

- 内部代码分析工具（Internal Code Analysis Tool）: 该工具利用静态分析能力遍历代码库，可获取指定内部组件的源代码及现有文档，定位目标组件的调用点，通过预计算依赖图或实时分析追踪依赖关系，并提取相关结构信息（如类继承层次、方法签名等）。
- 外部知识检索工具（External Knowledge Retrieval Tool）: 这个工具通过通用检索API介入外部知识源，能根据Reader对外部概念查询需求自动构建检索语句，并对返回结果进行处理以提取相关解释、定义或描述。

Searcher会将检索到的内部代码信息和外部知识合并为结构化格式，作为后续智能体运作的上下文依据。

就像两个人类合作完成项目并相互交流一样，在Searcher将检索到的信息发送回读者后，Reader会读取更新后的上下文和目标组件，并判断当前上下文是否足以生成文档。若Reader认为检索到的信息仍不充分，Reader可继续向Searcher发送信息请求。如此，信息请求与新检索到的内容将在Reader与Searcher之间循环传递，直至获取足够的信息为止。

#### Writer

Writer智能体的输入是目标组件的代码和由Searcher产出的结构化的内容，核心任务是生成代码文档。生成过程主要通过提示词根据组件类型指定所需结构和内容：

- 函数/方法（Function / Method）：通常需要摘要、额外描述、参数描述、返回值描述、引发的异常以及可能使用的示例（特别是对于公共组件）。
- 类：通常需要摘要、额外描述、初始化示例、构造器参数描述、公共参数描述

这个Writer Agent根据代码和提供的上下文生成符合要求的代码文档草稿。

#### Verifier

Verifier的输入是Writer提供的上下文、代码组件和生成的代码文档，根据预定义的质量标准评估代码文档：信息价值、详细程度和完整性。评估后，Verifier要么批准文档，要么通过结构化反馈提供具体的改进建议。

如果问题可以不通过额外上下文信息的情况下解决，Verifier可以和Writer进行交互。比如：格式问题，可以通过要求Writer重写来轻松解决。

如果问题是因为缺乏信息引起的，并且需要额外的内容，verifier也可以向Reader提供建议，并且将通过Reader-Searcher循环收集额外的信息。

#### Orchestrator

用Orchestrator管理agent工作流，来实现整体的迭代过程。整个周期：

1. Reader分析目标代码以及必要的上下文
2. Searcher收集信息
3. Writer写文档
4. Verifier评估文档质量，批准或退回以供修订
  
这个过程一直持续到生成令人满意的代码文档或达到最大迭代限制。

Adaptive Context Manage(自适应上下文管理)：为了处理Searcher检索到的很大的文本，尤其是复杂的组件，Orchestrator实现了一个自适应上下文截断机制。他监控提供给Writer的token总数，如果context超过了一个可配置的阈值（基于底层LLM的限制），Orchestrator将使用截断策略。它识别结构化后的上下文中最大部分（如：外部知识片段、特定的依赖细节），并从这些部分的末尾选择性地删除内容，以减少token数同时保持整体结构。

---

## 评估框架（Evaluation Framework）

存在的挑战：

1. 缺乏gold reference。传统的指标BLEU或ROUGE，不能适用。
2. 根据文档长度这样的方法不足以作为实际效用的指标
3. 人工评估具有主观性、成本高、难以拓展，大规模试验或持续集成场景不切实际

为了解决这些问题，作者提出了一个新的评估框架，包含几个方面：完整性（Completeness）、帮助性（Helpfulness）、真实性（Truthfulness）。

### 完整性（Completeness）

完整性评估依据标准结构的约定，并包括预期用于给定代码元素（例如，函数、类）的基本组件。高质量的代码文档不仅包括摘要，还包括参数描述、返回值、引发的异常以及可能的使用示例，这些信息依赖于目标组件的声明、主体和可见性。评估方法

主要评估方法，采用抽象语法树（AST）和正则表达式的自动化检查器。

- AST Parsing：识别代码组件（类、函数、方法）并提取其生成的文档。
- Code Analysis：分析代码声明和代码主体（例如：参数、返回语句、抛出异常是否存在）以及可见性（公共/私有），来确定文档部分。如没有参数的函数不需要“参数”部分，而公共类可能比私有函数更有用。
- Section Identification：检测文档字符串中标准部分（例如，摘要、描述、参数、返回值、抛出异常、示例、类属性）的存在，使用预定义模式和结构提示。
- Scoring：计算每个文档字符串的完整性得分，作为所需部分存在的比例。0.0～1.0之间。

这种确定性方法提供了一个结构遵循的客观度量，表明文档是否符合基本形式要求。

### 帮助性（Helpfulness）

帮助是评估文档内容内容的语义质量和实用价值，一个有帮助性的文档不仅限于重申代码元素；它阐明了代码的目的、使用上下文、设计理由和潜在限制。

- Clarity and Conciseness: 摘要信息是否简洁
- Descriptive Depth：扩展描述是否提供了足够的信息，解释了代码背后的“为什么”，或者提到了相关场景或边缘情况？
- Parameter/Attribute Utilty：输入和属性的描述是否有意义，是否指定了预期的类型、值范围或约束，而不是仅仅重复名称？
- Guidance：文档是否有效地指导开发者何时以及如何使用该组件？

评估方式，使用LLM进行评估，作者提出了一套框架。Component Specific Evaluation：作者将评估分解为分别评估文档的不同部分（摘要、主要描述、参数描述）使用针对每个部分的提示词。Structured Prompt Engineering:

1. 明确评分标准（Explicit Scoring Rubrics）：详细标准用5分制Likert特量表（1=Poor，5=Excellent）
2. 示例说明（Illustrative Examples）：具体对应不同等级的文档片段示例，确立评价标准
3. 分步说明（Step-by-Step Instructions）：引导LLM分析代码，将文档与评分标准进行比较，考虑代码的上下文，并为其评分提供理由。
4. 标准的输出格式（Standardized Output Format）：要求大模型提供结构化输出，包括详细的推理、具体的改进意见（如适用）以及最终的数值评分。

这种结构化方法允许对语义质量进行可扩展的评估，超越了表面层次的检查，以衡量文档对开发者的实际价值。

### 真实性（Truthfulness）

由于LLM存在幻觉，因此可能会导致生成文档时引用不存在的函数、参数或类，或者错误地表示组件之间的关系，因此有必要评估文档的真实性。作者通过验证生成的文档中提到的实体是否确实存在与目标的代码库中以及引用是否正确来评估真实性。主要过程包含三部分：

- 代码实体提取（Code Entity Extraction）：大模型用提示词从代码仓库中提取特定代码组件（类、函数、方法、属性）。提示词特别的引导大模型区分关键字、内置类型和常见外部库组件，重点关注内部引用。
- 真实构造（Ground Truth Construction）：作者利用Navigator构建的依赖图，将其用作基准事实，包含所有代码组件及其在存储库中的规范表示。
- 验证（Verification）：每个提取的实体提及都与依赖图进行交叉引用。

作者使用Existence Ratio来量化Truthfulness，即文档中提及的实体与代码库中的实际实体的比例。${Existence Ratio} = \frac{Verified Entities}{Extracted Entities}$

高比率表明文档在实际代码结构中很好地建立了基础，从而最大限度地减少了幻觉引用的风险。

## 实验 (Experimental)

### 基线（Baselines）

- FIM（Fill-in-the-middle）
  - 用途模拟：模拟内联代码补全工具，通过补全“中间空白”来预测注释内容。
  - 使用模型：使用的是 CodeLlama-13B（Roziere 等人，2023），这是一个开源模型，专门针对 FIM 任务进行训练（Bavarian 等人，2022）。
  - 简称：FIM-CL
- Chat
  - 用途模拟：将代码片段直接输入到聊天模型中，请其生成文档注释，代表了当前流行的基于对话的使用方式。
  - 测试模型：
    - GPT-4o mini：简称 Chat-GPT
    - CodeLlama-34B-instruct：简称 Chat-CL

### 实验设置（Experimental Setup）

Data: 选择了一组具有不同规模、复杂度和领域的Python代码库的代表性子集，以确保多样性。数据集包括具有不同依赖密度的模块、函数、方法和类。

System: 作者评估了 DocAgent 的两个变体，仅在所用底层大语言模型（LLM）上有所不同

- DA-GPT：基于 GPT-4o mini 的 DocAgent 版本
- DA-CL：基于 CodeLlama-34B-instruct 的 DocAgent 版本

Statistical: 所有关于统计显著性的结论均基于配对 t 检验（paired t-tests），
显著性阈值设为：p < 0.05

### 实验结果（Experimental Results）

作者主要对Completeness、Helpfulness和Truthfulness三个方面进行了评估。

![试验（Experiment）](/images/20250612102510.png)

#### 完整性试验

![完整性（Completeness）](/images/20250611161423.png)

从结果来看DocAgent的方法完整性显著优于Chat和FIM。说明DocAgent能够生成更全面的代码文档，包含更多必要的部分。

#### 帮助性试验

![帮助性（Helpfulness）](/images/20250611161836.png)

如表所示，DocAgent实现了最高的整体有用性得分，显著优于相应的Chat和FIM，展示了其通过利用检索到的上下文生成更清晰、更有信息量的内容的能力。即使生成有用的参数描述是非常困难的，但是在这里，DocAgent依旧是最高分，这表明其结构化方法有助于这项困难任务，尽管仍有改进空间。

#### 真实性试验

![真实性（Truthfulness）](/images/20250611162251.png)

上表的结果展示了DocAgent生成的文档在事实准确性方面的优越性。DocAgent实现了最高的存在比率，表明其大部分对内部代码组件的引用都是正确的。

#### 消融试验 - Helpfulness

DA-Rand表示Navigator部分不进行拓扑排序，随机处理代码组件。

![消融试验 - Helpfulness](/images/20250611163517.png)

由试验表明，经过拓扑排序的DocAgent在帮助性方面表现更好，表明依赖感知的处理顺序对生成有用的代码文档至关重要。

#### 消融试验 - Truthfulness

![消融试验 - Truthfulness](/images/20250611164426.png)

由试验表明，虽然没有Navigator，Searcher依然能检索依赖的组件。然而没有遵循“依赖优先”原则，这些组件不太可能有可用于上下文的现有文档。

总体而言，消融结果证实，导航器的依赖感知拓扑排序是DocAgent的关键组成部分，通过实现有效的增量上下文管理，显著提高了生成文档的有用性和事实准确性。

---

## 参考文献 (References)

### LLM Agent

- Guang Yang, Yu Zhou, Wei Cheng, Xiangyu Zhang, Xiang Chen, Terry Yue Zhuo, Ke Liu, Xin Zhou, David Lo, and Taolue Chen. 2024. Less is more: Docstring compression in code generation. arXiv preprint arXiv:2410.22793.
- Shinn Yao, Jeffrey Zhao, Dian Yu, Kang Chen, Karthik Narasimhan, and Yuan Cao. 2022. React: Synergizing reasoning and acting in language models. arXiv preprint arXiv:2210.03629.
- Noah Shinn, Margaret Labash, and Stefano Ermon. 2023. Reflexion: Language agents with verbal reinforcement learning. arXiv preprint arXiv:2303.11366.
- Xiang Li, Qinyuan Zhu, Yelong Cheng, Weizhu Xu, and Xi Liu. 2023b. Camel: Communicative agents for “mind” exploration. arXiv preprint arXiv:2303.17760.
- Ziniu Wu, Cheng Liu, Jindong Zhang, Xinyun Li, Yewen Wang, Jimmy Xin, Lianmin Zhang, Eric Xing, Yuxin Lu, and Percy Liang. 2023. Autogen: Enabling next-generation multi-agent communication with language models. arXiv preprint arXiv:2309.07864.
- Xiaoqing Zhang, Zhirui Wang, Lichao Yang, Wei Zhang, and Yong Zhang. 2023b. Mapcoder: Map-reducestyle code generation with multi-agent collaboration. arXiv preprint arXiv:2307.15808.
- Yuzhang Qian, Zian Zhang, Liang Pan, Peng Wang, Shouyi Liu, Wayne Xin Zhao, and Ji-Rong Wen. 2023. Chatdev: Revolutionizing software development with ai-collaborative agents. arXiv preprint arXiv:2307.07924.

### Code Summarization

- Zhangyin Feng, Daya Guo, Duyu Tang, Nan Duan, Xiaocheng Feng, Ming Gong, Linjun Shou, Bing Qin, Ting Liu, and Daxin Jiang. 2020. Codebert: A pre-trained model for programming and natural languages. In EMNLP.
- Daya Guo, Shuo Ren, Shuai Lu, Zhangyin Feng, Duyu Tang, Nan Duan, Ming Zhou, and Daxin Jiang. 2021. Graphcodebert: Pre-training code representations with data flow. In ICLR.
- Wasi U Ahmad, Saikat Chakraborty, Baishakhi Ray, and Kai-Wei Chang. 2021. Unified pre-training for program understanding and generation. In ACL.
- Yue Wang, Shuo Ren, Daya Lu, Duyu Tang, Nan Duan, Ming Zhou, and Daxin Jiang. 2021. Codet5: Identifier-aware unified pre-trained encoder-decoder  models for code understanding and generation. In EMNLP.
- Colin B Clement, Andrew Terrell, Hanlin Mao, Joshua Dillon, Sameer Singh, and Dan Alistarh. 2020. Pymt5: Multi-mode translation of natural language and python code with transformers. In EMNLP.
- Mark Chen, Jerry Tworek, Heewoo Jun, Qiming Yuan, Henrique Ponde de Oliveira Pinto, Jared Kaplan, Harri Edwards, Yuri Burda, Nicholas Joseph, Greg Brockman, and 1 others. 2021. Evaluating large language models trained on code. arXiv preprint arXiv:2107.03374.
- Raymond Li, Lewis Tunstall, Patrick von Platen, Jungtaek Kim, Teven Le Scao, Thomas Wolf, and Alexander M. Rush. 2023a. Starcoder: May the source be with you! Preprint, arXiv:2305.06161.
- Baptiste Roziere, Jonas Gehring, Fabian Gloeckle, Sten Sootla, Itai Gat, Xiaoqing Ellen Tan, Yossi Adi, Jingyu Liu, Romain Sauvestre, Tal Remez, and 1 others. 2023. Code llama: Open foundation models for code. arXiv preprint arXiv:2308.12950.

---

## 优点与创新点 (Strengths)

1. 有图论的理论支撑，通过依赖图和拓扑排序来处理复杂的代码依赖关系。
2. 试验客观性强，紧紧依赖于代码的AST和正则表达式，避免了人工评估的主观性。

---

## 我的思考 (Personal Thoughts)

是否可以考虑将DAG图用于通过需求生成复杂业务的代码中？
对于项目级别的代码生成后，能否通过DocAgent类似的方法再生成用户手册/开发文档？

---
