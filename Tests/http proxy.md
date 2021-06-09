# HTTP Proxy Testing

## ***DRAFT*** MissionView HTTP Proxy Test Case Design

As of version 1.6.0, MissionView includes support for a manually configured HTTP proxy, positioned between the NetAcquire Server and the MissionView host. This test case describes a reasonably thorough evaluation of MV functional support for an HTTP proxy. HTTPS proxy support is out-of-scope for this testing, but MV and NAS support of HTTPS must be tested when MV is configured to communicate over an HTTP proxy.

Test coverage is achieved by effecting each detail in each line of each test condition matrix. The right-most columns in the condition matrices headed by questions indicate expected test results. Lack of MV <-> NA Server communication may be indicated by server *offline* or *proxy-auth failure* status.

### Pre-Conditions, Constraints, Environment, & Assumptions
1. Supported pairing of MissionView <-> NetAcquire Server (e.g. MV v1.6.x <-> NAS v8.9.x).
2. NA Server configured for both IPv4 and IPv6 connectivity.
3. NA Server configured for `Require User Authentication`.
4. NA Server configured for `Encrypt API`.
5. NA Server signed certificate & CA certificate available for use.
6. Signed client certificate(s) available for use.
7. MV client hosts (Windows 10, Linux, MacOS) with network connectivity to NA Server via IPv4 and IPv6.
8. HTTP Proxy server, running and network-accessable to MV host and NA server.
9. HTTP Proxy host configured for both IPv4 and IPv6 connectivity. 
10. Administrator access to Proxy server (required for some test conditions).
11. HTTP Proxy administration skill and knowledge.
12. Except for Matrix ***D***, an MV instance is assumed to persist except when Network Settings IP or port are changed to non-null values, in which case MV prompts for restart. This detail is omitted from the condition matrices for brevity. 
13. Communication requirements between MV and the NA Server can be ramped-up by maintaining an MV layout with many applets and web pages open throughout these test conditions.

### Test Condition Matrix ***A*** -- Rudimentary HTTP Proxy Support
Condition# | Proxy Host IP:Port | Proxy Auth Required | MV Network Settings IP:Port | MV Proxy Auth Prompt? | MV <-> Server Communication?
----- | ------------ | ------------ | ------------- | ------------------ | ----------------
 1a | None/Down | N/A | None | No | Yes
 2a | None/Down | N/A | IPv4 : 3128 | No | No
 3a | None/Down | N/A | IPv6 : 3128 | No | No
 4a | IPv4 : 3128 | No | IPv4 : 3128 | No | Yes
 5a | IPv6 : 3128 | No | IPv6 : 3128 | No | Yes
 6a | IPv4 : 3128 | Yes | IPv4 : 3128 | Yes | Yes
 7a | IPv6 : 3128 | Yes | IPv6 : 3128 | Yes | Yes
 8a | IPv4 : 3128 | N/A | IPv4 : 312***9*** | No | No
 9a | IPv6 : 3128 | N/A | IPv6 : 312***9*** | No | No

### Test Condition Matrix ***B*** -- NA Server State Transitions
Preconditions for this matrix are any of conditions 4a through 7a above.

Condition# | NA Server HTTPS | NA Server Authentication | MV <-> Server Communication? 
----- | ------------ | ------------ | ------------- 
 1b | Not Selected | Password | Yes
 2b | Selected | Password | Yes
 3b | Selected | Certificate | Yes
 4b | Deselected | Password | Yes


### Test Condition Matrix ***C*** -- HTTP Proxy Server State Transitions
Preconditions for this matrix are any of conditions 4a through 7a, and 1b through 3b above.

Condition# | HTTP Proxy Server | MV <-> Server Communication? 
----- | -------------- | ------------- 
 1c | Down | No
 2c | Up | Yes
 3c | Down | No

### Test Condition Matrix ***D*** -- MissionView Restarts
Preconditions for this matrix are any of conditions 4a through 7a, and 2b or 3b above.

For each line in this matrix, MissionView is launched as a new process.

Condition# | HTTP Proxy Server | Proxy Auth Required | MV Network Settings IP:Port | MV Encrypted Communications Only | MV Proxy Auth Prompt? | MV <-> Server Communication?
----- | ------------ | ------------ | ------------- | ------------------ | ----------- | ------
 1d | Down | N/A | None | No | No | Yes
 2d | Down | N/A | None | Yes | No | Yes
 3d | Up | No | IPv4/6 : 3128 | No | No | Yes
 4d | Up | No | IPv4/6 : 3128 | Yes | No | Yes
 5d | Up | Yes | IPv4/6 : 3128 | No |  Yes | Yes 
 6d | Up | Yes | IPv4/6 : 3128 | Yes |  Yes | Yes

### Additional Test Conditions
1. For conditions 6a and 7a above, enter invalid username or password in MV Proxy Login prompt. Expect MV <-> NA Server communication failure.
2. Same as prior, except simply close the Proxy Login dialog. Expect MV <-> NA Server communication failure.

### Example Configuration for Squid version 3.5 on a CentOS 7 (Linux) Host
```
# IPv4 subnets
acl nanet src 172.20.0.0/16
acl labnet src 192.168.0.0/16

# IPv6 subnets
acl ipv6net src fd28:8b68:77ba:0000::/64

acl SSL_ports port 443
acl SSL_ports port 22

acl Safe_ports port 80          # http
acl Safe_ports port 443        # https
acl Safe_ports port 22          # ssh
acl CONNECT method CONNECT

# Deny requests to certain unsafe ports
http_access deny !Safe_ports

# Deny CONNECT to other than secure SSL ports
http_access deny CONNECT !SSL_ports

# Proxy authentication stuff
auth_param basic program /usr/lib64/squid/basic_ncsa_auth /etc/squid/passwd
auth_param basic children 5 startup=5 idle=1
auth_param basic realm Creds are admin:up9scale
auth_param basic credentialsttl 2 hours
auth_param basic casesensitive off

acl password proxy_auth REQUIRED

# Allowed
http_access allow nanet
http_access allow labnet
http_access allow ipv6net
http_access allow password

# Deny everything else
http_access deny all

# Listener port
http_port 3128

# Override natively configured DNS
append_domain .netacquire.dom
dns_nameservers 172.20.0.6
```