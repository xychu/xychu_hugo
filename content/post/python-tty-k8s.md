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

我已经将基于 Kubernetes Python client 实现的 `tty` 的例子提交了 PR： https://github.com/kubernetes-client/python/pull/515/

有兴趣的欢迎围观，发现问题还请不吝赐教。

同时，我录制了一个 try out 的上手视频，

[![asciicast](https://asciinema.org/a/fOznfIWkZcYdEslY1iJs4HeZc.png)](https://asciinema.org/a/fOznfIWkZcYdEslY1iJs4HeZc)

<script src="https://asciinema.org/a/fOznfIWkZcYdEslY1iJs4HeZc.js" id="asciicast-fOznfIWkZcYdEslY1iJs4HeZc" async></script>

## 重点讲解


## Refs:

* http://sqizit.bartletts.id.au/2011/02/14/pseudo-terminals-in-python/ "Pseudo Terminals in Python"
* https://github.com/python/cpython/blob/master/Lib/pty.py "Pseudo terminal utilities"
* https://github.com/kubernetes-client/python-base/blob/master/stream/ws_client.py "A websocket client with support for channels"