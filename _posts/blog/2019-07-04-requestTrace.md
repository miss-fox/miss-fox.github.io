---
layout: post
title: request trace and tcp/ip
description: request分析以及tcp/ip协议
category: blog
---

> * 从url输入到后端server经历了哪些过程？
> * 这些过程在实际开发中是怎么配置的？怎么做的网络优化？
> * 爬虫用的代理socks4,socks5,http，https区别是什么？vpn？
> * 七层负载均衡vs四层负载均衡？
> * 怎么通过demo实例化？

听了一次网络链路优化的疑问，梳理这些问题的知识点整理一下。

## OSI模型

![WechatIMG528](https://raw.githubusercontent.com/miss-fox/image/master/WechatIMG528.png)

![WechatIMG529](https://github.com/miss-fox/image/blob/master/WechatIMG529.png?raw=true)

发送过程就是封装包首部，上一层对下一层来说都只是data包，接收端就对应解包。

* 应用层，基于应用的协议，开发常用的ssh,http,telnet,smtp,ftp

* 传输层，

  - 包首部主要源端口，目的端口。
  - 主要有tcp(面向连接，发送之前会三次握手成功建立之后发送数据，可靠连接，有丢包重发，顺序控制等机制)，udp（无连接，快，即时性高，dns，ip电话）。
  - 而在操作系统层面是 socket，应用程序通过套接字来实现，这里的端口说明：对于服务端来说，服务端口一般是固定的，比如web的80,https的443,ssh的22，服务器通过bind一个端口然后常驻监听请求（像nginx监听80端口），而对应的客户端的端口一般是随机分配的需要建立连接时，操作系统分配一个端口，进行通信。

* 网络层

  - 包首部是源ip地址，目的ip地址
  - 常用的ping命令就是基于icmp协议
  - 路由器的理解：不同网段(192.168.128[网络标示，同一网段相同].10[主机标示])直接需要通过路由器来转发，而同一个网段的链路上通过MAC地址来传输。
  - NAT:a网络里的私有ip(10.0.0.1)通过全局ip（比如局域网的出口ip）和另外一个b网络通信,发出去的时候可以理解，当b回复的时候目的地址是全局ip,那怎么发给10.0.0.1的呢？就是通过nat，发出通信的时候会基于目的地（目的ip+目的端口+协议类型）和私有ip创建一条记录，对面回复的时候通过记录查到私有ip。udp打洞里面有nat穿透，有兴趣可以看看
  - DNS:域名和ip的映射，ip不好记，所以当我们访问google.com的时候dns会解析出对应的ip，后面过程分析里会讲到

* 不同分层的功能对理解上的帮助：

  * 比如不同的网络加速，下面提到的cdn加速，阿里云的全站加速和aws的cf是7层的协议栈支持http/https，那其他的应用是没办法加速的，aws的Global Accelerator是四层协议栈，支持tcp/udp，应用会更广一些。
  * 比如负载均衡，以tcp为例，四层的均衡器拿到包之后只用修改目的ip地址直接转发就可以，tcp连接（或者https的证书校验）都是直接回源的；而七层的均衡器，要对应用层内容解析，更像是一个代理，客户端先跟均衡器连接交互，这个时候均衡器相当于服务端；然后均衡器再和upstream交互，这时候充当客户端。

     

  ​	

  

## 请求过程分析及应用配置

![WechatIMG531](https://github.com/miss-fox/image/blob/master/WechatIMG531.png?raw=true)



![WechatIMG524](https://github.com/miss-fox/image/blob/master/WechatIMG524.png?raw=true)

粗略配图，捂脸.jpg，上面图纵向都是依赖一开始讲到的osi模型。横向上，

1. dns解析，实际应用中通过GeoDNS（aws的route 53），根据用户的地理位置来返回最有的接入ip。这里请求的时候，会先从缓存取，缓存没有去本地host文件取，再找不到才会去Dns服务器取(udp)。

2. cdn，当配置了cdn的情况下，dns解析返回的其实是边缘节点的ip。cdn加速：静态加速静态内容缓存，而动态加速可以理解成链路加速，请求走服务商优化过的公网或者服务商内网到达源站。很多服务商把动静加速合并一个服务，比如aws的CF，阿里云的全站加速。这个情况下证书一般也会在边缘节点配置。

3. 到达源站的时候，一般都是一组服务，通过负载均衡来分发请求到具体的某台机器上。到达机器如果像nginx+php的，会先到nginx （反向代理），然后再交给fpm处理。

4. http请求的时候直接发送数据报文。https请求的时候，在连接建立（比如tcp三次握手之后）后会先校验证书（双方约定ssl版本，随机数，服务端出示证书，客户端校验证书），发请求。所以https+tcp的方式安全可靠当然也会有更多的时间开销。https的通信总和=tcp+ssl+data=1.5rtt+1.5rtt+1rtt(一来一回为一个rtt)。

5. 443端口和真实应用程序的映射是在配置了证书服务器的upstream上做了映射（可以基于url做分组转发）

6. [nginx](https://skyao.gitbooks.io/learning-nginx/content/documentation/keep_alive.html)

   

## 代理

之前爬虫用过http/https/socks4/socks5代理，现在正好区分一下：

1. socks协议工作在五层会话层，当然支持的应用层就会多一些，不限于http,还可以是smtp等，socks4只支持tcp,socks5也支持udp；http工作在应用层，只能代理http；

2. http vs https:http就是普通的转发请求，而遇到https请求的时候，代理是没办法验证证书的，所以七层的是没办法work的；那怎么做到的呢？

   > HTTP 客户端通过 CONNECT 方法请求隧道代理创建一条到达任意目的服务器和端口的 TCP 连接，并对客户端和服务器之间的后继数据进行盲转发。
   
   浏览器挂代理之后https发出的方法是connect,下面的图是通过demo搭的代理截图。http请求get，https connect。
   
    ![proxy](https://github.com/miss-fox/image/blob/master/2731A138244FA673AF5BB4CF4D7B68A6.png?raw=true)
3. Vpn,vpn是专用网络，基于IPSec的都是在网络层

   

## demo实践

### go实现代理（http,socks5）

### nginx配置七层or四层负载均衡

### go实现一个socket通信







