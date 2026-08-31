# 配置限流
本章介绍如何配置限流功能。该功能将限制特定请求的速率，以对后端服务进行保护。

## 开启限流模块
在 conf/bfe.conf 中，打开该模块
```ini
Modules = mod_prison
```

## 模块配置
模块配置在目录 conf/mod_prison/ 中，包含两个文件：

```
# ls
mod_prison.conf	prison.data
```

与其他模块配置相似，mod_prison.conf 为模块基础配置文件，用于指定规则文件路径，并可选择是否开启 debug 日志。

```ini
[Basic]
ProductRulePath = mod_prison/prison.data

[Log]
OpenDebug = false
```

示例的 prison.data 如下:

```json
{
	"Version": "20190101000000",
	"Config": {
		"example_product": [{
			"Name": "example_prison",
			"Cond": "req_path_prefix_in(\"/prison\", false)",
			"AccessSignConf": {
				"UseClientIP": false,
				"UseUrl": false,
				"UseHost": false,
				"UsePath": false,
				"UseHeaders": false,
				"UrlRegexp": "",
				"Query": [],
				"Header": [],
				"Cookie": [
					"UID"
				]
			},
			"Action": {
				"Cmd": "CLOSE",
				"Params": []
			},
			"CheckPeriod": 10,
			"StayPeriod": 10,
			"Threshold": 5,
			"AccessDictSize": 1000,
			"PrisonDictSize": 1000
		}]
	}
}
```
上述示例对路径为“/prison”的请求进行限流。其中，"AccessSignConf" 指示了限制的流量的维度，具体见后续描述。本例子中，将统计 cookies 中的 "UID"，限制不同 "UID" 的访问流量。

## 限制特定维度的流量

通过 AccessSignConf 字段，我们可以指定请求统计的维度，判断统计值是否超过限流阈值。
可配置的维度包括：

* UseSocketIP / UseConnectID

配置文件中支持这两个字段，但当前源码实现中未实际参与签名计算，可按需保留。

* UseClientIP

对请求按 clientIp 进行统计做限流。可以限制每个 clientIP 的请求速度。

* UseUrl

对请求按 URL 进行统计。可以限制对每一个 URL 的请求速度。

* UseHost

对请求按 Host 进行统计。可以限制对每一个 Host 的请求速度。

* UsePath

对请求按 Path 进行统计。可以限制对每一种 Path 的请求速度。

* UrlRegexp

对请求的 URL 做正则匹配，以匹配结果为维度进行统计。可以实现按 URL 中的部分内容来进行限流。正则表达式必须包含至少一个捕获组，模块将使用捕获组内容生成签名。

* Header

以请求的特定 Header 为维度来做统计流量。可以限制该 Header 所标识的每一种消息的请求速度。

* Cookie

以请求中的特定 Cookie 为维度进行统计。可以限制该 Cookie 标识的每一种请求的请求速度。比如，如果我们用 Cookie 中的 UID 来标识不同用户，可以通过指定 UID，来限制每个用户的访问速度。

* Query

以指定的 query 为维度，统计请求量来做限流。多个等价的 query key 可以用 `|` 分隔，例如 `uid|user_id`。

* UseHeaders

以每个请求中的所有 Header 为维度来进行统计（合并一个消息的所有 header）。限制不同 Header 的每一种消息的请求速度。

## 设置限流门限

通过下述参数配置限流：

* CheckPeriod

设置统计周期，单位秒。

* StayPeriod

被限流后的惩罚时长。在该时间段内，该维度访问请求都将被限制。

* Threshold

限制的阈值。一个维度的统计数量超过该阈值将触发限流。

* AccessDictSize

访问统计表的大小。统计表保存了当前配置的维度的所有统计值。比如，如果以 ClientIp 为维度进行的统计，该表包含每个 ClientIp 的访问量。

* PrisonDictSize

访问封禁表的大小。按维度统计后，每类命中限流的请求，都会被保存在封禁表中。所以，封禁表保存了当前所有处于封禁状态的某类请求。比如某频繁访问，被限流的 IP 地址等。

## 设置限流动作

某类请求命中限流规则后，可以在 "Action" 中配置对该类请求的后续动作。

* CLOSE

直接关闭请求的连接。

* PASS

正常转发，不做任何处理。

* FINISH

返回 403 响应并关闭用户连接。

* REQ_HEADER_SET

正常转发，在请求 header 中插入指定 key 和 value。key 和 value 在 Params 中指定；其中 key 必须以 `X-BFE-` 开头。

## 监控项

| 监控项 | 描述 |
| ------ | ---- |
| ALL_CHECKED | 被检查的请求总数 |
| ALL_PRISON | 被封禁的请求总数 |

## links
上一章：[第二十四章 配置 redirect](../../operation/configuration/redirect.md)  
下一章：[第二十六章 支持更多协议](../../operation/configuration/protocol.md)
