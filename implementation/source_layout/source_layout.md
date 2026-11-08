# BFE的代码组织

BFE最新的代码可以从BFE开源项目发布页面 https://github.com/bfenetworks/bfe/releases 下载。本章将以BFE v1.9.0版本为例介绍BFE。

在代码目录的顶层可以看到BFE包含如下目录或文件：

```bash
$ ls bfe/
ADOPTERS.md   CHANGELOG.md   CODE_OF_CONDUCT.md   CONTRIBUTING.md   CONTRIBUTORS.md
AGENTS.md     Dockerfile     GOVERNANCE.md        LICENSE           MAINTAINERS.md
Makefile      NOTICE         README-CN.md         README.md         SECURITY.md
VERSION       bfe.go         go.mod               go.sum            revive.toml   release_local.sh
bfe_balance   bfe_basic      bfe_bufio            bfe_config        bfe_debug
bfe_expr      bfe_fcgi       bfe_http             bfe_http2         bfe_module
bfe_modules   bfe_net        bfe_proxy            bfe_route         bfe_server
bfe_spdy      bfe_stream     bfe_util             bfe_wasmplugin    bfe_websocket
conf          docs           examples             snap              tests
```

注意：自v1.9.0起，BFE移除了对Go标准库crypto/tls的私有fork（原bfe_tls目录），TLS相关实现直接基于标准库crypto/tls（详见[第三十三章 核心协议实现](../protocol/protocol.md)）。

按逻辑关系各目录的层次结构如下：

![source layout](./source_layout.png)

按自顶向下顺序各目录对应的功能模块说明如下。

## 网络协议
- `bfe_net`: BFE网络相关基础库代码
- `bfe_http`: BFE HTTP协议基础代码
- `bfe_http2`: BFE HTTP2协议基础代码
- `bfe_spdy`: BFE SPDY协议基础代码
- `bfe_stream`:	BFE TLS代理基础代码
- `bfe_websocket`: BFE WebSocket代理基础代码
- `bfe_proxy`: BFE Proxy协议基础代码

注：TLS协议实现自v1.9.0起直接基于Go标准库crypto/tls，不再有独立的bfe_tls目录。

## 分流转发
- `bfe_route`: BFE分流转发相关代码
- `bfe_balance`: BFE负载均衡相关代码

## 扩展模块
- `bfe_module`: BFE模块框架相关代码
- `bfe_modules`: BFE扩展模块相关代码，包含访问控制、重定向、重写、WAF、限流及WASM插件等模块。BFE还内置了一组面向AI网关场景的扩展模块（AI路由、Token鉴权、AI限流、AI缓存、Batch透传、意图路由、上下文压缩、流量镜像等），其设计、配置与实现详见[Rainway AI Gateway](https://github.com/rainway-ai-gateway/rainway-book/blob/main/README-CN.md)一书
- `bfe_wasmplugin`: BFE WASM插件运行时及ABI适配相关代码

## 服务框架
- `bfe_server`: BFE服务端主体部分

## 基础组件
- `bfe_basic`: BFE基础数据类型定义
- `bfe_config`: BFE配置加载相关代码
- `bfe_debug`: BFE模块调试开关相关代码
- `bfe_expr`: BFE条件表达式引擎相关代码（详见[第三十五章 条件表达式引擎](../../develop/expr_condition_engine/expr_condition_engine.md)）
- `bfe_util`: BFE基础库相关代码


## links
上一章：[第二十六章 支持更多协议](../../operation/configuration/protocol.md)  
下一章：[第二十八章 进程模型](../../implementation/process_model/process_model.md)
