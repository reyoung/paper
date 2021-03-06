Merging MLP & HMM: Some Experiments in Continuous Speech Recognition
####################################################################

Introduction
------------

HMM被广泛的应用在语音识别领域，但是如果使用MLE(最大似然估计)训练的话，判别性(Discriminant Properties)弱。使用另一种作为评价准则的训练，最大互信息，会获得更大的判别性，但是有更多的约束和假设需要确定。最后，声学和语音的上下文信息，需要一个复杂的HMM，和更大的存储能力。

另一方面，多层感知器，作为连接性的架构，作为一个可选的工具用在了模式识别领域(例如语音识别)中。他最有用的性质是他的判别能力和学习能力，和表示不明确知识的能力。同时，上下文相关的信息可以很容易的被表示。但是MLP经常被用在非序列化的数据中。同时呢，加入延迟，反馈的神经网络可以提供动态的和不明确的记忆能力。很多人提议使用这个类型的架构。

在这篇文章讨论了将统计模型(HMM)和连接模型(MLP)联系起来。这个假说被发现，当使用HMM并使用MLP的潜在解。it is shown 在理论上和经验上来说，用MLP的输出输出来估算的类别间的概率分布来限制HMM的输入(也就是最大后验概率，在这里也被叫做贝叶斯概率)。同时，it is shown这些使用了上下文相关信息作为输入的MLP的估算，让Frame 分类的性能相对于没有上下文相关助益的MLE和MAP概率估算的HMM模型的性能明显提高。

同时需要明确的是，使用Frame和Phoneme来做识别的时候，用MLP使Frame和Phoneme的性能提升，并不会简单的导致连续语音识别的性能提升。很多的修改和改进才能使性能在词级别有可接受的提升。这个修改会被在后文中介绍。

HMM
---

在常见的离散HMM模型中，语音向量会在前端处理中被量化。每个向量被一个距离它最近的向量 :math:`y_i` 替换。这个向量是一个预先指定的有限集合 :math:`\mathfrak{Y}` ，其基为 :math:`I`. 让 :math:`\mathfrak{Q}` 作为一个有 :math:`K` 个不同状态的集合，每个状态为 :math:`q(k)` ，其中 :math:`k=1,...,K` 。马尔科夫模型是由这些状态，按照预定的拓扑结构组成的。如果HMM使用MLE评价标准来训练，那么模型的参数以最大化 :math:`P(X|W)` , 其中 :math:`X` 是训练使用的量化声学向量 :math:`x_n \in \mathfrak{Y}` ，其中 :math:`n=1,...,N` ,而 :math:`W` 与马尔科夫模型的 :math:`L` 个状态 :math:`q_l \in \mathfrak{Q}` 相关， 其中 :math:`l=1,..., L` 。当然， :math:`L \ne K \ne N` ，因为同样的状态可能在不同的位置出现多次， 也因为所有的状态不一定非要在一个模型里全部出现，也因为状态间的循环是被允许的。不考虑特定HMM模型的parenthesized index，我们将状态 :math:`q_l` 在指定时间 :math:`n \in [1,N]` 出现定义为 :math:`q_l^n` 。 由于 :math:`q_l^n` 是一个互斥事件，我们可以将概率 :math:`P(X|W)` 改写成，对于任何一个 :math:`n` :

.. math:: P(X|W)=\sum_{l=1}^L P(q_l^n, X|W),    (1)

这里， :math:`P(q_l^n, X|W)` 表示 :math:`X` 被参数 :math:`W` 限制的模型，同时 :math:`x_n` 与状态 :math:`q_l` 关联的概率。 最大化上式可以使用Baum-Welch算法中的前向后向算法循环计算出来。

最大化上式中的 :math:`P(X|W)` 通常使用Viterbi算法来近似。它使用一个MLE评价标准的简化标准。仅仅使用在W中最有可能性的状态序列来计算X。为了明确的说所有可能的路径，上式可以被写成:

.. math:: P(X|W) = \sum_{t_1=1}^L \sum_{t_N=1}^{L}P(q_{l_1}^1, ..., q_{l_N}^N, X|W).

维特比算法会将上式中，所有的求和用"max"操作替代。上市可以近似的表示为:

.. math:: \bar{P}(X|W) = \max_{t_1, ..., t_N}P(q_t^1, ..., q_t^N, X|W)

而这个最大化，可以使用经典的DTW算法求解。对于这种情况，每一个训练向量都唯一的状态转移序列所关联，也就是在每个时间n中，仅仅有一种特殊的 :math:`{q(k) \rightarrow q(l)}` 状态转换，其中 :math:`q \in Q`. 

这两种方法(MLE和Viterbi)，其中的 :math:`P(X|W)` 和 :math:`\bar{P}(X|W)` 可以使用"局部"的付出 :math:`p[q_l^n,x_n|Q_1^{n-1},X,W]` 来递归的计算。其中 :math:`Q_1^{n-1}`表示状态序列在之前所绑定的观察序列 :math:`x_1, ..., x_n-1` 。 为了简化问题，我们通常假设我们使用的HMM模型是一个一阶HMM模型(也就是说，每个状态仅仅和他的前一个状态有关)，同时，声学向量没有相关性(也就是说，X可以在一定条件下被忽略)。这个"局部"付出可以被一些局部参数的集合估算出来， :math:`p[q(l), y_i|q^-(k),W] for i=1,...,I and k,l = 1,...,K` 。符号 :math:`q^-(k)` 和 :math:`q(l)` 表示在两个连续实例中观测到的状态, 其 :math:`\in \mathfrak{Q}` 。在这个Viterbi评价标准中，这些参数可以使用下式被估算:

.. math:: \hat{p}[q(l),y_i|q^-(k)] = \frac{n_{ikl}}{\sum_{j=1}^I \sum_{m=1}^K n_{jkm}}, \forall i \in [1,I], \forall k,l \in [1,K],    (3)

其中 :math:`n_{ikl}` 表示每一个原型向量 :math:`y_i` 在训练的过程中从一个特定的状态 :math:`q(k)` 到 :math:`q(l)` 转移的次数。为了减少参数的数量，上式可以分解为一个激发概率 :math:`p(y_i|q(l))` 和一个转移概率 :math:`p(p(l)|p^-(k))` 。但是，评价标准(1)和估算(3)都没有最小化凑无虑，无论是在词级别，还是在声学向量级别。MLE训练准则中，并没有包括判别性。因此，局部的概率并不是一个对原型向量 :math:`y_i` 的良好度量。也就是使用当前输入向量和指定前置状态，得到最可能当前状态的方法并不是一个良好的度量。事实上，一个更优化的决策应该基于最大后验概率(MAP)。这里，最可能的状态被定义为:

.. math:: l_{opt} = \begin{matrix} argmax \\
            l \end{matrix} p[q(l)|y_i, q^-(k)],     (4)
 
而这个并不基于(1)。

可以很容易的证明，对于贝叶斯概率(4)的估算为:

.. math:: \hat{p}[q(l)|y_i, q^-(k)] = \frac{n_{ikl}}{\sum_{m=1}^{K}} n{ikm}.    (5)

也就是说，对于在词和frame级别的解码错误的最优化准则，应该基于MAP。这个论断和其他语音识别的统计概率的角色在[Nadas et al., 1988]这篇论文中，解释的很清楚。同时，MAP估算对于语言模型非常差的情况，也就是前置词的概率被估算的很差的时候，会让语音识别训练更安全。 但是，这篇论文的结果仅仅在鼓励词识别上是有效的，并不会应用在更低层次的解码过程中(例如frame级别的在连续语音识别中的解码过程)。

下一节中，我们将介绍使用MLP的输出值作为MAP概率的估算。使用MLP，同时保留他的反馈机制，并将输入域扩展到包括上下文，基于一个固定长度的窗口，输出概率可以被生成。这些需要在[Proitz, 1988]的论文中被解释。

MLP的统计推断
-------------

设 :math:`q(k), with \quad k = 1, ..., K` 是MLP的输出单元所关联的不同类别(每一个都和HMM的特定状态 :math:`\mathfrak{Q}` 相关联)。假设
训练在一个以N维矢量化的向量输入 :math:`\{y_{i_1}, ..., y_{i_N}\}` 其中 :math:`y_{i_n} \in \mathfrak{Y}` 。关于MLP参数的训练，通常基
于最小二次误差的评价准则:

.. math:: E = \frac{1}{2}\sum_{n=1}^N \sum_{k=1}^K [g(i_n,k)-d(i_n, k)]^2, (6)

其中, :math:`g(i_n,k)` 表示k单元在给定输入 :math:`y_{i_n}` 时的输出。而 :math:`d(i_n,k)` 表示其输入绑定的输出值，而且如果已知输入属于类别
 :math:`q(l)` ，其与 Kronecker delta :math:`\delta_{kl}` 相等。同样的输入不一定非要映射到同样的class上，其可以有多重表示(矛盾训练)。将求
和展开，对于所有训练数据，(6)式可以被重写成:

.. math:: E=\frac{1}{2} \sum_{i=1}^I \sum_{k=1}^K \sum{l=1}^K n_{ik} \dot [g(i,l) - d(i,l)]^2, \quad (7)

其中, :math:`n_{ik}` 表示 :math:`y_i` 被分类，也就是从 :math:`q(k)` 泛化的次数。 也就是说， 无论MLP的拓扑结构是什么， 最优化的输出值
 :math:`g_{opt}(i,k)` 都可以使用消除偏导版本的E与 :math:`g(i,k)` 做比较得出。 这可以被很容易的证明，这么做，最优化的输出值可以写成

.. math:: g_{opt}(i,k) = \frac{n_{ik}}{\sum_{l=1}^K n_{il}}, \quad (8)

这个式子就是(5)式估算的bayes概率(并不包括转移概率)。但是，这个优化值仅仅能够在MLP含有足够多的参数的时候，而且不在训练中困在局部最小值中，并且需要很
长时间的训练才能达到最小值。

这个结果可以直接使用最小化评价函数就可以获得，而不是来自于模型的拓扑结构。同样的最优值可以通过其他评价函数获得，例如与熵有关，或者相互熵等评价函数。在
这个方法中，使用偏导的原则，也同时可以用BP算法训练MLP。在这个过程中，梯度的估算可以用误差的偏导获得，事实上:

.. math:: \frac{\partial{E}}{\partial{w_{i,j}}} =  \triangledown _{g}^{t} E . \frac{\partial{g}}{\partial{w_{ij}}} , \forall i,j

其中, t表示转置操作。也就是说，在输出空间的最小值( :math:`\triangledown_{g}E=0` )也是参数空间的最小值( :math:`\frac{\partial{E}}{\partial{w_{ij}}=0, \forall i,j}` )。
但是，参数空间的最小值不一定能导致输出空间的最小值，也就是神经网络算法搜索到了一个误差函数的局部最小值。在这个例子中，输出并不会是MAP概率。
事实上，神经网络从来不会保证输出值是一个概率，也就是不保证他们输出值求和为一。一个优雅的做法是将分类函数从sigmoid函数变成softmax函数，也就是对于
任意一个 :math:`i` , 有：

.. math:: g(i,k) = \frac{e^{x(i,k)}}{\sum_{l=1}^{K}e^{x(i,l)}}, \quad (9)

其中 :math:`x(i,k)` 是由输入 :math:`y_i` 经由一系列非线性操作，输出单元 :math:`k` 所产生的输出值。 这个方程可以归纳一个sigmoid函数，
并且与Gibbs分布具有良好的关系。

对于这些实验，我们使用一个离散输入的MLP来做“k中选一问题”的分类其，也就是对于每个类别，仅仅有一个能够输出。对于一个有足够参数的系统，如果训练没有
进入到局部最小值，那么MLP的输出会近似后验概率。 Section 4中会有一些实验证据证明这个观点。

这个结论也可以在连续输入中得到。这在LMSE评价标准(也包括其他标准，例如熵)中的回归理论中可以知道。如果有足够的训练数据，我们对于给定的输入，可以有条件
的期待输出结果。也就是说，这个估算会覆盖 :math:`E[d(t)|v(t)]` ，其中 :math:`v(t)` 代表在时间t的输入向量，而 :math:`d(t)` 和需要的输出相关。
在分类模式下， 由于 :math:`d(t)` 是“K中选一”模型， 我们就有等式 :math:`E[d(t)|v(t)]=P[d(t)|v(t)]` , 也就是对于输入数据的输出类别的概率分布。

由于这些结果和模型的拓扑结构相独立，对于线性判别函数也同样有效。实践中，性能通常受参数数量的制约，也不会保证获得最优解。但是，这种通过观察和最小化LMSE来得到
的判别函数具有一个内在的最好近似， 在最小均方的理论上，贝叶斯概率。

