# Configure HTTPS Service
BFE can support TLS offloading, i.e., decrypt the encrypted HTTPS traffic received, and then forward it to the backend service.

![https](./https_1.png)

## HTTPS Basic Configuration

HTTPS configuration includes the configuration of cipher suite, certificate file, and TLS protocol related parameters. To use HTTPS, you need to configure the corresponding certificate file. Please prepare the certificate file in advance.

### HTTPS Port
HTTPS ports can be configured in bfe.conf.

```ini
[Server] 
... 

# listen port for https request
HttpsPort = 8443
```

### HTTPS Basic Parameters
In the `[HttpsBasic]` section of bfe.conf, you can configure basic TLS parameters such as TLS versions, curve preferences, SSLv2 ClientHello compatibility, and client CA/CRL directories, in addition to cipher suites.

```ini
[HttpsBasic]
# supported curve preference
CurvePreferences=CurveP256

# supported min/max TLS versions
MinTlsVersion = VersionSSL30
MaxTlsVersion = VersionTLS12

# whether to support SSLv2 ClientHello (for compatibility with legacy clients)
EnableSslv2ClientHello = true

# client CA certificate and CRL directories
ClientCABaseDir = tls_conf/client_ca
ClientCRLBaseDir = tls_conf/client_crl
```

### Cipher Suite
The `CipherSuites` field in the `[HttpsBasic]` section of bfe.conf defines the cipher suites supported by TLS. Users can modify it as needed.

```ini
[HttpsBasic]
CipherSuites=TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256|TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256
CipherSuites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256|TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256
CipherSuites=TLS_ECDHE_RSA_WITH_RC4_128_SHA
CipherSuites=TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA
CipherSuites=TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA
CipherSuites=TLS_RSA_WITH_RC4_128_SHA
CipherSuites=TLS_RSA_WITH_AES_128_CBC_SHA
CipherSuites=TLS_RSA_WITH_AES_256_CBC_SHA
```

### Server Certificate
The bfe.conf also contains the path of the configuration file of the server certificate. You can directly use the default configuration.

```ini
ServerCertConf = tls_conf/server_cert_conf.data
```

Open the server certificate configuration file tls_conf/server_cert_conf.data, you can see the following certificate information:

```json
{
    "Version": "1",
    "Config": {
        "Default": "example.org.cert",
        "CertConf": {
            "example.org.cert": {
                "ServerCertFile": "tls_conf/certs/server.crt",
                "ServerKeyFile" : "tls_conf/certs/server.key",
                "OcspResponseFile": "tls_conf/certs/server.ocsp"
            }
        }
    }
}
```

This file contains the information of the server certificate, where the "Default" field indicates the default certificate of the server. "CertConf" contains the specific information of the certificate file. In the above example, a certificate named "example.org.cert" is configured.

"CertConf" can contain multiple certificates, which are identified by different names. The definition of each certificate contains two fields, "ServerCertFile" and "ServerKeyFile", indicating the server certificate file and the corresponding private key file respectively. It can also optionally include the "OcspResponseFile" field to enable OCSP Stapling by specifying the path to the certificate's OCSP response file.

The certificate name, such as "example.org.cert" in the above example, will be used in the following configuration in tls_rule_conf.data.

### TLS Rules

After preparing the certificate, you need to set TLS-related rules, such as how to select the certificate.

bfe.conf contains the path to the TLS rule file:

```ini
TlsRuleConf = tls_conf/tls_rule_conf.data
```

The following example is the rule we prepared for the tenant "example_product".

```json
{
    "Version": "1",
    "DefaultNextProtos": ["http/1.1"],
    "DefaultChacha20": false,
    "DefaultDynamicRecord": false,
    "Config": {
        "example_product": {
            "VipConf": ["10.199.4.14"],
            "SniConf": ["example.org"],
            "CertName": "example.org.cert",
            "NextProtos": [
                "h2",
                "http/1.1"
            ],
            "Grade": "C",
            "Chacha20": false,
            "DynamicRecord": false
        }
    }
}
```

"VipConf" indicates the TLS rule matched by VIP address; "SniConf" indicates the domain name using SNI. "CertName" specifies the certificate to use. "NextProtos" identifies the protocol used in ALPN negotiation, supporting `h2`, `spdy/3.1`, `http/1.1`, `stream`, and parameterized syntax such as `h2;level=0;mcs=200;isw=65535;rate=100;pp=1`. The security grade "Grade" defines the cipher suite that can be used by the server in TLS negotiation. See the following chapters for details. "Chacha20" controls whether to prefer ChaCha20 cipher suites, and "DynamicRecord" controls whether to enable dynamic TLS record size; when not configured, they inherit the top-level "DefaultChacha20" and "DefaultDynamicRecord" respectively.

After the above configuration is ready, restart BFE to make the configuration take effect. You can access the configured HTTPS port through HTTPS.

## Configure TLS Session Reuse

BFE supports two ways of TLS session reuse: Session Cache and Session Ticket.

### Configure Session Cache
BFE uses Redis for centralized TLS session information storage. Multiple BFE instances can connect to the shared Redis service. The connection is as follows:

![https session cache](./https_session_cache.png)

To enable session cache, set SessionCacheDisabled to false in bfe.conf.

```ini
[SessionCache]
# disable tls session cache or not
SessionCacheDisabled = false

# tcp address of redis server
Servers = "example.redis.cluster"

# prefix for cache key
KeyPrefix = "bfe"

# connection params (ms)
ConnectTimeout = 50
ReadTimeout = 50
WriteTimeout = 50

# max idle connections in connection pool
MaxIdle = 20

# expire time for tls session state (second)
SessionExpire = 3600

```

The "Servers" field in the above configuration indicates the name of the Redis server. This name must be resolved through the name service configuration file specified by `NameConf` in the `[Server]` section of bfe.conf (e.g., `NameConf = server_data_conf/name_conf.data`), and the specific address is specified in that configuration file (e.g., conf/server_data_conf/name_conf.data):

```json
{
    "Version": "1",
    "Config": {
        "example.redis.cluster": [
            {
                "Host": "192.168.3.1",
                "Port": 6379,
                "Weight": 10
            }
        ]
    }
}
```

### Configure Session Ticket

To enable TLS session tickets, set SessionTicketsDisabled to false in the bfe.conf file:

```
[SessionTicket]
# disable tls session ticket or not
SessionTicketsDisabled = false
# session ticket key
SessionTicketKeyFile = tls_conf/session_ticket_key.data
```
In the file pointed to by SessionTicketKeyFile, users can configure the key for encrypting the ticket. The format is a 96-character hexadecimal string representing 48 raw bytes, with characters limited to 0-9 and a-f. A raw 48-byte binary file is also supported as a fallback when JSON parsing fails.

After the above configuration is enabled, if the client supports session ticket, session reuse based on session ticket can be realized in TLS handshake.

## Configure Mutual Authentication 
In some scenarios, we need to configure mTLS(mutual TLS) to authenticate clients. Client certificates can be configured on BFE.

The directory of clientCA certificate can be configured in bfe.conf:

```ini

# client ca certificates base directory
# Note: filename suffix for ca certificate file should be ".crt", eg. example_ca_bundle.crt
ClientCABaseDir = tls_conf/client_ca

# client certificate crl base directory
# Note: filename suffix for crl file should be ".crl", eg. example_ca_bundle.crl
ClientCRLBaseDir = tls_conf/client_crl

```

Place the client's CA certificate in conf/tls_conf/client_ca, and the certificate name should end with .crt, as follows:

```
# ls conf/tls_conf/client_ca
example_ca.crt
```

To configure CRLs, create a subdirectory named after the CA under ClientCRLBaseDir and place the `.crl` files in that subdirectory, for example `conf/tls_conf/client_crl/example_ca/example_ca.crl`.

Modify  the configuration of tenant "example_product" in conf/tls_conf/tls_rule_conf.data, "ClientAuth" is set to true, "ClientCAName" is the name of the above certificate file, in this case "example_ca".

```json
{
    ...
    "Config": {
        "example_product": {
            ...
            "ClientAuth": true,
            "ClientCAName": "example_ca"
        }
    }
}
```

Note that in the current implementation, if BFE is required to verify the client certificate, "clientAuth" must be enabled in "Extended Key Usage". You can view the certificate information for confirmation. See the details of client.crt as follows:

```
# openssl x509 -in client.crt -text -noout

Certificate:
    ...
        X509v3 extensions:
            X509v3 Basic Constraints: 
                CA:FALSE
            X509v3 Key Usage: 
                Digital Signature, Non Repudiation, Key Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication, TLS Web Client Authentication, Code Signing, E-mail Protection
    ...
```

After the configuration is completed, the client can use the above certificate client.crt and the corresponding private key to access the HTTPS service port.

## Security Grade

The configuration file for TLS contains the configuration item "Grade", which specifies the security level of the cipher suite used.

BFE supports multiple security grades (A+/A/B/C). The difference of each security grade lies in the supported protocol version and cipher suite. Grade A+ has the highest security and the lowest connectivity; Grade C has the lowest security and the highest connectivity.

* Grade A+

| Supported Protocols | Supported Cipher Suites |
| ------------------- | ----------------------- |
| TLS1.2 | TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256<br>TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256<br>TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256<br>TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256<br>TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA<br>TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA<br>TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA<br>TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA<br>TLS_RSA_WITH_AES_128_CBC_SHA<br>TLS_RSA_WITH_AES_256_CBC_SHA |


* Grade A

| Supported Protocols | Supported Cipher Suites |
| ------------------- | ----------------------- |
| TLS1.2<br>TLS1.1<br>TLS1.0 | TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256<br>TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256<br>TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256<br>TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256<br>TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA<br>TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA<br>TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA<br>TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA<br>TLS_RSA_WITH_AES_128_CBC_SHA<br>TLS_RSA_WITH_AES_256_CBC_SHA |

* Grade B

| Supported Protocols | Supported Cipher Suites |
| ------------------- | ----------------------- |
| TLS1.2<br>TLS1.1<br>TLS1.0 | TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256<br>TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256<br>TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256<br>TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256<br>TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA<br>TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA<br>TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA<br>TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA<br>TLS_RSA_WITH_AES_128_CBC_SHA<br>TLS_RSA_WITH_AES_256_CBC_SHA |
| SSLv3 | TLS_ECDHE_RSA_WITH_RC4_128_SHA<br>TLS_ECDHE_ECDSA_WITH_RC4_128_SHA<br>TLS_RSA_WITH_RC4_128_SHA |

**Note:** The ECDSA RC4 suite under SSLv3 is available only in `onlyRC4` mode.

* Grade C

| Supported Protocols | Supported Cipher Suites |
| ------------------- | ----------------------- |
| TLS1.2<br>TLS1.1<br>TLS1.0 | TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256<br>TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256<br>TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256<br>TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256<br>TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA<br>TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA<br>TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA<br>TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA<br>TLS_RSA_WITH_AES_128_CBC_SHA<br>TLS_RSA_WITH_AES_256_CBC_SHA<br>TLS_ECDHE_RSA_WITH_RC4_128_SHA<br>TLS_ECDHE_ECDSA_WITH_RC4_128_SHA<br>TLS_RSA_WITH_RC4_128_SHA |
| SSLv3 | TLS_ECDHE_RSA_WITH_RC4_128_SHA<br>TLS_ECDHE_ECDSA_WITH_RC4_128_SHA<br>TLS_RSA_WITH_RC4_128_SHA |




## links
Previous: [Chap21 Configure Load Balancing Algorithm And Session Stickiness](../../../en_us/operation/config_scheduling/config_scheduling.md)  
Next: [Chap23 Configure Rewrite](../../../en_us/operation/config_rewrite/config_rewrite.md)