---
title: "Multi-Agent Design: Optimizing Agents with Better Prompts and Topologies"
date: 2025-06-19T11:27:17+08:00
draft: false
description: "多智能体系统设计"
categories: ["📚论文阅读"]
tags: ["多智能体", "智能体拓扑", "Prompt 优化"]
---

## 基本信息

- **标题**: Multi-Agent Design: Optimizing Agents with Better Prompts and Topologies
- **作者**: Han Zhou（通讯作者）、Ruoxi Sun、Hamid Palangi、Shariq Iqbal、Ivan Vulić、Anna Korhonen和Sercan Ö. Arık。
- **作者单位**: Google和剑桥大学
- **期刊/会议**: ArXiv
- **发表年份**: 2025.02.04
- **DOI**: [2502.02533](https://arxiv.org/abs/2502.02533)
- **开源地址**: [Github](https://github.com/facebookresearch/DocAgent)
- **关键词**: Multi-Agent System, Large Language Models

## 研究问题 (Research Questions)

大语言模型，作为多个相互交互协作的智能体，可以解决复杂任务。这些智能体通过声明其功能的提示词以及协调智能体间交互的拓扑结构进行编程。本文所研究的主要问题是优化多智能体系统中的提示词以及智能体的拓扑结构。

---

## 研究背景 (Background)

尽管最近的研究探讨了自动化智能体设计各个方面，但在理解哪些因素对改进MAS性能最为关键方面仍存在差距。例如，DSPy自动化了设计示例以改进提示编程的过程。J Li(More agents is all you need作者)提出通过扩大多数投票中的代理数量来优化MAS。ADAS通过基于LLM的元代理编程代码表达的新拓扑。AFlow在预定义操作集中使用蒙特卡洛树搜索来寻找更好的拓扑。然而，包括提示词和拓扑在内的多个设计空间之间的相互作用仍然不明确。

---

## 核心贡献 (Key Contributions)

- 深入分析了影响基于LLM的MAS性能的设计因素，强调了prompt的重要性，并确定了有影响力的拓扑结构。
- 提出了一种名为Mass的新型多阶段优化器，通过在具有影响力的搜索空间中交错优化prompt和拓扑结构来自动化MAS设计。
- 大量数据在各种评估基准上显示出显著的性能提升，为构建有效的未来多智能体系统提供了指导。

---

## 设计多智能体系统（Designing Multi-Agent Systems）

作者认为MAS的设计可以分为两个层级：块级设计(`Block-level`)和工作流编排(`Workflow-level orchestration`)。对于块级，目标是设计单个智能体，通过更好的提示词设计最好的提示词来表现出特定的角色。对于工作流编排，它的优化涉及要包含智能体的类型和数量，以及如何以最有效的方式编排他们，这被称为拓扑优化。

$\mathcal{W}^{*}(a)=\arg\max\mathbb{E}_{(x,y)\sim\mathcal{D}}[f(\mathcal{W}(a)(x)),y]$

### 块级：智能体的提示词设计（Block-level: Prompt Design for Agents）

对于块级，对下游任务影响最主要的是`prompt`，它定义了智能体的角色（例如，“You are an expert in reflecting on errors...”），提供额外的指令来塑造其行为（例如，“You should think step by step...”）以及可选地包含少量示例（zero-shot/one-shot/few-shot）来指导智能体的回复。比如，一种SOAT提示词优化器同时搜索指令和少量示例，其中示例是从模型自身在验证集上的正确预测中引导出来的，基于验证指标。基于这些示例，提示词优化器会为指令提出一些候选方案，并提供数据集摘要或各种提示词以提高候选方案的多样性。然后指令和示例会被联合优化。

尽管大家都指导LLMs对提示词很敏感，但将自动提示词优化（`APO`，automatic prompt optimization）技术应用与多智能体系统并非易事。与单轮任务不同，单轮任务中APO可以通过将提示词视为可优化变量并将验证集的性能作为目标来轻松执行。在 MAS 中，由于智能体之间的相互依赖性（例如，一个智能体的输出可能是另一个智能体的输入，且中间输出的真实响应不可用）以及随着参与智能体数量n的增加而呈指数级增长的组合优化复杂性，APO变得更加复杂。当n增加时，奖励信号也变得更加稀疏，使得难以在MAS系统中使用APO；因此，MAS中的许多先前工作仍然主要使用手动设计提示词，而不是将提示词作为MAS设计中可优化的组件。

为了系统地理解提示词设计在MAS中的影响，作者具体且定量地分析了提示词优化的效果，并将其有效性与MAS文献中常见的其他操作进行比较，例如使用更多代理进行扩展但使用默认提示。我们在CoT智能体上进行APO，结合通过MIPRO的指令优化和1-shot示例优化，并公平地将总推理token成本与self-consistency、self-refine和multi-agent debate进行比较。结果如下图所示，提示词为智能体提供了更具信息性的指令和示例，在token有效性方面显示出显著优势，优于其他构建模块。此外，通过在提示词优化智能体的基础上应用self-consistency，作者观察到token成本的扩展性能有所提高，而在扩展智能体数量的标准方法（例如SC或Reflect）中，饱和得更早。这一实证观察揭示了提示词的重要性，同时为设计有效的MAS提供了早期证据——在扩展拓扑之前先局部优化代理。

![Accuracy vs total token counts for prompt-optimized agent](/images/20250626105724.png)

### 工作流级搜索空间设计（Workflow-level Search Space Design）

在工作流层面，主要关注的是协调智能体以有效地达到最佳性能。作为MAS特有的一个相对较新的概念，`拓扑优化`最近引起了显著关注。然而，尽管现有研究大多强调`search methods`（例如发现识别最佳配置的最有效和高效的方法），但对搜索空间设计的关注较少，而搜索空间设计决定了任何搜索算法的范围和边界。这种不平衡与`neural architecture search（NAS）`的历史发展相似。最初，该领域集中于复杂的搜索方法，如贝叶斯优化和可微分搜索。后续的工作强调了搜索空间设计的重要性，这一方面常常被忽视，认为其同样重要，甚至更为关键。受到这一见解的启发，我们假设人为设计的拓扑可能不是最优的，而自动拓扑优化（可能被框定为一个严格的优化问题）可以通过精心设计MAS的搜索空间发挥同样关键的作用。为此，我们首先定义一个富有表现力的搜索空间，类似于之前的工作，它由以下`building blocks`之间的连接组成：

- `Aggregate`：智能体可以并行进行多样化的预测，之后再通过一个聚合操作获得一个统一的预测结果。聚合块可以通过$N_a$个智能体并行操作进行参数化。例如：Majority vote和Self-consistency。
- `Reflect`：智能体可以作为验证者，基于先前的预测提供批评和改进建议。然后将反馈输入预测器或反思者本身，以进行迭代改进。同样，`Reflect`可以通过定义`self-reflect`轮次数量的$N_r$进行参数化。例如：Self-refine和Relexion。
- `Debate`：在辩论中的智能体可以比单代理预测得出更真实的预测，其中每个辩论智能体会收集所有其他智能体的意见并提供更新后的回应。这种拓扑结构将涉及多种智能体，而$N_d$定义了辩论的回合数。
- `Custom Agents`：虽然前三种形式的智能体代表作为多个并行、串行和智能体混合的智能体拓扑结构的大多数，但更通用的智能体定义可以插入到MAS设计空间中。例如，对于特定任务的使用案例，作者引入一个智能体作为总结，以改善在可定制设计空间中的长上下文能力。
- `Tool-use`：构建有效的MAS，使智能体能够利用工具访问外部信息对于系统性能至关重要，例如使用检索器进行RAG和具有测试用例的执行器进行编码。作者引入工具使用作为一个可优化的二元“插入”决策$N_T \in \{0, 1\}$。

为了理解各个拓扑的影响，作者在下图中报告了各种拓扑的性能。值得注意的是，并非所有拓扑都对MAS设计有利，而是只有一小部分拓扑对其有积极影响。例如，在HotpotQA中，只有`Debate`带来了3%的提升，而其他拓扑未能改善甚至降低系统性能。我们在LiveCodeBench的测试输出预测子任务中再次观察到类似趋势。这强调了在搜索空间的影响集合中进行搜索的重要性，而包含递减的构建块不仅可能导致更高的搜索复杂性，还可能降低性能。

![The performance of different topologies](/images/20250626142930.png)

## MASS: Multi-Agent System Search

前面作者分析，定义搜索空间非常的重要，在这个基础上提出了一种多阶段优化算法，即多智能体系统搜索（MASS，Multi-Agent System Search）。这超越了之前只关注优化工作流拓扑而没有合适的提示词设计的研究。相反，作者的方法展示了通过适当优化提示词和精心设计的搜索空间进行MAS设计的更有效。MASS算法如下图所示，遵循从局部到全局、从块级到工作流的，通过以下详细的每阶段优化来克服组合优化的复杂性。

![Mass framework](/images/20250626152213.png)

![Algorithm MASS: Multi-Agent System Search](/images/20250626153418.png)

1）Block-level prompt optimization: 在组合智能体之前，首先确保单个智能体在块级别上经过彻底优化，这一步确保每个智能体在最易管理的计算预算中以最有效的指令为其角色做好准备。为了进一步克服在大型MAS空间上联合优化的复杂性，我们首先通过单智能体APO来热身初始`predictor`，$a_{0}^{*} \leftarrow O_{\mathcal{D}}(a_{0})$，其中指令和示例都与模块化提示词优化器O共同优化。接着在预热过`predictor`后的条件下，继续以最少数量的智能体优化每个拓扑，$a_{i}^{*} \leftarrow O_{\mathcal{D}}(a_{i} \mid a_{0}^{*})$，这样，2个`predictor`与1个`debator`配对形成最小构建块作为`debate`拓扑，从而降低优化的复杂性，并且该拓扑可以在以后通过更多的`predictor`和`debator`进行扩展，但都配备了优化的提示。为了衡量每个构建块的影响，我们在优化完成后存储验证性能。重要的是，尽管阶段（1）作为每个构建块的热身阶段，但它仍然是一个关键阶段，确保后续的拓扑优化在一个有效的空间中进行，组合表现良好的代理，而不是因任何手动提示的格式不佳的代理的复合影响而受苦。

2）Workflow topology optimization：在此阶段，专注于优化整体MAS结构，确定智能体之间最有效的排列和连接。之前的分析显示，有利的拓扑仅占整个设计空间的一小部分。因此，重点旨在将表现强劲的拓扑的精髓提炼到一个精简的空间，从而使工作流级别的拓扑搜索更加高效。在此，提出测量增量影响$I_{a_{i}} = \frac{\mathcal{E}(a_{i}^{*})}{\mathcal{E}(a_{0}^{*})}$，量化将拓扑$a_{i}$整合到初始代理$a_{0}$中的相对增益。根据直觉，影响力大的维度具有更高的选择概率，如果$u > p_{a}$，则激活相应的拓扑维度a，其中$u ∼ U(0, 1)$且$p_{a} = Softmax(I_{a}, t)$。为了将多样的拓扑组合成一个统一的空间，我们通过基于规则的顺序约束工作流以减少优化复杂性，遵循预定义的顺序，如[`summarize`，`reflect`，`debate`，`aggregate`]。在预定义的设计空间上整合拒绝采样，拒绝任何停用的维度或超过代理数量最大预算B的无效拓扑组合。

3）Workflow-level prompt optimization：作为最后一步，我们将整个MAS设计视为一个整体实体，并在第二阶段发现的最佳拓扑$\mathcal{W}^{*} = \mathcal{O}_{\mathcal{D}}(\mathcal{W}_{c}^{*})$的条件下进行额外一轮的提示优化。值得注意的是，尽管在第一阶段提示是在个体层面进行优化的，这一阶段起到了适应或微调的作用，确保提示适合在MAS中进行编排，并且优化了代理之间的相互依赖性。实验表明，这一阶段通常会带来实际的好处。

## 试验（Experiments）

### 基准（Benchmarks）

1. 推理：Hendryck's MATH、DROP
2. 多跳长文本：HotpotQA、MuSiQue、2WikiMultiHopQA
3. 编程：MBBPP、HumanEval、LCB(LiveCodeBench)

### 基线（Baselines）

1. CoT：通过零样本提示进行直接思维链推理。
2. CoT-SC：具有自一致性，从多样化的推理轨迹中找到最一致的答案。
3. Self-Refine：反思性智能体验证和自我改进预测。
4. Multi-Agent Debate：智能体通过辩论答案并聚合其他智能体的信息。
5. ADAS：一种自动智能体设计框架，基于LLM的元智能体根据先前的评估迭代提出新智能体
6. AFlow：通过蒙特卡罗树搜索在一组预定义的操作符上进行自动工作流设计。

作者通过将代理的最大数量限制为10来公平比较所有基线。

### 实验设置（Setup）

MASS整合了最先进的提示词优化器MIPRO，通过`贝叶斯替代模型（Bayesian surrogate model）`优化每个指令和示例。作者限制示例数量为3，候选指令数量为10，每个智能体迭代优化10次。在所有任务的拓扑优化中，作者通过`拒绝采样（Rejection sampling）`算法搜索10中不同的拓扑结构。与拓扑优化同时，每个拓扑在验证集上评估3次以稳定预测。优化后的MAS随后在保留的测试集上进行了三次运行并报告结果。对于模型，Temperature = 0.7, 最大输出token为4096，Softmax中的t设置为0.05以增强每个搜索维度的选择概率$p_{a}$。在所有阶段，作者都在评估器和优化器中实现了相同的LLM骨干模型。

### 主要结果（Main Results）

![Main Results](/images/20250626190515.png)

如结果所示，MASS在多智能体系统中取得了显著的提升。通过将MASS与最先进的自动智能体设计基线ADAS和AFlow进行比较，作者注意到，即使ADAS已经基于常见的智能体形式来生成元智能体，它也仅带来了细微的提升。元智能体不断提出复杂的拓扑结构，但没有优化提示词设计。另一方面，AFlow在2WikiMQA和HumanEval上表现出与Mass相当的竞争力。

作者将AFlow的性能归因于：1）其“扩展”阶段基于错误日志生成新节点，该日志将预测与真实情况进行对比，从而提供隐式文本梯度，以反映提示设计中的任何格式错误；2）在预定义的操作符集合中进行更精细的搜索空间。尽管AFlow在搜索空间设计的重要性上与MASS有相似的灵感，但它仍然缺乏提示词优化阶段来正确优化其预定义的操作符，导致在MATH和MuSiQue的MAS搜索结果中表现不佳。与这些基线不同，Mass带来的持续改进突显了在提示和拓扑设计空间中进行搜索的重要性。

### 消融优化阶段（Ablating optimization stages）

![Ablating optimization stages](/images/20250626190547.png)

首先，注意到块级优化和单智能体优化之间有很大的增益，平均为6%，这表明MAS通过在构建块内优化其智能体受益匪浅。此外，从阶段(1)到(2)，通过在搜索最佳配置时组合有影响力的拓扑结构，可以实现额外的3%增益。在此，作者提供了一个额外的消融研究，探讨在没有事先进行提示优化或没有搜索空间修剪的情况下进行阶段(2)。图（右）显示，这两者对于有效的搜索空间探索都是至关重要的。最后，MASS通过在最佳找到的拓扑上进行工作流级提示优化获得了进一步的增益（约2%），这表明优化提示以建模智能体之间的相互依赖性在MAS设计中是有益的。

### MASS的成本效益（Cost-effectiveness of MASS）

![Cost-effectiveness of MASS](/images/20250626190620.png)

作者对MASS的成本效益进行了分析。特别是，作者可视化了MASS的优化轨迹，如图所示。MASS的轨迹显示出一种稳定的优化趋势，通过交替搜索更好的提示词和拓扑结构，逐步提高验证性能。然而，对于没有明确提示词优化阶段的自动设计基线，由于MCTS的性质，AFlow在其优化中暴露出更大的方差，而ADAS则陷入发现过于复杂的拓扑结构，这些结构似乎不如提示词计设空间有效。总体而言，MASS的优化轨迹突出了在有效设计空间中进行优化的重要性，其中交替优化通过更多连续的奖励进一步解决了复杂性。

### 最佳MAS架构与原则（Best-found MAS architectures & Design principles）

![Best-found MAS architectures & Design principles](/images/20250626190659.png)

作者进一步检查了一个优化提示词的示例以及MASS在发现更有效拓扑结构中的轨迹，如图所示。优化从zero-shot CoT智能体开始，很快MASS在阶段(1)中通过其优化提示词识别出高性能拓扑。然而，如阶段(2)中所发现的，聚合更多并行智能体实际上比多智能体辩论更有优势。工作流级别的提示词优化随后导致了聚合的最佳预测器。整体优化流程为我们构建有效MAS的指南提供了启示：1）在将个体智能体组合成MAS之前，适当优化个体智能体是重要的；2）通过组合有影响力的拓扑可以构建更有效的MAS；3）建模智能体之间的相互依赖是有益的，可以通过工作流级别的联合优化实现。

## 我的思考 (Personal Thoughts)

本文对于智能体的优化，着重关注提示词的优化，这一下就抓到了LLM的核心。而对于工作流编排部分，着重需要优化的是工作流的拓扑结构。这给后续多智能体的开发提供了指导，可以增加一些对于智能体提示词和工作流拓扑结构的优化模块。

## 参考文献 (References)

### Forms of LLM-based agentic systems

- S. Yao, J. Zhao, D. Yu, N. Du, I. Shafran, K. R. Narasimhan, and Y. Cao. React: Synergizing reasoning and acting in language models. In The Eleventh International Conference on Learning Representations, 2023. URL https://openreview.net/forum?id=WE_vluYUL-X.
- Q. Wu, G. Bansal, J. Zhang, Y. Wu, S. Zhang, E. Zhu, B. Li, L. Jiang, X. Zhang, and C. Wang. Autogen: Enabling next-gen llm applications via multi-agent conversation framework. arXiv preprint arXiv:2308.08155, 2023.
- W. Chen, Y. Su, J. Zuo, C. Yang, C. Yuan, C.-M. Chan, H. Yu, Y. Lu, Y.-H. Hung, C. Qian, Y. Qin, X. Cong, R. Xie, Z. Liu, M. Sun, and J. Zhou. Agentverse: Facilitating multi-agent collaboration and exploring emergent behaviors. In The Twelfth International Conference on Learning Representations, 2024b. URL https://openreview.net/forum?id=EHg5GDnyq1.
- J. Li, Q. Zhang, Y. Yu, Q. FU, and D. Ye. More agents is all you need. Transactions on Machine Learning Research, 2024a. ISSN 2835-8856. URL https://openreview.net/forum?id=bgzUSZ8aeg.
- X. Wang, J. Wei, D. Schuurmans, Q. V. Le, E. H. Chi, S. Narang, A. Chowdhery, and D. Zhou. Selfconsistency improves chain of thought reasoning in language models. In The Eleventh International Conference on Learning Representations, 2023. URL https://openreview.net/forum?id= 1PL1NIMMrw.
- A. Madaan, N. Tandon, P. Gupta, S. Hallinan, L. Gao, S. Wiegreffe, U. Alon, N. Dziri, S. Prabhumoye, Y. Yang, et al. Self-refine: Iterative refinement with self-feedback. Advances in Neural Information Processing Systems, 36, 2024.
- A. Singh, A. Ehtesham, S. Kumar, and T. T. Khoei. Agentic retrieval-augmented generation: A survey on agentic rag. arXiv preprint arXiv:2501.09136, 2025.
- X. Chen, M. Lin, N. Schärli, and D. Zhou. Teaching large language models to self-debug. In The Twelfth International Conference on Learning Representations, 2024d. URL https://openreview. net/forum?id=KuPixIqPiq.
- L. Lin, J. Fu, P. Liu, Q. Li, Y. Gong, J. Wan, F. Zhang, Z. Wang, D. Zhang, and K. Gai. Just ask one more time! self-agreement improves reasoning of language models in (almost) all scenarios. In L.-W. Ku, A. Martins, and V. Srikumar, editors, Findings of the Association for Computational Linguistics: ACL 2024, pages 3829–3852, Bangkok, Thailand, Aug. 2024. Association for Computational Linguistics. doi: 10.18653/v1/2024.findings-acl.230. URL https://aclanthology.org/2024. findings-acl.230/.
- J. Chen, S. Saha, and M. Bansal. ReConcile: Round-table conference improves reasoning via consensus among diverse LLMs. In L.-W. Ku, A. Martins, and V. Srikumar, editors, Proceedings of the 62nd Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers), pages 7066–7085, Bangkok, Thailand, Aug. 2024a. Association for Computational Linguistics. doi: 10.18653/v1/2024.acl-long.381. URL https://aclanthology.org/2024.acl-long.381/.
- Q. Wang, Z. Wang, Y. Su, H. Tong, and Y. Song. Rethinking the bounds of LLM reasoning: Are multi-agent discussions the key? In L.-W. Ku, A. Martins, and V. Srikumar, editors, Proceedings of the 62nd Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers), pages 6106–6131, Bangkok, Thailand, Aug. 2024c. Association for Computational Linguistics. doi: 10.18653/v1/2024.acl-long.331. URL https://aclanthology.org/2024.acl-long.331/.
- J. Zhang, X. Xu, N. Zhang, R. Liu, B. Hooi, and S. Deng. Exploring collaboration mechanisms for LLM agents: A social psychology view. In L.-W. Ku, A. Martins, and V. Srikumar, editors, Proceedings of the 62nd Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers), pages 14544–14607, Bangkok, Thailand, Aug. 2024c. Association for Computational Linguistics. doi: 10.18653/v1/2024.acl-long.782. URL https://aclanthology.org/2024.acl-long.782/.
- Y. Du, S. Li, A. Torralba, J. B. Tenenbaum, and I. Mordatch. Improving factuality and reasoning in language models through multiagent debate. In Forty-first International Conference on Machine Learning, ICML 2024, Vienna, Austria, July 21-27, 2024. OpenReview.net, 2024. URL https: //openreview.net/forum?id=zj7YuTE4t8.
- A. Khan, J. Hughes, D. Valentine, L. Ruis, K. Sachan, A. Radhakrishnan, E. Grefenstette, S. R. Bowman, T. Rocktäschel, and E. Perez. Debating with more persuasive LLMs leads to more truthful answers. In Forty-first International Conference on Machine Learning, 2024. URL https://openreview. net/forum?id=iLCZtl7FTa.
- C. Qian, Z. Xie, Y. Wang, W. Liu, Y. Dang, Z. Du, W. Chen, C. Yang, Z. Liu, and M. Sun. Scaling large-language-model-based multi-agent collaboration. arXiv preprint arXiv:2406.07155, 2024.
- J. Wang, J. Wang, B. Athiwaratkun, C. Zhang, and J. Zou. Mixture-of-agents enhances large language model capabilities. arXiv preprint arXiv:2406.04692, 2024b.

### Automatic optimization for MAS
- S. Zhang, J. Zhang, J. Liu, L. Song, C. Wang, R. Krishna, and Q. Wu. Offline training of language model agents with functions as learnable weights. In Forty-first International Conference on Machine Learning, 2024d. URL https://openreview.net/forum?id=2xbkWiEuR1.
- W. Zhang, K. Tang, H. Wu, M. Wang, Y. Shen, G. Hou, Z. Tan, P. Li, Y. Zhuang, and W. Lu. Agentpro: Learning to evolve via policy-level reflection and optimization. In L.-W. Ku, A. Martins, and V. Srikumar, editors, Proceedings of the 62nd Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers), pages 5348–5375, Bangkok, Thailand, Aug. 2024e. Association for Computational Linguistics. doi: 10.18653/v1/2024.acl-long.292. URL https://aclanthology.org/2024.acl-long.292/.
- S. Qiao, N. Zhang, R. Fang, Y. Luo, W. Zhou, Y. Jiang, C. Lv, and H. Chen. AutoAct: Automatic agent learning from scratch for QA via self-planning. In L.-W. Ku, A. Martins, and V. Srikumar, editors, Proceedings of the 62nd Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers), pages 3003–3021, Bangkok, Thailand, Aug. 2024. Association for Computational Linguistics. doi: 10.18653/v1/2024.acl-long.165. URL https://aclanthology.org/2024. acl-long.165/.
- O. Khattab, A. Singhvi, P. Maheshwari, Z. Zhang, K. Santhanam, S. V. A, S. Haq, A. Sharma, T. T. Joshi, H. Moazam, H. Miller, M. Zaharia, and C. Potts. DSPy: Compiling declarative language model calls into state-of-the-art pipelines. In The Twelfth International Conference on Learning Representations, 2024. URL https://openreview.net/forum?id=sY5N0zY5Od.
- W. Zhou, Y. Ou, S. Ding, L. Li, J. Wu, T. Wang, J. Chen, S. Wang, X. Xu, N. Zhang, et al. Symbolic learning enables self-evolving agents. arXiv preprint arXiv:2406.18532, 2024d.
- Z. Li, S. Xu, K. Mei, W. Hua, B. Rama, O. Raheja, H. Wang, H. Zhu, and Y. Zhang. Autoflow: Automated workflow generation for large language model agents. arXiv preprint arXiv:2407.12821, 2024c.
- Y. Shang, Y. Li, K. Zhao, L. Ma, J. Liu, F. Xu, and Y. Li. Agentsquare: Automatic llm agent search in modular design space. arXiv preprint arXiv:2410.06153, 2024.
- Z. Liu, Y. Zhang, P. Li, Y. Liu, and D. Yang. A dynamic LLM-powered agent network for taskoriented agent collaboration. In First Conference on Language Modeling, 2024b. URL https: //openreview.net/forum?id=XII0Wp1XA9.
- J. Saad-Falcon, A. G. Lafuente, S. Natarajan, N. Maru, H. Todorov, E. Guha, E. K. Buchanan, M. Chen, N. Guha, C. Ré, et al. Archon: An architecture search framework for inference-time techniques. arXiv preprint arXiv:2409.15254, 2024.
- M. Zhuge, W. Wang, L. Kirsch, F. Faccio, D. Khizbullin, and J. Schmidhuber. GPTSwarm: Language agents as optimizable graphs. In Forty-first International Conference on Machine Learning, 2024. URL https://openreview.net/forum?id=uTC9AFXIhg.
- S. Hu, C. Lu, and J. Clune. Automated design of agentic systems. arXiv preprint arXiv:2408.08435, 2024a.
- J. Zhang, J. Xiang, Z. Yu, F. Teng, X. Chen, J. Chen, M. Zhuge, X. Cheng, S. Hong, J. Wang, et al. Aflow: Automating agentic workflow generation. arXiv preprint arXiv:2410.10762, 2024b.
