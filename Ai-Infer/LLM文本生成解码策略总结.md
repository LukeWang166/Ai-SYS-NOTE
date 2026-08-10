# 大模型文本生成解码策略全面总结

## 概述

大语言模型（LLM）训练完成后，如何从模型的概率分布中选择下一个 token，即**解码策略**（Decoding Strategy），是影响生成质量和多样性的关键。本文将系统性地介绍几种主流解码策略：贪心解码、随机采样、束搜索（Beam Search）、Top-k 采样、Top-p（核）采样以及 Temperature 采样，并说明它们如何组合使用。

---

## 一、基础解码方法

### 1. 贪心解码（Greedy Decoding）

**原理**：每一步直接选择概率最高的 token。

**优点**：简单高效，确定性强。

**缺点**：
- 生成的文本容易陷入单调和重复循环
- 局部最优不保证全局最优
- 缺乏多样性

> 可以理解为 k=1 的 Top-k 采样。

### 2. 随机采样（Random Sampling）

**原理**：按照概率分布随机选择下一个 token。

**优点**：增加生成多样性。

**缺点**：可能导致生成不连贯、无意义的内容。

### 3. 穷举搜索（Exhaustive Search）

**原理**：遍历所有可能的输出序列，选择条件概率最大的全局最优解。

**优点**：保证全局最优。

**缺点**：计算复杂度指数级增长，输出词典稍大即不可用。

> 例如输出词典大小为 3，序列长度为 3 时就有 3³=27 种组合；真实场景下词典大小通常为数万，完全不可行。

---

## 二、束搜索（Beam Search）

束搜索是介于贪心搜索和穷举搜索之间的**折中方案**，通过超参数 `beam size`（束宽，记为 k）来控制搜索空间。

### 工作原理

1. **第一个时间步**：选取条件概率最大的 k 个词作为候选
2. **后续每个时间步**：基于上一步的 k 个候选，从所有可能的扩展组合中选出条件概率最大的 k 个
3. **始终保持 k 个候选序列**
4. **最终**：从 k 个候选中选出概率最高的作为输出

### 示例（beam size = 2）

假设翻译任务 "我恨你" → "I H U"，词典为 {"I", "H", "U"}：

- **步1**：概率 top 2 为 I(0.6)、H(0.3) → 候选：I, H
- **步2**：从 II/IH/IU/HI/HH/HU 中选 top 2 → 候选：IH(0.42)、HI(0.36)
- **步3**：从 IHI/IHH/IHU/HII/HIH/HIU 中选 top 2 → 候选：IHH、IHU
- **输出**：从 IHH 和 IHU 中选最优 → IHU

### 特点

- ✅ 比贪心搜索空间更大，效果通常更好
- ❌ 不保证全局最优
- ❌ 可能过于保守、不自然
- 📌 beam size = 1 时退化为贪心搜索

---

## 三、Top-k 采样

### 原理

在每一步，**只从概率最高的 k 个 token 中进行随机采样**，忽略低概率 token。

```
例如 k=3：只从女孩(0.664)、鞋子(0.199)、大象(0.105) 中选择
忽略：西瓜(0.032) 等
```

### 代码实现

```python
import torch

class TopKSampler:
    def __init__(self, k: int, sampler):
        self.k = k
        self.sampler = sampler

    def __call__(self, logits: torch.Tensor):
        zeros = logits.new_ones(logits.shape) * float('-inf')
        values, indices = torch.topk(logits, self.k, dim=-1)
        zeros.scatter_(-1, indices, values)
        return self.sampler(zeros)
```

### 优点

- ✅ 避免采样到不合适或不相关的 token
- ✅ 保留一定随机性，增加多样性
- ✅ 可与其他策略（Temperature、Repetition Penalty 等）结合

### 缺点

- ❌ k 值难以确定最优值
- ❌ 可能忽略低概率但有创意或符合语境的 token
- ❌ 仅考虑概率排名，不考虑概率分布的形态

---

## 四、Top-p 采样（核采样 / Nucleus Sampling）

### 原理

动态选择 token 集合：**从累积概率超过阈值 p 的最小 token 集合中进行采样**，而非固定 k 个。

```
例如 p=0.8：
女孩(0.664) + 鞋子(0.199) = 0.863 > 0.8
→ 从 {女孩, 鞋子} 中选择
大象(0.105) 被排除
```

### 代码实现

```python
import torch
from torch import nn

class NucleusSampler:
    def __init__(self, p: float, sampler):
        self.p = p
        self.sampler = sampler
        self.softmax = nn.Softmax(dim=-1)

    def __call__(self, logits: torch.Tensor):
        probs = self.softmax(logits)
        sorted_probs, indices = torch.sort(probs, dim=-1, descending=True)
        cum_sum_probs = torch.cumsum(sorted_probs, dim=-1)
        nucleus = cum_sum_probs < self.p
        nucleus = torch.cat([nucleus.new_ones(nucleus.shape[:-1] + (1,)), nucleus[..., :-1]], dim=-1)
        sorted_log_probs = torch.log(sorted_probs)
        sorted_log_probs[~nucleus] = float('-inf')
        sampled_sorted_indexes = self.sampler(sorted_log_probs)
        res = indices.gather(-1, sampled_sorted_indexes.unsqueeze(-1))
        return res.squeeze(-1)
```

### 与 Top-k 的对比

| 特性 | Top-k | Top-p |
|------|-------|-------|
| 候选集大小 | 固定 k 个 | 动态调整 |
| 分布尖锐时 | 可能包含不合理的 token | 自动缩小候选集 |
| 分布平坦时 | 可能遗漏合理 token | 自动扩大候选集 |
| k/p 值选择 | 较难确定 | 更直观（累积概率阈值） |

> Top-p 的核心优势在于**动态自适应**：根据概率分布的形状自动调整候选 token 数量，避免 Top-k 中 k 值固定的缺陷。

---

## 五、Temperature 采样（温度采样）

### 原理

受统计热力学启发，通过将 logits 除以温度 T 来调整概率分布的"陡峭程度"：

```
softmax(logits / T)
```

### 效果

| 温度 T | 效果 | 应用场景 |
|--------|------|----------|
| T → 0 | 成为 argmax（贪心解码） | 确定性任务（翻译、摘要） |
| T = 1 | 原始概率分布 | 默认 |
| T > 1 | 分布更均匀，更多样 | 创意写作、对话 |
| T → ∞ | 均匀采样 | 最大随机性 |

### 直观理解

- **低温度**（如 T=0.5）：概率分布更尖锐，模型更"自信"，倾向于选择高概率 token
- **高温度**（如 T=1.5）：概率分布更平坦，模型更"不确定"，低概率 token 也有机会被选中

> 注意：温度并不直接控制"创造力"，而是调整概率分布的**确定性程度**。

### 代码实现

```python
import torch
from torch.distributions import Categorical

class TemperatureSampler:
    def __init__(self, temperature: float = 1.0):
        self.temperature = temperature

    def __call__(self, logits: torch.Tensor):
        dist = Categorical(logits=logits / self.temperature)
        return dist.sample()
```

---

## 六、联合采样策略

在实际的大模型中，**Top-k、Top-p、Temperature 通常联合使用**，典型执行顺序为：

```
Top-k → Top-p → Temperature → 采样
```

### 完整流程示例

假设原始概率分布：女孩 0.664、鞋子 0.199、大象 0.105、西瓜 0.032

1. **Top-k（k=3）**：保留女孩、鞋子、大象
2. **Top-p（p=0.8）**：累积概率 0.664+0.199=0.863>0.8，排除大象
3. **Temperature（T=0.7）**：重新归一化为 女孩 0.660、鞋子 0.340
4. **采样**：从以上分布中随机采样

### 常见参数配置参考

```json
{
  "top_k": 10,
  "temperature": 0.95,
  "num_beams": 1,
  "top_p": 0.8,
  "repetition_penalty": 1.5,
  "max_tokens": 30000
}
```

---

## 七、策略选择指南

| 任务类型 | 推荐策略 | 说明 |
|----------|----------|------|
| 机器翻译 | Beam Search + 低 Temperature | 需要准确、确定性的输出 |
| 文本摘要 | Top-p (p=0.9) + 低 Temperature | 兼顾准确性和流畅性 |
| 对话生成 | Top-p (p=0.9-0.95) + Top-k (k=40-50) + T=0.7-0.9 | 多样化且自然 |
| 创意写作 | 高 Temperature (T=0.9-1.2) + Top-p (p=0.95) | 鼓励创造性 |
| 代码生成 | 低 Temperature (T=0.2-0.4) + Top-p (p=0.9) | 需要精确性 |
| 问答系统 | Top-p (p=0.8) + 中等 Temperature | 平衡准确与多样 |

---

## 八、总结

- **贪心解码**：快但单调，适合确定性任务
- **Beam Search**：在搜索空间和质量之间折中，适合翻译等任务
- **Top-k 采样**：固定候选集大小，简单但 k 值难选
- **Top-p 采样**：动态候选集，自适应概率分布，比 Top-k 更灵活
- **Temperature**：调整分布"锐度"，控制确定性与多样性之间的平衡
- **联合使用**：综合各策略优点，按 Top-k → Top-p → Temperature 顺序执行

---

*本文综合整理自知乎专栏文章：*
- *《大模型文本生成——解码策略（Top-k & Top-p & Temperature）》*
- *《如何通俗的理解 beam search？》*
