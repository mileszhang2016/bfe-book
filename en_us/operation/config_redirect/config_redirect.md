## Configure Redirect
This chapter describes how to configure HTTP redirect. This function will return a response with redirect code for the received request, indicating the client to redirect to a new URL.

## Enable Redirect
In conf/bfe.conf, enable the module
```ini
Modules = mod_redirect
```

## Configuration
The configuration is located in the directory conf/mod_redirect/, contains two files:

```
$ ls
mod_redirect.conf	redirect.data
```

mod_redirect.conf is the basic configuration file of the module, pointing to the redirect rule file.

```
$ cat mod_redirect.conf 
[basic]
DataPath = mod_redirect/redirect.data

[Log]
OpenDebug = false
```
`Log.OpenDebug` controls whether debug logs are output (disabled by default) and usually does not need to be modified.

redirect.data contains redirect rules and can be loaded dynamically. The sample configuration file in the installation package is as follows:

```
{
    "Version": "1",
    "Config": {
        "example_product": [
            {
                "Cond": "req_path_prefix_in(\"/redirect\", false)",
                "Actions": [
                    {
                        "Cmd": "URL_SET",
                        "Params": ["https://example.org"]
                    }
                ],
                "Status": 301
            }
        ]
    }
}
```
The above configuration adds a rule to product *example_product*: for requests that meet the condition identified by "Cond" (the prefix of the request path is "/redirect"), execute the action identified by "Actions" (redirect to https://example.org）, the returned HTTP response code is 301.

Notes:
- Each rule currently supports only one action in "Actions".
- "Status" is the HTTP redirect status code returned to the client; commonly used values are 301, 302, 303, 307, 308, or other valid HTTP redirect status codes.

## Actions
"Cmd" in "Actions" indicates how to set the URL in redirect.

### URL_SET
Redirect the request to the specified URL. The parameter is the redirected URL.

Example:
```json    
{
    "Cmd": "URL_SET", 
    "Params": ["http://www.example.com/more"]
}
```

Result:
| | |
|-|-|
|Request     | http://www.example.com/unknown |
|Redirect to  | http://www.example.com/more |

### URL_FROM_QUERY
Set  redirect address to the value of a key in query. The parameter is the name of the key in query.

Example:
```json    
{
    "Cmd": "URL_FROM_QUERY", 
    "Params": ["url"]
}
```

Result:
| | |
|-|-|
|Request   | http://www.example.com/redirect?url=http://news.example.com |
|Redirect to   | http://news.example.com |

### URL_PREFIX_ADD
Set the redirect URL by concatenating a specific prefix with the original URI (path + query). Parameter is the prefix to be added.

Example:
```json    
{
    "Cmd": "URL_PREFIX_ADD", 
    "Params": ["/v1"]
}
```

Result:
| | |
|-|-|
|Request   | http://www.example.com/test.html |
|Redirect to   | http://www.example.com/v1/test.html |

### SCHEME_SET
Set the scheme of the redirect URL to HTTP or HTTPS. The parameter is specified as http or https.

Example:
```json    
{
    "Cmd": "SCHEME_SET", 
    "Params": ["https"]
}
```

Result:
| | |
|-|-|
|Request   | http://www.example.com/index.html |
|Redirect to   | https://www.example.com/index.html |


## links
Previous: [Chap23 Configure Rewrite](../../../en_us/operation/config_rewrite/config_rewrite.md)  
Next: [Chap25 Configure Traffic Limiting](../../../en_us/operation/config_traffic_limit/config_traffic_limit.md)