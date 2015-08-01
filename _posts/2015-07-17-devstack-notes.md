---
layout: post
title: DevStack 笔记
description: "记录一些我 DevStack 的用法和备忘"
category: articles
tags: [OpenStack, DevStack]
---

## 如何备份 /opt/stack

```bash
tar zcf ~/stack.tar.gz --exclude='/opt/stack/data' \
--exclude='/opt/stack/status' \
--exclude='/opt/stack/tempest/.tox' \
--exclude=.venv \
--exclude=.log \
--exclude=.pyc \
/opt/stack

tar cf ~/new_stack.tar stack --exclude='stack/data' --exclude='stack/logs' --exclude='stack/status' --exclude='stack/.wheelhouse' --exclude=.pyc --exclude=.venv
```

## 误区

长时间用同一份 `/opt/stack` 下面的源码。会引发一些 requirements 的问题。

## 强制从 Git 安装某些 client 包

DevStack 安装那些主要服务时，默认会从 Git 拉下来安装；装其它那些 client 包时，就直接从 pypi 装了。如果希望强制某些 client 包也从 Git 安装，可以这样：

```bash
LIBS_FROM_GIT=python-novaclient,python-neutronclient,oslo.concurrency,oslo.messaging,oslo.serialization,oslo.utils
```

## 让 Dashboard 拥有 VNC Console

从某一版本对 DevStack 后，novnc 和 cauth 就从默认安装里去除了。所以，我们得手动加上。

```bash
enable_service n-novnc n-cauth
```

## cirros

```
username: cirros
password: cubswin:)
```
