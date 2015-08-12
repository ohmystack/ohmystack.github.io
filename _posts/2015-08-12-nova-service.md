---
layout: post
title: Nova Service
description: "介绍 Nova 的 Service 是如何启动的，OpenStack 的其它 Service 也都类似"
category: articles
tags: [OpenStack, Nova]
---

# Service

> `nova/service.py`

整个服务创建的过程，由一个进程变出多个子进程，再到运行起来，一个 `server` 贯穿始终，成为主线。这里的 `server` 是一个 `Service` 对象，名字容易让人与 api 里的 server 混淆。

以 `nova-scheduler` 这个服务为例，它是从 `nova/cmd/scheduler.py` （以前是 `bin/nova-scheduler`） 启动整个 nova-scheduler 服务。它通过 `nova/service.py` 的类方法 create 创建了一个 topic 为 `nova.scheduler.rpcapi` 的 service，即后面会提到的 server。在创建时，还显式提供了一个 'binary' 参数，它只是一个以 'nova-' 开头的字符串，用于给其它参数赋上默认的参数，比如 binary 为 'nova-scheduler'，如果没有提供 manager 参数，那么就会默认去 CONF 里找名为 xxx_manager 的类，把它 import 作为 manager。Service 的 create 的全部参数如下：

```bash
  host             # defaults to CONF.host
  binary           # defaults to basename of executable
  topic            # defaults to bin_name - 'nova-' part
  manager          # defaults to CONF.<topic>_manager
  report_interval  # defaults to CONF.report_interval
  periodic_enable  # defaults to CONF.periodic_enable
  periodic_fuzzy_delay  # defaults to CONF.periodic_fuzzy_delay
  periodic_interval_max # if set, the max time to wait between runs
```

一个 Service 通过一个 manager 监听某个 topic 的 queue 来实现 rpc。Service 会通过 manager 运行定时任务(periodical tasks)，并向数据库的 service 表报告当前状态。

一个 Service 创建好后，需要让它运行起来。于是执行 `nova/service.py` 最后面的 `serve(xxx)` 方法，它会新生成一个 `ProcessLauncher` 对象（这个类也是在 `service.py` 中定义的），用来为这个 service 管理进程。然后，调用这个 `ProcessLauncher` 的 `launch_server(server, workers=<int>)` 函数（注意是 `ProcessLauncher` 类里面的 `launch_server` 方法），启动 `<int>` 个 worker（子进程），父进程只起管理子进程的功能，子进程的信息全部存在父进程所在的 wrap 里，子进程是真正干活的。子进程会执行 `_child_process(server)` 方法。

在 `_child_process(server)` 方法中，把子进程传进去。在对进程信号和 eventlet 进行开启。然后创建一个 Launcher 对象，把子进程的 server 扔进去，通过 Launcher 的 `run_server` 静态方法，把 `server.start()` 并让它 `server.wait()`。直到运行结束或出错抛出异常。

**到这里，一个 service 就运行起来了。**

在 `nova/service.py` 的其它类中，充满了各种利用进程信号量的处理。

`signal.signal(xxx,XXX)` 方法可以将系统对 xxx 事件的处理方法改为 XXX。

```bash
# 几个常用信号:

SIGINT    终止进程  中断进程 (control+c)
SIGTERM   终止进程  软件终止信号 (比较友好，如果进程不能中断，可以忽略这个信号)
SIGKILL   终止进程  杀死进程 (这个信号不能被忽略，让进程立刻停止)
SIGALRM   闹钟信号
SIGCHLD   一个子进程结束后自动向父进程发送SIGCHLD信号。
```

```bash
# 两种标准信号处理方法：

signal.SIG_DFL  按系统默认的对该信号处理的方法进行处理。
signal.SIG_IGN  忽略该信号。
```

> 关于 signal 的详细用法，参考：  
> http://docs.python.org/2/library/signal.html  
> http://blog.csdn.net/liangguohuan/article/details/7099978

关于 service 的 worker 的创建(`_start_child` 方法)，首先确保子进程不会被太快地 fork 出来，保证一秒一个。在 `pid = os.fork()` 这句执行时，子进程就被创建出来了，子进程在此时的 pid 变量值会被赋为 0，父进程的 pid 变量的值为子进程的系统真实 pid。

接下来，父进程和子进程都开始执行下一句代码，但是两者是异步的。所以接下来的一句 `if pid == 0:` 就会把父子进程区别对待，`if` 里的代码只会对子进程起作用，来防止子进程循环创建子进程。

最后，其实可以细看一下 `nova/service.py` 里 Service 类的 `start()` 方法，它将通过 manager 创建 `dispathcer`，用它创建出 `consumer`。并创建出定时任务。


# WSGI Service

`nova-api` service 跟上面的 Service 不同，它是通过 `nova.service.WSGIService` 类起来的。

它依次把 `nova.conf` 里的 `enabled_apis = ec2, osapi_compute, metadata` 模块载入。
载入的时候，在 `WSGIService` 里，会尝试载入名为 `xxx_manager` 的 class。

### WSGIService nova-api 如何起来

`nova.cmd.api -> main()` 启动 `api='osapi_compute'` 的 `WSGIService`

由于不存在 `osapi_compute_manager` 这样的类，所以 `self.manager` 为 `None`。

所以就在 `__init__` 里初始化了

```python
self.server = wsgi.Server(...)
```

然后直接在 start 时执行 `self.server.start()`

在 `wsgi.Server` 这个对象 `__init__` 的时候，

```python
self._socket = eventlet.listen(bind_addr, family, backlog=backlog)
```

向系统申请了指定 port 的 socket 资源。其中 `backlog` 是 The maximum number of queued connections.

`wsgi.Server.start()` 时，经过对 socket 的一些列准备（复制一份、设置 SO_KEEPALIVE、wrap ssl ...），
最后用 `eventlet.spawn(func, *args, **kwargs)` 把服务起起来。

而这里的：

* `func` 就是 `eventlet.wsgi.server`，这个方法就是 eventlet 根据 sock、site、custom_pool、log_format 等参数来起 http server。

参数里：

* `app` 是上面 Service 传进来的 `self.loader.load_app(name)`，Return the paste URLMap wrapped WSGI application.
* `protocol` 默认是 `eventlet.wsgi.HttpProtocol`
* `backlog` 默认是 `128`
* `custom_pool` 是 `eventlet.GreenPool(self.pool_size)`
