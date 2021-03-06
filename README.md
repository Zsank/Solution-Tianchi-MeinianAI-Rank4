# 阿里云美年大健康AI算法大赛Rank4

### 队伍：SYSU幼儿园
初赛：A榜Rank Top5/3152， B榜Rank4/3152

复赛：A榜Rank Top5/150，B榜Rank 4/150

团队成员：sysu-zjw，[Star Wave](https://github.com/mathcbc)，[deathcrow](https://github.com/deathcrow) ，mzs

> 第一次参加比赛，和小伙伴一起边走边学，其实学到的东西很多。\
从成绩来看其实我们的模型是很稳定的，用的是GBDT单模型，另外用了CNN做文本特征的特征工程处理。

### 一、比赛介绍
比赛的数据类型涉及数值、符号和文本，需要我们预测五个高血压相关指标，分别为收缩压、舒张压、甘油三酯、高密度脂蛋白胆固醇、低密度脂蛋白胆固醇 。
评分方法以及数据下载，传送门-->[比赛链接](https://tianchi.aliyun.com/competition/information.htm?spm=5176.100067.5678.2.57bb342dl2OdIC&raceId=231654)

### 二、运行说明
1. 从官网**下载训练**数据并存放到data目录下：
    - meinian_round1_data_part1_20180408.txt
    - meinian_round1_data_part2_20180408.txt
    - meinian_round1_train_20180408.csv
2. 执行顺序：
    - 在`scr`文件下执行`main.py`\
    **Noted**：NLP部分用的是第二个模型`model_v2_double_cnn`，不包含在`main.py`里面，需要自己去train。

### 三、模型思路
可以优先参考：[天池数据比赛top5经验分享](https://zhuanlan.zhihu.com/p/38977718)
1. **预处理**
    - **同一个用户同一特征下有多个结果**\
    读取时容易犯的错，直接取首项或尾项会有信息缺失。我们用List来存储这些结果，然后观察List的分布再给出解决方案。

    - **文本与数值都有的字段**\
    形如 pd.Series(["3.24","+","阴性","3.2.2"]，这种情况不敢批量替换文本，因为不能保证相同文本在各个字段中的分布一致\
    针对这个情况：我们通过EDA找到并重点关注——（a）target指标异常人群经常检查的字段；（b）缺失率最低的字段；（c）含高度相关关键词的文本字段。对于这接近40个字段肉眼观察分布并做预处理。
    
    - **高缺失率**\
    个人想法，欢迎指教：GBDT模型只关注排序，缺失只是一种特殊的数据类型。当数据的丰富性集中在少量样本时（缺失率高、单一值比例高），分裂后的其中一个方向包含的样本量会很少，这个方向的数据如果偏移，造成的影响会很大。在这种考量下，我们可以在调参的时候通过`min_child_weight` 来决定这些字段是否入模，而不是预处理的时候直接drop掉，这样模型会有更高的上限。
    

2. **文本处理**\
    **初赛**用的是词向量 + CNN + 提取全连接层 = 作为LightGBM的新特征
    - 针对约40个高度相关的文本变量建模
    - 剔除停留词等无关词语，重点关注主谓宾
    - 针对长短文本以及主谓宾结构选取不同size的filter，最后concat到一起
    - 词向量长度d(d=64)，长文本字段采用3×d，4×d，5×d这三种卷积层，短文本字段采用1×d，2×d，3×d这三种卷积层，在全连接层的时候进行concat

3. **调参**
    + 个人认为受益于体检数据的稳定性和准确性，GBDT模型越复杂呈现的效果越好，直到达到饱和点。线下和线上前期同单调，后期只相差0.001以内，可以说CV做得很稳定了
    + CNN这边需要注意的是：如果直接抽隐含层其实是有leak在里面的，可以考虑做个Kfold。但我们是后期加入的CNN，成绩也比较理想，就没有做这个步骤
    + 过拟合问题：我们预处理没有drop掉高缺失率字段，其实很可能出现过拟合的问题。但我们的CV结果后期一直和A榜结果只相差0.001以内，所以在过拟合上没有采取过多的措施，初赛B榜的成绩也验证了我们的判断\
    个人分析：是因为高缺失率字段属于极少数体检项目，会得到更多的重视，所以数据更为精准，没有把模型带偏。

### 四、总结
美年这个比赛数据有点脏乱，但妥当处理后会发现数据的稳定性很好，是很典型的工业界数据挖掘。\
预处理的时候基本是边观察数据分布边处理，所以代码较为混乱，暂时也没优化的打算，欢迎有心人pull request。
