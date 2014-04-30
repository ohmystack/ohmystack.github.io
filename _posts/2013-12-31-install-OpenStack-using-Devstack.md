---
layout: post
title: DevStack 安装 grizzly-eol 版本 OpenStack
description: "DevStack 在降低了无数搭建 OpenStack 的工作量的同时，也不是那么简单就用起来的。"
category: articles
tags: [openstack, DevStack]
image:
  feature: posts_img/2013-12-31-install-OpenStack-using-Devstack.jpg
---

[DevStack](http://DevStack.org/) 毫无疑问是现在 **最方便** 的搭建 OpenStack 的方法。于是，很多文章和视频就开使用 “ xx 分钟搭好 OpenStack” 这样的标题来吸引人，而结果呢，相信无数新手奉献给 OpenStack 的第一次，绝对不是只用了 “ xx 分钟”。

其实，是有几个值得注意的地方没有注意到，就直接开始使用 DevStack 了。而 DevStack 在简化了无数搭建的工作量的同时，也不是那么简单就用起来的。


### 你的环境是否干净？

DevStack 核心就是它的 stack.sh 脚本，它能一键安装成功的前提是： **你的系统足够干净** ，甚至连 MySQL、pip 等等的都可以等 DevStack 来帮你装。

如果你之前尝试过 DevStack 安装，或者 apt-get 源安装 OpenStack，那么你最好把它们清理干净（如果需要，包括考虑把 MySQL 也卸载了）。

清理步骤如下：

* 清理 DevStack 的安装，先运行 `unstack.sh` 和 `clean.sh` ，再检查以下几个地方确认没有 OpenStack 的文件残留。
  
```bash
/etc/            # 配置文件，删与 OpenStack 相关的，比如 nova, keystone, ...
/opt/stack/      # DevStack 从 GitHub 拉下来的所有代码，全部删除
/usr/local/bin/   # 一些需要安装包的位置，删与 OpenStack 相关的
```

* 清理 apt-get 的安装，除了 `sudo apt-get remove --purge <package_name>` 后，再检查以下几个地方确认没有 OpenStack 的文件残留。  

```bash
/usr/share/pyshared/
/usr/lib/python2.7/dist-packages/
```

### DevStack 首页上的标准安装

标准安装方法其实就是两步（先别急着照着做）：  

> 把 DevStack 的代码拉下来  
> `git clone https://github.com/openstack-dev/devstack.git`  
> 然后按官方的意思，就直接进去  
> `cd DevStack; ./stack.sh`  
> 安装完成了。访问 `http://127.0.0.1/` 就可以进去玩了。  
> 以后想再次启动装好的 OpenStack，只需运行  
> `./rejoin-stack.sh`


**这个方法适合那些希望用 GitHub 上 OpenStack 的最新源码的开发者。**用这个方法，的确有可能实现 xx 分钟装好 OpenStack（只要网速给力）。

但是，它忘记告诉你：

1. **不要用 root 用户运行这个 stack.sh 脚本。**尽管 stack.sh 在运行的时候会检查当前是不是 root 用户，如果是，它会为你的 Linux 系统建一个名为 stack 的用户，并用这个用户完成接下来的安装。但这个 stack 用户的默认 home 目录就是 `/opt/stack/` 了，当然你可以改，但这过程总让人觉得有点怪。所以还是在安装前就切到其他拥有 sudo 权限的用户比较好。 
2. **HOST_IP 配置。**这个要在 DevStack 目录下的 `localrc` 文件（如果没有，手动创建一个）里配置。可以写 `HOST_IP=127.0.0.1` ，这样装完的 DevStack，所有服务就知道来找 localhost 了，不然 DevStack 会默认用你当前的 ip 作为 HOST_IP，这样你的电脑如果是 DHCP 获取 ip 的，以后你的 ip 一变，整个 OpenStack 都跑不起来了。
3. **有些版本，在重启电脑后，会由于 Cinder 的 Volumes 没有挂载，而不能正常启动 OpenStack。** Cinder 会默认找系统上一块名为 `stack-volumes` 的卷。既然找不到它，解决方法就是我们造一块给它。方法参考[这里](https://lists.launchpad.net/openstack/msg25488.html)。
4. **安装前，系统要尽量干净。**这个我在前面已经说过了。
5. **确保网络畅通。**这个看似是一个很容易实现的东西，但在很多企业的网络环境下，各种防火墙、代理会把本来简单的安装过程搞复杂。问题集中在 pip 包的下载和 GitHub 代码的拉取。pip 可以配置 `~/.pip/pip.conf` 里的源，实在不行，企业自己搭个内部源。GitHub 如果无法拉取，可以尝试把它拉取的 url 里的 `https` 换成 `http`，主要修改两个地方：   `stack.sh` 里的 `IMAGE_URLS`，以及 `stackrc` 里的 `GIT_BASE`。其它一些通用的设置代理的方法，我就不一一说了。

### 如何安装 grizzly-eol 的 OpenStack

DevStack 其实和 OpenStack 的 repo 一样，也是有不同分支的。

**如果想要搭建 OpenStack 的 grizzly-eol 版本**，首先应该把 DevStack 的目录也切到 grizzly-eol 这个 tag，而不是直接在 `localrc` 指定好几个 xxx_BRANCH 就够的。

```bash
cd devstack
git checkout grizzly-eol
```

接下来，就来说说这个 `localrc` 文件。<sup>[[1]](#note1)</sup>

`stack.sh` 在运行的时候，按 `openrc` ➜ `stackrc` ➜ `localrc` 的顺序依次载入配置，并且如果有重复，后面的会覆盖前面的。

也就是说，你在 [`openrc`](http://devstack.org/openrc.html)、[`stackrc`](http://devstack.org/stackrc.html) 和 [`localrc`](http://devstack.org/localrc.html) 的官方说明文档里看到的所有设置，全都可以用在 `localrc` 这个文件中。

有这么几个常用的配置项：

*  **HOST_IP**  
	例如：`HOST_IP=127.0.0.1`  
	这一个上面已经介绍过了。

*  **XXX_PASSWORD**  
	例如：`MYSQL_PASSWORD=12345`  
	有好几个密码或 token，不过这个可以不用设，因为在 stack.sh 运行的时候，脚本会自动提示你设置。

*  **enable_service / disable_service**  
	例如：`disable_service n-net`  
	DevStack 有一个默认安装的 service 列表（详见[这里](http://devstack.org/stackrc.html)），用这两个方法，你可以指定在默认的基础上添加或去除哪个 service。（比如你想把 nova-network 换成 quantum。）

	但注意一点，我不推荐直接用 `ENABLED_SERVICES+=,quantum` 这样的方式管理要安装的 service；应该用 `enable_service quantum` 这样的方法，因为 enable_service 是一个真正开放给用户使用的 bash 方法，在里面它会帮你检查重复的 service 等等，确保配置的正确性。

*  **LOGFILE / VERBOSE / SCREEN_LOGDIR**  
	例如：  
`LOGFILE=/opt/stack/logs/stack.sh.log`  
`VERBOSE=True`  
`SCREEN_LOGDIR=/opt/stack/logs`   

	这三个是 log 方面的设置，设置后方便检查 DevStack 安装过程中遇到的问题。

*  <span style="color:#cc3c0f">**XXX_BRANCH**</span>  
<span style="color:#cc3c0f">**好！大家提提神！重点来了。**</span>  
	这里的 XX_BRANCH 泛指例如 "NOVA_BRANCH, KEYSTONE_BRANCH" 一类的变量。  
	具体有哪些变量名，可以到例如 `<devstack_path>/lib/nova` 这样的模块文件的开头去找。  

	这些变量指定了 DevStack 从 GitHub 拉哪个版本的代码下来。  
	比如，我们这次想搭 grizzly-eol 的 OpenStack，所以我们就把 nova、keystone、horizon、glance... 的版本写成  
`NOVA_BRANCH=grizzly-eol`  
`CINDER_BRANCH=grizzly-eol`  
`GLANCE_BRANCH=grizzly-eol`  
`HORIZON_BRANCH=grizzly-eol`  
`KEYSTONE_BRANCH=grizzly-eol`  
`QUANTUM_BRANCH=grizzly-eol`   

	这样是不是就可以了？
  
	**不是的，那些 client 也应该指定版本！**  

	否则，DevStack 会默认拉 master 分支上的 client 代码。这样会引起两个问题：
    1. 要是代码版本拉错了，还能将就跑起来，这样就后患无穷。对以后的测试、打包、发布都构成隐患。  
    2. 由于 client 代码较新，它们的 pip 依赖要求的一些第三方包也比较新，会与一些 grizzly-eol 要求的版本范围有冲突，导致 DevStack 无法一键安装成功。
	
	
	然而，当你去 GitHub 上看这些 client 的 tag 时，却发现它们根本没有 grizzly-eol 这样的版本。那我的办法是，根据 tag 的一些说明文字，再配合从 stable/grizzly 诞生一直到 grizzly-eol 的时间段，确定出每个 client 应该 checkout 到哪个 tag。
	
	我整理下来，grizzly-eol 的各 client 应该用的 tag 大致如下：  
`keystoneclient:0.2.5`  
`cinderclient:1.0.2`  
`glanceclient:0.8.0`  
`novaclient:2.13.0`  

	那么，XXX_BRANCH 是不是只能接受 branch 而不接受 tag 呢？不是的。由 `stack.sh` 引用的 `function` 这个文件的 `git_clone` 方法看出，XXX_BRANCH 可以接受 branch 或 tag。

最后，我用来安装 grizzly-eol 版本的 OpenStack 的 `localrc` 文件大致是下面这个样子（记得要改掉里面的`<your_password>`）：

```bash
# Maybe you can find an updated version of this file from 
# https://gist.github.com/jiangjun1990/7703371

ADMIN_PASSWORD=<your_password>
MYSQL_PASSWORD=<your_password>
RABBIT_PASSWORD=<your_password>
SERVICE_PASSWORD=<your_password>
SERVICE_TOKEN=<your_password>
 
HOST_IP=127.0.0.1
 
disable_service n-net
enable_service q-svc
enable_service q-agt
enable_service q-dhcp
enable_service q-l3
enable_service q-meta
enable_service q-lbaas
enable_service quantum
enable_service tempest
 
# Compute Service
NOVA_BRANCH=grizzly-eol
NOVACLIENT_BRANCH=${NOVACLIENT_BRANCH:-2.13.0}
# Volume Service
CINDER_BRANCH=grizzly-eol
CINDERCLIENT_BRANCH=${CINDERCLIENT_BRANCH:-1.0.2}
# Image Service
GLANCE_BRANCH=grizzly-eol
GLANCECLIENT_BRANCH=${GLANCECLIENT_BRANCH:-0.8.0}
# Web UI (Dashboard)
HORIZON_BRANCH=grizzly-eol
# Auth Services
KEYSTONE_BRANCH=grizzly-eol
KEYSTONECLIENT_BRANCH=${KEYSTONECLIENT_BRANCH:-0.2.5}
# Quantum (Network) service
QUANTUM_BRANCH=grizzly-eol
 
# Enable Logging
LOGFILE=/opt/stack/logs/stack.sh.log
VERBOSE=True
SCREEN_LOGDIR=/opt/stack/logs
```

尽管这里介绍的是 grizzly-eol 版本的安装，但不管今后版本怎样更新，如果想安装过去某个固定版本的 OpenStack，思路都是和上面相似的。

- - -

本文原标题是 “用 DevStack 安装 OpenStack”，只是，随着 Devstack 和 OpenStack 的不断更新，Grizzly 版本已经到了 eol(end-of-life) 要说再见的时候。感谢 Grizzly 版本这么长时间来的陪伴。

- - -

#####注释：
<a id="note1">[1]</a>: 这里提到的 localrc 的用法只适合 grizzly-eol 的安装。在使用最新的 Devstack 的时候，请阅读它的官方文档，推荐将配置写在 local.conf 中，而非 localrc。