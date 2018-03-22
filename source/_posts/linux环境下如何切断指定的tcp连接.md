---
title: linux环境下如何切断指定的tcp连接
date: 2018-03-22 16:12:13
categories: Technology
tags: [linux, network, tcp]
description:
toc: true
---
# 背景

调试Janus的时候，为了模拟websocket重连的场景，需要单独切断已经存在的websocket连接。
概括一下场景：
1. tcp连接
2. 已经建立
3. 不能直接关闭服务进程

一般面对这种和连接相关的问题，第一想法就是iptables；
不过因为iptables的禁用对已经建立的tcp连接无效（这是由tcp的机制决定的），因此在这个场景中并不适用。

# 解决过程

搜索引擎是最好的老师，Google一下~
不过这次略有些不顺，使用"切断""tcp连接"等作为关键词的时候并没有找到适用的解决方法。唯一一个明显适用这个场景的论坛提问里，大家讨论了3页也没有找到合适的解决方案...

然后试着使用英文关键词搜索了下("close""tcp connection""linus")，顺利查到答案（=´∇｀=）
大受好评的是[tcpkill](https://linux.die.net/man/8/tcpkill)和[killcx](http://killcx.sourceforge.net/)。前者是一个命令行工具，后者是一个perl脚本。

我这次使用的是tcpkill。

## tcpkill 介绍

tcpkill属于网络嗅探工具包dsniff，因此使用时安装dsniff就可以了。
```
# for CentOS
yum -y install dsniff --enablerepo=epel

# for Ubuntu
sudo apt-get install dsniff
```

在命令中使用man查看的话，可以看到如下介绍:
```
Name
tcpkill - kill TCP connections on a LAN

Synopsis
tcpkill [-i interface] [-1...9] expression

Description
tcpkill kills specified in-progress TCP connections (useful for libnids-based applications which require a full TCP 3-whs for TCB creation).

Options
-i interface

Specify the interface to listen on.
-1...9
Specify the degree of brute force to use in killing a connection. Fast connections may require a higher number in order to land a RST in the moving receive window. Default is 3.

expression
Specify a tcpdump(8) filter expression to select the connections to kill.
See Also
dsniff(8), tcpnice(8)

Author
Dug Song <dugsong@monkey.org>
```

## tcpkill 实际使用

我要关闭的是一个websocket连接，考虑到服务器的websocket端口其他client也会使用，所以决定通过指定client的发送端口来关闭连接。
首先使用netstat得到远端端口：
```
# 8188 是服务器的websocket端口
netstat -n |grep 8188
```
结果如下：
```
tcp        0      0 server_ip:8188     client_ip:54774     ESTABLISHED
```

接着使用tcpkill关闭查到的端口(54774)
```
# "-i" 后面的bond0是网卡，可以通过ifconfig找到
# "-9" 表示关闭连接的迫切程度，越大表示越强制，默认是3
# "port 54774" 是指定连接的表达式，这个和iptables差不多
tcpkill -i bond0 -9 port 54774
```

运行该命令后命令行提示`tcpkill: listening on bond0 [port 54774]`，然后挂起。
从服务器日志可以观察到连接并没有立刻中止，而是在服务器和客户端发生网络交互之后（命令行这边会打印相关交互信息），连接顺利中断(・´ω`・)

在tcpkill挂起的bash那边使用`ctrl+c`中止运行之后，指定的端口就又可以使用啦。

# 延申

到这里我的需求就全部完成了，不过还有一些问题没有解决；比如说tcpkill和killcx的区别，比如说它们切断tcp连接的原理，这里为有需要或者有兴趣的同学科普一下.(｡￫‿￩｡)

## tcpkill/killcx切断tcp连接的原理

实际上这些工具都是通过模拟发送RST包来强制关闭tcp连接的，RST标示复位、用于关闭异常的tcp连接。
但是发送RST包需要知道SEQ（Sequence Number）和ACK（Acknowledgment Number）号；
这里就体现出了tcpkill和killcx的区别啦~也就是下文要说的tcpkill的局限性所在。

## tcpkill的局限性

从“实际使用”中，大家也看到了，tcpkill不会立刻关闭已经建立的连接；而是在连接发生网络交互之后才关闭的。
这就是tcpkill的断流策略，它会监听指定的端口；发现符合条件的连接报文之后，根据报文中得到的SEQ/ACK号模拟RST包然后发送，从而断流。

在本文的使用场景中，这种策略是没有问题的；
但是在实际场景中，遇到的关闭不了的tcp连接，往往是“非活跃”的；这时tcpkill就起不到作用了。

不过，程序员是有办法的~

## 切断非活跃tcp连接的备选工具

现成的答案就是killcx。不同于tcpkill守株待兔的策略，killcx选择主动出击；
killcx伪造了一条tcp请求，发送给服务端；然后另一端会回送一个ACK报文，其中携带了正确的SEQ/ACK号；然后killcx就能顺利断流啦~

ps：不过写这篇博文查资料的时候看到有群众报告killcx没能成功关闭连接；然后一位兄台开创性地提出了tcpkill和killcx一起使用的建议；很有创见。

## 拓展阅读

1. [How to forcibly kill an established TCP connection in Linux](http://rtomaszewski.blogspot.sk/2012/11/how-to-forcibly-kill-established-tcp.html)

    我是在一个类似于stack overflow的英文网站查到tcpkill的；回头看了下那里的答案，是来源于一个英文blog；刚刚点开看了一下，里面的细节描述和清晰，每一步操作都有图示；英语ok的同学可以去看一看

2. [关于FIN_WAIT1](https://huoding.com/2014/11/06/383)

    这篇文章介绍了tcp连接关闭的一些细节，值得一看

3. [如何干掉一条tcp 连接（活跃/非活跃）](https://yq.aliyun.com/articles/59308)

    这篇文章重点突出了一个“莽”，作者遇到“非活跃”的连接问题，偏偏又只搜到了tcpkill这个工具，默默选择了查看源码/修改源码/重新编译的方法，硬是把tcpkill做成了killcx的效果，也是很厉害了。
