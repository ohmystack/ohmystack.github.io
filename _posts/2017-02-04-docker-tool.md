---
layout: post
title: dt (docker-tool)
description: "Convenient tool for Docker operations"
category: articles
tags: [docker, tool, cli, bash]
image:
  feature: posts_img/2017-02-04-docker-tool/logo.png
---

https://github.com/ohmystack/docker-tool

[dt (docker-tool)](https://github.com/ohmystack/docker-tool) is a command-line tool for Docker operations.

It is not like many other tools, you don't need to build it. All of it is a single bash. Magic!  
I have been using it for nearly 1 year, and it improves my efficiency of using Docker distinctly.

## It's not just an `alias`

`dt` gives you more commands that Docker doesn't give you.

#### `dt net`

Get the network type and info.  
You can even get the **veth pair info** of a container if it is using the "default" NetworkMode. This is very helpful when debugging the network.

![dt-net-default](/images/posts_img/2017-02-04-docker-tool/dt-net-default.png)

![dt-net-host](/images/posts_img/2017-02-04-docker-tool/dt-net-host.png)

#### `dt ns-net`

Enter the network namespace of a container.  
So that you can use the utils installed on your server to debug the container inside network.

![dt-ns-net](/images/posts_img/2017-02-04-docker-tool/dt-ns-net.png)

#### `dt ssh`

Go inside both container or image, automatically choose to use `bash` or `sh`.

> Before this, you always type  
> `docker exec -it xxx /bin/bash`,
> and then find that there is no `bash` in the container, then change to `sh`.
> Or, type  
> `docker run -it --rm --entrypoint /bin/bash xxx` to get into an image.
> Now, you only need the `dt ssh`.

![dt-ssh-container](/images/posts_img/2017-02-04-docker-tool/dt-ssh-container.png)

![dt-ssh-image](/images/posts_img/2017-02-04-docker-tool/dt-ssh-image.png)


## Powerful shortcuts

If you have tired with repeating typing the following commands, you can try the `dt` ones.

#### `dt pid`

Get the pid of a container.

![dt-pid](/images/posts_img/2017-02-04-docker-tool/dt-pid.png)

#### `dt ps` & `dt img`

Search (or list) the containers and images.

![dt-ps](/images/posts_img/2017-02-04-docker-tool/dt-ps.png)

![dt-img](/images/posts_img/2017-02-04-docker-tool/dt-img.png)

#### `dt logs`

Short for `docker logs --tail=50 -f <container-id/name>`.

---

More features, you can check `dt help`.

If you like `docker-tool`, please reward the repo a STAR. 🌟 

https://github.com/ohmystack/docker-tool
