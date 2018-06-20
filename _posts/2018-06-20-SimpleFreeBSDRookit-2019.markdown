---
layout:     post
title:      "A simple try on FreeBSD rootkit"
subtitle:   "一次简单的FreeBSD内rootkit的尝试"
date:       2018-06-20 16:36:00
author:     "Roche"
header-img: "img/home-bg.jpg"
tags:
    - FreeBSD
---

## 前言


[跳过废话](#build)


最近在忙着将防篡改软件移植到FreeBSD上。

对比Linux，FreeBSD的内核版本较少，可以直接在内核层内做个拦截。  

> Linux如果这样做的话，需要一直适配内核版本，这个很麻烦，大致是这两种思路：
>
> 每个linux内核版本都编译适配一下，对不同的内核版本适用不同的ko，需要无意义的编译很多次；  
> 对应不同的内核型号，产品在用户处条件编译，有各种风险。
>
> 据说也可以做一个系统无关的elf文件运行。但这个的实现看起来很麻烦。

看起来安全一些，不易被破坏。

那么首先做一个监控open的hook吧

参考这个 [BSD下第一个syscall hook，监视SYS_open的情况](http://www.cnblogs.com/bits/archive/2009/05/19/BSD_SYS_open_hook.html)

<p id = "build"></p>
---

## 正文

首先是一些系统调用有关的文件

 /usr/src/sys/kern/sys_generic.c  
 /usr/src/sys/kern/vfs_syscalls.c  
这两个有这次调用需要的sys_open的函数体

/usr/src/sys/sys/syscall.h  
这里是调用号

首先，需要一个处理load的函数，类似这样

    static int
    load(struct module *module, int cmd, void *arg)
    {
        int error = 0;
        switch (cmd) {
        case MOD_LOAD:
            uprintf("Hello, world!\n");
            break;
        case MOD_UNLOAD:
            uprintf("Good-bye, cruel world!\n");
            break;
        default:
            error = EOPNOTSUPP;
            break;
        }
        return(error);
    }

> 来自 Designing BSD Rootkits

据说，也可以省略注册处理函数这步,但是为了安全，还是加上更好
>Actually, this isn’t entirely true. You can have a KLD that just includes a sysctl. You can also dispense with module handlers if you wish and just use SYSINIT and SYSUNINIT directly to register functions to be invoked on load and unload, respectively. You can’t, however, indicate failure in those.


为了load后使用自定义的open_hook代替普通的open，需要这样改进

    switch (cmd) {
        case MOD_LOAD:
        sysent[SYS_open].sy_call = (sy_call_t *)open_hook;
        break;
        case MOD_UNLOAD:
        sysent[SYS_open].sy_call = (sy_call_t *)sys_open;
        break;
        default:
        error = EOPNOTSUPP;
        break;
    }


这样，在load的情况下，就会将SYS_open的处理函数变为open_hook了，卸载时自动恢复。

然后，为了打印open的参数，需要这样

    /* system call hook. */
    static int
    open_hook(struct thread *td, register struct open_args *uap)
    {
    // 输出 open 打开的文件到控制台
    uprintf("  SYS_open: \"%s\", flags: %d, mode: %X\n", uap->path, uap->flags, uap->mode);
    
    return (sys_open(td, uap));
    }

然后，编译加载，大概这样

    # ls .
    SYS_open: ".", flags: 1048576, mode: 0
    SYS_open: ".", flags: 1048576, mode: 0
    SYS_open: ".", flags: 1179652, mode: 0

可见新加入的组件已经开始工作了

