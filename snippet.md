# Internal-Proxy
- /etc/squid/squid.conf
```shell
# Example rule allowing access from your local networks.
# Adapt to list your (internal) IP networks from where browsing
# should be allowed
acl localnet src 10.0.0.0/8     # RFC1918 possible internal network
acl localnet src 172.16.0.0/12  # RFC1918 possible internal network
acl localnet src 192.168.0.0/16 # RFC1918 possible internal network
acl localnet src fc00::/7       # RFC 4193 local private network range
acl localnet src fe80::/10      # RFC 4291 link-local (directly plugged) machines

acl SSL_ports port 443
acl Safe_ports port 80
acl Safe_ports port 443
acl Safe_ports port 9010        # Gitlab
acl Safe_ports port 9011        # Gitlab
acl CONNECT method CONNECT 

# Deny requests to certain unsafe ports
http_access deny !Safe_ports

# Deny CONNECT to other than secure SSL ports
# http_access deny CONNECT !SSL_ports

# Only allow cachemgr access from localhost 
http_access allow localhost manager 
http_access deny manager 

# Deny access from other networks.
http_access deny !localnet

# And finally allow all other access to this proxy 
http_access allow all

# Set Multi-Stage proxys
cache_peer 10.3.20.154 parent 3128 0 proxy-only no-query login=PASS

acl internal dstdomain "/etc/squid/direct_access_list"
cache_peer_access 10.3.20.154 allow !internal

always_direct allow internal
never_direct  allow !internal

#------------------------------------------ 
http_port 3128

# Leave coredumps in the first cache dir
coredump_dir /var/spool/squid
```
- /etc/squid/direct_access_list

External-Proxyに転送しないサイトを登録。
```shell
gitlab.local
10.2.137.201
gitlabs3poc-s3-s3bucket-1sl4slnh99ynr.s3.ap-northeast-1.amazonaws.com
```

# Client(docker)
- 証明書の登録

下記手順で証明書をOSに登録
```shell
#Proxyの自己証明書をLinux OSの共有システム証明書ストレージに登録
sudo cp external-proxy.crt /etc/pki/ca-trust/source/anchors
sudo update-ca-trust External
```

- Docker
    - /etc/systemd/system/docker.service.d/http-proxy.conf
    ```shell
    [Service]
    Environment="HTTP_PROXY=http://userb:HogeHoge@10.2.177.245:3128"
    Environment="HTTPS_PROXY=http://userb:HogeHoge@10.2.177.245:3128"
    Environment="NO_PROXY=localhost,127.0.0.1,169.254.169.254,169.254.169.123"
    ```
    - /etc/docker/daemon.json
    ```shell
    {
        "insecure-registries": [
            "gitlab.local:9011"
        ]
    }
    ```

# External-Proxy
- /etc/squid/squid.conf
```shell
# Example rule allowing access from your local networks.
# Adapt to list your (internal) IP networks from where browsing
# should be allowed
acl localnet src 10.0.0.0/8     # RFC1918 possible internal network
acl localnet src 172.16.0.0/12  # RFC1918 possible internal network
acl localnet src 192.168.0.0/16 # RFC1918 possible internal network
acl localnet src fc00::/7       # RFC 4193 local private network range
acl localnet src fe80::/10      # RFC 4291 link-local (directly plugged) machines

acl SSL_ports port 443 
acl Safe_ports port 443
acl CONNECT method CONNECT 

# Deny for Basic authorization
auth_param basic program /usr/lib64/squid/basic_ncsa_auth /etc/squid/.htpasswd
auth_param basic children 5
auth_param basic realm Squid Basic Authentication
auth_param basic credentialsttl 2 hours
auth_param basic casesensitive off
acl pauth proxy_auth REQUIRED
http_access deny !pauth

# Deny requests to certain unsafe ports
http_access deny !Safe_ports

# Deny CONNECT to other than secure SSL ports
http_access deny CONNECT !SSL_ports

# Only allow cachemgr access from localhost 
http_access allow localhost manager 
http_access deny manager 

# Deny access from other networks.
http_access deny !localnet

# Only allow access to whitelisted destinations.
#acl whitelist dstdomain "/etc/squid/whitelist" 
acl whitelist url_regex "/etc/squid/whitelist"
http_access allow whitelist

# And finally deny all other access to this proxy 
http_access deny all 

#------------------------------------------ 
http_port 10.3.20.154:3128 ssl-bump generate-host-certificates=on dynamic_cert_mem_cache_size=4MB cert=/etc/squid/ssl_cert/external-proxy.crt key=/etc/squid/ssl_cert/external-proxy.key

# Leave coredumps in the first cache dir
coredump_dir /var/spool/squid

# for ssl_bump
sslcrtd_program /usr/lib64/squid/ssl_crtd -s /var/lib/squid -M 20MB
sslproxy_cert_error allow all
ssl_bump stare all
```
- /etc/squid/whitelist
```shell
#for docker Hub
.docker.com
.docker.io
docker-images-prod.s3.amazonaws.com

#for pypi
bootstrap.pypa.io
pypi.org
files.pythonhosted.org

#for AWS CLI v2
awscli.amazonaws.com

#for google chrome
dl.google.com

#for debug
.google.co.jp

#allow aws services
sts.amazonaws.com
cloudformation.*.amazonaws.com
```
