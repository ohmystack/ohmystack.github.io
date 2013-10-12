---
layout: post
title: 3分钟配好华丽实用的 Vim for Python
description: "这个vim配置非常适合用来写Python，而且安装方便，兼容Linux和MacOSX。"
category: articles
tags: [vim, python]
image:
  feature: posts_img/2013-10-12-vim-for-python-in-3-minutes_feature.png
  credit: 
  creditlink: 
---

从学习 vim 到现在，差不多两个月时间，但已经确实感受到 vim 的方便，至少在写 Python 代码这方面。

在这期间，我以 [eddie-vim](https://github.com/kaochenlong/eddie-vim) + [python-mode](https://github.com/klen/python-mode) 为基础，慢慢配出了自己的vim。（除了 vim 的基本功能外，着重加入了用来读/写 python 代码的各种实用插件，以及一些方便的快捷键设置。）

现在，只要按照步骤，3分钟，你也可以快速拥有一个我的强大vim，拿来直接用就可以。不用再像我那样辛辛苦苦地配置了（已分别在 Ubuntu 和 MacOSX 测试过，可以成功安装）

下面，就让我们开始*（这无节操的）*3分钟。

<br />

## The 1st minute: 准备
* 首先，要确保电脑上已经安装了编译时带有对 python 支持的 vim， 你可以用下面的命令检查。  
{% highlight bash %}
vim --version | grep +python
{% endhighlight %}
Ubuntu 和 MacOSX 都已经自带了符合我们条件 vim，绝大多数情况下，你可以忽略我上面说的。

* Linux 用户请 update 一下 apt-get 的源；Mac 用户请准备好 [MacPorts](http://www.macports.org/install.php)

* 确保装好了 pip

<br />

## The 2nd minute: 拉！

从 github 上拉下我的 repo 放到 ```~``` 目录下，并重命名为 ```.vim``` 
{% highlight bash %}
cd ~
git clone https://github.com/jiangjun1990/python-eddie-vim.git .vim
{% endhighlight %}

在 ```~``` 目录创建一个 soft link 的 .vimrc，指向我 repo 中的 vimrc 文件。
{% highlight bash %}
ln -s /<full-path-to-your-home>/.vim/vimrc ~/.vimrc
{% endhighlight %}

<br />


## The 3rd minute: 擦屁股

clone下来后，并不是所有插件都能直接使用，有一些是需要手动执行一些额外的安装，才能让该插件能正常使用。  
下面，我会针对它们单独进行安装。具体这些插件是做什么的、怎么用，我以后的文章会介绍。  
而且我以后会把下面的步骤写成一个shell脚本来简化安装，但现在就先照着我下面的命令手动装吧。

* Commend-t
{% highlight bash %}
sudo apt-get install ruby irb rdoc
sudo apt-get install build-essential libopenssl-ruby ruby1.8-dev
{% endhighlight %}
[Mac用户] 不需要安装这些，已经自带了。

* ctags
{% highlight bash %}
sudo apt-get install ctags
{% endhighlight %}
[Mac用户] 改用：```port install ctags```

* rope
{% highlight bash %}
sudo pip install rope ropevim
{% endhighlight %}

* python debug
{% highlight bash %}
sudo pip install dbgp vim-debug
install-vim-debug.py
{% endhighlight %}
是的，第二行是一句命令，直接在命令行里敲就可以了，前面不要加```python```.

<br />

好了！用 vim 打开个 Python 文件瞧瞧吧。

