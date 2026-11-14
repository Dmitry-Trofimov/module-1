# Модуль 1 — Настройка BR-RTR

Настройка маршрутизатора филиала BR-RTR.

В данной работе настраиваю подключение к ISP, связь с BR-FW, GRE-туннель до HQ-RTR, динамическую маршрутизацию OSPF и NAT.

## 1. Первоначальная настройка

```bash
enable
configure terminal
hostname br-rtr
ip domain-name nebula.core

username net_admin
password P@ssw0rd
role admin
exit
```

- `enable` — перехожу в привилегированный режим.
- `configure terminal` — перехожу в режим конфигурации.
- `hostname br-rtr` — задаю имя маршрутизатора.
- `ip domain-name nebula.core` — задаю доменное имя.
- `username net_admin` — создаю пользователя.
- `password P@ssw0rd` — задаю пароль пользователя.
- `role admin` — назначаю права администратора.
- `exit` — завершаю создание пользователя.

## 2. Настройка подключения к ISP ( description по схеме в задании!! )

```bash
interface int0
 description "ISP-BR"
 ip address 172.16.2.2/28
 ip nat outside
exit

port te0
 service-instance te0/int0
 encapsulation untagged
 connect ip interface isp
exit
exit
```

- `interface int0` — выбираю интерфейс в сторону провайдера ISP.
- `description "ISP"` — указываю назначение интерфейса.
- `ip address 172.16.2.2/28` — назначаю IP-адрес для подключения к ISP.
- `ip nat outside` — обозначаю интерфейс как внешний для NAT.
- `port te0` — выбираю физический порт te0.
- `service-instance te0/int0` — создаю сервис на физическом порту.
- `encapsulation untagged` — указываю, что трафик передаётся без VLAN-тега.
- `connect ip interface isp` — связываю сервис с внешним IP-интерфейсом.
- `exit` — завершаю настройку порта.

## 3. Настройка подключения к BR-FW ( description по схеме в задании!! )

```bash
interface int1
 description "FW-NET"
 ip address 172.16.3.1/30
 ip nat inside
exit

port te1
 service-instance te1/int1
 encapsulation untagged
 connect ip interface int1
exit
exit
```

- `interface int1` — выбираю интерфейс в сторону BR-FW.
- `description "BR-FW"` — указываю назначение интерфейса.
- `ip address 172.16.3.1/30` — назначаю IP-адрес для связи с межсетевым экраном.
- `ip nat inside` — обозначаю интерфейс как внутренний для NAT.
- `port te1` — выбираю физический порт te1.
- `service-instance te1/int1` — создаю сервис для интерфейса int1.
- `encapsulation untagged` — указываю передачу трафика без VLAN-тега.
- `connect ip interface int1` — связываю сервис с интерфейсом int1.
- `exit` — завершаю настройку порта.

## 4. Настройка маршрута по умолчанию

```bash
ip route 0.0.0.0/0 172.16.2.1
```

- `ip route` — добавляю статический маршрут.
- `0.0.0.0/0` — указываю маршрут для всех неизвестных сетей.
- `172.16.2.1` — задаю адрес следующего узла в сторону ISP.

## 5. Создание GRE-туннеля

```bash
interface tunnel.0
 description "GRE"
 ip address 10.10.10.2/30
 ip mtu 1400
 ip tunnel 172.16.2.2 172.16.1.2 mode gre
exit
```

- `interface tunnel.0` — создаю виртуальный туннельный интерфейс.
- `description "GRE"` — указываю назначение интерфейса.
- `ip address 10.10.10.2/30` — назначаю IP-адрес туннеля со стороны филиала.
- `ip mtu 1400` — устанавливаю максимальный размер пакета 1400 байт с учётом заголовков GRE.
- `ip tunnel 172.16.2.2 172.16.1.2 mode gre` — создаю GRE-туннель между BR-RTR и HQ-RTR.

## 6. Настройка OSPF

```bash
router ospf 1
 ospf router-id 10.10.10.2
 passive-interface default
 no passive-interface tunnel.0
 network 10.10.10.0/30 area 0
 network 172.16.3.0/30 area 0
exit
```

- `router ospf 1` — запускаю процесс динамической маршрутизации OSPF.
- `ospf router-id 10.10.10.2` — задаю уникальный идентификатор маршрутизатора.
- `passive-interface default` — делаю все интерфейсы пассивными по умолчанию.
- `no passive-interface tunnel.0` — разрешаю обмен OSPF через GRE-туннель.
- `network 10.10.10.0/30 area 0` — добавляю туннельную сеть в OSPF.
- `network 172.16.3.0/30 area 0` — добавляю сеть подключения BR-FW.
- `exit` — завершаю настройку OSPF.

## 7. Настройка аутентификации OSPF

```bash
interface tunnel.0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 P@ssw0rd
exit
```

- `interface tunnel.0` — выбираю GRE-интерфейс.
- `ip ospf authentication message-digest` — включаю проверку подлинности OSPF-пакетов.
- `ip ospf message-digest-key 1 md5 P@ssw0rd` — задаю ключ аутентификации MD5. На HQ-RTR должен использоваться такой же ключ.

## 8. Настройка NAT

```bash
ip nat pool BR-FW 172.16.3.2-172.16.3.2
ip nat source dynamic inside-to-outside pool BR-FW overload interface isp
```

- `ip nat pool BR-FW 172.16.3.2-172.16.3.2` — создаю NAT-пул с адресом BR-FW.
- `ip nat source dynamic` — включаю динамическую трансляцию исходных IP-адресов.
- `inside-to-outside` — задаю направление трансляции из внутренней сети во внешнюю.
- `pool BR-FW` — указываю созданный NAT-пул.
- `overload` — разрешаю нескольким подключениям использовать один адрес с преобразованием портов.
- `interface isp` — указываю внешний интерфейс для выхода к провайдеру.

## 9. Сохранение конфигурации

```bash
write memory
```

- `write memory` — сохраняю настройки маршрутизатора, чтобы они сохранились после перезагрузки.

## 10. Проверка настройки

После завершения настройки проверяю:

- Доступность шлюза ISP `172.16.2.1`.
- Связь между BR-RTR и BR-FW.
- Работу GRE-туннеля до HQ-RTR.
- Установление соседства OSPF.
- Получение маршрутов от HQ-RTR.
- Работу NAT.
- Доступность необходимых сетей.

---

**Результат:** BR-RTR настроен для подключения филиала к ISP, взаимодействия с BR-FW и обмена маршрутами с головным офисом HQ-RTR через GRE и OSPF.
