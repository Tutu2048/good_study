#### LLM

> LLM代表大型语言模型（Large Language Model）

大模型的**大**：是指其通用型

核心思想：迁移学习--例如：将动物的CV识别能力，迁移到人脸识别上

大语言模型的**任务**：其实就是**Next Token Prediction** ，任务都可以转化成这个任务

**Training** 基础思想： 若给出的Token与标准答案不同就更改参数

**Inference** ：从各个文本中给出各个token 列出概率分布，取大的那个

##### Training Methods

- **Self-Supervised Pre-Training** 阶段：学习的是未标注的数据，可以流畅的给出答案，但仅仅是**续写**，它并不知道这些知识怎么使用，来服务人类

  - **Contrastive learning** 对比学习：常用的自监督学习，Maximize the distance

    between positive and negative examples

- **Supervised Fine-Tuning （SFT）** 阶段 ：学习的是标注了的数据，为了让大模型能够理解问题、任务、指令，运用所学的知识更好地解决人的问题。

- **Human Feedback（RLHF）**  阶段：不再规定回答的格式增加模型灵活性，因为答案的描述并不是唯一的，SFT阶段并不能很好的解决这个问题，在此阶段给出人类偏好 ，让大模型效果更好

##### Emergent Abilities

- **In-context Learning**：在模型不变的情况下，给出一定样例 ，大模型就能找到其中的规律，并给出答案。
- **Instruction Following**：面对复杂指令，依旧可以理解（2022⬆️）
- **Chain-of-Thought**：不仅仅是映射规律了，而是可以给出推理过程了

---

#### Basics of Neural Networks

##### Structure

- **input layer** 

  - 向量输入✅
  - 标量输入（少）

- **Multiple hidden layers**

  - 多层的线形神经网络会塌缩成单层的神经网络，所以需要⬇️
  - 非线性激活函数f--enable neural nets to represent more complicated features
    - Sigmoid
    - Tanh
    - ReLU

  注：f是对向量乘积的每一个值生效的

- **output layer**

  输出模式：

  - linear output
  - sigmoid  解决2分类问题
  - softmax

##### RNN

> Recurrent Neural Networks 循环神经网络

顺序记忆：人脑在学习时，存在顺序的惯性，比如字母表ABCD... 我们可以很迅速的读出来，但是当字母表逆置后就存在一定的难度。

RNN利用顺序记忆的特性进行大模型训练。

###### RNN Cell

**时间步展开**：通过隐藏状态h~t~ 实现序列记忆
$$
\
h_t = f(W_{xh}x_t + W_{hh}h_{t-1} + b)
\
$$

**权重矩阵 W~xh~（Input-to-Hidden Weights）**

- **作用**：将当前输入 xt*x**t* 映射到隐藏空间，决定输入数据对当前状态的影响权重

**权重矩阵 W~hh~（Hidden-to-Hidden Weights）**

- **维度**：**W~hh~∈R^h×h^**（隐藏状态维度平方）
- **作用**：控制历史信息 ht−1*h**t*−1 如何传递到当前状态，捕捉时间序列的长期依赖

**偏置项 b（Bias）**

- **作用**：为每个隐藏神经元提供基线激活，增强模型表达能力

###### 优劣

- 优势：
  - 模型的复用
  - **量化价值**：天然适配金融时间序列（如价格、成交量、订单流）的时序依赖性建模
- 劣势：
  - 梯度消失 or 爆炸

###### Better RNNs 

**GRU（门控循环单元）**



**LSTM（长短期记忆网络）**

引入Cell state Ct值

**遗忘门 Forget Gate**：ft 计算出后若为0 ，则表示丢弃之前的关系

**输入门 Input Gate**：it 

**输出门 Output Gate**：ot

> Tips: 两个变体的思路是：保存周围几个单元信息，来解决长距离记忆问题



**双向RNN**

h~t~ 不仅仅受过去的的影响也受未来影响



##### CNN

> 诞生于计算机视觉
>
> 擅长**局部的特征**、可并行处理
>
> 一个矩阵，对着大的信息矩阵进行卷积

**结构**：

- 输入层 ：将输入的内容转换为变量表示
- 卷积层：通过卷积 获取特征向量
- 最大池化层：特征进一步提取，一般取最大
- 非线性层：对特征进行激活，转出最后输出



###### 注意力机制

> 为什么需要注意力机制？
>
> 因为传统 Encoder-Decoder 模型将整个输入序列压缩成一个**固定长度的向量**（上下文向量），而一个固定长度向量是否能存储足够大的信息呢？答案是不能，所以会导致长序列信息丢失。

优化后的RNN步骤

1. input vector
2. 计算注意力分数
3. 得到注意力分布
4. 得到与分布长度相等的向量
5. 顺序执行计算output

> [!NOTE]
>
> CNN不再关注前面的所有隐层 ，而是关注局部的几个 从而引出**N-gram** ：n个参数，对于自然语言模型而言，就是关注前面的n个单词 



###### Comparison between CNN and RNN

|                 | **CNNs**                                         | **RNNs**                                |
| --------------- | ------------------------------------------------ | --------------------------------------- |
| Advantages      | Extracting local and position-invariant features | Modeling long-range context dependency  |
| Parameters      | Fewer parameters                                 | More parameters                         |
| Parallelization | Better parallelization within sentences          | Cannot be parallelized within sentences |



##### transformer

> Architecture:encoder-decoder
>
> Input: byte pair encoding(BPE)文本切分方式  +positiongal encoding 位置标编码
>
> encodingModel: stack of severalencoder/decoder blocks
>
> Output: probability of the translated word
>
> Loss function: standard crossentropy loss over a softmax layer

> [!NOTE]
>
> **Transformer 其实就是将RNN处理序列的方式替换成了attention机制**
>
> attention机制，增加了记忆力的容量、缩小了链路

###### byte pair encoding(BPE)

从字母切分，记录出现频率，合并高频的相邻字符对，生词字词单元，重复以上操作，行成最终词表（最终词表长度是自定义的长度，也就是词表长度合并到预定值即可）。

BPE作用：

1. 处理未登录词

- 对未登录词 `"ChatGPT"`，BPE 可能切分为 `["Chat", "G", "PT"]`，即使 `"ChatGPT"` 不在词表中，模型仍能理解其部分含义。

**2. 多语言支持**

- 混合多种语言的语料训练 BPE，可生成跨语言的子词单元。
  *例如：* 英语 `"play"` + 德语 `"spielen"` → 共享子词 `"sp"`。

**3. 模型压缩**

- 减少词表大小，提升计算效率。



###### 🌟Transformer Block

**关键组件**：

1. **多头注意力（Multi-Head Attention）**
   并行运行多个自注意力机制，捕捉不同维度的上下文信息。
2. **位置编码（Positional Encoding）**
   由于 Transformer 没有顺序处理能力，需要通过位置编码为输入序列中的每个位置添加位置信息。
3. **前馈神经网络（Feed-Forward Network）**
   对每个位置的表示进行非线性变换。
4. **残差连接和层归一化**
   加速训练并提升模型稳定性。
   残差连接--将输入和输出相加方式解决梯度消失的问题。
   层归一化、正则化--将输入的平均值变为0 方差为1

> [!NOTE]
>
> 泛化能力是指面对未在训练时出现过的问题，依旧能够给出优解的能力

---

#### Basics of Large Language Models

##### 训练目标

**损失函数：**越小越好

- 均方差  ，即**实际训练结果**与**函数预估结果**的差值的平方和 
- 交叉熵  ， 例如文本训练时，情感类别的准确性

Q：如何最小化损失函数？

A：梯度下降法（TODO：偏导、微积分）

- 思想：是在每一个梯度，下降“一点点” 来取得整体的优化

- 前向传播

  例：**x -> W -> b -> f -> out**  每一个节点就是层

- 如何计算梯度？通过反向传播公式Backpropagation

  例：out对于b的梯度是多少？就是对数学公式的偏导



##### 预训练语言模型

> PLM （pre language model）

###### 技术演变

1. #### **传统方法：特征工程与统计模型（2000年前）**

   - **核心思路**：依赖人工设计特征（如词袋模型、TF-IDF）或统计方法（N-gram）提取文本特征。
   - **局限性**：无法捕捉语义关联，面临维度灾难（如词表过大导致稀疏性。
   - **典型模型**：N-gram、LSA（潜在语义分析）。

2. **静态词嵌入：分布式语义表示（2010-2017）**

   - **核心思路**：通过神经网络将词映射为低维稠密向量，捕捉词之间的语义关系。
   - **突破性技术**：
     - **Word2Vec**（2013）：通过CBOW/Skip-Gram模型学习词向量，解决词袋模型的语义缺失问题。  
   - **局限性**：词向量静态不变，无法处理一词多义（如“苹果”在不同上下文的语义差异）。

3. **上下文感知模型 In Context-Learning：动态语义建模（2018-2020）**

   - **核心思路**：通过深度神经网络捕捉上下文动态语义，实现一词多义区分。
   - **关键模型**：
     - **ELMo**（2018）：双向LSTM生成上下文相关词向量，首次实现动态语义表示。
     - **BERT**（2018）：基于Transformer的双向编码器，通过Masked Language Model（MLM）和Next Sentence Prediction（NSP）任务，实现句子级语义理解。
     - **GPT系列**（2018-2020）：基于单向Transformer的自回归模型，专注生成任务（如文本续写）。
   - **优势**：BERT在NLP任务（如问答、分类）中刷新多项记录，GPT-3展示出强大的零样本学习能力。

4. **大模型与多模态融合（2020至今）**

   - **核心思路**：扩大模型规模（千亿参数）、融合多模态数据（文本、图像、语音）。
   - **典型进展**：
     - **GPT-4/ChatGPT**：支持多模态输入，生成能力接近人类水平。
     - **多模态模型**：如DALL·E（文生图）、CLIP（图文匹配）。
     - **领域专用模型**：如Med-PaLM（医疗）、Codex（代码生成）。
   - **挑战**：训练成本高、伦理风险（如生成内容偏见。



###### 介绍

- **word2vec** feature-based 开启词嵌入时代，解决稀疏性问题 已经成为历史。

  - **固定滑动窗口**

    - Continuous bag-of-words(CBOW) 若干Context词推测target词

      bag of words 不考虑词的顺序


    - Continuous skip-gram 通过target 推测出context词


  - **one-hot 向量** :只有一个元素为1，其他都为0

    上述方式产生的计算量过大的问题，可以通过如下方式提高计算效率：

    - 负采样 Negative Sampling
    - Hierarchical softmax

  - **Other Learning Way**

    - Sub-Sampling 平衡常见词和罕见词的概率

    - Soft sliding window 思路：离target更近的context的词关系更加紧密

- **GPT**  Transform的Decoder生成式的结构。现在也支持理解     

  - **自回归**：核心思想是用数据自身的过去值预测当前值，广泛应用于时间序列分析、自然语言生成（如GPT）、图像生成等领域。其核心特点是**按顺序生成数据，每一步的预测都依赖于之前的输出结果**

  - 无监督

- **BERT** Transform的Eecoder的结构

  - 它是使用了**双向上下文**的训练，而双向训练存在一个已知的问题就是信息泄漏，它已知下一个，就会出现short cut 直接将下一个词填出，降低了训练的效果。Bert采用了**masked LM 方式**，将一些词（15%）随机的遮蔽掉。
  - 但mask的方式也会出现一些问题 ，当模型迁移时，下游并不会有15%的mask值，而在预训练时，模型很可能会着重于mask词，成为完形填空的高手，但这并不是下游所需要的。因此又提出了时间训练方案 80%时间mask ；10% 替换成一个新词； 10%正常

  存在问题：

  - next Sentence prediction 是训练句与句之间的关系，存在争议
  - 只有15%的词会被预测，而经典的单项训练时100% ，ELECTRA：提出Replaced Token Detection 就是对于每一个词判断是否发生了替换，这样其实也算是一种预测，然后丢弃小的辅助模型，生成100%的模型

- **T5** 相较于GPT、BERT softmax分类的结果，T5采用的任务结构是text2text，把所有任务转变成text2text的结构


> [!TIP]
>
> - Masked LM还在跨语言、跨模块方面有广泛的应用。
>
>
> - NLP 自然语言处理（Natural Language Processing, NLP）是人工智能领域的重要分支，专注于让计算机理解、生成和处理人类语言。
>
> - fine-tuning（微调） 方式基础任务 就是训练出一个好的通用语言模型，并迁移到下游，由下游微调处理下游任务。
>
> - TODO：GPT2 的原论文！虽然其模型并未出现，提出了“传统机器学习由于在单个任务上训练微调，导致其无法拥有泛化的能力”，后续迁移到下游，可能小的操作就可能导致维度的坍塌。



> [!NOTE]
>
> - 理解和生成 对于大模型是两个方向，可能一个大模型适用于理解，不擅长生成 ，反之也存
> - 当预训练自然语言模型训练完成后，其实**预训练模型的雏形**就完成了，有了很优秀的语言能力和一定的通用知识。
> - PPL（Perplexity）:语言困惑度，判断一个语言模型是否优秀流畅的标准

---

> [!IMPORTANT]
>
> **Post-trained**:**Adaptation、Aligned、Fine-tuning**都是后训练

随着模型的参数量变大，微调的成本逐步升高，如何降低成本，高效微调？⬇️

##### PEFT

**PEFT（Parameter-Efficient Fine-Tuning，参数高效微调）**

###### Delta Tuning

> **Delta Tuning** 是一种针对**大规模预训练模型**的**参数高效微调**方法，通过仅调整模型参数的“增量”（即少量参数）来适配下游任务（适用于不同的领域），显著降低计算和存储成本，同时保持与全参数微调相当的性能。

- **增量式方法（Addition-based）**：在原始模型中添加额外的可训练模块或参数，仅优化新增部分。

  - Adapters-Tuning：在每层插入小型神经网络模块，仅训练新增参数，但会使得模型更加臃肿

    - Ladder Side Tuning 放到外面，减少增加模块带来的性能损耗
    - Prefix Tuning 

  - Prompt-Tuning ：仅在输入层增加 

    :warning:不是后面的prompting 和prompt learning ，只是一个delta tuning的一种​

- **指定式方法（Specification-based）**：指定原模型中的部分参数可训练（如偏置项或特定层），其余参数冻结。

  - BitFit ：只微调bias terms

- **重参数化方法（Reparameterization-based）**：假设参数变化具有低秩或低维特性，将优化过程重参数化为低维子空间。

  - **LoRA （Low-Rank Adaptation）**：将权重矩阵的增量（两个矩阵的差值）分解为低秩矩阵，训练低秩部分。



- Delta tuning could effectively work on **super-large** models
  -  Optimizing only a small portion of parameters could stimulate big models
- The structure may become **less important** as the model scaling

---

###### Prompt-Learning

> 丢弃任务的概念，通过人类语言的上下文，作为机器的提示，让机器去学习
>
> **Prompt Learning**通过**重构输入**，将下游任务转化为预训练模型的原始目标（如掩码预测、文本生成），从而利用大模型已有的知识，无需修改模型参数或仅调整少量参数。

GPT3模型 提出Prompt ,在不修改参数的情况下，通过额外的上下文连接起预训练模型和微调模型，使得大模型生成更加优秀的结果

回顾之前的内容可以发现，pre-traning和fine-tuning直接存在一些任务的gap，比如

**pre-traning**: i like eating [mask]. -> apples 

**fine-tuning**: i like eating apples. ->class : positive

**Prompting-learning**: i like eating apples. **It was [mask].** ->positive?or negative? possible rate



**Prompt Learning 组件**

- **Tempplate**
  - 当认为去设计template的时候，机器结果会更加符合人的语义的
  - 自动去生成template的时候，机器很可能会产生一些非直觉的人的结果，但它可能完全符合机器语义（猜测：应该也是AI幻觉的来由吧）
  - 例如：关系抽取
- **Verbalizer（标签词映射器）** 是Prompt Learning中的关键组件，主要用于将模型预测的文本片段映射到具体的类别标签。**类比**：相当于给模型一个“词典”，告诉它哪些词对应哪些分类结果。
- **指令（Instruction）**
- **示例（Demonstration）**

**The Evolvement of Learning Strategy**

- Traditional: Learning from scratch;
- After BERT: Pre-training-then-fine-tuning;
- T5: Pre-training-then-fine-tuning with text-to-text format;
- GPT: Pre-training, then use prompt & in-context for zero- and few- shot;

**Prompt-learning Introduces New Learning Strategies**

- Pre-training, prompting, optimizing all the parameters (middle-size models, few-shot setting)

- Pre-training, adding soft prompts, freezing the model and optimizing the prompt embeddings (delta tuning perspective)

- Pre-training with prompted data, zero-shot inference (Instruction tuning& T0)

1. **Zero-Shot（零样本学习）**

   - **定义**：模型在**没有任何任务示例**的情况下，直接根据任务描述或指令完成新任务。

   - **原理**：依赖预训练阶段学到的通用知识，通过理解任务的自然语言描述（如“翻译成中文”）来生成结果。
     - **例子**：
       - 任务指令：“将英文句子翻译成中文：`Hello, how are you?`”
       - 模型直接输出：“你好，最近怎么样？”

   - **优点**：无需任何标注数据，灵活高效。

   - **缺点**：对任务描述的清晰度要求高，复杂任务效果可能不稳定。

   - **适用场景**：简单、定义明确的任务（如翻译、摘要、情感判断）。

2. **Few-Shot（少样本学习）**

   - **定义**：模型通过**少量示例**（通常几个到几十个）学习新任务，无需额外训练。

   - **原理**：示例作为上下文输入，帮助模型理解任务模式和输出格式。

     - **例子**：

       输入：

       ```
       示例1：苹果 -> 水果
       示例2：老虎 -> 动物
       问题：飞机 -> ?
       ```

       模型输出：“交通工具”。

   - **优点**：少量数据即可显著提升效果，适应复杂任务。

   - **缺点**：示例质量和数量影响结果，可能受上下文长度限制。

   - **适用场景**：需要特定格式或复杂逻辑的任务（如分类、代码生成）。

> [!IMPORTANT]
>
> - **From Prompt-learning to Instruction Tuning** 从一开始的提示学习到后来的指令微调，其实就是从**Few-shot learning on single task**到**Zero-shot Generalization Across tasks**
>
> - **超参数（Hyperparameters）** 是机器学习模型训练前需手动设定的配置参数，用于控制模型的学习过程和结构，直接影响模型性能和训练效率。

Q：那作用与训练的prompt数据是Less is More or More is More?

A：目前还存在争议，高质量少量数据（样本）和大量数据（样本）之间

---

##### Alignment

> Q:Why Alignment?
>
> A:Make LLMs Stronger, Reliable, and Safe

###### RLHF

> **Reinforcement Learning from Human Feedback**
>
> Use **human feedback**, i.e., preferences, as substitutes for correct answers

**Criterion (3H)**

- Helpfulness
- Harmless
- Honesty

**reward model**

通过奖励模型，量化回答的分数，让大模型更加优秀



由于去RLHF还需要建立奖励模型，那有没有方法让模型自己去给自己打分呢？DPO应运而生.

**DPO（Direct Preference Optimization）**

> DPO是一种**无需显式奖励模型的端到端优化方法**，直接将人类偏好数据映射到策略优化目标

---

#### Advcaned Architectures for LLMs

##### RAG

>  Retrieval Augmented Generation  检索增强式生成
>
>  简单来说，就是大模型通过去检索其他信息（也可以做成多模态 例：图片+文字...），来补充上下文结合用户输入**形成新的输入**，再做输出，来达到更好的输出效果

- **Hallucination（幻觉）**: LLMs generate incorrect responses or factual errors 

**优势：**

- **减少幻觉**：**RAG modeling** can help LLMs to **access external knowledge** to make the responses more **accurate and reliable**
- **动态更新知识**：无需重新训练模型，只需更新外部知识库即可让模型获取最新信息（如新闻、研究报告）。
- **领域适应性强**：通过接入垂直领域知识库（如医疗、法律），模型可以快速适应专业场景。
- 透明可解释：检索到的上下文可以作为生成结果的参考依据，提升可信度。

**Challenges of RAG Modeling**:

- Q: **When** to retrieve knowledge?

  - **Passive Retrieval**: Retriever is called once or every step of generation

    A : Forward-Looking Active RAG (FLARE)：Adaptive Retrieval ，适应性检索，当检索到可能的输出时，停止检索，给出答案

- Q：**How** to use retrieved knowledge?

  - **噪声**：The noise of retrieval will affect the performance of language models

    A：**Chain-of-Note（CoN）**：

    分步处理检索内容：对检索到的文档生成多个**中间注释（Notes）**，而**非直接使用原始文本**；

    动态评估与过滤：通过模型自动评估每个Note的相关性和可信度，剔除低质量或无关信息；

    链式精炼：基于筛选后的Notes进行多轮迭代，逐步聚焦到核心信息，最终生成高质量答案。

  - **表意不清**：The query is usually short, thus the user intention is usually unclear

    A：**Chain-of-Thought（CoT）**：Interactively generating the Chain-of-Thought（思考、推理过程） as queries

##### 混合专家（MoE）

> 为什么会出现MoE呢？有上文可知，预训练模型越大，大模型越智能越优秀，但是在实际操作过程中发现，随着数据量大增大，也会产生很多的优化问题，随之提出了MoE：将模型参数分散到多个“专家”网络中，每个任务仅激活相关专家，这里也借鉴了**人脑的结构**：模块化和稀疏化的构成。增加单个专家的训练量扩大数据量，后续设计调度模式将各个专家组合起来，生成一个更大的模型。（DeepSeek就是使用这个架构）

**FFN（前馈神经网络，Feed-Forward Network）**

在 Transformer 中，FFN 的结构为：

- `输入 → 线性变换（升维） → 激活函数 → 线性变换（降维） → 输出`

升维、降维其实就是✖️一个权重矩阵



**Q：怎么去区分Experts？**

A：we can **decompose FFN layers** to different experts.Neurons are **grouped together**, which are **frequently activated together**.

其实就是将矩阵进行线形变化，将激活的神经元集中到一起，形成一簇的感觉



**Q：如何去选择Experts？**

A：构建一个轻量级的Expert selection，可以看做一个路由



**Q：模型稀疏化，对于训练又有什么样的应用？**

A：**Switchable Sparse-Dense Learning (SSD)**：实现**少量训练**即可达到大训练量的效果，并将模型的稀疏度扩大到很高的程度（95%），减少训练成本。

​	steps：

​	1、dense learning 稠密学习到一个阈值（相似性变化微小）

​	2、sparse learning 稀疏学习

​	3、转回dense learning



##### LLMs for Long Sequences

> 长文本（例如：10分钟的1080p的视频）输入的需求是会一直增长的
>
> 而目前所需要的计算量是非常大的，时间复杂度也在O(n^2^)
>
> 那如何将时间复杂度降下来呢？

**Efficient Architecture** 高效的架构：

- **Sparse Attention**：稀疏注意力机制，为什么它能够适用？我们在阅读一本书时，对此刻句子的理解其实不需要对于书前面的很多内容进行关联；就像和一个人交友，不一定需要知道这个人的所有历史一样。
  - Sliding Window Attention：只关注距离近的，例如只关注前面的2个token
  - Global Attention：让大模型一定要记住的东西
  - Content-based Sparse Attention：基于内容，也就是抓住内容重点
- **Memory-based Methods**：联想RNN
  - **Linear Attention**
  - **State Space Model**：如果RNN可以并行计算

**Efficient Implementation**

- Sequence Parallelism 句子并行化



##### Scaling Law

> Large-Scale Data + Large-Scale Parameters：Acquiring general knowledge requires
>
> extensive data, while storing the knowledge necessitates large parameters
> 简单直观的经验，这让我们的训练更加昂贵。

Q：训练成本太高了怎么办？

A：Scaling Law的发现：为成本的降低提供了理论支持，当按照比例模型比例的缩小，损失函数的增长成一条平滑的曲线时，我们就可以通过缩小模型来预测大模型的效果了。



**Scaling law vs. Emergent abilities**：这两个概念，好像是冲突的。

Scaling law 拟合出的是概率，而能力一直存在，但是在低概率的情况下，较难发现，而当模型的增大，概率的增加，发现的几率变大，最终能力从0到1，是一次的变迁，被称之为涌现。



> [!NOTE]
>
> **拟合（Fitting）** 是指通过数学模型或函数对观测数据（或样本数据）进行近似描述的过程，目标是找到能**最佳匹配数据规律**的模型参数或函数形式。拟合是数据分析、统计学和机器学习的核心任务之一，其本质是**从数据中提取规律或预测未来趋势**。

---

#### Multimodal Models

> Cross-modal Tasks : 
>
> Image-to-text Generation; Text-to-image Generation; Cross-modal Retrieval



##### 

##### Basic Methods

###### Attention Mechanism 

- **Cross-modal Attention**
  - **Capture interactions** between word symbols and visual regions
  -  Perform cross-modal information fusion
- **Cross-modal Representation Learning**
  - Learn representations shared in different modalities

###### Pre-training



##### Representative Methods

###### BERT Era

> Cross-modal Understanding 

- **Single Stream** 

  - Architecture: Input text, detected object tags and locations into a single Transformer

  - Pre-trained tasks: Masked token modeling, image-text contrastive learning/matching

- **Two Stream(CLIP)**
  - Text and raw images are encoded by two Transformer encoders and pre-trained by contrastive learning only
  - Advantage: Better efficiency and scalability

> [!IMPORTANT]
>
> Approach (CLIP): Impressive Zero-shot Performance of CLIP
>
> Still an important backbone in LLM era, in providing text-aligned visual representations
>
> Still an active field with increasing capabilities

###### LLM Era

> Cross-modal Generation 

|                 | Image-to-text generation | Text-to-image generation |
| --------------- | ------------------------ | ------------------------ |
| Goal            | Image understanding      | Image generation         |
| Representatives | GPT-4V                   | DALLE-3                  |
| Key             | Techniques ViT + LLM     | LLM + Diffusion models   |
| LLM type        | Mostly autoregressive    | Mostly T5                |
| Model Size      | Can be large 7B-70B      | Not very large           |



> [!IMPORTANT]
>
> 人对于语言的学习，第一语言：是事物对语言的对齐，第二语言：是第一语言和第二语言的对齐，那当事物对第二语言对齐时就可以节省很多的工作。大语言模型受其启发进行训练，多语言模型，可以有效减少训练量和解决中文社区资源少、内容质量普遍不高的问题

---

**Limitations of old LLMs**

> 2023年前

- 缺乏对外部工具的使用
- 推理能力还是缺乏
- 大模型是作为个体实现，没有合作能力

既是不足，也是需要研究突破的地方，因此有了**Autonomous Agents**



#### Agent

> Autonomous agents are computational systems that inhabit some
> complex dynamic environment, **sense and act autonomously in this
> environment**, and by doing so realize a set of goals or tasks for which
> they are designed --- Maes (1995)



##### Framework

1. **Controller** : 理解语言，将任务拆解成小任务，提供planning
2. **Tool Set** : a collection of tools with different functionalities
3. **Environment** ： provides the platform where tools operate （可以是真实环境也可能是simulate）
4. **Perceiver** ：感知器 ，获取反馈给Controller



##### How to learn to use tool？

- **Imitation Learning**
  - By **recording data(标注数据)** on human tool usage behaviors, large models mimic human actions to learn about tools.The simplest and most direct method of tool learning.
  - 虽然直观且有效，但是人工标注的成本还是太高了

- **Tutorial Learning**

  - By having the model **read tool manuals (tutorials)**, it understands the functions of the

    tools and how to invoke them



XAgent  : dual-loop（粗planning +细planning） 双循环来解决大模型**大局观的缺失**



##### Collective Intelligence

- **Dual-agent** ： 进行debate，在辩论过程中，修正一些认知

- **Tri-agent** ： 在Dual-agent 之上加上一个judge 

- **Muti-agent**  ：

  - SMallVille小镇（Socail **Simulation**），每个Agent作为一个NPC个体模拟人类社会去生活

    - Role Definition
    - Hierarchical Planning
    - Memory Retrieval
    - Social Behavior

  - ChatDev (**Task** Solving) ，每个Agent都有一个身份 协作完成一个task

    - Role specialization

    - Memory Stream

    - Self-Reflection ：When both agents fail to trigger the termination

      conditions. Extract and retrieve memories. Enlist “pseudo self” as new questioner in fresh chat

    - Coding and Testing 开发

    - Documentation 手册的编写

  - AgentVerse: General Multi-Agent Framework

    - muti-agent **Recruitment** 招募机制

      The capabilities required for different tasks vary, and therefore, the agents recruited at
      this stage will also differ based on the task provided by the user根据任务找到适配的Agent

    - Collaborative Decision-making and Action Execution

    - Verification and Evaluation

    - Quantitative experiments

  





---

#### 大模型训练工具

to device 把计算交给显卡

##### BMTrain

> 更快

**显卡上需要保存的数据有：**

- 参数Parameter
- 梯度
- 中间值
- 优化器

**多显卡的合作模式**

- **广播算子** one->all
- **规约算子**（Reduce）all-->one
- **All Reduce** : all->all
- **Reduce Scatter** : 切分各单显卡的数据，从各个显卡取一部分拼接成一个Data -> 1个显卡
- **All Gather** : 所有显卡data拼接成一个data ->all



**Methods of 模型计算优化**

- **数据并行/分布式数据并行**
- **模型并行** ：减少单个显卡的参数
- **ZeRO**
  - ZeRO-1
  - ZeRO-2
  - ZeRO-3
- **Pipline Parallel 流水线并行** ：分层执行

**细节优化技巧**

- **混合精度训练**
- **offloading**：把大头优化器的存储交给CPU
- **overlaaping**：基于计算和存储异步的特性
- **checkpinting**：减少中间层数据 （transformer的线性层的中间结果）

**Demo**

https://colab.research.google.com/drive/1H-T7PmTjdcgwYUFfMikfxZ4_bMWRKu8h?usp=sharing#scrollTo=w0VnJ_b-CuCT

---

##### BMCook

> 更小、大模型的压缩

- **知识蒸馏 Knowledge Distillation**：抽象出大模型的功能，其实就是输入与输出的逻辑映射，如何用小模型去接受大模型隐含的映射关系，就是知识蒸馏的出发点。

- **模型剪枝 Model Pruning**：减少冗余数据的计算，如何判断数据是否重要或者冗余？

  - 非结构化的剪枝：
    - Weight pruning 权重的discard（丢失30%-40%经验值 能够保证与全模型准确性相差无几的效果）

  - 结构化剪枝：直接对一块、一行、一列进行剪枝，保持数据的结构化，对速度的优化更加明显
    -  Attention head pruning
    - Layer pruning

- **模型量化 Model Quantization**：

- **Other Methods**：

  - Weight Sharing 权重共享：跨层使用相同的权重矩阵
  - Low-rank Approximation ：低秩分解，将一些特征值低的对角线参数丢弃掉
  - Architecture Search：寻求最优的transformer的最优结构

加速、压缩训练的工具

https://github.com/OpenBMB/BMCook

---

##### BMInf

> 大模型推理工具
>
> 工具历史：BMInf->BMTrain->BMInf

为什么出现BMinf？

Q：免费算力的大模型访问，成本高，访问慢，那如何让所有人都可以使用大模型？

A：使用用户自己的算力

Q：随之又存在一个问题：如何让家用级别的显卡也能运行大模型？

A：简单来说Transformer 结构是由许多线性层组成的，而线性层就是矩阵乘法。降低精度并根据大小来映射成存储大小的范围（量化过程），来换取速度

- Quantization ：减少数据量
- Memory Scheduling ：
  - 虚存思路，把大于GPU大小的data放入GPU中
  - 固定一些层放在GPU中，来减少调度时产生的时间开销

---

##### fastTest 

一个文本筛选工具

##### Datadeduplication

1. shuffle
2. tokenizers
3. scaling tokenization

##### ChatML

一个聊天模版

load a dataset -> charML->SFT



---

#### Application

> NLP（Natural Language Processing） = NLU（Natural Language Understanding） + NLG（Natural Language Generation）

##### Information retrieval

**评估指标：**

- MRR (Mean Reciprocal Rank)
- MAP (Mean Average Precision)
- NDCG (Normalized Discounted Cumulative Gain)

传统IR**基于词汇匹配**召回文档，会存在同义词文档不能被找回的问题

Neural IR ： 

- Dual-Encoder :qd（queries documents）



**前沿热点**：

- 寻找更难寻找到的负例，对模型进行训练
- IR-oriented Pretraining 
- few-shot IR ：数据筛选器 是·为了去噪声



##### Question answering

-  **Machine Reading Comprehension:**
  -  Read specific documents and answer questions
- **Open-domain QA:**需要自己去寻找相关文档
  -  Search and read relevant documents to answer questions
-  **Knowledge-based QA:**
  - Answer questions based on knowledge graph
- **Conversational QA and dialog:**
  - Answer questions according to dialog history



##### Text generation

> Data to text  |Text ro text







---

#### Ex

##### MCP

> （Model Context Protocol）是一种面向AI接口的自动化工具或协议，旨在简化AI模型与外部数据源、工具的集成，提升开发效率

MCP由Anthropic提出，是一种开放协议，**专注于解决AI模型与外部系统交互的复杂性**，其核心功能包括：

- **标准化连接**：提供统一接口，允许大型语言模型（如Claude）与外部工具（如数据库、API、操作界面）直接交互，无需为每个工具单独编写代码。
- **自动化流程**：通过协议定义任务执行逻辑（如点击网页、填写表单），使AI能够像人类一样操作外部系统，例如Manus项目利用MCP协议实现自动化任务执行。
- **提升开发效率**：减少手动编程需求，开发者可通过自然语言指令配置AI与工具的交互，降低集成门槛。



