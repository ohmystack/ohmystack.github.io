---
layout: post
title: oslo.messaging 笔记
description: "OpenStack 消息队列的公共库"
category: articles
tags: [OpenStack, oslo, rpc, RabbitMQ]
---

## 消息消费者

`service.start()` 时，把 rpc 准备起来。

```python
self.rpcserver = rpc.get_server(target, endpoints, serializer)
self.rpcserver.start()
```

#### nova/rpc.py

里面 `get_server` 调用 oslo.messaging

```python
   messaging.get_rpc_server(TRANSPORT,
                            target,
                            endpoints,
                            executor='eventlet',
                            serializer=serializer)
    """Construct an RPC server.

    The executor parameter controls how incoming messages will be received and
    dispatched. By default, the most simple executor is used - the blocking
    executor.

    If the eventlet executor is used, the threading and time library need to be
    monkeypatched.

    :param transport: the messaging transport
    :type transport: Transport
    :param target: the exchange, topic and server to listen on
    :type target: Target
    :param endpoints: a list of endpoint objects
    :type endpoints: list
    :param executor: name of a message executor - for example
                     'eventlet', 'blocking'
    :type executor: str
    :param serializer: an optional entity serializer
    :type serializer: Serializer
    """
```

#### oslo_messaging/rpc/server.py

里面有 `get_rpc_server`

它创建一个 `RPCDispatcher`，再把 transport, dispatcher, executor 结合，创建出一个 `oslo_message.server.MessageHandlingServer` 返回给调用者。

> Server for handling messages.
>
> Connect a transport to a dispatcher that knows how to process the message using an executor that knows how the app wants to create new tasks.

这个 `MessageHandlingServer` 是可以被 `.start()` 的。Start handling incoming messages.

它里面也是载入 `oslo_messsaging/_executors` 里面的 driver(blocking, eventlet, ...)，用里面的 `.start()` 起来的。

而在创建这个 executor object 时，传入了 conf, listener, dispatcher。而这里的 listener 是 `dispatcher._listen(self.transport)`，而 dispatcher 里面又是让 `transport._listen(self.target)` 终于知道该听啥了。

最后是调用这个 `self._executor.start()` 开始 `listener.poll()` 坐等请求进来。

#### oslo_messaging/rpc/dispatcher.py

来了一个 RPC message，由 `RPCDispatcher` 来读懂格式，根据 namespace, version, method 把它匹配到对应的 endpoints。

直接 `__call__` 它的时候，
调用里面的 `_dispatch_and_reply`，
里面通过 `_dispatch` 一些逻辑，检查是否有这种方法，如果有，
交给 `_do_dispatch` 去解包 (deserialize) 得到 (consumer) 真正的 func 并执行。

endpoint 暴露着 namespace + version 特定的 method。endpoint 的所有 public methods 都可以被 client 远程调用。

-----------------

## 消息生产者

#### oslo_messaging/rpc/client.py

所谓 `oslo_messaging/rpc/client.py` 的 `RPCClient`
其实在 `.prepare` 时，是去用 `_CallContext._prepare(...)` 返回一个 `_CallContext` 对象。

`prepare` 无非是设置一些参数、属性。

#### oslo_messaging/_drivers/amqpdriver.py

`AMQPDriverBase` 里面可以看到，由 kombu 支撑起来的核心方法就是 `_send`。

`RPCClient` 的 call, cast 其实都是 `_CallContext` 的方法。

cast 就不说了，是靠 `self.transport._send()` 实现的。
call 的功能，其实也是靠 `self.transport._send()` 实现的。transport 的 `_send()` 又是由 `self._driver.send(...)` 完成的。

`AMQPDriverBase` (`oslo_messaging/_drivers/amqpdriver.py`) 里 `_send()` 可以做很多事情，让 `_send` 百变。
它能发 `conn.noticy_send`，能 `conn.fanout_send`，还能 `conn.topic_send`。
如果是 call，那它通过 `wait_for_reply` 参数，知道要提前去准备好一个 msg_id 和 _reply_q

```python
msg.update({'_msg_id': msg_id})
msg.update({'_reply_q': self._get_reply_q()})
```

后者创建了一个 `ReplyWaiter` object。
在消息发出后，

```python
result = self._waiter.wait(msg_id, timeout)
```

#### oslo_messaging/_drivers/impl_rabbit.py

里面的 RabbitDriver 继承自 AMQPDriverBase

#### oslo_messaging/transport.py

一个 Transport 只是 driver 的一层壳，把 driver 包起来。
