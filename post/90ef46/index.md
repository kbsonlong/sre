# ocserv部署


### 安装ocserv
```bash
apt-get install ocserv -y
```
### 申请免费证书
#### 修改dns指向服务器

#### 生成证书
```bash
certbot certonly --standalone --preferred-challenges http --agree-tos --email kbsonlong@gmail.com -d myvpn.alongparty.cn

# 续签证书
certbot renew --quiet --no-self-upgrade
```
<!--more-->
### 修改配置
```conf
auth = "plain[/etc/ocserv/ocpasswd]"
tcp-port = 443
udp-port = 443
run-as-user = nobody
run-as-group = daemon
socket-file = /var/run/ocserv-socket
server-cert = /etc/letsencrypt/live/myvpn.alongparty.cn/fullchain.pem
server-key = /etc/letsencrypt/live/myvpn.alongparty.cn/privkey.pem
isolate-workers = false
max-clients = 16
max-same-clients = 2
rate-limit-ms = 100
server-stats-reset-time = 604800
keepalive = 32400
dpd = 90
mobile-dpd = 1800
switch-to-tcp-timeout = 25
try-mtu-discovery = true
cert-user-oid = 0.9.2342.19200300.100.1.1
tls-priorities = "NORMAL:%SERVER_PRECEDENCE:%COMPAT:-VERS-SSL3.0:-VERS-TLS1.0:-VERS-TLS1.1"
auth-timeout = 240
min-reauth-time = 300
max-ban-score = 80
ban-reset-time = 1200
cookie-timeout = 300
deny-roaming = false
rekey-time = 172800
rekey-method = ssl
use-occtl = true
pid-file = /var/run/ocserv.pid
device = vpns
predictable-ips = true
default-domain = example.com
ipv4-network = 10.255.255.0
ipv4-netmask = 255.255.255.0
tunnel-all-dns = true
dns = 8.8.8.8
dns = 4.2.2.4
dns = 2001:4860:4860::8888
dns = 2001:4860:4860::8844
ping-leases = false
cisco-client-compat = true
dtls-legacy = true
```
