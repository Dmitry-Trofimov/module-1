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
echo 192.168.2.2/30 > /etc/net/ifaces/ens18/ipv4address
echo default via 192.168.2.1 > /etc/net/ifaces/ens18/ipv4route
echo nameserver 77.88.8.8 > /etc/net/resolv.conf
systemctl restart network
```

> **Расхождение:** схема показывает HQ-SRV как `192.168.2.2/28`, а исходный документ задаёт `/30`. Уточните маску по официальному условию.
