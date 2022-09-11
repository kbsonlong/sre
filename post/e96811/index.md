# openconnect双因素认证

### 修改认证方式

```bash
# vim /etc/ocserv/ocserv.conf
auth = "plain[passwd=/etc/ocserv/ocpasswd,otp=/etc/ocserv/ocserv.otp]"
```

### 配置pam
```bash
echo "auth requisite pam_oath.so debug usersfile=/etc/ocserv/ocserv.otp window=20" >> /etc/pam.d/ocserv
```
<!--more-->
### 创建用户OTP
```bash
username="zengshenglong"
company="Company"
site_name="Site"
key=$(head -c 16 /dev/urandom |xxd -c 256 -ps)
echo "HOTP/T30 ${username} - ${key}" >>/etc/ocserv/ocserv.otp 
oathtool --totp -w 5 $key
secret=$(echo 0x${key}|xxd -r -c 256|base32)
echo "otpauth://hotp/${username}@${site_name}?secret=${secret}&issuer=${company}" | qrencode -o - -t UTF8
echo "https://www.google.com/chart?chs=200x200&chld=M|0&cht=qr&chl=otpauth://hotp/${username}@${site_name}?secret=$(echo ${secret}|cut -d = -f 1)&issuer=${company}"
```


### 完整配置
```conf
auth = "plain[passwd=/etc/ocserv/ocpasswd,otp=/etc/ocserv/ocserv.otp]"
tcp-port = 443
udp-port = 443
run-as-user = nobody
run-as-group = daemon
socket-file = /var/run/ocserv-socket
server-cert = /etc/letsencrypt/live/myvpn.alongparty.cn/fullchain.pem
server-key = /etc/letsencrypt/live/myvpn.alongparty.cn/privkey.pem
isolate-workers = false
banner = "Welcome PCI DSS environment, please proceed with caution ! !"
pre-login-banner = "You will enter the PCI DSS environment, please proceed with caution ! !"
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
config-per-group = /etc/ocserv/group
default-group-config = /etc/ocserv/group/default
default-select-group = default
auto-select-group = false
cisco-client-compat = true
dtls-legacy = true
```

