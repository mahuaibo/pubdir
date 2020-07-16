# 一.安装环境

- 中标麒麟 V7.0
- Docker & Docker Compose

Docker 及 Docker Compose 的安装可参考《Docker环境安装手册》。



# 二.使用 Docker 镜像安装 Z-Ledger

Z-Ledger安装可以有两种方式，一种是直接安装 Docker 相关组件的镜像，然后直接在 Docker 环境中进行操作（推荐）；另一种是下载源码，通过编译源码得到需要的组件。

下面我们介绍从仓库直接下载相关镜像。

主要用到的镜像包括zhigui/z-ledger-baseos、zhigui/z-ledger-javaenv、zhigui/z-ledger-peer、zhigui/z-ledger-orderer、zhigui/z-ledger-ccenv 、zhigui/z-ledger-tools  和 zhigui/z-ledger-ca。下表是它们的基本介绍：

| 镜像名称                | 功能描述                                                     |
| :---------------------- | ------------------------------------------------------------ |
| zhigui/z-ledger-ccenv   | 支持 Go 语言的链码基础镜像，其中安装了 chaintool、Go 语言的链码 shim 层。<br />在链码容器生成过程中作为编译环境将链码编译为二进制文件，供链码容器使用，<br />方便保持链码容器自身的轻量化 |
| zhigui/z-ledger-peer    | peer 节点镜像，安装了 peer 相关文件                          |
| zhigui/z-ledger-orderer | orderer 节点镜像，安装了 orderer 相关文件                    |
| zhigui/z-ledger-tools   | 安装了 peer、cryptogen、configtxgen                          |
| zhigui/z-ledger-ca      | z-ledger-ca镜像，安装了 z-ledger-ca 相关                     |
| zhigui/z-ledger-baseos  | 基础镜像，用来生成其他镜像，包括Peer、Orderer、fabric-ca，以及 golang 链码容器 |
| zhigui/z-ledger-javaenv | 支持 Java 语言的链码基础镜像，其中安装了 Gradle、Maven、Java 链码 shim 层等。<br />可以用来生成 Java 链码镜像 |

#### 2.1  从私有仓库获取镜像

首先通过以下命令登陆仓库,回车后输入密码ksjf*$jJL12

```bash
$ docker login -u guomitmp@zhigui.com https://harbor.zhigui.com
```

接着通过以下命令拉取需要的镜像：

```bash

$ docker pull harbor.zhigui.com/z-ledger/z-ledger-ca:2.0 \
  && docker pull harbor.zhigui.com/z-ledger/z-ledger-orderer:2.0 \
  && docker pull harbor.zhigui.com/z-ledger/z-ledger-peer:2.0 \
	&& docker pull harbor.zhigui.com/z-ledger/z-ledger-ccenv:2.0 \
	&& docker pull harbor.zhigui.com/z-ledger/z-ledger-tools:2.0
```

下载好镜像后用以下命令修改镜像标签名称:

```bash
$ docker tag harbor.zhigui.com/z-ledger/z-ledger-ca:2.0  zhigui/z-ledger-ca:2.0 \
&& docker tag harbor.zhigui.com/z-ledger/z-ledger-orderer:2.0  zhigui/z-ledger-orderer:2.0 \
&& docker tag harbor.zhigui.com/z-ledger/z-ledger-peer:2.0 zhigui/z-ledger-peer:2.0 \
&& docker tag harbor.zhigui.com/z-ledger/z-ledger-ccenv:2.0 zhigui/z-ledger-ccenv:2.0 \
&& docker tag harbor.zhigui.com/z-ledger/z-ledger-tools:2.0 zhigui/z-ledger-tools:2.0 
```

查看下载好的镜像：

```bash
$ docker images zhigui/*
REPOSITORY                TAG                 IMAGE ID            CREATED             SIZE
zhigui/z-ledger-tools     2.0                 92159c7187c9        2 weeks ago         1.5GB
zhigui/z-ledger-ccenv     2.0                 f507e3cb0b47        2 weeks ago         1.37GB
zhigui/z-ledger-orderer   2.0                 b9d608460083        2 weeks ago         122MB
zhigui/z-ledger-peer      2.0                 f7957df14f30        2 weeks ago         129MB
zhigui/z-ledger-ca        2.0                 45d5c5b7a5e5        3 weeks ago         214MB

```



