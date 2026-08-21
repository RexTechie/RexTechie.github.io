---
title: "Dialogue SWE-Bench: A Benchmark for Dialogue-Driven Coding Agents"
date: 2026-08-11T16:37:56+08:00
draft: true
description: "Dialogue SWE-Bench 将真实仓库问题改造成必须与用户对话才能解决的评测，并用 schema-guided agent 检验信息收集对代码修复的作用。"
categories: ["📒论文笔记"]
tags: ["software engineering agents", "repository issue repair", "coding agent evaluation"]
---

## 基本信息

- **标题**：Dialogue SWE-Bench: A Benchmark for Dialogue-Driven Coding Agents
- **作者/单位**：Brendan King、Jeffrey Flanigan；University of California, Santa Cruz
- **来源/状态**：arXiv preprint；2026-06-12 提交，尚未核验到会议或期刊录用信息
- **版本**：arXiv:2606.13995v1，2026-06-12
- **DOI / arXiv**：10.48550/arXiv.2606.13995；arXiv:2606.13995
- **代码/数据**：[Dialogue SWE-Bench 项目页](https://jlab-nlp.github.io/dialogue-swe-bench/)

## 研究背景与问题

现有 SWE benchmark 通常假设问题说明完整且 agent 全程自主运行；真实编码助手却经常面对不完整需求，并通过对话澄清行为、约束和验收标准。论文因此把仓库级问题修复重新定义为“与用户进行目标导向对话，同时修改和执行代码”，并分别评估修复成功、对话自然度和对话连贯性（§1–§2）。

## 核心贡献

1. 提出 Dialogue-SWEBench：把 SWE-Bench Verified 的 500 个仓库级问题转成需要用户对话的评测任务。
2. 提出带 persona 和 self-revision 的用户模拟器，使用户回复受问题知识约束，并能稳定、可复现地评测多轮交互。
3. 提出 schema-guided coding agent，用动态生成和维护的对话 schema 组织待确认信息、代码探索、修改和验证，并与 OpenHands、交互式基线比较。

## 方法或系统架构

### 从自主 SWE 评测到对话评测

在标准全自主设置中，agent 从初始仓库和问题说明开始，在 observation-action loop 中执行编辑文件、运行测试等动作，并接收执行结果；`finish` 动作产生 patch，评测器把 patch 应用到基础仓库并运行任务对应测试，所有测试通过才算解决。Dialogue-SWEBench 保留这一执行判定，把输入的完整 issue 文本换成固定的初始用户查询，并在动作空间中加入 `message_user`。该动作携带一条消息，返回模拟用户的回复；agent 最终仍需提交能通过执行测试的 patch（§3.1–§3.2）。

### 初始查询与用户模拟器

每个问题的初始查询固定，以减少运行间方差。构造流程先让 LLM 根据 GitHub issue 标题和改写指令生成初始对话查询，再用修订提示删除 agent 可以直接利用、从而绕过对话的关键细节，最后人工检查查询是否忠实于原问题意图。论文报告只有 13% 的生成查询需要人工修改（§3.3）。

后续用户回复由一个开放权重 LLM 模拟。模拟器的系统提示包含该问题的完整 issue 文本和用户 persona，且把该问题中已有的所有对话消息放入上下文。它为每个问题采样姓名和手工 persona 描述，以增加对话行为的多样性。模拟器不能声称自己运行了新代码等超出环境能力的动作，因此每次先生成候选回复，再由同一个模拟模型做 self-revision：检查幻觉、任务指令违规和回复长度，发现问题后再次修订再返回给 agent（§3.3；Appendix A）。

### Schema-guided coding agent

该 agent 通过提示要求自己先判断问题类型，再建立对话 schema，schema 的键和值由 agent 根据当前问题动态决定。例如 bug 类型可能包含实际行为、预期行为和复现步骤。尚未讨论的字段标记为 `UNKNOWN`；agent 通过向用户提问、探索仓库、修改代码和运行验证逐步填写并维护这些字段，直到信息足够完成修复。系统使用 OpenHands Agent SDK 及其文件编辑、bash 和结束工具，额外加入 `message_user`（§4）。

### 评测指标与对话质量

核心指标是 resolution rate：对话结束后 patch 通过该问题全部执行测试的比例。论文还评估对话自然度和连贯性。自然度采用 1–3 分，衡量表达是否易懂、清晰和适合用户交流；连贯性同样采用 1–3 分，分别检查每一轮是否自然承接前文、以及整个对话是否朝解决问题推进，包括必要时澄清和正确结束。两个指标用 Gemma 4 31B-IT 作为 LLM-as-a-judge，并用人工标注检查判断可靠性（§3.4；§7）。

## 实验设置

### 数据、模型与工具约束

所有实验使用 SWE-Bench Verified 的 500 个问题的对话改写。模型包括闭源的 GPT-5、GPT-5-mini，开放权重的 Devstral 2 Small（24B）和 Qwen 3 Coder（30-A-3B）。用户模拟器使用量化的 Llama 3.3 70B。每个问题限制 agent 最多 100 步，并为所有 agent 提供相同的 `message_user` 工具；没有工具调用的 assistant 消息也按对话消息处理（§5）。

每个模型比较三种 agent：直接使用 OpenHands 的 off-the-shelf coding agent、为处理歧义而设计的 OH Interactive，以及论文的 schema-guided agent。三者共享同一组代码和对话工具。主要报告 resolution rate、平均对话轮数、agent 步数和每个问题会话的美元成本（§5.1）。

### 用户模拟器和额外标注

用户模拟器先以完整 issue 文本为知识边界生成回复，再用 self-revision 检查忠实性、目标一致性和环境一致性。为验证模拟器，论文抽取 120 个对话进行人工标注，并把每条回复/整个对话分别按这三项二元标准评价。为验证 LLM judge，针对每个模型—agent 组合标注 30 个对话，共 360 个对话；用二次加权 Cohen’s kappa 和系统排序准确率与人工结果比较（§7.3；§8.2）。

论文还做了去掉用户模拟器后续回复的消融：只提供固定首轮查询，后续对 `message_user` 回复“用户不可用”，并在按工程难度分层抽取的 50 个问题上运行 schema-guided agent，以验证任务成功是否确实依赖多轮交互（§8.1）。

## 主要结果

schema-guided agent 的平均解决率为 46.9%，高于 OpenHands 的 32.9% 和 OH Interactive 的 44.1%，同时平均成本最低；它通常发起更多信息寻求动作，但没有增加相应的总步数。结果还显示，代码能力更强的模型不一定拥有更好的对话能力，说明仓库级修复与用户意图协商需要分开评测（§6）。

[TODO] 原文 Figure 2：固定初始查询、用户模拟器和 agent observation-action loop 组成的 Dialogue-SWE-Bench 评测流程。

[TODO] 原文 Figure 3：schema-guided SWE agent 的 schema 生成、提问、探索、修改和验证流程。

[TODO] 原文 Table 1：不同模型和 agent scaffold 的解决率、轮数、步数与成本。

## 消融、成本与失败分析

信息寻求动作与解决率呈正相关；OpenHands 很少向用户提问，而表现更好的 agent 通常更积极地获取缺失信息。去掉用户模拟器的后续回复会让四种模型的解决率都明显下降，说明 benchmark 的成功不能只靠首轮问题和自主修复（§6；§8.1）。

自然度更多受模型选择影响，GPT-5 在部分对话中重复系统提示内容或不能正确结束，Devstral 还会在一部分会话中不回复用户而耗尽调用。连贯性更能区分 agent 设计；schema-guided agent 在部分模型上显著领先。自然度 judge 与人工标注的 kappa 为 0.70，连贯度为 0.51，后者只达到中等一致性（§7）。

用户模拟器的 self-revision 对质量很重要：去掉它后，尤其是环境一致性下降；完整模拟器在对话层面 97.5% 的样本同时满足忠实性、目标一致性和环境一致性，而去掉修订后为 82.5%。这些结果支持模拟器作为稳定评测组件，但不能把它等同于真实用户（§8.2）。

[TODO] 原文 Table 2：LLM-as-a-judge 与人工标注在自然度、连贯度上的一致性和排序准确率。

[TODO] 原文 Table 3：去掉用户模拟器后续回复的多轮交互消融。

[TODO] 原文 Table 4：用户模拟器的忠实性、目标一致性和环境一致性标注。

## 优点与局限

### 优点与创新

1. 把真实编码助手中的需求澄清纳入仓库级、执行测试判定的评测，而不是只测完整 issue 输入下的自主 patch 生成。
2. 用户模拟器有固定初始查询、知识边界、persona 和 self-revision，兼顾多轮交互的可复现性与行为多样性。
3. 同时报告修复成功、交互成本和对话质量，并用人工标注验证两个 LLM judge，避免把代码通过率当作完整的交互质量。

### 作者自述局限

1. benchmark 建立在 SWE-Bench Verified 上，继承其以 Python 仓库和特定 issue 类型为主的分布，未必覆盖更广泛的真实编码 agent 使用场景（§11）。

### 阅读分析

1. 用户回复由 LLM 模拟且知道完整 issue 文本；self-revision 可以约束明显违规，但仍不能完全模拟真实用户的知识、耐心和表达差异。
2. 任务使用经过验证的 issue，因此用于研究“如何澄清缺失需求”，但不完全代表真实生产中更不完整、更冲突的需求。
3. 连贯性 judge 与人工一致性只有中等水平；在评估需要长期上下文的对话时，人工或多评审协议仍有必要。

## 与 AutoBackend / AutoFullStack 的关系

- **关系与差异**：本轮没有提供 AutoBackend / AutoFullStack 的交互设计；如果它们允许用户通过自然语言驱动后端或全栈修改，Dialogue-SWEBench 提供了“先澄清意图、再改仓库、最后执行测试”的评测层，而不是具体生成架构。
- **可借鉴设计**：可把用户询问、确认后的需求字段、代码修改和测试结果记录成可回放轨迹，并同时测解决率、澄清轮数、token/美元成本和对话连贯性。
- **新研究问题**：在全栈任务中，schema 应如何表示 API 契约、数据迁移、UI 行为和部署约束；agent 何时应该提问、何时应该先运行测试或静态检查，才能最小化用户负担并提高最终修复率。

## 参考链接

- [论文](https://arxiv.org/abs/2606.13995)
- [PDF](https://arxiv.org/pdf/2606.13995)
- [代码/数据](https://jlab-nlp.github.io/dialogue-swe-bench/)

