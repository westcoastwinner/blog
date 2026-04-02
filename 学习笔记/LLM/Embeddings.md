文本无法被计算机理解 ，故需要将其结构化表示；



词袋法（Bag-of-Words）被废弃，因其单纯将句子切割为词汇，忽略了语义；

## Embedding Model

### Embeddings

**Embeddings**实际就是表示数据的多维向量（**Embeddings are vector representations of data that attempt to capture its meaning.**）

Embedding 是通过神经网络“训练”出来的，Embedding 并不是人工定义的数字，而是神经网络在学习任务（比如预测下一个词）时，自动学习到的内部表示。

网络刚开始运行时，所有的词向量都是随机的数字；在训练过程中（例如 Word2vec 或 BERT），网络会不断调整这些数字，使得在语境中经常一起出现的词，其向量在空间中靠得更近。神经网络的权重矩阵（Weight Matrix）中，输入层到隐藏层之间的那部分参数，实际上就成了该词的 Embedding 向量。

Embedding的每一个维度都代表词汇的某种属性（property），（实际上，这些属性通常相当晦涩难懂）



RNN（Recurrent Neural Network，循环神经网络）处理序列数据，存在健忘和无法并行的问题，被**Attention**取代



### Tokens & Embeddings

![](C:\Users\Lenovo\Desktop\English\blog\LLM\QQ_1773903774215.png)

**Tokenization（分词）**：从文本到数字 ID

计算机无法直接理解“你好”或“Hello”，它只能处理数字。Tokenization 的任务就是**把连续的字符串切分成一个个最小单位（Token），并映射成数字**。

过程：根据词表（Vocabulary）将文本切开。现在的 LLM 通常使用 **Subword（子词）** 算法（如 BPE、WordPiece）。例如：`Smartest` 可能会被切分为 `Smart` + `est`。这样即使遇到生僻词，模型也能通过组合基础词根来理解。每个 Token 都会对应词表里的一个唯一索引。

**Embedding（嵌入）**：从数字 ID 到语义向量

如果只停留在 Tokenization，模型只知道 `1203` 和 `1204` 是两个不同的编号，但不知道它们是什么关系。Embedding 模型的作用是将这些孤立的数字，经过**推理 (Inference/Embedding)**转换成包含语义信息的**稠密向量（Dense Vector）。**

核心原理：

高维空间：语言模型在其分词器中为每个词元都保存一个嵌入向量。这使得每个 Token 会被表示为一个长达数百甚至上千维的向量（比如 Llama-3 的维度是 $4096$）。

语义近邻：在向量空间中，意思相近的词，它们的坐标位置也会非常接近。

- 比如 `"猫"` 和 `"狗"` 的向量距离，会比 `"猫"` 和 `"手机"` 的距离近得多。

特征维度：向量的每一维可能暗含了某种特征（虽然人类不可读），比如一维代表“生物性”，一维代表“毛茸茸度”。



**Note1**：语言模型为tokenizer分词器词汇表中的每个Token都保存了一个Embedding 向量。当我们下载预训练语言模型时，模型的一部分就是这个包含所有这些向量的嵌入矩阵。在训练过程开始之前，这些向量像模型的其他权重一样被随机初始化，但训练过程会为它们赋予能够实现其训练目标的功能性行为的值。

**Note2**：高级用户很快意识到，仅靠生成模型本身并不能构成可靠的搜索引擎。这促成了检索增强生成（RAG）的兴起，它结合了搜索和语言生成模型（LLM）。



### Embedding & Embedding Matrix

当我们下载一个Embedding Model，其实还附带下载了一个**Embedding Matrix**, 这个东西实际是token id到embedding vector的映射矩阵。

![](C:\Users\Lenovo\Desktop\English\blog\学习笔记\LLM\QQ_1774922666977.png)

这个矩阵是embedding模型训练后的产物。一开始每个token id被赋予一个随机的vector,当模型在海量文本上进行预训练时（比如预测下一个词），误差会通过反向传播传递。梯度不仅会更新 Transformer 层里的 W_q, W_k, W_v 等权重，也会一直向前传递，更新 Embedding Matrix 中被选中的那些行。

训练结束后，这张表里的数值就不再是随机数，而变成了能够代表词义的坐标。比如 "King" 和 "Queen" 的向量在空间中会变得非常接近。

而在我们使用模型时，输入一系列 Tokens（如 `[I, love, AI]`），模型首先去 Embedding Matrix 里“查表”。这时得到的 Embedding 是**静态**的——无论 "bank" 是指银行还是河岸，查出来的初始向量都是同一行。



模型会给这些**静态向量加上位置信息**（位置编码-Positional Encoding），让模型知道词的顺序。然后这些静态向量通过**多层 Self-Attention**，之间充分进行交互（例如`The apple is a great company`,`apple`向量会充分吸收`company`向量的信息）。经过层层计算，最终输出的向量（最后一层的 Hidden State）已经不再是最初查表得到的那个静态向量，而是**包含了当前语境信息的动态 Embedding**（`apple`向量从表示·苹果·的向量改变为表示·苹果公司·的向量）。

### Embedding分类

**按历史演进分**（从静态embedding到上下文相关embedding）：

- 静态：Word2vec
- 上下文：ELMo, BERT, RoBERTa, GPT 系列 （背后通常是 Transformer 架构。同一个词在不同的句子中，经过 Attention 机制计算后，得到的向量是不同的。）



**按处理对象分**：

- **Token Embedding**：word - token - embedding
- **Text Embedding:** sentence/para/document - tokens - embeddings - (single) embedding
- 注意，Text Embedding是将一个chunk转化为一堆 Token Embeddings，再经过**池化层**处理变成一个 浓缩了整个chunk语义的 `Text Embedding`.RAG系统正是运用这种模型。
- **Image Embedding：**如 ResNet, ViT 提取的特征向量。
- **跨模态/多模态 Embedding (Multimodal Embeddings):**CLIP，将图片和描述图片的文字映射到同一个向量空间，你可以用“一只在草地上跑的柯基”这段文字的向量，直接在图片数据库里检索出对应的图片。



**按训练架构（双塔 vs. 单塔）分类**

在实际的搜索（Search）和推荐（Recommender Systems）系统中，这是最常用的分类

|      | **双塔模型 (Bi-Encoders) **                                  | 单塔模型 (Cross-Encoders)                                    |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ |
|      | 主流 Embedding 模型（如 OpenAI 的 `text-embedding-3-small` 或 HuggingFace 上的 `BGE` 系列）。 | 主要用于**Rerank（重排序）**。                               |
|      | 查询（Query）和文档（Doc）分别通过两个独立的（或共享权重的）Encoder 转换成定长向量。向量可以预先计算并存入向量数据库。检索时只需计算余弦相似度，速度极快。主要用于RAG、大规模语义搜索 | 将 Query 和 Doc 同时输入模型，让它们在 Attention 层直接进行交互。精度极高，因为模型能看到词与词之间所有的交叉关系。但因无法预计算，推理开销巨大。 |

在RAG中，**通常先用双塔模型捞出前 100 个结果，再用单塔模型精排**。





### Transformer

At a high level of abstraction, Transformer LLMs take a text prompt and output generated text.



An output token is appended to the prompt, then this new text is presented to the model again for another forward pass to generate the next token.



A Transformer LLM is made up of a tokenizer, a stack of Transformer blocks, and a language modeling head.       The tokenizer is followed by the neural network: a stack of Transformer blocks that do all of the processing. That stack is then followed by the LM head, which translates the output of the stack into probability scores for what the most likely next token is.



The tokenizer has a vocabulary of 50,000 tokens. The model has token embeddings associated with those embeddings.



![](C:\Users\Lenovo\Desktop\English\blog\学习笔记\LLM\QQ_1774936824796.png)

At the end of the forward pass, the model predicts a probability score for each token in the vocabulary.



The LM head is a simple neural network layer itself. It is one of multiple possible “heads” to attach to a stack of Transformer blocks to build different kinds of systems. Other kinds of Transformer heads include sequence classification heads and token classification heads.



Each processing stream takes a vector as input and produces a final resulting vector of the same size (often referred to as the model dimension).

![](C:\Users\Lenovo\Desktop\English\blog\学习笔记\LLM\QQ_1774939371786.png)

A Transformer block is made up of two successive components:
The attention layer is mainly concerned with incorporating relevant information from other input tokens and positions
The feedforward layerhouses the majority of the model’s processing capacity

![](C:\Users\Lenovo\Desktop\English\blog\学习笔记\LLM\QQ_1774942172352.png)

The training process produces threeprojection matrices that produce the components that interact in this calculation:
A query projection matrix
A key projection matrix
A value projection matrix

These matrices contain the information of the input tokens projected to three different spaces that help carry out the two steps of attention:Relevance scoring，Combining information

Attention is carried out by the interaction of the queries, keys, and values matrices. Those are produced by multiplying the layer’s inputs with the projection matrices

### Encoder-Only Models和 Decoder-Only Models

简单来说，**Transformer** 是这一系列模型的“母体架构”，而 **Encoder-Only** 和 **Decoder-Only** 则是根据不同的任务需求，对这个架构进行“截肢”或“侧重”后形成的两种变体。

我们可以把 Transformer 看作一个拥有“输入处理（左手）”和“输出生成（右手）”的完整机器人。

------

1. 它们与 Transformer 的关系

2017 年诞生的原始 Transformer 架构是一个 **Encoder-Decoder（编码器-解码器）** 结构。

- **Encoder（编码器）**：负责理解输入，把文字变成高维度的语义向量。
- **Decoder（解码器）**：负责根据语义向量，一个词接一个词地生成结果。

后来的研究发现，针对某些特定任务，其实不需要完整的机器人，只用“左手”或只用“右手”效果反而更好。

------

2. Encoder-Only Models（仅编码器模型/表示模型）

这种模型只保留了 Transformer 的左半部分。

- **工作原理**：它采用**双向注意力机制（Bi-directional Attention）**。这意味着在处理一个词时，它能同时看到这句话里左边和右边的所有词。

- **擅长任务**：**文本理解**。比如情感分析、命名实体识别（NER）、分类任务。它像是一个深思熟虑的阅读理解专家。

- **代表作**：**BERT** (Google), RoBERTa。

- **直观理解**：给它一段话，它能告诉你这段话在讲什么，或者其中的语法错误在哪。

  

  经过预训练的BERT通过一定微调可以用于特定工作：分类 、命名实体识别等：

  ![](C:\Users\Lenovo\Desktop\English\blog\LLM\QQ_1773991240982.png)

------

3. Decoder-Only Models（仅解码器模型/生成模型）

与BERT的仅编码器架构类似，2018年（与BERT同年）提出了一种仅解码器架构，旨在解决生成任务。该架构因其生成能力而被命名为Generative Pre-trained Transformer（GPT）.这种模型只保留了 Transformer 的右半部分，也就是目前 **LLM（大语言模型）**。

- **工作原理**：它采用**单向/掩码注意力机制（Causal/Masked Attention）**。在生成当前词时，它只能看到之前的词，不能“偷看”后面的答案。
- **擅长任务**：**文本生成**。比如写故事、写代码、对话聊天。它像是一个极速续写的作家。
- **代表作**：**GPT 系列** (OpenAI), Llama (Meta), Claude (Anthropic)。
- **直观理解**：给它一个开头，它能一直往下编，直到编出一篇完整的文章。

生成式 LLM 作为序列到序列的机器，接收一些文本并尝试自动补全。虽然这是一个很方便的功能，但它们真正的威力体现在作为聊天机器人进行训练时，通过微调这些模型，它们不仅可以补全文本，还可以被训练来回答问题。即我们可以创建能够遵循prompt的指导式聊天模型。

### **Pre-training（预训练）**和**Fine-tuning（微调）**

在大型语言模型（LLM）的生命周期中，**Pre-training（预训练）**和**Fine-tuning（微调）**是两个最核心的阶段。你可以把这个过程类比为“从通才到专家的进化”。

1. 什么是 Pre-training（预训练）？

预训练是模型学习的第一步，也是最耗费算力和时间的一步。

目标：让模型学习人类语言的基本规律、常识和逻辑推理。

数据：使用海量的、未标记的数据（如整个维基百科、GitHub代码、电子书、网页等）。

学习方式：通常是“自监督学习”。比如给模型一句话：“今天天气很____”，让它预测空格里的词。

结果：产生一个**基座模型（Base Model）**。它博学但“不听话”，你问它问题，它可能不会回答你，而是接着你的话继续往下写。

------

2. 什么是 Fine-tuning（微调）？

微调是在预训练模型的基础上，利用特定领域或特定任务的数据进行二次训练。

目标：让模型学会遵循指令、适应特定语气或掌握专业领域的知识。

数据：量级比预训练小得多（通常是几千到几万条），但质量极高且经过人工标注。

典型场景：**指令微调（Instruction Tuning）**：让模型明白“请帮我翻译这段话”是一个命令，而不是让它续写故事。**领域微调**：让模型学习医学影像分析、法律条文或特定公司的业务逻辑。

![](C:\Users\Lenovo\Desktop\English\blog\LLM\QQ_1773901243583.png)

### LLM 用途

检索评论是正面还是负面；建立相关文件检索查询系统；构建可以利用外部资源（例如工具和文档）的 LLM 聊天机器人









## RAG

**概念先行--chunk**：在英文的语言学习领域中，**chunks**表示a bunch of words/words that are used together with other words，即习惯用语、固定搭配或常见的句子组合。

在RAG中，**Chunk** 是处理长文本的物理单位。

**定义：** 将长篇大论切割成适合模型处理、且语义相对完整的数据段落。

**例子：** 假设你有一篇 5000 字的关于“数据交易法律”的文档：不分片（No Chunking）： 你直接把 5000 字喂给 Embedding 模型，它会因为太长而“消化不良”，生成的向量模糊不清。

**分片（Chunking）：** 你把它切成 10 个 500 字的 Chunks。Chunk 1: 专门讲“数据确权”。

Chunk 2: 专门讲“交易定价”。...

**这10个chunks经过embedding模型生成10个dense embedding，存入vector db。**当用户问“怎么定价？”时，系统能迅速通过向量匹配找到 Chunk 2，而不是把整本法律书扔给 AI。



![](C:\Users\Lenovo\Desktop\English\blog\学习笔记\LLM\QQ_1774099851202.png)

本意解决LLM的幻觉（hallucinate）和无法实时的问题；

**运作流程**：RAG先把数据(txt,pdf,html...)切分成chunk塞入向量数据库；

用户query通过embedding模型转为embedding，去向量数据库匹配最佳chunk，返回的文本(context)和query整合为一段提示词(prompt)然后去调用LLM执行生成。



**潜在问题**：由于存储的内容可能过时、被截断或结构损失（struvture lost）,检索回的文本（context）可能不完整、过时或压根就不对。但LLM不会识别这些，进而导致灾难回答。

实际上，根源来自两点：**数据的多格式多版本**；**用户的模糊问题**（vague questions）



**优化后的企业级RAG（Production RAG）**：

在数据源进入端，增加对数据结构的解析重构，Chunking时保留结构(Structure-Aware Chunking)，并且还会根据这些chunks通过LLM生成总结、提取关键词和生成虚拟问题（模拟用户可能提的问题），然后再一并存入数据库。

注意，还不只有向量数据库，还包含relational database存相关数据。这是为了解决上面提到的问题：数据过时、类型多、信息被截断，所以这个相关数据库存的是数据按时间、类型过滤的信息以及联合多个chunk拼凑完整信息。即储存Filter&Joining.



在用户query进入端，使用**混合搜索**（hybrid search）替代单一的向量搜索，如引入关键词搜索。如此，能更好匹配上一些专有名词。混合搜索得到的多个结果经过**重排序（Rerank）**来检索出最相关的chunks。

注意，之所以还要Rerank一下，实际是因为混合搜索得到的结果太多，可能超出 LLM限制，抑或是导致LLM 出现Lost in the middle的现象，上下文太长反而抓不住重点。rerank模型使用cross-encoding（交叉编码）如图，先combine query和documents，再经模型处理， 模型内部的 注意力机制（Attention） 会让 Query 中的每一个词和 Doc 中的每一个词进行全方位的“交叉比对”。

![](C:\Users\Lenovo\Desktop\English\blog\学习笔记\LLM\QQ_1774162205226.png)

除此之外，在对query的处理上，实际上还包含一层**REASONING ENGINE**用于详解用户意图，明确所需信息、执行步骤、调用工具。它会影响RAG接下来的动作，如直接执行检索、调用外部API或是执行一些计算。



更进一步，RAG在REASONING ENGINE后还接一层**MUTI-AGENT SYSTEM**，不同的Agent专精不同的事，比如一个负责检索金融讯息，一个负责计算，另一个负责总结。

![](C:\Users\Lenovo\Desktop\English\blog\学习笔记\LLM\QQ_1774103360266.png)



增加了这么多环节，但往往越复杂越容易出错。因此我们再给RAG上层保险-Validation&Evaluation

### Milvus  V.S. Chroma 



### 杂碎概念

**MVP (Minimum Viable Product)** —— 最小可行性产品

在 AI 开发中，MVP 指的是用最快速度、最低成本构建出的一个能跑通核心逻辑的版本。



**Tool (工具 / 插件)**： 大模型可以调用的**外部 API 或函数**。

大模型本质上是一个“预测下一个词”的概率引擎，它没有实时联网能力，也不会算复杂的数学题。Tool 就是赋予大模型外部干预的能力。



**Skill (技能)** —— AI 的“工作流”

**Skill** 是比 Tool 更高维度的概念。如果说 Tool 是一个单一的动作（如“查天气”），那么 Skill 就是一套完整的任务解决逻辑。

- **本质**：将 Prompt、多个 Tool 的组合、以及逻辑判断（If/Then）封装在一起的复用模块。

- **特征**：它通常带有特定的上下文和专业知识。

- **对比理解**：

  - **Tool**：调用发票识别接口。
  - **Skill**：**“财务报销助手”**（它知道先识别发票，再核对公司抬头，最后自动填入审批表）。

  

### Chunking -> Tokenization -> Embedding

**Q:**chunking完往往是一个段落，接下来是tokenization吗？如果是的话，这里的tokenization是怎么做的，难道是把整个段落转化为一个token id?

Gemini 说

这个问题触及了 NLP（自然语言处理）最底层的运作机制。简单直接的回答是：**是的，接下来是 Tokenization；但它绝对不是把整个段落变成“一个” ID，而是变成“一串” ID。**

如果你把整个段落变成一个 ID，那意味着模型必须为世界上所有可能的段落组合都预留一个位置，这在数学上是不可能的。以下是 **Tokenization** 在 Chunk 之后具体是怎么操作的：

------

**1. Tokenization 的微观操作：拆解而非打包**

Tokenization 的本质是**“拆解”**。它将你切好的 Chunk（比如 500 字的段落）切碎成模型能理解的最小单元（Tokens）。

- **输入：** 一个包含 500 个单词的 Chunk 字符串。
- **处理：** Tokenizer（分词器）扫描字符串，将其拆分为子词（Subwords）。
  - 例如，单词 `Smartphones` 可能会被拆成 `Smart` 和 `phones` 两个 Token。
- **输出：** 一个 **Token ID 列表**（一维数组）。
  - 比如：`[1543, 482, 92, 2301, ...]`，列表的长度通常略大于单词数。

------

**2. 核心区别：Token ID 列表 vs. Embedding 向量**

这是最容易产生误解的地方，请看这张对比表：

| 阶段                 | 产物形式                    | 含义                                                  |
| -------------------- | --------------------------- | ----------------------------------------------------- |
| Step 1: Chunking     | 字符串 (String)             | “这段话讲的是数据定价。”                              |
| Step 2: Tokenization | ID 列表 (List of Integers)  | “这段话由第 1543、482... 号词汇组成。”                |
| Step 3: Embedding    | 单个向量 (One Dense Vector) | “这段话在 1536 维空间里的坐标是 [0.12, -0.05, ...]。” |

------

**3. 从“一串 ID”到“一个向量”：Embedding 模型的魔力**

你可能会问：*“既然 Tokenization 产生了一串 ID，为什么最后只剩一个向量了？”*

这就是 **Embedding 模型（如 BERT 或 OpenAI 的模型）** 内部做的事情：

1. **逐词表征：** 模型先给这一串 ID 中的每个 Token 分配一个初始向量。
2. **上下文计算：** 模型通过 **Transformer 架构（注意：不是训练，是前向计算）**，让这些 Token 互相“打招呼”。每个 Token 都会吸收周围 Token 的信息。
3. **池化 (Pooling)：** 这是关键！模型会将这一串经过交互的向量通过数学手段（如取平均值或取第一个特殊 Token 的向量）**压缩成一个固定长度的向量**。

**这个最终的向量，就是整个 Chunk 的“语义浓缩精华”。**

### Prompt Engineering

即通过调整优化输入 （精心编写提示词）来指导LLM产生最准确相关的输出。

**核心**在于提供足够的**context**, **constraints**, and **structure**来最小化"hallucinations"

**Prompt的必要构成要素：**

1. **Instruction:** A specific task or move you want the model to perform (e.g., "Summarize," "Write code," "Analyze").
2. **Context:** External information or background that helps the model understand the "why" and "how" (e.g., "Act as a senior Java developer," or "The audience is 5-year-olds").
3. **Input Data:** The specific piece of information you want processed (e.g., a block of text, a CSV file, or a code snippet).
4. **Output Indicator:** The desired format or style of the result (e.g., "Return this as a JSON object," or "Format this as a table").

即 做什么、在什么背景下做、输入信息、输出格式。

**Prompt的必要技艺**：

给 LLM提供2-5个输入-输出示例，这会使其输出格式和风格更贴近你的意图。这称为**Few-Shot Prompting**。

对于数理逻辑推理任务，可以要求LLM **"think step-by-step."** ，这会强迫LLM将复杂问题拆解为 一个个小问题逐一解决。这称为**Chain-of-Thought (CoT)**。

使用明确的**分隔符**，使LLM更容易区分指令和数据。

使用具体的词汇、积极的限制（to do rather than not to do）。

**控制温度**（常用于API）。较低的温度（0.1–0.3）适用于事实性/编程任务，较高的温度（0.7–1.0）适用于创意写作。



## 深度学习

深度学习的核心是神经网络，神经网络模拟人脑的神经元结构，拥有多个层（layer），每一层都有若干神经元。每个神经元视作一个函数**W**=[w1,w2,w3]，对于前一层神经元的输出向量**X**=[X1,X2,X3]，该神经元作用于**X**，并加上一个偏置值b，通过激活函数 **sigma** 得到该神经元的标量输出。本层所有神经元的输出组合成新向量 **Y**，传递给下一层。

![](C:\Users\Lenovo\Desktop\English\blog\学习笔记\LLM\QQ_1775115350778.png)

### **Cost Function** & 梯度下降**Gradient descent** 

Cost Function（代价函数）本质是评价权重函数的合理性，

通常，代价函数是所有训练样本**损失函数（Loss Function）**的平均值：
$$
J(W, b) = \frac{1}{m} \sum_{i=1}^{m} L(\hat{y}^{(i)}, y^{(i)})
$$

$$
其中 \hat{y}是预测值，y 是真实标签。
$$
根据任务类型的不同，代价函数的形式主要分为以下两类：

**1. 回归任务：均方误差 (MSE)**

如果你在预测一个连续的数值（比如房价、股票价格），最常用的就是**均方误差（Mean Squared Error）**。

- **形式：**
  $$
  J = \frac{1}{2m} \sum_{i=1}^{m} (\hat{y}^{(i)} - y^{(i)})^2
  $$
  

- **直观理解：** 它计算预测值与真实值差值的平方。平方的作用有两个：一是确保结果为正数；二是**放大误差**——误差越大，代价增加得越快，从而惩罚那些离谱的预测。

**2. 分类任务：交叉熵损失 (Cross-Entropy)**

如果你在做分类（比如判断图片里是猫还是狗），通常使用**交叉熵（Cross-Entropy）**。这是目前深度学习中最核心的函数形式。

**二分类 (Binary Cross-Entropy)**

当输出层只有一个神经元，且使用 Sigmoid 激活函数输出概率 $p$ 时：

- **形式：**
  $$
  J = -\frac{1}{m} \sum_{i=1}^{m} [y^{(i)} \log(\hat{y}^{(i)}) + (1 - y^{(i)}) \log(1 - \hat{y}^{(i)})]
  $$

  $$
  直观理解：
  
   当真实标签 y=1 时，如果预测值 \hat{y} 接近 0，-\log(\hat{y}) 会趋向正无穷，代价极大。
   这个公式巧妙地利用了对数函数的特性，在预测错误时给予极高的惩罚，在预测正确时代价趋近于 0。
  $$

  

**多分类 (Categorical Cross-Entropy)**

当有多个类别（如手写数字 0-9），输出层配合 Softmax 函数使用：

- **形式：**
  $$
  J = -\sum_{i=1}^{K} y_i \log(\hat{y}_i)
  $$

  $$
  （这里 K 是类别数，y 通常是 One-hot 编码，即只有正确类别的位为 1，其余为 0）。
  $$



**Gradient descent** （梯度下降）则是通过对Cost Function的各个变量（即权重参数）求偏导，得到一个“最佳下降方向”，以使各个权重参数向这个方向进行修正，达到Cost Function最小化的目的，从而得到一套合理的权重参数。