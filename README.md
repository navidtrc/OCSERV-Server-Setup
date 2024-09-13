# OCSERV (AnyConnect & OpenConnect VPN Server) + User Management

## Requirements:

1- Server A: Foreign server (Ubuntu 20.0.4)

2- Server B: Iran server for tunneling for some ISP like Irancell (Ubuntu 20.0.4)

3- Domain with 2 sub address

4- occtl command tools


## Installation :

Step 0: Create 2 sub domain DNS (Iran & Foreign)  (NOTICE: temporally enter foreign IP for both Iran and foreign sub domain DNS)

## Server A: Foreign server:

1- Update server & reboot
```bash
>>> apt update && apt upgrade -y

>>> reboot
```

2- Install ocserv + certbot
```bash
>>> apt install ocserv certbot -y
```

3- Get certificate
```bash
>>> sudo certbot certonly --standalone --preferred-challenges http --agree-tos --email <EMAIL> -d <IRAN SUB DOMAIN>
```
then press N

4- In Cloudflare panel change Iran sub domain IP to Iran server IP

5- Change ocserv.conf
```bash
>>> nano /etc/ocserv/ocserv.conf
```
copy these content
```bash
auth = "plain[passwd=/etc/ocserv/ocpasswd]"
tcp-port = 443
run-as-user = nobody
run-as-group = daemon
socket-file = /run/ocserv.socket
server-cert = /etc/letsencrypt/live/<IR DOMAIN ADDRESS>/fullchain.pem
server-key = /etc/letsencrypt/live/<IR DOMAIN ADDRESS>/privkey.pem
ca-cert = /etc/ssl/certs/ssl-cert-snakeoil.pem
isolate-workers = true
max-clients = 0
max-same-clients = 2
server-stats-reset-time = 604800
keepalive = 30
dpd = 60
mobile-dpd = 300
switch-to-tcp-timeout = 25
try-mtu-discovery = true
cert-user-oid = 0.9.2342.19200300.100.1.1
compression = true
no-compress-limit = 256
tls-priorities = "NORMAL:%SERVER_PRECEDENCE:%COMPAT:-RSA:-VERS-SSL3.0:-ARCFOUR-128"
auth-timeout = 240
min-reauth-time = 300
max-ban-score = 0
ban-reset-time = 300
cookie-timeout = 300
deny-roaming = false
rekey-time = 172800
rekey-method = ssl
use-occtl = true
pid-file = /run/ocserv.pid
device = vpns
predictable-ips = true
default-domain = ir.techradio403.com
ipv4-network = 10.10.10.0
ipv4-netmask = 255.255.255.0
tunnel-all-dns = true
dns = 8.8.8.8
dns = 1.1.1.1
ping-leases = false
cisco-client-compat = true
dtls-legacy = true
```

6- Restart ocserv
```bash
>>> systemctl restart ocserv.service
>>> systemctl status ocserv.service
```

7- Create settings.sh in root path
```bash
>>> nano settings.sh
```
copy these content
```bash
#!/bin/bash

# ocserv-os-configs-commands-on-ubuntu-server

# ANSI escape codes for green text, yellow text, and resetting color
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
NC='\033[0m' # No Color

# Set your network interface name
NETWORK_INTERFACE="eth0"
echo -e "${GREEN}Your Network interface is $NETWORK_INTERFACE${NC}"

# Define your private network subnet
PRIVATE_SUBNET="10.10.10.0/24"
echo -e "${GREEN}Your private subnet is $PRIVATE_SUBNET${NC}"

# Enable IP Forwarding
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/60-custom.conf
echo -e "${GREEN}IP Forwarding Enabled${NC}"

# Enable TCP BBR algorithm to boost TCP speed
echo "net.core.default_qdisc=fq" | sudo tee -a /etc/sysctl.d/60-custom.conf
echo "net.ipv4.tcp_congestion_control=bbr" | sudo tee -a /etc/sysctl.d/60-custom.conf
echo -e "${GREEN}BBR algorithm Enabled${NC}"

# Apply the changes sysctl settings
sudo sysctl -p /etc/sysctl.d/60-custom.conf
echo -e "${GREEN}sysctl settings applied${NC}"

# Configure NAT (Masquerade)
sudo iptables -t nat -A POSTROUTING -s $PRIVATE_SUBNET -o $NETWORK_INTERFACE -j MASQUERADE
echo -e "${GREEN}NAT configured${NC}"

# Allow packet forwarding for the private network
sudo iptables -A FORWARD -s $PRIVATE_SUBNET -j ACCEPT
sudo iptables -A FORWARD -d $PRIVATE_SUBNET -j ACCEPT
echo -e "${GREEN}Packet Forwarding is allowed now${NC}"

# Create the /etc/iptables directory if it doesn't exist
sudo mkdir -p /etc/iptables

# Save IPv4 iptables rules to /etc/iptables/rules.v4
sudo apt-get install iptables-persistent -y
sudo iptables-save > /etc/iptables/rules.v4
echo -e "${GREEN}iptables configuration and rules saved to /etc/iptables/rules.v4${NC}"

# Checking NAT rule
echo -e "${YELLOW}Check if the NAT rule appears below...${NC}"
sudo iptables -t nat -L POSTROUTING
```

8- run settings.bash
```bash
>>> bash settings.sh
```

9- Install stunnel
```bash
>>> apt install stunnel4 -y
```

10- Change stunnel.conf
```bash
>>> nano /etc/stunnel/stunnel.conf
```
copy these content
```bash
cert = /etc/stunnel/stunnel.pem
pid = /etc/stunnel/stunnel.pid
output = /etc/stunnel/stunnel.log

[Cisco]
accept = 990
connect = 0.0.0.0:443
```

11- stunnel SSL
```bash
>>> cd /etc/letsencrypt/live/<IR Domain Address>
>>> cat privkey.pem fullchain.pem >> /etc/stunnel/stunnel.pem
>>> chmod 0400 /etc/stunnel/stunnel.pem
>>> cd ~
```

12- Create stunnel service
```bash
>>> nano /usr/lib/systemd/system/stunnel.service
```
copy these content
```bash
[Unit]
Description=SSL tunnel for network daemons
After=network.target
After=syslog.target

[Install]
WantedBy=multi-user.target
Alias=stunnel.target

[Service]
Type=forking
ExecStart=/usr/bin/stunnel /etc/stunnel/stunnel.conf
ExecStop=/usr/bin/pkill stunnel

# Give up if ping don't get an answer
TimeoutSec=600

Restart=always
PrivateTmp=false
```


13- Restart stunnel
```bash
>>> sudo systemctl start stunnel.service
>>> sudo systemctl enable stunnel.service
>>> sudo systemctl status stunnel.service
```
## Recommend: (add 2 floating IP and use them instead of your own IP. first IP for using direct VPN like MCI, and second IP for Iran server stunnel)
```bash
>>> nano /etc/netplan/60-floating-ip.yaml
```
copy these content
```bash
network:
   version: 2
   renderer: networkd
   ethernets:
     eth0:
       addresses:
       - <ip1 DIRECT>/32 
       - <ip2 STUNNEL>/32

```


## Server B: Iran server (Tunnel Server):

1- Update server & reboot
```bash
>>> apt update && apt upgrade -y

>>> reboot
```

2- Install stunnel
```bash
>>> apt install stunnel4 -y
```

3- Change stunnel.conf
```bash
>>> nano /etc/stunnel/stunnel.conf
```
copy these content
```bash
pid = /etc/stunnel/stunnel.pid
client = yes
output = /etc/stunnel/stunnel.log

[Cisco]
accept = 445
connect = <Foreign sub domain>:990
```

4- Create stunnel service
```bash
>>> nano /usr/lib/systemd/system/stunnel.service
```
copy these content
```bash
[Unit]
Description=SSL tunnel for network daemons
After=network.target
After=syslog.target

[Install]
WantedBy=multi-user.target
Alias=stunnel.target

[Service]
Type=forking
ExecStart=/usr/bin/stunnel /etc/stunnel/stunnel.conf
ExecStop=/usr/bin/pkill stunnel

# Give up if ping don't get an answer
TimeoutSec=600

Restart=always
PrivateTmp=false
```

5- Restart stunnel
```bash
>>> sudo systemctl start stunnel.service
>>> sudo systemctl enable stunnel.service
>>> sudo systemctl status stunnel.service
```
