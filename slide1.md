---
# You can also start simply with 'default'
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://cover.sli.dev
# some information about your slides (markdown enabled)
title: "UniTS: A Unified Multi-Task Time Series Model"
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# apply unocss classes to the current slide
class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
---

# UniTS: A Unified Multi-Task Time Series Model

简要介绍时间序列领域的一个大一统模型

<div @click="$slidev.nav.next" class="mt-12 py-1" hover:bg="white op-10">
  进入正题 <carbon:arrow-right />
</div>

<div class="abs-br m-6 text-xl">
  <a href="https://github.com/mims-harvard/UniTS" target="_blank" class="slidev-icon-btn">
    <carbon:logo-github />
  </a>
</div>

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

---
transition: fade-out
---

# UNITS（A Unified Multi-Task Time Series Model）重点包括以下几个方面

UniTS 的表现优于竞争性的任务专门时间序列模型。代码和数据集可在 https://github.com/mims-harvard/UniTS 上找到。

- 🙋‍ 一、输入格式统一与适配
- 🦜 二、UNITS的整体模型结构及各部分作用
- 📚 三、Prompt Learning在UNITS中的实现及意义
- 🤹 四、样本标记(Sample Tokens)、提示标记(Prompt Tokens)、任务标记(Task Tokens)的来源、格式和实现方法
- 🗼五、GEN tower和CLS tower分别有什么用处？
- 🔢六、数据预处理
- 🏠七、任务执行流程
- 🫏八、DyLinear模块的运算机制
- ❓九、如何在新数据集上使用？

<!--
You can have `style` tag in markdown to override the style for the current page.
Learn more: https://sli.dev/features/slide-scope-style
-->

<style>
h1 {
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 100%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent;
  -moz-text-fill-color: transparent;
}
</style>

<!--
Here is another comment.
-->

---
transition: slide-up
level: 2
---

# 一、输入格式的统一与适配

时间序列数据往往存在多种复杂情况

- 长度不统一：不同行为主体、不同数据集的序列长度 t 可能差别巨大（例如有的序列是100个时间点，有的是1000个时间点）。
- 通道数（变量数）不统一：有的时间序列只有1个变量（例如单温度记录），有的可能有数十甚至数百个变量（例如多传感器记录）。
- 异质性：来自不同数据集的时间序列可能采样率、数据分布、变量类型都有差异。
UNITS如何统一不同长度和不同通道数的输入？

---

## UNITS如何统一不同长度和不同通道数的输入？

**1.线性投影（Embedding）与Patch切分：**

输入的时间序列数据为 $x ∈ R^{t×v}$，其中 $t$ 是时间长度，$v$ 是变量数。UNITS对输入先进行分片（patching），即将时间序列沿时间维度分割成固定大小为 $k$ 的不重叠片段，从而得到 $s = t/k$ 个片段（如果 $t$ 不是 $k$ 的整数倍，一般会通过截断或padding方式对齐）。每个片段是一个 $v×k$ 的小块，然后通过线性投影将这个片段映射为一个隐藏维度 $d$ 的表示向量。

通过这种方式，无论原始序列长度 $t$ 多长，最终都会被分割并投影到统一的表示空间中，使得模型在输入层面具有统一格式：$(s个时间片段) × (v个变量) × (d维隐藏表示)$。这样，长度和变量数的差异通过柔性切片和线性投影统一到同一个形状空间中（当然 $v$ 仍可变，但模型在自注意力层会处理可变长度和可变通道数）。

**2.动态线性层 (DyLinear) 与可变长度适配：**

由于输入序列长度可变，模型中引入 DyLinear 模块（动态线性操作符）。DyLinear通过在前馈层中使用对不同时间长度参数的加权来适应不同长度输入，以捕获不同时间尺度的特征。这让模型在中间特征提取阶段拥有柔性适配能力：即使输入序列长度变化，模型可通过动态参数插值对特征进行适配。

这种机制使得UNITS不会因为长度变化而需要重新设计模型结构，从而实现对异构时间序列的统一处理。

---

## UNITS如何统一不同长度和不同通道数的输入？

**3.注意力（时间维与变量维）：**

模型采用在时间维和变量维上的自注意力机制（可看作时序维和特征维的分解式注意力）。这样一来，无论 $v$ 有多大或者 $t$ 有多大，模型都可以通过注意力机制从行与列两个方向捕捉相关性，从而统一处理不同变量数和不同序列长度的数据。这种注意力让模型对变量数的变化同样具有适应性，因为注意力机制对序列长度与通道数不做特定要求，只需对输入张量进行相应的加权求和与映射。

>总结：通过上述方法，UNITS在输入层面将各种形态不一的时间序列（多样的 t 和 v）统一为可处理的标准化格式（patch embedding + attention + DyLinear），从而达到“输入格式统一”的目的。

---

# 二、UNITS的模型结构及各部分作用

整体结构概览：

- 编码 (Embedding)层：对原始时间序列分片后投影到隐藏空间。
- 提示标记 (Prompt Tokens)：在输入序列前注入提示标记，用于特定任务的上下文信息。
- 任务标记 (Task Tokens)：GEN标记和CLS标记用于确定任务类型和输出模式。
- 自注意力模块 (Time & Variable Attention)：在时间和变量两个维度实施注意力计算，以捕获时序和特征关联。
- DyLinear模块：在前馈层中根据输入长度动态调整参数，加强模型对多尺度序列的适应性。
- 门控模块(Gating)：在各层输出后使用门控机制对特征进行再平衡。
- 生成塔(Generative Tower) & 分类塔(Classification Tower)：共享参数的高层模块，用于最终的生成或分类任务输出。

---

## 各部分作用

- Embedding和Patch分割：将原始序列转化为可处理的固定维度表示，以适配Transformer结构。
  
- Prompt Tokens：在输入序列前后预置提示标记，让模型通过上下文提示了解具体任务、数据来源或特定领域信息。这种提示使得同一模型在冻结参数的情况下也能快速转向新的任务或数据集。
  
- Task Tokens：
  - GEN标记：表示需要进行生成类任务，比如预测未来值或填补缺失点。
  - CLS标记：表示需要执行分类类任务，例如判断序列属于哪个类别。
  >Task Tokens可以看作对模型的“指令提示”，告诉模型当前的输出需求是序列值重建（生成）还是类别判断（分类）。

---

## 各部分作用

- 时间与变量自注意力模块：通过对时间维、变量维分别进行注意力计算，提取全面的时序-特征交互关系，使模型能对任意形状的时序数据进行有效建模。
- DyLinear：动态适配不同长度的输入，为多尺度的时序数据提供统一的特征提取能力。
- 门控模块(Gating)：对多任务数据混合训练情况下的特征扰动进行调节。门控模块可以减少噪声特征或无关特征的影响，提升模型在统一框架下处理多种任务的稳定性。
- 生成塔 & 分类塔：在模型顶层，类似于分类模型中常见的分类头，以共享权重的方式存在。模型不需要为每种任务设计独立的头部结构，而是通过任务标记指示，动态使用生成塔或分类塔的能力，为多类型任务统一输出结果。


---

# 三、Prompt Learning的作用
通过prompt实现unseen的推断

Prompt Learning在UNITS中的意义：
在语言模型中，Prompt Learning通过在输入文本中添加提示，让已经训练好的模型在无需大量额外参数或微调的情况下进行新任务。UNITS将这一理念移植到时间序列领域。

通过在输入序列中添加提示标记（Prompt Tokens），UNITS可以在无需重新训练整个模型的情况下适配新任务、新数据集或新数据域。这些提示标记可以是预先学习得到的嵌入向量，能够：
- 为特定数据集提供上下文信息（例如数据所属领域或传感器类型）。
- 引导模型在推断时重点关注序列中与当前任务最相关的特征或时间段。
- 在多任务场景中，通过不同的提示标记快速切换任务要求。
- 因为提示标记是可学习的嵌入，其训练可以在模型参数冻结的状态下进行，使得对新任务的适配成本极低（类似于在LLM中添加少量Soft Prompts引导任务）。

---

# 三、Prompt Learning的作用
**Prompt Learning**在UNITS中扮演着关键角色，通过任务标记化和提示标记化，实现模型对不同任务的灵活适配。

任务上下文提供：

通过提示标记 $z_p$，模型获得当前任务或数据集的上下文信息。例如，不同数据集可能具有不同的特性，提示标记帮助模型理解这些特性。

任务指令引导：
任务标记（GEN或CLS）向模型传达当前任务的具体要求。比如，GEN标记指示生成任务，CLS标记指示分类任务。

冻结模型参数进行快速适配：
通过提示标记和任务标记，模型在参数冻结的情况下即可适配新任务，无需重新训练或大规模微调。

支持少样本学习：
提示学习使模型在少量样本的情况下快速适配新任务，通过提示标记提供有效的任务指引。

>总结：Prompt Learning使UNITS能够在统一的模型架构下高效处理多种任务，通过任务和提示标记提供灵活的任务适配机制，提升模型的通用性和适应性。
---

# 四、标记类型的来源、格式与实现
在UNITS中有三类标记：样本标记、提示标记、任务标记。

## 样本标记（Sample Tokens）
来源与格式：
样本标记直接来自原始时间序列数据 $x ∈ R^{t×v}$。通过将时间序列沿时间轴分割成长度为 $k$ 的片段，共得到 $s = t/k$ 个时间块。每个时间块大小为 $(v×k)$ 的片段，通过线性映射映射为 $(v×d)$ 的隐藏表示。这里的 $v$ 是变量数、$d$ 是模型隐藏维度。
经过这种处理后，我们得到的样本标记集合为 $z_x ∈ R^{s×v×d}$，即 $s$ 个时间片段，每个片段拥有 $v$ 个变量，每个变量映射到 $d$ 维的向量空间中。
实现：
在代码实现中，这通常是数据输入层的操作。例如：
```python
# x: [batch, t, v]
# 划分片段
x_patches = x.reshape(batch, s, k, v)  # 重塑为 s 个 (k×v)的block
# 对每个时间片段进行线性投影
z_x = linear_projection(x_patches)  # 得到 [batch, s, v, d]
```
同时还可以加入可学习的位置编码，为每个时间片段增加位置信息。

---

## 提示标记（Prompt Tokens）
来源与格式：
提示标记 $z_p ∈ R^{p×v×d}$ 是模型中可学习的参数，独立于输入数据，可视为模型的“记忆槽”或“上下文说明”。这些提示在训练中通过梯度更新学习得到。
在多任务场景中，不同数据集或任务可分配不同的提示标记，以使模型在处理特定数据时具备必要的上下文信息。
实现：
提示标记在实现上通常是模型内部定义的可训练参数矩阵。例如：
```python
self.prompt_tokens = nn.Parameter(torch.randn(p, v, d))
```
推理时将其与样本标记拼接起来输入模型：
```python
input_sequence = torch.cat([self.prompt_tokens, z_x], dim=0)
```

---

## 任务标记（Task Tokens）
来源与格式：
任务标记包括 GEN 和 CLS 两种类型。其中：
GEN 标记用于生成类任务，例如预测未来值、数据填补或异常检测。对于预测未来 $f$ 步，可复制 GEN 标记 $f$ 次作为生成的“查询”。
CLS 标记用于分类任务。若有 $C$ 个类别，可设定相应数量的 CLS 标记，以帮助模型聚合并输出分类结果。
实现：
类似提示标记，任务标记也是可学习参数，在模型中定义为嵌入向量：
```python
self.gen_token = nn.Parameter(torch.randn(1, v, d)) # 单个GEN标记，可重复使用f次
self.cls_tokens = nn.Parameter(torch.randn(C, v, d)) # C个CLS标记
```
在输入时将任务标记与样本标记、提示标记拼接起来得到最终的输入序列：
```python
# 对于预测任务，需要f个GEN标记:
gen_tokens = self.gen_token.repeat(f, 1, 1)
# 最终输入:
input_sequence = torch.cat([self.prompt_tokens, z_x, gen_tokens], dim=0)
```
通过上述标记体系，UNITS可以灵活地在同一个模型输入端添加不同的标记，从而实现对多样任务的统一本质化表示。Prompt Tokens为上下文，Sample Tokens为数据内容，Task Tokens为目标任务的信号，这三者结合，使得UNITS在处理任意时间序列任务时，都能以统一格式进行建模。

---

# 五、GEN tower和CLS tower分别有什么用处？
类似目标检测模型中的分类头

**GEN塔：**

用途：主要用于生成类任务，如时间序列预测、数据填充和异常检测。这些任务需要生成新的时间序列数据或预测未来的值。

工作机制：在生成任务中，GEN塔负责处理输入的样本标记和任务标记（GEN标记），通过自注意力机制和动态线性模块生成所需的输出。例如，在预测任务中，GEN塔会根据输入的历史数据生成未来的预测值。

**CLS塔：**

用途：主要用于分类任务，对整个时间序列样本进行分类。这类任务需要将输入的时间序列映射到预定义的类别中。

工作机制：在分类任务中，CLS塔处理样本标记和任务标记（CLS标记），通过自注意力机制和门控模块提取特征，并最终输出分类结果。CLS塔通过与类别嵌入的对比来确定样本所属的类别。
>总结：GEN塔和CLS塔通过共享模型的底层表示（如自注意力层和DyLinear模块），分别优化生成和分类任务，确保在统一的框架下高效处理多种任务类型。

---

# 六、数据预处理
原始时间序列数据：形式为 $x ∈ R^{t×v}$，其中 $t$ 是时间步数，$v$ 是变量数。

**样本标记化（Sample Tokenization）：**

- **切分**：将时间序列沿时间维度分割成固定大小为 $k$ 的不重叠片段，得到 $s = \frac{t}{k}$ 个片段。
- **线性投影**：每个片段 $x_i ∈ R^{k×v}$ 通过线性投影映射到隐藏维度 $d$，得到嵌入向量 $z_x ∈ R^{s×v×d}$。
- **位置嵌入**：为每个样本标记添加可学习的位置嵌入，提供时间位置信息。

**提示标记化（Prompt Tokenization）：**

提示标记 $z_p ∈ R^{p×v×d}$ 是可学习的嵌入，提供任务或数据集的上下文信息。

**任务标记化（Task Tokenization）：**

- **GEN标记**：用于生成任务，根据预测步数复制相应次数。例如，预测5步则复制5次GEN标记。
- **CLS标记**：用于分类任务，根据类别数目设置CLS标记数量。

---

# 七、任务执行流程
以常见任务为例：

**预测任务（Forecasting）：**

输入：样本标记 $z_x$，提示标记 $z_p$，GEN标记（复制5次或10次）。

模型运算：通过双向自注意力、DyLinear模块处理输入，GEN塔生成未来的时间步值。

输出：预测的时间序列值。

**分类任务（Classification）：**

输入：样本标记 $z_x$，提示标记 $z_p$，CLS标记（根据类别数目设置）。

模型运算：通过双向自注意力、DyLinear模块处理输入，CLS塔提取特征并与类别嵌入对比。

输出：分类结果（类别标签）。

---

# 八、DyLinear模块的运算机制
适应不同时间长度的关键

## 运算原理
DyLinear模块的核心思想是根据输入序列的长度动态调整线性层的权重参数，确保模型在处理不同长度序列时保持性能。具体步骤如下：

**输入特征**：假设输入特征为 $h ∈ R^{s×v×d}$，其中 $s$ 是切分后的片段数。

**动态权重生成**：
根据输入序列的长度 $s$，生成动态权重 $W_s$。
权重生成可以通过插值或基于长度的参数选择实现。例如，使用一组基础权重，通过插值计算出当前长度下的权重。

**线性变换**：
使用动态生成的权重 $W_s$ 对输入特征进行线性变换：$h' = h ⋅ W_s$
这里，$W_s$ 是一个与当前长度 $s$ 相关的权重矩阵，确保线性变换适应不同的序列长度。
---

### 举例说明
假设有两种输入长度：
- 输入A：序列长度 $s = 10$
- 输入B：序列长度 $s = 20$

DyLinear模块将根据 $s$ 动态生成不同的权重 $W_{10}$ 和 $W_{20}$，分别用于处理输入A和输入B。

**具体过程**：

- 输入A：
  - 序列长度 $s = 10$
  - 生成权重 $W_{10}$（通过插值或参数选择）
  - 进行线性变换 $h_A' = h_A ⋅ W_{10}$
- 输入B：
  - 序列长度 $s = 20$
  - 生成权重 $W_{20}$
  - 进行线性变换 $h_B' = h_B ⋅ W_{20}$

---

# 九、如何在新数据集上使用？
如何将提示标记（Prompt Tokens）与数据集关联，以及提示标记如何包含数据集的上下文信息？
**提示标记**在UNITS模型中用于提供任务或数据集的上下文信息，帮助模型理解当前处理的数据的特性和任务要求。

关联方式：
- 独立的提示标记：每个数据集或任务可以被分配一组独立的提示标记。这些提示标记是可学习的嵌入向量，在训练过程中专门学习以捕捉与特定数据集或任务相关的上下文信息。
- 数据集标识：在多数据集、多任务的训练环境中，可以为每个数据集定义一个唯一的标识符（例如，一个数据集ID）。这个标识符被用来选择对应的提示标记。该步骤在为数据集编写Config文件的时候完成。

包含上下文信息的机制：
可学习的嵌入：提示标记作为可学习的嵌入向量，在训练过程中学习到特定数据集的特性。这些嵌入向量可以捕捉数据集的领域知识、变量分布、采样率等特征。
任务适配：在处理不同数据集时，模型通过输入对应的提示标记，激活与该数据集相关的上下文信息，使模型能够根据提示标记调整其内部表示，以更好地适应当前数据集的特性。

生成提示标记的方法：
初始化：为每个数据集预先定义一组可学习的提示标记嵌入，通常在模型初始化时随机初始化。
训练过程：在训练过程中，通过反向传播更新提示标记，使其逐步学习到与特定数据集相关的有效上下文信息。
推理阶段：在推理时，根据输入数据所属的数据集，选择相应的提示标记并将其与样本标记和任务标记一起输入模型，从而提供必要的上下文信息。

---

# UniTS
```python
class Model(nn.Module):
    """
    UniTS: Building a Unified Time Series Model
    """

    def __init__(self, args, configs_list, pretrain=False):
        super().__init__()
    def tokenize(self, x, mask=None):
    def prepare_prompt(self, x, n_vars, prefix_prompt, task_prompt, task_prompt_num, task_name=None, mask=None):
    def mark2token(self, x_mark):
    def backbone(self, x, prefix_len, seq_len):
    def forecast(self, x, x_mark, task_id):
    def classification(self, x, x_mark, task_id):
    def imputation(self, x, x_mark, mask, task_id):
    def anomaly_detection(self, x, x_mark, task_id):
    def random_masking(self, x, min_mask_ratio, max_mask_ratio):
    def right_masking(self, x, min_mask_ratio, max_mask_ratio)
    def choose_masking(self, x, right_prob, min_mask_ratio, max_mask_ratio):
    def get_mask_seq(self, mask, seq_len):
    def pretraining(self, x, x_mark, task_id, enable_mask=False):
    def forward(self, x_enc, x_mark_enc, x_dec=None, x_mark_dec=None,
                mask=None, task_id=None, task_name=None, enable_mask=None):
```

---

# forword()
```python
 def forward(self, x_enc, x_mark_enc, x_dec=None, x_mark_dec=None,
                mask=None, task_id=None, task_name=None, enable_mask=None):
        if task_name == 'long_term_forecast' or task_name == 'short_term_forecast':
            dec_out = self.forecast(x_enc, x_mark_enc, task_id)
            return dec_out  # [B, L, D]
        if task_name == 'imputation':
            dec_out = self.imputation(
                x_enc, x_mark_enc, mask, task_id)
            return dec_out  # [B, L, D]
        if task_name == 'anomaly_detection':
            dec_out = self.anomaly_detection(x_enc, x_mark_enc, task_id)
            return dec_out  # [B, L, D]
        if task_name == 'classification':
            dec_out = self.classification(x_enc, x_mark_enc, task_id)
            return dec_out  # [B, N]
        if 'pretrain' in task_name:
            dec_out = self.pretraining(x_enc, x_mark_enc, task_id,
                                       enable_mask=enable_mask)
            return dec_out
        return None
```

---

# forecast()
```python
def forecast(self, x, x_mark, task_id):
        dataset_name = self.configs_list[task_id][1]['dataset']
        task_data_name = self.configs_list[task_id][0]
        prefix_prompt = self.prompt_tokens[dataset_name]
        task_prompt = self.mask_tokens[dataset_name]
        task_prompt_num = self.cls_nums[task_data_name][0]
        task_seq_num = self.cls_nums[task_data_name][1]
        real_seq_len = self.cls_nums[task_data_name][2]

        x, means, stdev, n_vars, _ = self.tokenize(x)
        x = self.prepare_prompt(
            x, n_vars, prefix_prompt, task_prompt, task_prompt_num, task_name='forecast')
        seq_token_len = x.shape[-2]-prefix_prompt.shape[2]
        x = self.backbone(x, prefix_prompt.shape[2], seq_token_len)
        x = self.forecast_head(
            x, real_seq_len, seq_token_len)
        x = x[:, -task_seq_num:]
        # De-Normalization from Non-stationary Transformer
        x = x * (stdev[:, 0, :].unsqueeze(1).repeat(1, x.shape[1], 1))
        x = x + (means[:, 0, :].unsqueeze(1).repeat(1, x.shape[1], 1))
        return x
```
---

# ForecastHead
```python
class ForecastHead(nn.Module):
    def __init__(self, d_model, patch_len, stride, pad, head_dropout=0, prefix_token_length=None):
        super().__init__()
        d_mid = d_model
        self.proj_in = nn.Linear(d_model, d_mid)
        self.mlp = Mlp(
            in_features=d_model,
            hidden_features=int(d_model * 4),
            act_layer=nn.GELU,
            drop=head_dropout,
        )
        self.proj_out = nn.Linear(d_model, patch_len)
        self.pad = pad
        self.patch_len = patch_len
        self.stride = stride
        self.pos_proj = DynamicLinear(
            in_features=128, out_features=128, fixed_in=prefix_token_length)

    def forward(self, x_full, pred_len, token_len):
        pass
        return x
```

---

# ForecastHead
```python
class ForecastHead(nn.Module):
    def __init__(self, d_model, patch_len, stride, pad, head_dropout=0, prefix_token_length=None):
        super().__init__()
        pass
    def forward(self, x_full, pred_len, token_len):
        x_full = self.proj_in(x_full)
        x_pred = x_full[:, :, -token_len:]
        x = x_full.transpose(-1, -2)
        x = self.pos_proj(x, token_len)
        x = x.transpose(-1, -2)
        x = x + x_pred
        x = self.mlp(x)
        x = self.proj_out(x)

        bs, n_vars = x.shape[0], x.shape[1]
        x = x.reshape(-1, x.shape[-2], x.shape[-1])
        x = x.permute(0, 2, 1)
        x = torch.nn.functional.fold(x, output_size=(
            pred_len, 1), kernel_size=(self.patch_len, 1), stride=(self.stride, 1))
        x = x.squeeze(dim=-1)
        x = x.reshape(bs, n_vars, -1)
        x = x.permute(0, 2, 1)
        return x
```
---

# backbone()
```python
def backbone(self, x, prefix_len, seq_len):
    attn_mask = None
    for block in self.blocks:
        x = block(x, prefix_seq_len=prefix_len +
                  seq_len, attn_mask=attn_mask)
    return x
```
```python
self.blocks = nn.ModuleList(
    [BasicBlock(dim=args.d_model, num_heads=args.n_heads, qkv_bias=False, qk_norm=False,
                mlp_ratio=8., proj_drop=args.dropout, attn_drop=0., drop_path=0.,
                init_values=None, prefix_token_length=args.prompt_num) for l in range(args.e_layers)]
)
```

---

# BasicBlock
```python
class BasicBlock(nn.Module):
    def __init__(
            self,
            dim,
            num_heads,
            mlp_ratio=8.,
            qkv_bias=False,
            qk_norm=False,
            proj_drop=0.,
            attn_drop=0.,
            init_values=None,
            drop_path=0.,
            act_layer=nn.GELU,
            norm_layer=nn.LayerNorm,
            prefix_token_length=0,
    ):
        super().__init__()
    def forward(self, x, prefix_seq_len, attn_mask):
        x = self.seq_att_block(x, attn_mask)
        x = self.var_att_block(x)
        x = self.dynamic_mlp(x, prefix_seq_len=prefix_seq_len)
        return x
```
