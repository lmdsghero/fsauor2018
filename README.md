# fsauor2018
### 题目介绍
该题目为《细粒度用户评论情感分析》，来源于“全球AI挑战赛”。https://challenger.ai/competition/fsauor2018
对于题目内容以及任务描述均可从该网站获取，这里不再赘述。
### 成绩
单个模型最好的F1指标为：75.04  
整体20个模型的综合F1指标为：68  
### 数据集
数据集由比赛平台提供。包含105000条训练样本以及15000条测试样本。  
关于数据集的标注可以点击[这里](src/sentiment_analysis_trainingset_annotations.pdf)查看。  
### 环境依赖
tensorflow 1.8.0-gpu
### 整体思路
由于我们需要对一条评论从20个层面进行情感分析，是一个典型的多标签学习任务。我这里通过训练20个4-分类器，对于一个评论文本，
每个分类器对其进行分析。最终20个分类器得到的结果汇总起来即为该模型对这条评论的结果。
### 模型架构
![模型结构](/src/model.png) 
**这里放一个tensorboard的图，这是在训练第三个标签时候各项指标的变化。**  
![tensorboard](/src/tensorboard.png)
### 超参数的选择  
* 学习率: 3e-4  
学习率尝试过1e-4、3e-4、1e-3等，发现取3e-4时F1值和1e-3差不多，但是取3e-4时模型的最终loss更低。
* dropout: 1.0  
dropout尝试过0.5、0.75以及1.0，发现取1.0的时候效果最好，0.5的时候效果最差。
* 句子截断长度：350个词语  
句子截断长度取350个词语时，仅5%的评论被截断，其余不足350个词语的评论在词嵌入的时候补0。
* 词向量维度：300  
词向量维度尝试过250、300、350等，但是对模型的性能提升不大，一般都选用300。（维度过高会延长训练时间）
* batchSize：32
* lstm单元的个数，注意力的大小我这里参考文献[1]的选择。  
 
### 主要难点  
1. **句子风格偏口语化。**  
　　评论来自于美食类网站中店铺下方的评论，评论者为消费者，评论对象为某店铺，因此评论风格偏口语化。  
　　评论中含有较多的表情符号，如^_^，O(∩_∩)O等，也含有较多的emoji表情。同时，也包含一些语法不规范的句式，比如语气词较多，标点符号不规范等。  
　　因此比起时评，新闻或者文学作品，模型更难把握这种风格句式的情感倾向。  
　　尝试过传统的句法分析提取特征[参考文献2]，但是口语化的语句提取效果比较差，因此采用深度学习。  
　　我在预处理阶段，先把繁体字转化为简体字后，把所有非中文字符移除（移除标点以及emoji表情），然后用jieba分词。分词之后，进行embedding（使用gensim中的word2vec库）。*（预处理阶段我并没有去掉停用词）*  
　　*也可以直接下载网络上已经训练好的词向量模型，我这里仅在官方给的评论上进行词嵌入。不过我看这次比赛冠军的[github](https://github.com/chenghuige/wenzheng/tree/master/projects/ai2018/sentiment)中提到他用的是1kw的外部点评数据，效果比较明显。*
2. **个别标签类别不均衡现象严重**  
　　以“排队等候时间”标签为例，在训练集中，有93.60%的评论没有提及某店铺是否容易停车。  
　　*点击[这里](src/labelDistribuion.md)查看每个标签的类别分布。*
3. **对“中性评价”的预测总是最差的**  
　　个人认为对于“中性评价”来说，比起“好评”或者“差评”，很多时候人自己也不太好把握。
4. **有些标签本身就不太容易被消费者提及，或者被“准确”评价**  
　　就“停车便利性”来说，93.59%的评论都没有提及，可能因为很多人都是步行路过去吃饭。  
　　就“菜品-外观”来说，我这里的结果是该标签的F1值是20个标签中最低的（53%左右）。可能是因为每个人的审美都不同，对于菜品外观的评价标准也都不一样，很难把握特征，强行预测的话和随机猜测差不多。但是如果训练集全部为同一个店铺的评论的话，我觉得效果应该能好很多。  

### demo演示
　　训练完20个模型后，我这里简单实现了一个demo。  
　　开一个python的socket进程作为服务端，接收一个待分析的评论文本，将其输入到20个模型中，得到一个(20, 2)的矩阵。
每一行是一个标签的分析结果，结果包含两项：情感倾向以及置信度。最后将结果返回给客户端。  
　　用户在web页面中填写待分析的评论文本，点击“开始分析”按钮，index.jsp将该字符串传递给handler.jsp页面。再由该页面通过调用Analyze.java向服务端发起socket请求，把该字符串发送给服务端进行分析。
分析完毕后接收服务端发来的结果，跳转回index.jsp页面，并将结果显示给用户。  
　　*点击[这里](src/demo.md)查看demo演示效果以及性能分析。*  

### 项目结构
* main.py  
main函数，可以进行训练或者测试模型性能，调用代码：
```
python main.py --type train --label 2 --iter 6600
python main.py --type test --label 1 --id 1287471 
```
　　--iter：训练多少个批次。3300个批次大概训练完一遍训练集，6600个批次训练3遍（此时并没有明显的过拟合）。  
　　--id：模型的序号。
* load_model.py  
加载计算图
* load_batch.py  
读取数据，每次next()返回一个(32, 350, 300)和(32, 4)的张量，由评论文本以及对应的标签转化而来。
* word2vec.py  
词向量的训练
* content.py、zh_wiki.py、langconv.py  
原始评论的预处理，包含繁转简，去除标点、表情以及分词，结果保存到本地。
* demo
    * src
        * loadModel.py
        * load_server.py
        * Analyze.java
        * model
    * WebContent
        * index.jsp
        * handler.jsp  
        
### 可以改进的地方
1. 对于单个标签，可以采用集成学习训练若干个子学习器，最后的结果根据这些子学习器的结果汇总得到。[xueyouluo](https://github.com/xueyouluo/fsauor2018)在模型的最后采用了该集成学习的方法，模型架构类似的情况下，他的模型F1值比我高出5%左右。
感兴趣的同学我推荐采用[Adaboost算法](https://en.wikipedia.org/wiki/AdaBoost)来提高模型性能。
2. 对于类别不均衡的问题，一般可以采用上采样或下采样技术来均衡训练集中正负样本的比例，使之分布接近1：1，但是这么做会改变数据分布。虽然会提高模型在训练集上对于少数类的预测效果，但是同时会降低模型对于多数类的预测效果，并且不能提高泛化性能（因为此时你的训练集并不是从真实数据中无偏采样得到的），总体来看这么做反而会使模型效果变差。  
　　在处理正负样本极度不均衡的问题时，比如10：1以上，可以考虑采用异常检测。对于小于10：1的训练集，Adaboost方法或其他集成学习的方法也可以提高模型的F1值。（但我目前还没实现这一部分）  
3. 训练的时候我发现很多时间都花费在了数据读取、词嵌入方面，也尝试过使用pickle和tfrecorder预先存储词嵌入后的数据，然后直接读取数据而不是读取评论文本。但是由于训练集太大，嵌入后得到的105000\*350\*300的float64张量实在太占硬盘了，遂放弃。目前还没找到合适的方式通过预读取数据来加快模型的训练。  

### 参考文献
[1]姜坤. 基于LSTM和注意力机制的情感分析服务设计与实现[D].南京大学,2018.  
[2]李纲,刘广兴,毛进, 等.一种基于句法分析的情感标签抽取方法[J].图书情报工作,2014,(14).  

### 联系方式  
仓促之中错误在所难免，很多分析都是我自己对这个问题的看法，可能会比较片面，不足之处还请多多指教！欢迎大家提issue一起讨论。  
我的邮箱是: jblei@mail.ustc.edu.cn  
