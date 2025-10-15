### ��������� OSPF

� �������� ��������� ���������� ������������� � �������� ���������� �������� OSPF.

���������� ��������� ��������� OSPF �� ������� ������� 1. OSPF ��������� �� ��������������� � ������������ ������ ����/�������������.
����� ������� ����� ���� ��������������� � ���� L3-������������ � ����, ����� ������������ area 0. ���� ����������� ��������� Loopback-����������.
�������� ����������� ������� OSPF �� �����������: Hello == 3, Dead == 12 � ����� ��������� ���������� ���������. ������� silent-interface all
(passive interface default), ����� �� ���������� hello � ���������� ����� � ��� �������. ��������� ������� ospf network-type p2p ��� ����������� ������ ���������.

����� ���� ������� ������������ �� �������. 
![Region1.svg](./img/Region1.svg)

������� OSPF router-id, ��� �������� ��������� � Loopback-���:

| NE | router-id | 
|-----| ----| 
|Reg1-R1 | 11.11.11.11 |
|Reg1-R2 | 12.12.12.12 |
|Reg1-DSW1 | 111.111.111.111 |
|Reg1-DSW2 | 112.112.112.112 |


���� ��������� ��������� ��������� OSPF �� ���� CE � DSW.
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
