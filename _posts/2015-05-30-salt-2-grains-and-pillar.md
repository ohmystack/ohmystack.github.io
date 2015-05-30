---
layout: post
title: Salt (2) 数据：Grains & Pillar
description: "Salt 中的两种静态数据"
category: articles
tags: [salt]
---

[上一篇 Salt (1) 入门](/articles/salt-1-basic) 介绍到：

* `Grains` 每一台 Minion 自身的 **静态** 属性。以 Python 字典的形式存放在 Minion 端。
* `Pillar` 存放 key-value 变量。存放在 Master 端，由 Master 编译好后，下发给 Minion。所以，可以存放密码之类的涉密的或是一些需要统一配置的变量。

这篇文章，就来详细说说 `Grains` 和 `Pillar` 。


## YAML 语法

> http://docs.saltstack.com/en/latest/topics/yaml/index.html

在介绍 Grains 和 Pillar 之前呢，我们先来简单介绍一下 YAML 这种格式，因为我们自定义的 Grains、Pillar 以及下一篇要介绍的 States 都要用 YAML 语法。可以说，在 salt 中 YAML 无处不在。YAML 其实跟 `json` 类似，但是比 json 简洁许多，现在越来越多的地方也开始用 YAML 了。如果已经对 YAML 熟悉的朋友，可以跳过这第一部分。

理解这种数据表示方式，很简单：

1. 注意缩进
2. key:value 用 `:` （冒号）
3. list 用 `-` （短横线）
4. 注释用 `#` （井号）

例子：

```yaml
# ========
# SAMPLE 1
# ========
# in Python
{'my_key': 'my_value'}
# in yaml
my_key:
  my_value

# ========
# SAMPLE 2
# ========
# in Python
{
    'first_level_dict_key': {
        'second_level_dict_key': 'value_in_second_level_dict'
    }
}
# in yaml
first_level_dict_key:
  second_level_dict_key: value_in_second_level_dict

# ========
# SAMPLE 3
# ========
# in Python
{'my_dictionary': ['list_value_one', 'list_value_two', 'list_value_three']}
# in yaml
my_dictionary:
  - list_value_one
  - list_value_two
  - list_value_three
```


------



## Grains - minion 自身的一些静态信息

> http://docs.saltstack.com/en/latest/topics/targeting/grains.html

```
salt '*' grains.ls       #查看grains分类
salt '*' grains.items    #查看grains所有信息
salt '*' grains.item os  #查看grains某个信息
salt '*' grains.get os
```

默认的这些，称为 *core grains*。

可以在 minion 的以下地方自定义 grains，如果重复定义，则会被依次覆盖。

1. Core grains.
2. Custom grain modules in `_grains` directory, synced to minions.
3. Custom grains in `/etc/salt/grains`.
4. Custom grains in `/etc/salt/minion`.

> **注意：** grains 一旦设定，就可以认为 minion 是将这个值存入了一个 python dict 中，并且这个值不会再变化了。这一点在当你写自定义的 `_grains` 时尤其要注意，你写的函数实际上只会被调用一次，而不是每次获取 grains 都去调用。如果想刷新 grains，只有通过 `saltutil.sync_grains` 或者 `saltutil.sync_all` 。
>
> 这也正是我们强调 grains 是 **静态的** 的原因。

刷新 Grains

```
salt '*' saltutil.sync_grains
# OR
salt '*' saltutil.sync_all
```


------


## Pillar - 用来提供（敏感的）全局静态数据

> http://docs.saltstack.com/en/latest/topics/pillar/index.html
> http://docs.saltstack.com/en/latest/topics/best_practices.html#structuring-pillar-files

* Pillar 以树状结构定义在 Salt Master 上。默认位置：`/srv/pillar`
* 目录里面是由很多 `sls` 文件组成。由 `top.sls` 统管。
* Pillar 可以存放结构化数据。key/value, list ...
* Pillar 里面的数据可以存一些敏感信息。它们会被安全送达相关的 minion。

> Pillar 与 Grains 简单区别：
>
> * Grains 是 minion 自己生成、自己保存的信息。
> * Pillar 是 Salt Master 下发的信息。

#### Pillar 简单示例

```
# 查看 minion 的 pillar 数据
salt '*' pillar.items
salt '*' pillar.get xxx
```

#### Pillar 配置

```yaml
# /srv/pillar/top.sls:
# 说明所有 minions 可以访问 data.sls 里的数据
base:
  '*':
    - data
```

#### Pillar 的内容

它就是一些 `.sls` 格式的 YAML 文件。关于 `.sls` 是什么，我们会在下一篇讲到。

我可以写一个 pillar，存放一些 git 相关的变量：

```yaml
# /srv/pillar/global.sls
git: 
  username: ohmystack
  email: jiangjun1990@gmail.com
  clone_to_home_dir: Dev
```

> #####如果两个 pillar 的 key 重复了，会怎样？
> 呵呵。Remember: conflicting keys will be overwritten in a non-deterministic manner!

#### Pillar 与 State 结合

可以在一个 pillar 目录下，放一个 `init.sls`，只要 include 这层目录，就能把这个 pillar 的数据导入。

例如：

```yaml
# /srv/pillar/users/init.sls:
users:
  thatch: 1000
  shouse: 1001
  utahdave: 1002
  redbeard: 1003
  
# /srv/pillar/top.sls:
base:
  '*':
    - data
    - users
```

然后在 State 里结合 pillar

```jinja
{% raw  %}
# /srv/salt/users/init.sls
{% for user, uid in pillar.get('users', {}).items() %}
{{user}}:
  user.present:
    - uid: {{uid}}
{% endfor %}
{% endraw  %}
```

#### 为不同情况定制 pillar data

```jinja
{% raw  %}
#/srv/pillar/pkg/init.sls:
pkgs:
  {% if grains['os_family'] == 'RedHat' %}
  apache: httpd
  vim: vim-enhanced
  {% elif grains['os_family'] == 'Debian' %}
  apache: apache2
  vim: vim
  {% elif grains['os'] == 'Arch' %}
  apache: apache
  vim: vim
  {% endif %}
  
#/srv/salt/apache/init.sls:
apache:
  pkg.installed:
    - name: {{ pillar['pkgs']['apache'] }}
{% endraw  %}
```

#### 为 pillar 提供默认值

Pillar 目前来说，就是 Python 里的 `dict` 。可以用 `get` 、`items` ...

```jinja
{% raw  %}
apache:
  pkg.installed:
    - name: {{ salt['pillar.get']('pkgs:apache', 'httpd') }}
{% endraw  %}
```

#### 刷新 pillar

修改 pillar 后，可以用下面命令立即同步，下发到 minion 上。

```
sudo salt '*' saltutil.refresh_pillar
# OR
salt '*' saltutil.sync_all
```

#### 进阶：Pillar 还可以用来干什么？

它还可以和 Salt 的 environment 功能结合，为指定范围的 minion 赋予 `role` 的概念。（其实随便叫什么都可以，`role` 只是普普通通的一个 key 而已。）


> ##### 详见：
> http://docs.saltstack.com/en/latest/topics/tutorials/states_pt4.html
> http://docs.saltstack.com/en/latest/topics/best_practices.html#structuring-pillar-files
