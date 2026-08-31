# 支持更多协议

本章将介绍如何配置BFE以支持更多的协议，如：HTTP/2、SPDY、TLS、WebSocket、FastCGI等。

## HTTP/2配置
为支持HTTP/2 over TLS，用户需修改TLS规则配置文件 conf/tls_conf/tls_rule_conf.data。

在该文件中，对需要配置的产品线，修改"NextProtos"域，增加"h2"选项，这将在TLS握手的ALPN协商中增加对HTTP2的支持。如果客户端也支持HTTP/2，建立的连接将使用HTTP/2。
如下述实例，我们对产品线"example_product"进行修改，增加了"h2"。

```json
{
    "Version": "20190101000000",
    "DefaultNextProtos": ["http/1.1"],
    "Config": {
        "example_product": {
            ...
            "NextProtos": ["h2", "http/1.1"],
            ...
        }
    }
}
```

其中，顶层"DefaultNextProtos"为未显式配置"NextProtos"的产品线指定默认的ALPN协议列表，默认值为`["http/1.1"]`。

"NextProtos"中的每项还支持参数化语法，格式为`协议名;level=0;mcs=200;isw=65535;rate=100;pp=0`。各参数含义如下：

| 参数 | 含义 | 默认值 | 合法范围 |
| ---- | ---- | ------ | -------- |
| level | 协商级别。0表示可选；1表示协商但不可被关闭；2表示强制 | 0 | [0, 2] |
| mcs | 单连接最大并发流数（对h2生效） | 200 | > 0 |
| isw | 服务端初始流窗口大小（对h2生效） | 65535 | [65535, 262144] |
| rate | 可选协议的生效比例 | 100 | [0, 100] |
| pp | 向后端发送PROXY协议版本，0关闭，1为v1，2为v2（仅对stream生效） | 0 | [0, 2] |

使用curl连接BFE的HTTPS端口进行测试，可以看到连接已经使用HTTP/2。

```
# curl -v -k -H "Host: example.org"  https://localhost:8443
* Rebuilt URL to: https://127.0.0.1:8443/
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 8443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
...
* ALPN, server accepted to use h2
...
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x7fabe6000000)
> GET / HTTP/2
> Host: example.org
> User-Agent: curl/7.54.0
> Accept: */*
...
```

上述"NextProtos"配置同时包含了["h2", "http/1.1"]，这表示会同时支持HTTP/2和HTTP/1.1，其中HTTP/2的优先级较高。

如果希望只使用HTTP/2，可以将"NextProtos"设置为"h2;level=2"。level为协商级别，2表示必选。如下示例：

```json
{
    "Version": "20190101000000",
    "Config": {
        "example_product": {
            ...
            "NextProtos": ["h2;level=2"],
            ...
        }
    }
}
```

## SPDY配置
与HTTP/2相似，在"NextProtos"中设置"spdy/3.1"，可开启对SPDY的支持。
```json
{
    "Version": "20190101000000",
    "Config": {
        "example_product": {
            ...
            "NextProtos": ["spdy/3.1", "http/1.1"],
            ...
        }
    }
}
```

## WebSocket配置
* 支持WS（WebSocket over TCP）

支持ws无需特殊配置，只需配置好支持ws的后端服务和正确的转发路由即可。

使用curl连接BFE的HTTP端口进行测试，带上header："Connection: Upgrade"和"Upgrade: websocket"。我们可以看到连接升级为使用websocket，如下：

```
# curl -v \
       -H "Host: example.org" \
       -H "Connection: Upgrade"  \
       -H "Upgrade: websocket" \
       -H "sec-websocket-key:1234567890"  \
       http://127.0.0.1:8080
* Rebuilt URL to: http://127.0.0.1:8080/
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to 127.0.0.1 (127.0.0.1) port 8080 (#0)
> GET / HTTP/1.1
> Host: example.org
> User-Agent: curl/7.54.0
> Accept: */*
> Connection: Upgrade
> Upgrade: websocket
> sec-websocket-key:1234567890
> 
< HTTP/1.1 101 Switching Protocols
< Connection: Upgrade
< Sec-Websocket-Accept: qrAsbG+EeIo8ooFLgckbiuFt1YE=
< Upgrade: websocket
```

* 支持WSS（WebSocket over TLS）

WSS是WebSocket over TLS，BFE在TLS握手终止后按HTTP/1.1处理请求，因此只需确保"NextProtos"中包含"http/1.1"即可支持WSS，无需使用"stream"。

```json
{
    "Version": "20190101000000",
    "Config": {
        "example_product": {
            ...
            "NextProtos": ["http/1.1"],
            ...
        }
    }
}
```

使用curl连接BFE的HTTPS端口进行测试，可以看到连接也升级为使用WebSocket：
```
# curl -v -k \
       -H "Host: example.org" \
       -H "Connection: Upgrade"  \
       -H "Upgrade: websocket" \
       -H "sec-websocket-key:1234567890"  \
       https://127.0.0.1:8443

* Rebuilt URL to: https://127.0.0.1:8443/
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to 127.0.0.1 (127.0.0.1) port 8443 (#0)
...
> GET / HTTP/1.1
> Host: example.org
> User-Agent: curl/7.54.0
> Accept: */*
> Connection: Upgrade
> Upgrade: websocket
> sec-websocket-key:1234567890
> 
< HTTP/1.1 101 Switching Protocols
< Connection: Upgrade
< Sec-Websocket-Accept: qrAsbG+EeIo8ooFLgckbiuFt1YE=
< Upgrade: websocket
...
```

## L4 Stream代理

"stream"用于开启四层TCP流代理。与WSS不同，stream不解析HTTP/1.1请求，而是将TLS连接直接透传到后端。如需强制使用stream，可在"NextProtos"中配置"stream;level=2"。

```json
{
    "Version": "20190101000000",
    "Config": {
        "example_product": {
            ...
            "NextProtos": ["stream;level=2;pp=2"],
            ...
        }
    }
}
```

其中"pp"参数可控制向后端发送PROXY协议版本：0为关闭，1为v1，2为v2。

## 连接后端服务的协议

BFE支持使用不同的协议连接后端服务器。我们可以在后端集群配置文件conf/server_data_conf/cluster_conf.data中指定该配置。支持的协议包括：
* HTTP
* HTTPS
* h2c
* FastCGI
* TCP
* WS

### HTTP

HTTP为连接后端使用的缺省协议，无需特殊配置。

### HTTPS

为支持HTTPS，将"BackendConf"中的"Protocol"设置为"https"，并在"HTTPSConf"中配置后端服务的hostname、证书、CA等信息。

```json
{
    "Version": "20190101000000",
    "Config": {
        "cluster_example": {
            ...
            "BackendConf": {
                "Protocol": "https",
                ...
            },
            "HTTPSConf": {
                "RSHost": "www.example.org",
                "RSInsecureSkipVerify": false
            },
            ...
        }
    }
}
```

### h2c

为支持h2c，将"BackendConf"中的"Protocol"设置为"h2c"。

```json
{
    "Version": "20190101000000",
    "Config": {
        "cluster_example": {
            ...
            "BackendConf": {
                "Protocol": "h2c",
                ...
            },
            ...
        }
    }
}
```

### FastCGI
为支持FastCGI，将"BackendConf"中的"Protocol"设置为"fcgi"。其他可设置的参数包括请求文件的根路径"Root"和环境变量"EnvVars"。

```json
{
    "Version": "20190101000000",
    "Config": {
        "cluster_example": {
            "BackendConf": {
                "Protocol": "fcgi",
                ...
                "FCGIConf": {
                    "Root": "/home/work",
                    "EnvVars": {
                        "VarKey": "VarVal"
                    }
                }
            }
        }
    }
}
```

### TCP

为支持四层TCP代理后端，将"BackendConf"中的"Protocol"设置为"tcp"。该协议常与L4 stream代理配合使用。

```json
{
    "Version": "20190101000000",
    "Config": {
        "cluster_example": {
            ...
            "BackendConf": {
                "Protocol": "tcp",
                ...
            },
            ...
        }
    }
}
```

### WS

为支持WebSocket后端，将"BackendConf"中的"Protocol"设置为"ws"。

```json
{
    "Version": "20190101000000",
    "Config": {
        "cluster_example": {
            ...
            "BackendConf": {
                "Protocol": "ws",
                ...
            },
            ...
        }
    }
}
```

## links
上一章：[第二十五章 配置限流](../../operation/configuration/prison.md)  
下一章：[第二十七章 BFE的代码组织](../../implementation/source_layout/source_layout.md)