# Модуль 3 — Безопасность и мониторинг

---

## Задание 1. Импорт пользователей в домен

Выполняется на **BR-SRV**.

```bash
mount -o loop /dev/sr0 /mnt/ -v
ls -l /mnt/
iconv -f UTF-8 -t UTF-8//IGNORE /mnt/Users.csv > /root/users_fix.csv
```

Скрипт импорта (`import.sh`):

```bash
cat << "EOF" > import.sh
#!/bin/bash

while IFS=';' read -r name fam role phone ou street zip city country pass; do
    username="$fam"."${name:0:1}"
    samba-tool user create "${username,,}" "P@ssw0rd1" \
        --given-name="$fam" \
        --surname="$name" --job-title="$role" \
        --telephone-number="$phone"
    [[ -n "$ou" ]] && samba-tool ou create "OU=$ou" 2>/dev/null
    [[ -n "$ou" ]] && samba-tool user move "${username,,}" "OU=$ou" 2>/dev/null
done < users_fix.csv
EOF

bash import.sh
samba-tool user list
samba-tool user show *имя_пользователя*
```

Проверка: зайти через GUI на HQ-CLI под любым импортированным пользователем (пароль `P@ssw0rd1`).

---

## Задание 2. Центр сертификации (CA)

### ISP — создание CA и сертификатов

```bash
echo "PermitRootLogin yes" >> /etc/openssh/sshd_config
systemctl restart sshd
```

Установка ПО:

```bash
apt-get install openssl openssl-engines -y
apt-get install openssl-gost-engine -y
control openssl-gost enabled

openssl engine
openssl ciphers | tr ':' '\n' | grep GOST
```

Создание корневого CA:

```bash
openssl genpkey -algorithm gost2012_256 \
    -pkeyopt paramset:TCB -out ca.key

openssl req -new -x509 -md_gost12_256 \
    -days 90 -key ca.key -out ca.crt
# Common Name: ROOT-CA.AU-TEAM.IRPO
```

Создание ключей для сайтов:

```bash
openssl genpkey -algorithm gost2012_256 \
    -pkeyopt paramset:A -out web.au-team.irpo.key

openssl genpkey -algorithm gost2012_256 \
    -pkeyopt paramset:A -out docker.au-team.irpo.key
```

Создание CSR:

```bash
openssl req -new -md_gost12_256 \
    -key web.au-team.irpo.key \
    -out web.au-team.irpo.csr
# Common Name: WEB.AU-TEAM.IRPO

openssl req -new -md_gost12_256 \
    -key docker.au-team.irpo.key \
    -out docker.au-team.irpo.csr
# Common Name: DOCKER.AU-TEAM.IRPO
```

Подпись сертификатов:

```bash
openssl x509 -req -in web.au-team.irpo.csr \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -out web.au-team.irpo.crt -days 30

openssl x509 -req -in docker.au-team.irpo.csr \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -out docker.au-team.irpo.crt -days 30

ls -l
```

### ISP — настройка nginx с SSL

Приводим `/etc/nginx/sites-available.d/r-proxy.conf` к виду:

```nginx
server {
    listen 443 ssl;
    server_name web.au-team.irpo;

    ssl_certificate /root/web.au-team.irpo.crt;
    ssl_certificate_key /root/web.au-team.irpo.key;
    ssl_ciphers GOST2012-GOST8912-GOST8912;
    ssl_protocols TLSv1.2;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://172.16.1.10:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        auth_basic "Restricted Access";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }
}

server {
    listen 443 ssl;
    server_name docker.au-team.irpo;

    ssl_certificate /root/docker.au-team.irpo.crt;
    ssl_certificate_key /root/docker.au-team.irpo.key;
    ssl_ciphers GOST2012-GOST8912-GOST8912;
    ssl_protocols TLSv1.2;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://172.16.2.10:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
nginx -t
systemctl restart nginx
```

### HQ-CLI — установка сертификата и КриптоПро

```bash
scp root@172.16.1.1:~/ca.crt /etc/pki/ca-trust/source/anchors/
update-ca-trust extract

apt-get install cryptopro-preinstall -y
echo "https://disk.yandex.ru/d/vgZJjOsNfH8szw" > /home/user/link.txt
```

В **GUI HQ-CLI**:
1. Скачать архив по ссылке, распаковать в домашнюю директорию
2. `./install_gui.sh`
3. Выбрать пакеты:
   - Криптопровайдер КС1
   - Графические диалоги
   - cptools, многоцелевое графическое приложение
   - Браузерный плагин + CAdES
   - Импортировать корневые сертификаты из ОС
4. Установить, пропустить ввод ключа
5. Перезапустить браузер
6. Открыть `https://web.au-team.irpo` и `https://docker.au-team.irpo`

> **Внимание!** Если ошибка/предупреждение о ненадёжности — сертификат не подтянулся, нужно установить вручную:
> - Меню -> Инструменты КриптоПро
> - Сертификаты -> Установить сертификаты
> - Путь: `/etc/pki/ca-trust/source/anchors/`

---

## Задание 3. OpenVPN-туннель (замена GRE)

### HQ-RTR — генерация ключа

```bash
apt-get update && apt-get install openvpn -y
openvpn --genkey secret /etc/openvpn/keys/static.key
chmod og-rw /etc/openvpn/keys/static.key
cat /etc/openvpn/keys/static.key
# Копируем всё содержимое
```

### BR-RTR — настройка клиента

```bash
apt-get update && apt-get install openvpn -y

cat > /etc/openvpn/keys/static.key
# Вставляем скопированный ключ
chmod og-rw /etc/openvpn/keys/static.key

vim /etc/openvpn/client/tun0.conf
```

```
remote 172.16.1.10
dev tun0
cipher AES-256-CBC
auth-nocache
ifconfig 192.168.5.2 192.168.5.1
secret /etc/openvpn/keys/static.key
```

```bash
systemctl enable openvpn-client@tun0
rm -rf /etc/net/ifaces/gre1/ && reboot
```

### HQ-RTR — настройка сервера

```bash
vim /etc/openvpn/server/tun0.conf
```

```
dev tun0
cipher AES-256-CBC
auth-nocache
ifconfig 192.168.5.1 192.168.5.2
secret /etc/openvpn/keys/static.key
```

```bash
systemctl enable openvpn-server@tun0
rm -rf /etc/net/ifaces/gre1/ && reboot
```

### Обновление маршрутизации (на обоих)

```bash
sed -i 's/interface gre1/interface tun0/' /etc/frr/frr.conf
grep tun0 -A6 /etc/frr/frr.conf
systemctl restart frr
ip -br -c a
ip r
```

---

## Задание 4. Межсетевой экран (nftables)

> **Внимание! Задание ломает сеть.**

Выполняется на **HQ-RTR** и **BR-RTR**.

```bash
nft add table filter
nft add chain filter INPUT '{ type filter hook input priority 0; policy drop ; }'

nft add rule filter INPUT iif "lo" accept
nft add rule filter INPUT ct state established,related accept
nft add rule filter INPUT iif enp7s1 icmp type { echo-request, echo-reply } accept
nft add rule filter INPUT iif enp7s1 udp dport { 53, 123 } accept
nft add rule filter INPUT iif enp7s1 tcp dport { 80, 443, 2026, 8080, 53 } accept

nft add chain filter FORWARD '{ type filter hook forward priority 0; policy drop ; }'
nft add rule filter FORWARD ct state established,related accept
nft add rule filter FORWARD iif enp7s2 oif enp7s1 accept
nft add rule filter FORWARD iif enp7s2 oif tun0 accept
nft add rule filter FORWARD iif enp7s1 oif enp7s2 ct state established,related accept

nft add chain filter OUTPUT '{ type filter hook output priority 0; policy accept ; }'

nft list ruleset
nft list ruleset > /etc/nftables/nftables.nft
systemctl restart nftables
nft list ruleset
```

---

## Задание 5. Принт-сервер CUPS

### HQ-SRV

```bash
apt-get update && apt-get install cups cups-pdf -y
vim /etc/cups/cupsd.conf
```

```
<Location />
<Location /admin>
<Location /admin/conf>
Allow all
Listen 192.168.1.10:631
```

```bash
systemctl enable --now cups
systemctl restart cups
ss -ltnp4 | grep 631
```

### HQ-CLI (GUI)

Меню -> Параметры печати -> Удаляем принтер -> Добавить новый -> Поиск сетевого принтера -> `192.168.1.10` -> Отменяем вход -> Выбираем принтер из списка -> Далее (везде).

---

## Задание 6. Логирование (rsyslog)

### BR-SRV, HQ-RTR, BR-RTR — клиенты

```bash
apt-get install rsyslog -y
grep 'Syslog' /etc/systemd/journald.conf
vim /etc/rsyslog.d/00_common.conf
```

Раскомментировать:

```
module(load="imjournal") # ...
module(load="imuxsock") # ...
```

```bash
echo -e "ForwardToSyslog=yes\nMaxLevelSyslog=warning" >> /etc/systemd/journald.conf
echo "*.warn @192.168.1.10" > /etc/rsyslog.d/10_to_server.conf

systemctl restart systemd-journald
systemctl enable --now rsyslog
```

### HQ-SRV — сервер

```bash
apt-get install rsyslog-classic rsyslog-server-listen -y

cat << "EOF" > /etc/rsyslog.d/91_template.conf
$template DynFile,"/opt/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?DynFile
& stop
EOF

vim /etc/rsyslog.d/10_classic.conf
# Закомментировать: module(load="imuxsock") -> #
systemctl restart rsyslogd
```

### Проверка

На хостах HQ-RTR, BR-RTR, BR-SRV:

```bash
logger -p local2.warning "Syslog test message"
```

На HQ-SRV:

```bash
ls -l /opt
```

### Logrotate

```bash
apt-get install logrotate -y

cat << EOF > /etc/logrotate.d/rsyslog
/opt/**/*.log
{
    weekly
    missingok
    notifempty
    compress
    minsize 10M
}
EOF

logrotate -d /etc/logrotate.d/rsyslog

export EDITOR=/usr/bin/vim
crontab -e
```

```
0	0	*	*	0	root /usr/sbin/logrotate /etc/logrotate.d/rsyslog
```

---

## Задание 7. Мониторинг (Prometheus + Grafana)

### BR-SRV — node_exporter

```bash
apt-get install prometheus-node_exporter -y
systemctl enable --now prometheus-node_exporter
ss -ltnp | grep 9100

samba-tool dns add br-srv.au-team.irpo au-team.irpo mon CNAME hq-srv.au-team.irpo -U Administrator
samba-tool dns query br-srv.au-team.irpo au-team.irpo mon CNAME -U administrator
```

### HQ-SRV — Prometheus + Grafana

```bash
apt-get install prometheus grafana prometheus-node_exporter -y
vim /etc/prometheus/prometheus.yml
```

```yaml
  - job_name: 'prometheus'
    scrape_interval: 5s
    scrape_timeout: 5s
    static_configs:
      - targets: ['localhost:9090']

  - job_name: hq-srv
    static_configs:
      - targets: ['192.168.1.10:9100']
  - job_name: br-srv
    static_configs:
      - targets: ['192.168.3.10:9100']
```

```bash
systemctl enable --now prometheus-node_exporter
systemctl enable --now prometheus
systemctl enable --now grafana-server
systemctl status prometheus-node_exporter prometheus grafana-server
```

### Настройка в браузере

- `http://hq-srv.au-team.irpo:9090` — Status -> Target (должны быть наши сервера)
- `http://hq-srv.au-team.irpo:3000` — Grafana
  - Логин: `admin`, пароль: `admin` → меняем на `P@ssw0rd`
  - Connections -> Data Source -> `http://hq-srv:9090`
  - Dashboard -> New -> Import -> 1860
- `http://mon.au-team.irpo:3000`

---

## Задание 8. Инвентаризация через Ansible

Выполняется на **BR-SRV**.

```bash
ls -l /mnt/
mount -o loop /dev/sr0 /mnt/ -v
cp /mnt/playbook/get_hostname_address.yml /etc/ansible/

cd /etc/ansible/
mkdir PC-INFO
```

Создаём `get_hostname_address.yml`:

```yaml
---
- name: "Get data from hosts"
  gather_facts: true
  hosts:
    - HQ-SRV
    - HQ-CLI
  tasks:
    - name: "Creating a data file"
      copy:
        dest: /etc/ansible/PC-INFO/{‌{ ansible_hostname }}.yml
        content: |
          Hostname: {‌{ ansible_hostname }}
          IP_Address: {‌{ ansible_default_ipv4.address }}
      delegate_to: localhost
```

```bash
ansible-playbook --syntax-check get_hostname_address.yml
ansible-playbook get_hostname_address.yml

cat PC-INFO/hq-srv.yml
cat PC-INFO/hq-cli.yml
```

---

## Задание 9. Защита SSH (fail2ban)

Выполняется на **HQ-SRV**.

```bash
apt-get install fail2ban python3-module-systemd -y
sed -i 's/before = paths-altlinux.conf/before = paths-altlinux-systemd.conf/' /etc/fail2ban/jail.conf
```

Создаём `/etc/fail2ban/jail.d/sshd.conf`:

```ini
[sshd]
enabled = true
port = 2026
filter = sshd
backend = systemd
maxretry = 3
bantime = 1m
```

```bash
systemctl enable --now fail2ban
systemctl status fail2ban.service
fail2ban-client status sshd
```

Проверка с HQ-CLI:

```bash
ssh -p 2026 sshuser@hq-srv
# 3 раза вводим неверный пароль
```

На HQ-SRV проверяем:

```bash
fail2ban-client status sshd
```

---

## Задание 10. Резервное копирование (CyberBackup)

### HQ-SRV — подготовка

```bash
vim /etc/fstab
# Удаляем последнюю строку /raid

apt-get update && apt-get install kernel-source-6.1 kernel-headers-modules-un-def gcc make kmod-sign -y
update-kernel -y
reboot
uname -r

# Создать пользователя irpoadmin с паролем P@ssw0rd
useradd irpoadmin
passwd irpoadmin

ls -l /lib/modules/6.1.169-un-def-alt1/build
mount /dev/sr1 /mnt
/mnt/cyberbackup_17.4.36200.x86_64 --skip-prereq-check
```

Выбрать компоненты:
- `[*] Management Server`
- `[*] Agent for Linux` — 0
- `[ ] Bootable Media Builder` — a
- `[ ] Agent for CommuniGate Pro` — a
- `[*] Agent for MySQL/MariaDB` — a

```bash
ss -ltnp | grep 9877
control mysqld server
systemctl restart mysqld
ss -ltnp | grep 3306
```

### HQ-CLI — установка Storage Node

```bash
apt-get update && apt-get install kernel-source-6.1 kernel-headers-modules-un-def gcc make kmod-sign -y
update-kernel -y
reboot
uname -r

mount /dev/sr0 /mnt
/mnt/cyberbackup_17.4.36200.x86_64 --skip-prereq-check
```

Выбрать:
- `[ ] Management Server`
- `[*] Agent for Linux` — 0
- `[ ] Bootable Media Builder` — a
- `[*] Storage Node` — a

Указать: IP `192.168.1.10`, логин `root`, пароль `toor`.

### Веб-интерфейс

Открыть `https://hq-srv.au-team.irpo:9877` (или `192.168.1.10:9877`).
- Логин: `root`, пароль: `toor`
- Запустить пробную лицензию

#### Настройки
- Настройки -> Учётные записи -> Организации -> Создать отдел: `irpo`
- Выбрать `irpo` -> Добавить учётную запись: `irpoadmin`

#### План №1 — бэкап /etc
- Планы -> Создать план -> Добавить устройство: `hq-srv`
- Место сохранения: Узел хранений -> Путь: создать папку `backup`
- Имя хранилища: `backup_dir`
- Пользователь: root/toor
- Выбор данных: Файлы/папки -> `/etc`
- Имя плана: `etc_backup`

#### План №2 — бэкап БД
- Место сохранения: `backup_dir`
- Резервное копирование приложения -> MySQL/MariaDB -> Добавить экземпляр: root/toor
- Имя плана: `webdb_backup`
- Добавить устройство: `hq-srv`

#### Запуск
Устройства -> Создать резервную копию (`webdb_backup` и `etc_backup`).

### Решение проблемы с подключением к MariaDB

Если MariaDB не подключается:

```bash
systemctl stop mariadb
mysqld_safe --skip-grant-tables &
mysql -u root

FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'toor';
FLUSH PRIVILEGES;
EXIT;

pkill mysqld
systemctl start mariadb
```

После этого повторно запустить бэкап.

### Очистка диска

```bash
tune2fs -m 1 /dev/sda2
rpm -e --nodeps MonitoringServer-17.4.36200-1.x86_64
rpm -e --nodeps WebConsole-17.4.36200-1.x86_64
rpm -e --nodeps AcronisCentralizedManagementServer-17.4.36200-1.x86_64
rpm -e --nodeps dkms-3.0.3.3-100.noarch
rpm -e --nodeps snapapi26_modules-1.0.9-1.noarch
```
