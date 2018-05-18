---
title: "Python exec into k8s container with tty enabled"
summary: pty with k8s stream, with resize by multiplexing
topics:
  -  k8s
tags: ["tty", "kubernetes"]
date: 2018-05-15T23:00:36+08:00
---

# Python 实现开启 tty 的交互式 kuberntes 容器控制

熟悉 `docker` 及 `kubectl` 的同学们，很大概率使用过 `-it` 的交互方式，代表的是  


## 基本原理

## show me the code

## 重点讲解


## Refs:

* http://sqizit.bartletts.id.au/2011/02/14/pseudo-terminals-in-python/ "Pseudo Terminals in Python"
* https://github.com/python/cpython/blob/master/Lib/pty.py "Pseudo terminal utilities"
* https://github.com/kubernetes-client/python-base/blob/master/stream/ws_client.py "A websocket client with support for channels"