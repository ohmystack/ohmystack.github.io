---
layout: post
title: 用 Devstack 安装 OpenStack
description: "Devstack 在降低了无数搭建 OpenStack 的工作量的同时，也不是那么简单就用起来的。"
category: articles
tags: [openstack, devstack]
---

[Devstack](http://devstack.org/) 毫无疑问是现在**最方便**的搭建 OpenStack 的方法。于是，很多文章和视频就开使用“xx分钟搭好 OpenStack”这样的标题来吸引人，而结果呢，相信无数新手奉献给 OpenStack 的第一次，绝对不是只用了“xx分钟”。

其实呢，是有几个值得注意的地方没有注意到，就直接开始使用 Devstack 了。而 Devstack 在简化了无数搭建的工作量的同时，也不是那么简单就用起来的。

## 你的环境是否干净？

Devstack 核心就是它的 stack.sh 脚本，一步成功的前提是：**你的系统足够干净**，甚至连 MySQL、pip 等等的都可以等 Devstack 来帮你装。

如果你之前尝试过 Devstack 安装，或者 apt-get 源安装 OpenStack，那么你最好把它们清理干净（如果需要，包括考虑把 MySQL 也卸载了）。

* 清理 Devstack 的安装，检查以下几个地方

```
/etc/            # 配置文件，删与 OpenStack 相关的，比如 nova, keystone, ...
/opt/stack/      # Devstack 从 GitHub 拉下来的所有代码，全部删除
/usr/local/bin/   # 一些需要安装包的位置，删与 OpenStack 相关的
```
* 清理 apt-get 的安装，除了 `sudo apt-get remove --purge <package_name>` 后，再检查以下几个地方确认没有文件残留

```
/usr/share/pyshared/
/usr/lib/python2.7/dist-packages/
```

## Devstack 首页上的标准安装方法怎么样？

标准安装方法其实就是两步：  
> 把 Devstack 的代码拉下来  
> `git clone https://github.com/openstack-dev/devstack.git`  
> 然后按官方的意思，就直接进去  
> `cd devstack; ./stack.sh`  
> 安装完成了。访问 `http://127.0.0.1/` 就可以进去玩儿了。

**这个方法适合那些希望用 GitHub 上 OpenStack 的最新源码的开发者。**用这个方法，的确有可能实现 xx 分钟装好 OpenStack（只要网速给力）。

但是，它忘记告诉你：

1. **不要用 root 用户运行这个 stack.sh 脚本。**尽管 stack.sh 在运行的时候会检查当前是不是 root 用户，如果是，它会为你的 Linux 系统建一个名为 stack 的用户，并用这个用户完成接下来的安装。但这个 stack 用户的默认 home 目录就是 `/opt/stack/` 了，当然你可以改，但这过程总让人觉得有点怪。所以还是在安装前就切到别的有 sudo 权限的用户比较好。 
2. **HOST_IP 配置。**这个要在 Devstack 目录下的 `localrc` 文件（如果没有，手动创建一个）里配置。可以写 `HOST_IP=127.0.0.1` ，这样装完的 Devstack，所有服务就知道来找 localhost 了，不然 Devstack 会默认用你当前的 ip 作为 HOST_IP，这样你的电脑如果是 DHCP 获取 ip 的，以后你的 ip 一变，整个 OpenStack 都跑不起来了。
3. **有些版本，在重启电脑后，会由于 Cinder 的 Volumns 没有挂载，而不能正常启动 OpenStack。** Cinder 会默认找系统上一块名为 `stack-volumes` 的卷。既然找不到它，解决方法就是我们早一块给它。方法参考[这里](https://lists.launchpad.net/openstack/msg25488.html)。
4. **安装前，系统要尽量干净。**这个我在前面已经说过了。
5. **确保网络畅通。**这个看似是一个很容易实现的东西，但在很多企业的网络环境下，各种防火墙、代理会把本来简单的安装过程搞复杂。问题集中在 pip 包的下载和 GitHub 代码的拉取。pip 可以配置 `~/.pip/pip.conf` 里的源，实在不行，企业自己搭个内部源。GitHub 如果无法拉取，可以尝试把 `stack.sh` 里