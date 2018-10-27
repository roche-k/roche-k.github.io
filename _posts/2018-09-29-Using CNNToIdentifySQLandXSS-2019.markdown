---
layout:     post
title:      "Using CNN to identify SQL and XSS"
subtitle:   "使用CNN做SQL和XSS的识别"
date:       2018-09-29 22:00:00 
author:     "Roche" 
header-img: "img/home-bg.jpg" 
tags:
    - CNN 
---

来篇正式点的Post吧  
~~燃燃燃然~~

# 前言

最近在FreeBUF上看到这样一篇文章  
[基于卷积神经网络的SQL注入检测](http://www.freebuf.com/articles/web/176709.html)

我觉得CNN算法就其本身而言，应用与NLP是没有问题的。  
但是这个检测率应该可以再提升一点。  
因为SQL，XSS的注入方法，就语法上而言，比NLP要简单一些。

还可以用C写一个检测程序。  
这样就更容易让这个模型跑在实际的环境。

而且，样本在送检之前，首先做了循环解码，再做归一化处理。  
循环解码是需要的，但是归一化处理不一样。  
安全性不论，每条数据先要跑正则再送检，这个效率要比传统的正则匹配法低一些。

那么，让我在巨人的肩膀上，再做一些小改进。

# 理论

对于通过CNN来做NLP处理的论文，国内外都可以找到很多。

大致的原理是这样：

![大概这样](https://roche-k.github.io/img/in-post/deep_learnig/2018-09-29-NLP-CNN.png)

句子->词向量->卷积层->池化层->链接层

简单的说，就是把句子内的词转化为词向量，后面接单层的CNN。


# 训练数据

首先，我们需要训练集，对照集，检测集
这个可以用github的一些数据。这里，就使用前一文中提到的数据集来检测。

首先，循环读取每一条数据，对每条数据循环解码，打上标签，最后存档。  
这里的UrlDecode()完成循环解码的功能。

    for each_line in sql_file:
        str=UrlDecode(each_line)
        f.writelines('sql' + '\t' + str + '\n')
        count += 1
    print("sql end at %d" % count)

对于训练集的第一条数据  

    1%29%20AND%205628%3D4794  

经过改写后，变成这样  

    sql	1)and5628=4794

全部改写完成后，就可以用之前提到的模型训练了

整个训练在6次迭代后完成。
这个是tensorboard的损失图像。

![大概这样](https://roche-k.github.io/img/in-post/deep_learnig/2018-09-29-tensotboard.png)

最后使用test集，可以达到99%以上的检测率。  
混淆矩阵也很好看。

![大概这样](https://roche-k.github.io/img/in-post/deep_learnig/2018-09-29-matrix.png)

# 检测

之前说过，我们需要让这个模型能应用于实际的生产中。  
那么，结合Tensorflow的C库，来写一个检测程序。  
关于Tensorflow C库的使用，有很多现有的例子，就不多写了。

大致的流程是这样

    int main(int argc, char* argv[])
    {
        if (argc < 2) return 1;

        // pre load
        tf_model_t* model = LoadTFModel("./tf_model.pb");
        FillTensor(argv[1]);

        printf("Model Returns: %d\n", RunTFModel(model));

        FreeTFModel(model);
        return 0;
    }

这样，就完成了使用C来检测注入的目标。  
如果是需要重复的使用，可以在系统启动时调用LoadTFModel()，检测时只需调用FillTensor()和RunTFModel()即可。  
可以省略掉加载的时间。  
不用时调用FreeTFModel()清空资源。

如果需要看下单次处理的速度之类。  
在FillTensor()和RunTFModel()之间加个循环，前后记录时间，转double除一下就可以。

我在i7-2700上跑了50万次，平均不到100微秒。  
大致1秒1万-2万多次的样子。  
有时间调调参，或者使用GPU的话应该会更高。

# 结语

撸猫撸猫
