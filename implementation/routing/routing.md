# 请求路由

BFE可同时接入多个租户的流量，每个租户包含多个集群，各集群分别处理不同类型业务的请求。**请求路由**指在BFE转发请求过程中，确定请求所属的租户及目标集群的过程。

BFE路由转发的过程可以参考[BFE的转发模型](../../design/model/model.md)中的说明。


## 关键数据结构

在路由模块 bfe_route/host_table.go 中定义了用于管理路由规则的数据结构。其中主要包含以下类型。

`HostTable` 中的关键字段如下：

```go
// bfe_route/host_table.go

type HostTable struct {
    versions Versions

    hostTable      host_rule_conf.Host2HostTag    // 域名 => host-tag
    hostTagTable   host_rule_conf.HostTag2Product // host-tag => 租户
    vipTable       vip_rule_conf.Vip2Product      // VIP => 租户
    defaultProduct string                         // 默认租户

    hostTrie *trie.Trie

    productBasicRouteTable    route_rule_conf.ProductBasicRouteRule
    productBasicRouteTree     route_rule_conf.ProductBasicRouteTree
    productAdvancedRouteTable route_rule_conf.ProductAdvancedRouteRule
}
```

### **域名表**(hostTable) 与 **域名查找树**(hostTrie)

BFE 使用多组数据结构共同管理域名到租户的映射：

- `hostTable`（类型 `host_rule_conf.Host2HostTag`）：保存域名到 host-tag 的映射。
- `hostTagTable`（类型 `host_rule_conf.HostTag2Product`）：保存 host-tag 到租户（product）的映射。
- `hostTrie`（类型 `*trie.Trie`）：在内存中构建的 Trie 树，用于根据请求 Host 快速查找对应的 host-tag 及租户。

`hostTrie` 是一棵 Trie 树，每个叶子节点到根节点的路径代表了一个域名，节点中存放有租户名称和 host-tag。为支持泛域名查找，节点包含精确匹配入口与泛域名入口。

Trie树节点的数据类型如下：

- Children指向该节点的所有子节点
- Entry存放该节点到根节点路径所代表的域名(例x.example.com)，对应的 route{product, tag}
- SplatEntry存放该节点到根节点路径所代表的泛域名(例*.x.example.com)，对应的 route{product, tag}

```go
// bfe_route/trie/trie.go 

type trieChildren map[string]*Trie

type Trie struct {
    Entry      interface{}
    SplatEntry interface{} // to match xxx.xxx.*
    Children   trieChildren 
}
```

### **VIP表**(vipTable)

vipTable用于管理VIP与租户标识的映射关系。
vipTable 是一个哈希表类型的数据结构。其中键代表VIP，值代表租户名称。

```go
// bfe_config/bfe_route_conf/vip_rule_conf/vip_table_load.go

type Vip2Product map[string]string
```

### **基础路由规则表**(productBasicRouteTable) 与 **基础路由树**(productBasicRouteTree)

`productBasicRouteTable` 用于保存各租户的基础规则列表；`productBasicRouteTree` 则基于 Radix 树构建 host+path 索引，用于根据 Host 和 Path 快速定位目标集群。

基础规则使用域名（Host）和路径（Path）作为匹配条件，按精确优先、最长匹配原则进行匹配。基础规则的目标集群既可以是普通集群名，也可以是 `ADVANCED_MODE`，表示命中后继续进入高级规则表匹配。

```go
// bfe_config/bfe_route_conf/route_rule_conf/route_table_load.go

type BasicRouteRule struct {
    Hostname    []string
    Path        []string
    ClusterName string
}

type ProductBasicRouteRule map[string]BasicRouteRules
type ProductBasicRouteTree map[string]*BasicRouteRuleTree
```

Radix 树的具体实现位于 `bfe_config/bfe_route_conf/route_rule_conf/basic_rule_tree.go`。

### **高级路由规则表**(productAdvancedRouteTable)

`productAdvancedRouteTable` 用于管理各租户的高级规则表。其中的键代表租户名称, 值为该租户的高级规则表。

各租户的高级规则表包含了一组有序的高级规则。每条高级规则由条件表达式及目的集群组成。

```go
// bfe_config/bfe_route_conf/route_rule_conf/route_table_load.go

type AdvancedRouteRule struct {
    Cond        condition.Condition
    ClusterName string
}

type ProductAdvancedRouteRule map[string]AdvancedRouteRules
```

![product route table](product_route_table.png)

（上图主要示意高级规则表的组织形式；当前实现还包括基础规则表与基础路由树。）

### **分流条件**(Condition)

分流条件是一个由条件原语及操作符组成的条件表达式。条件表达式的语法可详见[BFE的路由转发机制](../../design/route/route.md)中的说明。

条件表达式在BFE的内部数据结构，是一个中缀表达式形式的二叉树。二叉树的非叶子节点代表了操作符。叶子节点代表条件原语。对请求执行与分流条件匹配时，相当于对该中缀表达式进行求值。其返回值是布尔类型，代表请求是否匹配规则。

![condition expression](cond_expr.png)



## 目的租户路由

HostTable的LookupHostTagAndProduct()实现了目的租户的查找。

查找的基本流程如下：

- 步骤一: 根据请求的Host字段值，尝试查找 hostTrie 并返回命中的租户名称。
- 步骤二: 如果查找失败，根据请求的访问VIP值，尝试查找vipTable并返回命中的租户名称。
- 步骤三: 如果查找失败，返回缺省的租户名称。

```go
// bfe_route/host_table.go

// LookupHostTagAndProduct find hosttag and product with given hostname.
func (t *HostTable) LookupHostTagAndProduct(req *bfe_basic.Request) error {
    hostName := req.HttpRequest.Host

    // lookup product by hostname
    hostRoute, err := t.findHostRoute(hostName)

    // if failed, try to lookup product by visited vip
    if err != nil {
        if vip := req.Session.Vip; vip != nil {
            hostRoute, err = t.findVipRoute(vip.String())
        }
    }

    // if failed, use default proudct
    if err != nil && t.defaultProduct != "" {
        hostRoute, err = route{product: t.defaultProduct}, nil
    }

    // set hostTag and product
    req.Route.HostTag = hostRoute.tag
    req.Route.Product = hostRoute.product
    req.Route.Error = err

    return err
}
```

## 目的集群路由

HostTable的LookupCluster()实现了目的集群的查找。

查找的基本流程如下：

- 步骤一：先使用 `productBasicRouteTree` 按请求的 Host 与 Path 进行基础规则匹配；若命中且目标集群不是 `ADVANCED_MODE`，直接返回该目标集群。
- 步骤二：若基础规则命中 `ADVANCED_MODE` 或未命中，则在租户的高级规则表 `productAdvancedRouteTable` 中按顺序将请求与各规则的条件进行匹配。
- 步骤三：返回最先命中的高级规则所包含的目的集群。若租户无高级规则表或没有任何规则命中，则返回错误。

```go
// bfe_route/host_table.go

// LookupCluster find clusterName with given request.
func (t *HostTable) LookupCluster(req *bfe_basic.Request) error {
    var clusterName string

    // match basic route rules
    basicRules, ok := t.productBasicRouteTree[req.Route.Product]
    if ok {
        host := strings.SplitN(req.HttpRequest.Host, ":", 2)[0]

        path := ""
        if req.HttpRequest.URL != nil {
            path = req.HttpRequest.URL.Path
        }

        clusterName, found := basicRules.Get(host, path)
        if found && clusterName != route_rule_conf.AdvancedMode {
            // set clusterName
            req.Route.ClusterName = clusterName
            return nil
        }
    }

    // match advanced route rules
    rules, ok := t.productAdvancedRouteTable[req.Route.Product]
    if !ok {
        req.Route.ClusterName = ""
        req.Route.Error = ErrNoProductRule
        return req.Route.Error
    }

    // matching route rules
    for _, rule := range rules {
        if rule.Cond.Match(req) {
            clusterName = rule.ClusterName
            break
        }
    }

    if clusterName == "" {
        req.Route.ClusterName = ""
        req.Route.Error = ErrNoMatchRule
        return req.Route.Error
    }

    // set clusterName
    req.Route.ClusterName = clusterName

    return nil
}
```

## AI 请求路由

在 AI 网关场景下，BFE 还通过 `mod_ai_route` 模块对 AI 请求进行路由：按 **apikey → entity → global** 三级优先级查找路由表，命中后返回目标集群、模型及降级列表，供 AI 网关转发路径 `ServeHTTPForAI` 使用。`mod_ai_route` 独立于上述租户级别的基础/高级规则表。其详细设计与配置参见 [Rainway AI Gateway](https://github.com/rainway-ai-gateway/rainway-book/blob/main/design/chapter11-ai-route-rules.md)一书的[AI路由规则设计](https://github.com/rainway-ai-gateway/rainway-book/blob/main/implementation/chapter31-mod-ai-route.md)。


## links
上一章：[第三十章 模块框架](../../implementation/model_framework/model_framework.md)  
下一章：[第三十二章 负载均衡](../../implementation/balancing/balancing.md)
