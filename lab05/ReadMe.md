# ������������ ������ �5 "������������� �� ������ ������� (PBR)"
## **�������**

1. ��������� �������� ������������� ��� ����� �����.
2. ������������ ������ ����� ����� ������� � �����������.
3. ��������� ������������ ����� ����� ���������� IP SLA.(������ ��� IPv4)
4. ��������� ��� ����� ���������� ������� ��-���������.

����� ���� c IP-���������� ������������ �� ������� ����:

![Topology-with-IP.jpg](./img/Topology-with-IP.jpg)

### ������� ������������� ��������� ������������

| Net                                                          | Vlan | Name                              | Description     |
|--------------------------------------------------------------|------|-----------------------------------|-----------------|
| 10.77.0.0/16                                                 | -    | ���������� ����                   | ������          |
| 10.78.0.0/16                                                 | -    |                                   | �����-��������� |
| 10.89.0.0/16                                                 | -    |                                   | ����������      |
| 10.14.0.0/16                                                 | -    |                                   | ��������         |
| 10.111.0.0/16                                                | -    |                                   | ������          |
| 10.131.0.0/16                                                | -    |                                   | �����           |
| 10.152.0.0/16                                                | -    |                                   | ������          |
| 10.77.1.0/24                                                 | -    | ������������ ��� �������������    | ������          |
| 10.78.1.0/24                                                 | -    |                                   | �����-��������� |
| 10.89.1.0/24                                                 | -    |                                   | ����������      |
| 10.14.1.0/24                                                 | -    |                                   | ��������         |
| 10.111.1.0/24                                                | -    |                                   | ������          |
| 10.131.1.0/24                                                | -    |                                   | �����           |
| 10.152.1.0/24                                                | -    |                                   | ������          |
| 10.77.0.0/24                                                 | -    | ������������ ��� Loopback         | ������          |
| 10.78.0.0/24                                                 | -    |                                   | �����-��������� |
| 10.89.0.0/24                                                 | -    |                                   | ����������      |
| 10.14.0.0/24                                                 | -    |                                   | ��������         |
| 10.111.0.0/24                                                | -    |                                   | ������          |
| 10.131.0.0/24                                                | -    |                                   | �����           |
| 10.152.0.0/24                                                | -    |                                   | ������          |
| 111.111.111.0/24                                             | -    | ������� ����                      | ������          |
| 131.131.131.0/24                                             | -    |                                   | �����           |
| 152.152.152.0/24                                             | -    |                                   | ������          |
| 10.77.10.0/24                                                | 10   | VLAN10, ������������              | ������          |
| 10.78.10.0/24                                                | 10   |                                   | �����-��������� |
| 10.14.10.0/24                                                | 10   |                                   | ��������         |
| 10.77.20.0/24                                                | 20   | VLAN20, ������������              | ������          |
| 10.78.20.0/24                                                | 20   |                                   | �����-��������� |
| 10.14.20.0/24                                                | 20   |                                   | ��������         |
| 10.77.100.0/24                                               | 100  | VLAN100, ���������� ������������� | ������          |
| 10.78.100.0/24                                               | 100  |                                   | �����-��������� |
| 10.14.100.0/24                                               | 100  |                                   | ��������         |


### ������� ������������� IP-������� ��� ���������� ������ � ������������� � ��������� � ����������

HostName|Loopback|Port|IP-address|Network|Location|Description|�����������
---|---|---|---|---|---|---|---
R27|10.89.0.27|e0/0|152.152.152.18|152.152.152.16/30|����������|***\_to\_R27\_(e0/1)\_Triada|
R28|10.14.0.28|e0/0|152.152.152.10|152.152.152.8/30|��������|***\_to\_R26\_(e0/1)\_Triada|
R28|10.14.0.28|e0/1|152.152.152.14|152.152.152.12/30|��������|***\_to\_R25\_(e0/3)\_Triada|
R28|10.14.0.28|e0/2|-|-|��������|***\_to\_SW29\_(e0/2)|
R28|10.14.0.28|e0/2.10|10.14.10.28|10.14.10.0/24|��������|***\_VLAN\_10|
R28|10.14.0.28|e0/2.20|10.14.20.28|10.14.20.0/24|��������|***\_VLAN\_20|
R28|10.14.0.28|e0/2.100|10.14.100.28|10.14.100.0/24|��������|***\_VLAN\_100\_MNG|
R23|10.152.0.23|e0/0|111.111.111.10|111.111.111.8/30|������ (���������)|***\_to\_R22\_(e0/2)\_Kitorn|
R23|10.152.0.23|e0/1|10.152.1.5|10.152.1.4/30|������ (���������)|***\_to\_R25\_(e0/0)|
R23|10.152.0.23|e0/2|10.152.1.1|10.152.1.0/30|������ (���������)|***\_to\_R24\_(e0/2)|
R24|10.152.0.24|e0/0|131.131.131.6|131.131.131.4/30|������ (���������)|***\_to\_R21\_(e0/2)\_Lamas|
R24|10.152.0.24|e0/1|10.152.1.13|10.152.1.12/30|������ (���������)|***\_to\_R26\_(e0/0)|
R24|10.152.0.24|e0/2|10.152.1.2|10.152.1.0/30|������ (���������)|***\_to\_R23\_(e0/2)|
R24|10.152.0.24|e0/3|152.152.152.1|152.152.152.0/30|������ (���������)|***\_to\_R18\_(e0/2)\_SPB|
R25|10.152.0.25|e0/0|10.152.1.6|10.152.1.4/30|������ (���������)|***\_to\_R23(e0/1)|
R25|10.152.0.25|e0/1|152.152.152.17|152.152.152.16/30|������ (���������)|***\_to\_R27\_(e0/0)\_LBT|
R25|10.152.0.25|e0/2|10.152.1.9|10.152.1.8/30|������ (���������)|***\_to\_R26\_(e0/2)|
R25|10.152.0.25|e0/3|152.152.152.13|152.152.152.12/30|������ (���������)|***\_to\_R28\_(e0/1)\_CHK|
R26|10.152.0.26|e0/0|10.152.1.14|10.152.1.12/30|������ (���������)|***\_to\_R24\_(e0/1)|
R26|10.152.0.26|e0/1|152.152.152.9|152.152.152.8/30|������ (���������)|***\_to\_R28\_(e0/0)\_CHK|
R26|10.152.0.26|e0/2|10.152.1.10|10.152.1.8/30|������ (���������)|***\_to\_R25\_(e0/2)|
R26|10.152.0.26|e0/3|152.152.152.5|152.152.152.4/30|������ (���������)|***\_to\_R18\_(e0/3)\_SPB|
VPC30|-|eth0|10.14.10.130|10.14.10.0/24|��������||VLAN10
VPC31|-|eth0|10.14.20.131|10.14.20.0/24|��������||VLAN20
SW29|-|Vlan100|10.14.100.29|10.14.100.0/24|��������||VLAN100

## 1. ��������� �������� �������������

� �������� DGW ��� VLAN-�� ��������� ��������� ������������� R28.
��� ���������� ������ ����� �������� ����������� �������� �������� � ��������� ���� ��������� �� ��������� �������� (R25, R26).

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

�������������� ���������� ������ ���������:


� VPC30:
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
� VPC31:

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

## 2. ��������� ������������� �������

�������� ������� ������������ ������ ��������� �������:
VLAN10 --- ��� �� ����� � R25
VLAN20 --- ��� �� ����� � R26

��������� ��������� ACL ��� VLAN10 � VLAN20 �� R28:

```
ip access-list standard ACL_VLAN10
  permit 10.14.10.0 0.0.0.255
exit

ip access-list standard ACL_VLAN20
  permit 10.14.20.0 0.0.0.255     
exit
```

����������� route-map:

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

��������� route-map �� ��������������� ������� ����������:

```
interface e0/2.10
  ip policy route-map BALANCING_POLICY
exit

interface e0/2.20                
  ip policy route-map BALANCING_POLICY
end
```

�.�. ������� �� VPC30 (VLAN10) �������� ������ �� ����� � R25, � �� VPC31 (VLAN20) --- �� ����� � R26


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

## 3. ��������� ������������ ����� ����� ���������� IP SLA


������ IP SLA 152.152.152.9 (R26 e0/1) � 152.152.152.13 (R25 e0/3) � ����������� route map-�:

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

�������� (� VPC30):

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

�������� ������������� R25:
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

��� �������� � 60 ������ (������� ����):


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
�.�. ��������� �����������.

�� R28 (��):
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

�� R28 (�����):

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

## 4. ��������� �������� �� ���������

����������� ��� ���������� ������� �� ���������.


�� R27:
```
ip route 0.0.0.0 0.0.0.0 152.152.152.17
```

��������:

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

�.�. �������� "Gateway of last resort".

���������������e ����� ����� ����� �� [������](./cfg).
