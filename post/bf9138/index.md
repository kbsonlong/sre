# openconnect设置用户组实现多路由




### 1.首先添加两个带分组的用户
```bash
ocpasswd -c /etc/ocserv/ocpasswd -g gruop1 user1 
ocpasswd -c /etc/ocserv/ocpasswd -g gruop2 user2
```

### 2.添加创建路由表组
```bash
mkdir /etc/ocserv/group 
echo -e "route = 10.10.0.0/255.255.255.0" >> /etc/ocserv/group/group1 
echo -e "no-route = 211.80.0.0/255.240.0.0" >> /etc/ocserv/group/group2
```
> 以上连个路由表是演示group1和group2随便写的 请自行添加路由规则
> 此外路由表里还可以写DNS 短线时间的参数
<!--more-->
### 3.添加新的配置到ocserv.conf
```bash
config-per-group = /etc/ocserv/group/ 
default-group-config = /etc/ocserv/group/group1 #如果创建用户的时候不分组 group1就是默认分组 用的就是group1的路由表 
default-select-group = group1 #如果创建用户的时候不分组 group1就是默认分组 用的就是group1的路由表 
auto-select-group = false
```


### 4.完整配置

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
config-per-group = /etc/ocserv/group/
default-group-config = /etc/ocserv/group/default
default-select-group = default
auto-select-group = false
cisco-client-compat = true
dtls-legacy = true
```

```bash
# cat /etc/ocserv/group/default 
route = 10.48.16.124/32

# cat /etc/ocserv/ocpasswd
sysop01:sysop:$5$R7HQXGtZ1MpB.N82$iNf5viGa/XT/qLjfpFhPNqlvdEyw5KKaKZvK2jIVEG4

```
