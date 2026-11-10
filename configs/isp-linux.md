# ISP и HQ-SRV и BR-SRV и HQ-CLI и FW-DT

## ISP (ALT Linux / сетевые интерфейсы)

```bash
hostnamectl set-hostname ISP ; exec bash
exec bash

mkdir /etc/net/ifaces/ens19
cp /etc/net/ifaces/ens18/options /etc/net/ifaces/ens19
echo 172.16.1.1/28 > /etc/net/ifaces/ens19/ipv4address
vim /etc/net/ifaces/ens19/options (измени графу на BOOTROTO=static)
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

## FW-DT (IDECO)
```bash
ЗАХОДИМ ПОД ЛОГИНОМ И ПАРОЛЕМ admin и admin
далее создаем интерфейс!
выбираем мак адрес устройства который смотрит в сторону br-rtr (внимательно проверяй эту тему! а то ничего работать не будет)
на предложение настройки по dhcp ты отказываешься и нажимаешь n 
далее предложат написать айпи адресс пишешь как на схеме 172.16.3.2/30
шлюз 172.16.3.1
больше тебе ничего не адо и графу с вланом оставляешь пустым
ДАЛЕЕ НАЖИМАЕШЬ 19 ЭТО ВЫХОД 
И ТЕБЕ ДАЮТ АДРЕС КОТОРОЙ НУЖНО ВВЕСТИ НА HQ-CLI В БРАУЗЕРЕ
```

## HQ-CLI (ALT Linux)
```bash
В МЕНЮ ПИШЕМ System management center > заходим > пароль toor > нажимаем на Интернет интерфейс > сверху пишем имя hq-cli.au-team.irpo ( напоминай что доменное имя разное! смотри по заданию) > далее жми apply и если ты настроил dhcp на роутере правильно то там появиться полная адресация > ( по заданию могут попросить устновить яндекс браузер! устанавливаем) > заходим в браузер и пишем в адресную строку адресс который дал нам ideco это 172.16.3.1:8443 
пароль он admin и admin
в столбце выбираем (СЕРВИСЫ) > СЕТЕВЫЕ ИНТЕРФЕЙСЫ > нажимаем добавить имя пишем BR-SRV ниже в графе айпи адрес пишим адрес самого br-srv 172.16.4.1/30 НАЖИМАЕМ ДОБАВИТЬ! 
ДАЛЕЕ в столбце выбираем ( правило трафика ) > файрвол > сверху где будет написано forward нажимаем кнопку ДОБАВИТЬ проверяем что везде стоит ЛЮБОЙ и сохраням нажав на кнопку ДОБАВИТЬ! потом сверху так же есть SNAT так же выбираем добавить и добавляем листаем в самый низ и там будет ВКЛЮЧИТЬ ПРАВИЛО включаем и сохраняем нажав на кнопку ДОБАВИТЬ! 
ДАЛЕЕ нужно перейти в панель мониторинга и там разрешить форвард
```

## BR-SRV (ALT Linux)
```bash
hostnamectl set-hostname br-srv.nebula.core ; exec bash
mkdir /etc/net/ifaces/ens18
cp /etc/net/ifaces/ens1/options /etc/net/ifaces/ens18
echo 172.16.4.2/30 > /etc/net/ifaces/ens18/ipv4address
echo default via 192.168.4.1 > /etc/net/ifaces/ens18/ipv4route
vim /etc/net/ifaces/ens18/options (измени графу на BOOTROTO=static)
systemctl restart network
```
## HQ-SRV (ALT Linux)

```bash
hostnamectl set-hostname hq-srv.nebula.core ; exec bash
mkdir /etc/net/ifaces/ens18
cp /etc/net/ifaces/ens1/options /etc/net/ifaces/ens18
echo 192.168.2.2/30 > /etc/net/ifaces/ens18/ipv4address
echo default via 192.168.2.1 > /etc/net/ifaces/ens18/ipv4route
echo nameserver 77.88.8.8 > /etc/net/resolv.conf (ВРЕМЕННЫЙ DNS СЕРВЕР)
vim /etc/net/ifaces/ens18/options (измени графу на BOOTROTO=static)
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
    file "nebula.core";
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
cp /var/lib/bind/etc/zone/empty /var/lib/bind/etc/zone/nebula.core
cp /var/lib/bind/etc/zone/empty /var/lib/bind/etc/zone/2.168.192.in-addr.arpa
cp /var/lib/bind/etc/zone/empty /var/lib/bind/etc/zone/3.168.192.in-addr.arpa
```
Шаг 5. Настройка прямой зоны nebula.core
vim /var/lib/bind/etc/zone/nebula.core
```bash
$TTL    1D
@       IN      SOA     nebula.core. root.nebula.core. (
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
@       IN      SOA     nebula.core. root.nebula.core. (
                        2023102401      ; serial
                        12H             ; refresh
                        1H              ; retry
                        1W              ; expire
                        1H )            ; ncache

        IN      NS      nebula.core.

1       IN      PTR     hq-rtr.nebula.core.
2       IN      PTR     hq-srv.nebula.core.
```
Шаг 7. Настройка обратной зоны VLAN 200 (192.168.3.0/28)
vim /var/lib/bind/etc/zone/3.168.192.in-addr.arpa:
```bash
$TTL    1D
@       IN      SOA     nebula.core. root.nebula.core. (
                        2023102401      ; serial
                        12H             ; refresh
                        1H              ; retry
                        1W              ; expire
                        1H )            ; ncache

        IN      NS      nebula.core.

1       IN      PTR     hq-rtr.nebula.core.
2       IN      PTR     hq-cli.nebula.core.
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
