---
title: "Distributed Tensorflow and more"
tag: ["AI", "Deep Learning", "tenforflow", "Distributed System"]
date: 2018-04-28T23:00:36+08:00
draft: true
---

# Tensorflow 分布式训练的各种玩儿法 - 蹭 1.8 的热度

Tensorflow 1.8 发布了！ 保持着差不多一个月一个版本，够可以的！
完整 Release Note 请移步 [Github](https://github.com/tensorflow/tensorflow/releases/tag/v1.8.0)。

抛开 Tensorflow Lite 不说，我特别关心的是这一条：

> Can now pass `tf.contrib.distribute.MirroredStrategy()` to `tf.estimator.RunConfig()`
to run an `Estimator` model on multiple GPUs on one machine

解读下：TF 高层（high level） API `Estimator` 通过 `MirroredStrategy()`
支持单机多卡分布式了（In-graph replication, all-reduce synchronous）。
我们有理由相信，后面应该会有更多的分布式策略被支持，多节点，异步，模型并行等。

而我们的目标呢，在某些场景下基于目前的机房建设肯定是多机多卡才够劲儿。
所以，今天就简单总结下，我了解的 Tensorflow 分布式训练的各种玩儿法，以及接下来会继续跟进的几个方向。

## 经典 ps & worker 模式

假定大家对 Tensorflow 的一些基本[概念][tf_intro]及[架构][tf_arch]已经有所了解，在开始介绍经典模式之前，
只简单介绍下分布式涉及到的一些重点概念及策略对比：

- 模型并行 vs 数据并行

    模型并行：模型的各个部分并行于多个计算设备上；适应场景大模型，单个设备容不下；或者模型本身有比较好的并行性；

    数据并行：多个模型副本分别处理不同的训练数据，以提高整体吞吐量；是常见的分布式训练策略；

    ![data_parallelism](../images/data_paralle.jpg)

- `in-graph` replication vs `between-graph` replication

    `in-graph`: 图内复制，只构建一个 client 并把包含所有的 `worker` 设备都在定义一个图中，
    如果 worker 节点及设备过多，计算图过大会导致性能下降；而且只构建一个 client，分发数据的效率以及容错性都不好；

    `between-graph`: 图间复制，每个 `worker` 都初始化一个计算图副本， 通过 `ps`( parameter server) 
    共享变量参数，只需要更新参数，免除了数据分发环节，在规模较大的情况下，相比 `in-graph` 提高了训练并行效率和容错性；

- 同步训练 vs 异步训练

    同步训练：每一次梯度更新，需要等所有的 `worker` 处理完待训练数据，先做聚会处理后再更新参数；
    优势是 loss 下降稳定；劣势是每一步的处理速度都取决于最慢的那个 `worker`；

    异步训练：各个 `worker` 的计算及模型更新都是相互独立的，没有统一控制；
    优势是速度，优化计算资源利用率；劣势是 loss 下降不稳定；

    ![sync_vs_async](../images/tf_sync_async.png)

因为`数据同步`相比较`模型同步`具有更普适的应用场景，所以针对`数据同步`的分布式训练的支持也就更适合作为 Tensorflow 通用特性来在框架级支持。

而源于 `DistBelief` 的基于 `ps` 和 `worker` 分布式训练架构在 Tensorflow 很早的版本中便提供了支持，这也就是这里称之为 `经典` 模式的原因。

在 Tensorflow 的 `ps` 和 `worker` 模式下，`in-graph` 和 `between-graph` replication 都有支持，但是基于性能和实用性考虑，可能 `between-graph` 使用的更多一些，`同步`和`异步`则更多的是根据模型的实际效果以及项目的具体需求来选择。


### 具体样例

官方上手指南：[Distributed Tensorflow](https://www.tensorflow.org/deploy/distributed)

### 集群描述 `tf.train.ClusterSpec`

```python
tf.train.ClusterSpec({
    "worker": [
        "worker0.example.com:2222",
        "worker1.example.com:2222",
        "worker2.example.com:2222"
    ],
    "ps": [
        "ps0.example.com:2222",
        "ps1.example.com:2222"
    ]})

# 其中， "ps" 及 "worker" 为 `job_name`, 还需要 `task_index` 来创建具体的 `tf.train.Server` 实例。
```

### 实践中需要留意的点

- 同步还是异步的选择
- `ps` 和 `worker` 的个数比率的调整
- `ps` 带宽占用过高时，`ps` 及 `worker` 的调度策略
- 分布式训练的状态机定义，包括终止态定义，以及当有 `worker` 训练失败后，是否支持重启训练等

## 高层 `Estimator` API

### 集群描述 `TF_CONFIG` 环境变量

非 chief 节点：
```python
  cluster = {'chief': ['host0:2222'],
             'ps': ['host1:2222', 'host2:2222'],
             'worker': ['host3:2222', 'host4:2222', 'host5:2222']}
  os.environ['TF_CONFIG'] = json.dumps(
      {'cluster': cluster,
       'task': {'type': 'worker', 'index': 1}})
  config = RunConfig()
  assert config.master == 'host4:2222'
  assert config.task_id == 1
  assert config.num_ps_replicas == 2
  assert config.num_worker_replicas == 4
  assert config.cluster_spec == server_lib.ClusterSpec(cluster)
  assert config.task_type == 'worker'
  assert not config.is_chief
```
chief 节点：
```python
  cluster = {'chief': ['host0:2222'],
             'ps': ['host1:2222', 'host2:2222'],
             'worker': ['host3:2222', 'host4:2222', 'host5:2222']}
  os.environ['TF_CONFIG'] = json.dumps(
      {'cluster': cluster,
       'task': {'type': 'chief', 'index': 0}})
  config = RunConfig()
  assert config.master == 'host0:2222'
  assert config.task_id == 0
  assert config.num_ps_replicas == 2
  assert config.num_worker_replicas == 4
  assert config.cluster_spec == server_lib.ClusterSpec(cluster)
  assert config.task_type == 'chief'
  assert config.is_chief
```

## All-reduce

### 单机多卡 `Estimator` + `tf.contrib.distribute.MirroredStrategy()`

### 多机多卡 [Horovod](https://github.com/uber/horovod) (From Uber)



## Distributed Tensorflow on Kubernetes


## 总结




## Refs:
* https://zhuanlan.zhihu.com/p/35083779 "分布式 TensorFlow 入门教程"
* https://zhuanlan.zhihu.com/p/34172340 "【第一期】AI Talk：TensorFlow 分布式训练的线性加速实践"
* https://www.oreilly.com/ideas/distributed-tensorflow
* http://download.tensorflow.org/paper/whitepaper2015.pdf
* https://www.usenix.org/system/files/conference/osdi16/osdi16-abadi.pdf
* https://arxiv.org/pdf/1708.02637.pdf


[tf_intro]: https://www.tensorflow.org/programmers_guide/low_level_intro
[tf_arch]: https://www.tensorflow.org/extend/architecture
