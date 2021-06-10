---
layout: post
title:  "项目基础知识培训二"
date:   2021-06-06
tags:
    - 项目
    - 培训
---

Docker 是构建 Linux 容器的绝佳工具（“事实上的”标准）。
而 Docker-Compose 是以可复制的方式在本地进行开发的工具。
Docker Swarm 模式则是同 Docker-Compose 在本地使用的相同格式的文件，而在分布式集群中将应用程序堆栈部署到生产环境的一种方案。

因此，使用 Docker Swarm 模式的几个好处：

* 代码的可复制性，生产环境使用与本地开发时相同的文件。
* 敏捷性，开发和运维更加简单和快速的进行部署。
* 稳定性和安全性，原生具有容错集群机制。

## 如何启用 Docker Swarm 模式

如果您安装了 Docker，那么您已经拥有 Docker Swarm，它已集成到 Docker 中。

您无需安装其他任何东西。

## 备选方案

类似的替代方案还有：

* Kubernets
* Mesos

然而要使用它们中的任何一个，您需要学习大量的新概念、配置、文件、命令等。

## 关于 Docker Swarm 模式

Docker Swarm 模式可与`上述备选方案`相媲美。

但是，此模式，更适合少于 200 名开发人员的团队或少于 1000 台机器的集群。

这包括中小型组织（例如当您不是 Google 或 Amazon 时）、初创公司、单人项目和“业余爱好”项目。

下面请跟着我请尝试一下。

设置一个准备用于生产的分布式集群。

...需要大约 20 分钟。

如果你觉着它不适合你，那么你可以选择 Kubernetes、Mesos 或任何其他方案。

虽然这些都是很棒的工具。但学习它们可能需要数周时间。因此，在这里花费的 20 分钟并不多（并且到这里您已经花费了 3 分钟）。

## 单台服务器

使用 Docker Swarm 模式，您可以从单个服务器的“集群”开始。

您也可以在 30 元/月的虚拟服务器上进行设置、部署应用程序并完成所有操作。

然后，当用户增长时，您可以使用一行命令向集群添加更多服务器。

并且您可以从单个小型服务器开始，从一开始就使用此模式创建您的应用程序，为大规模部署做好准备。

### 先决条件

* 了解一些 Linux 命令。
* 了解一些 Docker 命令。

## 安装和设置

### 使用 Docker 安装新的 Linux 服务器

* 创建一个新的远程 VPS（“虚拟专用服务器”）。
* 部署最新的 Ubuntu LTS（“长期支持”）版本。本文使用 Ubuntu 18.04
* 通过 SSH 连接到它，例如：
```s
ssh root@172.173.174.175
```

* 使用您拥有的域名的子域名定义服务器名称，例如 dog.example.com
* 确保子域名 DNS 记录指向您的 VPS 的 IP 地址。
* 使用主机名创建一个临时环境变量以供稍后使用，例如：
```s
export USE_HOSTNAME=dog.example.com
```
* 设置服务器hostname：
```s
# Set up the server hostname
echo $USE_HOSTNAME > /etc/hostname
hostname -F /etc/hostname
```
`注意：`如果您不是root用户，则可能需要添加sudo到这些命令中。当您没有足够的权限时，shell 会告诉您。`请注意，` sudo 默认情况下不保留环境变量，但这可以通过 -E 标志启用。

* 更新包：
```s
# Install the latest updates
apt-get update
apt-get upgrade -y
```
* 按照官方指南安装 Docker ...
* ...或者，运行官方的快速脚本：
```s
# Download Docker
curl -fsSL get.docker.com -o get-docker.sh
# Install Docker using the stable channel (instead of the default "edge")
CHANNEL=stable sh get-docker.sh
# Remove Docker install script
rm get-docker.sh
```
* 如果您要设置多个节点（服务器/VPS），请对每个节点重复这些步骤。
* 确保为每个节点使用不同的域/子域。

## 设置集群模式

在 Docker Swarm 模式中，您需要配置一个或多个“管理”节点和一个或多个“工作”节点（可以是相同的管理节点）。

第一步是配置一个（或多个）管理节点。

* 在主管理节点上，运行：
```s
docker swarm init
```
`注意：`如果您看到如下错误：
```
Error response from daemon: could not choose an IP address to advertise since this system has multiple addresses on interface eth0 (138.68.58.48 and 10.19.0.5) - specify one with --advertise-addr
```
...使用外网 IP（例如138.68.58.48在本例中），然后再次运行命令并增加 --advertise-addr，例如：
```s
docker swarm init --advertise-addr 138.68.58.48
```
* 添加管理节点（可选）
在主管理器节点上，对于要设置的每个附加管理器节点，运行：
```s
docker swarm join-token manager
```
* 复制结果并将其粘贴到附加管理器节点的终端中，它将类似于：
```s
 docker swarm join --token SWMTKN-1-5tl7yaasdfd9qt9j0easdfnml4lqbosbasf14p13-f3hem9ckmkhasdf3idrzk5gz 172.173.174.175:2377
 ```
* 添加工作节点（可选）
在主管理器节点上，对于要设置的每个附加工作节点，运行：
```s
docker swarm join-token worker
```
复制结果并将其粘贴到附加工作节点的终端中，它将类似于：
```s
docker swarm join --token SWMTKN-1-5tl7ya98erd9qtasdfml4lqbosbhfqv3asdf4p13-dzw6ugasdfk0arn0 172.173.174.175:2377
```
### 检查
* 检查集群是否已连接并设置所有节点：
```
docker node ls
```
```s
ID                            HOSTNAME             STATUS    AVAILABILITY    MANAGER STATUS    ENGINE VERSION
ndcg2iavasdfrm6q2qwere2rr *   dog.example.com      Ready     Active          Leader            18.06.1-ce
3jrutmd3asdf1ombqwerr9svk     cat.example.com      Ready     Active          Reachable         18.06.1-ce
i9ec9hjasdfaoyyjqwerr3iqa     snake.example.com    Ready     Active          Reachable         18.06.1-ce
```
### 集群部署完毕
就酱。

您已经设置了一个分布式 Docker swarm 模式集群。

后续文章将帮助你了解如何设置 HTTPS，您还有时间，20 分钟还没有结束。

然后你可以看到如何部署`stacks`等。

你已经完成了困难的部分，剩下的很容易。

