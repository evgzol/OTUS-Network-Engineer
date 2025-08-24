# Лабораторная работа №13 "VPN. GRE. DmVPN"

## Задание

1. Настройте GRE между офисами Москва и С.-Петербург
2. Настройте DMVMN между Москва и Чокурдах, Лабытнанги
3. Все узлы в офисах в лабораторной работе должны иметь IP связность


Схема сети представлена на рисунке ниже:

![Topology-with-IP2.jpg](./img/Topology-with-IP3.jpg)


## Решение

Туннели буду поднимать на "Loopback-ах", для данного функционала добавлю Loopback-интерфейсы на маршрутизаторах Москвы (R14, R15) согласно следующей таблицы:

NE|Loopback int|Loopback IP|Технология
---|---|---|---
R14|Loopback100|77.77.77.114/32|GRE
R14|Loopback200|77.77.77.214/32|DmVPN
R15|Loopback100|77.77.77.115/32|GRE
R15|Loopback200|77.77.77.215/32|DmVPN
---|---|---|---

R14:
```
interface Loopback100
 description *** for GRE ***
 ip address 77.77.77.114 255.255.255.255
exit

interface Loopback200
 description *** for DmVPN ***
 ip address 77.77.77.214 255.255.255.255
end
```

R15:
```
interface Loopback100
 description *** for GRE ***
 ip address 77.77.77.115 255.255.255.255
exit

interface Loopback200
 description *** for DmVPN ***
 ip address 77.77.77.215 255.255.255.255
end
```

Кроме этого, назначу дополнительную IP-адресацию для туннелей, см. таблицу:
Network|Технология
---|---
10.100.14.0/24|GRE
10.100.15.0/24|GRE
10.200.14.0/24|DmVPN
10.200.15.0/24|DmVPN
---|---|---

### Настройте GRE между офисами Москва и С.-Петербург

#### Настройка
Настрою GRE-туннель между офисами Москвы и Санкт-Петербурга.
Для обеспечения отказоустойсивости настрою туннели R14 <---> R18 и R15 <---> R18.

*Москва*
R14:

```
interface Tunnel0
 ip address 10.100.14.14 255.255.255.0
 tunnel source 77.77.77.114
 tunnel destination 78.78.78.18
end
```
R15:

```
interface Tunnel0
 ip address 10.100.15.15 255.255.255.0
 tunnel source 77.77.77.115
 tunnel destination 78.78.78.18
end
```

*Санкт-Петербург*

R18:

```
interface Tunnel0
 ip address 10.100.14.18 255.255.255.0
 tunnel source 78.78.78.18
 tunnel destination 77.77.77.114
exit

interface Tunnel1
 ip address 10.100.15.18 255.255.255.0
 tunnel source 78.78.78.18
 tunnel destination 77.77.77.115
end
```


#### Проверка


Проверию корректность создания туннелей

R14:
```
R14#show ip int brief
Interface                  IP-Address      OK? Method Status                Protocol
Ethernet0/0                10.77.1.5       YES TFTP   up                    up
Ethernet0/1                10.77.1.9       YES TFTP   up                    up
Ethernet0/2                111.111.111.2   YES TFTP   up                    up
Ethernet0/3                10.77.1.1       YES TFTP   up                    up
Ethernet1/0                77.77.77.1      YES TFTP   up                    up
Ethernet1/1                unassigned      YES TFTP   administratively down down
Ethernet1/2                unassigned      YES TFTP   administratively down down
Ethernet1/3                unassigned      YES TFTP   administratively down down
Loopback0                  10.77.0.14      YES TFTP   up                    up
Loopback1                  77.77.77.19     YES manual up                    up
Loopback2                  77.77.77.14     YES manual up                    up
Loopback100                77.77.77.114    YES manual up                    up
Loopback200                77.77.77.214    YES manual up                    up
NVI0                       10.77.1.5       YES unset  up                    up
Tunnel0                    10.100.14.14    YES manual up                    up
```

R15:
```
R15#show ip int brief
Interface                  IP-Address      OK? Method Status                Protocol
Ethernet0/0                10.77.1.17      YES TFTP   up                    up
Ethernet0/1                10.77.1.13      YES TFTP   up                    up
Ethernet0/2                131.131.131.2   YES TFTP   up                    up
Ethernet0/3                10.77.1.21      YES TFTP   up                    up
Ethernet1/0                77.77.77.2      YES TFTP   up                    up
Ethernet1/1                unassigned      YES TFTP   administratively down down
Ethernet1/2                unassigned      YES TFTP   administratively down down
Ethernet1/3                unassigned      YES TFTP   administratively down down
Loopback0                  10.77.0.15      YES TFTP   up                    up
Loopback1                  77.77.77.15     YES manual up                    up
Loopback100                77.77.77.115    YES manual up                    up
Loopback200                77.77.77.215    YES manual up                    up
NVI0                       10.77.1.17      YES unset  up                    up
Tunnel0                    10.100.15.15    YES manual up                    up
```


R18:
```
R18#show ip int brief
Interface                  IP-Address      OK? Method Status                Protocol
Ethernet0/0                10.78.1.5       YES TFTP   up                    up
Ethernet0/1                10.78.1.1       YES TFTP   up                    up
Ethernet0/2                152.152.152.2   YES TFTP   up                    up
Ethernet0/3                152.152.152.6   YES TFTP   up                    up
Loopback0                  10.78.0.18      YES TFTP   up                    up
Loopback1                  78.78.78.18     YES TFTP   up                    up
NVI0                       10.78.1.5       YES unset  up                    up
Tunnel0                    10.100.14.18    YES manual up                    up
Tunnel1                    10.100.15.18    YES manual up                    up
```

Теперь проверим связность между офисами по GRE-туннелям:


R18:
```
R18#ping 10.100.14.14
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.100.14.14, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/2/3 ms
R18#ping 10.100.15.15
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.100.15.15, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/2/4 ms
```

R14:
```
R14#ping 10.100.14.18
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.100.14.18, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/2/4 ms
```

R15:
```
R18#ping 10.100.15.18
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.100.15.18, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 4/4/5 ms
```


Таблица маршрутизации на R18:


```
R18#sh ip ro
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       a - application route
       + - replicated route, % - next hop override

Gateway of last resort is 152.152.152.5 to network 0.0.0.0

...

C        10.100.14.0/24 is directly connected, Tunnel0
L        10.100.14.18/32 is directly connected, Tunnel0
C        10.100.15.0/24 is directly connected, Tunnel1
L        10.100.15.18/32 is directly connected, Tunnel1

...
```



### Настройте DMVMN между Москва и Чокурдах, Лабытнанги

#### Настройка

Распределим роли.
Офис в Москве -- HUB:
R14: 77.77.77.214/32
R15: 77.77.77.215/32

Лабынтаги (R27) -- SPOKE:
10.200.14.27/24
10.200.15.27/24

Чокурдах (R28) -- SPOKE:
10.200.14.28/24
10.200.15.28/24


R14:
```
interface Tunnel1
 ip address 10.200.14.14 255.255.255.0
 no ip redirects
 ip nhrp map multicast dynamic
 ip nhrp network-id 1400
 tunnel source 77.77.77.214
 tunnel mode gre multipoint
 tunnel key 1400
end
```


R15:
```
interface Tunnel1
 ip address 10.200.15.15 255.255.255.0
 no ip redirects
 ip nhrp map multicast dynamic
 ip nhrp network-id 1500
 tunnel source 77.77.77.215
 tunnel mode gre multipoint
 tunnel key 1500
end
```

R27:
```
interface Tunnel0
 ip address 10.200.14.27 255.255.255.0
 no ip redirects
 ip nhrp map 10.200.14.14 77.77.77.214
 ip nhrp map multicast 77.77.77.214
 ip nhrp network-id 1400
 ip nhrp nhs 10.200.14.14
 ip nhrp registration no-unique
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 1400
exit

interface Tunnel1
 ip address 10.200.15.27 255.255.255.0
 no ip redirects
 ip nhrp map 10.200.15.15 77.77.77.215
 ip nhrp map multicast 77.77.77.215
 ip nhrp network-id 1500
 ip nhrp nhs 10.200.15.15
 ip nhrp registration no-unique
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 1500
end
```

R28:
```
interface Tunnel0
 ip address 10.200.14.28 255.255.255.0
 no ip redirects
 ip nhrp map 10.200.14.14 77.77.77.214
 ip nhrp map multicast 77.77.77.214
 ip nhrp network-id 1400
 ip nhrp nhs 10.200.14.14
 ip nhrp registration no-unique
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 1400
exit

interface Tunnel1
 ip address 10.200.15.28 255.255.255.0
 no ip redirects
 ip nhrp map 10.200.15.15 77.77.77.215
 ip nhrp map multicast 77.77.77.215
 ip nhrp network-id 1500
 ip nhrp nhs 10.200.15.15
 ip nhrp registration no-unique
 tunnel source Ethernet0/1
 tunnel mode gre multipoint
 tunnel key 1500
end

```

#### Проверка

```
R14#sh dmvpn
Legend: Attrb --> S - Static, D - Dynamic, I - Incomplete
        N - NATed, L - Local, X - No Socket
        # Ent --> Number of NHRP entries with same NBMA peer
        NHS Status: E --> Expecting Replies, R --> Responding, W --> Waiting
        UpDn Time --> Up or Down Time for a Tunnel
==========================================================================

Interface: Tunnel1, IPv4 NHRP Details
Type:Hub, NHRP Peers:2,

 # Ent  Peer NBMA Addr Peer Tunnel Add State  UpDn Tm Attrb
 ----- --------------- --------------- ----- -------- -----
     1 152.152.152.18     10.200.14.27    UP 00:09:29     D
     1 152.152.152.10     10.200.14.28    UP 00:08:19     D

R14#sh ip nhrp
10.200.14.27/32 via 10.200.14.27
   Tunnel1 created 00:21:50, expire 01:59:29
   Type: dynamic, Flags: registered used nhop
   NBMA address: 152.152.152.18
10.200.14.28/32 via 10.200.14.28
   Tunnel1 created 00:20:41, expire 01:59:06
   Type: dynamic, Flags: registered used nhop
   NBMA address: 152.152.152.10
R14#ping 10.200.14.27
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.200.14.27, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
R14#ping 10.200.14.28
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.200.14.28, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
```

```
R15#sh dmvpn
Legend: Attrb --> S - Static, D - Dynamic, I - Incomplete
        N - NATed, L - Local, X - No Socket
        # Ent --> Number of NHRP entries with same NBMA peer
        NHS Status: E --> Expecting Replies, R --> Responding, W --> Waiting
        UpDn Time --> Up or Down Time for a Tunnel
==========================================================================

Interface: Tunnel1, IPv4 NHRP Details
Type:Hub, NHRP Peers:2,

 # Ent  Peer NBMA Addr Peer Tunnel Add State  UpDn Tm Attrb
 ----- --------------- --------------- ----- -------- -----
     1 152.152.152.18     10.200.15.27    UP 00:10:46     D
     1 152.152.152.14     10.200.15.28    UP 00:09:37     D
R15#sh ip nhrp
10.200.15.27/32 via 10.200.15.27
   Tunnel1 created 00:25:30, expire 01:59:06
   Type: dynamic, Flags: registered used nhop
   NBMA address: 152.152.152.18
10.200.15.28/32 via 10.200.15.28
   Tunnel1 created 00:24:20, expire 01:59:28
   Type: dynamic, Flags: registered used nhop
   NBMA address: 152.152.152.14
R15#ping 10.200.15.27
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.200.15.27, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
R15#ping 10.200.15.28
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.200.15.28, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
```




Конфигурационныe файлы можно найти по [ссылке](./cfg).
