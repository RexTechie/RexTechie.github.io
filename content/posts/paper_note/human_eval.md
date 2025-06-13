---
title: "Evaluating Large Language Models Trained on Code"
date: 2025-01-01T14:47:39+08:00
draft: false
description: "一种评估大模型生成代码能力的方法"
categories: ["📒论文笔记"]
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

## 研究背景(Background)

对于代码生成任务由于语言模型的推动最早的研究在1963、1971年就开始了（[Experiments with a heuristic compiler](https://dl.acm.org/doi/pdf/10.1145/321186.321192)、[Toward automatic program synthesis](https://dl.acm.org/doi/pdf/10.1145/362566.362568)）,最近的有基于掩码语言模型(masked language modeling)和跨度预测(span prediction)改编用于训练其编程对应物的有[CodeBert](https://arxiv.org/pdf/2002.08155)和[PyMT5](https://arxiv.org/pdf/2010.03150)。OpenAI在GPT3发布的时候，就已经可以通过文档注释(docstrings)生成python代码了，但是能力还是比较有限。

---

## 研究目标(Research Objectives)

- 提出一种评估大型语言模型生成代码能力的方法，包括pass@k指标和HumanEval数据集。
- 提出Codex，通过对GPT-3进行微调。
- 提出Codex-S，通过有监督微调Codex。
- 提出Codex-D，通过训练docstring生成模型。

---

## 评估框架 (Evaluation Framework)

作者定义了pass@k指标，以及HumanEval数据集，用于评估Codex的性能。

### 功能的正确性(Functional Correctness)

评估生成质量可以通过匹配的方式：精确匹配或模糊匹配（如BLEU得分）。然而有研究揭示了基于匹配的代码评估指标的不足。如论文[CodeBLEU: a Method for Automatic Evaluation of Code Synthesis](https://arxiv.org/pdf/2009.10297)中发现BLEU在捕捉代码特有的语义特征方面存在问题，并建议对分数进行几个语义的修改。更加根本的问题是，基于匹配的方式无法评估庞大的并且复杂的代码。近期无监督的代码翻译或伪代码到代码的翻译相关问题也转向了功能性准确性的评估，如果样本通过了测试用例，那么就认为生成的代码是正确的。作者认为这种方式应用到代码生成的评估中是合理的。**评估功能正确性最有说服力的理由是程序员常使用这种方式进行评估（单元测试）。**

Kual等人在2019([SPoC: Search-based Pseudocode to Code](https://proceedings.neurips.cc/paper/2019/file/7298332f04ac004a0ca44cc69ecf6f6b-Paper.pdf))就用了pass@k，对于每个问题生成k个代码样本，如果其中有一个样本通过了单元测试就认为问题已得到了解决，并计算解决问题的总比例。然而作者认为以这种方式计算pass@k会有很高的方差，而且计算量大。相反，为了评估pass@k，**我们给每个任务生成n个样本（n > k, 论文中使用n = 200, k = 100）, 统计通过单元测试的正确样本数c（c <= n），并计算无偏估计量**。其中 $\text{pass@}k := \mathbb{E}_{\text{Problems}} \left[ 1 - \frac{\binom{n-c}{k}}{\binom{n}{k}} \right]$。

直接计算这个估计量会导致得到非常大的数字并且数值不稳定。在下**图3**中包含了数值稳定性的numpy实现，它简化了表达式并逐项评估乘积项。有人也许会尝试用1-(1-p_hat)^k来估pass@k, 其中p_hat是pass@1的经验估计，但是作者已经证明了这个方式是有偏差的。
![Figure 3](/images/20250101201329.png)
上图的推导证明如下所示
![tuidao](/images/20250107123946.png)
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

- 数据集名称：HumanEval
- 数据集规模：164个**手写**编程问题
- 数据集结构：函数签名、问题描述、主体和单元测试（每个问题平均有7.7个单元测试i）。
- 数据集来源：数据集必须是手写的，因为模型本身已经从Github训练了，Github中已经包含了解决方案。
- 数据集地址：[GitHub](https://www.github.com/openai/human-eval)

### 执行生成函数的沙箱(Sandbox for Executing Generated Functions)

由于公开的代码具有未知意图，因此在执行生成的代码时，需要一个沙箱环境，以防止恶意代码的执行。开发沙箱环境的目标是防止这些程序修改、在主机或网络上获得持久性、访问敏感资源或从主机或网络中窃取数据。由于OpenAI的训练是在K8s集群和云服务上，因此沙箱也一样。

作者选择了gVisor容器作为主要主机保护组件（Unsupervised Translation of Programming Languages）。不使用Docker，是因为Docker容器运行时可以与容器共享主机资源，恶意容器可能会危及主机。gVisor通过模拟其资源在主机和容器之间引入一个安全边界来保护主机。网络相邻的主机和服务受基于eBPF防火墙规则的保护，这些规则阻止了入站和出站的链接（实验所需连接以外的连接）

---

## 代码微调 (Code Fine-Tuning)

微调GPT(12B参数) -> Codex

### 数据收集 (Data Collection)

- 来源：Github上54亿个公共仓库
- 规模：178GB的Python文件，每个Python文件大小不到1MB。过滤了之后剩下159GB的数据。
- 数据清洗：过滤掉了可能是自动生成、平均行数大于100、最大行数大于1000或者包含一小部分字母数字字符的文件。

### 方法 (Method)

- 学习率：与GPT相同
  - 学习率预热（防止模型开始训练时不稳定，让开始训练的几个step学习率小一些）：175步
  - 学习率调整：余弦学习率衰减
  - 优化器：Adam优化器，β1 = 0.9, β2 = 0.95, €= 10−8，权重衰减系数为0.1
- 词法分析器：代码词法分析器（code lexer），由于Github上的代码和自然语言在数据分布上不一样，因此词法分析器也要不一样。造成模型低效的的最大的原因是对里面whitespace的编码，因此作者添加了一组token来表示不同长度的whitespace。
- 评估
  - 使用pass@k指标
  - 评估数据：将每个HumanEval问题整理成Prompt，包含元注解（header）、函数声明（signature）、文档注释（docstring）。如下图2所示。
    ![Figure 2](/images/20250102145409.png)
  - 停止采样：停止采样符（['\nclass', '\ndef', '\n#', '\nif', '\nprint']中的一个，否则模型会生成额外的函数或语句）
  - 采样策略：采用核采样（Top-p），并设置top-p=0.96

### 实验结果 (Results)

![Figure 4](/images/20250102151153.png)
如Kaplan等人[Scaling Laws for Neural Language Models](https://arxiv.org/pdf/2001.08361)指出的，语言模型测试损失在模型大小上遵循幂律。代码微调后的测试损失也遵循类似的形式，$\left( \frac{N}{5.92 \times 10^7} \right)^{-0.13}$，其中N是模型中非embedding参数的参数数量。

![Figure 5](/images/20250102152215.png)
探究不同的k值与temperature的关系。图中可以得出temperature越高，k值越高，则结果越好，因为样本具有更高的多样性，而度量reward只关注是否生成正确解。

![Figure 6](/images/20250102153514.png)
对于679M参数的模型，pass@1的最佳temperature是0.2， pass@100的最佳temperature是0.8。由实验结果可知，他们的pass@k随参数量的变化非常的平滑。

![Figure 7](/images/20250102181307.png)
Pass@k是评估k个样本中有多少个通过了单元测试，为了更加接近真实值，引入了n和c，只要n足够大，得到的pass@k就会更接近于真实值。但在实际推理过程的选择中，我们也必须从生成的样本中选择一个样本。根据上图可知选择具有最高平均token对数概率比随机选择要好，但是随机选择比依据对数概率总和来选要稍差。这个图表说明了选择样本的启发式算法的重要性。对于Docstring backtranslation后文会提到，TODO。

![Figure 8](/images/20250102181559.png)
上图是计算Codex-12B（temperatire = 0.8） HumanEval样本与其参考解决方案的BLEU分数。纵坐标是概率密度，横坐标是BLEU分数。显然，BLEU分数的提高可能并不代表在实际中运行正确。

### 相关模型和系统的对比分析 (Comparative Analysis of Related Models and Systems)

类似Codex，[GPT-Neo](https://github.com/eleutherai/gpt-neo)、[GPT-J](https://github.com/kingoflolz/mesh-transformer-jax)基于[The Pile](https://arxiv.org/pdf/2101.00027)进行训练。这个数据集来自各种来源以及8%的Github代码。作者使用HumanEval数据集对GPT-Neo和GPT-J进行了评估。

![Table 1](/images/20250103103810.png)

对于GPT-Neo，pass@1达到了6.4%，pass@100达到了21.3%，而同等规模的GPT模型在这两个指标上都接近于0%。GPT-Neo-2.7B相当于Codex-85M（参数量少了30倍）。temperature为0.2、0.4和0.8时结果最好。
对于GPT-J-6B，pass@1达到了11.7%，pass@100达到了27.7%。相当于Codex-300M（参数量少了20倍）。temperature为0.2和0.8时结果最好。

与当时最好的代码补全模型Tabine做了比较，Tabine在pass@1(T = 0.4)为2.6%，pass@100(T = 0.8)为7.6%。相当于Codex-12M。

### APPS数据集上的结果（Results on the APPS Dataset）

APPS(Hendrycks2021年提出, Pre-training of deep bidirectional transformers for language understanding)是一个用于评估代码生成模型的数据集。这个数据集包含5000个训练样本和5000个测试样本，都是关于代码问题，每个都包含单元测试。对于训练数据，包含正确的解决方案。

与HumanEval不同，APPS数据测试类似OJ判题，从sdin和sdout读取输入输出。

APPS论文中测试了找到问题正确解决方案的比例（strict accuracy）和单元测试的通过率（即使答案不对，但只要通过测试）。后一个的度量是为了减少方差（因为第一个准确率很低），因此在测试时要尽量避免strict accuracy。

- 在程序竞赛和APPS数据上，会给出三个输入输出用例。作者从模型中采样1000个解决方案，将通过这3个单元测试的解决方案筛选出来。最后计算这个筛选后的样本的pass@k。
- 考虑算法效率，在算法竞赛中超时是不能被接受的.但是对于Codex，作者评估中使用了3秒的超时时间。

![Table 2](/images/20250103104118.png)

由于Codex没有在APPS上进行指令微调，因此在提示词中添加了input/output示例。在表中表示为"1-shot"。实验结果表明用3个input/output测试过滤出来的样本，再去评估pass@k，效果更好。

## 有监督微调 (Supervised Fine-Tuning)

有些从Github上找到的代码是包含类的实现、配置文件、脚本、甚至用来存储数据，这些与生成代码无关。作者假定这些错误匹配的代码会影响HumanEval的性能。作者分别收集了两类数据，一类是来自编程竞赛网站（和我一开始对于做代码生成的想法一摸一样，不过看来三年前就已经有人在做了🤔。。。），一类是持续集成仓库。用这些数据来训练出来的模型，作者称之为Codex-S。

### 编程竞赛中的问题（Problems from Competitive Programming）

OJ判题大多通过隐藏的单元测试来自动判断提交的功能正确性，这些单元测试通常有优秀的测试覆盖率，是经过了精心设计的。作者从多个流行的编程竞赛和面试网站手机了相关的问题描述、函数声明以及解决方案，把他们整理成类似HumanEval的格式，使用问题描述作为docstring。由于网站的单元测试是不公开的，作者依据问题的描述创建单元测试，或者通过提交错误解决方案提取了额外的测试用例。总共，作者收集了10000个问题。（🤡这方面的工作已经比较成熟了，感觉不好从这方面继续做研究了。）

### 持续集成中的问题（Problems from Continuous Integration）

对于使用了持续集成（CI）相关的项目，使用python中sys.setprofile可以记录函数调用的输入和输出，这些数据天然的可以作为单元测试。作者考虑使用GitHub仓库，其中使用Travis和Tox作为它们的持续集成（CI）框架，因为它们是最受欢迎的CI工具之一。作者最终从数百万个问题中收集了40000个问题，因为不是所有的问题都接受输入并返回输出的。

### 过滤问题（Filtering Problems）

为了控制模型的训练数据质量，作者使用Codex-12B对每一个任务生成100个样本，如果没有一个样本通过测试，则说明这个问题的描述是不清晰的，作者会将这个问题过滤掉。

### 微调方法 (Method)

用这些最后的训练数据对Codex进行有监督微调得到Codex-S。作者训练主要是最小化参考解决方案的负对数似然。他们使用了训练Codex学习率的1/10的学习率来微调Codex-S，但学习率计划是一样的，训练直到验证集损失达到平稳。

### 微调实验结果 (Results)

和Codex一样，他们首先计算pass@k的最佳temperature（1 <= k <= 100）。作者发现对于k > 1，Codex-S更喜欢略高的temperature，这可能反映了Codex-S的分布比Codex更狭窄。作者对于pass@1使用temperature=0, pass@100使用temperature=1。

![Figure 9](/images/20250106184335.png)

随后作者比较了Codex-S和Codex在pass@1和pass@100。实验结果表明，Codex-S在pass@1
平均比Codex高出6.5%，在pass@100上平均高出15.1%。

![Figure 10](/images/20250106184406.png)

作者同时还绘制了Codex-S-12B的不同样本选择启发式算法在Codex-S-12B上的性能与Codex-12B上的相同启发式算法进行比较。当按平均对数概率从1到100个样本进行排名时，平均比随机高11.6，这比Codex高出2%。

![Figure 10](/images/20250106184444.png)

---

## docstring生成 (Docstring Generation)

用docstring生成代码是可能的（合理的），因为代码通常紧跟在docstring之后，但是通过代码生成docstring是困难的。因此作者又开发了一个编写docstring的模型，可以描述生成代码的意图。使用前一节的训练问题描述，可以简单的创建一个训练数据集用于依据代码生成描述。

具体来说，对于每一个训练问题，作者集成了训练样本的函数声明、参考解决方案和docstring。和之前一样，他们训练docstring生成模型（Codex-D）最小化负对数似然。

对于生成代码，可以在HumanEval上测量pass@k作为基准测试，其中正确性由单元测试来定义。然而，没有类似的方法可以自动评估docstring样本.因此，他们通过手动来评估这些样本，如果文档字符串独特且准确的描述了代码，则认为是正确的。由于工作量大，作者只对每个问题生成的10个样本进行了评估，总共1640个问题，用temperature=0.8的Codex-D-12B。当模型简单的将代码体里的代码复制到docstring中时，作者认为这是错误的。最常见的错误是遗漏了重要的细节（如：结果保留两位小数）或者过度的要求函数名但是函数体的内容与函数名无关。

![Table 3](/images/20250106191104.png)

实验结果表明，Codex-D的通过率较低，但是与相同temperature下的Codex-S相当。由于自然语言语法没有代码语法严格，所以生成的文档可能更宽容。docstring的质量相对较低，因为程序员🧑‍💻不会花很多时间来写docstring。

最后，使用docstring模型，又有了一种从k个样本中选出一个样本的方法。不同与之前用最佳平均对数概率选择样本，可以最大化back-translation的P（GT docstring | 生成的样本），其中P是由Codex-D得出的。然而，如下图所示，通过back-translation进行排序的表现不如平均对数概率排序，尽管他优于随机排序。

---

## 局限性与不足 (Limitations)

1. Codex在训练上并不高效：作者认为没必要用Github上这么多的代码，实际上一个完成了学习的科班学生，预计能解决更大一部分的问题（比起Codex-12B）。
2. Codex可能会出现语法错误或未定义的代码，并且调用未定义或不在定义域内的函数、变量和属性。此外Codex在解析越来越长、更高层次或系统级的要求的时候会比较困难。为了具体说明这个问题，作者创建了一个由13个基本构建快组成的合成数据集。比如“将字符串转换为小写”或“移除每个字符串中第三个元素”（附录C里有详细的内容）。作者发现随着构建块链长度的增加，Codex的性能会呈指数下降。
  ![Figure 11](/images/20250106200331.png)
3. 和其他模态中的文本条件生成模型很难将属性绑定到对象上一样，Codex在将操作绑定到对象上也会出错，尤其是文档涉及的操作和变量很大的时候。如下面代码所示，Codex-12B并没有减少w变量，也没有返回四个变量的乘积。

```Python
def do_work(x, y, z, w):  
  """ Add 3 to y, then subtract 4 from both x and w. Return the product of the four numbers. """ 
  t=y+3
  u=x-4
  v=z*w 
  return v
```

---

## 核心贡献 (Key Contributions)

总结本文的主要贡献点：

1. 提出了一个评估大型语言模型生成代码能力的方法，包括pass@k指标和HumanEval数据集。
2. 作者提出了Codex，通过对GPT-3进行微调，使其在HumanEval数据集上的性能得到了进一步提升。
3. 作者提出了Codex-S，通过有监督微调Codex，使其在HumanEval数据集上的性能得到了进一步提升。
4. 作者提出了Codex-D，通过训练docstring生成模型，使其在HumanEval数据集上的性能得到了进一步提升。

---

## 参考文献 (References)

1. Kulal, Sumith, et al. "Spoc: Search-based pseudocode to code." Advances in Neural Information Processing Systems 32 (2019).
2. Hendrycks, Dan, et al. "Measuring coding challenge competence with apps." arXiv preprint arXiv:2105.09938 (2021).

## 备注 (Notes)

1. 论文提到一个现象（Introduction），12B的Codex可以解决28.8%的问题，而300M的Codex只能解决13.2的问题。这是为什么？
2. 论文提到了启发式（Heuristic），指的是什么
3. 优化后的pass@k有什么含义。
4. 这篇文章的Related work写了这个领域的发展，最早2011年就有人开始研究代码生成。这部分内容值得深入探索。

---

## 我的思考 (Personal Thoughts)

1. 本文与我研究的相关性：提出了一个评估大型语言模型生成代码能力的方法，对于代码生成的研究有一定的参考价值。
2. 是否有可以改进的地方：数据集的数量已经足够，但是数据集的质量可能还有待提高。作者提到了Codex在解析越来越长、更高层次或系统级的要求的时候会比较困难，这个问题也值得进一步研究。
3. 后续可能的研究方向：研究更多相关评估大语言模型生成代码能力的数据集，如MBPP、APPS等数据集。

---
