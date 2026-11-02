# ISP и HQ-SRV — команды из исходного документа

## ISP (ALT Linux / сетевые интерфейсы)

```bash
hostnamectl set-hostname ISP
exec bash

mkdir /etc/net/ifaces/ens19
cp /etc/net/ifaces/ens18/options /etc/net/ifaces/ens19
echo 172.16.1.1/28 > /etc/net/ifaces/ens19/ipv4address
vim /etc/net/ifaces/ens19/options
# Проверьте, что конфигурация статическая, затем сохраните файл.

systemctl restart network

mkdir /etc/net/ifaces/ens20
cp /etc/net/ifaces/ens19/options /etc/net/ifaces/ens20
echo 172.16.2.1/28 > /etc/net/ifaces/ens20/ipv4address
systemctl restart network

apt-get update
apt-get -y install iptables
vim /etc/net/sysctl.conf
# Включите IPv4 forwarding в соответствии с требованиями вашей версии ОС.

iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
systemctl restart network
```

> В исходном файле строка записана как `ip tables` и `systemctl enable –now` с длинным тире. Здесь показан типовой вариант написания; проверьте команды на используемой версии ОС.

## HQ-SRV (ALT Linux)

```bash
mkdir /etc/net/ifaces/ens18
cp /etc/net/ifaces/ens1/options /etc/net/ifaces/ens18
echo 192.168.2.2/30 > /etc/net/ifaces/ens18/ipv4address
echo default via 192.168.2.1 > /etc/net/ifaces/ens18/ipv4route
echo nameserver 77.88.8.8 > /etc/net/resolv.conf
systemctl restart network
```
НАСТРОЙКА DNS 
Шаг 1. Установка пакетов BIND
Выполните установку необходимых пакетов на сервере **HQ-SRV**:
```bash
apt-get update && apt-get install bind bind-utils -y
```
Шаг 2. Настройка основного файла options.conf
vim /var/lib/bind/etc/options.conf
```bash
options {
    version "unknown";
    directory "/etc/bind/zone";
    dump-file "/var/run/named/named_dump.db";
    statistics-file "/var/run/named/named.stats";
    recursing-file "/var/run/named/named.recursing";
    secroots-file "/var/run/named/named.secroots";

    // Disables the use of a PID file
    pid-file none;

    // Слушаем только локальный IP-адрес сервера HQ-SRV
    listen-on { 192.168.2.2; };
    listen-on-v6 { none; };

    // Перенаправление запросов, на которые нет ответа в локальных зонах
    forwarders { 77.88.8.8; };

    // Разрешаем запросы от любых хостов
    allow-query { any; };
};
```
Шаг 3. Настройка файла зон rfc1912.conf
im /var/lib/bind/etc/rfc1912.conf
ВСЕ ЗАПИСИ В КОНЕЦ ФАЙЛА!
```bash
zone "au-team.irpo" {
    type master;
    file "au-team.irpo";
};

zone "2.168.192.in-addr.arpa" {
    type master;
    file "2.168.192.in-addr.arpa";
};

zone "3.168.192.in-addr.arpa" {
    type master;
    file "3.168.192.in-addr.arpa";
};
```
Шаг 4. Создание файлов зон
```bash
cp /var/lib/bind/etc/zone/empty /var/lib/bind/etc/zone/au-team.irpo
cp /var/lib/bind/etc/zone/empty /var/lib/bind/etc/zone/2.168.192.in-addr.arpa
cp /var/lib/bind/etc/zone/empty /var/lib/bind/etc/zone/3.168.192.in-addr.arpa
```
Шаг 5. Настройка прямой зоны au-team.irpo
vim /var/lib/bind/etc/zone/au-team.irpo
```bash
$TTL    1D
@       IN      SOA     au-team.irpo. root.au-team.irpo. (
                        2023102401      ; serial
                        12H             ; refresh
                        1H              ; retry
                        1W              ; expire
                        1H )            ; ncache

        IN      NS      au-team.irpo.
hq-srv  IN      A       192.168.2.2
hq-cli  IN      A       192.168.3.2
hq-rtr  IN      A       192.168.2.1
hq-rtr  IN      A       192.168.3.1
hq-rtr  IN      A       172.16.1.2
isp     IN      A       172.16.1.1
isp     IN      A       172.16.2.1
br-rtr  IN      A       172.16.2.2
br-rtr  IN      A       172.16.3.1
br-fw   IN      A       172.16.3.2
br-fw   IN      A       172.16.4.1
br-srv  IN      A       172.16.4.2
```
Шаг 6. Настройка обратной зоны VLAN 100 (192.168.2.0/28)
vim /var/lib/bind/etc/zone/2.168.192.in-addr.arpa
```bash
$TTL    1D
@       IN      SOA     au-team.irpo. root.au-team.irpo. (
                        2023102401      ; serial
                        12H             ; refresh
                        1H              ; retry
                        1W              ; expire
                        1H )            ; ncache

        IN      NS      au-team.irpo.

1       IN      PTR     hq-rtr.au-team.irpo.
2       IN      PTR     hq-srv.au-team.irpo.
```
Шаг 7. Настройка обратной зоны VLAN 200 (192.168.3.0/28)
vim /var/lib/bind/etc/zone/3.168.192.in-addr.arpa:
```bash
$TTL    1D
@       IN      SOA     au-team.irpo. root.au-team.irpo. (
                        2023102401      ; serial
                        12H             ; refresh
                        1H              ; retry
                        1W              ; expire
                        1H )            ; ncache

        IN      NS      au-team.irpo.

1       IN      PTR     hq-rtr.au-team.irpo.
2       IN      PTR     hq-cli.au-team.irpo.
```
Шаг 8. Настройка утилиты rndc
Сгенерируйте ключ безопасности и обрежьте лишние строки:
```bash
rndc-confgen > /var/lib/bind/etc/rndc.key
sed -i '6,$d' /var/lib/bind/etc/rndc.key
```
Шаг 9. Проверка прав и синтаксиса
```bash
root@hq-srv ~]# cat /etc/bind/rndc.key
                                ( # Start of rndc.conf                                          
                                key "rndc-key" {
                                        algorithm hmac-sha256;
                                        secret "SKB1FfUzpWq44SPhErFg5+q5SiSuzp8fk1brZQuNuzs=";
                                };
[root@hq-srv ~]# named-checkconf
[root@hq-srv ~]# 
[root@hq-srv ~]# chown -R root:named /etc/bind/zone/*
[root@hq-srv ~]# ls -l /etc/bind/zone/
total 32
-rw-r----- 1 root named  378 Sep  4 10:14 100.168.192.in-addr.arpa
-rw-r----- 1 root named  212 May 22 11:22 127.in-addr.arpa
-rw-r----- 1 root named  378 Sep  4 10:15 200.168.192.in-addr.arpa
-rw-r----- 1 root named  558 Sep  4 10:13 au-team.irpo
-rw-r----- 1 root named  309 May 22 11:22 empty
-rw-r----- 1 root named  208 May 22 11:22 localdomain
-rw-r----- 1 root named  178 May 22 11:22 localhost
drwxr-x--- 2 root named 4096 May 22 11:22 slave
[root@hq-srv ~]#
[root@hq-srv ~]# named-checkconf -z
zone localhost/IN: loaded serial 2025062300
zone localdomain/IN: loaded serial 2025062300
zone 127.in-addr.arpa/IN: loaded serial 2025062300
zone 0.in-addr.arpa/IN: loaded serial 2025062300
zone 255.in-addr.arpa/IN: loaded serial 2025062300
zone au-team.irpo/IN: loaded serial 2025062300
zone 2.168.192.in-addr.arpa/IN: loaded serial 2025062300
zone 3.168.192.in-addr.arpa/IN: loaded serial 2025062300
[root@hq-srv ~]#
```
Шаг 10. Запуск и проверка работы
Запустите службу и добавьте её в автозагрузку:
```bash
systemctl enable --now bind.service
systemctl status bind.service


если все зеленое то все зашибись!
```

> **Расхождение:** схема показывает HQ-SRV как `192.168.2.2/28`, а исходный документ задаёт `/30`. Уточните маску по официальному условию.
