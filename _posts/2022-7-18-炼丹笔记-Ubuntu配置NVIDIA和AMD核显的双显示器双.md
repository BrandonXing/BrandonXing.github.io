---
layout: post
title: 炼丹笔记Ubuntu配置NVIDIA和AMD核显的双显示器双
date: 2022-07-18 17:18 +0800
last_modified_at: 
tags: [炼丹笔记, 配置环境]
categories: [学习笔记]
toc:  true
---

最近买了个新显示器，但是连接到Ubuntu 20.04后不工作，经过一番折腾，结果如下。

## 硬件环境

AMD Ryzen 5 + NVIDIA 2060

- 添加 amdgpu.exp_hw_support=1（实验性的Renior驱动支持） 到 /etc/default/grub.
{% highlight js %}
sudo gedit /etc/default/grub
{% endhighlight %}
改为
{% highlight js %}
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash amdgpu.exp_hw_support=1"
{% endhighlight %}
- 保存退出，并运行以下指令更新grub：
{% highlight js %}
sudo update-grub
{% endhighlight %}
- 重启电脑
- 将 nouveau driver加入黑名单，在 /etc/modprobe.d/ 内创建配置文件

{% highlight js %}
sudo gedit /etc/modprobe.d/blacklist-nouveau.conf
{% endhighlight %}

加入以下内容

{% highlight js %}
blacklist nouveau
options nouveau modeset=0
{% endhighlight %}

- 保存退出，并运行以下指令更新
{% highlight js %}
sudo update-initramfs -u
{% endhighlight %}

- 重启电脑

- 卸载NVIDIA驱动

{% highlight js %}
sudo apt-get remove --purge '^nvidia-.*'
sudo apt-get install ubuntu-desktop
sudo rm /etc/X11/xorg.conf (如果没有该文档并不影响)
{% endhighlight %}

- 重启电脑，然后安装NVIDIA官方驱动

- 设置NVIDIA显卡位首要显卡，更改配置文件
{% highlight js %}
sudo gedit /usr/share/X11/xorg.conf.d/10-amdgpu.conf
{% endhighlight %}

修改为：

{% highlight js %}
Section "OutputClass"
    Identifier "AMDgpu"
    MatchDriver "amdgpu"
    Driver "amdgpu"
    Option "PrimaryGPU" "no"
EndSection
{% endhighlight %}

- 修改NVIDIA显卡设置

{% highlight js %}
sudo gedit /usr/share/X11/xorg.conf.d/10-nvidia.conf
{% endhighlight %}

修改为
{% highlight js %}
Section "OutputClass"
	Identifier "nvidia"
	MatchDriver "nvidia-drm"
	Driver "nvidia"
	Option "AllowEmptyInitialConfiguration"
	Option "PrimaryGPU" "yes"
	ModulePath "/usr/lib/x86_64-linux-gnu/nvidia/xorg"
EndSection
{% endhighlight %}

- 重启电脑



