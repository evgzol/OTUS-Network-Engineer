# Лабораторная работа №5 "Маршрутизация на основе политик (PBR)"
## **Задание**

1. Настроите политику маршрутизации для сетей офиса.
2. Распределите трафик между двумя линками с провайдером.
3. Настройте отслеживание линка через технологию IP SLA.(только для IPv4)
4. Настройте для офиса Лабытнанги маршрут по-умолчанию.


## 1. Настройка политики маршрутизации

В качестве DGW для VLAN-ов Чокурдаха выступает маршрутизатор R28.
Для корректной работы пинга настроим статические обратные маршруты в локальную сеть Чокурдаха на граничных роутерах (R25, R26).

```
R25(config)#ip route 10.14.10.0 255.255.255.0 152.152.152.14
R25(config)#ip route 10.14.20.0 255.255.255.0 152.152.152.14
R25(config)#ip route 10.14.100.0 255.255.255.0 152.152.152.14
```

```
R26(config)#ip route 10.14.10.0 255.255.255.0 152.152.152.10
R26(config)#ip route 10.14.20.0 255.255.255.0 152.152.152.10
R26(config)#ip route 10.14.100.0 255.255.255.0 152.152.152.10
```

Маршрутизаторы провайдера Триада пингуются:


с VPC30:
```
VPC30> ping 152.152.152.9

84 bytes from 152.152.152.9 icmp_seq=1 ttl=254 time=1.500 ms
84 bytes from 152.152.152.9 icmp_seq=2 ttl=254 time=1.652 ms
84 bytes from 152.152.152.9 icmp_seq=3 ttl=254 time=2.021 ms
84 bytes from 152.152.152.9 icmp_seq=4 ttl=254 time=1.816 ms
84 bytes from 152.152.152.9 icmp_seq=5 ttl=254 time=1.937 ms

VPC30> ping 152.152.152.13

84 bytes from 152.152.152.13 icmp_seq=1 ttl=254 time=2.409 ms
84 bytes from 152.152.152.13 icmp_seq=2 ttl=254 time=2.330 ms
84 bytes from 152.152.152.13 icmp_seq=3 ttl=254 time=1.631 ms
84 bytes from 152.152.152.13 icmp_seq=4 ttl=254 time=6.408 ms
84 bytes from 152.152.152.13 icmp_seq=5 ttl=254 time=1.416 ms
```
с VPC31:

```
VPC31> ping 152.152.152.9

84 bytes from 152.152.152.9 icmp_seq=1 ttl=254 time=2.180 ms
84 bytes from 152.152.152.9 icmp_seq=2 ttl=254 time=1.955 ms
84 bytes from 152.152.152.9 icmp_seq=3 ttl=254 time=4.438 ms
84 bytes from 152.152.152.9 icmp_seq=4 ttl=254 time=2.119 ms
84 bytes from 152.152.152.9 icmp_seq=5 ttl=254 time=2.585 ms

VPC31> ping 152.152.152.13

84 bytes from 152.152.152.13 icmp_seq=1 ttl=254 time=4.360 ms
84 bytes from 152.152.152.13 icmp_seq=2 ttl=254 time=2.161 ms
84 bytes from 152.152.152.13 icmp_seq=3 ttl=254 time=2.909 ms
84 bytes from 152.152.152.13 icmp_seq=4 ttl=254 time=1.748 ms
84 bytes from 152.152.152.13 icmp_seq=5 ttl=254 time=2.179 ms
```

## 2. Настройка распределения трафика

Согласно задания распределяем трафик следующим образом:
VLAN10 --- идёт по линку с R25
VLAN20 --- идёт по линку с R26

Произведём настройку ACL для VLAN10 и VLAN20 на R28:

```
ip access-list standard ACL_VLAN10
  permit 10.14.10.0 0.0.0.255
exit

ip access-list standard ACL_VLAN20
  permit 10.14.20.0 0.0.0.255     
exit
```

Настраиваем route-map:

```
route-map BALANCING_POLICY permit 10
  match ip address ACL_VLAN10
  set ip next-hop 152.152.152.13
exit

route-map BALANCING_POLICY permit 20  
  match ip address ACL_VLAN20  
  set ip next-hop 152.152.152.9
exit
```

Назначаем route-map на соответствующие входные интерфейсы:

```
interface e0/2.10
  ip policy route-map BALANCING_POLICY
exit

interface e0/2.20                
  ip policy route-map BALANCING_POLICY
end
```

Т.о. траффик от VPC30 (VLAN10) проходит только по линку с R25, а от VPC31 (VLAN20) --- по линку с R26


```
VPC30> ping 152.152.152.13

84 bytes from 152.152.152.13 icmp_seq=1 ttl=254 time=2.738 ms
84 bytes from 152.152.152.13 icmp_seq=2 ttl=254 time=2.552 ms
84 bytes from 152.152.152.13 icmp_seq=3 ttl=254 time=2.957 ms
84 bytes from 152.152.152.13 icmp_seq=4 ttl=254 time=2.596 ms
84 bytes from 152.152.152.13 icmp_seq=5 ttl=254 time=3.036 ms

VPC30> ping 152.152.152.9

*152.152.152.13 icmp_seq=1 ttl=254 time=2.187 ms (ICMP type:3, code:1, Destination host unreachable)
*152.152.152.13 icmp_seq=2 ttl=254 time=1.799 ms (ICMP type:3, code:1, Destination host unreachable)
*152.152.152.13 icmp_seq=3 ttl=254 time=2.690 ms (ICMP type:3, code:1, Destination host unreachable)
*152.152.152.13 icmp_seq=4 ttl=254 time=2.872 ms (ICMP type:3, code:1, Destination host unreachable)
*152.152.152.13 icmp_seq=5 ttl=254 time=2.526 ms (ICMP type:3, code:1, Destination host unreachable)
```

```
VPC31> ping 152.152.152.13

*152.152.152.9 icmp_seq=1 ttl=254 time=2.604 ms (ICMP type:3, code:1, Destination host unreachable)
*152.152.152.9 icmp_seq=2 ttl=254 time=1.757 ms (ICMP type:3, code:1, Destination host unreachable)
*152.152.152.9 icmp_seq=3 ttl=254 time=2.943 ms (ICMP type:3, code:1, Destination host unreachable)
*152.152.152.9 icmp_seq=4 ttl=254 time=1.504 ms (ICMP type:3, code:1, Destination host unreachable)
*152.152.152.9 icmp_seq=5 ttl=254 time=1.299 ms (ICMP type:3, code:1, Destination host unreachable)

VPC31> ping 152.152.152.9

84 bytes from 152.152.152.9 icmp_seq=1 ttl=254 time=1.688 ms
84 bytes from 152.152.152.9 icmp_seq=2 ttl=254 time=19.346 ms
84 bytes from 152.152.152.9 icmp_seq=3 ttl=254 time=2.438 ms
84 bytes from 152.152.152.9 icmp_seq=4 ttl=254 time=2.517 ms
84 bytes from 152.152.152.9 icmp_seq=5 ttl=254 time=3.438 ms
```

## 3. Настройка отслеживания линка через технологию IP SLA


Создаю IP SLA 152.152.152.9 (R26 e0/1) и 152.152.152.13 (R25 e0/3) и корректирую route map-ы:

```
ip sla 1
  icmp-echo 152.152.152.13 source-ip 152.152.152.14
  frequency 10
ip sla schedule 1 life forever start-time now

ip sla 2
  icmp-echo 152.152.152.9 source-ip 152.152.152.10
  frequency 10
ip sla schedule 2 life forever start-time now

track 1 ip sla 1 reachability
  delay down 60 up 60
exit

track 2 ip sla 2 reachability
  delay down 60 up 60          
exit

route-map BALANCING_POLICY permit 10  
  no set ip next-hop 152.152.152.13
  set ip next-hop verify-availability 152.152.152.13 10 track 1
  set ip next-hop verify-availability 152.152.152.9 20 track 2
exit

route-map BALANCING_POLICY 20 
  no set ip next-hop 152.152.152.9
  set ip next-hop verify-availability 152.152.152.9 10 track 2
  set ip next-hop verify-availability 152.152.152.13 20 track 1
exit

```

Проверка (с VPC30):

```
VPC30> ping 152.152.152.13

84 bytes from 152.152.152.13 icmp_seq=1 ttl=254 time=1.771 ms
84 bytes from 152.152.152.13 icmp_seq=2 ttl=254 time=1.972 ms
84 bytes from 152.152.152.13 icmp_seq=3 ttl=254 time=3.628 ms
84 bytes from 152.152.152.13 icmp_seq=4 ttl=254 time=2.370 ms
84 bytes from 152.152.152.13 icmp_seq=5 ttl=254 time=2.662 ms

VPC30> ping 152.152.152.9

*152.152.152.13 icmp_seq=1 ttl=254 time=4.139 ms (ICMP type:3, code:1, Destination host unreachable)
*152.152.152.13 icmp_seq=2 ttl=254 time=2.514 ms (ICMP type:3, code:1, Destination host unreachable)
*152.152.152.13 icmp_seq=3 ttl=254 time=2.135 ms (ICMP type:3, code:1, Destination host unreachable)
*152.152.152.13 icmp_seq=4 ttl=254 time=2.373 ms (ICMP type:3, code:1, Destination host unreachable)
*152.152.152.13 icmp_seq=5 ttl=254 time=2.056 ms (ICMP type:3, code:1, Destination host unreachable)

```

Выключаю маршрутизатор R25:
```

VPC30> ping 152.152.152.13

152.152.152.13 icmp_seq=1 timeout
152.152.152.13 icmp_seq=2 timeout
152.152.152.13 icmp_seq=3 timeout
^C^[[A^[[A
VPC30> ping 152.152.152.9

152.152.152.9 icmp_seq=1 timeout
152.152.152.9 icmp_seq=2 timeout
152.152.152.9 icmp_seq=3 timeout
152.152.152.9 icmp_seq=4 timeout
^C
```

Ждём задержку в 60 секунд (указали выше):


```
VPC30> ping 152.152.152.13

*152.152.152.9 icmp_seq=1 ttl=254 time=2.977 ms (ICMP type:3, code:1, Destination host unreachable)
*152.152.152.9 icmp_seq=2 ttl=254 time=2.939 ms (ICMP type:3, code:1, Destination host unreachable)
*152.152.152.9 icmp_seq=3 ttl=254 time=1.677 ms (ICMP type:3, code:1, Destination host unreachable)
*152.152.152.9 icmp_seq=4 ttl=254 time=3.283 ms (ICMP type:3, code:1, Destination host unreachable)
*152.152.152.9 icmp_seq=5 ttl=254 time=2.339 ms (ICMP type:3, code:1, Destination host unreachable)

VPC30> ping 152.152.152.9

84 bytes from 152.152.152.9 icmp_seq=1 ttl=254 time=1.649 ms
84 bytes from 152.152.152.9 icmp_seq=2 ttl=254 time=5.996 ms
84 bytes from 152.152.152.9 icmp_seq=3 ttl=254 time=2.469 ms
84 bytes from 152.152.152.9 icmp_seq=4 ttl=254 time=2.457 ms
84 bytes from 152.152.152.9 icmp_seq=5 ttl=254 time=2.270 ms
```
Т.о. произошла перестройка.

На R28 (до):
```
show route-map 

R28#show route-map
route-map BALANCING_POLICY, permit, sequence 10
  Match clauses:
    ip address (access-lists): ACL_VLAN10
  Set clauses:
    ip next-hop verify-availability 152.152.152.13 10 track 1  [up]
    ip next-hop verify-availability 152.152.152.9 20 track 2  [up]
  Policy routing matches: 25 packets, 2550 bytes
route-map BALANCING_POLICY, permit, sequence 20
  Match clauses:
    ip address (access-lists): ACL_VLAN20
  Set clauses:
    ip next-hop verify-availability 152.152.152.9 10 track 2  [up]
    ip next-hop verify-availability 152.152.152.13 20 track 1  [up]
  Policy routing matches: 10 packets, 1020 bytes

```

На R28 (после):

```
R28#
*Jul 30 14:28:29.371: %TRACK-6-STATE: 1 ip sla 1 reachability Up -> Down
R28#show route-map
route-map BALANCING_POLICY, permit, sequence 10
  Match clauses:
    ip address (access-lists): ACL_VLAN10
  Set clauses:
    ip next-hop verify-availability 152.152.152.13 10 track 1  [down]
    ip next-hop verify-availability 152.152.152.9 20 track 2  [up]
  Policy routing matches: 43 packets, 4386 bytes
route-map BALANCING_POLICY, permit, sequence 20
  Match clauses:
    ip address (access-lists): ACL_VLAN20
  Set clauses:
    ip next-hop verify-availability 152.152.152.9 10 track 2  [up]
    ip next-hop verify-availability 152.152.152.13 20 track 1  [down]
  Policy routing matches: 10 packets, 1020 bytes
```

## 4. Настройка маршрута по умолчанию

Настраиваем для Лабытнанги маршрут по умолчанию.


На R27:
```
ip route 0.0.0.0 0.0.0.0 152.152.152.17
```

Проверка:

```
R27#sh ip rou
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       a - application route
       + - replicated route, % - next hop override

Gateway of last resort is 152.152.152.17 to network 0.0.0.0

S*    0.0.0.0/0 [1/0] via 152.152.152.17
      10.0.0.0/32 is subnetted, 1 subnets
C        10.89.0.27 is directly connected, Loopback0
      152.152.0.0/16 is variably subnetted, 2 subnets, 2 masks
C        152.152.152.16/30 is directly connected, Ethernet0/0
L        152.152.152.18/32 is directly connected, Ethernet0/0
```

Конфигурационныe файлы можно найти по [ссылке](./cfg).
