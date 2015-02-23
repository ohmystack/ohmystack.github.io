---
layout: post
title: 用终端下载 Youtube 视频
description: 登上墙外的 VM，一口气把视频下载好再回来看
category: articles
tags: [youtube]
---

首先登上墙外的 VM。

```bash
$ sudo apt-get install youtube-dl
```

简单看下 `youtube-dl` 我们主要关注的几个参数

```bash
$ man youtube-dl

# Video Format Options:
#            -f, --format FORMAT              video format code, specify the order of
#                                             preference using slashes: "-f 22/17/18".
#                                             "-f mp4" and "-f flv" are also supported.
#                                             You can also use the special names "best",
#                                             "bestaudio", "worst", and "worstaudio". By
#                                             default, youtube-dl will pick the best
#                                             quality.
#            --all-formats                    download all available video formats
#            --prefer-free-formats            prefer free video formats unless a specific
#                                             one is requested
#            --max-quality FORMAT             highest quality format to download
#            -F, --list-formats               list all available formats
```

假设我想下载 https://www.youtube.com/watch?v=bBeFZgEYYI8 这个视频

### Step1. 

列出这一个视频有哪些格式 & 分辨率的组合可供下载

```bash
$ youtube-dl https://www.youtube.com/watch?v=bBeFZgEYYI8 -F

[youtube] Setting language
[youtube] bBeFZgEYYI8: Downloading webpage
[youtube] bBeFZgEYYI8: Downloading video info webpage
[youtube] bBeFZgEYYI8: Extracting video information
[info] Available formats for bBeFZgEYYI8:
format code extension resolution  note
171         webm      audio only  DASH webm audio , audio@ 48k (worst)
140         m4a       audio only  DASH audio , audio@128k
160         mp4       192p        DASH video
242         webm      240p        DASH webm
133         mp4       240p        DASH video
243         webm      360p        DASH webm
134         mp4       360p        DASH video
244         webm      480p        DASH webm
135         mp4       480p        DASH video
247         webm      720p        DASH webm
136         mp4       720p        DASH video
248         webm      1080p       DASH webm
137         mp4       1080p       DASH video
17          3gp       176x144
36          3gp       320x240
5           flv       400x240
43          webm      640x360
18          mp4       640x360
22          mp4       1280x720    (best)
```

### Step2. 

如果要下载 720p mp4 的视频，我们找到它的 `format code` 为 `136`，

```bash
$ youtube-dl https://www.youtube.com/watch?v=bBeFZgEYYI8 -f 136
```

这样视频就下载下来了。

> NOTE:  
> 最好不要去死记这个 `format code`，因为 Youtube 不一定为每个视频都准备齐这么多格式组合。
> 我们最好还是在下载前，`-F` 探测一下格式有哪些。

------

##### 一点花絮

事出有因，大过年的，我 Linode Tokyo 节点的 IP 被限速了。
好在似乎只是被家这边的电信限速，用公司的 VPN （换成上海的 IP）过去速度还是不错的。
所以我就想到把视频下载到墙外的 VM，然后用公司 VPN 传回来。

发现在 Linode 下载 Youtube 视频的速度真是杠杠的。
