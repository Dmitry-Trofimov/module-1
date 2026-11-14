# Модуль 1 — Настройка HQ-RTR

Настройка маршрутизатора головного офиса HQ-RTR.

В данной работе выполняю первоначальную настройку маршрутизатора, настройку VLAN, NAT, GRE, OSPF и DHCP.

## 1. Первоначальная настройка

```bash
enable
configure terminal
hostname hq-rtr
ip domain-name nebula.core
username net_admin
password P@ssw0rd
role admin
exit
```

- `enable` — перехожу в привилегированный режим.
- `configure terminal` — перехожу в режим конфигурации.
- `hostname hq-rtr` — задаю имя маршрутизатора.
- `ip domain-name nebula.core` — задаю доменное имя.
- `username net_admin` — создаю пользователя.
- `password P@ssw0rd` — задаю пароль пользователя.
- `role admin` — назначаю права администратора.
- `exit` — выхожу из режима настройки пользователя.

## 2. Настройка IP-адресов интерфейсов ( description по схеме в задании!! )

```bash
interface int0
 description "ISP-HQ"
 ip address 172.16.1.2/28
 ip nat outside
exit

interface int1
 description "SRV-NET"
 ip address 192.168.2.1/28
exit

interface int2
 description "CLI-NET"
 ip address 192.168.3.1/28
exit

interface vl999
 description "HQ-NET"
 ip address 192.168.99.1/29
exit
```

- `interface int1` — выбираю интерфейс в сторону HQ-SRV.
- `description "srv"` — указываю назначение интерфейса.
- `ip address 192.168.2.1/28` — назначаю адрес шлюза серверной сети.
- `interface int2` — выбираю интерфейс в сторону HQ-CLI.
- `description "cli"` — указываю назначение интерфейса.
- `ip address 192.168.3.1/28` — назначаю адрес шлюза клиентской сети.
- `interface vl999` — выбираю интерфейс VLAN 999.
- `description "switch"` — добавляю описание интерфейса.
- `ip address 192.168.99.1/29` — назначаю IP-адрес интерфейсу.
- `interface int0` — выбираю интерфейс в сторону ISP.
- `description "ISP"` — указываю направление подключения.
- `ip address 172.16.1.2/28` — назначаю внешний IP-адрес.
- `ip nat outside` — обозначаю интерфейс как внешний для NAT.

## 3. Настройка физических портов и VLAN

### te0

```bash
port te0
 service-instance te0/int0
 encapsulation untagged
 connect ip interface int0
exit
exit
```

- `port te0` — выбираю физический порт.
- `service-instance te0/int0` — создаю сервис для интерфейса int0.
- `encapsulation untagged` — принимаю трафик без VLAN-тега.
- `connect ip interface int0` — связываю сервис с IP-интерфейсом int0.
- `exit` — завершаю настройку порта.

### Порт te1

```bash
port te1
 service-instance te1/int1
 encapsulation dot1q 100
 rewrite pop 1
 connect ip interface int1
exit

service-instance te1/int2
 encapsulation dot1q 200
 rewrite pop 1
 connect ip interface int2
exit

service-instance te1/vl999
 encapsulation dot1q 999
 rewrite pop 1
 connect ip interface vl999
exit
exit
```

- `port te1` — выбираю физический порт для подключения VLAN.
- `service-instance te1/int1` — создаю сервис для VLAN 100.
- `encapsulation dot1q 100` — указываю VLAN 100.
- `rewrite pop 1` — удаляю один VLAN-тег.
- `connect ip interface int1` — направляю трафик в int1.
- `service-instance te1/int2` — создаю сервис для VLAN 200.
- `encapsulation dot1q 200` — указываю VLAN 200.
- `rewrite pop 1` — удаляю один VLAN-тег.
- `connect ip interface int2` — направляю трафик в int2.
- `service-instance te1/vl999` — создаю сервис для VLAN 999.
- `encapsulation dot1q 999` — указываю VLAN 999.
- `rewrite pop 1` — удаляю один VLAN-тег.
- `connect ip interface vl999` — направляю трафик в vl999.

## 4. Настройка NAT на интерфейсах

```bash
interface int1
 ip nat inside
exit

interface int2
 ip nat inside
exit

interface vl999
 ip nat inside
exit
```

- `interface int1` — выбираю серверный интерфейс.
- `ip nat inside` — обозначаю его как внутренний для NAT.
- `interface int2` — выбираю клиентский интерфейс.
- `ip nat inside` — обозначаю его как внутренний для NAT.
- `interface vl999` — выбираю интерфейс VLAN 999.
- `ip nat inside` — обозначаю его как внутренний для NAT.

## 5. Маршрут по умолчанию

```bash
ip route 0.0.0.0/0 172.16.1.1
```

- `ip route` — добавляю статический маршрут.
- `0.0.0.0/0` — указываю маршрут для всех неизвестных сетей.
- `172.16.1.1` — задаю следующий узел в сторону ISP.

## 6. Создание GRE-туннеля

```bash
interface tunnel.0
 description "GRE"
 ip address 10.10.10.1/30
 ip mtu 1400
 ip tunnel 172.16.1.2 172.16.2.2 mode gre
exit
```

- `interface tunnel.0` — создаю виртуальный туннельный интерфейс.
- `description "GRE"` — указываю назначение интерфейса.
- `ip address 10.10.10.1/30` — назначаю IP-адрес туннеля.
- `ip mtu 1400` — устанавливаю максимальный размер пакета 1400 байт с учётом дополнительных заголовков GRE.
- `ip tunnel 172.16.1.2 172.16.2.2 mode gre` — создаю GRE-туннель между HQ-RTR и BR-RTR.

## 7. Настройка OSPF

```bash
router ospf 1
 ospf router-id 10.10.10.1
 passive-interface default
 no passive-interface tunnel.0
 network 10.10.10.0/28 area 0
 network 192.168.2.0/28 area 0
 network 192.168.3.0/28 area 0
 network 192.168.99.0/29 area 0
exit
```

- `router ospf 1` — запускаю процесс динамической маршрутизации OSPF.
- `ospf router-id 10.10.10.1` — задаю идентификатор маршрутизатора.
- `passive-interface default` — делаю все интерфейсы пассивными по умолчанию.
- `no passive-interface tunnel.0` — разрешаю обмен OSPF через GRE-туннель.
- `network 10.10.10.0/28 area 0` — добавляю туннельную сеть в OSPF.
- `network 192.168.2.0/28 area 0` — добавляю серверную сеть.
- `network 192.168.3.0/28 area 0` — добавляю клиентскую сеть.
- `network 192.168.99.0/29 area 0` — добавляю VLAN 999.
- `exit` — завершаю настройку OSPF.

## 8. Аутентификация OSPF

```bash
interface tunnel.0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 P@ssw0rd
exit
```

- `interface tunnel.0` — выбираю GRE-интерфейс.
- `ip ospf authentication message-digest` — включаю аутентификацию OSPF.
- `ip ospf message-digest-key 1 md5 P@ssw0rd` — задаю ключ аутентификации MD5.

## 9. Создание NAT-пулов

```bash
ip nat pool VLAN100 192.168.2.2-192.168.2.14
ip nat pool VLAN200 192.168.3.2-192.168.3.14
ip nat pool VLAN999 192.168.99.2-192.168.99.4
```

- `ip nat pool VLAN100` — создаю пул адресов для серверной сети.
- `ip nat pool VLAN200` — создаю пул адресов для клиентской сети.
- `ip nat pool VLAN999` — создаю пул адресов для VLAN 999.

## 10. Настройка динамического NAT

```bash
ip nat source dynamic inside-to-outside pool VLAN100 overload interface int0
ip nat source dynamic inside-to-outside pool VLAN200 overload interface int0
ip nat source dynamic inside-to-outside pool VLAN999 overload interface int0
```

- `ip nat source dynamic` — включаю динамическую трансляцию исходных IP-адресов.
- `inside-to-outside` — задаю направление трансляции из внутренней сети во внешнюю.
- `pool VLAN100` — использую пул серверной сети.
- `pool VLAN200` — использую пул клиентской сети.
- `pool VLAN999` — использую пул VLAN 999.
- `overload` — разрешаю нескольким устройствам использовать один внешний адрес с преобразованием портов.
- `interface int0` — указываю внешний интерфейс в сторону ISP.

## 11. Создание DHCP-пула

```bash
ip pool VLAN200
 range 192.168.3.2-192.168.3.14
exit
```

- `ip pool VLAN200` — создаю пул DHCP для клиентской сети.
- `range 192.168.3.2-192.168.3.14` — задаю диапазон IP-адресов для клиентов.
- `exit` — завершаю настройку пула.

## 12. Настройка DHCP-сервера

```bash
dhcp-server 1
 pool VLAN200 1
 mask 28
 gateway 192.168.3.1
 dns 192.168.2.2
 domain-name nebula.core
exit
```

- `dhcp-server 1` — создаю DHCP-сервер.
- `pool VLAN200 1` — подключаю созданный пул адресов.
- `mask 28` — задаю маску сети `/28`.
- `gateway 192.168.3.1` — указываю шлюз для клиентов.
- `dns 192.168.2.2` — указываю DNS-сервер HQ-SRV.
- `domain-name nebula.core` — задаю доменное имя для клиентов.
- `exit` — завершаю настройку DHCP.

## 13. Привязка DHCP к интерфейсу

```bash
interface int2
 dhcp-server 1
exit
```

- `interface int2` — выбираю интерфейс в сторону HQ-CLI.
- `dhcp-server 1` — привязываю DHCP-сервер к клиентской сети.
- `exit` — завершаю настройку интерфейса.

## 14. Часовой пояс и сохранение конфигурации

```bash
ntp timezone utc+3
write memory
```

- `ntp timezone utc+3` — задаю часовой пояс UTC+3.
- `write memory` — сохраняю конфигурацию маршрутизатора.

## 15. Проверка настройки

После завершения настройки проверяю:

- Работу сетевых интерфейсов.
- Доступность шлюза ISP.
- Работу GRE-туннеля.
- Установление соседства OSPF.
- Наличие маршрутов.
- Выдачу IP-адресов через DHCP.
- Работу DNS-сервера.
- Доступ в интернет из локальных сетей.

---

**Результат:** HQ-RTR настроен для маршрутизации между локальными сетями, подключения к ISP и взаимодействия с BR-RTR через GRE и OSPF.
