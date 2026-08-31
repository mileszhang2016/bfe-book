# Configure Traffic Limiting
This chapter describes how to configure the traffic limiting function. This function will limit the rate of specific requests to protect backend services.

## Enable Traffic Limiting
In conf/bfe.conf, enable the module

```ini
Modules = mod_prison
```

## Configuration
The configuration is located in the directory conf/mod_prison/, and contains two files:

```
# ls
mod_prison.conf	prison.data
```

Similar to other module configurations, mod_prison.conf is the basic configuration file of the module. It specifies the prison rule file path and optionally enables debug logging.

```ini
[Basic]
ProductRulePath = mod_prison/prison.data

[Log]
OpenDebug = false
```

The example prison.data is as follows:

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
The above example restricts requests with the path "/prison". "AccessSignConf" indicates the dimensions used for traffic limiting. See the following description for details. In this example, the "UID" in cookies is counted to limit traffic for different UIDs.

## Limit Traffic for Specific Dimensions

Through the fields in "AccessSignConf", we can specify the dimension for counting requests and determine whether the count exceeds the traffic limiting threshold.

Configurable dimensions include:

* UseSocketIP / UseConnectID: These fields are supported in the configuration file, but the current source implementation does not actually include them in the signature calculation.

* UseClientIP: Limit requests by client IP. You can limit the request rate of each client IP.

* UseUrl: Count requests by URL. You can limit the rate of requests for each URL.

* UseHost: Count requests by Host. You can limit the rate of requests for each host.

* UsePath: Count requests by Path. You can limit the rate of requests for each path.

* UrlRegexp: Match the requested URL with a regular expression, and count based on the matching results. You can limit requests according to part of the URL content. The regular expression must contain at least one capturing group; the module uses the captured substrings to generate the signature.

* Header: Use a specific header of the request as the dimension to count traffic. You can limit the request rate of each request identified by the header.

* Cookie: Statistics are based on specific cookies in the request. You can limit the request rate of each request identified by the cookie. For example, if the UID in the cookie is used to identify different users, the access rate of each user can be limited by specifying UID.

* Query: Take the specified query keys as the dimension and count the number of requests to limit traffic. Multiple equivalent query keys can be separated by `|`, for example `uid|user_id`.

* UseHeaders: Use all headers in each request as the dimension for statistics (merging all headers of a request). Limit the request rate of each request with different headers.

## Set Threshold

Configure traffic limiting through the following parameters:

* CheckPeriod: Set the statistical period, in seconds.

* StayPeriod: The penalty duration after being restricted. During this time period, all requests for this dimension will be restricted.

* Threshold: The threshold of traffic limit. The traffic limit is triggered when the statistical quantity of a dimension exceeds the threshold.

* AccessDictSize: The size of the access statistics table. The statistics table saves all statistics of the currently configured dimension. For example, if the ClientIp dimension is used for statistics, the table contains the access statistics of each ClientIp.

* PrisonDictSize: The size of the blocking table. After counting by dimension, each type of request that hits the limit threshold will be saved in the blocking table. Therefore, the blocking table saves all currently blocked requests of a certain type, such as an IP address that is frequently accessed and restricted.

## Set Action

After a certain type of request hits the traffic limit rule, you can configure the subsequent action for this type of request in "Action".

* CLOSE: Close the connection directly.

* PASS: Forward normally without any processing.

* FINISH: Return a 403 response and close the connection.

* REQ_HEADER_SET: Forward normally, and insert the specified key and value into the request header. Key and value are specified in Params; the key must start with `X-BFE-`.

## Monitor

| Monitor Item | Description |
| ------------ | ----------- |
| ALL_CHECKED | Total number of checked requests |
| ALL_PRISON | Total number of blocked requests |

## links
Previous: [Chap24 Configure Redirect](../../../en_us/operation/config_redirect/config_redirect.md)  
Next: [Chap26 Support More Protocols](../../../en_us/operation/config_protocols/config_protocols.md)
