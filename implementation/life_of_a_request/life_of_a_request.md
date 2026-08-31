# 请求处理流程及响应



## 连接的建立

BFE的监听器处理协程（BfeServer.Serve函数)循环接受新到的客户端连接，并创建新协程处理该连接。

```go
// bfe_server/http_server.go

func (srv *BfeServer) Serve(l net.Listener, raw net.Listener, proto string) error {
    ...
	  for {
			  // accept new connection
			  rw, e := l.Accept()
			  ...
			  // start goroutine for new connection
			  go func(rwc net.Conn, srv *BfeServer) {
				    c, err := newConn(rw, srv)
				    ...
			  	  c.serve()
			  }(rw, srv)
	  }
}
```

## 连接的处理

BFE的连接bfe_server.conn 执行serve函数处理该连接，主要包含：

- 步骤1. **回调点处理**：执行HandleAccept回调点的回调链函数

```go
// bfe_server/http_conn.go

// Callback for HANDLE_ACCEPT
hl = c.server.CallBacks.GetHandlerList(bfe_module.HandleAccept)
if hl != nil {
    retVal = hl.FilterAccept(c.session)
    ...
}
```


- 步骤2. **握手及协商**：执行TLS握手（如果用户发起TLS连接）

在TLS握手成功后，执行HandleHandshake回调点的回调链函数

```go
// bfe_server/http_conn.go

// Callback for HANDLE_HANDSHAKE
hl = c.server.CallBacks.GetHandlerList(bfe_module.HandleHandshake)
if hl != nil {
    retVal = hl.FilterAccept(c.session)
    ...
}
```

然后基于协商协议，选择并执行应用层协议Handler（HTTP2/SPDY/STREAM）

```go
// bfe_server/bfe_server.go

tlsNextProto[tls_rule_conf.SPDY31] = bfe_spdy.NewProtoHandler(nil)
tlsNextProto[tls_rule_conf.HTTP2] = bfe_http2.NewProtoHandler(nil)
tlsNextProto[tls_rule_conf.STREAM] = bfe_stream.NewProtoHandler(
		&bfe_stream.Server{BalanceHandler: srv.Balance})

httpNextProto[bfe_websocket.WebSocket] = bfe_websocket.NewProtoHandler(
		&bfe_websocket.Server{BalanceHandler: srv.Balance})
```

- 步骤3. **连接协议处理**：区分连接的协议，执行：
  - 如果是HTTP(S)连接，在当前协程中顺序读取请求并处理；如果首个请求携带WebSocket升级请求头，则升级到WebSocket协议处理
  - 如果是HTTP2/SPDY连接，在新建协程中并发读取请求并处理
  - 如果是STREAM连接，在新建协程处理数据的双向转发（下文略去）
  - 如果是WebSocket连接，在新建协程中完成握手并代理双向数据流
  
  关于协议的实现说明，详见[核心协议实现](../protocol/protocol.md)一章。
  
  

## 请求的处理

BFE的连接对象bfe_server.conn 执行serveRequest函数处理请求。虽然HTTP/HTTPS/HTTP2/SPDY/使用不同方式传输数据，但BFE从协议层接收到HTTP请求后，在上层都转化为相同的内部请求类型(bfe_http.Request)，并执行统一的逻辑处理。

当服务端配置启用AI Gateway（`Server.EnableAiGateway`）时，请求进入专门的AI网关处理路径：`ReverseProxy.ServeHTTPForAI`。该路径复用常规请求处理中的租户路由、回调点等逻辑，但会基于AI路由结果（`AiRouteResult`）选择目标模型集群，并支持重试与回退。未启用AI Gateway时，使用常规的`ReverseProxy.ServeHTTP`路径。

请求处理的具体流程如下：

- 步骤1. **回调点处理**

  执行HandleBeforeLocation回调点的回调链函数

```go
// bfe_server/reverseproxy.go

// Callback for HandleBeforeLocation
hl = srv.CallBacks.GetHandlerList(bfe_module.HandleBeforeLocation)
if hl != nil {
    retVal, res = hl.FilterRequest(basicReq)
    ...
}
```

- 步骤2. **租户路由**：

  查找请求归属的租户。详见[请求路由](../routing/routing.md)中的说明。

```go
// bfe_server/reverseproxy.go

// find product
if err := srv.findProduct(basicReq); err != nil {
    ...
}
```

- 步骤3.**回调点处理**：

  执行HandleFoundProduct回调点的回调链函数
```go
// bfe_server/reverseproxy.go

// Callback for HandleFoundProduct
hl = srv.CallBacks.GetHandlerList(bfe_module.HandleFoundProduct)
if hl != nil {
    retVal, res = hl.FilterRequest(basicReq)
    ...
}
```

- 步骤4.**集群路由**：

  查找请求归属的目的集群。详见[请求路由](../routing/routing.md)中的说明。
```go
// bfe_server/reverseproxy.go

if err = srv.findCluster(basicReq); err != nil {
    ...
}
```

- 步骤5.**回调点处理**：执行HandleAfterLocation回调点的回调链函数
```go
// bfe_server/reverseproxy.go

// Callback for HandleAfterLocation
hl = srv.CallBacks.GetHandlerList(bfe_module.HandleAfterLocation)
if hl != nil {
    ...
}
```

- 步骤6.**回调点处理**：执行HandleForward回调点的回调链函数

```go
// bfe_server/reverseproxy.go

// Callback for HandleForward
hl = srv.CallBacks.GetHandlerList(bfe_module.HandleForward)
if hl != nil {
    retVal = hl.FilterForward(basicReq)
    ...
}
```

- 步骤7.**请求预处理**：

  对请求最终转发前，对请求进行预处理并获取转发参数（例如超时时间）

- 步骤8.**负载均衡及转发**：

  向下游集群转发HTTP请求。详见[负载均衡](../balancing/balancing.md)中的说明

```go
// bfe_server/reverseproxy.go

res, action, err = p.clusterInvoke(srv, cluster, basicReq, rw)
basicReq.HttpResponse = res
```


- 步骤9.**回调点处理**：执行HandleReadResponse回调点的回调链函数
```go
// bfe_server/reverseproxy.go

// Callback for HandleReadResponse
hl = srv.CallBacks.GetHandlerList(bfe_module.HandleReadResponse)
if hl != nil {
    ...
}
```


- 步骤10.**响应发送**：向用户端发送响应

```go
// bfe_server/reverseproxy.go

err = p.sendResponse(rw, res, resFlushInterval, cancelOnClientClose)
if err != nil {
    ...
}
```

## 请求的结束

- 执行HandleRequestFinish回调点的回调链函数

```go
// bfe_server/reverseproxy.go

// Callback for HandleRequestFinish
hl := srv.CallBacks.GetHandlerList(bfe_module.HandleRequestFinish)
if hl != nil {
    ...
}
```

- 检查连接是否需关闭（例如请求被封禁或HTTP KeepAlive未启用）
  - 若回调链返回`BfeHandlerClose`，对应动作`closeDirectly`将直接关闭连接，不再向客户端发送响应
  - 若返回`BfeHandlerFinish`或出现处理错误，对应动作`closeAfterReply`在发送响应后关闭连接
- 如需关闭，连接将停止读取后续请求并执行关闭操作


## 连接的结束

连接在结束前，还需要执行以下操作：

- 执行HandleFinish回调点的回调链函数

```go
// bfe_server/http_conn.go

// Callback for HandleFinish
hl := srv.CallBacks.GetHandlerList(bfe_module.HandleFinish)
if hl != nil {
    hl.FilterFinish(c.session)
}
```

- 写出连接缓存区数据并关闭连接


## links
上一章：[第二十八章 进程模型](../../implementation/process_model/process_model.md)  
下一章：[第三十章 模块框架](../../implementation/model_framework/model_framework.md)