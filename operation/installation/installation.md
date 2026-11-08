# BFE服务的安装部署
本章将对BFE的下载安装方式进行具体介绍，帮助读者了解如何运行BFE服务。

## BFE安装方式
BFE支持多种方式进行安装，本章将介绍以下五种方式：
* 软件包下载安装
* 源代码编译安装
* go方式安装
* snap方式安装
* docker方式安装

## 软件包方式下载安装

### 获取BFE软件安装包
BFE的安装软件包可以直接从BFE项目在github.com的页面中下载。
对于不同的操作系统(Linux/MacOS/windows)，页面上都提供了相应的软件包。

下载地址如下：

    https://github.com/bfenetworks/bfe/releases
读者可根据操作系统类型，选择所需的软件版本。

### 下载BFE软件包
以Linux环境为例，下载BFE的1.8.5版本。

在上述下载页面上，找到"BFE v1.9.0"，点击展开“Asserts”，如下图：

![](img/v100_assert.png)

点击下载名为"bfe_1.8.5_linux_amd64.tar.gz"的压缩包。

### BFE软件包中的内容

解压下载的文件bfe_1.8.5_linux_amd64.tar.gz：

```
sh-4.2# tar zxf bfe_1.8.5_linux_amd64.tar.gz 
sh-4.2# 
sh-4.2# cd bfe_1.8.5_linux_amd64
sh-4.2# 
sh-4.2# ls 
CHANGELOG.md	LICENSE		README.md	bin		conf
```

其中包括了两个主要目录：
* bin ： 该目录包含可执行程序bfe。

```
sh-4.2$ ls bin
bfe
```

* conf： 该目录中包含了bfe的配置文件。其中，bfe.conf为BFE的主配置文件。

```
sh-4.2$ ls conf
bfe.conf
cluster_conf
mod_access
mod_access_pb3
mod_ai_batch
mod_ai_cache
mod_ai_context
mod_ai_intent
mod_ai_rate_limit
mod_ai_route
mod_ai_token_auth
mod_auth_basic
mod_auth_jwt
mod_auth_request
mod_block
mod_body_process
mod_compress
mod_cors
mod_doh
mod_errors
mod_geo
mod_header
mod_key_log
mod_markdown
mod_prison
mod_redirect
mod_rewrite
mod_session_sticky
mod_static
mod_tag
mod_tcp_keepalive
mod_trace
mod_traffic_mirror
mod_trust_clientip
mod_unified_waf
mod_userid
mod_waf
mod_wasm
server_data_conf
tls_conf
wasm_plugin
```

### 运行BFE服务

执行如下命令，在系统后台启动一个BFE实例。该实例使用了缺省配置启动。
```
sh-4.2# cd bin
sh-4.2# ./bfe -c ../conf -l ../log &
[1] 31024
```

检查端口8080，可以发现已经处于listen状态
```
sh-4.2$ ss -nltp | grep 8080
0      2048                           *:8080                          *:*      users:(("bfe",31024,10))
```

### 停止BFE服务

如需要停止BFE服务，直接执行kill命令: 

```
sh-4.2$ kill 31024
sh-4.2$ 
[1]+  Done                    ./bfe -c ../conf -l ../log
```

### 在MacOs或windows中下载安装BFE
与Linux系统下安装过程相似：在下载页面 *https://github.com/bfenetworks/bfe/releases* 中下载对应操作系统版本的BFE安装包后，就可启动BFE服务。


## 源代码编译方式安装
BFE源代码完全在github.com上开源，用户也可以通过自行编译源代码的方式进行适配安装。

### 环境准备
- 环境要求
  - golang 1.22+
  - git 2.0+
  - glibc-static 2.17+（Linux 下静态编译需要）



+ Go语言环境准备

  下载地址为 *https://golang.org/dl/* 或者 *https://golang.google.cn/dl/*。

  在下载页面，用户可以根据使用的操作系统环境，下载相应的版本。
  下载后，按照 *https://golang.org/doc/install* 或者 *https://golang.google.cn/doc/install* 的说明，进行安装。



+ git安装

  用户需要安装git命令，具体安装不再赘述。

### BFE源码下载
通过 *git clone* 命令，下载BFE源代码
```bash
$ git clone https://github.com/bfenetworks/bfe
```

### BFE源代码编译
进入目录*bfe*，执行*make*命令，编译BFE源代码:

```bash
$ cd bfe
$ make
```

> tips: 如果遇到超时错误"https fetch: Get ... connect: connection timed out", 请设置GO_PROXY代理后重试.

### 运行BFE
编译后完成后，可执行目标文件在目录*output/bin/*下:

```bash
$ file output/bin/bfe
output/bin/bfe: ELF 64-bit LSB executable, ...
```

执行如下命令，运行BFE服务:

```bash
$ cd output/bin/
$ ./bfe -c ../conf -l ../log
```

## go方式安装

### 环境准备
- golang 1.22+

### 安装

```bash
$ go install github.com/bfenetworks/bfe@latest
```

可执行目标文件位置: `${GOPATH}/bin/bfe` 或 `$(go env GOPATH)/bin/bfe`。

> tips: 如果遇到超时错误，请设置 GOPROXY 代理后重试。

### 运行

```bash
$ cd $(go env GOPATH)/bin/
$ ./bfe -c $(go env GOPATH)/src/github.com/bfenetworks/bfe/conf/ -l ../log
```

## snap方式安装

在 Linux 环境可以使用 snap 工具安装 BFE。如果系统还未安装 snap 工具，参见[安装 snap](https://snapcraft.io/docs/installing-snapd)。

### 安装

```bash
$ sudo snap install --edge bfe
```

> tips: 配置文件位于 `/var/snap/bfe/common/conf/`，日志文件位于 `/var/snap/bfe/common/log/`。

### 运行

```bash
$ sudo /snap/bin/bfe
```

## docker方式安装运行
BFE也提供了基于docker的容器镜像，可以方便地通过docker进行安装部署。

### docker环境设置
参考 *docker.com*，设置好系统的docker环境。

### 直接运行已有镜像

当前推荐镜像源为 GitHub Container Registry：

```bash
$ docker run -d -p 8080:8080 -p 8443:8443 -p 8421:8421 ghcr.io/bfenetworks/bfe:latest
```

上述命令将运行一个 BFE 容器，并把容器内的三个默认端口映射到本地。

如需挂载本地配置和日志目录，可按如下方式运行（路径请按需修改）：

```bash
$ docker run -d \
  -p 8080:8080 -p 8443:8443 -p 8421:8421 \
  -v /path/to/conf:/home/work/bfe/conf \
  -v /path/to/log:/home/work/bfe/log \
  ghcr.io/bfenetworks/bfe:latest
```

### 从源码构建镜像

在仓库根目录执行：

```bash
$ make docker
```

构建完成后会得到形如 `bfe:v1.9.0`、`bfe:v1.9.0-debug` 以及 `bfe:latest` 的镜像。

如需将镜像推送到远程仓库（例如 GHCR），可使用：

```bash
$ make docker-push REGISTRY=ghcr.io/your-org
```

详细参数说明可参考 `bfe/docs/zh_cn/installation/install_using_docker.md`。


## BFE命令行参数

在上面的章节中，我们使用了最常用的命令行参数 *-c* 和 *-l* 。BFE支持的命令行参数如下。

|选项|说明|
|---|----|
|-c \<config dir>   |配置文件的根目录，默认路径./conf   |
|-l \<log dir>      |日志文件的根目录，默认路径./log    |
|-s                 |打印log到stdout                |
|-d                 |打印debug日志                  |
|-v                 |显示bfe的版本号                 |
|-V                 |显示bfe的版相关的详细信息        |
|-t                 |测试配置文件并退出               |
|-h                 |显示帮助                       |



## 查看BFE服务的运行状态

BFE提供了接口，可以通过该接口查看服务运行的各种状态数据。端口缺省为8421。用户可以直接使用浏览器访问该端口，如下图：

![](img/v100_monitor.png)



## links
上一章：[第十八章 信息的透传](../../design/info_pass_through/pass_through.md)  
下一章：[第二十章 BFE服务的基础配置](../../operation/configuration/basic.md)