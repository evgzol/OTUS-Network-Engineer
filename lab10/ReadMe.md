# Лабораторная работа №10 "Масштабируемость и дизайн iBGP"

## Цель
Настроить iBGP в офисе Москва
Настроить iBGP в сети провайдера Триада
Организовать полную IP связанность всех сетей 

## Задание

1. Настроите iBGP в офисом Москва между маршрутизаторами R14 и R15.
2. Настроите iBGP в провайдере Триада, с использованием RR.
3. Настройте офиса Москва так, чтобы приоритетным провайдером стал Ламас.
4. Настройте офиса С.-Петербург так, чтобы трафик до любого офиса распределялся по двум линкам одновременно.
5. Все сети в лабораторной работе должны иметь IP связность.

Схема сети представлена на рисунке ниже:

![Topology-with-IP.jpg](./img/Topology-with-IP.jpg)


## Решение

### Настройка

#### Москва

Настраиваю R14 с маршрутизатором по R15 (по lo0), использую as-path prepend на для управления входящим трафиком, чтобы ухудшить маршрут через Киторн (R22). Также использую route map FILTER чтобы не анонсировать по BGP "серые" подсети. 
```
router bgp 1001
 bgp router-id 10.77.0.14
 bgp log-neighbor-changes
 network 10.77.0.0 mask 255.255.0.0
 redistribute connected route-map FILTER
 neighbor 10.77.0.15 remote-as 1001
 neighbor 10.77.0.15 update-source Loopback0
 neighbor 10.77.0.15 next-hop-self
 neighbor 111.111.111.1 remote-as 101
 neighbor 111.111.111.1 prefix-list MSK_SUM out
 neighbor 111.111.111.1 route-map PREP out
!
ip forward-protocol nd
!
!
no ip http server
no ip http secure-server
ip route 10.77.0.0 255.255.0.0 Null0 210
!
!
ip prefix-list MSK_SUM seq 10 permit 0.0.0.0/0 le 24
!
ip prefix-list FILTER seq 5 deny 10.0.0.0/8 le 32
ip prefix-list FILTER seq 10 permit 0.0.0.0/0 le 32
!
route-map FILTER permit 10
 match ip address prefix-list FILTER
!
route-map PREP permit 10
 set as-path PREP 1001 1001 1001 1001 1001 1001
```
Настраиваю R15 с маршрутизатором по R14 (по lo0) так, чтобы приоритетным провайдером стал Ламас. Для этого использую атрибут Local preference чтобы управлять исходящим траффиком.
```
router bgp 1001
 bgp router-id 10.77.0.15
 bgp log-neighbor-changes
 network 10.77.0.0 mask 255.255.0.0
 redistribute connected route-map FILTER
 neighbor 10.77.0.14 remote-as 1001
 neighbor 10.77.0.14 update-source Loopback0
 neighbor 10.77.0.14 next-hop-self
 neighbor 131.131.131.1 remote-as 301
 neighbor 131.131.131.1 prefix-list MSK_SUM out
 neighbor 131.131.131.1 route-map LAMAS in
 neighbor 131.131.131.1 remote-as 301
!
ip route 10.77.0.0 255.255.0.0 Null0 210
!
!
ip prefix-list MSK_SUM seq 10 permit 0.0.0.0/0 le 24
!
ip prefix-list FILTER seq 5 deny 10.0.0.0/8 le 32
ip prefix-list FILTER seq 10 permit 0.0.0.0/0 le 32
!
route-map LAMAS permit 10
 set local-preference 200
```


#### Триада
В качестве RR выберем R23.

Настройки ниже.

R23:
```
router bgp 520
 bgp log-neighbor-changes
 bgp listen range 10.152.0.0/24 peer-group TRIADA
 network 10.152.1.0 mask 255.255.255.0
 neighbor TRIADA peer-group
 neighbor TRIADA remote-as 520
 neighbor TRIADA update-source Loopback0
 neighbor TRIADA route-reflector-client
 neighbor TRIADA next-hop-self
 neighbor 111.111.111.9 remote-as 101
```

R24
```
router bgp 520
 bgp log-neighbor-changes
 network 10.152.1.0 mask 255.255.255.0
 neighbor 10.152.0.23 remote-as 520
 neighbor 10.152.0.23 update-source Loopback0
 neighbor 152.152.152.2 remote-as 2042
 neighbor 131.131.131.5 remote-as 301
```

R25
```
router bgp 520
 bgp log-neighbor-changes
 neighbor 10.152.0.23 remote-as 520
 neighbor 10.152.0.23 update-source Loopback0
```

R26
```
router bgp 520
 bgp log-neighbor-changes
 network 10.152.1.0 mask 255.255.255.0
 neighbor 10.152.0.23 remote-as 520
 neighbor 10.152.0.23 update-source Loopback0
 neighbor 152.152.152.6 remote-as 2042
```

#### Санкт-Петербург
Отделение СПб настраиваю так, чтобы трафик до любого офиса распределялся по двум линкам одновременно (R18):

```
router bgp 2042
 bgp router-id 10.78.0.18
 bgp log-neighbor-changes
 network 10.78.0.0 mask 255.255.0.0
 neighbor 152.152.152.1 remote-as 520
 neighbor 152.152.152.5 remote-as 520
 maximum-paths 2
```

### Проверка

Проверяем соседства.

На R15:
```
R15#sh ip bgp summ
BGP router identifier 10.77.0.15, local AS number 1001
BGP table version is 10, main routing table version 10
7 network entries using 980 bytes of memory
8 path entries using 640 bytes of memory
12/7 BGP path/bestpath attribute entries using 1728 bytes of memory
4 BGP AS-PATH entries using 96 bytes of memory
0 BGP route-map cache entries using 0 bytes of memory
0 BGP filter-list cache entries using 0 bytes of memory
BGP using 3444 total bytes of memory
BGP activity 7/0 prefixes, 9/1 paths, scan interval 60 secs

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.77.0.14      4         1001     129     132       10    0    0 01:52:22        2
131.131.131.1   4          301     989     990       10    0    0 14:55:19        4
```

На R23:

```
R23#sh ip bgp summ
BGP router identifier 10.152.0.23, local AS number 520
BGP table version is 8, main routing table version 8
5 network entries using 700 bytes of memory
9 path entries using 720 bytes of memory
7/5 BGP path/bestpath attribute entries using 1008 bytes of memory
6 BGP AS-PATH entries using 144 bytes of memory
0 BGP route-map cache entries using 0 bytes of memory
0 BGP filter-list cache entries using 0 bytes of memory
BGP using 2572 total bytes of memory
BGP activity 5/0 prefixes, 9/0 paths, scan interval 60 secs

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
*10.152.0.24    4          520     992     997        8    0    0 14:57:49        4
*10.152.0.25    4          520     989     996        8    0    0 14:57:58        0
*10.152.0.26    4          520     994     996        8    0    0 14:57:48        2
111.111.111.9   4          101     995     996        8    0    0 14:58:02        3
* Dynamically created based on a listen range command
Dynamically created neighbors: 3, Subnet ranges: 1

BGP peergroup TRIADA listen range group members:
  10.152.0.0/24

Total dynamically created neighbors: 3/(100 max), Subnet ranges: 1
```



Таблица маршрутизации BGP на R15, лучшие маршруты идут через Ламас (131.131.131.1):

```
R15#sh ip ro bgp
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       a - application route
       + - replicated route, % - next hop override

Gateway of last resort is not set

      10.0.0.0/8 is variably subnetted, 20 subnets, 4 masks
B        10.78.0.0/16 [20/0] via 131.131.131.1, 15:05:47
B        10.152.1.0/24 [20/0] via 131.131.131.1, 15:05:47
      111.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
B        111.111.111.0/24 [20/0] via 131.131.131.1, 15:05:47
B        111.111.111.0/30 [200/0] via 10.77.0.14, 02:04:23
      131.131.0.0/16 is variably subnetted, 3 subnets, 3 masks
B        131.131.131.0/24 [20/0] via 131.131.131.1, 15:06:10
```

Проверию таблицу BGP на R14, лучше маршруты будут через R15(10.77.0.15):
```
R14#sh ip ro bgp
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       a - application route
       + - replicated route, % - next hop override

Gateway of last resort is not set

      10.0.0.0/8 is variably subnetted, 20 subnets, 4 masks
B        10.78.0.0/16 [200/0] via 10.77.0.15, 02:06:49
B        10.152.1.0/24 [200/0] via 10.77.0.15, 02:06:49
      111.0.0.0/8 is variably subnetted, 3 subnets, 3 masks
B        111.111.111.0/24 [200/0] via 10.77.0.15, 02:06:49
      131.131.0.0/16 is variably subnetted, 2 subnets, 2 masks
B        131.131.131.0/24 [200/0] via 10.77.0.15, 02:06:49
B        131.131.131.0/30 [200/0] via 10.77.0.15, 02:06:49
```
Проверяю маршруты на Киторн (R22):
```
R22#sh ip bgp all
For address family: IPv4 Unicast

BGP table version is 6, local router ID is 10.111.0.22
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *   10.77.0.0/16     111.111.111.2            0             0 1001 1001 1001 1001 1001 1001 1001 i
 *                    111.111.111.10                         0 520 301 1001 i
 *>                   111.111.111.6                          0 301 1001 i
 *   10.78.0.0/16     111.111.111.2                          0 1001 1001 1001 1001 1001 1001 1001 301 520 2042 i
 *                    111.111.111.6                          0 301 520 2042 i
 *>                   111.111.111.10                         0 520 2042 i
 *   10.152.1.0/24    111.111.111.2                          0 1001 1001 1001 1001 1001 1001 1001 301 520 i
 *                    111.111.111.6                          0 301 520 i
 *>                   111.111.111.10                         0 520 i
 *>  111.111.111.0/24 0.0.0.0                  0         32768 i
 *   131.131.131.0/24 111.111.111.2                          0 1001 1001 1001 1001 1001 1001 1001 301 i
     Network          Next Hop            Metric LocPrf Weight Path
 *                    111.111.111.10                         0 520 301 i
 *>                   111.111.111.6            0             0 301 i

For address family: IPv4 Multicast


For address family: L2VPN E-VPN


For address family: MVPNv4 Unicast

```

На R18(Санкт-Петербург) видно, что тесть два пути во внешние сети:
```
R18#sh ip ro bgp
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

      10.0.0.0/8 is variably subnetted, 8 subnets, 4 masks
B        10.77.0.0/16 [20/0] via 152.152.152.5, 15:14:40
                      [20/0] via 152.152.152.1, 15:14:40
B        10.152.1.0/24 [20/0] via 152.152.152.5, 15:15:11
                       [20/0] via 152.152.152.1, 15:15:11
      111.0.0.0/24 is subnetted, 1 subnets
B        111.111.111.0 [20/0] via 152.152.152.5, 15:14:40
                       [20/0] via 152.152.152.1, 15:14:40
      131.131.0.0/24 is subnetted, 1 subnets
B        131.131.131.0 [20/0] via 152.152.152.5, 15:14:41
                       [20/0] via 152.152.152.1, 15:14:41
```

Конфигурационныe файлы можно найти по [ссылке](./cfg).
