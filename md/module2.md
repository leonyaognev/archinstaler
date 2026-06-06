# Модуль 2 — Службы и сервисы

---

> **Важно!** Стенд Модуля 2, установленный из скрипта, имеет недостаток — он проявляется при выполнении Задания №5.
>
> Необходимо самостоятельно включить SSH-доступ на машинах **HQ-RTR**, **BR-RTR**, **HQ-CLI**.
>
> На каждой машине:
> ```bash
> vim /etc/openssh/sshd_config
> # Добавить в самое начало: Port 2026
> systemctl enable --now sshd
> ```

---

## Задание 1. Samba DC, DNS, домен

### BR-SRV — установка Samba Domain Controller

```bash
echo "nameserver 192.168.1.10" >> /etc/net/ifaces/enp7s1/resolv.conf
systemctl restart network
cat /etc/resolv.conf
ping ya.ru -c 2
```

```bash
apt-get update && apt-get install task-samba-dc -y
rm -f /etc/samba/smb.conf
rm -rf /var/lib/samba /var/cache/samba
mkdir -p /var/lib/samba/sysvol
samba-tool domain provision
# Везде соглашаемся со значениями по умолчанию (Enter)
# Пароль: P@ssw0rd
```

```bash
mv /etc/krb5.conf /etc/krb5.conf.back
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
systemctl enable --now samba
systemctl status samba
samba-tool domain info 127.0.0.1
```

### DNS-записи

```bash
samba-tool dns add br-srv.au-team.irpo au-team.irpo hq-srv A 192.168.1.10 -U Administrator
samba-tool dns add br-srv.au-team.irpo au-team.irpo hq-rtr A 192.168.1.1 -U Administrator
samba-tool dns add br-srv.au-team.irpo au-team.irpo br-rtr A 192.168.3.1 -U Administrator
samba-tool dns add br-srv.au-team.irpo au-team.irpo web.au-team.irpo A 172.16.1.1 -U Administrator
samba-tool dns add br-srv.au-team.irpo au-team.irpo docker.au-team.irpo A 172.16.2.1 -U Administrator
samba-tool dns query br-srv.au-team.irpo au-team.irpo @ ALL -U administrator
```

```bash
sed -i 's/nameserver 192.168.1.10/nameserver 127.0.0.1/' /etc/net/ifaces/enp7s1/resolv.conf
systemctl restart network
cat /etc/resolv.conf
kinit administrator@AU-TEAM.IRPO
```

### Группы и пользователи

```bash
samba-tool group add hq
for i in {1..5}; do samba-tool user add hquser$i P@ssw0rd; done
for i in {1..5}; do samba-tool group addmembers hq hquser$i; done
samba-tool group listmembers hq
```

### HQ-RTR — смена DNS в dnsmasq

```bash
sed -i 's/192.168.1.10/192.168.3.10/' /etc/dnsmasq.conf; systemctl restart dnsmasq
cat /etc/dnsmasq.conf
```

### HQ-CLI — ввод в домен

Работаем в **графической оболочке** HQ-CLI (под user):

```
acc
Пользователи (Аутентификация) -> Галочка (Домен Active Directory) -> Применить
Пароль: P@ssw0rd
# Если всё верно — появится окно "Добро пожаловать в домен AU-TEAM.IRPO"
Меню -> Выйти -> Перезагрузить

# Вход: hquser1 / P@ssw0rd
```

Работаем в **консольной оболочке** HQ-CLI (под root):

```bash
control libnss-role
# Должна выдать "enabled"

roleadd hq wheel
echo "WHEEL_USERS ALL=(ALL:ALL) /bin/cat, /bin/grep, /usr/bin/id" >> /etc/sudoers
tail /etc/sudoers
```

Перезагружаемся, заходим как `hquser1`, открываем терминал:

```bash
sudo id        # вводим пароль
sudo cat /etc/resolv.conf   # вводим пароль
sudo grep      # вводим пароль
```

---

## Задание 2-3. RAID + NFS

### HQ-SRV — RAID

```bash
lsblk

parted /dev/sdb
	mklabel msdos
	mkpart primary 1MiB 100%
	set 1 raid on
	print
	select /dev/sdc
	mklabel msdos
	mkpart primary 1MiB 100%
	set 1 raid on
	print
	quit

mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb1 /dev/sdc1
mdadm --detail --scan >> /etc/mdadm.conf
mdadm --detail --scan
mkfs.ext4 /dev/md0
mkdir /raid
cp /etc/fstab /etc/fstab.back
echo "/dev/md0 /raid ext4 defaults 0 0" >> /etc/fstab
mount -av
df -T
```

### HQ-SRV — NFS

```bash
apt-get update && apt-get install nfs-server nfs-utils -y
mkdir /raid/nfs
chmod 777 /raid/nfs
cp /etc/exports /etc/exports.back
echo "/raid/nfs 192.168.2.0/27(rw,no_subtree_check,no_root_squash)" >> /etc/exports
systemctl enable --now nfs-server
```

### HQ-CLI — монтирование NFS

Работаем в **консольной оболочке** HQ-CLI (под root):

```bash
mkdir /mnt/nfs
chmod -R 777 /mnt/nfs
showmount -e hq-srv
cp /etc/fstab /etc/fstab.back
echo "192.168.1.10:/raid/nfs /mnt/nfs nfs rw,soft,_netdev 0 0" >> /etc/fstab
mount -av
df -T

# Создать файл на HQ-CLI
touch /mnt/nfs/test_file
ls -l /mnt/nfs/test_file
```

На HQ-SRV проверяем:

```bash
ls -l /raid/nfs
```

---

## Задание 4. NTP (chrony)

### ISP — chrony-сервер

```bash
control chrony server
sed -i 's/pool pool.ntp.org iburst/pool pool.ntp.org iburst prefer minstratum 4/' /etc/chrony.conf
grep pool /etc/chrony.conf
sed -i 's/\#local stratum 10/local stratum 5/' /etc/chrony.conf
grep "local stratum" /etc/chrony.conf
systemctl restart chronyd
```

### HQ-SRV, BR-RTR, BR-SRV — клиенты

```bash
sed -i 's/pool pool.ntp.org iburst/server 172.16.1.1 iburst/' /etc/chrony.conf && systemctl restart chronyd

# Через 10 секунд проверяем:
chronyc sources
```

### HQ-CLI

```bash
echo "server 172.16.1.1 iburst" >> /etc/chrony.conf && systemctl restart chronyd

# Через 10 секунд проверяем:
chronyc sources
```

Ожидаемый вывод:

```
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* 172.16.1.1                    5   6    17     1  +1438ns[  +43us] +/-   38ms
```

---

## Задание 5. Ansible

Выполняется на **BR-SRV**.

```bash
apt-get install ansible sshpass -y
cp -r /etc/ansible/ansible.cfg /etc/ansible/ansible.cfg.back
rm -rf /etc/ansible/ansible.cfg
vim /etc/ansible/ansible.cfg
```

```ini
[defaults]
host_key_checking = False
interpreter_python=/usr/bin/python3
inventory       = /etc/ansible/hosts
```

```bash
vim /etc/ansible/hosts
```

```ini
HQ-SRV ansible_user=user ansible_password=resu ansible_port=2026
HQ-RTR ansible_user=net_admin ansible_password=P@ssw0rd ansible_port=2026
BR-RTR ansible_user=net_admin ansible_password=P@ssw0rd ansible_port=2026
HQ-CLI ansible_user=user ansible_password=resu ansible_port=2026
```

Проверка:

```bash
ansible all -m ping
# Все должны ответить "pong"
```

---

## Задание 6. Docker (сайт на BR-SRV)

```bash
apt-get install docker-engine docker-compose-v2 -y
systemctl enable --now docker.service

mount -o loop /dev/sr0 /mnt/ -v
ls -l /mnt/docker/
cat /mnt/docker/readme.txt

docker load < /mnt/docker/site_latest.tar
docker load < /mnt/docker/mariadb_latest.tar
docker image ls
```

Создаём `docker-compose.yml`:

```bash
cat << EOF > docker-compose.yml
services:
  database:
    container_name: db
    image: mariadb:latest
    restart: always
    ports:
      - "3306:3306"
    environment:
      MARIADB_DATABASE: testdb
      MARIADB_USER: testc
      MARIADB_PASSWORD: P@ssw0rd
      MARIADB_ROOT_PASSWORD: P@ssw0rd
    volumes:
      - db_data:/var/lib/mysql

  app:
    container_name: tespapp
    image: site:latest
    restart: always
    ports:
      - "8080:8000"
    environment:
      DB_HOST: database
      DB_PORT: 3306
      DB_NAME: testdb
      DB_USER: testc
      DB_PASS: P@ssw0rd
      DB_TYPE: maria
    depends_on:
      - database
volumes:
  db_data:
EOF
```

```bash
docker compose config
docker compose up -d
docker ps
ss -ltnp4 | grep 8080
```

Заходим на HQ-CLI по адресу `192.168.3.10:8080`.

> **Пометка:** По заданию мы должны попадать на сайт через домен `docker.au-team.irpo`, но пока не настроен реверс-прокси на ISP (nginx). Это будет в Задании №9.

Проверка пересоздания:

```bash
docker rm -f $(docker ps -qa)
docker compose up -d
```

---

## Задание 7. LAMP-сервер (HQ-SRV)

```bash
mount -o loop /dev/sr0 /mnt/ -v
apt-get install lamp-server -y
cp /mnt/web/index.php /var/www/html
cp /mnt/web/logo.png /var/www/html
systemctl enable --now mariadb
```

Настройка БД:

```bash
mariadb -e "CREATE DATABASE webdb;"
mariadb -e "
CREATE USER 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost';
"
mariadb webdb < /mnt/web/dump.sql
mariadb -e "USE webdb; SHOW TABLES;"
```

Редактируем `/var/www/html/index.php` — указываем параметры подключения:

```php
$servername = "localhost";
$username = "web";
$password = "P@ssw0rd";
$dbname = "webdb";
```

```bash
systemctl enable --now httpd2.service
```

Заходим на HQ-CLI по адресу `192.168.1.10`.

> **Пометка:** По заданию мы должны попадать на сайт через домен `web.au-team.irpo` с аутентификацией, но это будет настроено позже (Задание №9).

---

## Задание 8. DNAT (проброс портов)

### HQ-RTR

```bash
nft add chain nat prerouting { type nat hook prerouting priority dstnat \; }
nft add rule nat prerouting iif "enp7s1" tcp dport 2026 dnat to 192.168.1.10
nft add rule nat prerouting iif "enp7s1" tcp dport 8080 dnat to 192.168.1.10:80
nft list ruleset
nft list ruleset > /etc/nftables/nftables.nft
systemctl restart nftables
nft list ruleset
```

### BR-RTR

```bash
nft add chain nat prerouting { type nat hook prerouting priority dstnat \; }
nft add rule nat prerouting iif "enp7s1" tcp dport { 8080, 2026 } dnat to 192.168.3.10
nft list ruleset
nft list ruleset > /etc/nftables/nftables.nft
systemctl restart nftables
nft list ruleset
```

---

## Задание 9-10. Nginx reverse-proxy + HTTP auth

Выполняется на **ISP**.

```bash
apt-get update && apt-get install nginx -y
vim /etc/nginx/sites-available.d/r-proxy.conf
```

```nginx
server {
    listen 80;
    server_name web.au-team.irpo;
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
    listen 80;
    server_name docker.au-team.irpo;
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
ln -s /etc/nginx/sites-available.d/r-proxy.conf /etc/nginx/sites-enabled.d/
nginx -t
systemctl enable --now nginx
systemctl status nginx
```

### HTTP-аутентификация

```bash
apt-get install apache2-htpasswd -y
htpasswd -c /etc/nginx/.htpasswd WEBc
# Вводим пароль P@ssw0rd (2 раза)
cat /etc/nginx/.htpasswd
nginx -t
systemctl restart nginx
```

Проверка на **HQ-CLI**:
- `http://web.au-team.irpo/` — логин `WEBc`, пароль `P@ssw0rd`
- `http://docker.au-team.irpo/`

---

## Задание 11. Установка Яндекс Браузера

Выполняется на **HQ-CLI**.

```bash
apt-get update && apt-get install yandex-browser-stable -y
```

> Установку браузера отметьте в отчёте.
