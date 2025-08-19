# Лабораторная работа №12 "Основные протоколы сети Интернет"

## Задание

1. Настройте NAT(PAT) на R14 и R15. Трансляция должна осуществляться в адрес автономной системы AS1001.
2. Настройте NAT(PAT) на R18. Трансляция должна осуществляться в пул из 5 адресов автономной системы AS2042.
3. Настройте статический NAT для R20.
4. Настройте NAT так, чтобы R19 был доступен с любого узла для удаленного управления.
5. Настройте статический NAT(PAT) для офиса Чокурдах.
6. Настройте для IPv4 DHCP сервер в офисе Москва на маршрутизаторах R12 и R13. VPC1 и VPC7 должны получать сетевые настройки по DHCP.
7. Настройте NTP сервер на R12 и R13. Все устройства в офисе Москва должны синхронизировать время с R12 и R13.


Схема сети представлена на рисунке ниже:

![Topology-with-IP2.jpg](./img/Topology-with-IP2.jpg)

Добавлены "белые" адреса для AS Москвы: 77.77.77.0/24 и Санкт-Петербурга: 78.78.78.0/24.

## Решение

### Настройте NAT(PAT) на R14 и R15. Трансляция должна осуществляться в адрес автономной системы AS1001

#### Настройка

Настроим NAT(PAT) в московском отделении.

R14:
```
interface Ethernet0/0
 ip nat inside
 ip virtual-reassembly in
exit 

interface Ethernet0/1
 ip nat inside
 ip virtual-reassembly in
exit 

interface Ethernet0/2
 ip nat outside
 ip virtual-reassembly in
exit 

interface Ethernet0/3
 ip nat inside
 ip virtual-reassembly in
 ip ospf 1 area 101
exit 

interface Ethernet1/0
 ip nat outside
 ip virtual-reassembly in
exit 

ip nat pool MSK_NAT 77.77.77.14 77.77.77.14 netmask 255.255.255.0
ip nat inside source list MSK_CLIENT pool MSK_NAT overload

ip access-list extended MSK_CLIENT
 permit ip 10.77.10.0 0.0.0.255 any
 permit ip 10.77.20.0 0.0.0.255 any
end

```
R15:
```
interface Ethernet0/0
 ip nat inside
 ip virtual-reassembly in
exit

interface Ethernet0/1
 ip nat inside
 ip virtual-reassembly in
exit

interface Ethernet0/2
 ip nat outside
 ip virtual-reassembly in
exit

interface Ethernet0/3
 ip nat inside
 ip virtual-reassembly in
exit

interface Ethernet1/0
 ip nat outside
 ip virtual-reassembly in
exit

ip nat pool MSK_NAT 77.77.77.15 77.77.77.15 netmask 255.255.255.0
ip nat inside source list MSK_CLIENT pool MSK_NAT overload
exit

ip access-list extended MSK_CLIENT
 permit ip 10.77.10.0 0.0.0.255 any
 permit ip 10.77.20.0 0.0.0.255 any
end

```
#### Проверка

R15:
```
R15#sh ip nat translations
Pro Inside global      Inside local       Outside local      Outside global
--- 77.77.77.15        10.77.0.20         ---                ---
icmp 77.77.77.15:57632 10.77.10.101:57632  78.78.78.18:57632      78.78.78.18:57632
icmp 77.77.77.15:57888 10.77.10.101:57888  78.78.78.18:57888      78.78.78.18:57888
icmp 77.77.77.15:58144 10.77.10.101:58144  78.78.78.18:58144      78.78.78.18:58144
icmp 77.77.77.15:58400 10.77.10.101:58400  78.78.78.18:58400      78.78.78.18:58400
icmp 77.77.77.15:58656 10.77.10.101:58656  78.78.78.18:58656      78.78.78.18:58656
R15#
```
### Настройте NAT(PAT) на R18. Трансляция должна осуществляться в пул из 5 адресов автономной системы AS2042

#### Настройка

R18:
```
interface Ethernet0/0
 ip nat inside
 ip virtual-reassembly in
exit

interface Ethernet0/1
 ip nat inside
 ip virtual-reassembly in
exit

interface Ethernet0/2
 ip nat outside
 ip virtual-reassembly in
exit

interface Ethernet0/3
 ip nat outside
 ip virtual-reassembly in
exit

ip nat pool SPB_NAT 78.78.78.1 78.78.78.5 netmask 255.255.255.0
ip nat inside source list NAT_CLIENT pool SPB_NAT overload

ip access-list extended NAT_CLIENT
 permit ip 10.78.10.0 0.0.0.255 any
 permit ip 10.78.20.0 0.0.0.255 any
end

```

#### Проверка

```
R18#sh ip nat translations
Pro Inside global      Inside local       Outside local      Outside global
icmp 78.78.78.1:47891 10.78.10.8:47891  77.77.77.1:47891      77.77.77.1:47891
icmp 78.78.78.1:48147 10.78.10.8:48147  77.77.77.1:48147      77.77.77.1:48147
icmp 78.78.78.1:48403 10.78.10.8:48403  77.77.77.1:48403      77.77.77.1:48403
icmp 78.78.78.1:48659 10.78.10.8:48659  77.77.77.1:48659      77.77.77.1:48659
icmp 78.78.78.1:48915 10.78.10.8:48915  77.77.77.1:48915      77.77.77.1:48915
R18#
```

### Настройте статический NAT для R20

#### Настройка


R15:
```
ip nat inside source static 10.77.0.20 77.77.77.15
```
#### Проверка

```
R20#ping 78.78.78.18 source 10.77.0.20
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 78.78.78.18, timeout is 2 seconds:
Packet sent with a source address of 10.77.0.20
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/3/3 ms
R20#
```

### Настройте NAT так, чтобы R19 был доступен с любого узла для удаленного управления

#### Настройка

Поднимем доступ к R19 по SSH:
- генерируем пару ключей (закрытый и открытый);
- заведём пользователя admin/admin;
- добавляю логин по SSH:
```
crypto key generate rsa modulus 2048

ip domain name local
exit

username admin privilege 15 password 0 admin
exit

line vty 0 4
 login local
 transport input ssh
end
```


На R14 переопределяю порт 22 --> 3456:

```
interface Loopback1
 ip address 77.77.77.19 255.255.255.255
exit

ip nat inside source static tcp 10.77.0.19 22 77.77.77.19 3456 extendable
end
```
#### Проверка
Залогинюсь по SSH с R18 на R19:
```
R18#ssh -p 3456 -1 admin 77.77.77.19
Password:
R19#
R19#exit
[Connection to 77.77.77.19 closed by foreign host]
R18#
```

### Настройте статический NAT(PAT) для офиса Чокурдах

#### Настройка
Настрою route-map, с помощью его определяем с какого IP-source и через
какой интерфейс пойдёт трафик для принятия решения о выборе outside-интерфейса для NAT-трансляции.

R28:
```
interface Ethernet0/0
 ip nat outside
 ip virtual-reassembly in
exit

interface Ethernet0/1
 ip nat outside
 ip virtual-reassembly in
exit

interface Ethernet0/2.10
 ip nat inside
 ip virtual-reassembly in
exit

interface Ethernet0/2.20
 ip nat inside
 ip virtual-reassembly in
exit

ip nat inside source route-map ETH00 interface Ethernet0/0 overload
ip nat inside source route-map ETH01 interface Ethernet0/1 overload
exit

ip access-list standard USERS
 permit 10.14.10.0 0.0.0.255
 permit 10.14.20.0 0.0.0.255
exit

route-map ETH01 permit 10
 match ip address USERS
 match interface Ethernet0/1
exit

route-map ETH00 permit 10
 match ip address USERS
 match interface Ethernet0/0
end 
 
```
#### Проверка
R28:
```
R28#show ip nat tra
Pro Inside global      Inside local       Outside local      Outside global
icmp 152.152.152.10:28918  10.14.10.130:28918  77.77.77.1:28918   77.77.77.1:28918
icmp 152.152.152.10:29174  10.14.10.130:29174  77.77.77.1:29174   77.77.77.1:29174
icmp 152.152.152.10:29430  10.14.10.130:29430  77.77.77.1:29430   77.77.77.1:29430
icmp 152.152.152.10:29686  10.14.10.130:29686  77.77.77.1:29686   77.77.77.1:29686
icmp 152.152.152.10:29942  10.14.10.130:29942  77.77.77.1:29942   77.77.77.1:29942
icmp 152.152.152.14:30966  10.14.20.131:30966  77.77.77.1:30966   77.77.77.1:30966
icmp 152.152.152.14:31222  10.14.20.131:31222  77.77.77.1:31222   77.77.77.1:31222
icmp 152.152.152.14:31478  10.14.20.131:31478  77.77.77.1:31478   77.77.77.1:31478
icmp 152.152.152.14:31734  10.14.20.131:31734  77.77.77.1:31734   77.77.77.1:31734
icmp 152.152.152.14:31990  10.14.20.131:31990  77.77.77.1:31990   77.77.77.1:31990
R28#
```

### Настройте для IPv4 DHCP сервер в офисе Москва на маршрутизаторах R12 и R13. VPC1 и VPC7 должны получать сетевые настройки по DHCP

#### Настройка

R12:
```
ip dhcp excluded-address 10.77.10.128 10.77.10.255
ip dhcp excluded-address 10.77.20.128 10.77.20.255
exit

ip dhcp pool GROUP10
 network 10.77.10.0 255.255.255.0
 default-router 10.77.10.254
 dns-server 8.8.8.8 8.8.4.4
exit

ip dhcp pool GROUP20
 network 10.77.20.0 255.255.255.0
 default-router 10.77.20.254
 dns-server 8.8.8.8 8.8.4.4
end
```
R13:
```
ip dhcp excluded-address 10.77.10.0 10.77.10.127
ip dhcp excluded-address 10.77.20.0 10.17.20.127
exit

ip dhcp pool GROUP10
 network 10.77.10.0 255.255.255.0
 default-router 10.77.10.254
 dns-server 8.8.8.8 8.8.4.4
exit

ip dhcp pool GROUP20
 network 10.77.20.0 255.255.255.0
 default-router 10.77.20.254
 dns-server 8.8.8.8 8.8.4.4
end
```
DHCP-relay на коммутаторах:

SW4 и SW5:
```
interface Vlan10
 ip helper-address 10.77.0.13
 ip helper-address 10.77.0.12
exit

interface Vlan20
 ip helper-address 10.77.0.13
 ip helper-address 10.77.0.12
end 
```

#### Проверка
Получаем IP-шники на VPC:

VPC1
```
VPC1> ip dhcp
DDORA IP 10.77.10.1/24 GW 10.77.10.254

VPC1>
```
VPC7
```
VPC7> ip dhcp
DDORA IP 10.77.20.1/24 GW 10.77.20.254

VPC7>
```

DHCP работает.

### Настройте NTP сервер на R12 и R13. Все устройства в офисе Москва должны синхронизировать время с R12 и R13

#### Настройка

На R12 и R13:
```
ntp master 2
ntp update-calendar
end
```


Настроим NTP-клиента на R14:
```
ntp server 10.77.0.12 prefer
ntp server 10.77.0.13
```
#### Проверка
R14:
```
R14#sh ntp status
Clock is synchronized, stratum 3, reference is 10.77.0.12
nominal freq is 250.0000 Hz, actual freq is 250.0000 Hz, precision is 2**10
ntp uptime is 8300 (1/100 of seconds), resolution is 4000
reference time is EC4C78CD.224DD350 (15:50:37.134 UTC Sun Aug 17 2025)
clock offset is 0.5000 msec, root delay is 1.00 msec
root dispersion is 194.78 msec, peer dispersion is 189.47 msec
loopfilter state is 'CTRL' (Normal Controlled Loop), drift is 0.000000000 s/s
system poll interval is 128, last update was 72 sec ago.
R12#
```

R12:
```
R12#sh ntp status
Clock is synchronized, stratum 2, reference is 127.127.1.1
nominal freq is 250.0000 Hz, actual freq is 250.0000 Hz, precision is 2**10
ntp uptime is 497600 (1/100 of seconds), resolution is 4000
reference time is EC4C8B96.C4189590 (17:10:46.766 UTC Sun Aug 17 2025)
clock offset is 0.0000 msec, root delay is 0.00 msec
root dispersion is 2.40 msec, peer dispersion is 1.20 msec
loopfilter state is 'CTRL' (Normal Controlled Loop), drift is 0.000000000 s/s
system poll interval is 16, last update was 16 sec ago.
R12#sh ntp associations

  address         ref clock       st   when   poll reach  delay  offset   disp
*~127.127.1.1     .LOCL.           1      9     16   377  0.000   0.000  1.204
 * sys.peer, # selected, + candidate, - outlyer, x falseticker, ~ configured
```

Конфигурационныe файлы можно найти по [ссылке](./cfg).
