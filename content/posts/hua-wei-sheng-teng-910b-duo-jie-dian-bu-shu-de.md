---
title: "华为昇腾 910B 多节点部署 Deepseek R1 指北"
date: 2025-06-12T08:00:00+00:00
# weight: 1
# aliases: ["/first"]
tags: ["Deepseek", "910B", "Atlas800", "国产算力", "多节点", "推理"]
author: "马致良"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "由于中国多次经历芯片被限制性销售的“卡脖子”的困境，所以AI大模型推理/训练的芯片也被加在国产化备忘录中。此文章尽可能的详细，读者可以选择性的阅读。  "
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com//mapinxue/blog/content"
    Text: "建议修改" # edit text
    appendFilePath: true # to append file path to Edit link
---

   
## 多个节点连接组成集群   
如果你试图部署满血版 Deepseek R1 或者任何参数量大于 64 \* 8 = 512B 的大语言模型，那么必须使用多个服务器通过高速网络连接纳管到一起使用。 

模型以及其需要的资源：   
- DeepSeek R1 671b 满血版：由于910B 只支持 BF16 和 INT8(w8a8)，所以对于真正的满血版（BF16）需要 671 \* 2 = 1342，再加上性能损耗所以最少需要4台Atlas 800。   
    - 下载地址：[https://www.modelscope.cn/models/deepseek-ai/DeepSeek-R1-0528](https://www.modelscope.cn/models/deepseek-ai/DeepSeek-R1-0528)    
- Deepseek R1 671b 满血版 Int8 量化: 其需要至少两台 Atlas 800(16 \* 64GB) 。   
    - 下载链接：[https://www.modelscope.cn/models/TensorLake/DeepSeek-R1-0528-W8A8](https://www.modelscope.cn/models/TensorLake/DeepSeek-R1-0528-W8A8)    
- 一般小于 450B 的模型均可用单台服务器部署推理。   
 --- 
   
很多情况下，如果我们只是做个Demo演示，那么可以使用家用/机房的 2.5G/10Gbps 交换机来组成集群，但是这种情况下搭建的集群同步速度非常缓慢，部署的模型推理速度远低于一般自然人的文字输出速度，所以基本没法用。如果要私有化部署大模型用来生产的话，必须有1台超过200Ge的交换机做 Peer-Link。   

实际上，当你有实体华为昇腾服务器的时候，可以看到服务器背板的下侧有 8\*（100Ge光纤\*2），这些光口就是用来连接高速交换机的。交换机需要一些配置和调试才能使用，交换机的配置和调试工作一般是专业的交付人员做的，这里做罗列的提示：   
- 配置 VLAN   
- 配置 Roce：指 RDMA 参数面网络，具体内容不清楚。   
    - Wiki：[https://en.wikipedia.org/wiki/RDMA\_over\_Converged\_Ethernet](https://en.wikipedia.org/wiki/RDMA_over_Converged_Ethernet)    
   
这样交换机的配置就结束了，下面就是到服务器侧。  

-----

服务器的互相连通性配置有以下步骤：   
- 使用 hccn\_tool 交换机上配置的VLAN 为每个NPU设置 ip。   
- 使用 hccn\_tool 测试联通性。   
