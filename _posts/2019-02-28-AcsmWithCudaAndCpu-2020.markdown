---
layout:     post
title:      "Using cuda to accelerat aho-corasick"
subtitle:   "使用CUDA加速AC"
date:       2019-02-28 03:00:00 
author:     "Roche" 
header-img: "img/home-bg.jpg" 
tags:
    - CUDA 

---

最近在学CUDA，作为一个初期post发一下。  
算法Dalao们可以直接略过这个。

## 原理

首先，这个的原理我觉得不需要多说，不清楚的可以左转搜索引擎。  

从整个算法实现上来说，我们要做的事情应该有以下几个

-   确定要匹配的字符串，对字符串做最先的预处理
-   将字符串转化为模式树
-   使用这个树对输入数据做匹配

一般情况下，也是最先想到的实现方法，我们可以用内存结构实现模式树。

## CPU计算

为了节省时间，我们可以用这篇的方法实现

[Multi-Pattern Search Engine](https://github.com/yaoweibin/aho_corasick_state_machine.git)

当然，我们的最终目的是需要获取平均单次运行时间，所以需要对源码做一定的修改

在这里，直接贴一下git diff的结果

    diff --git a/acsm.c b/acsm.c
    index 7c7d7d9..7b4d666 100644
    --- a/acsm.c
    +++ b/acsm.c
    @@ -1,5 +1,10 @@
    
    #include <stdarg.h>
    +#include <sys/types.h>
    +#include <sys/stat.h>
    +#include <sys/time.h>
    +#include <fcntl.h>
    +#include <unistd.h>
    #include "acsm.h"
    
    
    @@ -417,9 +422,10 @@ int acsm_compile(acsm_context_t *ctx)
    }
    
    
    -int acsm_search(acsm_context_t *ctx, u_char *text, size_t len)
    +unsigned long acsm_search(acsm_context_t *ctx, u_char *text, unsigned long len)
    {
        int state = 0;
    +    unsigned long res = 0;
        u_char *p, *last, ch;
    
        p = text;
    @@ -435,30 +441,45 @@ int acsm_search(acsm_context_t *ctx, u_char *text, size_t len)
            state = ctx->state_table[state].next_state[ch];
    
            if (ctx->state_table[state].match_list) {
    -            return 1;
    +            res += 1;
            }
    
            p++;
        }
    
    -    return 0;
    +    return res;
    }
    
    #if 1
    
    -char *test_patterns[] = {"hers", "his", "she", "he", NULL};
    -char *text = 
    -"In the beginning God created the heaven and the earth. \n" \
    -"And the earth was without form, and void; and darkness was upon the face of the deep. And the Spirit of God moved upon the face of the waters. \n" \
    -"And God said, Let there be light: and there was light.\n" \
    -"And God saw the light, that it was good: and God divided the light from the darkness.\n" \
    -"And God called the light Day, and the darkness he called Night. And the evening and the morning were the first day.\n";
    -
    +char *test_patterns[] = {"AAA","BBB","CCC","DDD","EEE","FFF", NULL};
    
    int main() 
    {
        u_char         **input;
        acsm_context_t  *ctx;
    +    int                 fd;
    +    char                *path="./data.txt";
    +    char                *buffer = NULL;
    +    unsigned long       file_size;
    +    unsigned long       res;
    +    float               total_time = 0;
    +    struct stat         f_stat;
    +    struct timeval             start;
    +    struct timeval             end;
    +
    +    if (stat(path, &f_stat) < 0) return -1;
    +    file_size = f_stat.st_size;
    +
    +    fd = open(path, O_RDONLY);
    +    buffer = malloc(file_size);
    +    if (!buffer) {
    +        close(fd);
    +        return -1;
    +    }
    +    res = read(fd, buffer, file_size);
    +    close(fd);
    +    printf("read %ld \n", res);
    
        ctx = acsm_alloc(NO_CASE);
        if (ctx == NULL) {
    @@ -485,14 +506,23 @@ int main()
            return -1;
        }
    
    -    if (acsm_search(ctx, (u_char *)text, acsm_strlen(text))) {
    -        printf("match!\n");
    -    }
    -    else {
    -        printf("not match!\n");
    +    float time_added;
    +    for(int i = 0; i < 20; i++) {
    +        gettimeofday(&start, NULL);
    +        res = acsm_search(ctx, (u_char *)buffer, acsm_strlen(buffer));
    +        gettimeofday(&end, NULL);
    +        time_added = end.tv_sec * 1000000 + end.tv_usec - 
    +            start.tv_sec * 1000000 - start.tv_usec;
    +        printf("matched %ld time %f us \n", res, time_added);
    +        total_time += time_added;
        }
    +    total_time = total_time / 20;
    +    printf("average_time %f ms\n", total_time / 1000);
    +    
    +
    
        acsm_free(ctx);
    +    free(buffer);
    
        return 0;
    }
    diff --git a/acsm.h b/acsm.h
    index 3d238e8..bb559ac 100644
    --- a/acsm.h
    +++ b/acsm.h
    @@ -70,6 +70,6 @@ void acsm_free(acsm_context_t *ctx);
    
    int acsm_add_pattern(acsm_context_t *ctx, u_char *string, size_t len); 
    int acsm_compile(acsm_context_t *ctx);
    -int acsm_search(acsm_context_t *ctx, u_char *string, size_t len);
    +unsigned long acsm_search(acsm_context_t *ctx, u_char *string, size_t len);

然后，新建一个100M整的文件，写入随机字符，使用编译后的结果搜索目标字符串，打印运行时间及搜索到的次数。

运行结果

    read 104857600 
    matched 9830901 time 434649.000000 us 
    matched 9830901 time 428849.000000 us 
    matched 9830901 time 428837.000000 us 
    matched 9830901 time 428797.000000 us 
    matched 9830901 time 428946.000000 us 
    matched 9830901 time 428784.000000 us 
    matched 9830901 time 429331.000000 us 
    matched 9830901 time 428847.000000 us 
    matched 9830901 time 428949.000000 us 
    matched 9830901 time 429180.000000 us 
    matched 9830901 time 428823.000000 us 
    matched 9830901 time 429086.000000 us 
    matched 9830901 time 428966.000000 us 
    matched 9830901 time 428789.000000 us 
    matched 9830901 time 428868.000000 us 
    matched 9830901 time 428736.000000 us 
    matched 9830901 time 428731.000000 us 
    matched 9830901 time 428813.000000 us 
    matched 9830901 time 429733.000000 us 
    matched 9830901 time 428945.000000 us 
    average_time 429.232941 ms

可见在CPU下，我们使用AC做匹配的时间大约是430ms。

## GPU计算

实际的代码参考这里

[cuda-aho-corasick-wu-manber
](https://github.com/iassael/cuda-aho-corasick-wu-manber)

这里，我们开8个block，每个block给1024个thread。  

实现步骤与CPU相似。
不同的是，在这里我们需要把实际计算放进GPU。

在GPU的kernel函数和CPU内是相近的，简单的贴一下

    for ( col = start_t; ( col < stop_t && col < n ); col++ ) {
		while ( ( s = d_stat[r_t * e_patch + (d_text[col] - (unsigned char)'A')] ) == -1 ) {
			r_t = d_state_s[r_t];
        }
		r_t = s;
		d_out[col] = d_state_f[r_t];
	}

然后多次运行，取平均时间

    matched 9830901 time 160.637253ms
    matched 9830901 time 105.985184ms
    matched 9830901 time 45.960064ms
    matched 9830901 time 27.152800ms
    matched 9830901 time 26.009727ms
    matched 9830901 time 24.208927ms
    matched 9830901 time 24.292896ms
    matched 9830901 time 24.148319ms
    matched 9830901 time 23.329664ms
    matched 9830901 time 22.742176ms
    matched 9830901 time 22.847040ms
    matched 9830901 time 22.818592ms
    matched 9830901 time 22.322432ms
    matched 9830901 time 22.271936ms
    matched 9830901 time 22.363104ms
    matched 9830901 time 22.382561ms
    matched 9830901 time 22.337696ms
    matched 9830901 time 22.309536ms
    matched 9830901 time 22.325279ms
    matched 9830901 time 22.421057ms
    matched 9830901 time 22.334017ms
    matched 9830901 time 22.304640ms
    matched 9830901 time 22.332064ms
    matched 9830901 time 22.436607ms
    matched 9830901 time 22.344608ms
    matched 9830901 time 22.289503ms
    matched 9830901 time 22.351168ms
    matched 9830901 time 22.335360ms
    matched 9830901 time 22.355295ms
    matched 9830901 time 22.310656ms
    average time 22.389656ms

可见单次的匹配时间在22ms左右

## 结论

首先，贴一下硬件平台

-   CPU: AMD Ryzen 5 1600X
-   GPU: Nvidia GeForce GTX 1060

在这个平台上，使用GPU做计算的时间是22ms，使用CPU做计算的时间是230ms。  
整体速度提高了10倍左右。  

完结撒花~~~

