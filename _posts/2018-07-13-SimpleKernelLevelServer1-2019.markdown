---
layout:     post
title:      "A simple kernel-level server 1"
subtitle:   "一个简单的内核级服务器"
date:       2018-07-13 00:49:00
author:     "Roche"
header-img: "img/home-bg.jpg"
tags:
    - FreeBSD
    - Kernel
    - Server
---

## 前言


[跳过废话](#build)

FreeBSD上的Rootkit终于写好啦～  

本来想着在一个不是特别熟的内核环境，写一个内核程序，会比较慢。   
~~实际上是相当慢。~~  
~~vnode很烦，内存空间很烦~~  
但是终于完成了，撒花～～～～

最近看到，很久之前FreeBSD hackers在讨论做的一个内核层的http服务器，感觉挺有意思的。  
然后打算能不能自己写一个。  

当然不是向过去的代码里添加https模块，   
当然也不是一开始从头造轮子，做一个类似的。

从一个简单的聊天服务器开始吧。  
~~几乎确定会写一点点，然后弃坑。~~


<p id = "build"></p>
---

## 正文

作为一只服务器

第一步，需要接受信息，这里就需要从BSD的协议栈上，把原有的处理函数hook下来。  

这是一个很麻烦的事情。
hook下TCP UDP报文，判定需求的属性，然后根据报文不同的类型，采取不同的策略。  
并决定是否将数据通入服务器函数内，或者交给原有函数，或者丢弃报文。

总之，是一个几乎需要复制原有处理函数的工作。
还需要加上服务器处理函数。  
~~确定会弃坑~~

那么，先写一个ICMP的hook练手吧，毕竟时间挺晚了，明天还要上班。  
好像和服务器没什么关系，并且也没打算一次写完。
所以打算看服务器应该怎么写的观众可以右上角了。

既然是需要hook下协议处理函数，首先就是这个函数的注册方法。

ICMP长这样

    {
        .pr_type =		SOCK_RAW,
        .pr_domain =		&inetdomain,
        .pr_protocol =		IPPROTO_ICMP,
        .pr_flags =		PR_ATOMIC|PR_ADDR|PR_LASTHDR,
        .pr_input =		icmp_input,
        .pr_ctloutput =		rip_ctloutput,
        .pr_usrreqs =		&rip_usrreqs
    },

TCP

    {
        .pr_type =		SOCK_STREAM,
        .pr_domain =		&inetdomain,
        .pr_protocol =		IPPROTO_TCP,
        .pr_flags =		PR_CONNREQUIRED|PR_IMPLOPCL|PR_WANTRCVD,
        .pr_input =		tcp_input,
        .pr_ctlinput =		tcp_ctlinput,
        .pr_ctloutput =		tcp_ctloutput,
        .pr_init =		tcp_init,
        .pr_slowtimo =		tcp_slowtimo,
        .pr_drain =		tcp_drain,
        .pr_usrreqs =		&tcp_usrreqs
    },

UDP

    {
        .pr_type = 		SOCK_DGRAM,
        .pr_domain = 		&inetdomain,
        .pr_protocol = 		IPPROTO_UDP,
        .pr_flags = 		PR_ATOMIC|PR_ADDR,
        .pr_input = 		udp_input,
        .pr_ctlinput = 		udp_ctlinput,
        .pr_ctloutput = 	ip_ctloutput,
        .pr_init = 		udp_init,
        .pr_usrreqs = 		&udp_usrreqs
    },

然后，做出一个只会打印的函数函数

    void
    icmp_input_hook(struct mbuf *m, int off)
    {
        uprintf("hooked iemp\n");
    }

再之后，把这个函数挂上去

    ...
    ase MOD_LOAD:
		inetsw[ip_protox[IPPROTO_ICMP]].pr_input = icmp_input_hook;
		break;
    ...

然后，理论上所有的ICMP包就这样被截住了。  
~~一定会出问题~~  

如果没弃坑，下次再补充ICMP递交回去多内容吧。