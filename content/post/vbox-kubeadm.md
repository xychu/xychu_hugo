---
title: "在 VirtualBox 虚拟机上通过 `kubeadm` 快速搭建本地 Kubernetes 集群"
slug: "build k8s cluster via kubeadm on vbox vms"
summary: quick setup for local test
topics:
  -  k8s
tags: ["kubernetes"]
date: 2018-06-16T15:30:36+08:00
---

Kubernetes 生态越来约成熟，围绕 k8s 做产品或者平台的开发，有时为了快速验证或者做 demo 免不了需要在本地设备上搭建 k8s 集群，
而且目前大家使用的设备普遍差不多 8 核 16G 内存的配置，通过 VirtualBox 基本可以创建 1-2 台 2C4G 的虚拟机出来，
做为本地临时 k8s 集群也足够了。

今天就总结一下，如何从创建 VirtualBox 虚拟机开始，到通过 `kubeadm` 搭建本地 k8s 集群，希望对准备上手的同学们有所帮助。

> __注：__ 部分步骤需要科学上网，解放社会主义生产力。

## VirtualBox 虚拟机创建

### 下载 CentOS 镜像

这里我们使用 CentOS 7，安装镜像下载地址：https://www.centos.org/download/

点击 "DVD ISO" 并选择喜欢的镜像下载链接就好。

### vm 创建

VirtualBox 通过界面创建 vm 过程非常简单，只需要点击“新建”，填写名称，下一步，设置内存大小，下一步，点击“创建” 即可。

### vm 启动前配置

启动虚拟机前，有三个地方需要在“设置”里配置下：

1. 虚拟 CPU 的个数

    ![vbox_cpu](/images/vbox-kubeadm/vbox_cpu.png)

1. 网络，推荐配置两个网卡，一块 Host-Only 用于组本地局域网，一般情况下 IP 固定；另一块桥接网络（Bridged）或者 NAT，用来连接网络，包括局域网及公网，优先尝试桥接网络，如果发现不能分配到 IP 或者网络功能不正常，停掉配置成 NAT； 
 
    ![vbox_network](/images/vbox-kubeadm/vbox_network.png)

1. 加载安装镜像，选择上面下载好的 CentOS 7 ISO 镜像文件
 
    ![vbox_iso](/images/vbox-kubeadm/vbox_config.png)

配置好后，就可以启动虚拟机进行 CentOS 7 的安装了。

> P.S. 启动虚拟机的时候，推荐使用无界面启动，或者分离式启动，这样通过其他常用终端工具 ssh 连接后，就可以关掉 vbox 窗口了。

### CentOS 安装

CentOS 安装交互已经比较友好了，这里就不一步步说明了，重点截图如下：

![centos_summary](/images/vbox-kubeadm/centos_install_summary.png)

重点提一下三个地方：

1. 时区及网络时间
2. 磁盘自动分区
3. 网络 OnBoot 开启（这里忘了打开，CentOS 默认网卡开机不开启，会导致分不到对应的 IP，解决办法是可以进到系统里，修改 network config 文件，重启网络服务，具体的操作这里不列出了，大家自行谷歌 :-P）

    ![centos_network](/images/vbox-kubeadm/centos_network.png)

安装完成，重启后，就基本完成了 CentOS 的安装了，大家可以登录进去，看一下网络情况，如果没有问题，就可以开始 k8s 集群的安装了。

> 这里推荐大家利用 VirtualBox 做一个系统备份，这个方便以后回滚、复制以及导出，后续如果再需要 CentOS 就不需要再从头安装了，直接 Clone （记得勾选重新初始化 MAC 地址）就好了，会快很多，而且避免出错。

## `kubeadm` 创建 k8s 集群

`kubeadm` 是一个 k8s 官方推出的可以用来方便的创建 kubernetes 集群的工具。

具体的安装及使用的文档，建议直接参考官方说明。

我这里只是把一些注意事项帮大家列出来，免得大家掉坑儿。

### 准备工作

1. Disable `firewalld`

    ```
    systemcctl stop firewalld && systemctl disable firewalld
    ```

1. 安装 Docker

    ```
    yum install -y docker
    systemctl enable docker && systemctl start docker
    ```

1. Disable Swap

    ```
    swapoff -a

    # 修改 /etc/fstab 文件，注释掉 SWAP 的自动挂载，使用free -m 确认 swap 已经关闭。
    sed -i -e '/swap/d' /etc/fstab

    # swappiness 参数调整，修改 /etc/sysctl.d/k8s.conf 添加下面一行：
    echo 'vm.swappiness=0' > /etc/sysctl.d/k8s.conf

    # 执行使修改生效
    sysctl -p /etc/sysctl.d/k8s.conf
    ```

1. Disable SELinux

    ```
    vim /etc/sysconfig/selinux
    # 修改 SELINUX=enforcing 为 SELINUX=disabled
    # 重启 vm
    ```

1. 配置 sysctl iptables

    ```
    cat <<EOF >  /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    EOF
    sysctl --system
    ```

1. 重启

    ```
    reboot
    ```

### 集群初始化

做好上面的准备后，就可以开始初始化集群了，命令很简单：

```
kubeadm init <args> 
```

样例：

```
kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=0.0.0.0 --node-name=192.168.56.105
```

> 推荐使用 `--dry-run` 来做启动前的检查，原因就是执行完 `kubeadm init` 如果发现有问题，需要先 drain node 再 delete node 之后才能 `kubeadm reset` 做清理，比较麻烦，通过 `--dry-run` 可以提前发现一些可以避免的问题。

## 其他方式

其实构建 k8s 集群还有很多其他的方式：

- 二进制直接安装，我个人认为刚接触 k8s 一定要用二进制的方式做一遍，虽然比较繁琐，但是能够获得对 k8s 各个组件配置及相互关系有一个直观感受，增进理解；
- 源代码中的 `./hack/local-cluster-up.sh`，更适合开发及调试 k8s 本身的问题时使用，通过环境变量可以控制持久化 etcd 数据；

## Refs:

* https://kubernetes.io/docs/tasks/tools/install-kubeadm/ "Install kubeadm"
* https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/ "Creating a single master cluster with kubeadm"
* https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/calico "Installing Calico for policy and networking "
