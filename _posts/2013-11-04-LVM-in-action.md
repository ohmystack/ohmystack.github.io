---
layout: post
title: LVM 实战：磁盘扩容
description: "在不影响现有文件系统的前提下给磁盘扩容。"
category: articles
tags: [LVM, Linux]
image:
  feature: posts_img/2013-11-04-LVM-in-action.jpg
  credit: wikipedia
  creditlink: http://en.wikipedia.org/wiki/VHS
---

>  **情境描述**：虚拟机用的磁盘 image 已经扩容，或对应于物理机的话，就是磁盘的容量已经增加了。
然后我们希望把扩大的容量用起来，而且不影响现有的文件系统（不格盘）。

实际使用过程中，我们有时候需要对虚拟机镜像的硬盘扩容，比如，一开始我们创建虚拟机的时候，以为 20G 的磁盘空间就够了，可某一次我们可能一次性就要拷贝一个 10G+ 的文件进虚拟机，这时候我们就傻了。

我们通过 VMware 或者 VirtualBox 的图形界面或者一些命令，我们可以很轻松地扩大虚拟机的磁盘大小，但是，磁盘变大后，系统并不会把它们利用起来。所以这时候，我们就要考虑怎么才能让这些多出来的空间能够被虚拟机里的 Linux 系统用起来。

在此之前，先补充一个“磁盘 MBR”的知识<sup>[[1]](#note1)</sup> ：

> 1个硬盘分为两个区域，一个是MBR（主引导分区），一个是数据区域。
> 
> MBR里记录了两个重要信息，分别是引导程序与磁盘分区表。
> 
> 分区表定义了“第n个磁盘块是从第x个柱面到第y个柱面”，所以，系统每次都取n磁盘块时，就只会读取第x到第y个扇区之间数据。
> 
> 由于MBR容量有限，设计的时候，就只有设计成4个分区记录，4个主分区或者3个主分区和一个扩展分区。
> 
> 如果超过四个分区，系统允许在额外的硬盘空间放另一份磁盘分区信息，那就是扩展分区，当硬盘被出一个扩展分区的时候，实际上扩展分区在MBR磁盘分区表中的信息为另外那份分区表的位置。所以，在扩展分区里面还要划分逻辑分区才能使用。
> 
> 主分区每个硬盘最多只允许4个，其他的分区只能放在扩展分区中。


这样就明白了，因为主分区的个数有限，而且我们希望增加的容量也只是作为存储使用，所以做成拓展分区 (extended) 就可以了。（而如果你是土豪，总共4个主分区，你还打算这次再用一个主分区的名额，那你可以跳过 Part1，直接看下面的 Part2 了。）

## PART1

我们要把增加的容量做成逻辑分区，但系统上 extended 分区的大小已经固定，所以要先对 extended 分区进行扩容。这个 `fdisk` 就做不了了，需要用 `parted` 命令（如果系统不自带 parted，那就从源上装一个）：
{% highlight bash%}
parted /dev/xxx
{% endhighlight %}
进入交互模式，用 `help` 查看帮助命令。

一些值得特别说明的命令：

* `print` 查看分区表。留意要操作的分区 'Number' 这一项，后面操作要用到。
* `unit` 改变 parted 所用的描述大小的默认单位(比如设为 'compact' 就是以 'MB' 为单位)。  
值得注意的是，如果用 MB/GB 这样的单位，磁盘 sector 的选取会有误差的。parted 会为你选最近的 sector，但未必精确。比如 unit 为 MB，那么可能产生 +-500KB 的误差；如果是 GB，那就可能 +-500MB 的误差，这就无法容忍了。所以如果是'创建分区'这样的操作，建议用 'MiB' 这样的单位，而不是 'MB'。'MiB' 会是一个精确值，parted 不会像对待 'MB' 那样去找它最近的单元。
* `resize <minor> <start> <end>` 对指定 minor 号（或 Number 号）的分区从 start 位置到 end 位置
这里 start/end 可以是 xxxMB，也可以是负值，表示从磁盘末尾往前多少的位置，比如 `-0` 就是指到磁盘的末尾。

更多命令详情请参考：
[http://www.gnu.org/software/parted/manual/html_chapter/parted_toc.html](http://www.gnu.org/software/parted/manual/html_chapter/parted_toc.html)


### 实战：

操作前，`print` 结果如下。现有磁盘62.3G，只分给 extended 8G，还有50多G根本没分配。
{% highlight bash%}
Number  Start   End     Size    Type      File system  Flags
1      1049kB  256MB   255MB   primary   ext2         boot
2      257MB   8589MB  8332MB  extended
5      257MB   8589MB  8332MB  logical                lvm
{% endhighlight %}

我希望把这50多G全部用于扩大extended。

用命令：
{% highlight bash%}
resize 2 257MB -0
{% endhighlight %}
其实，只需输入 `resize 2 `，回车，剩下的两个参数，parted 会通过交互的方式让你填写的。`-0` 表示到那个分区的磁盘末尾。

现在再 `print` 看一下，
{% highlight bash%}
Number  Start   End     Size    Type      File system  Flags
1      1049kB  256MB   255MB   primary   ext2         boot
2      257MB   62.3GB  62.0GB  extended
5      257MB   8589MB  8332MB  logical                lvm
{% endhighlight %}

extended 区已经扩大成功了。

extended 区只是相当于“一块物理硬盘”，想把增加出来的空间用上，还要把 Number 为 5 的 lv 扩大。

而 logic volumn 的扩大依赖于它所在的 volumn group 的大小。因为 logic volumn 是从 volumn group 里分出来的，如果 volumn group 不变大，那么 logic volumn 是无法超过 volumn group 的。所以**真正是应该把空间加到 volumn group 上去**。


## PART2

要增加 volumn group 的大小，先用 fdisk 在 extended 上，利用刚才增加但还未分配出去的磁盘空间创建出一个新分区。通过 `fdisk <disk_dev_name>` 进入交互模式，可以通过命令 `m` 查看帮助。首先，输入 `n` 创建新分区，然后选择 `l` 设置新分区为逻辑分区，接下来依次设置分区的起始、终止位置（默认即完全利用这块磁盘上剩余的所有空间，所以默认即可）。创建出的分区，编号为 `6`。可以用命令 `p` 看一下。

{% highlight bash%}
   Device Boot      Start         End      Blocks   Id  System
/dev/vda1   *        2048      499711      248832   83  Linux
/dev/vda2          501758   121634815    60566529    5  Extended
/dev/vda5          501760    16775167     8136704   8e  Linux LVM
/dev/vda6        16777216   121634815    52428800   83  Linux
{% endhighlight %}

接下来，由于我们要用 LVM 来管理这个新分区，我们需要把新分区的管理系统从 Linux 改为 Linux LVM。在交互模式下，输入命令 `t`，然后选择刚才创建的 `6`，输入 `8e` (Linux LVM 的代号)。最后，我们要把刚才的这些操作真正写入硬盘，输入命令 `w`。

至此，我们通过 `fdisk -l` 已经可以看到 `/dev/vda6` 被创建出来了。


再执行 
{% highlight bash%}
vgextend <your_vg_name> /dev/vda6 
{% endhighlight %}
把新分区加进 volumn group (VG Name 可通过 `vgdisplay` 查到)。

现在用 `vgs` 查看 volumn group 的状态，发现 volumn group 已经变大。
{% highlight bash%}
  VG         #PV #LV #SN Attr   VSize  VFree
  jiang51-vg   2   2   0 wz--n- 57.75g 50.03g
{% endhighlight %}
然后把这个 volumn group 里面的 logic volumn 变大。

命令(最后那个'Logic Volumn name'可通过 `lvdisplay` 查到)：
{% highlight bash%}
lvresize -l +100%FREE <Logic Volumn name>
{% endhighlight %}

> **警告：**
> 如果操作时出现下面这样的 warning，就说明现在 logic volumn 的总大小还不对，resize 不但不增加空间，反而在缩小空间，如果继续操作下去，必将丢失数据。应立即停止！按 `n` 取消。
>{% highlight bash%}
WARNING: Reducing active and open logical volume to 32.00 MiB
  THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce root? [y/n]
{% endhighlight %}

最后，要更新 logic volumn 上的文件系统，不然从 `df` 看出文件系统是不知道 logic volumn 变大的。

用命令(其中的 file_system_name 通过 `df` 找到)：
{% highlight bash%}
resize2fs -p <file_system_name>
{% endhighlight %}

这样，磁盘 extended 分区的扩容终于完成了。

- - -

#####注释：
<a id="note1">[1]</a>: [linux磁盘管理学习笔记（上）](http://www.sourcejoy.com/other_dev_tech/linux-disk-manage-1.html)
