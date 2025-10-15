### Настройка OSPF

В качестве протокола внутренней маршрутизации в регионах используем протокол OSPF.

Рассмотрим настройку протокола OSPF на примере Региона 1. OSPF поднимаем на маршрутизаторах и коммутаторах уровня ядра/распределения.
Ввиду наличия всего двух маршрутизаторов и двух L3-коммутаторов в сети, можно ограничиться area 0. Всем устройствам назначены Loopback-интеряейсы.
Изменены стандартные таймеры OSPF на интерфейсах: Hello == 3, Dead == 12 с целью улучшения сходимости протокола. Включен silent-interface all
(passive interface default), чтобы не отправлять hello в клиентские порты и вне региона. Применена команда ospf network-type p2p для оптимизации работы протокола.

Схема сети региона представлена на рисунке. 
![Region1.svg](./img/Region1.svg)

Таблица OSPF router-id, для простоты совпадает с Loopback-ами:

| NE | router-id | 
|-----| ----| 
|Reg1-R1 | 11.11.11.11 |
|Reg1-R2 | 12.12.12.12 |
|Reg1-DSW1 | 111.111.111.111 |
|Reg1-DSW2 | 112.112.112.112 |


Ниже приведена настройка протокола OSPF на всех CE и DSW.
<details>
<summary>  Reg1-R1 </summary>
```   
ospf 1 router-id 11.11.11.11
 import-route direct
 silent-interface all
 undo silent-interface GE0/1
 undo silent-interface GE0/2
 undo silent-interface GE5/0
  area 0
 quit
quit

int loopback0
 ospf 1 area 0
quit

int GE0/1
 ospf 1 area 0
 ospf network-type p2p
 ospf timer hello 3
 ospf timer dead 12
quit

int GE0/2
 ospf 1 area 0
 ospf network-type p2p
 ospf timer hello 3
 ospf timer dead 12
quit

int GE5/0
 ospf 1 area 0
 ospf network-type p2p
 ospf timer hello 3
 ospf timer dead 12
quit

```   
</details>


<details>
<summary>  Reg1-R2 </summary>
```   
ospf 1 router-id 12.12.12.12
 import-route direct
 silent-interface all
 undo silent-interface GE0/1
 undo silent-interface GE0/2
 undo silent-interface GE5/0
  area 0
 quit
quit

int loopback0
 ospf 1 area 0
quit

int GE0/1
 ospf 1 area 0
 ospf network-type p2p
 ospf timer hello 3
 ospf timer dead 12
quit

int GE0/2
 ospf 1 area 0
 ospf network-type p2p
 ospf timer hello 3
 ospf timer dead 12
quit

int GE5/0
 ospf 1 area 0
 ospf network-type p2p
 ospf timer hello 3
 ospf timer dead 12
quit
```   
</details>


<details>
<summary>  Reg1-DSW1 </summary>
```  
ospf 1 router-id 111.111.111.111
 silent-interface all
 undo silent-interface GE1/0/1
 undo silent-interface GE1/0/2
  area 0
 quit
quit

int loopback0
 ip address 111.111.111.111 32
 ospf 1 area 0
quit

int GE1/0/1
 ospf 1 area 0
 ospf network-type p2p
 ospf timer hello 3
 ospf timer dead 12
quit

int GE1/0/2
 ospf 1 area 0
 ospf network-type p2p
 ospf timer hello 3
 ospf timer dead 12
quit

interface Vlan-interface10
 ospf 1 area 0
quit

interface Vlan-interface20
 ospf 1 area 0
quit

interface Vlan-interface99
 ospf 1 area 0
quit
```   
</details>

<details>
<summary>  Reg1-DSW2 </summary>
```  
ospf 1 router-id 112.112.112.112
 silent-interface all
 undo silent-interface GE1/0/1
 undo silent-interface GE1/0/2
  area 0
 quit
quit

int loopback0
 ip address 112.112.112.112 32
 ospf 1 area 0
quit

int GE1/0/1
 ospf 1 area 0
 ospf network-type p2p
 ospf timer hello 3
 ospf timer dead 12
quit

int GE1/0/2
 ospf 1 area 0
 ospf network-type p2p
 ospf timer hello 3
 ospf timer dead 12
quit

interface Vlan-interface10
 ospf 1 area 0
quit

interface Vlan-interface20
 ospf 1 area 0
quit

interface Vlan-interface99
 ospf 1 area 0
quit

```   
</details>


Диагностика протокола приведена ниже.

<details>
<summary> На маршрутизаторах </summary>
```   
[Reg1-R1]disp ip routing-table

Destinations : 22       Routes : 27

Destination/Mask   Proto   Pre Cost        NextHop         Interface
0.0.0.0/32         Direct  0   0           127.0.0.1       InLoop0
10.1.0.0/31        Direct  0   0           10.1.0.0        GE0/1
10.1.0.0/32        Direct  0   0           127.0.0.1       InLoop0
10.1.0.2/31        Direct  0   0           10.1.0.2        GE0/2
10.1.0.2/32        Direct  0   0           127.0.0.1       InLoop0
10.1.0.4/31        Direct  0   0           10.1.0.4        GE5/0
10.1.0.4/32        Direct  0   0           127.0.0.1       InLoop0
10.1.0.6/31        O_INTRA 10  2           10.1.0.1        GE0/1
                   O_INTRA 10  2           10.1.0.5        GE5/0
10.1.0.8/31        O_INTRA 10  2           10.1.0.1        GE0/1
                   O_INTRA 10  2           10.1.0.3        GE0/2
10.1.10.0/24       O_INTRA 10  2           10.1.0.3        GE0/2
                   O_INTRA 10  2           10.1.0.5        GE5/0
10.1.20.0/24       O_INTRA 10  2           10.1.0.3        GE0/2
                   O_INTRA 10  2           10.1.0.5        GE5/0
10.1.99.0/24       O_INTRA 10  2           10.1.0.3        GE0/2
                   O_INTRA 10  2           10.1.0.5        GE5/0
11.11.11.11/32     Direct  0   0           127.0.0.1       InLoop0
12.12.12.12/32     O_INTRA 10  1           10.1.0.1        GE0/1
111.111.111.111/32 O_INTRA 10  1           10.1.0.3        GE0/2
112.112.112.112/32 O_INTRA 10  1           10.1.0.5        GE5/0
127.0.0.0/8        Direct  0   0           127.0.0.1       InLoop0
127.0.0.1/32       Direct  0   0           127.0.0.1       InLoop0
127.255.255.255/32 Direct  0   0           127.0.0.1       InLoop0
224.0.0.0/4        Direct  0   0           0.0.0.0         NULL0
224.0.0.0/24       Direct  0   0           0.0.0.0         NULL0
255.255.255.255/32 Direct  0   0           127.0.0.1       InLoop0
[Reg1-R1]
[Reg1-R1]disp ospf routing

         OSPF Process 1 with Router ID 11.11.11.11
                  Routing Table

 Routing for network
 Destination        Cost     Type    NextHop         AdvRouter       Area
 10.1.0.2/31        1        Stub    0.0.0.0         11.11.11.11     0.0.0.0
 112.112.112.112/32 1        Stub    10.1.0.5        112.112.112.112 0.0.0.0
 10.1.0.4/31        1        Stub    0.0.0.0         11.11.11.11     0.0.0.0
 10.1.20.0/24       2        Stub    10.1.0.3        111.111.111.111 0.0.0.0
 10.1.20.0/24       2        Stub    10.1.0.5        112.112.112.112 0.0.0.0
 10.1.10.0/24       2        Stub    10.1.0.3        111.111.111.111 0.0.0.0
 10.1.10.0/24       2        Stub    10.1.0.5        112.112.112.112 0.0.0.0
 111.111.111.111/32 1        Stub    10.1.0.3        111.111.111.111 0.0.0.0
 10.1.99.0/24       2        Stub    10.1.0.3        111.111.111.111 0.0.0.0
 10.1.99.0/24       2        Stub    10.1.0.5        112.112.112.112 0.0.0.0
 11.11.11.11/32     0        Stub    0.0.0.0         11.11.11.11     0.0.0.0
 10.1.0.8/31        2        Stub    10.1.0.1        12.12.12.12     0.0.0.0
 10.1.0.8/31        2        Stub    10.1.0.3        111.111.111.111 0.0.0.0
 10.1.0.6/31        2        Stub    10.1.0.1        12.12.12.12     0.0.0.0
 10.1.0.6/31        2        Stub    10.1.0.5        112.112.112.112 0.0.0.0
 10.1.0.0/31        1        Stub    0.0.0.0         11.11.11.11     0.0.0.0
 12.12.12.12/32     1        Stub    10.1.0.1        12.12.12.12     0.0.0.0

 Total nets: 17
 Intra area: 17  Inter area: 0  ASE: 0  NSSA: 0
[Reg1-R1]disp ospf peer

         OSPF Process 1 with Router ID 11.11.11.11
               Neighbor Brief Information

 Area: 0.0.0.0
 Router ID       Address         Pri Dead-Time  State             Interface
 12.12.12.12     10.1.0.1        1   10         Full/ -           GE0/1
 111.111.111.111 10.1.0.3        1   10         Full/ -           GE0/2
 112.112.112.112 10.1.0.5        1   11         Full/ -           GE5/0
[Reg1-R1]disp ospf peer verbose

         OSPF Process 1 with Router ID 11.11.11.11
                 Neighbors

 Area 0.0.0.0 interface 10.1.0.0(GigabitEthernet0/1)'s neighbors
 Router ID: 12.12.12.12      Address: 10.1.0.1         GR State: Normal
   State: Full  Mode: Nbr is master  Priority: 1
   DR: None   BDR: None   MTU: 0
   Options is 0x42 (-|O|-|-|-|-|E|-)
   Dead timer due in 9   sec
   Neighbor is up for 00:01:36
   Authentication Sequence: [ 0 ]
   Neighbor state change count: 5
   BFD status: Disabled

 Area 0.0.0.0 interface 10.1.0.2(GigabitEthernet0/2)'s neighbors
 Router ID: 111.111.111.111  Address: 10.1.0.3         GR State: Normal
   State: Full  Mode: Nbr is master  Priority: 1
   DR: None   BDR: None   MTU: 0
   Options is 0x42 (-|O|-|-|-|-|E|-)
   Dead timer due in 9   sec
   Neighbor is up for 00:00:57
   Authentication Sequence: [ 0 ]
   Neighbor state change count: 5
   BFD status: Disabled

 Area 0.0.0.0 interface 10.1.0.4(GigabitEthernet5/0)'s neighbors
 Router ID: 112.112.112.112  Address: 10.1.0.5         GR State: Normal
   State: Full  Mode: Nbr is master  Priority: 1
   DR: None   BDR: None   MTU: 0
   Options is 0x42 (-|O|-|-|-|-|E|-)
   Dead timer due in 10  sec
   Neighbor is up for 00:00:51
   Authentication Sequence: [ 0 ]
   Neighbor state change count: 5
   BFD status: Disabled

 Last Neighbor Down Event:
 Router ID: 112.112.112.112
 Local Address: 10.1.0.4
 Remote Address: 10.1.0.5
 Time: Oct 13 05:55:32 2025
 Reason: Ospf Interface Parameters Changed

[Reg1-R1]disp ospf lsdb

         OSPF Process 1 with Router ID 11.11.11.11
                 Link State Database

                         Area: 0.0.0.0
 Type      LinkState ID    AdvRouter       Age  Len   Sequence  Metric
 Router    111.111.111.111 111.111.111.111 73   120   80000023  0
 Router    112.112.112.112 112.112.112.112 68   120   80000023  0
 Router    11.11.11.11     11.11.11.11     66   108   80000033  0
 Router    12.12.12.12     12.12.12.12     67   108   8000002B  0

                 AS External Database
 Type      LinkState ID    AdvRouter       Age  Len   Sequence  Metric
 External  11.11.11.11     11.11.11.11     995  36    80000017  1
 External  10.1.0.4        11.11.11.11     995  36    80000017  1
 External  10.1.0.0        11.11.11.11     995  36    80000017  1
 External  10.1.0.2        11.11.11.11     995  36    80000017  1
 External  12.12.12.12     12.12.12.12     1439 36    80000016  1
 External  10.1.0.8        12.12.12.12     1439 36    80000016  1
 External  10.1.0.6        12.12.12.12     1439 36    80000016  1
 External  10.1.0.0        12.12.12.12     1439 36    80000016  1

```   
</details>


<details>
<summary> На коммутаторах </summary>
```   
[Reg1-DSW1]disp ip routing-table

Destinations : 32       Routes : 34

Destination/Mask   Proto   Pre Cost        NextHop         Interface
0.0.0.0/32         Direct  0   0           127.0.0.1       InLoop0
10.1.0.0/31        O_INTRA 10  2           10.1.0.2        GE1/0/1
                                           10.1.0.8        GE1/0/2
10.1.0.2/31        Direct  0   0           10.1.0.3        GE1/0/1
10.1.0.3/32        Direct  0   0           127.0.0.1       InLoop0
10.1.0.4/31        O_INTRA 10  2           10.1.0.2        GE1/0/1
10.1.0.6/31        O_INTRA 10  2           10.1.0.8        GE1/0/2
10.1.0.8/31        Direct  0   0           10.1.0.9        GE1/0/2
10.1.0.9/32        Direct  0   0           127.0.0.1       InLoop0
10.1.10.0/24       Direct  0   0           10.1.10.251     Vlan10
10.1.10.0/32       Direct  0   0           10.1.10.251     Vlan10
10.1.10.251/32     Direct  0   0           127.0.0.1       InLoop0
10.1.10.255/32     Direct  0   0           10.1.10.251     Vlan10
10.1.20.0/24       Direct  0   0           10.1.20.251     Vlan20
10.1.20.0/32       Direct  0   0           10.1.20.251     Vlan20
10.1.20.251/32     Direct  0   0           127.0.0.1       InLoop0
10.1.20.255/32     Direct  0   0           10.1.20.251     Vlan20
10.1.99.0/24       Direct  0   0           10.1.99.251     Vlan99
10.1.99.0/32       Direct  0   0           10.1.99.251     Vlan99
10.1.99.251/32     Direct  0   0           127.0.0.1       InLoop0
10.1.99.254/32     Direct  1   0           127.0.0.1       InLoop0
10.1.99.255/32     Direct  0   0           10.1.99.251     Vlan99
11.11.11.11/32     O_INTRA 10  1           10.1.0.2        GE1/0/1
12.12.12.12/32     O_INTRA 10  1           10.1.0.8        GE1/0/2
111.111.111.111/32 Direct  0   0           127.0.0.1       InLoop0
112.112.112.112/32 O_INTRA 10  2           10.1.0.2        GE1/0/1
                                           10.1.0.8        GE1/0/2
127.0.0.0/8        Direct  0   0           127.0.0.1       InLoop0
127.0.0.0/32       Direct  0   0           127.0.0.1       InLoop0
127.0.0.1/32       Direct  0   0           127.0.0.1       InLoop0
127.255.255.255/32 Direct  0   0           127.0.0.1       InLoop0
224.0.0.0/4        Direct  0   0           0.0.0.0         NULL0
224.0.0.0/24       Direct  0   0           0.0.0.0         NULL0
255.255.255.255/32 Direct  0   0           127.0.0.1       InLoop0
[Reg1-DSW1]disp ospf routing

         OSPF Process 1 with Router ID 111.111.111.111
                  Routing Table

                Topology base (MTID 0)

 Routing for network
 Destination        Cost     Type    NextHop         AdvRouter       Area
 10.1.10.0/24       1        Stub    0.0.0.0         111.111.111.111 0.0.0.0
 12.12.12.12/32     1        Stub    10.1.0.8        12.12.12.12     0.0.0.0
 10.1.0.0/31        2        Stub    10.1.0.2        11.11.11.11     0.0.0.0
 10.1.0.0/31        2        Stub    10.1.0.8        12.12.12.12     0.0.0.0
 10.1.0.2/31        1        Stub    0.0.0.0         111.111.111.111 0.0.0.0
 10.1.0.4/31        2        Stub    10.1.0.2        11.11.11.11     0.0.0.0
 10.1.0.6/31        2        Stub    10.1.0.8        12.12.12.12     0.0.0.0
 10.1.0.8/31        1        Stub    0.0.0.0         111.111.111.111 0.0.0.0
 112.112.112.112/32 2        Stub    10.1.0.2        112.112.112.112 0.0.0.0
 112.112.112.112/32 2        Stub    10.1.0.8        112.112.112.112 0.0.0.0
 11.11.11.11/32     1        Stub    10.1.0.2        11.11.11.11     0.0.0.0
 10.1.20.0/24       1        Stub    0.0.0.0         111.111.111.111 0.0.0.0
 10.1.99.0/24       1        Stub    0.0.0.0         111.111.111.111 0.0.0.0
 111.111.111.111/32 0        Stub    0.0.0.0         111.111.111.111 0.0.0.0

 Total nets: 14
 Intra area: 14  Inter area: 0  ASE: 0  NSSA: 0
[Reg1-DSW1]disp ospf peer

         OSPF Process 1 with Router ID 111.111.111.111
               Neighbor Brief Information

 Area: 0.0.0.0
 Router ID       Address         Pri Dead-Time  State             Interface
 11.11.11.11     10.1.0.2        1   11         Full/ -           GE1/0/1
 12.12.12.12     10.1.0.8        1   9          Full/ -           GE1/0/2
[Reg1-DSW1]disp ospf peer verbose

         OSPF Process 1 with Router ID 111.111.111.111
                 Neighbors


 Area 0.0.0.0 interface 10.1.0.3(GigabitEthernet1/0/1)'s neighbors
 Router ID: 11.11.11.11      Address: 10.1.0.2         GR state: Normal
   State: Full  Mode: Nbr is slave  Priority: 1
   DR: None   BDR: None   MTU: 0
   Options is 0x42 (-|O|-|-|-|-|E|-)
   Dead timer due in 9   sec
   Neighbor is up for 00:03:17
   Authentication sequence: [ 0 ]
   Neighbor state change count: 5
   BFD status: Disabled


 Area 0.0.0.0 interface 10.1.0.9(GigabitEthernet1/0/2)'s neighbors
 Router ID: 12.12.12.12      Address: 10.1.0.8         GR state: Normal
   State: Full  Mode: Nbr is slave  Priority: 1
   DR: None   BDR: None   MTU: 0
   Options is 0x42 (-|O|-|-|-|-|E|-)
   Dead timer due in 11  sec
   Neighbor is up for 00:03:17
   Authentication sequence: [ 0 ]
   Neighbor state change count: 5
   BFD status: Disabled

 Last Neighbor Down Event:
 Router ID: 12.12.12.12
 Local Address: 10.1.0.9
 Remote Address: 10.1.0.8
 Time: Oct 13 05:57:07 2025
 Reason: Ospf Interface Parameters Changed

[Reg1-DSW1]disp ospf lsdb

         OSPF Process 1 with Router ID 111.111.111.111
                 Link State Database

                         Area: 0.0.0.0
 Type      LinkState ID    AdvRouter       Age  Len   Sequence  Metric
 Router    111.111.111.111 111.111.111.111 202  120   80000023  0
 Router    112.112.112.112 112.112.112.112 198  120   80000023  0
 Router    11.11.11.11     11.11.11.11     197  108   80000033  0
 Router    12.12.12.12     12.12.12.12     197  108   8000002B  0

                 AS External Database
 Type      LinkState ID    AdvRouter       Age  Len   Sequence  Metric
 External  12.12.12.12     12.12.12.12     1571 36    80000016  1
 External  11.11.11.11     11.11.11.11     1128 36    80000017  1
 External  10.1.0.8        12.12.12.12     1571 36    80000016  1
 External  10.1.0.4        11.11.11.11     1128 36    80000017  1
 External  10.1.0.6        12.12.12.12     1571 36    80000016  1
 External  10.1.0.0        11.11.11.11     1128 36    80000017  1
 External  10.1.0.0        12.12.12.12     1571 36    80000016  1
 External  10.1.0.2        11.11.11.11     1128 36    80000017  1

```   
</details>


Проверка сетевой связности (с VPC пингуются коммутаторы и маршрутизаторы по Loopback-ам):
```   
Reg1-VPC1> ping 11.11.11.11
84 bytes from 11.11.11.11 icmp_seq=1 ttl=254 time=2.000 ms
84 bytes from 11.11.11.11 icmp_seq=2 ttl=254 time=1.500 ms
84 bytes from 11.11.11.11 icmp_seq=3 ttl=254 time=3.000 ms
84 bytes from 11.11.11.11 icmp_seq=4 ttl=254 time=3.000 ms
84 bytes from 11.11.11.11 icmp_seq=5 ttl=254 time=2.000 ms

Reg1-VPC1> ping 111.111.111.111
84 bytes from 111.111.111.111 icmp_seq=1 ttl=255 time=4.000 ms
84 bytes from 111.111.111.111 icmp_seq=2 ttl=255 time=1.500 ms
84 bytes from 111.111.111.111 icmp_seq=3 ttl=255 time=2.000 ms
84 bytes from 111.111.111.111 icmp_seq=4 ttl=255 time=1.500 ms
84 bytes from 111.111.111.111 icmp_seq=5 ttl=255 time=2.000 ms

Reg1-VPC1> ping 12.12.12.12
84 bytes from 12.12.12.12 icmp_seq=1 ttl=254 time=2.000 ms
84 bytes from 12.12.12.12 icmp_seq=2 ttl=254 time=2.000 ms
84 bytes from 12.12.12.12 icmp_seq=3 ttl=254 time=1.500 ms
84 bytes from 12.12.12.12 icmp_seq=4 ttl=254 time=2.500 ms
84 bytes from 12.12.12.12 icmp_seq=5 ttl=254 time=2.000 ms

Reg1-VPC1> ping 112.112.112.112
84 bytes from 112.112.112.112 icmp_seq=1 ttl=255 time=1.500 ms
84 bytes from 112.112.112.112 icmp_seq=2 ttl=255 time=1.500 ms
84 bytes from 112.112.112.112 icmp_seq=3 ttl=255 time=1.000 ms
84 bytes from 112.112.112.112 icmp_seq=4 ttl=255 time=1.000 ms
84 bytes from 112.112.112.112 icmp_seq=5 ttl=255 time=1.500 ms
```   

С маршрутизатора пингуется VPC:
```   
[Reg1-R1]ping 10.1.10.1
Ping 10.1.10.1 (10.1.10.1): 56 data bytes, press CTRL+C to break
56 bytes from 10.1.10.1: icmp_seq=0 ttl=63 time=7.193 ms
56 bytes from 10.1.10.1: icmp_seq=1 ttl=63 time=2.076 ms
56 bytes from 10.1.10.1: icmp_seq=2 ttl=63 time=2.008 ms
56 bytes from 10.1.10.1: icmp_seq=3 ttl=63 time=2.527 ms
56 bytes from 10.1.10.1: icmp_seq=4 ttl=63 time=2.293 ms

--- Ping statistics for 10.1.10.1 ---
5 packet(s) transmitted, 5 packet(s) received, 0.0% packet loss
round-trip min/avg/max/std-dev = 2.008/3.219/7.193/1.995 ms
```   
Настройка в остальных регионах аналогична.

Конфигурационныe файлы можно найти по [ссылке](./cfg).
