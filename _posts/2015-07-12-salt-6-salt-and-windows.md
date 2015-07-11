---
layout: post
title: Salt (6) Salt & Windows
description: "Salt Minion 在 Windows 还是会有一些坑的，列举一些我遇到过的"
category: articles
tags: [salt]
---

Salt Minion 在 Windows 还是会有一些坑的。这里只能列举一些我遇到过的。

## pip

`pip.installed` 这样的 state 会执行失败，报 `State 'pip.installed' found in SLS 'xxx' is unavailable` 这样的错误。

这是因为 Salt 用的是它自己的 portable Python，也有自己的 `pip.exe` 。安装官方的说法，`pip` 的 State 时，需要显式指明 `cwd` 和 `bin_env` （pip.exe 的位置）。可以参考： https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.pip.html

但我实际试下来，并不管用。我最后还是简单粗暴地用 `cmd.run` 来执行 `pip` 命令。

```yaml
install-requirements:
  cmd.run:
    - name: pip install -r C:\xxx\requirements.txt
    - cwd: C:\Python27\Scripts
```
