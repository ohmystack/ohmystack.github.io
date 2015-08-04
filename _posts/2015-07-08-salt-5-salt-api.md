---
layout: post
title: Salt (5) RESTful API：Salt-api
description: "有了封装的 RESTful API，就可以远程调用 Salt 命令了"
category: articles
tags: [salt]
---

在 `2014.7` 这个版本后，salt-api 模块的代码已被 merge 进 salt-master，有两种启动方式：

1. `rest_wsgi` 方式。直接配置、部署在 Apache/Nginx 上。(http://docs.saltstack.com/en/latest/ref/netapi/all/salt.netapi.rest_wsgi.html)
2. `rest_cherrypy` 方式。不依赖 Apache/Nginx，模块自己用 `CherryPy` 起了一个 Web Service。（据文档说，已符合生产使用的要求。）这种方式需要安装 `salt-api` 这个包。(http://docs.saltstack.com/en/latest/ref/netapi/all/salt.netapi.rest_cherrypy.html)

试下来，选用 `rest_cherrypy` 支撑 Salt-api 能拥有更出色的性能。

下面以 `rest_cherrypy` 方式为例，起一个最简单的 HTTP（不用 HTTPS）的 Salt RESTful API 服务。

```bash
sudo apt-get install salt-api
```

配置 `/etc/salt/master.d/salt-api.conf`。

```yaml
rest_cherrypy:
    port: 8000
    host: 0.0.0.0
    debug: True
    disable_ssl: True
```

接下来解决 API 请求的权限问题。

`salt-api` 使用了 salt 的 [External Authentication System](http://docs.saltstack.com/en/latest/topics/eauth/index.html#acl-eauth) 。

修改 `/etc/salt/master.d/salt-api.conf`，配置 `external_auth`。

```yaml
external_auth:
  pam:
    saltuser:
      - .*
      - '@wheel'
      - '@runner'
      - '@jobs'
```

上面就为 `saltuser` 这个 Linux 用户赋予了几乎 salt 里所有的权限。

重启 `salt-master` 和 `salt-api` 让配置生效

```bash
sudo service salt-master restart
sudo service salt-api restart
```

然后创建一个 `saltuser` 用户

```bash
sudo adduser saltuser
```

至此，HTTP salt RESTful API 就搭好了。

让我们试一下。

首先在 salt-master 上直接用命检验一下这个用户的权限。

```bash
sudo salt -a pam '*' test.ping
username: saltuser
password:
```

看到正常的返回结果后，说明权限配置正确了。

接下来试一下 RESTful API。

有两种方式来发 HTTP 请求：

1. 每次请求里都带 `username` 和 `password`
2. 第一次请求一个 `token`，后面就把它放进 HTTP 请求的 `X-Auth-Token` header 里。

我们采用 `token` 的方式。

```bash
curl -X POST \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -H "Accept: application/json" \
     -d 'username=saltuser&password=xxxxxx&eauth=pam' \
     http://10.18.5.147:8000/login
```

返回结果：

```
{
  "return": [
    {
      "perms": [
        ".*",
        "@wheel",
        "@runner",
        "@jobs"
      ],
      "start": 1421154569.16507,
      "token": "77af3748fb3385fdeedab4d7a6b70d94c36b7139",
      "expire": 1421197769.165072,
      "user": "saltuser",
      "eauth": "pam"
    }
  ]
}
```

有了 `token`，来试一下 `test.ping`

```bash
curl http://10.18.5.147:8000/ \
     -X POST \
     -H "Accept: application/json" \
     -H "X-Auth-Token: 77af3748fb3385fdeedab4d7a6b70d94c36b7139" \
     -H "Content-Type: application/json" \
     -d \
'[
  {
    "client": "local",
    "tgt": "*",
    "fun": "test.ping"
  }
]'
```

返回结果：

```
{
  "return": [
    {
      "jj-dev002.matrix.xxx.com": true,
      "pc-dst52056": true,
      "jj-dev001.matrix.xxx.com": true
    }
  ]
}
```

上面只是最简单的 `test.ping`，如果有更复杂的参数怎么办？

可以看 http://docs.saltstack.com/en/latest/ref/clients/#salt.client.LocalClient.cmd 里面 `Parameters` 这部分，知道有哪些参数可以写在 json 格式的 request body 里。

> 搭建 salt HTTPS RESTful API 的文章
>
> http://www.saltstack.cn/projects/cssug-kb/wiki/Salt-api-deploy-and-use
> http://bencane.com/2014/07/17/integrating-saltstack-with-other-services-via-salt-api/

当然也有几个坑：

1. 重启 salt-master 和 salt-api 会让之前的 token 提前失效。
2. 如果 token 失效，salt-api 的返回仍是 `200`，只是内容里会提示 `Please log in`。
