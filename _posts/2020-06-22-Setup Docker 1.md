---
layout: post
title:  "项目基础知识培训一"
date:   2020-06-22
tags:
    - 项目
    - 培训
---

云原生（CloudNative）是一个组合词，Cloud+Native。 Cloud表示应用程序位于云中，而不是传统的数据中心；Native表示应用程序从设计之初即考虑到云的环境，原生为云而设计，在云上以最佳姿势运行，充分利用和发挥云平台的弹性+分布式优势。

## 1. 项目知识点

* ETL，是英文 Extract-Transform-Load 的缩写，用来描述将数据从来源端经过抽取（extract）、转换（transform）、加载（load）至目的端的过程。 ETL 一词较常用在数据仓库，但其对象并不限于数据仓库。

* FHIR，是一个平台规范，它定义了一套贯穿卫生健康流程、可以在不同国家和地区的各种场境下使用的功能。

* Hadoop，是一个由 Apache 基金会所开发的分布式系统基础架构。用户可以在不了解分布式底层细节的情况下，开发分布式程序。充分利用集群的威力进行高速运算和存储。Hadoop 实现了一个分布式文件系统（Hadoop Distributed File System），简称 HDFS。HDFS 有高容错性的特点，并且设计用来部署在低廉的（low-cost）硬件上；而且它提供高吞吐量（high throughput）来访问应用程序的数据，适合那些有着超大数据集（large data set）的应用程序。HDFS放宽了（relax）POSIX 的要求，可以以流的形式访问（streaming access）文件系统中的数据。Hadoop 的框架最核心的设计就是：HDFS 和 MapReduce。HDFS 为海量的数据提供了存储，而 MapReduce 则为海量的数据提供了计算。

* Mysql，请自行百度

* VPN，请自行百度

* KVM，请自行百度

* Docker，容器的一种，lxc

* Nvidia CUDA，（Compute Unified Device Architecture），是显卡厂商 NVIDIA 推出的运算平台。 CUDA™ 是一种由 NVIDIA 推出的通用并行计算架构，该架构使GPU能够解决复杂的计算问题。 它包含了 CUDA 指令集架构（ISA）以及 GPU 内部的并行计算引擎。 开发人员可以使用 C 语言来为 CUDA™ 架构编写程序，C语言是应用最广泛的一种高级编程语言。所编写出的程序可以在支持 CUDA™ 的处理器上以超高性能运行。CUDA3.0 已经开始支持 C++ 和 FORTRAN。

* Google Tensorflow，是一个端到端开源机器学习平台。它拥有一个全面而灵活的生态系统，其中包含各种工具、库和社区资源，可助力研究人员推动先进机器学习技术的发展，并使开发者能够轻松地构建和部署由机器学习提供支持的应用。

## 2. 边缘节点

### 2.1. 基础环境搭建

服务器上架、安装系统（ Ubuntu16.04 ）、安装驱动（Nvidia-SMI4XX+CUDA10.X）
……

### 2.2. 单机容器环境

* 卸载旧版本 Docker<19.03

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
```

* 更新索引

```bash
sudo apt-get update
```

* 安装依赖

```bash
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

```

* 增加官方GPG密钥

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo apt-key fingerprint 0EBFCD88
```

* 增加最新版本Docker源

```bash
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

```

* 安装Docker最新版19.03

```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

* 测试 Docker 是否正常运行

```bash
sudo docker run hello-world
```

### 2.3. 安装nvidia-docker中间件

<img src="https://cloud.githubusercontent.com/assets/3028125/12213714/5b208976-b632-11e5-8406-38d379ec46aa.png" width="100%" height="100%">

```bash
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list

sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit

sudo systemctl restart docker
```

### 2.4. 验证环境安装效果

```docker
docker run --gpus all --rm nvidia/cuda nvidia-smi
```

### 2.5. 安装用户程序

```docker
docker run -itd --gpus all --name ai saitron/ai
```

### 2.6. 进入容器运行程序

```docker
docker exec -it ai bash
```

## 3. 未来1.0版本核心架构

<img src="https://img.worldeyes.cn/chl/uPic/202012CIDI.png" width="100%" height="100%">

## 4. 国内用户单机容器环境的一些补充

```bash
* 增加华为镜像站 GPG 公钥

curl -fsSL https://mirrors.huaweicloud.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -

* 更换为华为amd64软件仓库

sudo add-apt-repository "deb [arch=amd64] https://mirrors.huaweicloud.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"

* 更新索引并安装最新版 Docker

sudo apt-get update
sudo apt-get install docker-ce

* 开启 Docker TCP 2375 远程管理

vi /lib/systemd/system/docker.service

ExecStart= xxxx -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock

# xxx是代表原有的参数，追加 -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock 内容
# 保存启动文件后重启服务

systemctl daemon-reload
systemctl restart docker

# 检查是否生效

ss -unlpt | grep 2375

#配置镜像加速器

sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<- 'EOF'
{
    "registry-mirrors": ["https://registry.cn-hangzhou.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker

```