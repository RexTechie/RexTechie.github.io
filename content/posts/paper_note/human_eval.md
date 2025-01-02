---
title: "Evaluating Large Language Models Trained on Code"
date: 2025-01-01T14:47:39+08:00
draft: true
draft: false
description: "一种评估大模型生成代码能力的方法"
categories: ["论文笔记"]
tags: ["代码生成",]
---

## 基本信息

- **标题**: Evaluating Large Language Models Trained on Code
- **作者**: Mark Chen、Jerry Tworek、Heewoo Jun、Qiming Yuan等
- **作者单位**: OpenAI, Anthropic AI, Zipline
- **期刊/会议**: arXiv
- **发表时间**: 2021年7月14日
- **DOI**: [2107.03374](https://arxiv.org/abs/2107.03374)
- **开源地址**: [GitHub](https://www.github.com/openai/human-eval)
- **关键词**: 代码生成, 代码评估, 代码理解

---

## 研究背景 (Background)

OpenAI在GPT3发布的时候，就已经可以通过文档(docstrings)生成python代码了。本篇文章假定有一个大规模的语言模型，Codex，可以生成代码，作者希望通过这篇文章来评估Codex的性能。

---

## 研究问题 (Research Questions)

本文旨在设定一个评估指标和方法来对模型(生成代码的模型，假定为odex)生成Python代码的能力进行评估。

---

## 评估框架 (Evaluation Framework)

作者定义了pass@k指标，以及HumanEval数据集，用于评估Codex的性能。

### 功能的正确性(Functional Correctness)

评估生成质量可以通过匹配的方式：精确匹配或模糊匹配（如BLEU得分）。然而有研究揭示了基于匹配的代码评估指标的不足。如Ren等人（CodeBLEU: a Method for Automatic Evaluation of Code Synthesis）发现BLEU在捕捉代码特有的语义特征方面存在问题，并建议对分数进行几个语义的修改。更加根本的问题是，基于匹配的方式无法评估庞大的并且复杂的代码。近期无监督的代码翻译或伪代码到代码的翻译相关问题也转向了功能性准确性的评估，如果样本通过了测试用例，那么就认为生成的代码是正确的。作者认为这种方式应用到代码生成的评估中是合理的。**评估功能正确性最有说服力的理由是程序员常使用这种方式进行评估（单元测试）。**

Kual等人在2019(SPoC: Search-based Pseudocode to Code)就使用了pass@k指标评估功能正确性。pass@k：对于每个问题生成k个代码样本，如果其中有一个样本通过了单元测试就认为问题已得到了解决，并报告解决问题的总比例。然而以这种方式计算pass@k会有很高的方差（Why?：也许是因为一个问题准确率非0即1，所以方差太大，如pass@5，有五个都通过则是1，有4个通过则是0，因此用这种方式不合适）。相反，为了评估pass@k，**我们给每个任务生成n个样本（n > k, 论文中使用n = 200, k = 100）, 统计通过单元测试的正确样本数c（c <= n），并计算无偏估计量**。其中 $\text{pass@}k := \mathbb{E}_{\text{Problems}} \left[ 1 - \frac{\binom{n-c}{k}}{\binom{n}{k}} \right] $

直接计算这个估计量会导致得到非常大的数字并且数值不稳定。在下**图3**中包含了数值稳定性的numpy实现，它简化了表达式并逐项评估乘积项。有人也许会尝试用1-(1-p_hat)^k来估pass@k, 其中p_hat是pass@1的经验估计，但是作者已经证明了这个方式是有偏的（附录A）。
![Figure 3](/images/20250101201329.png)
例子🌰：
给定n为10，k为2

| id | n | c | k | pass@k |
|----|---|---|---|--------|
| 0  | 10 | 10 | 2 | 1 |
| 1  | 10 | 9 | 2 | 1 |
| 2  | 10 | 8 | 2 | 0.98 |
| 4  | 10 | 7 | 2 | 0.93 |
| 5  | 10 | 6 | 2 | 0.86 |
| 6  | 10 | 5 | 2 | 0.78 |
| 7  | 10 | 4 | 2 | 0.67 |
| 8  | 10 | 3 | 2 | 0.53 |
| 9  | 10 | 2 | 2 | 0.38 |
| 10  | 10 | 1 | 2 | 0.2 |
| 11 | 10 | 0 | 2 | 0 |

手写部分计算过程如下所示
![calc_pass@k_by_hand](/images/20250101213539.png)

### HumanEval数据集(HumanEval: Hand-Wirtten Evaluation Set)

### 执行生成函数的沙箱(Sandbox for Executing Generated Functions)

## 方法与模型 (Methods & Models)

---

## 核心贡献 (Key Contributions)

总结本文的主要贡献点：
1.  
2.  
3.  

---

## 实验结果 (Results)

概述实验的关键结果和作者的主要发现。

---

## 参考文献 (References)

列举一些重要的参考文献

## 备注 (Notes)

1. 论文提到一个现象（Introduction），12B的Codex可以解决28.8%的问题，而300M的Codex只能解决13.2的问题。这是为什么？

---

## 优点与创新点 (Strengths)

列出本文的优点和创新点：
1.  
2.  

---

## 局限性与不足 (Limitations)

列出本文的局限性和不足：
1.  
2.  

---

## 我的思考 (Personal Thoughts)

1. 本文与我研究的相关性：  
2. 是否有可以改进的地方：  
3. 后续可能的研究方向：  

---


