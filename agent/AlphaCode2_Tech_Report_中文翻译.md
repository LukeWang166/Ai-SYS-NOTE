# AlphaCode 2 技术报告

> **原文**: AlphaCode 2 Technical Report
> **作者**: AlphaCode Team, Google DeepMind（通讯作者：remileblond@google.com）
> **日期**: 2023-12-06
> **链接**: https://storage.googleapis.com/deepmind-media/AlphaCode2/AlphaCode2_Tech_Report.pdf
> **版权**: © 2023 Google DeepMind. All rights reserved
> **译者说明**: 本文为完整全文翻译，图注按图意译出并补充说明；带 *（译者注）* 标记的段落为译者补充的背景说明，非原文内容。

---

## 摘要

AlphaCode（Li et al., 2022）是首个在**竞技编程（competitive programming）**任务上达到参赛者中位数水平的 AI 系统——这是一项涉及高等数学、逻辑与计算机科学的困难推理任务。本文提出 **AlphaCode 2**：一个全新且大幅增强的系统，性能获得巨大提升，由 **Gemini**（Gemini Team, Google, 2023）驱动。AlphaCode 2 的核心在于**强大的语言模型**与**定制化的搜索与重排序（search and reranking）机制**的结合。在与原版 AlphaCode 相同的平台上进行评估时，我们发现 AlphaCode 2 多解出了 **1.7 倍**的题目，表现优于 **85%** 的参赛者。

---

## 引言

竞技编程是检验编码能力的终极试金石之一。参赛者需要在有限时间内编写代码来解决复杂问题，而这些问题要求批判性思维、逻辑能力，以及对算法、编程和自然语言的理解。因此，它是评测**高级推理与问题求解能力**的绝佳基准。

AlphaCode（Li et al., 2022）是首个在该任务上达到竞赛水平的 AI 系统。它的后继者 **AlphaCode 2** 在一个大幅改进的系统中，将多个基于 Gemini（Gemini Team, Google, 2023）的模型作为组件加以利用。在 Codeforces 平台——竞技编程的主要阵地——上评估时，AlphaCode 2 在**10 次尝试内**解决了 **43%** 的题目，接近原版 AlphaCode（25%）的两倍。其前身的表现相当于参赛者的中位数水平，而我们估计 AlphaCode 2 平均可以达到 **第 85 百分位**。

将 Gemini 作为 AlphaCode 2 所有组件的基础模型，是达成这一性能水平的关键。AlphaCode 2 的成功印证了 Gemini 的**灵活性与可适配性**：我们能够对它进行微调，并针对包括**代码生成**和**代码重排序**在内的多个不同任务优化性能。

---

## 整体系统

AlphaCode 2 依赖强大的大语言模型，并结合一套为竞技编程量身定制的**高级搜索与重排序机制**。如图 1 所示，其主要组件包括：

- **一组策略模型（policy models）**，为每道题生成代码样本；
- **采样机制（sampling mechanism）**，鼓励生成高度多样的代码样本，以便在可能的程序空间中充分搜索；
- **过滤机制（filtering mechanism）**，剔除不符合题目描述的代码样本；
- **聚类算法（clustering algorithm）**，将语义相似的代码样本归为一组，从而避免冗余；
- **打分模型（scoring model）**，用于从 10 个最大的代码样本簇中各挑出最佳候选。

> **图 1 | AlphaCode 2 系统高层总览。** 数据流为：**题目（Problem）** → **AlphaCode 2 模型（生成 C++ 代码样本）** → **大规模采样（Massive sampling）** → **执行与过滤（Execution & filtering）** → **聚类（Clustering）** → **重排序（Reranking）** → 向 Codeforces **提交（Submit!）**。
> 训练侧：以 **Gemini Pro** 为基础做**微调（Fine-Tuning）**；使用 **CodeContests v2** 数据集（题目 / 题解），是**更高质量的数据集**；产出**中间态 AlphaCode 2 模型（Intermediate AlphaCode 2 models）**，再经**采样与评估（Sampling & Evaluation）**得到最终 **AlphaCode 2 模型**；另有独立的**打分模型（Scoring model）**。

### 策略模型与微调

我们的起点是 **Gemini Pro** 模型（Gemini Team, Google, 2023），在其之上我们以 **GOLD**（Pang and He, 2020）作为训练目标，连续进行**两轮微调**。

第一轮，我们在 **CodeContests 数据集**的更新版本上微调（该版本包含更多题目、更多题解，以及验证集上更高质量的人工筛选测试）。这个数据集包含约 **1.5 万道题**和 **3000 万条人类代码样本**。我们通过调整超参数生成了多个微调模型，最终得到**一族（a family of）微调模型**。

第二轮，我们在另一个**更高质量的数据集**上再做若干步额外微调。

> *（译者注）GOLD = Generation by Off-policy Learning from Demonstrations，即"从示范中做离策略学习"的文本生成目标。相比逐 token 最大似然，GOLD 引入了重要性权重来缓解训练与推理时分布不一致的问题，在 AlphaCode 系列中被用于提升采样的多样性与质量。*

采用**一族策略而非单一策略**，是为了最大化多样性——而这正是攻克难题的关键。

### 采样

我们的采样方式与 AlphaCode 接近。我们对每道题最多生成 **100 万个**代码样本，并为每个样本使用**随机化的温度参数（temperature）**以鼓励多样性。我们还会对 prompt 中加入的特定元数据（如**题目难度评级**与其**类别标签**）做随机化处理。

我们将采样预算**平均分配**给这一族微调模型。AlphaCode 使用 Python 和 C++ 两种语言采样，而 AlphaCode 2 **只使用 C++ 样本**，因为我们发现其质量更高。

> *（译者注）关于目标语言的选择：AlphaCode 2 通过消融发现 C++ 样本质量优于 Python，故放弃多语言采样，把全部预算集中在单语言上以换取更高质量的候选池。*

大规模采样使我们能够**彻底搜索模型分布**、生成高度多样的代码样本，从而最大化"至少产出一些正确样本"的概率。鉴于样本量之大，**过滤与重排序**对系统整体性能至关重要——因为每道题我们**最多只提交 10 个**代码样本。

### 过滤

每道竞技编程题都至少包含一个**公开的输入/输出测试（public test）**，用于说明代码应有的行为。我们把每个代码样本在对应的测试输入上执行，过滤掉所有**未能产生预期输出**（因而不可能正确）的样本，以及不到 **5%** 的**无法编译**的样本。平均而言，这一过滤步骤会剔除约 **95%** 的样本。

### 聚类

过滤之后，每道题平均仍有约 **5 万个**候选，而我们最多只能提交 10 个。为进一步削减候选，我们依据样本的**运行时行为**对其进行聚合：与 AlphaCode 一样，我们训练一个**单独的模型**为每道题生成**新的测试输入**，然后在剩余样本上执行这些新输入。所产生的输出构成一个**签名（signature）**，我们用它把行为相似的代码样本归入同一**簇（cluster）**。随后我们按簇的**基数（大小）**排序，只保留**最大的 10 个簇**。

聚类的意义在于**避免冗余**：同一簇内的代码样本行为相似，所以我们每个簇只需向在线评测系统（online judge）提交**一个**样本，即可获得该行为模式下的最佳结果。

### 打分模型

我们微调了第二个 Gemini Pro 模型，用于为代码样本赋予一个 **0 到 1 之间**的**估计正确性得分**。利用这个打分模型，我们为剩余各簇中的每个代码样本计算得分；然后基于预测得分，从每个簇中选出**最佳候选样本**，从而构成最终 **10 个提交**的列表。

---

## 评估

我们在 **Codeforces** 上评估 AlphaCode 2——与原版 AlphaCode 相同的平台。我们选取了 **12 场最近的比赛**，均拥有 **8000 名以上**参赛者，来自 **div. 2** 或难度更高的 **"1+2"** 分区，总计 **77 道题**。对每道题，我们采样 **100 万个**候选，并按照上述流程选取与排序、最多提交 **10 个**题解，直到找到一个正确解，或候选耗尽为止。

我们发现 AlphaCode 2 解决了这些竞赛题中的 **43%**，相比此前创纪录的 AlphaCode 系统（解决 **25%**）提升了近 **2 倍**。将其映射到竞赛排名上，我们估计 AlphaCode 2 平均处于 **第 85 百分位**——即其表现优于 **85%** 的参赛者，排名恰好介于 Codeforces 的 **"Expert（专家）"** 与 **"Candidate Master（候选大师）"** 两个等级之间。

> **图 2 | AlphaCode 2 的估计排名。** 横轴将人类参赛者的 Codeforces 得分**归一化到 [0, 1]**（每场除以该场人类最高分），纵轴为"得分低于该值的参赛者百分比"，在评估所用的 12 场比赛上取平均。曲线下方区域对应人类选手分布，图中标注了 **87%** 与 **46%** 两个参考点；我们再计算 AlphaCode 2 的平均归一化得分，并将其标注在排名轴上，使其稳居 **85 百分位之上**。该排名还计入了一项**模拟的时间惩罚**：假设 AlphaCode 2 按难度递增顺序解题，并在 **2 小时**节点完成最后一题的采样。

这相比 AlphaCode 是一次显著进步——后者仅优于约 **46%** 的参赛者。在 AlphaCode 2 表现最好的两场比赛中，它**超越了 99.5% 以上**的参赛者！

我们还评估了**增加每题样本量**所带来的影响。与 AlphaCode 的情况一致，我们发现性能随样本数增加大致呈**对数线性（log-linear）**提升。AlphaCode 2 只需约 **100 个样本**即可达到 AlphaCode 使用 **100 万个样本**时的性能水平，**样本效率提升超过 10000 倍**。

> **图 3 | 在 12 场近期比赛上的解题率，作为每题样本数的函数。** 横轴为**每题采样预算**（对数刻度，$10^0$ 到 $10^6$），纵轴为**解题率（Solve rate）**，从 0% 到 40%。曲线显示 AlphaCode 2 的解题率随采样量对数线性增长；作为对照，图中标出 AlphaCode 在 **100 万样本**时的水平——AlphaCode 2 在极小的采样预算下即已越过该水平线。

---

## 讨论与结论

竞技编程与其他编码任务**非常不同**，后者通常遵循**命令式范式（imperative paradigm）**：用户给出清晰指令，模型输出所需代码。相比之下，竞技编程问题是**开放式的（open-ended）**。要解决它们，在编写代码实现之前，需要先**理解、分析和推理**问题本身，而这涉及高等数学与计算机科学的概念。

这解释了一般通用 AI 系统为何在该基准上表现不佳。AlphaCode 2 在竞技编程比赛中的成功，代表了在这项**极其困难的推理任务**上令人印象深刻的一次**跨越式（step change）**进步。

将 **Gemini Pro 作为基础模型**，使系统两个关键组件的性能均获得显著提升：生成代码样本的**策略模型**，以及用于从中选出最优者的**打分模型**。我们能够把 Gemini 微调到在这两个差异极大的任务上都达到高水平，这充分说明了它**惊人的灵活性**。我们推测，若改用 **Gemini Ultra** 作为基础模型——它具备更强的编码与推理能力——AlphaCode 2 的整体方案有望进一步提升。

尽管 AlphaCode 2 取得了令人瞩目的成果，但距离看到能够**可靠达到顶尖人类程序员水准**的系统，仍有大量工作要做。我们的系统需要**大量的试错**，且**规模化运行的成本仍然过高**。此外，它高度依赖能否**过滤掉明显错误的代码样本**。

这为**系统与人类程序员之间的正向互动**打开了大门——人类可以指定额外的过滤条件。在这种 **AlphaCode 2 + 人类**的协作模式下，我们的得分**超过了第 90 百分位**！我们希望这种**交互式编程**会成为编程的未来：程序员把能力极强的 AI 模型作为**协作工具**，帮助自己推理问题、提出代码设计并协助实现。我们正努力把 AlphaCode 2 的独特能力融入我们的基础 Gemini 模型，作为让这一新编程范式惠及所有人的第一步。

---

## 参考文献

1. Gemini Team, Google. **Gemini: A Family of Highly Capable Multimodal Models.** 2023. URL: https://storage.googleapis.com/deepmind-media/gemini/gemini_1_report.pdf
2. Leblond et al. **AlphaCode 2 Technical Report.** 2023. URL: https://storage.googleapis.com/deepmind-media/AlphaCode2/AlphaCode2_Tech_Report.pdf
3. Y. Li, D. Choi, J. Chung, N. Kushman, J. Schrittwieser, R. Leblond, T. Eccles, J. Keeling, F. Gimeno, A. D. Lago, T. Hubert, P. Choy, C. de Masson d'Autume, I. Babuschkin, X. Chen, P.-S. Huang, J. Welbl, S. Gowal, A. Cherepanov, J. Molloy, D. J. Mankowitz, E. S. Robson, P. Kohli, N. de Freitas, K. Kavukcuoglu, and O. Vinyals. **Competition-level code generation with AlphaCode.** *Science*, 2022. URL: https://www.science.org/doi/abs/10.1126/science.abq1158
4. R. Y. Pang and H. He. **Text generation by learning from demonstrations.** *arXiv preprint* arXiv:2009.07839, 2020. URL: https://arxiv.org/pdf/2009.07839.pdf

### 引用本文

本文是由 Google DeepMind 提供的免费开放获取论文。最终版本已在线发表。引用格式：

> Leblond et al. *AlphaCode 2 Technical Report.* 2023. URL: https://storage.googleapis.com/deepmind-media/AlphaCode2/AlphaCode2_Tech_Report.pdf

---

## 贡献与致谢

**负责人（Leads）**
- Felix Gimeno，技术负责人
- Florent Altché，技术负责人
- Rémi Leblond，负责人

**核心贡献者（Core contributors）**
- Alaa Saade、Anton Ruddock、Corentin Tallec、George Powell、Jean-Bastien Grill、Maciej Mikuła、Matthias Lochbrunner、Michael Mathieu、Paul Caron

**贡献者（Contributors）**
- Disha Shrivastava、Eric Mitchell、Grace Margand、Jacob Kelly、Jakub Sygnowski、James Keeling、Junyoung Chung、Nate Kushman、Nikolay Savinov、Petko Yotov、Tobenna Peter Igwe、Wojciech Stokowiec、Yujia Li

**战略顾问（Strategic Advisors）**
- Koray Kavukcuoglu、Nando de Freitas、Oriol Vinyals、Pushmeet Kohli、Satinder Baveja

> 在每一类角色内部，作者按字母顺序排列；排序不代表贡献程度的差异。

我们由衷感谢 Google DeepMind 的同事们给予的支持，尤其是 **Gemini 团队**、**原 AlphaCode 团队**和**硬件团队**——没有他们，AlphaCode 2 就不可能实现。

我们感谢 **Eleanor Tomlison** 和 **Gaby Pearl** 为我们绘制了非常精美的 AlphaCode 2 系统总览图。

我们还要感谢我们的领导层、审稿人和同事对本文提出的宝贵讨论与反馈：Aakanksha Chowdhery、Aliya Ahmad、Antonia Paterson、Arielle Bier、Ben Bariach、Dawn Bloxwich、Demis Hassabis、Eli Collins、Emily Hossellman、Fred Alcober、Jeff Dean、Joel Moss、Jon Small、Koray Kavukcuoglu、Lily Lin、Megha Goel、Oriol Vinyals、Petar Veličković、Rebecca Bland、Sanah Choudhry 以及 Tessa Lueth。

AlphaCode 2 演示中所用的题目（展示于 Gemini 主页）已获得 **Codeforces** 的许可——该平台定期举办来自世界各地参赛者的比赛，供他们检验自己的编码能力。该题来自 **CodeTON Round 4** 比赛。

---

## 术语与背景速览（译者整理）

| 术语 | 含义 |
| --- | --- |
| Competitive programming（竞技编程） | 在限时内解决算法竞赛题的编程形式，题目开放式、难度高，代表平台为 Codeforces |
| Policy model（策略模型） | 生成代码样本的模型；AlphaCode 2 用一族微调模型而非单个模型以提升多样性 |
| GOLD | 从示范中学习的离策略文本生成目标（Pang & He, 2020），替代逐 token 极大似然 |
| CodeContests v2 | 训练数据集，约 1.5 万题、3000 万条人类代码样本，含人工筛选的高质量测试 |
| Massive sampling（大规模采样） | 每题最多采样 100 万样本，配合随机温度与随机化元数据（难度、标签）以鼓励多样性 |
| Filtering（过滤） | 用公开测试用例执行样本，剔除不符预期输出者与无法编译者，平均去掉约 95% |
| Clustering（聚类） | 用生成的额外测试输入构造"输出签名"，把行为相似的样本聚簇，仅保留最大的 10 簇 |
| Scoring model（打分模型） | 第二个微调 Gemini Pro，为样本预测 0–1 的正确性得分，用于簇内择优 |
| Percentile（百分位） | 表示击败了多少比例的参赛者；AlphaCode 2 平均约第 85 百分位，与人协作时超第 90 百分位 |

**核心数字速记**

- 解题率：**43%**（10 次尝试内），原版 AlphaCode 为 25%，提升约 **1.7–2 倍**
- 排名：平均 **第 85 百分位**（介于 Expert 与 Candidate Master 之间），最佳两场比赛超越 **99.5%** 参赛者
- 样本效率：约 **100 样本**即达 AlphaCode 100 万样本水平，**>10000×** 提升
- 消融：过滤平均剔除 **~95%** 样本；每题平均剩余 **~5 万** 候选；最终只提交 **10 个**
- 人机协作：**AlphaCode 2 + 人类**指定额外过滤条件后，得分 **>第 90 百分位**
