---
layout: post
title: 如何学习 OpenStack
description: "这是我在知乎上的一篇回答，写一些关于 OpenStack 入门的方法"
category: articles
tags: [OpenStack]
---

> 这是我在知乎上的一篇回答。  
> http://www.zhihu.com/question/33466516/answer/56968879

在玩之前先搞清楚 OpenStack 各组件是干什么的。

然后，要有自己的环境才能开始玩。（先搞个梯子吧，不然下载代码会让你很痛苦。）  
可以用 DevStack 试着搭一个 all-in-one 的。（以后再慢慢尝试单独的 Network Node。）  
Devstack 的几个最基本的配置写好即可。（配置先搞最简单的，网络也用最简单的。）  
目标只有一个，搭起来后，能从 Dashboard 创建虚拟机，而且虚拟机能拿到网络。  
遇到困难，耐心地去查 OpenStack 各组件的文档，或者重新配 Devstack 的配置，重新来。这是整体了解 OpenStack 最好的过程。这时候基本上还不用看 OpenStack 的代码，最多看两眼 Devstack 的脚本，了解下填的参数被用到了哪里。  

环境好了以后，其实就是去体会各种角色。

### OpenStack Cloud Administrator

可以玩一下 OpenStack 的各组件的命令，例如 [novaclient commands](http://docs.openstack.org/cli-reference/content/novaclient_commands.html)  
现在，你的角色相当于是一位 **OpenStack Cloud Administrator**。  
可以看 [OpenStack Cloud Administrator Guide](http://docs.openstack.org/admin-guide-cloud/content/ch_getting-started-with-openstack.html)

然后，你可能会想玩玩更多的功能，配置更多的参数。  
可以看 [OpenStack Configuration Reference](http://docs.openstack.org/kilo/config-reference/content/)

### OpenStack OPS

但玩着玩着，肯定会出现问题，调不通了，甚至把环境玩坏了，这时候，你就要变身 **OpenStack OPS** 了。  
[OpenStack Operations Guide](http://docs.openstack.org/openstack-ops/content/openstack-ops_preface.html)
这里面就教你很多 troubleshooting 的技能，还有 monitoring、logging，甚至一点架构的设计。这些都是以后非常实用的技能。

### OpenStack Architect

说到架构，也有专门的文档 [OpenStack Architecture Design Guide](http://docs.openstack.org/arch-design/content/ch_preface.html)  
这时，你就成了 **IaaS 的架构师**。

### OpenStack Developer

而当你发现有 bug，有新需求，或者想了解细节的实现了，需要看代码了，你就变身 **OpenStack Developer**。
可以看 [OpenStack - Python Developer Guide (docs)](http://docs.openstack.org/developer/openstack-projects.html)
对于开发者来说，这个文档有一定作用，但其实最好还是直接读源码。

----------------------------------

还有更多文档，看 [OpenStack Docs](http://docs.openstack.org/) 这里的汇总。其实，我对里面 [OpenStack Docs: OpenStack End User Guide](http://docs.openstack.org/user-guide/) 这个文档充满期待，以后我觉得它会是很好的入门教材，但现在还在完善中，里面很多文章，其实都是原本别人用 MarkDown 写在 Github 的文章，后来被整理到这里。

之后，就是不断搜集各个牛人的 RSS，多读人家的文章。以后再补充
