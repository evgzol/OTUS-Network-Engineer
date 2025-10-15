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
