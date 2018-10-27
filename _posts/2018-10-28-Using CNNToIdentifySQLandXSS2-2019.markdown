---
layout:     post
title:      "Using CNN to identify SQL and XSS 2"
subtitle:   "使用CNN做SQL和XSS的识别 2"
date:       2018-10-28 02:00:00 
author:     "Roche" 
header-img: "img/home-bg.jpg" 
tags:
    - CNN 
---

# 前言

之前写过这个

使用CNN做SQL和XSS的识别

在前一个post里，模型的推理速度大约是100us。  
在这一篇post里，我将对这个模型做一些改进，让这个模型的预测速度更快一点。

# 实际的步骤

首先，我是利用一个类似与word2vec的思路，将数据转化为模型输入的数据的。  
总体的算法是一个循环查找匹配的动作。  
这个过程基于数据量的大小，有很多可以作为优化选项的算法。  

这里，随手利用一个～
不做主体来说

然后，如果仔细观察下CPU利用率的话，  
可以发现，在循环跑session的过程中，CPU并没有拉满。
可见在session运行中，有一些动作会使CPU空转，或者本身预测的过程，CPU就不容易拉起来。

这一点，可以启动多个session，并行处理数据

首先，定义总循环量，和单线程循环量

    #define TRUE_LOOP_TIME 100000
    #define LOOP_TIMES (TRUE_LOOP_TIME / 8)

因为我是使用笔记本的i7 4700mq在跑，所以这里定义8条线程  

然后这里是线程共有的信息  
需要的判定字符串，以及作为最后统计使用的最大时间

    const char* input_line;
    long long total_time;

然后定义线程应该做的事情

-   读取model，载入
-   循环开始
-   填充tensor
-   跑session
-   循环结束
-   记录时间


        void TF_Thraed(void)
        {
            int i;
            struct timeval start, end;

            model_t* model = TF_LoadModel("frozen.pb");
            assert(model != NULL);
            
            gettimeofday( &start, NULL );
            for(i = 0; i < LOOP_TIMES; i ++) {
                if (!TF_FillTensor(model, input_line)) {
                    puts("TF_FillTensor failed");
                    return;
                }
                TF_RunModel(model);
            }
            gettimeofday( &end, NULL );

            total_time += ((end.tv_sec - start.tv_sec) * 1000000) + end.tv_usec - start.tv_usec;

            TF_FreeModel(model);
            pthread_detach(pthread_self());
        }


对应，主线程这里需要做一些修改

创建8条线程，并等待运行结束  
计算出平均时间，打印出来

这里是假设全线程会占用全部的性能。
线程扩充的时间需要平均被每条线程吃掉。
实际上，线程完成的速度会稍有区别，得到的平均时间会稍大。

        int main(int argc, char const *argv[])
        {
            pthread_t tf_thread[8];
            double out_time;
            
            if (argc < 2 || !argv[1] ||!InitVocab()) {
                puts("No input");
                return 1;
            }
            input_line = argv[1];
            total_time = 0;

            pthread_create(&tf_thread[0], NULL, (void *)TF_Thraed, NULL);
            pthread_create(&tf_thread[1], NULL, (void *)TF_Thraed, NULL);
            pthread_create(&tf_thread[2], NULL, (void *)TF_Thraed, NULL);
            pthread_create(&tf_thread[3], NULL, (void *)TF_Thraed, NULL);
            pthread_create(&tf_thread[4], NULL, (void *)TF_Thraed, NULL);
            pthread_create(&tf_thread[5], NULL, (void *)TF_Thraed, NULL);
            pthread_create(&tf_thread[6], NULL, (void *)TF_Thraed, NULL);
            pthread_create(&tf_thread[7], NULL, (void *)TF_Thraed, NULL);

            pthread_join(tf_thread[7], NULL);

            sleep(2);
            out_time = (total_time / 8) / TRUE_LOOP_TIME;
            printf("max time = %f us\n", out_time);

            return 0;
        }

这里还可以把线程对应CPU绑定，或者使用更好的线程同步的方法。
~~post不想写这些~~

最后这里的平均计算时间为17微妙，比上一篇post，提高了6倍左右

![大概这样](https://roche-k.github.io/img/in-post/deep_learnig/2018-10-28-time.png)


# 结语

撸猫撸猫
