---
layout:     post
title:      "A simple kernel-level server 2"
subtitle:   "一个简单的内核级服务器"
date:       2018-07-26 01:45:00
author:     "Roche"
header-img: "img/home-bg.jpg"
tags:
    - FreeBSD
    - Kernel
    - Server
---

## 前言


[跳过废话](#build)

总觉得睡不着。  
~~那么就完善下之前写的通讯协议的rootkit麻痹下~~


<p id = "build"></p>
---

## 正文

上一次，最终我获取到了一个简单的把所有ICMP包置空的函数。  
这个函数是不完美的。  
但是至少开启了通信协议rootkit的头。

那么，这次加装一些代码，让通信变得正常。  
并且，为了以后加入一些简单多服务器功能，这次就用UDP的协议。

之前说过，UDP长成这个样子

    {
        .pr_type = 		    SOCK_DGRAM,
        .pr_domain = 		&inetdomain,
        .pr_protocol = 		IPPROTO_UDP,
        .pr_flags = 		PR_ATOMIC|PR_ADDR,
        .pr_input = 		udp_input,
        .pr_ctlinput = 		udp_ctlinput,
        .pr_ctloutput = 	ip_ctloutput,
        .pr_init = 		    udp_init,
        .pr_usrreqs = 		&udp_usrreqs
    },

那么，要处理的就是这个函数指针 pr_input，这个原先存放的是udp_input，现在我要把它变成我自己的函数。

首先，必不可少的

    case MOD_LOAD:
            /* Replace icmp_input with icmp_input_hook. */
            inetsw[ip_protox[IPPROTO_UDP]].pr_input = udp_input_hook;
            break;

把pr_input拿下

然后开设设计

    int
    udp_input_hook(struct mbuf **mp, int *offp, int proto);

首先，处理一下作为传参进入的iphlen长度

    if (iphlen > sizeof (struct ip)) {
		ip_stripoptions(m);
		iphlen = sizeof(struct ip);
	}

为了保证之后有UDP信息，UDP头和IP头的长度一定要保证下来。  
剩余长度不满足UDP头的话，接下来就没办法处理了，所以这里把数据返回。

	ip = mtod(m, struct ip *);
	if (m->m_len < iphlen + sizeof(struct udphdr)) {
		if ((m = m_pullup(m, iphlen + sizeof(struct udphdr))) == NULL) {

			UDPSTAT_INC(udps_hdrops);
			return (IPPROTO_DONE);
		}
		ip = mtod(m, struct ip *);
	}

然后，获取UDP头

	uh = (struct udphdr *)((caddr_t)ip + iphlen);

再剥离头

	m->m_len -= iphlen + sizeof(struct udphdr);
	m->m_data += iphlen + sizeof(struct udphdr);

这样，当前m_data的位置就是UDP数据区的位置了  
把它打印出来

	printf("load=%s \n", m->m_data);

然后把m_buf复原

    m->m_len += iphlen + sizeof(struct udphdr);
	m->m_data -= iphlen + sizeof(struct udphdr);

返回给原处理函数

	return udp_input(mp, offp, proto);

这样，就完成了把每次pr_input都打印出的一个内核协议的rootkit。

接下来，编译打入，就可以通过内核打印查看UDP内容了。  
这样，就获得了一个非阻断性的内核层recive_from()。

以后有时间再写吧。