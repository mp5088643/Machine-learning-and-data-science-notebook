# Graph2seq：基于注意的神经网络的图到序列学习

**摘要:**著名的顺序学习（seq2seq）技术及其众多变体在许多任务上都取得了优异的性能。然而，许多机器学习任务的输入自然地表示为图形；现有的seq2seq模型在实现从图形形式到适当序列的精确转换方面面临着重大挑战。为了解决这一难题，我们引入了一种新的通用端到端图序列神经编码器解码模型，该模型将输入图映射到矢量序列，并使用基于注意的LSTM方法对这些矢量的目标序列进行解码。我们的方法首先使用改进的基于图的神经网络和一种新的聚合策略生成节点和图的嵌入，以将边缘方向信息合并到节点嵌入中。我们还介绍了一种注意机制，将节点嵌入和解码序列对齐，以便更好地处理大型图形。对babi、最短路径和自然语言生成任务的实验结果表明，我们的模型达到了最先进的性能，显著优于现有的图神经网络、seq2seq和tree2seq模型；使用所提出的双向节点嵌入聚合n策略，模型可以快速收敛到最优性能。

## 一、简介

著名的序列学习序列（SEQ2SEQ）技术及其众多变体在神经机器翻译（BahanDau等人，2014；GeHern等人，2017），自然语言生成（NLG）（Sun等人，2017）和语音识别（张）等许多任务上取得了优异的性能。等，2017年）。大多数拟议的SEQ2seq模型可被视为一系列编码器解码器（Sutskever等人，2014年；Cho等人，2014年；Bahdanau等人，2014年），其中编码器以序列形式将源输入读取和编码为固定维的连续矢量表示和解码器T。获取编码向量并输出目标序列。许多其他的增强，包括双向复发神经网络（bi-rnn）（Schuster和Paliwal，1997）或双向长短期记忆网络（bi-lstm）（Graves和Schmidhuber，2005）作为编码器，以及注意力机制（Bahdanau等人，2014；Luong等人，2015），已被提议进一步提高其在一般或特定领域应用中的实际性能。

尽管seq2seq模型具有灵活性和表达能力，但它的一个显著限制是，它们只能应用于输入以序列表示的问题。然而，序列可能是最简单的结构化数据，许多重要的问题最好用更复杂的结构来表示，例如具有更大的能力对数据中复杂的成对关系进行编码的图。例如，NLG应用程序中的一个任务是将图结构语义表示（如抽象意义表示）转换为表示其意义的文本（Banarescu等人，2013）。此外，移动机器人的路径规划（Hu&Yang，2004年）和babi任务中的问答路径查找（Li等人，2015年）也可以被转换为序列问题的图形。

另一方面，即使原始输入最初是以序列形式表示的，它仍然可以从具有附加信息的增强输入中获益（以形成图形输入）。例如，对于语义分析任务（文本到AMR或文本到SQL），通过使用其他结构信息（如依赖性分析树）增加原始句子序列，可以更好地显示它们的性能（Pust等人，2015）。直观地说，图形到序列任务的理想解决方案是构建一个更强大的编码器，它能够学习输入表示，而不管其固有结构如何。

为了处理图到序列的问题，一种简单而直接的方法是将更复杂的结构化图数据直接转换为序列（Iyer等人，2016；Gomez Bombarelli等人，2016；Liu等人，2017），并将序列模型应用于结果序列。然而，seq2seq模型在这些问题上往往无法达到预期的效果，部分原因是由于将复杂的结构化数据转换为序列，特别是当输入数据自然地表示为图形时，不可避免地会遭受重大的信息损失。最近，一系列研究工作致力于通过提取句法信息，如源句的短语结构（tree2seq）（eriguchi et al.，2016），利用输入集的注意机制（set2seq）（vinyals et al.，2015a），整合额外的信息。将句子递归地编码为树（Socher等人，2010年；Tai等人，2015年）。尽管这些方法在某些类型的问题上取得了有希望的结果，但大多数所提出的技术在很大程度上依赖于底层应用程序，并且可能无法以一般的方式概括为广泛的一类问题。

为了解决这一问题，我们提出了一种新的基于注意的图到序列学习神经网络模型graph2seq。graph2seq模型遵循传统的编码器-解码器方法，包含两个主要组件：图形编码器和序列解码器。本文提出的图形编码器的目的是学习表达节点嵌入，然后将它们重新组合成相应的图形嵌入。为此，在最近的图表示学习方法（Hamilton等人，2017a）的启发下，我们提出了一种基于归纳图的神经网络，通过有向图和无向图的邻域信息聚合，从节点属性中学习节点嵌入，探索了两种方法。在每个节点上限制聚合器，以生成两个连接起来形成最终节点嵌入的表示。此外，我们还设计了一种基于注意的RNN序列译码器，该译码器以图形嵌入为初始隐藏状态，通过学习根据与相应节点相关的上下文向量和所有先前的预测进行联合对齐和翻译，输出一个目标预测。ONS。我们的代码和数据可以在https://github.com/ibm/graph2seq上找到。

graph2seq既简单又通用，具有高度的可扩展性，它的两个构建块（图形编码器和序列解码器）可以被图形卷积（注意）网络（kipf&welling，2016；velickovic等人，2017）或其扩展（schlichtkrull等人，201）等其他模型所取代。7）和LSTM（Hochreiter和Schmidhuber，1997年）。我们重点介绍了本文的三个主要贡献：

- 我们提出了一个新的基于注意力的神经网络模型，它优雅地处理图到序列的学习问题，学习图结构输入到序列输出之间的映射，而当前的seq2seq和tree2seq可能不足以处理。
- 提出了一种新的图形编码器，利用不同的聚合策略，对具有节点属性的有向图和无向图进行双向节点嵌入，并利用两种不同的图形嵌入技术来学习图形级嵌入。同样重要的是，我们提出了一种注意机制来学习节点和序列元素之间的对齐，以便更好地处理大型图。
- 实验结果表明，我们的模型在最近引入的三个图到序列任务上达到了最先进的性能，显著优于现有的图神经网络、seq2seq和tree2seq模型。

## 二、相关工作

我们的模型从图形表示学习、图形上的神经网络和神经编码器-解码器模型的研究领域得到了启发。

**图形表示学习：**图表示学习已被证明对广泛的基于图的分析和预测任务非常有用（Hamilton等人，2017b；Goyal和Ferrara，2017）。图表示学习的主要目标是学习将节点作为点嵌入到低维向量空间中的映射。这些表示学习方法可以大致分为两类，包括基于矩阵分解的算法和基于随机游走的方法。一行研究通过矩阵分解学习图节点的嵌入（Roweis和Saul，2000；Belkin和Niyogi，2002；Ahmed等人，2013；Cao等人，2015；Ou等人，2016）。这些方法直接对训练和测试数据的各个节点的嵌入进行联合训练，从而内在地转换。另一个工作系列是使用基于随机游走的方法，通过探索单个大型图的邻域信息来学习节点的低维嵌入（Duran&Niepert，2017；Hamilton et al.，2017a；Tang et al.，2015；Grover&Leskovec，2016；Perozzi et al.，2014；VelickovIC等人，2017年）。

GraphSAGE（Hamilton等，2017a）是这样一种技术，它通过使用节点属性或程度进行归纳学习来从节点局部邻域通过聚合来学习节点嵌入，其具有更好的能力来为先前看不见的数据生成节点嵌入。

我们的图形编码器是对图形库的扩展，有两个主要区别。首先，我们将原始节点按照边缘方向拆分为前向节点（节点指向）和后向节点（直接指向节点），并将两个不同的聚合函数应用到这些类型的节点上，从而使其能够处理有向图和无向图。其次，我们利用两种不同的方案（基于池的和基于超节点的）来重新组合学习的节点嵌入，以生成图嵌入，这在图形存储中没有研究过。在我们的实验中，我们展示了我们的图形编码器比图形库的优势。我们的图形编码器是对GraphSAGE的扩展，有两个主要区别。首先，我们将原始节点按照边缘方向拆分为前向节点（节点指向）和后向节点（直接指向节点），并将两个不同的聚合函数应用到这些类型的节点上，从而使其能够处理有向图和无向图。其次，我们利用两种不同的方案（基于池的和基于超节点的）来重新组合学习的节点嵌入，以生成图嵌入，这在图形存储中没有研究过。在我们的实验中，我们展示了我们的图形编码器比GraphSAGE的优势。

**图神经网络：**在过去的几年里，已经有了一系列的方法来学习图节点的表示，或整个（子）图，这些方法基于图神经网络（GNN），将包括RNN和CNN在内的著名网络结构扩展到图数据（Gori等人，2005；Scarselli等人，2009；Li等人，2015；Bruna等人，2013；Duvenaud等人，2015；Niepert等人，2016；Deffarrard等人，2016；Yang等人，2016；Kipf&Welling，2016；Chen等人，2018）。一项研究是神经网络，它以RNN的形式在图上运行（Gori等人，2005年；Scarselli等人，2009年），最近由Li等人扩展。（Li等人，2015）通过在原始GNN框架中引入RNN的现代实践（使用GRU更新）。另一个最近引起人们快速增长兴趣的重要工作是基于光谱图理论的图卷积网络（GCN），由Bruna等人介绍。（2013年），然后由Defferrard等人进行扩展。（2016）具有快速本地化卷积。这些方法中的大多数无法扩展到大型图，这是通过使用光谱图卷积的局部一阶近似（kipf&welling，2016）和进一步配备用于推导快速GCN的重要采样而改进的（Chen等人，2018）。

与我们的图形编码器密切相关的工作是GCN（kipf&welling，2016），它设计用于在转导设置中的半监督学习，要求在培训期间提供全图形拉普拉斯图，通常适用于单个大型无向图。GCN的一个扩展可以证明在数学上与我们在无向图上的图编码器的一个变体相关。我们在实验中比较了图形编码器和GCN之间的差异。另一项相关工作是门控图序列神经网络（GGS NNS）（Li等人，2015）。虽然它也设计用于输出序列，但它本质上是一个预测模型，学习预测嵌入在图中的序列，而我们的方法是一个生成模型，学习图输入和序列输出之间的映射。我们提出的graph2seq和GGs-NNs之间的一个很好的类比是卷积seq2seq和rnn之间的关系。

**神经编码器解码器模型：**最成功的编码器-解码器架构之一是序列到序列学习（Sutskever等人，2014；Cho等人，2014；Bahdanau等人，2014；Luong等人，2015；Gehring等人，2017），最初建议用于机器翻译。最近，经典的seq2seq模型及其变体已被应用于多个应用中，其中这些模型可以执行从对象到序列的映射，包括从图像到句子的映射（vinyals等人，2015c），从pytho的问题语句计算映射的模型。n方案到他们的解决方案（方案的答案）（Zaremba和Sutskever，2014年）、一组点的旅行推销员问题（Vinyals等人，2015b）和药物发现中现有已知分子生成分子的深度生成模型。很容易看出，在列出的示例中，映射到序列的对象通常是自然地用图形表示的，而不是序列。

最近，在处理更复杂的数据时，为了解决seq2seq的局限性，已经做出了许多研究工作和关键贡献，这些数据利用附加到基础目标应用程序（包括tree2seq）的专用神经模型来利用外部信息（eriguchi等人，2016）、set2seq（Vinyals等人，2015a）、递归神经网络（Socher等人，2010）和三结构LSTM（Tai等人，2015）。由于在图形表示和图形卷积网络方面的最新进展，许多研究已经研究了如何利用各种GNN来提高SEQ2seq模型在机器翻译和图形生成领域的性能（Bastings等人，2017年；Beck等人，2018年）。；Simonovsky和Komodakis，2018年；Li等人，2018年）。这些作品和我们的作品有几个不同之处。首先，我们的模型是第一个通用的编码器-解码器模型，用于图形到序列学习，适用于不同的应用，而前面的研究必须利用特定领域的信息。其次，我们为我们的图形解码器设计了自己的图形嵌入技术，而其他大多数工作直接将现有的GNN应用到它们的问题上。

## 三、图到序列模型

如图1所示，我们的图到序列模型包括图编码器、序列解码器和节点注意机制。按照传统的编码器-解码器体系结构，图形编码器首先生成节点嵌入，然后根据学习到的节点嵌入构造图形嵌入。最后，序列译码器以图形嵌入和节点嵌入为输入，在生成序列时注意节点嵌入。在本节中，我们首先介绍了一种节点嵌入生成算法，该算法通过将来自图中节点的前向和后向邻域的信息聚合而得到双向节点嵌入。在这些节点嵌入的基础上，我们提出了两种生成图形嵌入的方法来捕获整个图形信息。

### 3.1 节点嵌入生成

灵感来源于Hamilton等人。（2017a），我们设计了一种新的感应式节点嵌入算法，该算法通过对有向图和无向图的K跳内的节点局部前向和后向邻域的信息进行聚合来生成双向节点嵌入。为了让说明更简洁，以节点$v \in \mathcal{V}$的嵌入生成过程为例，说明了我们的节点嵌入生成算法：

1）我们首先通过查找嵌入矩阵$\mathrm{W}_{e}$，将节点v的文本属性转换为特征向量$\boldsymbol{a}_{v}$。请注意，对于$v^{\prime} \mathrm{s}$的文本属性可能是单词序列的某些任务，可以另外使用一个神经网络层（例如LSTM层）来生成$\boldsymbol{a}_{v}$。

2）根据边缘方向将V的邻域分为前邻、$\mathcal{N}_{\mathrm{H}}(v)$和后邻$\mathcal{N}_{+}(v)$。特别是，n`（v）返回v指向的节点，na（v）返回v指向的节点；特别是，n`（v）返回v指向的节点，na（v）返回v指向的节点；

3）我们将v的前向邻居$\left\{\mathbf{h}_{u^{H}}^{k-1}, \forall u \in \mathcal{N}_{\dagger}(v)\right\}$的前向表示集合为单个向量$\mathbf{h}_{\mathcal{N}_{\mathrm{H}}}^{k}(v)$，其中k∈{1，.，K}是迭代索引。在我们的实验中，我们发现聚合器的选择AGGREGATE`k可能严重影响总体性能，我们将在后面讨论它。注意，在迭代k时，该聚合器只使用kk1生成的表示。每个节点的初始前向表示是其在步骤(1)中计算的特征向量；

4）我们将v的当前正向表示hkv`1与新生成的邻域向量hkn`（v）连接起来。这个连接向量被输入到一个具有非线性激活函数σ的完全连接层中，它更新了v，hkv`的前向表示，以便在下一次迭代中使用；

5）我们使用步骤（3）和（4）中介绍的类似程序更新了v，hkva的后向表示，除了在后向表示上操作而不是前向表示；

6）我们重复步骤（3）～（5）k次，最后向前和向后表示的串联用作v的最终双向表示。由于来自不同跃点的相邻信息对节点嵌入的影响不同，我们在每次迭代。

聚合器架构。由于一个节点邻域没有自然排序，聚合函数应该对其输入的排列保持不变，以确保我们的神经网络模型可以训练并应用于任意排序的节点邻域特征集。在实践中，我们检查了以下三个聚合函数：

平均聚合器：该聚合器函数取hukk`1、u∈n`（v）和hku a1、u∈na（v）中向量的元素平均值。

LSTM聚合器：与（Hamilton等人，2017a）相似，我们还研究了一种基于长期短期记忆（LSTM）架构的更复杂的聚合器。请注意，lstms本身不是对称的，因为它们按顺序处理输入。我们使用lstms对无序集进行操作，只需将它们应用于节点邻居的单个随机排列。

池聚合器：在此聚合器中，每个邻居的向量通过完全连接的神经网络进行馈送，并应用元素顺序的最大池操作：![1562660966714](F:\Machine-learning-and-data-science-notebook\images\graph2seq\1562660966714.png)

其中max表示元素方向的max运算符，σ是非线性激活函数。通过应用max pooling，该模型可以捕获整个邻域集的不同信息。

### 3.2 图形嵌入生成

现有的图卷积神经网络大多集中于节点嵌入，而不是图嵌入，因为它们的重点是节点分类任务。然而，传达整个图形信息的图形嵌入对于下游解码器是必不可少的。在本文中， 我们引入了两种方法（即基于池和基于节点）来从节点嵌入中生成这些图形嵌入。

基于池的图形嵌入。在这种方法中，我们研究了三种池技术：最大池、最小池和平均池。在我们的实验中，我们将节点嵌入到一个完全连接的神经网络中，并将每个池方法元素应用到各个方面。我们发现三种不同的池方法之间没有显著的性能差异；因此，我们采用最大池方法作为默认池方法。

基于节点的图形嵌入。在这种方法中，我们将一个超级节点vs添加到输入图中，并将图中的所有其他节点直接添加到vs中。我们使用前面提到的节点嵌入生成算法，通过聚合相邻节点的嵌入来生成vs的嵌入。捕获所有节点信息的vs的嵌入被视为图形嵌入。

### 3.3 基于注意的解码器

序列译码器是一个递归神经网络（RNN），它预测下一个符号y i，给定所有前面的单词y<i=y1，…，yii1，时间i的rnn隐藏状态si，以及一个上下文向量ci，它将注意力引引到编码器端。尤其是，上下文向量CI依赖于一组节点表示（z1，…，zv），图形编码器将输入图形映射到这些节点表示。每个节点表示zi包含关于整个图的信息，重点关注输入图的第i个节点周围的部分。上下文向量ci作为这些节点表示的加权和计算，每个节点表示的权重αij通过以下公式计算：

$c_{i}=\sum_{j=1}^{v} \alpha_{i j} h_{j},$ where $\alpha_{i j}=\frac{\exp \left(e_{i j}\right)}{\sum_{k=1}^{V} \exp \left(e_{i k}\right)}, e_{i j}=a\left(s_{i-1}, h_{j}\right)$

其中a是一个对齐模型，它对位置j周围的输入节点和位置i处的输出匹配程度进行评分。分数基于RNN隐藏状态SII1和输入图的j节点表示。我们将对准模型A参数化为前馈神经网络，该神经网络与该系统的其他组件联合训练。我们的模型是联合训练，以最大化条件日志概率的正确描述给定的源图。在推理阶段，我们使用波束搜索生成波束大小为5的序列。

## 四、实验

通过实验验证了该方法的有效性和有效性。根据（Li等人，2015）中的实验设置，我们首先在两个选定的任务（包括Babi任务19和最短路径任务）上，将其性能与基于经典LSTM、GGS-NN和GCN的方法进行比较。然后，我们将graph2seq与其他基于seq2seq的方法进行比较，这些方法基于实际的应用程序-自然语言生成任务。请注意，所有基线的参数都是根据开发集的性能设置的。

**实验设置：**我们建议的模型使用Adam优化器进行培训（Kingma&BA，2014年），最小批量为30。学习率设置为0.001。我们采用了退出策略（srivastava等人，2014），解码器层的比率为0.5，以避免过度匹配。当其范数大于20时，梯度被剪短。对于图形编码器，默认跃点大小k设置为6，节点初始特征向量大小设置为40，非线性函数σ为relu（glorot等人，2011），聚合器参数随机初始化。解码器有1层，隐藏状态大小为80。由于使用平均聚合器和基于池的图形嵌入的graph2seq通常比其他配置执行得更好（我们将此讨论推迟到sec。4.4），我们在下面的部分中使用此设置作为默认模型。

### 4.1 BABI任务19

**设置：**Babi人工智能（AI）任务（Weston等人，2015）旨在测试人工智能系统的推理能力。在这些任务中，任务19（寻径）可以说是最具挑战性的任务（见，例如（Sukhbaatar等人，2015年），报告所有不使用强监督的方法的准确度均低于20%。我们应用（Li等人，2015）中介绍的转换过程将描述转换为图2所示的图形。左边部分是babi任务19的一个例子：给定一组描述一对对象o1和o2相对地理位置的句子，我们的目标是找到o1和o2之间的地理路径。

然后将该问题视为找到两个节点（no1和no2）之间的最短路径，它们在图中表示o1和o2。为了用graph2seq解决这个问题，我们用文本属性start注释no1，用文本属性end注释no2。对于其他节点，我们在图中指定它们的ID作为它们的文本属性。值得注意的是，在我们的模型中，开始和结束标记是节点特征，其向量表示首先是随机初始化的，然后由模型稍后学习。相比之下，在GGS-NN中，起始节点和结束节点的向量表示被设置为一个热向量，这是专门为最短路径任务设计的。

为了将边缘信息聚合到节点嵌入中，对于每个边缘，我们还将表示此边缘的节点添加到图中，并将边缘的文本指定为其文本属性。我们生成1000个培训示例、1000个开发示例和1000个测试示例，其中每个示例都是一个图-路径对。我们使用标准LSTM模型（Hochreiter&Schmidhuber，1997）和GGSNN（Li等人，2015）作为基线。由于GCN（kipf&welling，2016）本身无法输出序列，因此我们还创建了一个基线，将GCN与序列解码器结合在一起。

**结果：**从表1中，我们可以看到LSTM模型在执行此任务时失败，而我们的模型进行了完美的预测，这突出了使用图形编码器直接对图形进行编码的重要性，而不是对从图形转换的输入使用序列模型。与对不同类型的节点（如开始和结束）使用精心设计的初始嵌入的GGSNN相比，我们的模型使用一种纯端到端的方法，该方法基于文本属性中单词嵌入的随机初始化生成初始节点特征向量。然而，我们仍然显著地优于GGS-NN，展示了我们的图形编码器的表现力，它考虑了向前和向后的信息流。当我们将整个graph2seq模型与GCN和解码器进行比较时，我们观察到了类似的结果，这主要是因为GCN的当前形式（kipf&welling，2016）是为无向图设计的，因此在将有向图转换为无向图时，可能会有信息丢失，如（kipf）所示。&惠灵顿，2016年）。

### 4.2 最短路径任务

设置。我们进一步评估了最短路径（SP）任务的模型，其目标是在图中找到两个节点之间的最短有向路径，介绍于（Li等人，2015年）。对于这个任务，我们通过生成随机图，并选择由唯一的最短有向路径连接的对随机节点A和B来创建数据集。因为我们可以控制生成的图的大小，所以在增加图的大小时，我们可以很容易地测试每个模型的性能变化。创建了两个这样的数据集：SP-S和SP-L，分别包含小型（节点大小=5）和大型图（节点大小=100）。我们将sp-s生成的最短路径的长度限制为至少2个，而sp-l生成的最短路径的长度限制为至少4个。对于每个数据集，我们使用1000个培训示例和1000个开发示例进行参数调整，并对1000个测试示例进行评估。我们选择与前一节中介绍的相同的基线。

结果。表1显示了LSTM模型在这两个数据集上仍然失败。我们的graph2seq模型实现了与ggs-nn相当的性能，这两个模型都可以在sp-s数据集上实现100%的精度，而在sp-l数据集上的较大图形上实现更好的精度。这是因为我们的图形编码器在使用双向聚合器学习图形结构信息时更具表现力，双向聚合器是图形尺寸增大时保持良好性能的关键，而GGS-NN的性能由于Capturi的硬度而显著降低。在大尺寸图中的长期依赖关系。与GCN相比，它实现了比GGS-NN更好的性能，但仍然比我们的graph2seq低很多，部分原因是图形编码器效率低，并且无法处理有向图形。



### 4.3 自然语言生成任务

设置。最后，我们在一个实际的应用程序-自然语言生成（NLG）任务上评估我们的模型，在这个任务中，我们将结构化语义表示（在本例中是结构化查询语言（SQL）查询）转换为表达其含义的自然语言描述。如（Spiliopoulou和Hatzopulos，1992）所示，SQL查询的结构本质上是一个图。因此，我们自然而然地将此任务作为图形到序列模型的应用，序列模型将表示语义结构的图形作为输入并输出序列。图3说明了通过我们的graph2seq模型2将SQL查询转换为相应的自然语言描述的过程。

我们使用Bleu-4评分来评估wikisql数据集上的模型（Zhong等人，2017年），该数据集包含87726个自然语言问题、SQL查询和SQL表的手写注释实例。wikisql被创建为基于表的问答任务的基准数据集（其最先进的性能是82.6%的执行精度（Yu等人，2018））；这里我们反向使用该数据集，将SQL查询作为输入，并以生成corr为目标。ECT英语问题。这些wikisql SQL查询分为培训、开发和测试集，分别包含61297个查询、9145个查询和17284个查询。

由于SQL到文本任务可以被强制转换为“机器翻译”类型的问题，因此我们实现了几个基线来解决此任务。第一个模型是（Bahdanau等人，2014）提出的基于注意的序列对序列（Seq2seq）模型；第二个模型还介绍了解码器端的复制机制（Gu等人，2016）；第三个模型是（Eriguchi等人，2016）提出的树对序列（Tree2seq）模型我们的基线；第四个是将GCN（kipf&welling，2016）与我们的PGE图形嵌入与序列解码器结合；第五个是将GGS-NN（li等人，2015）与我们的序列解码器结合。为了应用这些基线，我们使用一些模板将SQL查询转换为序列或树，我们将在附录中详细讨论这些模板。

结果。从表2可以看出，我们的graph2seq模型的性能明显优于seq2seq、tree2seq和graph2seq基线。由于SQL查询的结构本质上是一个图形，尽管它的表达式是按顺序排列的，并且图形编码器能够直接在图形中捕获更多的信息，因此预期会得到这个结果。在所有的graph2seq模型中，我们的graph2seq模型表现最好，部分原因是更有效的图形编码器。与seq2seq相比，tree2seq实现了更好的性能，因为它基于树的编码器显式地考虑了SQL查询的语法结构。graph2seq模型的两个变体可以显著优于tree2seq，这表明独立于复杂数据中不同结构信息的一般图到序列模型非常有用。有趣的是，我们还观察到graph2seq pge（基于池的图嵌入）比graph2seq nge（基于节点的图嵌入）性能更好。一个潜在的原因是基于节点的图嵌入方法人为地在图中添加了一个超级节点，从而改变了原有的图拓扑结构，给图中带来了不必要的噪声。

### 4.4 聚集器、跳跃大小和注意力机制对garph2seq模型的影响

设置。我们现在研究聚合器和跃点大小对graph2seq模型的影响。
在前面的sp任务之后，我们进一步创建了三个合成数据集3：i）图是有向非循环图（DAG）的sdpdg；ii）图是始终包含循环的有向循环图（DCG）的sdpdg；iii）图本质上是连续线的sdpseq。
对于每个数据集，我们随机生成10000个图，图大小为100，并将其拆分为8000/1000/1000，用于培训/开发/测试集。对于每个图，我们通过选择两个随机节点来生成一个SDP查询，其中有一个约束条件，即连接这两个节点的路径应该是唯一的最短路径，并且其长度应该至少为4。

在节点嵌入生成过程中，我们创建了六个不同聚合策略耦合的graph2seq模型变体。前三个（graph2seq ma，-la，-pa）分别使用平均聚合器、lstm聚合器和池聚合器聚合节点邻居信息。
与这三个聚合前向和后向节点信息的模型不同，其他两个模型（graph2seq-ma-f，-ma-b）只考虑使用平均聚合器聚合前向节点或后向节点的信息，尊重主动。我们使用路径精度来评估这些模型。跃点大小设置为10。

聚合器的影响。表3显示，在sdpseq数据集上，graph2seq ma和graph2seq pa都达到了最佳性能。在更复杂的结构化数据（如SDPDAG和SDPDCG）上，graph2seq ma（我们的默认模型）的性能也优于其他变体。我们还可以看到，graph2seq ma在sdpdag和sdpseq上的性能优于graph2seq-ma-f和graph2seqma-b，因为它从两个方向捕获更多信息以学习更好的节点嵌入。然而，在SDPDCG上，graph2seq-ma-f和graph2seq-ma-b的性能与graph2seq-ma相当。这是因为在几乎95%的图中，90%的节点可以通过遍历给定跃点大小的图来达到彼此，这将显著地恢复其信息丢失。

**Hop Size**的影响。为了研究跳跃大小的影响，我们创建了一个sdpdg数据集sdp1000，结果如图4所示。我们发现，当增加跃点大小时，graph2seq的所有变体的性能都会收敛到其最佳性能。具体来说，在只考虑单向传播的情况下，graph2seq ma的性能明显优于对应的ma，尤其是在跃点较小的情况下。随着跃点大小的增加，性能差异减小。这是所需的属性，因为graph2seq ma可以使用更小的跃点大小（大约一半）来实现与graph2seq-ma-f或更大大小的graph2seq-ma-b相同的性能。这对于增加跃点大小可能需要大量计算资源和长运行时间的大型图尤其有用。我们还将graph2seq与gcn进行了比较，其中hop大小表示gcn设置中的层数。令人惊讶的是，即使是graph2seq-ma-f或graph2seq-ma-b，尽管它们在这两个体系结构之间有大致的等效性，但在希望大小相同的情况下，也可以显著优于gcn。它再次说明了可以同时考虑有向图和无向图的方法的重要性。对于不同尺寸图的跳跃尺寸影响的其他实验结果，请参考附录C中的表4。

注意机制的影响。为了研究注意力机制对graph2seq模型的影响，我们仍然评估了我们的模型对sdpdag、sdpdg和sdpseq数据集的影响，但没有考虑注意力策略。如表4所示，我们发现注意力策略显著提高了graph2seq所有变体的性能至少14.9%。由于对于较大的图形来说，编码器更难将所有必要的信息压缩成固定长度的矢量，因此可以预期这一结果；正如预期的那样，在解码中应用注意力机制使我们提出的graph2seq模型能够成功地处理较大的图形。

## 5、结论

本文对图到序列问题进行了研究，提出了一种新的通用灵活的图到序列模型，该模型遵循编码器-解码器体系结构。结果表明，利用我们提出的双向节点嵌入聚合策略，图形编码器可以成功地学习三类有向图的表示，即有向无环图、有向循环图和序列样式图。三个任务的实验结果表明，在合成和实际应用数据集上，我们的模型明显优于现有的图神经网络、seq2seq和tree2seq基线。我们还表明，在解码过程中引入节点表示上的注意机制，大大提高了模型从大图中生成正确目标序列的能力。由于许多符号数据被表示为图形，许多任务将其期望的输出表示为序列，因此我们期望graph2seq广泛适用于统一符号ai及更高版本。