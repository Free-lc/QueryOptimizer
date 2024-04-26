论文笔记
AutoCE: An Accurate and Efficient Model Advisor for Learned Cardinality Estimation (ICDE'23)
为不同的数据集选择恰当的learned基数估计器
Speeding Up End-to-end Query Execution via Learning-based Progressive Cardinality Estimation (sigmod'23)
motivatetion
文章首先提出了3种方式的基数估计：data-driven, query-driven, and hybrid。数据驱动将基数估计建模为非监督学习，去建模关系表的联合分布；查询驱动将基数估计建模为QueryContent-CardinalityValues的回归问题；混合将同事考虑查询和数据分布。
由于query-driven的估计器有3种优点：1）不需要数据参与：它们只需要查询的采样和结果基数，模型的训练和推理不需要数据参与，对大数据是友好的。模型的升级也对数据库系统的其它功能是透明的（数据access）2）数据隐私性：查询驱动不需要访问关系表，数据隐私性更好。3）更短的推理时间：降低end-to-end的查询执行时间。
通过测试query-driven的估计器是不准确的，尤其是当遇到复杂的joint查询，因为当有更多的谓词参与时估算的错误会传递并放大（其它两种估算起也有这个问题）。
根据这个观察文章将查询驱动的估算其结合到渐进式估计的query re-optimization。1）这是可行的因为所执行操作符的确切基数可以被访问，这直接影响其余操作符的中间结果基数。2）并且这使得我们不需要一个非常精确的初始估算器（复杂的估算器会带来长的推理时间）。
根据例子可以看出来渐进估计可以带来更好的plan。但是会为有效性和高效性带来挑战：有效性：估算器应该使用已经执行谓词的信息来为剩余的谓词提供更精准的预测。高效性：预测模型应该有较短的预测时间（初始模型和渐进优化都会增加查询执行时间）。
为了应对这些挑战文章提出了LPCE，LPCE-I是一个使用小的成本来提供精确的初始估计，LPCE-R是为了渐进式的优化基数估计为了查询执行中剩余的没执行的谓词。
overview
工作流程如下：
1. When a query is submitted, it is sent to LPCE-I for initial cardinality estimations of all its possible sub-plans;
2. Using the initial estimations, the query optimizer chooses a good execution plan based on its plan search algorithm;
3. The chosen plan is executed by the database, and checkpoints are placed at some operators to monitor if the initial estimations are significantly different from the actual cardinalities;
4. Once large estimation error is detected, query execution is paused and LPCE-R is invoked to refine the cardinality estimations for the feasible sub-plans of the remaining operators;
5. Based on the refined estimations, the query optimizer searches a better execution plan for the remaining operators.
 基本思想是：已经执行的谓词结果是可以获得的，并且这个结果可以为之后执行的谓词提供有用的信息来优化基数估计。
设计目标：高准确性，快推断，渐进优化。
高精确度： Query re-optimization 增加了查询执行的开销，为了少的调用re-optimization，初始的估算器应该较为准确，为了这个目的为了初始的估算起采用了node-wise loss function。
快推断：LPCE-I and LPCE-R都会增加推断时间，查询优化需要预估许多子计划的基数，这些通过机器学习模型的推断是非常耗时的。这里采用了轻量化的模型骨架并且使用知识蒸馏来压缩模型。
渐进式优化：为了找到好的执行计划，LPCE-R在更多的谓词被执行时应该高效地减少预测错误。为了这个-R需要充分利用已经执行的子计划信息，特别是它们的真实基数。
基数估计模型
● 文章首先介绍了基于学习模型的基数估计器的通用流程：
  ○ Sample collection：基数估计一般是作为一个回归问题的，输入为执行的查询计划输出为基数值。在构建基于查询的数据库性能估计或优化模型时，训练样本可以从历史执行日志中收集。对于新数据库在“冷启动”阶段（即刚开始运行时，还没有足够的历史数据来训练模型），可以根据底层数据集的关系图随机生成样本查询。
  ○ Feature encoding: 机器学习的模型通常使用向量作为输入，这里面的feature encoding参考的是其它文章的思想，大致意思是使用one hot编码来表示操作符，涉及到的column。

  ○ 模型学习：这里面提到了两个方法，MSCN将plan的特征向量堆积使用多集卷积网络来处理；TLSTM递归的处理查询树结构使用LSTM。MSCN由于忽略了plan的树结构有较大的估算错误，TLSTM具有高的计算复杂读
● 基于SRU的轻量化模型
  ○ 使用SRU待提到LSTM带来更小的内存使用，更快的推理速度，性能的衰减不明显。LSTM有8个矩阵乘法运算而SRU只有3个。
● node-wise Loss Function 这里的思想很好，分解大的loss成为每一个部件的loss从而带来更多的insight
  ○ MSCN和TLSTM都是基于整个plan的real&estimated基数的错误来作为loss的；node-wise则考虑plan中的每个节点
  ○ 这种node-wise有两大好处：更多种类的数据样本；直接针对node的监督。
● 用来压缩的知识蒸馏
  ○ 使用一个复杂的模型来纠正简单的模型
  ○ 
基数的精进模型

query-driven的估计器在应对哪些model没有学习到的会产生基数预测有较大的错误。LPCE-R的思想是使用已经执行的节点来精进基数估计，因为没有执行的节点会依赖于哪些已经执行的节点中间结果。这其中有3个问题需要解决：1.需要从已经执行的sub-plan中提取什么样的数据。2.如何有效地利用这些数据来进行基数的精进。3.在查询的执行过程中如何精进估计器。
● 信息提取的两个模块：已经执行的节点的基数结果的错误估计会很大影响后面的预估，所以真实的基数是重要的；已经执行的子计划的语意文本信息（查询内容）如谓词、joined的列对剩余节点的预估有很大影响。所以信息提取模块需要提取real cardinality和query content需要被提取出来。同样，这个结论也能通过实验来验证。
● 学习的信息融合：




