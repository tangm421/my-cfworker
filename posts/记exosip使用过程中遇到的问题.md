# 1. 记exosip使用过程中遇到的问题

`exosip`是在基于`osip`之上再封装的更高级API的GPL开源库，`osip`库只提供sip协议栈状态机管理以及细粒度的接口，不提供任何网络方面的相关组件。`exosip`则基于`osip`提供了udp、tcp、tls、dtls协议的网络组件，并封装了更便于使用的接口。

exosip官网：[http://savannah.nongnu.org/projects/exosip/](http://savannah.nongnu.org/projects/exosip/)

## 1.1. 宏 ENABLE_MAIN_SOCKET
在exosip 5.1.2版本中使用接口`eXosip_listen_addr`监听TCP端口时，系统中查询不到指定端口被监听的任何信息，这是因为该版本中有一个宏将TCP套接字的绑定和监听进行条件限制了，需要在编译时指定该预编译宏（默认是未指定）。5.0.0版本中并无此问题。  
对于UDP则给定什么端口就监听什么端口。

使用该预编译宏之后，指定的端口会被绑定并监听，但是由exosip主动发起的sip消息使用的却是一个随机的端口号，并不是指定的端口号，包括5.0.0版本一样会有此问题。关于这一点在exosip的源代码有简短的交代：  
大意是exosip中的tcp消息进出使用的是两个不同的socket。
![tcp_tl_send_message](https://images.cnblogs.com/cnblogs_com/tangm421/1893241/o_201203101535tcp_tl_send_message.png)

## 1.2. 宏 SOCK_CLOEXEC
在exosip 5.1.2版本中加入了这个预编译宏，暂且先不详尽关于此宏的作用。5.0.0中没有此宏。