# Модуль 1 — Настройка сетевой инфраструктуры

---

## Задание 1. Настройка имён хостов

Установка hostname на каждой машине:

```bash
hostnamectl set-hostname isp.au-team.irpo; exec bash
hostnamectl set-hostname hq-rtr.au-team.irpo; exec bash
hostnamectl set-hostname hq-srv.au-team.irpo; exec bash
hostnamectl set-hostname hq-cli.au-team.irpo; exec bash
hostnamectl set-hostname br-rtr.au-team.irpo; exec bash
hostnamectl set-hostname br-srv.au-team.irpo; exec bash
```

---

## Задание 1 (продолжение). Настройка IP-адресов

### Интерфейсы ISP

```bash
mkdir -p /etc/net/ifaces/enp7s{2,3}
echo 'TYPE=eth' | tee /etc/net/ifaces/enp7s{2,3}/options
echo '172.16.1.1/28' > /etc/net/ifaces/enp7s2/ipv4address
echo '172.16.2.1/28' > /etc/net/ifaces/enp7s3/ipv4address
systemctl restart network
ip -c --br a
```

### Интерфейсы BR-RTR

```bash
mkdir -p /etc/net/ifaces/{enp7s{1,2},gre1}
echo 'TYPE=eth' | tee /etc/net/ifaces/enp7s{1,2}/options

# ---- to ISP ----
echo '172.16.2.2/28' > /etc/net/ifaces/enp7s1/ipv4address
echo 'default via 172.16.2.1' > /etc/net/ifaces/enp7s1/ipv4route
echo 'nameserver 8.8.8.8' > /etc/net/ifaces/enp7s1/resolv.conf

# ---- to BR-SRV ----
echo '192.168.0.1/28' > /etc/net/ifaces/enp7s2/ipv4address

# ---- включение маршрутизации ----
vim /etc/net/sysctl.conf
systemctl restart network
ip -c --br a
```

### Интерфейсы BR-SRV

```bash
echo 'TYPE=eth' > /etc/net/ifaces/enp7s1/options
echo '192.168.0.2/28' > /etc/net/ifaces/enp7s1/ipv4address
echo 'default via 192.168.0.1' > /etc/net/ifaces/enp7s1/ipv4route
echo $'search au-team.irpo\nnameserver 192.168.100.2' > /etc/net/ifaces/enp7s1/resolv.conf
systemctl restart network
ip -c --br a
```

### Интерфейсы HQ-RTR

```bash
mkdir -p /etc/net/ifaces/{enp7s{1,2},vlan{100,200,999},gre1}
echo 'TYPE=eth' | tee /etc/net/ifaces/enp7s{1,2}/options

# ---- to ISP ----
echo '172.16.1.2/28' > /etc/net/ifaces/enp7s1/ipv4address
echo 'default via 172.16.1.1' > /etc/net/ifaces/enp7s1/ipv4route
echo 'nameserver 8.8.8.8' > /etc/net/ifaces/enp7s1/resolv.conf

# ---- настройка VLAN ----
echo $'100\n200\n999' | xargs -i bash -c 'echo -e "TYPE=vlan\nHOST=enp7s2\nVID={}" > /etc/net/ifaces/vlan{}/options'
cat /etc/net/ifaces/vlan999/options

echo '192.168.100.1/27' > /etc/net/ifaces/vlan100/ipv4address
echo '192.168.200.1/24' > /etc/net/ifaces/vlan200/ipv4address
echo '192.168.99.1/29' > /etc/net/ifaces/vlan999/ipv4address

# ---- включение маршрутизации ----
vim /etc/net/sysctl.conf
systemctl restart network
ip -c --br a
```

### Интерфейсы HQ-SRV

```bash
echo 'TYPE=eth' > /etc/net/ifaces/enp7s1/options
echo '192.168.100.2/27' > /etc/net/ifaces/enp7s1/ipv4address
echo 'default via 192.168.100.1' > /etc/net/ifaces/enp7s1/ipv4route
echo 'nameserver 8.8.8.8' > /etc/net/ifaces/enp7s1/resolv.conf
systemctl restart network
ip -c --br a
```

---

## Задание 2. NAT (маскарадинг) на ISP

Редактируем `/etc/net/sysctl.conf` — включаем `net.ipv4.ip_forward = 1`:

```bash
vim /etc/net/sysctl.conf
# net.ipv4.ip_forward = 1
systemctl restart network
apt-get update && apt-get install nftables nano -y
```

Создаём правило NAT:

```bash
nano /etc/nftables/nftables.nft
```

```
#!/usr/sbin/nft -f
flush ruleset

table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat;
        oifname "enp7s1" masquerade
    }
}
```

```bash
systemctl enable --now nftables
systemctl start nftables
nft flush ruleset
nft -f /etc/nftables/nftables.nft
nft list ruleset
```

Проверка:

```bash
sysctl net.ipv4.ip_forward
ping -c4 ya.ru
```

---

## Задание 3. Создание пользователей

### HQ-SRV

```bash
useradd -u 2026 sshuser
echo "sshuser:P@ssw0rd" | chpasswd
usermod -aG wheel sshuser
echo "WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL" > /etc/sudoers.d/sshuser
su -l sshuser
sudo id
exit
```

### BR-SRV

```bash
useradd -u 2026 sshuser
echo "sshuser:P@ssw0rd" | chpasswd
usermod -aG wheel sshuser
echo "WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL" > /etc/sudoers.d/sshuser
su -l sshuser
sudo id
exit
```

### HQ-RTR

```bash
useradd net_admin
echo "net_admin:P@ssw0rd" | chpasswd
usermod -aG wheel net_admin
apt-get update && apt-get install sudo -y
echo "WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL" > /etc/sudoers.d/net_admin
su -l net_admin
sudo id
exit
```

### BR-RTR

```bash
useradd net_admin
echo "net_admin:P@ssw0rd" | chpasswd
usermod -aG wheel net_admin
apt-get update && apt-get install sudo
echo "WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL" > /etc/sudoers.d/net_admin
su -l net_admin
sudo id
```

---

## Задание 4-5. Настройка SSH

### HQ-SRV

Редактируем `/etc/openssh/sshd_config`:

```
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/openssh/ssh_banner
```

```bash
echo "Authorized access only" | tee /etc/openssh/ssh_banner
systemctl restart sshd
```

Проверка с HQ-RTR:

```bash
ssh -p 2026 sshuser@192.168.100.2
exit
```

### BR-SRV

Редактируем `/etc/openssh/sshd_config`:

```
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/openssh/ssh_banner
```

```bash
echo "Authorized access only" | tee /etc/openssh/ssh_banner
systemctl restart sshd
```

Проверка с BR-RTR:

```bash
ssh -p 2026 sshuser@192.168.0.2
exit
```

---

## Задание 6. GRE-туннель

### BR-RTR

```bash
vim /etc/net/ifaces/gre1/options
```

```
TYPE=iptun
TUNTYPE=gre
TUNLOCAL=172.16.2.2
TUNREMOTE=172.16.1.2
TUNTTL=64
TUNOPTIONS='ttl 64'
```

```bash
echo "10.10.10.2/30" > /etc/net/ifaces/gre1/ipv4address
systemctl restart network
ip -br -c a
```

### HQ-RTR

```bash
vim /etc/net/ifaces/gre1/options
```

```
TYPE=iptun
TUNTYPE=gre
TUNLOCAL=172.16.1.2
TUNREMOTE=172.16.2.2
TUNTTL=64
TUNOPTIONS='ttl 64'
```

```bash
echo "10.10.10.1/30" > /etc/net/ifaces/gre1/ipv4address
systemctl restart network
ip -br -c a
ping 10.10.10.2 -c 3
```

---

## Задание 7. OSPF

### HQ-RTR

```bash
apt-get update && apt-get install frr -y
sed -i 's/ospfd=no/ospfd=yes/' /etc/frr/daemons ; grep ospf /etc/frr/daemons
vim /etc/frr/frr.conf
```

```
interface gre
 no ip ospf passive
exit
!
interface gre1
 ip ospf area 0
 ip ospf authentication
 ip ospf authentication-key P@ssw0rd
 no ip ospf passive
exit
!
interface vlan100
 ip ospf area 0
exit
!
interface vlan200
 ip ospf area 0
exit
!
interface vlan999
 ip ospf area 0
exit
!
router ospf
 passive-interface default
exit
```

```bash
systemctl enable --now frr
systemctl restart network
```

### BR-RTR

```bash
apt-get update && apt-get install frr -y
sed -i 's/ospfd=no/ospfd=yes/' /etc/frr/daemons ; grep ospf /etc/frr/daemons
vim /etc/frr/frr.conf
```

```
interface gre
 no ip ospf passive
exit
!
interface gre1
 ip ospf area 0
 ip ospf authentication
 ip ospf authentication-key P@ssw0rd
 no ip ospf passive
exit
!
interface enp7s2
 ip ospf area 0
exit
!
router ospf
 passive-interface default
exit
```

```bash
systemctl enable --now frr
systemctl restart network
```

---

## Задание 8. NAT на HQ-RTR и BR-RTR

```bash
apt-get install nftables nano -y
nano /etc/nftables/nftables.nft
```

```
#!/usr/sbin/nft -f
flush ruleset

table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat;
        oifname "enp7s1" masquerade
    }
}
```

```bash
systemctl enable --now nftables
systemctl restart nftables.service
```

---

## Задание 9. DNS/DHCP (dnsmasq) на HQ-RTR

```bash
apt-get install dnsmasq -y

# смена DNS
rm -f /etc/net/ifaces/enp7s1/resolv.conf
echo $'search au-team.irpo\nnameserver 192.168.100.2' > /etc/net/ifaces/vlan100/resolv.conf

# DHCP
sed -i 's/AUTO_LOCAL_RESOLVER=yes/AUTO_LOCAL_RESOLVER=no/' /etc/sysconfig/dnsmasq
grep AUTO_LOCAL_RESOLVER /etc/sysconfig/dnsmasq
nano /etc/dnsmasq.conf
```

```
port=0
interface=vlan200
listen-address=192.168.200.1
dhcp-authoritative
dhcp-range=interface:vlan200,192.168.200.2,192.168.200.2,255.255.255.240,6h
dhcp-option=3,192.168.200.1
dhcp-option=6,192.168.100.2
leasefile-ro
```

```bash
systemctl enable --now dnsmasq ; ss -lun | grep 67
systemctl restart network
cat /etc/resolv.conf
```

---

## Задание 10. DNS-сервер (BIND) на HQ-SRV

```bash
apt-get update && apt-get install bind bind-utils nano -y
echo $'search au-team.irpo\nnameserver 127.0.0.1' > /etc/net/ifaces/enp7s1/resolv.conf
rndc-confgen -a -c /etc/bind/rndc.key
```

Редактируем `/etc/bind/options.conf`:

```
options {
    listen-on { 127.0.0.1; 192.168.100.2; };
    forwarders { 77.88.8.7; 77.88.8.3; };
    recursion yes;
    allow-recursion { any; };
    allow-query { any; };
    dnssec-validation no;
    directory "/etc/bind/zone";
    dump-file "/var/run/named/named_dump.db";
    statistics-file "/var/run/named/named.stats";
    pid-file "/var/run/named/named.pid";
};

logging {
    category default { default_syslog; };
};

zone "au-team.irpo" {
    type master;
    file "au-team.irpo";
};

zone "168.192.in-addr.arpa" {
    type master;
    file "168.192.in-addr.arpa";
};
```

### Прямая зона

```bash
cp -r /etc/bind/zone/127.in-addr.arpa /etc/bind/zone/au-team.irpo
nano /etc/bind/zone/au-team.irpo
```

```
$TTL 1D
@ IN SOA au-team.irpo. root.au-team.irpo. (
    2025020600
    12H
    1H
    1W
    1H
)
@       IN NS    hq-srv.au-team.irpo.

hq-rtr  IN A     192.168.100.1
hq-srv  IN A     192.168.100.2
hq-cli  IN A     192.168.200.2
br-rtr  IN A     192.168.0.1
br-srv  IN A     192.168.0.2
docker  IN A     172.16.1.1
web     IN A     172.16.2.1
```

### Обратная зона

```bash
cp -r /etc/bind/zone/127.in-addr.arpa /etc/bind/zone/168.192.in-addr.arpa
nano /etc/bind/zone/168.192.in-addr.arpa
```

```
$TTL 1D
@ IN SOA au-team.irpo. root.au-team.irpo. (
    2025020600
    12H
    1H
    1W
    1H
)
@       IN NS    au-team.irpo.

1.100   IN PTR   hq-rtr.au-team.irpo.
2.100   IN PTR   hq-srv.au-team.irpo.
2.200   IN PTR   hq-cli.au-team.irpo.
```

```bash
chown :named /etc/bind/zone/au-team.irpo /etc/bind/zone/168.192.in-addr.arpa
systemctl enable --now bind
service network restart
systemctl restart bind.service
host br-rtr
host -t PTR 192.168.100.2
```

---

## Задание 11. Настройка часового пояса

```bash
apt-get update && apt-get install tzdata -y
timedatectl set-timezone Asia/Novosibirsk
timedatectl
```
