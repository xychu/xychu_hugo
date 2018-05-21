---
title: "Python exec into k8s container with tty enabled"
summary: pty with k8s stream, with resize by multiplexing
topics:
  -  k8s
tags: ["tty", "kubernetes"]
date: 2018-05-15T23:00:36+08:00
---

# Python 实现开启 tty 的交互式 Kuberntes 容器控制

熟悉 `docker` 及 `kubectl` 的同学们，很大概率使用过 `-it` 的交互方式，效果是通过一个可交互的伪终端来实现对目标容器的控制。

今天介绍一下，如何通过 Python 程序实现类似 `kubectl exec -it` 的 Kubernetes 容器控制程序。

## 基本原理

在开始控制 Kubernetes 容器之前，我们首先要了解下伪终端或 `pty` 的基本原理。

## show me the code

话不多说，直接上代码，请戳这里：https://github.com/kubernetes-client/python/pull/515/

我已经将基于 Kubernetes Python client 实现的 `tty` 的例子提交到了上面的 PR。

有兴趣的欢迎围观，发现问题还请不吝赐教。

同时，我录制了一个 try out 的上手视频，里边包含了想要查看运行效果的基本步骤，
包括 clone 代码，mkvirtualenv, pip 安装依赖, 运行样例等。

<script src="https://asciinema.org/a/fOznfIWkZcYdEslY1iJs4HeZc.js" id="asciicast-fOznfIWkZcYdEslY1iJs4HeZc" async></script>

整体效果受限于当时的设备和环境，可能尺寸偏大，不便于查看，望见谅。

> 另：其中调整窗口尺寸部分 asciinema 好像还不能很好的支持，所以视频中没有体现出效果，
> 大家亲自尝试的时候，可以通过终端的多行输出看出容器中的 `tty` 尺寸是会随着当前的终端尺寸变化而调整的。


## 重点讲解


## Refs:

* http://sqizit.bartletts.id.au/2011/02/14/pseudo-terminals-in-python/ "Pseudo Terminals in Python"
* https://github.com/python/cpython/blob/master/Lib/pty.py "Pseudo terminal utilities"
* https://github.com/kubernetes-client/python-base/blob/master/stream/ws_client.py "A websocket client with support for channels"