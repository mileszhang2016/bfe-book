# Request Routing

BFE can process the traffic of multiple tenants at the same time. Each tenant contains multiple clusters, and each cluster handles requests for different types of services. **Request routing** refers to the process of determining the tenant and target cluster to which the request belongs during BFE forwarding request.

The BFE forwarding process can refer to the description in [Forwarding Model](../../design/model/model.md).


## Key Data Structures

In routing module bfe_route/host_table.go, the data structure used to manage routing rules is defined. It mainly includes the following types.

The key fields of `HostTable` are as follows:

```go
// bfe_route/host_table.go

type HostTable struct {
    versions Versions

    hostTable      host_rule_conf.Host2HostTag    // hostname => host-tag
    hostTagTable   host_rule_conf.HostTag2Product // host-tag => product
    vipTable       vip_rule_conf.Vip2Product      // VIP => product
    defaultProduct string                         // default product

    hostTrie *trie.Trie

    productBasicRouteTable    route_rule_conf.ProductBasicRouteRule
    productBasicRouteTree     route_rule_conf.ProductBasicRouteTree
    productAdvancedRouteTable route_rule_conf.ProductAdvancedRouteRule
}
```

### **Domain Name Table**(hostTable) and **Domain Name Lookup Tree**(hostTrie)

BFE uses several data structures together to manage the mapping from domain name to tenant:

- `hostTable` (type `host_rule_conf.Host2HostTag`): stores the mapping from hostname to host-tag.
- `hostTagTable` (type `host_rule_conf.HostTag2Product`): stores the mapping from host-tag to product (tenant).
- `hostTrie` (type `*trie.Trie`): a Trie tree built in memory, used to quickly look up the host-tag and product according to the request Host.

`hostTrie` is a Trie tree. The path from each leaf node to the root node represents a domain name, and the node stores `route{product, tag}`. To support wildcard domain name lookup, the node contains an exact match entry and a wildcard match entry.

The data types of Trie tree nodes are as follows:

- Children points to all child nodes of this node
- Entry stores the domain name represented by the path from the node to the root node (e.g., x.example.com), and the corresponding `route{product, tag}`
- SplatEntry stores the wildcard domain name (e.g., *.x.example.com) represented by the path from the node to the root node, and the corresponding `route{product, tag}`

```go
// bfe_route/trie/trie.go 

type trieChildren map[string]*Trie

type Trie struct {
    Entry      interface{}
    SplatEntry interface{} // to match xxx.xxx.*
    Children   trieChildren 
}
```

### **VIP Table**(vipTable)

VipTable is used to manage the mapping relationship between VIP and tenant ID.

VipTable is a hash table type data structure. Where key represents VIP and value represents tenant name.

```go
// bfe_config/bfe_route_conf/vip_rule_conf/vip_table_load.go

type Vip2Product map[string]string
```

### **Basic Routing Rule Table**(productBasicRouteTable) and **Basic Routing Tree**(productBasicRouteTree)

`productBasicRouteTable` stores the basic rule list of each tenant; `productBasicRouteTree` builds a host+path index based on a Radix tree, which is used to quickly locate the target cluster according to Host and Path.

Basic rules use domain name (Host) and path (Path) as matching conditions, and are matched according to the principles of exact first and longest match. The target cluster of a basic rule can be a normal cluster name or `ADVANCED_MODE`, which means the request continues to be matched in the advanced rule table after hitting this rule.

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

The specific implementation of the Radix tree is in `bfe_config/bfe_route_conf/route_rule_conf/basic_rule_tree.go`.

### **Advanced Routing Rule Table**(productAdvancedRouteTable)

ProductAdvancedRouteTable is used to manage the advanced routing rule table of each tenant. The key represents the name of the tenant, and the value is the advanced routing rule table of the tenant.

The advanced routing rule table of each tenant contains an orderly set of advanced routing rules. Each advanced rule consists of a condition expression and a destination cluster.

```go
// bfe_config/bfe_route_conf/route_rule_conf/route_table_load.go

type AdvancedRouteRule struct {
    Cond        condition.Condition
    ClusterName string
}

type ProductAdvancedRouteRule map[string]AdvancedRouteRules
```

![product route table](product_route_table.png)

(The figure above mainly illustrates the organization of the advanced rule table; the current implementation also includes a basic rule table and a basic routing tree.)

### **Routing Condition**(Condition)

A routing condition is a condition expression composed of condition primitives and operators. For the syntax of condition expression, please refer to the description in [Traffic Routing](../../design/route/route.md).

The condition expression is a binary tree in the form of infix expression in the internal data structure of BFE. The non-leaf node of the binary tree represents the operator. Leaf nodes represent condition primitives. When the request matches the routing condition, it is equivalent to evaluating the infix expression. The return value is Boolean, representing whether the request matches the rule.

![condition expression](cond_expr.png)



## Tenant Routing

LookupHostTagAndProduct() of HostTable implements the search of the target tenant.

The basic process of searching is as follows:

- Step 1: According to Host field of the request, try to search the hostTrie and return the hit tenant name.

- Step 2: If the search fails, try to search the vipTable and return the hit tenant name according to the requested VIP.

- Step 3: If the search fails, return the default tenant name.

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

## Cluster Routing

LookupCluster() of HostTable implements the lookup of the destination cluster.

The basic process of searching is as follows:

- Step 1: First use `productBasicRouteTree` to match the Host and Path of the request according to basic rules; if hit and the target cluster is not `ADVANCED_MODE`, return the target cluster directly.
- Step 2: If the basic rule hits `ADVANCED_MODE` or does not hit, match the conditions of each rule in order in the tenant's advanced rule table `productAdvancedRouteTable`.
- Step 3: Return the destination cluster specified by the first matched advanced rule. If the tenant has no advanced rule table or no rule matches, an error is returned.

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

## AI Request Routing

In the AI gateway scenario, BFE also routes AI requests through the `mod_ai_route` module. This module runs at the `HandleFoundProduct` callback, and looks up the routing table in the priority order of **apikey → entity → global** according to `AiBasicInfo.ClientApiKey`. After a rule is hit, it returns a `targets` list containing cluster, model, and weight, as well as a `fallbacks` degradation list, which are used by the AI gateway forwarding path `ServeHTTPForAI`.

`mod_ai_route` is independent of the tenant-level basic/advanced rule tables described above and is mainly used in the AI gateway scenario. For related configuration and implementation, see `../../../bfe/docs/zh_cn/modules/mod_ai_route/mod_ai_route.md` and `../../../bfe/docs/zh_cn/sys_design/mod_ai_route.md`.


## links
Previous: [Chap30 Module Framework](../../../en_us/implementation/module_framework/module_framework.md)  
Next: [Chap32 Load Balancing](../../../en_us/implementation/balancing/balancing.md)
