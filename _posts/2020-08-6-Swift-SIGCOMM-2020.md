---
layout:     post
title:      Swift SIGCOMM 2020
subtitle:   Swift Delay is Simple and Effective for Congestion Control in the Datacenter
date:       2020-7-26
author:     Yiran
header-img: img/post-bg-mountain1.jpg
catalog: true
tags:
    - Congestion Control
    - Transport in Datacenter
---




Swift是谷歌提出的数据中心拥塞控制协议，继承[Timely](https://yi-ran.github.io/2019/03/27/Timely-NSDI-2015/)的思想，使用delay作为拥塞信号


### Core idea

- 区分 fabric congestion 和 endpoint congestion

<img width="700" height="400" src="/img/post-swift-1.png"/>





### Design

使用***Target Delay***而不是delay gradient, 窗口演变仍使用***AIMD***原则

<img width="400" height="800" src="/img/post-swift-2.png"/>
  

### Thoughts




### 参考文献

[Swift Delay is Simple and Effective for Congestion Control in the Datacenter](https://dl.acm.org/doi/pdf/10.1145/3387514.3406591)




