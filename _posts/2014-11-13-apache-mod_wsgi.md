---
layout: post
title: Apache 运行模式 & mod_wsgi
description: "介绍 Apache 的进程、线程模型，以及 Prefork、Worker、mod_wsgi 模块"
category: articles
tags: [apache, mod_wsgi, prefork, worker, horizon]
---

无论是 Horizon 用的 Django，还是很受欢迎的 Flask，Bottle，最后如果选择部署到 Apache 上，都要写一个 .wsgi 文件，再写一个 Apache 的 xml 配置文件，放在 `sites-available` 下面。

> 一般我们网上参考这些 Web 框架的官方文档，照搬一个 xml 文件就行。但我这次写的一个项目，由于我要在 Web Service 起来但时候，检查上次未完成的任务，并恢复它继续执行，还得控制不能重复恢复它两次。那我就希望了解 Apache 到底是如何起这个 wsgi 应用的，还得了解我用的这种 Apache 的工作模式里能否通过共享数据，来判别这个“恢复任务”是否已执行过。这样我才能让我的程序高性能地、按我想要的方式运行下去。

这里就来介绍一下 Prefork & Worker 模式，以及 mod_wsgi 到底能做些什么。这些模式中的进程、线程、数据共享是怎样的。

首先介绍一下 Apache 的两种基本工作模式：Prefork & Worker.

## Prefork (multi-process)

**Prefork 模式下，Apache 会预创一定数量子进程，每个进程起一个线程，接一个请求，不够就再创进程。数量到设置的最大值后，就拒绝新请求。**

* **优点**：天然线程安全
* **缺点**：占用内存多。上下文切换消耗大。

```bash
sudo apt-get install apache2-mpm-prefork
```

Prefork 模式下 WSGI environment key/value 是：

```
wsgi.multithread   False
wsgi.multiprocess  True
```

## Worker (multi-threaded)

**Worker 模式下，Apache 可创若干个进程，每个进程起多个 worker 线程，每个线程去接请求。（一样可以设置具体上限。）不同线程在处理请求时可以是并行的。**

如果一个进程的所有 worker 线程都满负荷了，则会启用另一个进程，在另一个进程里再开 worker 线程。

由于一个进程中的多个 worker 线程可以同时执行，那么这几个 worker 线程就共享同一份 shared data。所以就要考虑线程安全问题。   
（关于线程安全：http://effbot.org/zone/thread-synchronization.htm ）

* **优点**：占用内存少。
* **缺点**：要人工保证，程序以及所用到的第三方库能做到线程安全。 

```bash
sudo apt-get install apache2-mpm-worker
```

这里，我们不用因为 Python 的 GIL (global interpreter lock) 担心性能问题。

因为从 Apache 接收请求，到给 URL 做 mapping 到 WSGI，或者处理静态文件，这些都是用 C 语言写出的 Apache 做的，不是 Python 做的。

每个线程单独做 apache 交给它的事，所以线程之间就不会有 GIL 限制。这样，Apache 可以充分利用多进程、多线程，同时接更多的请求，更好地利用多 cpu (multiple CPUs) 或是多核 cpu (CPUs with multiple cores)。

worker 模式下 WSGI environment key/value 是：

```
wsgi.multithread True
wsgi.multiprocess True
```

**值得一提的是**

查看当前 Apache 用哪种模式：

```bash
apache2ctl -l
```

Prefork 和 Worker 这两种模式，子进程都会在以下情况被 kill：

1. 使用了一定时间后
2. 处理过的请求达到一定数量
3. interpreter reloading is enabled 时，修改配置文件（touch wsgi 文件）触发 reload 整个应用时


## mod_wsgi

现在我们来讲 mod_wsgi。

[mod_wsgi](https://code.google.com/p/modwsgi/) 是 Apache 的一个模块，功能是让 Apache 能高性能地跑起 Python 写的 Web 应用（只要求应用支持 Python WSGI 借口就可以）。

Apache 用哪种模式，全靠装的是哪个版本的 Apache。  
但如果用了 mod_wsgi，就可以在任何模式下将两者混合起来用。只取决于运行时的配置。

它为我们带来了 mod_wsgi daemon 模式。

### mod_wsgi Daemon

与 Pre-fork 和 Worker 相似，mod_wsgi daemon 也是一种 Apache 的工作模式。但这种 daemon 模式只能用于 WSGI 应用，不能提供静态文件服务。

我们可以通过巧妙地控制参数，用这一种模式模拟出前面的 Pre-fork 和 Worker 模式。

```bash
# Pre-fork Mode:
WSGIDaemonProcess example processes=5 threads=1

# Worker Mode:
WSGIDaemonProcess example processes=2 threads=25

# 强制禁用 wsgi.multiprocess，即不写 procesesses。
# （不要写成 processes=1，因为这样 wsgi.multiprocess 依然会为 True）
WSGIDaemonProcess example threads=25
```

### Python Sub Interpreters 与 数据共享

一个 WSGI application 拥有一个 **Python 的子解释器 (Python Sub Interpreters)**。  
每个 **Python Sub Interpreters** 的生命周期、每个 **子进程的 global data** 的生命周期，都是伴随着它所在 **进程** 的，会一直到这个 WSGI application 被重载/销毁。

**子解释器 != 进程**

一般，如果一个进程中运行了多个 WSGI Application，那么这一个进程中就运行了多个 Python Sub Interpreters。  
但是，可以通过设置 `WSGIApplicationGroup` 让 Application 在公共的 Python Sub Interpreters 中运行，即便如此，不同的进程还是无法共享 global data。也就是说，WSGIApplicationGroup 这种共享是“同个进程中不同应用”之间的共享。

* 从一个 WSGI application 的角度看，在 **同一个子进程** 里处理 **同一个应用的多个请求**，这几个请求会 **共享** 同一份 global data；而 **其它子进程** 所处理的请求，就拥有不同的 global data。

* 换另个角度，在每个子进程中，可以运行着多个 WSGI Application，但由于每个 Application 都有自己的 Python Sub Interpreters，所以，即使在 **同个进程** 中， **不同 Application** 之间仍然 **不共享** global data。

* 在同一个 Python Sub Interpreters 中，如果要用 Python module 的全局变量 (global variables)，操作时需要注意为它加锁。

* 多个子解释器或多个进程间的数据共享必须依靠第三方（数据库、共享内存等）。


---

##### 参考文章

http://www.zerigo.com/article/apache_multi-threaded_vs_multi-process_pre-forked
https://code.google.com/p/modwsgi/wiki/ProcessesAndThreading
