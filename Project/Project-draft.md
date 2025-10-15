# Построение георазнесённой мультисервисной сети оператора связи

Данная работа является курсовым проектом по курсу "Network Engineer. Professional" компании OTUS.

Проектная работа реализована в среде H3C Cloud Lab на оборудовании H3C.

## Оглавление
1. [Цель](#цель)
2. [Задачи](#задачи)
3. [Сетевая топология](#Сетевая-топология)
4. [План IP адресации](#План-IP-адресации)
5. [Протоколы L2-L3 внутри региона](#Протоколы-L2-L3-внутри-региона)

    5.1. [Настройка агрегации линков](#Настройка-агрегации-линков)
	
    5.2. [Настройка VRRP](#Настройка-VRRP)
	
    5.3. [Настройка Spanning Tree](#Настройка-Spanning-Tree)
   
    5.4. [Объединение коммутаторов в стек](#Объединение-коммутаторов-в-стек)
   
    5.5. [Проблемы стекирования](#Проблемы-стекирования)
   
    5.6. [Настройка BFD MAD](#Настройка-BFD-MAD)
   
    5.7. [Настройка M-LAG](#Настройка-M-LAG)
   
    5.8. [Настройка OSPF](#Настройка-OSPF)
   
6. [Протоколы L3 транспортной сети](#Протоколы-L3-транспортной-сети)

    6.1. [Настройка IS-IS](#Настройка-IS-IS)

    6.2. [Настройка eBGP](#Настройка-EBGP)
	
    6.3. [Настройка iBGP](#Настройка-iBGP)

    6.4. [Настройка MPLS](#Настройка-MPLS)

    6.5. [Настройка TE](#Настройка-TE)

7. [Выводы](#Выводы)

## Цель

Разработка архитектуры георазнесённой мультисервисной сети оператора связи с проработкой транспортной и региональных составляющих.
## Задачи

- Проектирование транспортной сети оператора связи
- Проектирование сетей региональных филиалов
- Проектирование гибкого адресного пространства
- Проектирование сетевых протоколов, обеспечивающих мультисервисность, отказоустойчивость и быструю сходимость 

## Сетевая топология

Топология сети приведена на рисунке (рекомендуется открыть рисунок в отдельной вкладке/окне).

![Topology.svg](./img/Topology.svg)

В данной топологии приведена упрощённая модель георазнесённой сети оператора сотовой связи, состоящей из трёх регионов и транспортной сети. В каждом регионе есть два сервиса;
- Radio &mdash; обеспечивает связность сетевых элементов, таких как BSC, RNC, eNodeB с SGSN/SGW (их роль выпоняют VPC);
- Tech &mdash; технологическая сеть оператора.

В каждом регионе применена упрощённая трёх-уровневая архитектура Cisco-сети, где уровень ядра и распределения объединены в один уровень (коммутаторы DSW), уровень доступа оставлен без изменений (коммутаторы ASW).

Транспортная сеть также упрощена: маршрутизаторы транспортной сети выпоняют совмещённую функцию P/PE-маршрутизатора.

Данные упрощения применены для возможности реализации в лабораторной среде: лишние элементы потребляют память и производительность ядер процессора хост-системы. ;-)

Принятые графические обозначения:

P/PE-маршрутизатор транспортной сети:

![PE.svg](./img/PE.svg)

CE-маршрутизатор региона:

![CE.svg](./img/CE.svg)

Коммутатор уровня ядра/распределения региона:

![DSW.svg](./img/DSW.svg)

Коммутатор уровня доступа региона:

![ASW.svg](./img/ASW.svg)

Loopback-интерфейсы сетевых элементов:

![Loopback.svg](./img/Loopback.svg)

## План IP адресации

### Таблица распределения адресного пространства

| Net          | Vlan | Name                             | Description |
|--------------|------|----------------------------------|-------------|
| 10.0.0.0/16  | -    | Сеть оператора                   | Транспорт   |
| 10.1.0.0/16  | -    |                                  | Регион 1    |
| 10.2.0.0/16  | -    |                                  | Регион 2    |
| 10.3.0.0/16  | -    |                                  | Регион 3    |
| 10.0.0.0/24  | -    | Пространство для интерконнекта   | Транспорт   |
| 10.1.0.0/24  | -    |                                  | Регион 1    |
| 10.2.0.0/24  | -    |                                  | Регион 2    |
| 10.3.0.0/24  | -    |                                  | Регион 3    |
| 10.1.10.0/24 | 10   | VLAN10                           | Регион 1    |
| 10.2.10.0/24 | 10   |                                  | Регион 2    |
| 10.3.10.0/24 | 10   |                                  | Регион 3    |
| 10.1.20.0/24 | 20   | VLAN20                           | Регион 1    |
| 10.2.20.0/24 | 20   |                                  | Регион 2    |
| 10.3.20.0/24 | 20   |                                  | Регион 3    |
| 10.1.99.0/24 | 99   | VLAN99, управление коммутаторами | Регион 1    |
| 10.2.99.0/24 | 99   |                                  | Регион 2    |
| 10.3.99.0/24 | 99   |                                  | Регион 3    |


### Таблица распределения IP-адресов

| HostName             | Loopback        | Port     | IP-address  | Network      | Location  | Description            | Комментарий |
|----------------------|-----------------|----------|-------------|--------------|-----------|------------------------|-------------|
|                      |                 |          |             |              |           |                        |             |
| PE1                  | 1.1.1.1         | GE0/0/0  | 10.0.0.0    | 10.0.0.0/31  | Транспорт | to PE2 (GE0/0/0)       |             |
| PE1                  | 1.1.1.1         | GE0/0/1  | 10.0.0.2    | 10.0.0.2/31  | Транспорт | to PE3 (GE0/0/1)       |             |
| PE1                  | 1.1.1.1         | GE0/0/2  | 10.0.0.4    | 10.0.0.4.31  | Транспорт | to PE4 (GE0/0/2)       |             |
| PE1                  | 1.1.1.1         | GE0/0/7  | 10.0.0.12   | 10.0.0.12/31 | Транспорт | to Reg1-R1 (GE0/0)     |             |
| PE2                  | 2.2.2.2         | GE0/0/0  | 10.0.0.1    | 10.0.0.0/31  | Транспорт | to PE1 (GE0/0/0)       |             |
| PE2                  | 2.2.2.2         | GE0/0/1  | 10.0.0.8    | 10.0.0.8/31  | Транспорт | to PE4 (GE0/0/1)       |             |
| PE2                  | 2.2.2.2         | GE0/0/2  | 10.0.0.7    | 10.0.0.6/31  | Транспорт | to PE3 (GE0/0/2)       |             |
| PE2                  | 2.2.2.2         | GE0/0/7  | 10.0.0.14   | 10.0.0.14/31 | Транспорт | to Reg2-R1 (GE0/0)     |             |
| PE3                  | 3.3.3.3         | GE0/0/0  | 10.0.0.10   | 10.0.0.10/31 | Транспорт | to PE4 (GE0/0/0)       |             |
| PE3                  | 3.3.3.3         | GE0/0/1  | 10.0.0.3    | 10.0.0.2/31  | Транспорт | to PE1 (GE0/0/1)       |             |
| PE3                  | 3.3.3.3         | GE0/0/2  | 10.0.0.6    | 10.0.0.6/31  | Транспорт | to PE2 (GE0/0/2)       |             |
| PE3                  | 3.3.3.3         | GE0/0/7  | 10.0.0.16   | 10.0.0.16/31 | Транспорт | to Reg1-R2 (GE0/0)     |             |
| PE3                  | 3.3.3.3         | GE0/0/8  | 10.0.0.20   | 10.0.0.20/31 | Транспорт | to Reg3-R1 (GE0/0)     |             |
| PE4                  | 4.4.4.4         | GE0/0/0  | 10.0.0.11   | 10.0.0.10/31 | Транспорт | to PE3 (GE0/0/0)       |             |
| PE4                  | 4.4.4.4         | GE0/0/1  | 10.0.0.9    | 10.0.0.8/31  | Транспорт | to PE2 (GE0/0/1)       |             |
| PE4                  | 4.4.4.4         | GE0/0/2  | 10.0.0.5    | 10.0.0.4.31  | Транспорт | to PE1 (GE0/0/2)       |             |
| PE4                  | 4.4.4.4         | GE0/0/7  | 10.0.0.18   | 10.0.0.18/31 | Транспорт | to Reg2-R2 (GE0/0)     |             |
| PE4                  | 4.4.4.4         | GE0/0/8  | 10.0.0.22   | 10.0.0.22/31 | Транспорт | to Reg3-R2 (GE0/0)     |             |
| Reg1-R1              | 11.11.11.11     | GE0/0    | 10.0.0.13   | 10.0.0.12/31 | Регион 1  | to PE1 (GE0/7)         |             |
| Reg1-R1              | 11.11.11.11     | GE0/1    | 10.1.0.0    | 10.1.0.0/31  | Регион 1  | to Reg1-R2 (GE0/1)     |             |
| Reg1-R1              | 11.11.11.11     | GE0/2    | 10.1.0.2    | 10.1.0.2/31  | Регион 1  | to Reg1-DSW1 (GE1/0/1) |             |
| Reg1-R1              | 11.11.11.11     | GE5/0    | 10.1.0.4    | 10.1.0.4/31  | Регион 1  | to Reg1-DSW2 (GE1/0/2) |             |
| Reg1-R2              | 12.12.12.12     | GE0/0    | 10.0.0.17   | 10.0.0.16/31 | Регион 1  | to PE3 (GE0/7)         |             |
| Reg1-R2              | 12.12.12.12     | GE0/1    | 10.1.0.1    | 10.1.0.0/31  | Регион 1  | to Reg1-R1 (GE0/1)     |             |
| Reg1-R2              | 12.12.12.12     | GE0/2    | 10.1.0.6    | 10.1.0.6/31  | Регион 1  | to Reg1-DSW2 (GE1/0/1) |             |
| Reg1-R2              | 12.12.12.12     | GE5/0    | 10.1.0.8    | 10.1.0.8/31  | Регион 1  | to Reg1-DSW1 (GE1/0/2) |             |
| Reg1-DSW1            | 111.111.111.111 | GE1/0/1  | 10.1.0.3    | 10.1.0.2/31  | Регион 1  | to Reg1-R1 (GE0/2)     |             |
| Reg1-DSW1            | 111.111.111.111 | GE1/0/2  | 10.1.0.9    | 10.1.0.6/31  | Регион 1  | to Reg1-R2 (GE5/0)     |             |
| Reg1-DSW2            | 112.112.112.112 | GE1/0/1  | 10.1.0.7    | 10.1.0.8/31  | Регион 1  | to Reg1-R2 (GE0/2)     |             |
| Reg1-DSW2            | 112.112.112.112 | GE1/0/2  | 10.1.0.5    | 10.1.0.4/31  | Регион 1  | to Reg1-R1 (GE5/0)     |             |
| Reg2-R1              | 21.21.21.21     | GE0/0    | 10.0.0.15   | 10.0.0.14/31 | Регион 2  | to PE2 (GE0/7)         |             |
| Reg2-R1              | 21.21.21.21     | GE0/1    | 10.2.0.0    | 10.2.0.0/31  | Регион 2  | to Reg2-R2 (Gi2/0/1)   |             |
| Reg2-R1              | 21.21.21.21     | Gi2/0/2  | 10.2.0.2    | 10.2.0.2/31  | Регион 2  | to Reg2-DSW1 (Gi1/0/1) |             |
| Reg2-R2              | 22.22.22.22     | GE0/0    | 10.0.0.19   | 10.0.0.18/31 | Регион 2  | to PE4 (GE0/7)         |             |
| Reg2-R2              | 22.22.22.22     | GE0/1    | 10.2.0.1    | 10.2.0.0/31  | Регион 2  | to Reg2-R1 (Gi2/0/1)   |             |
| Reg2-R2              | 22.22.22.22     | Gi2/0/2  | 10.2.0.6    | 10.2.0.6/31  | Регион 2  | to Reg2-DSW1 (Gi2/0/1) |             |
| Reg2-DSW             | 121.121.121.121 | Gi1/0/1  | 10.2.0.3    | 10.2.0.2/31  | Регион 2  | to Reg2-R1 (GE0/2)     |             |
| Reg2-DSW             | 121.121.121.121 | Gi2/0/1  | 10.2.0.7    | 10.2.0.6/31  | Регион 2  | to Reg2-R2 (GE0/2)     |             |
| Reg3-R1              | 31.31.31.31     | GE0/0    | 10.0.0.21   | 10.0.0.20/31 | Регион 3  | to PE3 (E0/8)          |             |
| Reg3-R1              | 31.31.31.31     | GE0/1    | 10.3.0.0    | 10.3.0.0/31  | Регион 3  | to Reg3-R2 (GE0/1)     |             |
| Reg3-R1              | 31.31.31.31     | GE0/2    | 10.3.0.2    | 10.3.0.2/31  | Регион 3  | to Reg3-DSW1 (GE1/0/1) |             |
| Reg3-R1              | 31.31.31.31     | GE5/0    | 10.3.0.4    | 10.3.0.4/31  | Регион 3  | to Reg3-DSW2 (GE1/0/2) |             |
| Reg3-R2              | 32.32.32.32     | GE0/0    | 10.0.0.23   | 10.0.0.22/31 | Регион 3  | to PE4 (GE0/8)         |             |
| Reg3-R2              | 32.32.32.32     | GE0/1    | 10.3.0.1    | 10.3.0.0/31  | Регион 3  | to Reg3-R1 (GE0/1)     |             |
| Reg3-R2              | 32.32.32.32     | GE0/2    | 10.3.0.6    | 10.3.0.6/31  | Регион 3  | to Reg3-DSW2 (GE1/0/1) |             |
| Reg3-R2              | 32.32.32.32     | GE5/0    | 10.3.0.8    | 10.3.0.8/31  | Регион 3  | to Reg3-DSW1 (GE1/0/2) |             |
| Reg3-DSW1            | 131.131.131.131 | GE1/0/1  | 10.3.0.3    | 10.3.0.2/31  | Регион 3  | to Reg3-R1 (GE0/2)     |             |
| Reg3-DSW1            | 131.131.131.131 | GE1/0/2  | 10.3.0.9    | 10.3.0.6/31  | Регион 3  | to Reg3-R2 (GE5/0)     |             |
| Reg3-DSW2            | 132.132.132.132 | GE1/0/1  | 10.3.0.7    | 10.3.0.8/31  | Регион 3  | to Reg3-R2 (GE0/2)     |             |
| Reg3-DSW2            | 132.132.132.132 | GE1/0/2  | 10.3.0.5    | 10.3.0.4/31  | Регион 3  | to Reg3-R1 (GE5/0)     |             |
| Reg1-DSW1, Reg1-DSW2 | -               | -        | 10.1.10.254 | 10.1.10.0/24 | Регион 1  |                        | VRRP VLAN10 |
| Reg1-DSW1, Reg1-DSW2 | -               | -        | 10.1.20.254 | 10.1.20.0/24 | Регион 1  |                        | VRRP VLAN20 |
| Reg1-DSW1, Reg1-DSW2 | -               | -        | 10.1.99.254 | 10.1.99.0/24 | Регион 1  |                        | VRRP VLAN99 |
| Reg2-R1, Reg2-R2     | -               | -        | 10.2.10.254 | 10.2.10.0/24 | Регион 2  |                        | VRRP VLAN10 |
| Reg2-R1, Reg2-R2     | -               | -        | 10.2.20.254 | 10.2.20.0/24 | Регион 2  |                        | VRRP VLAN20 |
| Reg2-R1, Reg2-R2     | -               | -        | 10.2.99.254 | 10.2.99.0/24 | Регион 2  |                        | VRRP VLAN99 |
| Reg3-DSW1, Reg3-DSW2 | -               | -        | 10.3.10.254 | 10.3.10.0/24 | Регион 3  |                        | VRRP VLAN10 |
| Reg3-DSW1, Reg3-DSW2 | -               | -        | 10.3.20.254 | 10.3.20.0/24 | Регион 3  |                        | VRRP VLAN20 |
| Reg3-DSW1, Reg3-DSW2 | -               | -        | 10.3.99.254 | 10.3.99.0/24 | Регион 3  |                        | VRRP VLAN99 |
| Reg1-DSW1            |                 | Vlan10   | 10.1.10.251 | 10.1.10.0/24 | Регион 1  |                        | VLAN10      |
| Reg1-DSW2            |                 | Vlan10   | 10.1.10.252 | 10.1.10.0/24 | Регион 1  |                        | VLAN10      |
| Reg1-DSW1            |                 | Vlan20   | 10.1.20.251 | 10.1.20.0/24 | Регион 1  |                        | VLAN20      |
| Reg1-DSW2            |                 | Vlan20   | 10.1.20.252 | 10.1.20.0/24 | Регион 1  |                        | VLAN20      |
| Reg1-DSW1            |                 | Vlan99   | 10.1.99.251 | 10.1.99.0/24 | Регион 1  |                        | VLAN99      |
| Reg1-DSW2            |                 | Vlan99   | 10.1.99.252 | 10.1.99.0/24 | Регион 1  |                        | VLAN99      |
| Reg2-R1              |                 | Vlan10   | 10.2.10.251 | 10.2.10.0/24 | Регион 2  |                        | VLAN10      |
| Reg2-R2              |                 | Vlan10   | 10.2.10.252 | 10.2.10.0/24 | Регион 2  |                        | VLAN10      |
| Reg2-R1              |                 | Vlan20   | 10.2.20.251 | 10.2.20.0/24 | Регион 2  |                        | VLAN20      |
| Reg2-R2              |                 | Vlan20   | 10.2.20.252 | 10.2.20.0/24 | Регион 2  |                        | VLAN20      |
| Reg2-R1              |                 | Vlan99   | 10.2.99.251 | 10.2.99.0/24 | Регион 2  |                        | VLAN99      |
| Reg2-R2              |                 | Vlan99   | 10.2.99.252 | 10.2.99.0/24 | Регион 2  |                        | VLAN99      |
| Reg3-DSW1            |                 | Vlan10   | 10.3.10.251 | 10.3.10.0/24 | Регион 1  |                        | VLAN10      |
| Reg3-DSW2            |                 | Vlan10   | 10.3.10.252 | 10.3.10.0/24 | Регион 1  |                        | VLAN10      |
| Reg3-DSW1            |                 | Vlan20   | 10.3.20.251 | 10.3.20.0/24 | Регион 1  |                        | VLAN20      |
| Reg3-DSW2            |                 | Vlan20   | 10.3.20.252 | 10.3.20.0/24 | Регион 1  |                        | VLAN20      |
| Reg3-DSW1            |                 | Vlan99   | 10.3.99.251 | 10.3.99.0/24 | Регион 1  |                        | VLAN99      |
| Reg3-DSW2            |                 | Vlan99   | 10.3.99.252 | 10.3.99.0/24 | Регион 1  |                        | VLAN99      |
| Reg2-DSW             |                 | Vlanif99 | 10.2.99.101 | 10.2.99.0/24 | Регион 2  |                        | VLAN99      |
| Reg2-ASW1            |                 | Vlanif99 | 10.2.99.201 | 10.2.99.0/24 | Регион 2  |                        | VLAN99      |
| Reg2-ASW2            |                 | Vlanif99 | 10.2.99.202 | 10.2.99.0/24 | Регион 2  |                        | VLAN99      |
| Reg1-VPC1            | -               | eth0     | 10.1.10.1   | 10.1.10.0/24 | Регион 1  |                        | VLAN10      |
| Reg1-VPC2            | -               | eth0     | 10.1.20.2   | 10.1.20.0/24 | Регион 1  |                        | VLAN20      |
| Reg2-VPC1            | -               | eth0     | 10.2.10.1   | 10.2.10.0/24 | Регион 2  |                        | VLAN10      |
| Reg2-VPC2            | -               | eth0     | 10.2.20.2   | 10.2.20.0/24 | Регион 2  |                        | VLAN20      |
| Reg3-VPC1            | -               | eth0     | 10.3.10.1   | 10.3.10.0/24 | Регион 3  |                        | VLAN10      |
| Reg3-VPC2            | -               | eth0     | 10.3.20.2   | 10.3.20.0/24 | Регион 3  |                        | VLAN20      |


## Протоколы L2-L3 внутри региона

### Настройка агрегации линков

На оборудовании H3C агрегация линков называется Bridge-Aggregation Group (BAGG) для коммутаторов
и Router-Aggregation Group (RAGG) для маршрутизаторов. Совместима с Ether-Trunk от Huawei и
Port-Channel от Cisco. Настраиваю BAGG100 между DSW-коммутаторами на портах 40GBase, BAGG101 между DSW И ASW на портах 10GBase.

Reg1-DSW1:
```   
interface Bridge-Aggregation100
 description to Reg1-DSW2 (BAGG100)
 port link-type trunk
 port trunk permit vlan 10 20 99
 link-aggregation mode dynamic
quit
 
interface Bridge-Aggregation101
 description to Reg1-ASW (BAGG1)
 port link-type trunk
 port trunk permit vlan 10 20 99
 link-aggregation mode dynamic
quit

int XGE1/0/49
 description to Reg1-ASW1 (XGE1/0/49)
 port link-type trunk
 port trunk permit vlan 10 20 99
 port link-aggregation group 101
quit

int XGE1/0/50
 description to Reg1-ASW1 (XGE1/0/50)
 port link-type trunk
 port trunk permit vlan 10 20 99
 port link-aggregation group 101
quit


int FGE1/0/53
 description to Reg1-DSW2 (FGE1/0/53)
 port link-type trunk
 port trunk permit vlan 10 20 99
 port link-aggregation group 100
quit

int FGE1/0/54
 description to Reg1-DSW2 (FGE1/0/54)
 port link-type trunk
 port trunk permit vlan 10 20 99
 port link-aggregation group 100
quit
```   
На Reg1-DSW2 настройка аналогична.


### Настройка VRRP

Настройка:

Reg1-DSW1:

```   
interface Vlan-interface99
 ip address 10.1.99.251 24
 vrrp vrid 1 virtual-ip 10.1.99.254
 vrrp vrid 1 priority 100
quit


interface Vlan-interface10
 ip address 10.1.10.251 24
 vrrp vrid 2 virtual-ip 10.1.10.254
 vrrp vrid 2 priority 100
quit

interface Vlan-interface20
 ip address 10.1.20.251 24
 vrrp vrid 3 virtual-ip 10.1.20.254
 vrrp vrid 3 priority 100
quit
```   
Reg1-DSW2:

```   
interface Vlan-interface99
 ip address 10.1.99.252 24
 vrrp vrid 1 virtual-ip 10.1.99.254
 vrrp vrid 1 priority 200
 quit

interface Vlan-interface10
 ip address 10.1.10.252 24
 vrrp vrid 2 virtual-ip 10.1.10.254
 vrrp vrid 2 priority 200
quit

interface Vlan-interface20
 ip address 10.1.20.252 24
 vrrp vrid 3 virtual-ip 10.1.20.254
 vrrp vrid 3 priority 200
quit
```   



Вывод состояния VRRP:

```   
[Reg1-DSW1]disp vrrp verbose
IPv4 Virtual Router Information:
 Running mode : Standard
 Total number of virtual routers : 3
   Interface Vlan-interface10
     VRID             : 2                   Adver Timer  : 100
     Admin Status     : Up                  State        : Backup
     Config Pri       : 100                 Running Pri  : 100
     Preempt Mode     : Yes                 Delay Time   : 0
     Become Master    : 0ms left
     Auth Type        : Not supported
     Version          : 3
     Virtual IP       : 10.1.10.254
     Virtual MAC      : 0000-5e00-0102
     Master IP        : 10.1.10.252

   Interface Vlan-interface20
     VRID             : 3                   Adver Timer  : 100
     Admin Status     : Up                  State        : Backup
     Config Pri       : 100                 Running Pri  : 100
     Preempt Mode     : Yes                 Delay Time   : 0
     Become Master    : 0ms left
     Auth Type        : Not supported
     Version          : 3
     Virtual IP       : 10.1.20.254
     Virtual MAC      : 0000-5e00-0103
     Master IP        : 10.1.20.252

   Interface Vlan-interface99
     VRID             : 1                   Adver Timer  : 100
     Admin Status     : Up                  State        : Backup
     Config Pri       : 100                 Running Pri  : 100
     Preempt Mode     : Yes                 Delay Time   : 0
     Become Master    : 0ms left
     Auth Type        : Not supported
     Version          : 3
     Virtual IP       : 10.1.99.254
     Virtual MAC      : 0000-5e00-0101
     Master IP        : 10.1.99.252

```   


```   
[Reg1-DSW2]disp vrrp verbose
IPv4 Virtual Router Information:
 Running mode : Standard
 Total number of virtual routers : 3
   Interface Vlan-interface10
     VRID             : 2                   Adver Timer  : 100
     Admin Status     : Up                  State        : Master
     Config Pri       : 200                 Running Pri  : 200
     Preempt Mode     : Yes                 Delay Time   : 0
     Auth Type        : Not supported
     Version          : 3
     Virtual IP       : 10.1.10.254
     Virtual MAC      : 0000-5e00-0102
     Master IP        : 10.1.10.252

   Interface Vlan-interface20
     VRID             : 3                   Adver Timer  : 100
     Admin Status     : Up                  State        : Master
     Config Pri       : 200                 Running Pri  : 200
     Preempt Mode     : Yes                 Delay Time   : 0
     Auth Type        : Not supported
     Version          : 3
     Virtual IP       : 10.1.20.254
     Virtual MAC      : 0000-5e00-0103
     Master IP        : 10.1.20.252

   Interface Vlan-interface99
     VRID             : 1                   Adver Timer  : 100
     Admin Status     : Up                  State        : Master
     Config Pri       : 200                 Running Pri  : 200
     Preempt Mode     : Yes                 Delay Time   : 0
     Auth Type        : Not supported
     Version          : 3
     Virtual IP       : 10.1.99.254
     Virtual MAC      : 0000-5e00-0101
     Master IP        : 10.1.99.252
```   


Проверка работы VRRP:

```   
Reg1-VPC1> ping 10.1.10.254
84 bytes from 10.1.10.254 icmp_seq=1 ttl=255 time=3.500 ms
84 bytes from 10.1.10.254 icmp_seq=2 ttl=255 time=2.000 ms
84 bytes from 10.1.10.254 icmp_seq=3 ttl=255 time=2.000 ms
84 bytes from 10.1.10.254 icmp_seq=4 ttl=255 time=2.000 ms
84 bytes from 10.1.10.254 icmp_seq=5 ttl=255 time=2.500 ms
Reg1-VPC1>
```   
Меняю vrrp priority на значение 200 на коммутаторе Reg1-DSW1, на Reg1-DSW2 ставлю в значение 100:


```   
[Reg1-DSW2]interface Vlan-interface99
[Reg1-DSW2-Vlan-interface99]vrrp vrid 1 priority 100

[Reg1-DSW1]interface Vlan-interface99
[Reg1-DSW1-Vlan-interface99]vrrp vrid 1 priority 200
```   

Провека:
```   
[Reg1-DSW1]disp vrrp verbose
IPv4 Virtual Router Information:
 Running mode : Standard
 Total number of virtual routers : 3
   Interface Vlan-interface10
     VRID             : 2                   Adver Timer  : 100
     Admin Status     : Up                  State        : Backup
     Config Pri       : 100                 Running Pri  : 100
     Preempt Mode     : Yes                 Delay Time   : 0
     Become Master    : 0ms left
     Auth Type        : Not supported
     Version          : 3
     Virtual IP       : 10.1.10.254
     Virtual MAC      : 0000-5e00-0102
     Master IP        : 10.1.10.252

   Interface Vlan-interface20
     VRID             : 3                   Adver Timer  : 100
     Admin Status     : Up                  State        : Backup
     Config Pri       : 100                 Running Pri  : 100
     Preempt Mode     : Yes                 Delay Time   : 0
     Become Master    : 0ms left
     Auth Type        : Not supported
     Version          : 3
     Virtual IP       : 10.1.20.254
     Virtual MAC      : 0000-5e00-0103
     Master IP        : 10.1.20.252

   Interface Vlan-interface99
     VRID             : 1                   Adver Timer  : 100
     Admin Status     : Up                  State        : Master
     Config Pri       : 200                 Running Pri  : 200
     Preempt Mode     : Yes                 Delay Time   : 0
     Auth Type        : Not supported
     Version          : 3
     Virtual IP       : 10.1.99.254
     Virtual MAC      : 0000-5e00-0101
     Master IP        : 10.1.99.251
```   


```   
[Reg1-DSW2-Vlan-interface99]disp vrrp verbose
IPv4 Virtual Router Information:
 Running mode : Standard
 Total number of virtual routers : 3
   Interface Vlan-interface10
     VRID             : 2                   Adver Timer  : 100
     Admin Status     : Up                  State        : Master
     Config Pri       : 200                 Running Pri  : 200
     Preempt Mode     : Yes                 Delay Time   : 0
     Auth Type        : Not supported
     Version          : 3
     Virtual IP       : 10.1.10.254
     Virtual MAC      : 0000-5e00-0102
     Master IP        : 10.1.10.252

   Interface Vlan-interface20
     VRID             : 3                   Adver Timer  : 100
     Admin Status     : Up                  State        : Master
     Config Pri       : 200                 Running Pri  : 200
     Preempt Mode     : Yes                 Delay Time   : 0
     Auth Type        : Not supported
     Version          : 3
     Virtual IP       : 10.1.20.254
     Virtual MAC      : 0000-5e00-0103
     Master IP        : 10.1.20.252

   Interface Vlan-interface99
     VRID             : 1                   Adver Timer  : 100
     Admin Status     : Up                  State        : Backup
     Config Pri       : 100                 Running Pri  : 100
     Preempt Mode     : Yes                 Delay Time   : 0
     Become Master    : 0ms left
     Auth Type        : Not supported
     Version          : 3
     Virtual IP       : 10.1.99.254
     Virtual MAC      : 0000-5e00-0101
     Master IP        : 10.1.99.251

```   
Видно, что Master для Vlan-interface99 переместился на Reg1-DSW1.

Настройка на маршрутизаторах (Регион 2) аналогична.


### Настройка Spanning Tree RSTP

Рассмотрим настройку RSTP (Rapid Spanning-tree Protocol) на примере Региона 2.

Для предотвращения образования петель поднимем на всех коммутаторах региона RSTP с помощью команды:
```   
stp mode rstp
stp global enable
```   

Посмотрим состояние портов на всех коммутаторах:

```   
[Reg2-DSW1]disp stp brief
 MST ID   Port                                Role  STP State   Protection
 0        FortyGigE1/0/53                     ROOT  FORWARDING  NONE
 0        FortyGigE1/0/54                     ALTE  DISCARDING  NONE
 0        Ten-GigabitEthernet1/0/49           DESI  FORWARDING  NONE
 0        Ten-GigabitEthernet1/0/50           DESI  FORWARDING  NONE

[Reg2-DSW2]disp stp br
 MST ID   Port                                Role  STP State   Protection
 0        FortyGigE1/0/53                     DESI  FORWARDING  NONE
 0        FortyGigE1/0/54                     DESI  FORWARDING  NONE
 0        Ten-GigabitEthernet1/0/49           DESI  FORWARDING  NONE
 0        Ten-GigabitEthernet1/0/50           DESI  FORWARDING  NONE

[Reg2-ASW1]disp stp br
 MST ID   Port                                Role  STP State   Protection
 0        GigabitEthernet1/0/1                DESI  FORWARDING  NONE
 0        Ten-GigabitEthernet1/0/49           ALTE  DISCARDING  NONE
 0        Ten-GigabitEthernet1/0/50           ROOT  FORWARDING  NONE

[Reg2-ASW2]disp stp br
 MST ID   Port                                Role  STP State   Protection
 0        GigabitEthernet1/0/1                DESI  FORWARDING  NONE
 0        Ten-GigabitEthernet1/0/49           ALTE  DISCARDING  NONE
 0        Ten-GigabitEthernet1/0/50           ROOT  FORWARDING  NONE
```   

Корневым мостом является Reg2-DSW2 (RootPort ID==0.0)

```   
[Reg2-DSW1]disp stp | include RootPort ID
 RootPort ID         : 128.54

[[Reg2-DSW2]disp stp | include RootPort ID
 RootPort ID         : 0.0

[Reg2-ASW1]disp stp | include RootPort
 RootPort ID         : 128.51

[Reg2-ASW2]disp stp | include RootPort
 RootPort ID         : 128.51
```   

Для наглядности DISCARDING порты отмечены на рисунке крестиком:




![Reg2-RSTP-default.jpg](./img/Reg2-RSTP-default.jpg)


### Объединение коммутаторов в стек

Технология стэкирования не стандартизована и у каждого вендора своя. У H3C она называется IRF (Intelligent Resilient Framework), у Huawei &mdash; iStack, у Cisco &mdash; Stack.

Объединим DSW-коммутаторы в Регионе 2 в стек.

Reg2-DSW1:
```   
# блокируем интерфейсы
int range fge1/0/53 fge1/0/54
 shut
quit

# добавляем физические порты в виртуальный порт irf-port
irf member 1 renumber 1

irf-port 1/1
port group int fge1/0/53
port group int fge1/0/54
int range fge1/0/53 fge1/0/54

# задаём приоритет == 32, чтобы данный физический коммутатор был первым в стеке
irf member 1 priority 32


undo shutdown

save

# Для активации перезагружаем коммутатор (одновременно с Reg2-DSW2)

irf-port-configuration active
quit
reboot
Y

```   

На Reg2-DSW2 настройка аналогична, но задаю "второй номер" в стеке:
```   
int range fge1/0/53 fge1/0/54
shut
quit

irf member 1 renumber 2

irf-port 1/2
port group int fge1/0/53
port group int fge1/0/54
int range fge1/0/53 fge1/0/54
undo shutdown

save

irf-port-configuration active
quit
reboot
Y            
```   
Коммутаторы проводят выборы мастера. Тот, которой не пройдет, автоматически перезапустится. После завершения перезапуска формируется IRF.

Поскольку первый коммутатор вручную настроен с приоритетом 32 (значение по умолчанию равно 1, чем больше значение, тем выше приоритет, а максимальное значение равно 32), он будет выбран в качестве основного, и DSW2 перезапустится. После перезагрузки настройка IRF будет завершена и системное имя объединяется в Reg2-DSW1..

Присвою имя стекированному коммутатору Reg2-DSW (уже без порядкового номера). На втором коммутаторе (при подключении к нему консолью) имя изменится автоматически.

```   
<Reg2-DSW1>
<Reg2-DSW>
```   

Т.о. у нас теперь один логичесий коммутатор DSW в Регионе 2.
Что видно в настройках:

```   
[Reg2-DSW]disp cur
#
 version 7.1.070, Alpha 7170
#
 sysname Reg2-DSW
#

...

irf-port 1/1
 port group interface FortyGigE1/0/53
 port group interface FortyGigE1/0/54
#
irf-port 2/2
 port group interface FortyGigE2/0/53
 port group interface FortyGigE2/0/54
#
...
```   
Видно, что порты на втором коммутаторе стека отображаются как порты виртуальной "второй интерфейсной карты" (**2**/0/XX), как если бы это был коммутатор с несколькими интерфейсными картами.

Посмотрим, что стало с RSTP:


```   
[Reg2-DSW]disp stp brief
 MST ID   Port                                Role  STP State   Protection
 0        Ten-GigabitEthernet1/0/49           DESI  FORWARDING  NONE
 0        Ten-GigabitEthernet1/0/50           DESI  FORWARDING  NONE
 0        Ten-GigabitEthernet2/0/49           DESI  FORWARDING  NONE
 0        Ten-GigabitEthernet2/0/50           DESI  FORWARDING  NONE

[Reg2-ASW1]disp stp brief
 MST ID   Port                                Role  STP State   Protection
 0        GigabitEthernet1/0/1                DESI  FORWARDING  NONE
 0        Ten-GigabitEthernet1/0/49           ROOT  FORWARDING  NONE
 0        Ten-GigabitEthernet1/0/50           ALTE  DISCARDING  NONE

[Reg2-ASW2]disp stp brief
 MST ID   Port                                Role  STP State   Protection
 0        GigabitEthernet1/0/1                DESI  FORWARDING  NONE
 0        Ten-GigabitEthernet1/0/49           ROOT  FORWARDING  NONE
 0        Ten-GigabitEthernet1/0/50           ALTE  DISCARDING  NONE
```   

Корневым мостом теперь является Reg2-DSW (RootPort ID==0.0), т.е. наш стекированный коммутатор.
```   
[Reg2-DSW]disp stp | include RootPort
 RootPort ID         : 0.0

[Reg2-ASW1]disp stp | include RootPort
 RootPort ID         : 128.50

[Reg2-ASW2]disp stp | include RootPort
 RootPort ID         : 128.50
```   

Далее настраиваю Bridge-Aggregation Groups в LACP.


Теперь нет DISCARDING портов:

```   
[Reg2-DSW]disp stp br
 MST ID   Port                                Role  STP State   Protection
 0        Bridge-Aggregation101               DESI  FORWARDING  NONE
 0        Bridge-Aggregation102               DESI  FORWARDING  NONE

[Reg2-ASW1]disp stp br
 MST ID   Port                                Role  STP State   Protection
 0        Bridge-Aggregation1                 ROOT  FORWARDING  NONE
 0        GigabitEthernet1/0/1                DESI  FORWARDING  NONE

[Reg2-ASW2]disp stp br
 MST ID   Port                                Role  STP State   Protection
 0        Bridge-Aggregation1                 ROOT  FORWARDING  NONE
 0        GigabitEthernet1/0/1                DESI  LEARNING    NONE
[Reg2-ASW2]disp stp br
 MST ID   Port                                Role  STP State   Protection
 0        Bridge-Aggregation1                 ROOT  FORWARDING  NONE
 0        GigabitEthernet1/0/1                DESI  FORWARDING  NONE

```
Т.е. построена схема без петель.

Т.о. получили отказоустойчивую схему L2 с резервирование 1+1 коммутатора уровня ядра/распределения и резервирование каналов 1+1 до коммутаторов уровня доступа.


### Проблемы стекирования

Смоделируем отказ межкоммутаторных линков, образующих стек:

```
[Reg2-DSW]int range fge1/0/53 fge1/0/54
[Reg2-DSW-if-range]shut
%Oct  4 20:16:23:282 2025 Reg2-DSW LLDP/6/LLDP_DELETE_NEIGHBOR: -Slot=2; Nearest bridge agent neighbor deleted on port FortyGigE2/0/53 (IfIndex 566), neighbor's chassis ID is 1ac2-93f6-1100, port ID is FortyGigE1/0/53.
%Oct  4 20:16:23:760 2025 Reg2-DSW IFNET/3/PHY_UPDOWN: Physical state on the interface FortyGigE1/0/53 changed to down.
%Oct  4 20:16:23:761 2025 Reg2-DSW IFNET/5/LINK_UPDOWN: Line protocol state on the interface FortyGigE1/0/53 changed to down.
%Oct  4 20:16:23:761 2025 Reg2-DSW IFNET/3/PHY_UPDOWN: Physical state on the interface FortyGigE2/0/53 changed to down.
%Oct  4 20:16:23:762 2025 Reg2-DSW IFNET/5/LINK_UPDOWN: Line protocol state on the interface FortyGigE2/0/53 changed to down.
%Oct  4 20:16:23:318 2025 Reg2-DSW LLDP/6/LLDP_DELETE_NEIGHBOR: -Slot=2; Nearest bridge agent neighbor deleted on port FortyGigE2/0/54 (IfIndex 567), neighbor's chassis ID is 1ac2-93f6-1100, port ID is FortyGigE1/0/54.
%Oct  4 20:16:23:789 2025 Reg2-DSW STM/3/STM_LINK_DOWN: IRF port 1 went down.
%Oct  4 20:16:23:790 2025 Reg2-DSW DEV/3/BOARD_REMOVED: Board was removed from slot 2, type is H3C S6850.
%Oct  4 20:16:23:800 2025 Reg2-DSW LAGG/6/LAGG_INACTIVE_NODEREMOVE: Member port XGE2/0/49 of aggregation group BAGG101 changed to the inactive state, because the card that hosts the port was absent.
%Oct  4 20:16:23:819 2025 Reg2-DSW LAGG/6/LAGG_INACTIVE_NODEREMOVE: Member port XGE2/0/50 of aggregation group BAGG102 changed to the inactive state, because the card that hosts the port was absent.
%Oct  4 20:16:23:845 2025 Reg2-DSW IFNET/3/PHY_UPDOWN: Physical state on the interface FortyGigE1/0/54 changed to down.
%Oct  4 20:16:23:848 2025 Reg2-DSW IFNET/5/LINK_UPDOWN: Line protocol state on the interface FortyGigE1/0/54 changed to down.
```

Стэк развалился, оба коммутатора считают себя главными, в сети два устройства с одинаковым именем.

Ниже диагностика с первого коммутатора стека:
```
<Reg2-DSW>disp irf
MemberID    Role    Priority  CPU-Mac         Description
 *+1        Master  32        1ac2-93f6-1104  ---
--------------------------------------------------
 * indicates the device is the master.
 + indicates the device through which the user logs in.

 The bridge MAC of the IRF is: 1ac2-93f6-1100
 Auto upgrade                : yes
 Mac persistent              : 6 min
 Domain ID                   : 0
<Reg2-DSW>disp irf link
Member 1
 IRF Port  Interface                             Status
 1         FortyGigE1/0/53                       DOWN
           FortyGigE1/0/54                       DOWN
 2         disable                               --
<Reg2-DSW>disp irf conf
<Reg2-DSW>disp irf configuration
 MemberID NewID    IRF-Port1                     IRF-Port2
 1        1        FortyGigE1/0/53               disable
                   FortyGigE1/0/54
```

И со второго:
```
<Reg2-DSW>disp irf
MemberID    Role    Priority  CPU-Mac         Description
 *+2        Master  1         1ac2-82c2-1004  ---
--------------------------------------------------
 * indicates the device is the master.
 + indicates the device through which the user logs in.

 The bridge MAC of the IRF is: 1ac2-93f6-1100
 Auto upgrade                : yes
 Mac persistent              : 6 min
 Domain ID                   : 0
<Reg2-DSW>disp irf link
Member 2
 IRF Port  Interface                             Status
 1         disable                               --
 2         FortyGigE2/0/53                       DOWN
           FortyGigE2/0/54                       DOWN
<Reg2-DSW>disp irf conf
<Reg2-DSW>disp irf configuration
 MemberID NewID    IRF-Port1                     IRF-Port2
 2        2        disable                       FortyGigE2/0/53
                                                 FortyGigE2/0/54
```


Также аварийные сообщения с коммутаторов уровня доступа:
```
[Reg2-ASW1]
%Oct  4 20:19:30:851 2025 Reg2-ASW1 LAGG/6/LAGG_INACTIVE_PHYSTATE: Member port XGE1/0/50 of aggregation group BAGG1 changed to the inactive state, because the physical state of the port is down.
%Oct  4 20:19:30:853 2025 Reg2-ASW1 IFNET/3/PHY_UPDOWN: Physical state on the interface Ten-GigabitEthernet1/0/50 changed to down.
%Oct  4 20:19:30:853 2025 Reg2-ASW1 IFNET/5/LINK_UPDOWN: Line protocol state on the interface Ten-GigabitEthernet1/0/50 changed to down.


[Reg2-ASW2]
%Oct  4 20:19:31:553 2025 Reg2-ASW2 LAGG/6/LAGG_INACTIVE_PHYSTATE: Member port XGE1/0/50 of aggregation group BAGG1 changed to the inactive state, because the physical state of the port is down.
%Oct  4 20:19:31:555 2025 Reg2-ASW2 IFNET/3/PHY_UPDOWN: Physical state on the interface Ten-GigabitEthernet1/0/50 changed to down.
%Oct  4 20:19:31:555 2025 Reg2-ASW2 IFNET/5/LINK_UPDOWN: Line protocol state on the interface Ten-GigabitEthernet1/0/50 changed to down.
```

Восстановим стек:

```
[Reg2-DSW-if-range]undo shut
%Oct  4 20:33:35:573 2025 Reg2-DSW LAGG/6/LAGG_LACP_RECEIVE_TIMEOUT: -Slot=2; LACPDU reception timed out on member port XGE2/0/50 in aggregation group BAGG102.
%Oct  4 20:33:37:113 2025 Reg2-DSW LLDP/6/LLDP_CREATE_NEIGHBOR: Nearest bridge agent neighbor created on port FortyGigE1/0/54 (IfIndex 55), neighbor's chassis ID is 1ac2-93f6-1100, port ID is FortyGigE2/0/54.
%Oct  4 20:33:37:124 2025 Reg2-DSW LAGG/6/LAGG_ACTIVE: Member port XGE2/0/50 of aggregation group BAGG102 changed to the active state.
%Oct  4 20:33:35:890 2025 Reg2-DSW LLDP/6/LLDP_CREATE_NEIGHBOR: -Slot=2; Nearest bridge agent neighbor created on port Ten-GigabitEthernet2/0/50 (IfIndex 563), neighbor's chassis ID is 1ac3-6812-1400, port ID is Ten-GigabitEthernet1/0/50.
%Oct  4 20:33:37:466 2025 Reg2-DSW IFNET/3/PHY_UPDOWN: Physical state on the interface FortyGigE2/0/53 changed to up.
%Oct  4 20:33:35:902 2025 Reg2-DSW LLDP/6/LLDP_CREATE_NEIGHBOR: -Slot=2; Nearest bridge agent neighbor created on port FortyGigE2/0/54 (IfIndex 567), neighbor's chassis ID is 1ac2-93f6-1100, port ID is FortyGigE1/0/54.
%Oct  4 20:33:37:471 2025 Reg2-DSW IFNET/5/LINK_UPDOWN: Line protocol state on the interface FortyGigE2/0/53 changed to up.
%Oct  4 20:33:37:472 2025 Reg2-DSW IFNET/3/PHY_UPDOWN: Physical state on the interface FortyGigE2/0/54 changed to up.
%Oct  4 20:33:37:476 2025 Reg2-DSW IFNET/5/LINK_UPDOWN: Line protocol state on the interface FortyGigE2/0/54 changed to up.
%Oct  4 20:33:37:478 2025 Reg2-DSW IFNET/3/PHY_UPDOWN: Physical state on the interface Ten-GigabitEthernet2/0/50 changed to up.
%Oct  4 20:33:37:480 2025 Reg2-DSW IFNET/5/LINK_UPDOWN: Line protocol state on the interface Ten-GigabitEthernet2/0/50 changed to up.
%Oct  4 20:33:37:671 2025 Reg2-DSW HA/5/HA_BATCHBACKUP_STARTED: Batch backup of standby board in slot 2 started.
%Oct  4 20:33:37:774 2025 Reg2-DSW IFNET/3/PHY_UPDOWN: Physical state on the interface Ten-GigabitEthernet2/0/49 changed to up.
%Oct  4 20:33:36:185 2025 Reg2-DSW LAGG/6/LAGG_LACP_RECEIVE_TIMEOUT: -Slot=2; LACPDU reception timed out on member port XGE2/0/49 in aggregation group BAGG101.
%Oct  4 20:33:36:191 2025 Reg2-DSW LLDP/6/LLDP_CREATE_NEIGHBOR: -Slot=2; Nearest bridge agent neighbor created on port Ten-GigabitEthernet2/0/49 (IfIndex 562), neighbor's chassis ID is 1ac3-5bae-1300, port ID is Ten-GigabitEthernet1/0/50.
%Oct  4 20:33:37:799 2025 Reg2-DSW LAGG/6/LAGG_ACTIVE: Member port XGE2/0/49 of aggregation group BAGG101 changed to the active state.
%Oct  4 20:33:37:814 2025 Reg2-DSW IFNET/5/LINK_UPDOWN: Line protocol state on the interface Ten-GigabitEthernet2/0/49 changed to up.
%Oct  4 20:33:38:675 2025 Reg2-DSW HA/5/HA_BATCHBACKUP_FINISHED: Batch backup of standby board in slot 2 has finished.
%Oct  4 20:33:57:258 2025 Reg2-DSW LLDP/6/LLDP_CREATE_NEIGHBOR: -Slot=2; Nearest bridge agent neighbor created on port FortyGigE2/0/53 (IfIndex 566), neighbor's chassis ID is 1ac2-93f6-1100, port ID is FortyGigE1/0/53.
```


Диагностика:

```
<Reg2-DSW>disp irf
MemberID    Role    Priority  CPU-Mac         Description
 *+1        Master  32        1ac2-93f6-1104  ---
   2        Standby 1         1ac2-82c2-1004  ---
--------------------------------------------------
 * indicates the device is the master.
 + indicates the device through which the user logs in.

 The bridge MAC of the IRF is: 1ac2-93f6-1100
 Auto upgrade                : yes
 Mac persistent              : 6 min
 Domain ID                   : 0
<Reg2-DSW>disp irf link
Member 1
 IRF Port  Interface                             Status
 1         FortyGigE1/0/53                       UP
           FortyGigE1/0/54                       UP
 2         disable                               --
Member 2
 IRF Port  Interface                             Status
 1         disable                               --
 2         FortyGigE2/0/53                       UP
           FortyGigE2/0/54                       UP

<Reg2-DSW>disp irf configuration
 MemberID NewID    IRF-Port1                     IRF-Port2
 1        1        FortyGigE1/0/53               disable
                   FortyGigE1/0/54
 2        2        disable                       FortyGigE2/0/53
                                                 FortyGigE2/0/54
```

На восстановление стека в лабораторных условиях ушло две минуты, что достаточно много по времени. Т.о. даже при случайной блокировке портов, например ошибка ввода/скрипта при проведении работ, возможна аварийная ситуации высокого приоритета.

Ниже рассмотрим, какие средства есть для защиты стека.

### Настройка BFD MAD

Разделение IRF происходит, когда фабрика IRF распадается на несколько фабрик IRF из-за сбоев в работе каналов IRF. Разделённые фабрики IRF работают с одним и тем же IP-адресом.
Такое разделение приводит к проблемам маршрутизации и пересылки в сети.

Для защиты стека IRF у Н3С предусмотрено четыре механизма Multi-active detection (MAD):
- LACP MAD &mdash; требуется третье устройство, подключенное к обоим коммутаторам;
- BFD MAD &mdash; на уровне L2 создаётся прямое подключение между коммутаторами;
- ARP MAD &mdash; через запрос Gratuitous ARP, с идентификатором активного коммутатора;	
- ND MAD &mdash; с помощью пакетов NS (протокол Neighbor Discovery IPv6).

У каждого из этих механизмов есть свои преимущества и недостатки. Так, у LACP MAD высокая скорость обнаружения, не требуется дополнительного линка, но требуется промежуточное устройство, работающее по LACP. У BFD MAD также высокая скорость обнаружения, но требуется дополнительный канал между устройствами фабрики IRF. У ARP MAD низкая скорость обнаружения, но не требуются дополнительные устройства и каналы. У ND MAD скорость обнаружения ниже, чем у LACP и BFD MAD, не требуются дополнительные устройства и каналы, но используются сценарии IPv6.

Реализуем BFD MAD. Для этого добавим дополнительный линк между коммутаторами (GE1/0/48 &mdash; GE2/0/48).
Схема представлена на рисунке:


![Reg2-BFD-MAD.jpg](./img/Reg2-BFD-MAD.jpg)

Настройка:

```   
vlan 999
 description FOR_BFD_MAD_ONLY
 port GE1/0/48 GE2/0/48
quit

int Vlan-interface999
 mad bfd enable
 mad ip address 192.168.99.1 24 member 1
 mad ip address 192.168.99.2 24 member 2
quit


# отключаем STP на портах BFD, т.к. spanning-tree и BFD MAD взаимоисключающие:

int GE1/0/48
 undo stp enable
quit

int GE2/0/48
 undo stp enable
quit

```   

Проверка:
```   
[Reg2-DSW]disp bfd session
 Total sessions: 1     Up sessions: 0     Init mode: Active

 IPv4 session working in control packet mode:

 LD/RD            SourceAddr      DestAddr        State  Holdtime    Interface
 32833/0          192.168.99.1    192.168.99.2    Down      /        Vlan999
[Reg2-DSW]disp mad verbose
Multi-active recovery state: No
Excluded ports (user-configured):
Excluded ports (system-configured):
  IRF physical interfaces:
    FortyGigE1/0/53
    FortyGigE1/0/54
    FortyGigE2/0/53
    FortyGigE2/0/54
  BFD MAD interfaces:
    Vlan-interface999
MAD ARP disabled.
MAD ND disabled.
MAD LACP disabled.
MAD BFD enabled interface: Vlan-interface999
  MAD status                 : Normal
  Member ID   MAD IP address       Neighbor   MAD status
  1           192.168.99.1/24      2          Normal
  2           192.168.99.2/24      1          Normal
```   

Отключаем порты стека.
IRF разделяется, MAD обнаруживает разделение IRF, отключает все сетевые порты на втором коммутаторе, второй коммутатор не работает, первый коммутатор работает. Статус сеанса BFD ненадолго изменится с "Down" на "Up", а затем снова изменится на "Down", так что статус сеанса BFD, который мы видим, всегда будет "Down". Статус в выводе команды **disp mad** изменится с "Normal" на "Faulty".

```   
[Reg2-DSW]disp bfd sess
 Total sessions: 1     Up sessions: 0     Init mode: Active

 IPv4 session working in control packet mode:

 LD/RD            SourceAddr      DestAddr        State  Holdtime    Interface
 32833/0          192.168.99.1    192.168.99.2    Down      /        Vlan999

[Reg2-DSW]disp mad verbose
Multi-active recovery state: No
Excluded ports (user-configured):
Excluded ports (system-configured):
  IRF physical interfaces:
    FortyGigE1/0/53
    FortyGigE1/0/54
  BFD MAD interfaces:
    Vlan-interface999
MAD ARP disabled.
MAD ND disabled.
MAD LACP disabled.
MAD BFD enabled interface: Vlan-interface999
  MAD status                 : Faulty
  Member ID   MAD IP address       Neighbor   MAD status
  1           192.168.99.1/24      2          Faulty

```   
Конфигурация доступна по ссылке: [ссылке](./cfg/Reg2-DSW-H3C-BFD-MAD.cfg)

### Настройка M-LAG

В Регионе 3 применим технологию объединения коммутатров M-LAG. Данная технология избавлена от
недостатком классического стекирования, т.к. control-plane разнесён между коммутаторами.

Схема представлена на рисунке.

![MLAG-VRRP.svg](./img/MLAG-VRRP.svg)


Настроим M-LAG со стороны Reg3-DSW1, также выполним настройку VRRP:

```   

m-lag system-mac 1-1-1
m-lag system-number 1
m-lag system-priority 123
m-lag keepalive ip destination 111.111.111.112 source 111.111.111.111

int ge1/0/48
 port link-mode route
 ip address 111.111.111.111 24
quit

m-lag mad exclude interface ge1/0/48

int bridge-aggregation 100
 link-aggregation mode dynamic
quit

int fge1/0/53
 port link-aggregation group 100
quit

int fge1/0/54
 port link-aggregation group 100
quit

int bridge-aggregation 100
 port m-lag peer-link 1
quit

int bridge-aggregation 101
 link-aggregation mode dynamic
 port m-lag group 1
quit

interface xge1/0/49
 port link-aggregation group 101
quit

int bridge-aggregation 102
 link-aggregation mode dynamic
 port m-lag group 2
quit

int xge1/0/50
 port link-aggregation group 102
quit

vlan 10
 description Radio
quit

vlan 20
 description Tech
quit

interface bridge-aggregation 101
 port link-type trunk
 port trunk permit vlan 10
quit

interface bridge-aggregation 102
 port link-type trunk
 port trunk permit vlan 20
quit

interface vlan-interface 10
 ip address 10.3.10.251 24
quit

interface vlan-interface 20
 ip address 10.3.20.251 24
quit

m-lag mad exclude interface vlan-interface 10
m-lag mad exclude interface vlan-interface 20

interface vlan-interface 10
 vrrp vrid 1 virtual-ip 10.3.10.254
 vrrp vrid 1 priority 200
quit

interface vlan-interface 20
 vrrp vrid 2 virtual-ip 10.3.20.254
 vrrp vrid 2 priority 200
quit
```   
Настройка на Reg3-DSW2 аналогична.

На коммутаторах уровня доступа (на примере Reg3-ASW1) настройки следующие:

```   
interface bridge-aggregation 101
 link-aggregation mode dynamic
quit

interface range xge1/0/49 to xge1/0/50
 port link-aggregation group 101
quit

# Create VLAN 10.

vlan 10
 description Radio
quit

interface bridge-aggregation 101
 port link-type trunk
 port trunk permit vlan 10
quit
```   

Проверим, что Reg3-DSW1 сформировал систему M-LAG с Reg3-DSW2:

```   
[Reg3-DSW1]display m-lag summary
Flags: A -- Aggregate interface down, B -- No peer M-LAG interface configured
       C -- Configuration consistency check failed

Peer-link interface: BAGG100
Peer-link interface state (cause): UP
Keepalive link state (cause): UP

                     M-LAG interface information
M-LAG IF    M-LAG group  Local state (cause)  Peer state  Remaining down time(s)
BAGG101     1            UP                   UP          -
BAGG102     2            UP                   UP          -
[Reg3-DSW1]display m-lag verbose
Flags: A -- Home_Gateway, B -- Neighbor_Gateway, C -- Other_Gateway,
       D -- PeerLink_Activity, E -- DRCP_Timeout, F -- Gateway_Sync,
       G -- Port_Sync, H -- Expired

Peer-link interface/Peer-link interface ID: BAGG100/1
State: UP
Cause: -
Local DRCP flags/Peer DRCP flags: ABDFG/ABDFG
Local Selected ports (index): FGE1/0/53 (54), FGE1/0/54 (55)
Peer Selected ports indexes: 54, 55
Reserved VLANs: -

M-LAG interface/M-LAG group ID: BAGG101/1
Local M-LAG interface state: UP
Peer M-LAG interface state: UP
M-LAG group state: UP
Local M-LAG interface down cause: -
Remaining M-LAG DOWN time: -
Local M-LAG interface LACP MAC: Config=N/A, Effective=0001-0001-0001
Peer M-LAG interface LACP MAC: Config=N/A, Effective=0001-0001-0001
Local M-LAG interface LACP priority: Config=32768, Effective=123
Peer M-LAG interface LACP priority: Config=32768, Effective=123
Local DRCP flags/Peer DRCP flags: ABDFG/ABDFG
Local Selected ports (index): XGE1/0/49 (50)
Peer Selected ports indexes: 50

M-LAG interface/M-LAG group ID: BAGG102/2
Local M-LAG interface state: UP
Peer M-LAG interface state: UP
M-LAG group state: UP
Local M-LAG interface down cause: -
Remaining M-LAG DOWN time: -
Local M-LAG interface LACP MAC: Config=N/A, Effective=0001-0001-0001
Peer M-LAG interface LACP MAC: Config=N/A, Effective=0001-0001-0001
Local M-LAG interface LACP priority: Config=32768, Effective=123
Peer M-LAG interface LACP priority: Config=32768, Effective=123
Local DRCP flags/Peer DRCP flags: ABDFG/ABDFG
Local Selected ports (index): XGE1/0/50 (51)
Peer Selected ports indexes: 51
```   

Проверяем, что Reg3-ASW1 и Reg3-ASW2 корректно сформировали агрегированные соединения с системой M-LAG:

```   
[Reg3-ASW1]display link-aggregation verbose
Loadsharing Type: Shar -- Loadsharing, NonS -- Non-Loadsharing
Port: A -- Auto
Port Status: S -- Selected, U -- Unselected, I -- Individual
Flags:  A -- LACP_Activity, B -- LACP_Timeout, C -- Aggregation,
        D -- Synchronization, E -- Collecting, F -- Distributing,
        G -- Defaulted, H -- Expired

Aggregate Interface: Bridge-Aggregation101
Aggregation Mode: Dynamic
Loadsharing Type: Shar
System ID: 0x8000, 268f-4241-1e00
Local:
  Port                Status  Priority Oper-Key  Flag
--------------------------------------------------------------------------------
  XGE1/0/49           S       32768    1         {ACDEF}
  XGE1/0/50           S       32768    2         {ACDEF}
Remote:
  Actor               Priority Index    Oper-Key SystemID               Flag
--------------------------------------------------------------------------------
  XGE1/0/49(R)        32768    16386    40001    0x7b  , 0001-0001-0001 {ACDEF}
  XGE1/0/50           32768    32770    40001    0x7b  , 0001-0001-0001 {ACDEF}




[Reg3-ASW2]display link-aggregation verbose
Loadsharing Type: Shar -- Loadsharing, NonS -- Non-Loadsharing
Port: A -- Auto
Port Status: S -- Selected, U -- Unselected, I -- Individual
Flags:  A -- LACP_Activity, B -- LACP_Timeout, C -- Aggregation,
        D -- Synchronization, E -- Collecting, F -- Distributing,
        G -- Defaulted, H -- Expired

Aggregate Interface: Bridge-Aggregation102
Aggregation Mode: Dynamic
Loadsharing Type: Shar
System ID: 0x8000, 26a5-e3d0-1f00
Local:
  Port                Status  Priority Oper-Key  Flag
--------------------------------------------------------------------------------
  XGE1/0/49           S       32768    1         {ACDEF}
  XGE1/0/50           S       32768    2         {ACDEF}
Remote:
  Actor               Priority Index    Oper-Key SystemID               Flag
--------------------------------------------------------------------------------
  XGE1/0/49(R)        32768    16387    40002    0x7b  , 0001-0001-0001 {ACDEF}
  XGE1/0/50           32768    32771    40002    0x7b  , 0001-0001-0001 {ACDEF}
```   
Конфигурационныe файлы можно найти по [ссылке](./cfg).

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


Ниже приведена настройка протокола OSPF на всех CE и DSW региона.
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

## Протоколы L3 транспортной сети

### Настройка IS-IS

В качестве протокола внутренней маршрутизации на транспортной сети оператора применим протокол IS-IS.

Выбор протокола обусловлен простотой настройки, масштабируемостью, гибкостью при построении сетей операторов, простотой расширения для IPv6.

Сконфигурируем IS-IS как level-L1 для взаимодействия в рамках одной общей зоны. Назначим NSAP-адреса для P/PE-маршрутизаторов согласно таблицы:
| Маршрутизатор| NSAP-адрес   |
| --------     | -------     |
| PE1          | 49.6500.0000.0000.0001.00 |
| PE2          | 49.6500.0000.0000.0002.00 |
| PE3          | 49.6500.0000.0000.0003.00 |
| PE4          | 49.6500.0000.0000.0004.00 |

Где 49 &mdash; указывает тип адреса (приватный), 6500 &mdash; номер зоны, 0000.0000.000**X** &mdash; ID устройства (у нас будет по номеру маршрутизатора), 00 - селектор (всегда ноль).

На интерфейсах настроим isis circuit-type p2p для оптимизации работы протокола.
Заданы is-name для передачи информации о hostname ISIS соседям в пределах общей зоны.
На интерфейсах подключения регионов настроим режим isis silent, чтобы не отправлять hello в клиентские порты.

Настройка на примере маршрутизатора PE3:
```   
isis 65000
 is-level level-1
 network-entity 49.6500.0000.0000.0003.00
 is-name PE3
quit

interface GE0/0/0
 isis enable 65000
 isis circuit-type p2p 
quit

interface GE0/0/1
 isis enable 65000
 isis circuit-type p2p 
quit

interface GE0/0/2
 isis enable 65000
 isis circuit-type p2p 
quit

interface GE0/0/7
 isis enable 65000
 isis silent
quit

interface GE0/0/8
 isis enable 65000
 isis silent
quit

int loopback0
  isis enable 65000
quit
```   

Для остальных маршрутизаторов настройки аналогичны.

Диагностика протокола на примере PE3 приведена ниже.
```   
[PE3]display ip routing-table

Destinations : 20       Routes : 23

Destination/Mask   Proto   Pre Cost        NextHop         Interface
1.1.1.1/32         IS_L1   15  10          10.0.0.2        GE0/0/1
2.2.2.2/32         IS_L1   15  10          10.0.0.7        GE0/0/2
3.3.3.3/32         Direct  0   0           127.0.0.1       Loop0
4.4.4.4/32         IS_L1   15  10          10.0.0.11       GE0/0/0
10.0.0.0/31        IS_L1   15  20          10.0.0.2        GE0/0/1
                   IS_L1   15  20          10.0.0.7        GE0/0/2
10.0.0.2/31        Direct  0   0           10.0.0.3        GE0/0/1
10.0.0.3/32        Direct  0   0           127.0.0.1       GE0/0/1
10.0.0.4/31        IS_L1   15  20          10.0.0.2        GE0/0/1
                   IS_L1   15  20          10.0.0.11       GE0/0/0
10.0.0.6/31        Direct  0   0           10.0.0.6        GE0/0/2
10.0.0.6/32        Direct  0   0           127.0.0.1       GE0/0/2
10.0.0.8/31        IS_L1   15  20          10.0.0.7        GE0/0/2
                   IS_L1   15  20          10.0.0.11       GE0/0/0
10.0.0.10/31       Direct  0   0           10.0.0.10       GE0/0/0
10.0.0.10/32       Direct  0   0           127.0.0.1       GE0/0/0
10.0.0.12/31       IS_L1   15  20          10.0.0.2        GE0/0/1
10.0.0.16/31       Direct  0   0           10.0.0.16       GE0/0/7
10.0.0.16/32       Direct  0   0           127.0.0.1       GE0/0/7
127.0.0.0/8        Direct  0   0           127.0.0.1       InLoop0
127.0.0.1/32       Direct  0   0           127.0.0.1       InLoop0
127.255.255.255/32 Direct  0   0           127.0.0.1       InLoop0
255.255.255.255/32 Direct  0   0           127.0.0.1       InLoop0
[PE3]
[PE3]display ip routing-table protocol isis

Summary count : 15

ISIS Routing table status : <Active>
Summary count : 10

Destination/Mask   Proto   Pre Cost        NextHop         Interface
1.1.1.1/32         IS_L1   15  10          10.0.0.2        GE0/0/1
2.2.2.2/32         IS_L1   15  10          10.0.0.7        GE0/0/2
4.4.4.4/32         IS_L1   15  10          10.0.0.11       GE0/0/0
10.0.0.0/31        IS_L1   15  20          10.0.0.2        GE0/0/1
                   IS_L1   15  20          10.0.0.7        GE0/0/2
10.0.0.4/31        IS_L1   15  20          10.0.0.2        GE0/0/1
                   IS_L1   15  20          10.0.0.11       GE0/0/0
10.0.0.8/31        IS_L1   15  20          10.0.0.7        GE0/0/2
                   IS_L1   15  20          10.0.0.11       GE0/0/0
10.0.0.12/31       IS_L1   15  20          10.0.0.2        GE0/0/1

ISIS Routing table status : <Inactive>
Summary count : 5

Destination/Mask   Proto   Pre Cost        NextHop         Interface
3.3.3.3/32         IS_L1   15  0           0.0.0.0         Loop0
10.0.0.2/31        IS_L1   15  10          0.0.0.0         GE0/0/1
10.0.0.6/31        IS_L1   15  10          0.0.0.0         GE0/0/2
10.0.0.10/31       IS_L1   15  10          0.0.0.0         GE0/0/0
10.0.0.16/31       IS_L1   15  10          0.0.0.0         GE0/0/7
[PE3]
[PE3]display isis peer

                       Peer information for IS-IS(65000)
                       ---------------------------------

 System ID: PE4
 Interface: GE0/0/0                 Circuit Id:  001
 State: Up     HoldTime: 22s        Type: L1           PRI: --

 System ID: PE1
 Interface: GE0/0/1                 Circuit Id:  001
 State: Up     HoldTime: 25s        Type: L1           PRI: --

 System ID: PE2
 Interface: GE0/0/2                 Circuit Id:  001
 State: Up     HoldTime: 26s        Type: L1           PRI: --
```   

Проверка сетевой связности (пинги по Loopback-ам):

С маршрутизатора PE1:
```  
[PE1]ping 2.2.2.2
Ping 2.2.2.2 (2.2.2.2): 56 data bytes, press CTRL+C to break
56 bytes from 2.2.2.2: icmp_seq=0 ttl=255 time=3.257 ms
56 bytes from 2.2.2.2: icmp_seq=1 ttl=255 time=0.472 ms
56 bytes from 2.2.2.2: icmp_seq=2 ttl=255 time=0.620 ms
56 bytes from 2.2.2.2: icmp_seq=3 ttl=255 time=0.759 ms
56 bytes from 2.2.2.2: icmp_seq=4 ttl=255 time=0.632 ms

--- Ping statistics for 2.2.2.2 ---
5 packet(s) transmitted, 5 packet(s) received, 0.0% packet loss
round-trip min/avg/max/std-dev = 0.472/1.148/3.257/1.058 ms

[PE1]ping 3.3.3.3
Ping 3.3.3.3 (3.3.3.3): 56 data bytes, press CTRL+C to break
56 bytes from 3.3.3.3: icmp_seq=0 ttl=255 time=1.462 ms
56 bytes from 3.3.3.3: icmp_seq=1 ttl=255 time=0.690 ms
56 bytes from 3.3.3.3: icmp_seq=2 ttl=255 time=0.629 ms
56 bytes from 3.3.3.3: icmp_seq=3 ttl=255 time=0.508 ms
56 bytes from 3.3.3.3: icmp_seq=4 ttl=255 time=0.960 ms

--- Ping statistics for 3.3.3.3 ---
5 packet(s) transmitted, 5 packet(s) received, 0.0% packet loss
round-trip min/avg/max/std-dev = 0.508/0.850/1.462/0.340 ms

[PE1]ping 4.4.4.4
Ping 4.4.4.4 (4.4.4.4): 56 data bytes, press CTRL+C to break
56 bytes from 4.4.4.4: icmp_seq=0 ttl=255 time=1.013 ms
56 bytes from 4.4.4.4: icmp_seq=1 ttl=255 time=0.487 ms
56 bytes from 4.4.4.4: icmp_seq=2 ttl=255 time=0.481 ms
56 bytes from 4.4.4.4: icmp_seq=3 ttl=255 time=0.631 ms
56 bytes from 4.4.4.4: icmp_seq=4 ttl=255 time=1.778 ms

--- Ping statistics for 4.4.4.4 ---
5 packet(s) transmitted, 5 packet(s) received, 0.0% packet loss
round-trip min/avg/max/std-dev = 0.481/0.878/1.778/0.490 ms

[PE1]
```  

И с маршрутизатора PE3:
```   
[PE3]ping 1.1.1.1
Ping 1.1.1.1 (1.1.1.1): 56 data bytes, press CTRL+C to break
56 bytes from 1.1.1.1: icmp_seq=0 ttl=255 time=1.593 ms
56 bytes from 1.1.1.1: icmp_seq=1 ttl=255 time=0.457 ms
56 bytes from 1.1.1.1: icmp_seq=2 ttl=255 time=0.539 ms
56 bytes from 1.1.1.1: icmp_seq=3 ttl=255 time=0.459 ms
56 bytes from 1.1.1.1: icmp_seq=4 ttl=255 time=0.776 ms

--- Ping statistics for 1.1.1.1 ---
5 packet(s) transmitted, 5 packet(s) received, 0.0% packet loss
round-trip min/avg/max/std-dev = 0.457/0.765/1.593/0.430 ms

[PE3]ping 2.2.2.2
Ping 2.2.2.2 (2.2.2.2): 56 data bytes, press CTRL+C to break
56 bytes from 2.2.2.2: icmp_seq=0 ttl=255 time=2.649 ms
56 bytes from 2.2.2.2: icmp_seq=1 ttl=255 time=0.742 ms
56 bytes from 2.2.2.2: icmp_seq=2 ttl=255 time=1.204 ms
56 bytes from 2.2.2.2: icmp_seq=3 ttl=255 time=0.581 ms
56 bytes from 2.2.2.2: icmp_seq=4 ttl=255 time=0.698 ms

--- Ping statistics for 2.2.2.2 ---
5 packet(s) transmitted, 5 packet(s) received, 0.0% packet loss
round-trip min/avg/max/std-dev = 0.581/1.175/2.649/0.767 ms

[PE3]
[PE3]ping 4.4.4.4
Ping 4.4.4.4 (4.4.4.4): 56 data bytes, press CTRL+C to break
56 bytes from 4.4.4.4: icmp_seq=0 ttl=255 time=2.417 ms
56 bytes from 4.4.4.4: icmp_seq=1 ttl=255 time=0.462 ms
56 bytes from 4.4.4.4: icmp_seq=2 ttl=255 time=0.639 ms
56 bytes from 4.4.4.4: icmp_seq=3 ttl=255 time=0.733 ms
56 bytes from 4.4.4.4: icmp_seq=4 ttl=255 time=0.511 ms

--- Ping statistics for 4.4.4.4 ---
5 packet(s) transmitted, 5 packet(s) received, 0.0% packet loss
round-trip min/avg/max/std-dev = 0.462/0.952/2.417/0.738 ms

```   

Конфигурационныe файлы можно найти по [ссылке](./cfg).

### Настройка eBGP

Настроим eBGP стыки между регионами и транспортной сетью.

Настройка PE-маршрутизаторов транспортной сети.

PE1:
```   
bgp 65000
 router-id 1.1.1.1
 peer 10.0.0.13 as-number 65001
 peer 10.0.0.13 description Reg1-R1
  address-family ipv4 unicast
  peer 10.0.0.13 enable
  network 1.1.1.1 32
  network 10.0.0.12 31
 quit
quit
```   

PE2:
```   
bgp 65000
 router-id 2.2.2.2
 peer 10.0.0.15 as-number 65002
 peer 10.0.0.15 description Reg2-R1
  address-family ipv4 unicast
  peer 10.0.0.15 enable
  network 2.2.2.2 32
  network 10.0.0.14 31
 quit
quit
```
   
PE3:
```   
bgp 65000
 router-id 3.3.3.3
 peer 10.0.0.17 as-number 65001
 peer 10.0.0.17 description Reg1-R2
 peer 10.0.0.21 as-number 65003
 peer 10.0.0.21 description Reg3-R1
 address-family ipv4 unicast
  peer 10.0.0.17 enable
  peer 10.0.0.21 enable
  network 3.3.3.3 32
  network 10.0.0.16 31
  network 10.0.0.20 31
 quit
quit
```   

PE4:
```   
bgp 65000
 router-id 4.4.4.4
 peer 10.0.0.19 as-number 65002
 peer 10.0.0.19 description Reg2-R2
 peer 10.0.0.23 as-number 65003
 peer 10.0.0.23 description Reg3-R2
 address-family ipv4 unicast
  peer 10.0.0.19 enable
  peer 10.0.0.23 enable
  network 4.4.4.4 32
  network 10.0.0.18 31
  network 10.0.0.22 31
 quit
quit
```   

Настройка CE-маршрутизаторов на примере региона 1.


Reg1-R1:
```   
bgp 65001
 router-id 11.11.11.11
 peer 10.0.0.12 as-number 65000
 peer 10.0.0.12 description PE1
  address-family ipv4 unicast
  peer 10.0.0.12 enable
  network 11.11.11.11 32
  network 10.0.0.12 31
 quit
quit
```   

Reg1-R2:
```   
bgp 65001
 router-id 12.12.12.12
 peer 10.0.0.16 as-number 65000
 peer 10.0.0.16 description PE3
  address-family ipv4 unicast
  peer 10.0.0.16 enable
  network 12.12.12.12 32
  network 10.0.0.16 31
 quit
quit
```   


#### Диагностика

PE1:
```   
[PE1]display ip routing-table

Destinations : 25       Routes : 28

Destination/Mask   Proto   Pre Cost        NextHop         Interface
1.1.1.1/32         Direct  0   0           127.0.0.1       Loop0
2.2.2.2/32         IS_L1   15  10          10.0.0.1        GE0/0/0
3.3.3.3/32         IS_L1   15  10          10.0.0.3        GE0/0/1
4.4.4.4/32         IS_L1   15  10          10.0.0.5        GE0/0/2
10.0.0.0/31        Direct  0   0           10.0.0.0        GE0/0/0
10.0.0.0/32        Direct  0   0           127.0.0.1       GE0/0/0
10.0.0.2/31        Direct  0   0           10.0.0.2        GE0/0/1
10.0.0.2/32        Direct  0   0           127.0.0.1       GE0/0/1
10.0.0.4/31        Direct  0   0           10.0.0.4        GE0/0/2
10.0.0.4/32        Direct  0   0           127.0.0.1       GE0/0/2
10.0.0.6/31        IS_L1   15  20          10.0.0.1        GE0/0/0
                   IS_L1   15  20          10.0.0.3        GE0/0/1
10.0.0.8/31        IS_L1   15  20          10.0.0.1        GE0/0/0
                   IS_L1   15  20          10.0.0.5        GE0/0/2
10.0.0.10/31       IS_L1   15  20          10.0.0.3        GE0/0/1
                   IS_L1   15  20          10.0.0.5        GE0/0/2
10.0.0.12/31       Direct  0   0           10.0.0.12       GE0/0/7
10.0.0.12/32       Direct  0   0           127.0.0.1       GE0/0/7
10.0.0.14/31       IS_L1   15  20          10.0.0.1        GE0/0/0
10.0.0.16/31       IS_L1   15  20          10.0.0.3        GE0/0/1
10.0.0.18/31       IS_L1   15  20          10.0.0.5        GE0/0/2
10.0.0.20/31       IS_L1   15  20          10.0.0.3        GE0/0/1
10.0.0.22/31       IS_L1   15  20          10.0.0.5        GE0/0/2
11.11.11.11/32     BGP     255 0           10.0.0.13       GE0/0/7
127.0.0.0/8        Direct  0   0           127.0.0.1       InLoop0
127.0.0.1/32       Direct  0   0           127.0.0.1       InLoop0
127.255.255.255/32 Direct  0   0           127.0.0.1       InLoop0
255.255.255.255/32 Direct  0   0           127.0.0.1       InLoop0
[PE1]
[PE1]display ip routing-table protocol bgp

Summary count : 1

BGP Routing table status : <Active>
Summary count : 1

Destination/Mask   Proto   Pre Cost        NextHop         Interface
11.11.11.11/32     BGP     255 0           10.0.0.13       GE0/0/7

BGP Routing table status : <Inactive>
Summary count : 0
[PE1]disp bgp peer ipv4

 BGP local router ID: 1.1.1.1
 Local AS number: 65000
 Total number of peers: 1                 Peers in established state: 1

 * - Dynamically created peer
 Peer                    AS  MsgRcvd  MsgSent OutQ  PrefRcv Up/Down  State

 10.0.0.13            65001       21       22    0        2 00:14:17 Established
[PE1]disp bgp peer ipv4 verbose

        Peer: 10.0.0.13 Local: 1.1.1.1
        Type: EBGP link
        Peer's description: "Reg1-R1"
        BGP version 4, remote router ID 11.11.11.11
        Update group ID: 0
        BGP current state: Established, Up for 00h14m22s
        BGP current event: RecvKeepalive
        BGP last state: OpenConfirm
        Port:  Local - 179      Remote - 52160
        Configured: Active Hold Time: 180 sec   Keepalive Time: 60 sec
        Received  : Active Hold Time: 180 sec
        Negotiated: Active Hold Time: 180 sec   Keepalive Time: 60 sec
        Peer optional capabilities:
        Peer supports BGP multi-protocol extension
        Peer supports BGP route refresh capability
        Peer supports BGP route AS4 capability
        Address family IPv4 Unicast: advertised and received

 InQ updates: 0, OutQ updates: 0
 NLRI statistics:
        Rcvd:   UnReach NLRI          0,      Reach NLRI          2
        Sent:   UnReach NLRI          0,      Reach NLRI          2

 Message statistics:
 Msg type     Last rcvd time/      Current rcvd count/      History rcvd count/
              Last sent time       Current sent count       History sent count
 Open         20:15:45-2025.10.14  1                        1
              20:15:45-2025.10.14  1                        1
 Update       20:15:46-2025.10.14  4                        4
              20:15:45-2025.10.14  3                        3
 Notification -                    0                        0
              -                    0                        0
 Keepalive    20:29:48-2025.10.14  16                       16
              20:29:31-2025.10.14  18                       18
 RouteRefresh -                    0                        0
              -                    0                        0
 Total        -                    21                       21
              -                    22                       22

 Maximum allowed prefix number: 4294967295
 Threshold: 75%
 Authentication type configured: None
 Minimum time between advertisements is 30 seconds
 Optional capabilities:
  Multi-protocol extended capability has been enabled
  Route refresh capability has been enabled
  Extended nexthop encoding has been enabled
 Peer preferred value: 0
 Site-of-Origin: Not specified
 Routing policy configured:
 No routing policy is configured
[PE1]
```   

PE4:
```   
[PE4]display ip routing-table

Destinations : 27       Routes : 30

Destination/Mask   Proto   Pre Cost        NextHop         Interface
1.1.1.1/32         IS_L1   15  10          10.0.0.4        GE0/0/2
2.2.2.2/32         IS_L1   15  10          10.0.0.8        GE0/0/1
3.3.3.3/32         IS_L1   15  10          10.0.0.10       GE0/0/0
4.4.4.4/32         Direct  0   0           127.0.0.1       Loop0
10.0.0.0/31        IS_L1   15  20          10.0.0.4        GE0/0/2
                   IS_L1   15  20          10.0.0.8        GE0/0/1
10.0.0.2/31        IS_L1   15  20          10.0.0.4        GE0/0/2
                   IS_L1   15  20          10.0.0.10       GE0/0/0
10.0.0.4/31        Direct  0   0           10.0.0.5        GE0/0/2
10.0.0.5/32        Direct  0   0           127.0.0.1       GE0/0/2
10.0.0.6/31        IS_L1   15  20          10.0.0.8        GE0/0/1
                   IS_L1   15  20          10.0.0.10       GE0/0/0
10.0.0.8/31        Direct  0   0           10.0.0.9        GE0/0/1
10.0.0.9/32        Direct  0   0           127.0.0.1       GE0/0/1
10.0.0.10/31       Direct  0   0           10.0.0.11       GE0/0/0
10.0.0.11/32       Direct  0   0           127.0.0.1       GE0/0/0
10.0.0.12/31       IS_L1   15  20          10.0.0.4        GE0/0/2
10.0.0.14/31       IS_L1   15  20          10.0.0.8        GE0/0/1
10.0.0.16/31       IS_L1   15  20          10.0.0.10       GE0/0/0
10.0.0.18/31       Direct  0   0           10.0.0.18       GE0/0/7
10.0.0.18/32       Direct  0   0           127.0.0.1       GE0/0/7
10.0.0.20/31       IS_L1   15  20          10.0.0.10       GE0/0/0
10.0.0.22/31       Direct  0   0           10.0.0.22       GE0/0/8
10.0.0.22/32       Direct  0   0           127.0.0.1       GE0/0/8
22.22.22.22/32     BGP     255 0           10.0.0.19       GE0/0/7
32.32.32.32/32     BGP     255 0           10.0.0.23       GE0/0/8
127.0.0.0/8        Direct  0   0           127.0.0.1       InLoop0
127.0.0.1/32       Direct  0   0           127.0.0.1       InLoop0
127.255.255.255/32 Direct  0   0           127.0.0.1       InLoop0
255.255.255.255/32 Direct  0   0           127.0.0.1       InLoop0
[PE4]
[PE4]display ip routing-table protocol bgp

Summary count : 2

BGP Routing table status : <Active>
Summary count : 2

Destination/Mask   Proto   Pre Cost        NextHop         Interface
22.22.22.22/32     BGP     255 0           10.0.0.19       GE0/0/7
32.32.32.32/32     BGP     255 0           10.0.0.23       GE0/0/8

BGP Routing table status : <Inactive>
Summary count : 0
[PE4]
[PE4]disp bgp peer ipv4

 BGP local router ID: 4.4.4.4
 Local AS number: 65000
 Total number of peers: 2                 Peers in established state: 2

 * - Dynamically created peer
 Peer                    AS  MsgRcvd  MsgSent OutQ  PrefRcv Up/Down  State

 10.0.0.19            65002       13       16    0        2 00:07:52 Established
 10.0.0.23            65003       11       18    0        2 00:05:28 Established
[PE4]
[PE4]disp bgp peer ipv4 verbose

        Peer: 10.0.0.19 Local: 4.4.4.4
        Type: EBGP link
        Peer's description: "Reg2-R2"
        BGP version 4, remote router ID 22.22.22.22
        Update group ID: 0
        BGP current state: Established, Up for 00h07m56s
        BGP current event: KATimerExpired
        BGP last state: OpenConfirm
        Port:  Local - 179      Remote - 19584
        Configured: Active Hold Time: 180 sec   Keepalive Time: 60 sec
        Received  : Active Hold Time: 180 sec
        Negotiated: Active Hold Time: 180 sec   Keepalive Time: 60 sec
        Peer optional capabilities:
        Peer supports BGP multi-protocol extension
        Peer supports BGP route refresh capability
        Peer supports BGP route AS4 capability
        Address family IPv4 Unicast: advertised and received

 InQ updates: 0, OutQ updates: 0
 NLRI statistics:
        Rcvd:   UnReach NLRI          0,      Reach NLRI          2
        Sent:   UnReach NLRI          0,      Reach NLRI          4

 Message statistics:
 Msg type     Last rcvd time/      Current rcvd count/      History rcvd count/
              Last sent time       Current sent count       History sent count
 Open         20:23:28-2025.10.14  1                        1
              20:23:28-2025.10.14  1                        1
 Update       20:23:29-2025.10.14  3                        3
              20:25:51-2025.10.14  5                        5
 Notification -                    0                        0
              -                    0                        0
 Keepalive    20:30:41-2025.10.14  9                        9
              20:31:02-2025.10.14  10                       10
 RouteRefresh -                    0                        0
              -                    0                        0
 Total        -                    13                       13
              -                    16                       16

 Maximum allowed prefix number: 4294967295
 Threshold: 75%
 Authentication type configured: None
 Minimum time between advertisements is 30 seconds
 Optional capabilities:
  Multi-protocol extended capability has been enabled
  Route refresh capability has been enabled
  Extended nexthop encoding has been enabled
 Peer preferred value: 0
 Site-of-Origin: Not specified
 Routing policy configured:
 No routing policy is configured

        Peer: 10.0.0.23 Local: 4.4.4.4
        Type: EBGP link
        Peer's description: "Reg3-R2"
        BGP version 4, remote router ID 32.32.32.32
        Update group ID: 0
        BGP current state: Established, Up for 00h05m32s
        BGP current event: RecvKeepalive
        BGP last state: OpenConfirm
        Port:  Local - 179      Remote - 26432
        Configured: Active Hold Time: 180 sec   Keepalive Time: 60 sec
        Received  : Active Hold Time: 180 sec
        Negotiated: Active Hold Time: 180 sec   Keepalive Time: 60 sec
        Peer optional capabilities:
        Peer supports BGP multi-protocol extension
        Peer supports BGP route refresh capability
        Peer supports BGP route AS4 capability
        Address family IPv4 Unicast: advertised and received

 InQ updates: 0, OutQ updates: 0
 NLRI statistics:
        Rcvd:   UnReach NLRI          0,      Reach NLRI          2
        Sent:   UnReach NLRI          0,      Reach NLRI         20

 Message statistics:
 Msg type     Last rcvd time/      Current rcvd count/      History rcvd count/
              Last sent time       Current sent count       History sent count
 Open         20:25:51-2025.10.14  1                        1
              20:25:51-2025.10.14  1                        1
 Update       20:25:52-2025.10.14  3                        3
              20:25:57-2025.10.14  9                        9
 Notification -                    0                        0
              -                    0                        0
 Keepalive    20:31:18-2025.10.14  7                        7
              20:31:11-2025.10.14  8                        8
 RouteRefresh -                    0                        0
              -                    0                        0
 Total        -                    11                       11
              -                    18                       18

 Maximum allowed prefix number: 4294967295
 Threshold: 75%
 Authentication type configured: None
 Minimum time between advertisements is 30 seconds
 Optional capabilities:
  Multi-protocol extended capability has been enabled
  Route refresh capability has been enabled
  Extended nexthop encoding has been enabled
 Peer preferred value: 0
 Site-of-Origin: Not specified
 Routing policy configured:
 No routing policy is configured
[PE4]
```   


Reg1-R1:
```   
[Reg1-R1]display ip routing-table

Destinations : 16       Routes : 17

Destination/Mask   Proto   Pre Cost        NextHop         Interface
0.0.0.0/0          Static  60  0           10.0.0.12       GE0/0
                   Static  60  0           10.0.0.16       GE0/1
0.0.0.0/32         Direct  0   0           127.0.0.1       InLoop0
1.1.1.1/32         BGP     255 0           10.0.0.12       GE0/0
10.0.0.12/31       Direct  0   0           10.0.0.13       GE0/0
10.0.0.13/32       Direct  0   0           127.0.0.1       InLoop0
10.0.0.16/31       O_ASE2  150 1           10.1.0.1        GE0/1
10.1.0.0/31        Direct  0   0           10.1.0.0        GE0/1
10.1.0.0/32        Direct  0   0           127.0.0.1       InLoop0
11.11.11.11/32     Direct  0   0           127.0.0.1       InLoop0
12.12.12.12/32     O_INTRA 10  1           10.1.0.1        GE0/1
127.0.0.0/8        Direct  0   0           127.0.0.1       InLoop0
127.0.0.1/32       Direct  0   0           127.0.0.1       InLoop0
127.255.255.255/32 Direct  0   0           127.0.0.1       InLoop0
224.0.0.0/4        Direct  0   0           0.0.0.0         NULL0
224.0.0.0/24       Direct  0   0           0.0.0.0         NULL0
255.255.255.255/32 Direct  0   0           127.0.0.1       InLoop0
[Reg1-R1]
[Reg1-R1]display ip routing-table protocol bgp

Summary count : 1

BGP Routing table status : <Active>
Summary count : 1

Destination/Mask   Proto   Pre Cost        NextHop         Interface
1.1.1.1/32         BGP     255 0           10.0.0.12       GE0/0

BGP Routing table status : <Inactive>
Summary count : 0
[Reg1-R1]
[Reg1-R1]disp bgp peer ipv4

 BGP local router ID: 11.11.11.11
 Local AS number: 65001
 Total number of peers: 1                 Peers in established state: 1

  * - Dynamically created peer
  Peer                    AS  MsgRcvd  MsgSent OutQ PrefRcv Up/Down  State

  10.0.0.12            65000       25       22    0       2 00:15:57 Established
[Reg1-R1]
[Reg1-R1]disp bgp peer ipv4 verbose

        Peer: 10.0.0.12 Local: 11.11.11.11
        Type: EBGP link
        Peer's description: "PE1"
        BGP version 4, remote router ID 1.1.1.1
        BGP current state: Established, Up for 00h16m01s
        BGP current event: RecvKeepalive
        BGP last state: OpenConfirm
        Port:  Local - 52160    Remote - 179
        Configured: Active Hold Time: 180 sec   Keepalive Time: 60 sec
        Received  : Active Hold Time: 180 sec
        Negotiated: Active Hold Time: 180 sec   Keepalive Time: 60 sec
        Peer optional capabilities:
        Peer supports BGP multi-protocol extension
        Peer supports BGP route refresh capability
        Peer supports BGP route AS4 capability
        Address family IPv4 Unicast: advertised and received

 InQ updates: 0, OutQ updates: 0
 NLRI statistics:
        Rcvd:   UnReach NLRI          0,      Reach NLRI          2
        Sent:   UnReach NLRI          0,      Reach NLRI          2

 Message statistics:
 Msg type     Last rcvd time/      Current rcvd count/      History rcvd count/
              Last sent time       Current sent count       History sent count
 Open         20:13:32-2025.10.14  1                        1
              20:13:32-2025.10.14  1                        1
 Update       20:13:33-2025.10.14  3                        3
              20:13:33-2025.10.14  3                        3
 Notification -                    0                        0
              -                    0                        0
 Keepalive    20:29:07-2025.10.14  21                       21
              20:28:51-2025.10.14  18                       18
 RouteRefresh -                    0                        0
              -                    0                        0
 Total        -                    25                       25
              -                    22                       22

 Maximum allowed prefix number: 4294967295
 Threshold: 75%
 Minimum time between advertisements is 30 seconds
 Optional capabilities:
  Multi-protocol extended capability has been enabled
  Route refresh capability has been enabled
 Peer preferred value: 0
 Site-of-Origin: Not specified
 Routing policy configured:
 No routing policy is configured
[Reg1-R1]
```   


<details>
<summary> Reg3-R2 </summary>

```   
[Reg3-R2]display ip routing-table

Destinations : 18       Routes : 18

Destination/Mask   Proto   Pre Cost        NextHop         Interface
0.0.0.0/0          Static  60  0           10.0.0.22       GE0/0
0.0.0.0/32         Direct  0   0           127.0.0.1       InLoop0
4.4.4.4/32         BGP     255 0           10.0.0.22       GE0/0
10.0.0.18/31       BGP     255 0           10.0.0.22       GE0/0
10.0.0.20/31       O_ASE2  150 1           10.3.0.0        GE0/1
10.0.0.22/31       Direct  0   0           10.0.0.23       GE0/0
10.0.0.23/32       Direct  0   0           127.0.0.1       InLoop0
10.3.0.0/31        Direct  0   0           10.3.0.1        GE0/1
10.3.0.1/32        Direct  0   0           127.0.0.1       InLoop0
22.22.22.22/32     BGP     255 0           10.0.0.22       GE0/0
31.31.31.31/32     O_INTRA 10  1           10.3.0.0        GE0/1
32.32.32.32/32     Direct  0   0           127.0.0.1       InLoop0
127.0.0.0/8        Direct  0   0           127.0.0.1       InLoop0
127.0.0.1/32       Direct  0   0           127.0.0.1       InLoop0
127.255.255.255/32 Direct  0   0           127.0.0.1       InLoop0
224.0.0.0/4        Direct  0   0           0.0.0.0         NULL0
224.0.0.0/24       Direct  0   0           0.0.0.0         NULL0
255.255.255.255/32 Direct  0   0           127.0.0.1       InLoop0
[Reg3-R2]
[Reg3-R2]display ip routing-table protocol bgp

Summary count : 3

BGP Routing table status : <Active>
Summary count : 3

Destination/Mask   Proto   Pre Cost        NextHop         Interface
4.4.4.4/32         BGP     255 0           10.0.0.22       GE0/0
10.0.0.18/31       BGP     255 0           10.0.0.22       GE0/0
22.22.22.22/32     BGP     255 0           10.0.0.22       GE0/0

BGP Routing table status : <Inactive>
Summary count : 0
[Reg3-R2]disp bgp peer ipv4

 BGP local router ID: 32.32.32.32
 Local AS number: 65003
 Total number of peers: 1                 Peers in established state: 1

  * - Dynamically created peer
  Peer                    AS  MsgRcvd  MsgSent OutQ PrefRcv Up/Down  State

  10.0.0.22            65000       21       13    0       4 00:08:05 Established
[Reg3-R2]
[Reg3-R2]disp bgp peer ipv4 verbose

        Peer: 10.0.0.22 Local: 32.32.32.32
        Type: EBGP link
        Peer's description: "PE4"
        BGP version 4, remote router ID 4.4.4.4
        BGP current state: Established, Up for 00h08m10s
        BGP current event: KATimerExpired
        BGP last state: OpenConfirm
        Port:  Local - 26432    Remote - 179
        Configured: Active Hold Time: 180 sec   Keepalive Time: 60 sec
        Received  : Active Hold Time: 180 sec
        Negotiated: Active Hold Time: 180 sec   Keepalive Time: 60 sec
        Peer optional capabilities:
        Peer supports BGP multi-protocol extension
        Peer supports BGP route refresh capability
        Peer supports BGP route AS4 capability
        Address family IPv4 Unicast: advertised and received

 InQ updates: 0, OutQ updates: 0
 NLRI statistics:
        Rcvd:   UnReach NLRI          0,      Reach NLRI          8
        Sent:   UnReach NLRI          0,      Reach NLRI          2

 Message statistics:
 Msg type     Last rcvd time/      Current rcvd count/      History rcvd count/
              Last sent time       Current sent count       History sent count
 Open         20:25:41-2025.10.14  1                        1
              20:25:41-2025.10.14  1                        1
 Update       20:25:46-2025.10.14  9                        9
              20:25:42-2025.10.14  3                        3
 Notification -                    0                        0
              -                    0                        0
 Keepalive    20:33:15-2025.10.14  11                       11
              20:33:47-2025.10.14  10                       10
 RouteRefresh -                    0                        0
              -                    0                        0
 Total        -                    21                       21
              -                    14                       14

 Maximum allowed prefix number: 4294967295
 Threshold: 75%
 Minimum time between advertisements is 30 seconds
 Optional capabilities:
  Multi-protocol extended capability has been enabled
  Route refresh capability has been enabled
 Peer preferred value: 0
 Site-of-Origin: Not specified
 Routing policy configured:
 No routing policy is configured
[Reg3-R2]

```   
</details>


Проверка связности (с Reg1-R1 все PE):
```   
[Reg1-R1]ping 1.1.1.1
Ping 1.1.1.1 (1.1.1.1): 56 data bytes, press CTRL+C to break
56 bytes from 1.1.1.1: icmp_seq=0 ttl=255 time=1.000 ms
56 bytes from 1.1.1.1: icmp_seq=1 ttl=255 time=1.000 ms
56 bytes from 1.1.1.1: icmp_seq=2 ttl=255 time=1.000 ms
56 bytes from 1.1.1.1: icmp_seq=3 ttl=255 time=0.000 ms
56 bytes from 1.1.1.1: icmp_seq=4 ttl=255 time=0.000 ms

--- Ping statistics for 1.1.1.1 ---
5 packet(s) transmitted, 5 packet(s) received, 0.0% packet loss
round-trip min/avg/max/std-dev = 0.000/0.600/1.000/0.490 ms

[Reg1-R1]ping 2.2.2.2
Ping 2.2.2.2 (2.2.2.2): 56 data bytes, press CTRL+C to break
56 bytes from 2.2.2.2: icmp_seq=0 ttl=254 time=2.000 ms
56 bytes from 2.2.2.2: icmp_seq=1 ttl=254 time=1.000 ms
56 bytes from 2.2.2.2: icmp_seq=2 ttl=254 time=2.000 ms
56 bytes from 2.2.2.2: icmp_seq=3 ttl=254 time=1.000 ms
56 bytes from 2.2.2.2: icmp_seq=4 ttl=254 time=1.000 ms

--- Ping statistics for 2.2.2.2 ---
5 packet(s) transmitted, 5 packet(s) received, 0.0% packet loss
round-trip min/avg/max/std-dev = 1.000/1.400/2.000/0.490 ms

[Reg1-R1]
[Reg1-R1]ping 3.3.3.3
Ping 3.3.3.3 (3.3.3.3): 56 data bytes, press CTRL+C to break
56 bytes from 3.3.3.3: icmp_seq=0 ttl=254 time=2.000 ms
56 bytes from 3.3.3.3: icmp_seq=1 ttl=254 time=1.000 ms
56 bytes from 3.3.3.3: icmp_seq=2 ttl=254 time=2.000 ms
56 bytes from 3.3.3.3: icmp_seq=3 ttl=254 time=1.000 ms
56 bytes from 3.3.3.3: icmp_seq=4 ttl=254 time=0.000 ms

--- Ping statistics for 3.3.3.3 ---
5 packet(s) transmitted, 5 packet(s) received, 0.0% packet loss
round-trip min/avg/max/std-dev = 0.000/1.200/2.000/0.748 ms

[Reg1-R1]
[Reg1-R1]ping 4.4.4.4
Ping 4.4.4.4 (4.4.4.4): 56 data bytes, press CTRL+C to break
56 bytes from 4.4.4.4: icmp_seq=0 ttl=254 time=2.000 ms
56 bytes from 4.4.4.4: icmp_seq=1 ttl=254 time=1.000 ms
56 bytes from 4.4.4.4: icmp_seq=2 ttl=254 time=1.000 ms
56 bytes from 4.4.4.4: icmp_seq=3 ttl=254 time=2.000 ms
56 bytes from 4.4.4.4: icmp_seq=4 ttl=254 time=1.000 ms

--- Ping statistics for 4.4.4.4 ---
5 packet(s) transmitted, 5 packet(s) received, 0.0% packet loss
round-trip min/avg/max/std-dev = 1.000/1.400/2.000/0.490 ms
```   


Reg1-R1 &rarr; Reg2-R1:
```   
[Reg1-R1]ping 10.0.0.15
Ping 10.0.0.15 (10.0.0.15): 56 data bytes, press CTRL+C to break
56 bytes from 10.0.0.15: icmp_seq=0 ttl=253 time=2.000 ms
56 bytes from 10.0.0.15: icmp_seq=1 ttl=253 time=2.000 ms
56 bytes from 10.0.0.15: icmp_seq=2 ttl=253 time=1.000 ms
56 bytes from 10.0.0.15: icmp_seq=3 ttl=253 time=2.000 ms
56 bytes from 10.0.0.15: icmp_seq=4 ttl=253 time=2.000 ms

--- Ping statistics for 10.0.0.15 ---
5 packet(s) transmitted, 5 packet(s) received, 0.0% packet loss
round-trip min/avg/max/std-dev = 1.000/1.800/2.000/0.400 ms
```   

Reg1-R1 &rarr; Reg2-R2:
```   
[Reg1-R1]ping 10.0.0.19
Ping 10.0.0.19 (10.0.0.19): 56 data bytes, press CTRL+C to break
56 bytes from 10.0.0.19: icmp_seq=0 ttl=253 time=5.000 ms
56 bytes from 10.0.0.19: icmp_seq=1 ttl=253 time=1.000 ms
56 bytes from 10.0.0.19: icmp_seq=2 ttl=253 time=2.000 ms
56 bytes from 10.0.0.19: icmp_seq=3 ttl=253 time=2.000 ms
56 bytes from 10.0.0.19: icmp_seq=4 ttl=253 time=3.000 ms

--- Ping statistics for 10.0.0.19 ---
5 packet(s) transmitted, 5 packet(s) received, 0.0% packet loss
round-trip min/avg/max/std-dev = 1.000/2.600/5.000/1.356 ms
```   

Reg1-R1 &rarr; Reg3-R1:
```   
[Reg1-R1]ping 10.0.0.21
Ping 10.0.0.21 (10.0.0.21): 56 data bytes, press CTRL+C to break
56 bytes from 10.0.0.21: icmp_seq=0 ttl=253 time=2.000 ms
56 bytes from 10.0.0.21: icmp_seq=1 ttl=253 time=2.000 ms
56 bytes from 10.0.0.21: icmp_seq=2 ttl=253 time=2.000 ms
56 bytes from 10.0.0.21: icmp_seq=3 ttl=253 time=2.000 ms
56 bytes from 10.0.0.21: icmp_seq=4 ttl=253 time=2.000 ms

--- Ping statistics for 10.0.0.21 ---
5 packet(s) transmitted, 5 packet(s) received, 0.0% packet loss
round-trip min/avg/max/std-dev = 2.000/2.000/2.000/0.000 ms
```   

Reg1-R1 &rarr; Reg3-R2:
```   
[Reg1-R1]ping 10.0.0.23
Ping 10.0.0.23 (10.0.0.23): 56 data bytes, press CTRL+C to break
56 bytes from 10.0.0.23: icmp_seq=0 ttl=253 time=2.000 ms
56 bytes from 10.0.0.23: icmp_seq=1 ttl=253 time=2.000 ms
56 bytes from 10.0.0.23: icmp_seq=2 ttl=253 time=1.000 ms
56 bytes from 10.0.0.23: icmp_seq=3 ttl=253 time=2.000 ms
56 bytes from 10.0.0.23: icmp_seq=4 ttl=253 time=3.000 ms

--- Ping statistics for 10.0.0.23 ---
5 packet(s) transmitted, 5 packet(s) received, 0.0% packet loss
round-trip min/avg/max/std-dev = 1.000/2.000/3.000/0.632 ms
```   

Конфигурационныe файлы можно найти по [ссылке](./cfg).	
### Настройка iBGP

Настроим iBGP на транспортной сети.
В качестве Route Reflector-а выберем PE1. Соседства установим "по лупбэкам", для удобства.


<details>
<summary> PE1 </summary>

```   
bgp 65000
 peer 2.2.2.2 as-number 65000
 peer 2.2.2.2 description Route_Reflector_Clients
 peer 2.2.2.2 connect-interface LoopBack0
 peer 3.3.3.3 as-number 65000
 peer 3.3.3.3 description Route_Reflector_Clients
 peer 3.3.3.3 connect-interface LoopBack0
 peer 4.4.4.4 as-number 65000
 peer 4.4.4.4 description Route_Reflector_Clients
 peer 4.4.4.4 connect-interface LoopBack0
 address-family ipv4 unicast
  network 10.0.0.12 255.255.255.254
  peer 2.2.2.2 enable
  peer 2.2.2.2 reflect-client
  peer 3.3.3.3 enable
  peer 3.3.3.3 reflect-client
  peer 4.4.4.4 enable
  peer 4.4.4.4 reflect-client
 quit
quit
```   
</details>

Настройка на PE2&ndash;PE4 одинаковая.

<details>
<summary> PE2&ndash;PE4 </summary>

```   
bgp 65000
 peer 1.1.1.1 as-number 65000
 peer 1.1.1.1 connect-interface LoopBack0
 address-family ipv4 unicast
  peer 1.1.1.1 enable
  peer 1.1.1.1 next-hop-local
 quit
quit
```   
</details>


#### Диагностика

До настройки с помощью команды **display bgp routing-table ipv4**  посмотрели таблицу маршрутизации BGP. Видно, что присутствуют только записи eBGP.

<details>
<summary> До настройки iBGP </summary>

```   
<PE1>display bgp routing-table ipv4

 Total number of routes: 7

 BGP local router ID is 1.1.1.1
 Status codes: * - valid, > - best, d - dampened, h - history,
               s - suppressed, S - stale, i - internal, e - external
               a - additional-path
       Origin: i - IGP, e - EGP, ? - incomplete

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >  1.1.1.1/32         127.0.0.1       0                     32768   i
* >  10.0.0.12/31       10.0.0.12       0                     32768   i
*  e                    10.0.0.13       0                     0       65001i
* >e 11.11.11.11/32     10.0.0.13       0                     0       65001i


[PE2]display bgp routing-table ipv4

 Total number of routes: 7

 BGP local router ID is 2.2.2.2
 Status codes: * - valid, > - best, d - dampened, h - history,
               s - suppressed, S - stale, i - internal, e - external
               a - additional-path
       Origin: i - IGP, e - EGP, ? - incomplete

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >  2.2.2.2/32         127.0.0.1       0                     32768   i
* >  10.0.0.14/31       10.0.0.14       0                     32768   i
*  e                    10.0.0.15       0                     0       65002i
* >e 21.21.21.21/32     10.0.0.15       0                     0       65002i
[PE2]


<PE3>display bgp routing-table ipv4

 Total number of routes: 7

 BGP local router ID is 3.3.3.3
 Status codes: * - valid, > - best, d - dampened, h - history,
               s - suppressed, S - stale, i - internal, e - external
               a - additional-path
       Origin: i - IGP, e - EGP, ? - incomplete

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >  3.3.3.3/32         127.0.0.1       0                     32768   i
* >  10.0.0.16/31       10.0.0.16       0                     32768   i
*  e                    10.0.0.17       0                     0       65001i
* >  10.0.0.20/31       10.0.0.20       0                     32768   i
*  e                    10.0.0.21       0                     0       65003i
* >e 12.12.12.12/32     10.0.0.17       0                     0       65001i
* >e 31.31.31.31/32     10.0.0.21       0                     0       65003i


<PE4>display bgp routing-table ipv4

 Total number of routes: 7

 BGP local router ID is 4.4.4.4
 Status codes: * - valid, > - best, d - dampened, h - history,
               s - suppressed, S - stale, i - internal, e - external
               a - additional-path
       Origin: i - IGP, e - EGP, ? - incomplete

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >  4.4.4.4/32         127.0.0.1       0                     32768   i
* >  10.0.0.18/31       10.0.0.18       0                     32768   i
*  e                    10.0.0.19       0                     0       65002i
* >  10.0.0.22/31       10.0.0.22       0                     32768   i
*  e                    10.0.0.23       0                     0       65003i
* >e 22.22.22.22/32     10.0.0.19       0                     0       65002i
* >e 32.32.32.32/32     10.0.0.23       0                     0       65003i
```   
</details>


После настройки появились записи iBGP.
<details>
<summary> После настройки iBGP </summary>

```   
[PE1]display bgp routing-table ipv4

 Total number of routes: 17

 BGP local router ID is 1.1.1.1
 Status codes: * - valid, > - best, d - dampened, h - history,
               s - suppressed, S - stale, i - internal, e - external
               a - additional-path
       Origin: i - IGP, e - EGP, ? - incomplete

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >  1.1.1.1/32         127.0.0.1       0                     32768   i
* >i 2.2.2.2/32         2.2.2.2         0          100        0       i
* >i 3.3.3.3/32         3.3.3.3         0          100        0       i
* >i 4.4.4.4/32         4.4.4.4         0          100        0       i
* >  10.0.0.12/31       10.0.0.12       0                     32768   i
*  e                    10.0.0.13       0                     0       65001i
* >i 10.0.0.14/31       2.2.2.2         0          100        0       i
* >i 10.0.0.16/31       3.3.3.3         0          100        0       i
* >i 10.0.0.18/31       4.4.4.4         0          100        0       i
* >i 10.0.0.20/31       3.3.3.3         0          100        0       i
* >i 10.0.0.22/31       4.4.4.4         0          100        0       i
* >e 11.11.11.11/32     10.0.0.13       0                     0       65001i
* >i 12.12.12.12/32     3.3.3.3         0          100        0       65001i
* >i 21.21.21.21/32     2.2.2.2         0          100        0       65002i
* >i 22.22.22.22/32     4.4.4.4         0          100        0       65002i
* >i 31.31.31.31/32     3.3.3.3         0          100        0       65003i
* >i 32.32.32.32/32     4.4.4.4         0          100        0       65003i


[PE2]display bgp routing-table ipv4

 Total number of routes: 7

 BGP local router ID is 2.2.2.2
 Status codes: * - valid, > - best, d - dampened, h - history,
               s - suppressed, S - stale, i - internal, e - external
               a - additional-path
       Origin: i - IGP, e - EGP, ? - incomplete

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >i 1.1.1.1/32         1.1.1.1         0          100        0       i
* >  2.2.2.2/32         127.0.0.1       0                     32768   i
* >i 10.0.0.12/31       1.1.1.1         0          100        0       i
* >  10.0.0.14/31       10.0.0.14       0                     32768   i
*  e                    10.0.0.15       0                     0       65002i
* >i 11.11.11.11/32     10.0.0.13       0          100        0       65001i
* >e 21.21.21.21/32     10.0.0.15       0                     0       65002i
[PE2]

[PE3]display bgp routing-table ipv4

 Total number of routes: 18

 BGP local router ID is 3.3.3.3
 Status codes: * - valid, > - best, d - dampened, h - history,
               s - suppressed, S - stale, i - internal, e - external
               a - additional-path
       Origin: i - IGP, e - EGP, ? - incomplete

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >i 1.1.1.1/32         1.1.1.1         0          100        0       i
* >i 2.2.2.2/32         2.2.2.2         0          100        0       i
* >  3.3.3.3/32         127.0.0.1       0                     32768   i
* >i 4.4.4.4/32         4.4.4.4         0          100        0       i
* >i 10.0.0.12/31       1.1.1.1         0          100        0       i
* >i 10.0.0.14/31       2.2.2.2         0          100        0       i
* >  10.0.0.16/31       10.0.0.16       0                     32768   i
*  e                    10.0.0.17       0                     0       65001i
* >i 10.0.0.18/31       4.4.4.4         0          100        0       i
* >  10.0.0.20/31       10.0.0.20       0                     32768   i
*  e                    10.0.0.21       0                     0       65003i
* >i 10.0.0.22/31       4.4.4.4         0          100        0       i
* >i 11.11.11.11/32     10.0.0.13       0          100        0       65001i
* >e 12.12.12.12/32     10.0.0.17       0                     0       65001i
* >i 21.21.21.21/32     2.2.2.2         0          100        0       65002i
* >i 22.22.22.22/32     4.4.4.4         0          100        0       65002i
* >e 31.31.31.31/32     10.0.0.21       0                     0       65003i
* >i 32.32.32.32/32     4.4.4.4         0          100        0       65003i

[PE4]display bgp routing-table ipv4

 Total number of routes: 18

 BGP local router ID is 4.4.4.4
 Status codes: * - valid, > - best, d - dampened, h - history,
               s - suppressed, S - stale, i - internal, e - external
               a - additional-path
       Origin: i - IGP, e - EGP, ? - incomplete

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >i 1.1.1.1/32         1.1.1.1         0          100        0       i
* >i 2.2.2.2/32         2.2.2.2         0          100        0       i
* >i 3.3.3.3/32         3.3.3.3         0          100        0       i
* >  4.4.4.4/32         127.0.0.1       0                     32768   i
* >i 10.0.0.12/31       1.1.1.1         0          100        0       i
* >i 10.0.0.14/31       2.2.2.2         0          100        0       i
* >i 10.0.0.16/31       3.3.3.3         0          100        0       i
* >  10.0.0.18/31       10.0.0.18       0                     32768   i
*  e                    10.0.0.19       0                     0       65002i
* >i 10.0.0.20/31       3.3.3.3         0          100        0       i
* >  10.0.0.22/31       10.0.0.22       0                     32768   i
*  e                    10.0.0.23       0                     0       65003i
* >i 11.11.11.11/32     10.0.0.13       0          100        0       65001i
* >i 12.12.12.12/32     3.3.3.3         0          100        0       65001i
* >i 21.21.21.21/32     2.2.2.2         0          100        0       65002i
* >e 22.22.22.22/32     10.0.0.19       0                     0       65002i
* >i 31.31.31.31/32     3.3.3.3         0          100        0       65003i
* >e 32.32.32.32/32     10.0.0.23       0                     0       65003i
```   
</details>

Ниже приведена остальная диагностика.
<details>
<summary> PE1 </summary>

```   
[PE1]display ip routing-table

Destinations : 30       Routes : 33

Destination/Mask   Proto   Pre Cost        NextHop         Interface
1.1.1.1/32         Direct  0   0           127.0.0.1       Loop0
2.2.2.2/32         IS_L1   15  10          10.0.0.1        GE0/0/0
3.3.3.3/32         IS_L1   15  10          10.0.0.3        GE0/0/1
4.4.4.4/32         IS_L1   15  10          10.0.0.5        GE0/0/2
10.0.0.0/31        Direct  0   0           10.0.0.0        GE0/0/0
10.0.0.0/32        Direct  0   0           127.0.0.1       GE0/0/0
10.0.0.2/31        Direct  0   0           10.0.0.2        GE0/0/1
10.0.0.2/32        Direct  0   0           127.0.0.1       GE0/0/1
10.0.0.4/31        Direct  0   0           10.0.0.4        GE0/0/2
10.0.0.4/32        Direct  0   0           127.0.0.1       GE0/0/2
10.0.0.6/31        IS_L1   15  20          10.0.0.1        GE0/0/0
                   IS_L1   15  20          10.0.0.3        GE0/0/1
10.0.0.8/31        IS_L1   15  20          10.0.0.1        GE0/0/0
                   IS_L1   15  20          10.0.0.5        GE0/0/2
10.0.0.10/31       IS_L1   15  20          10.0.0.3        GE0/0/1
                   IS_L1   15  20          10.0.0.5        GE0/0/2
10.0.0.12/31       Direct  0   0           10.0.0.12       GE0/0/7
10.0.0.12/32       Direct  0   0           127.0.0.1       GE0/0/7
10.0.0.14/31       IS_L1   15  20          10.0.0.1        GE0/0/0
10.0.0.16/31       IS_L1   15  20          10.0.0.3        GE0/0/1
10.0.0.18/31       IS_L1   15  20          10.0.0.5        GE0/0/2
10.0.0.20/31       IS_L1   15  20          10.0.0.3        GE0/0/1
10.0.0.22/31       IS_L1   15  20          10.0.0.5        GE0/0/2
11.11.11.11/32     BGP     255 0           10.0.0.13       GE0/0/7
12.12.12.12/32     BGP     255 0           3.3.3.3         GE0/0/1
21.21.21.21/32     BGP     255 0           2.2.2.2         GE0/0/0
22.22.22.22/32     BGP     255 0           4.4.4.4         GE0/0/2
31.31.31.31/32     BGP     255 0           3.3.3.3         GE0/0/1
32.32.32.32/32     BGP     255 0           4.4.4.4         GE0/0/2
127.0.0.0/8        Direct  0   0           127.0.0.1       InLoop0
127.0.0.1/32       Direct  0   0           127.0.0.1       InLoop0
127.255.255.255/32 Direct  0   0           127.0.0.1       InLoop0
255.255.255.255/32 Direct  0   0           127.0.0.1       InLoop0
[PE1]
[PE1]display ip routing-table protocol bgp

Summary count : 14

BGP Routing table status : <Active>
Summary count : 6

Destination/Mask   Proto   Pre Cost        NextHop         Interface
11.11.11.11/32     BGP     255 0           10.0.0.13       GE0/0/7
12.12.12.12/32     BGP     255 0           3.3.3.3         GE0/0/1
21.21.21.21/32     BGP     255 0           2.2.2.2         GE0/0/0
22.22.22.22/32     BGP     255 0           4.4.4.4         GE0/0/2
31.31.31.31/32     BGP     255 0           3.3.3.3         GE0/0/1
32.32.32.32/32     BGP     255 0           4.4.4.4         GE0/0/2

BGP Routing table status : <Inactive>
Summary count : 8

Destination/Mask   Proto   Pre Cost        NextHop         Interface
2.2.2.2/32         BGP     255 0           2.2.2.2         GE0/0/0
3.3.3.3/32         BGP     255 0           3.3.3.3         GE0/0/1
4.4.4.4/32         BGP     255 0           4.4.4.4         GE0/0/2
10.0.0.14/31       BGP     255 0           2.2.2.2         GE0/0/0
10.0.0.16/31       BGP     255 0           3.3.3.3         GE0/0/1
10.0.0.18/31       BGP     255 0           4.4.4.4         GE0/0/2
10.0.0.20/31       BGP     255 0           3.3.3.3         GE0/0/1
10.0.0.22/31       BGP     255 0           4.4.4.4         GE0/0/2
[PE1]
[PE1]disp bgp peer ipv4

 BGP local router ID: 1.1.1.1
 Local AS number: 65000
 Total number of peers: 4                 Peers in established state: 4

 * - Dynamically created peer
 Peer                    AS  MsgRcvd  MsgSent OutQ  PrefRcv Up/Down  State

 2.2.2.2              65000       18       28    0        3 00:13:53 Established
 3.3.3.3              65000       15       21    0        5 00:08:16 Established
 4.4.4.4              65000       15       21    0        5 00:08:08 Established
 10.0.0.13            65001      987      998    0        2 12:39:05 Established
[PE1]
[PE1]disp bgp peer ipv4 verbose

        Peer: 2.2.2.2   Local: 1.1.1.1
        Type: IBGP link
        Peer's description: "Route_Reflector_Clients"
        BGP version 4, remote router ID 2.2.2.2
        Update group ID: 1
        BGP current state: Established, Up for 00h13m57s
        BGP current event: KATimerExpired
        BGP last state: OpenConfirm
        Port:  Local - 179      Remote - 2753
        Configured: Active Hold Time: 180 sec   Keepalive Time: 60 sec
        Received  : Active Hold Time: 180 sec
        Negotiated: Active Hold Time: 180 sec   Keepalive Time: 60 sec
        Peer optional capabilities:
        Peer supports BGP multi-protocol extension
        Peer supports BGP route refresh capability
        Peer supports BGP route AS4 capability
        Address family IPv4 Unicast: advertised and received

 InQ updates: 0, OutQ updates: 0
 NLRI statistics:
        Rcvd:   UnReach NLRI          0,      Reach NLRI          3
        Sent:   UnReach NLRI          0,      Reach NLRI         13

 Message statistics:
 Msg type     Last rcvd time/      Current rcvd count/      History rcvd count/
              Last sent time       Current sent count       History sent count
 Open         19:10:52-2025.10.15  1                        1
              19:10:52-2025.10.15  1                        1
 Update       19:10:53-2025.10.15  3                        3
              19:16:37-2025.10.15  10                       10
 Notification -                    0                        0
              -                    0                        0
 Keepalive    19:24:45-2025.10.15  15                       15
              19:24:48-2025.10.15  18                       18
 RouteRefresh -                    0                        0
              -                    0                        0
 Total        -                    19                       19
              -                    29                       29

 Maximum allowed prefix number: 4294967295
 Threshold: 75%
 Authentication type configured: None
 Minimum time between advertisements is 15 seconds
 Optional capabilities:
  Multi-protocol extended capability has been enabled
  Route refresh capability has been enabled
  Extended nexthop encoding has been enabled
 It's route-reflector-client
 Connect-interface has been configured
 Peer preferred value: 0
 Site-of-Origin: Not specified
 Routing policy configured:
 No routing policy is configured

        Peer: 3.3.3.3   Local: 1.1.1.1
        Type: IBGP link
        Peer's description: "Route_Reflector_Clients"
        BGP version 4, remote router ID 3.3.3.3
        Update group ID: 1
        BGP current state: Established, Up for 00h08m20s
        BGP current event: KATimerExpired
        BGP last state: OpenConfirm
        Port:  Local - 179      Remote - 19522
        Configured: Active Hold Time: 180 sec   Keepalive Time: 60 sec
        Received  : Active Hold Time: 180 sec
        Negotiated: Active Hold Time: 180 sec   Keepalive Time: 60 sec
        Peer optional capabilities:
        Peer supports BGP multi-protocol extension
        Peer supports BGP route refresh capability
        Peer supports BGP route AS4 capability
        Address family IPv4 Unicast: advertised and received

 InQ updates: 0, OutQ updates: 0
 NLRI statistics:
        Rcvd:   UnReach NLRI          0,      Reach NLRI          5
        Sent:   UnReach NLRI          0,      Reach NLRI         18

 Message statistics:
 Msg type     Last rcvd time/      Current rcvd count/      History rcvd count/
              Last sent time       Current sent count       History sent count
 Open         19:16:29-2025.10.15  1                        1
              19:16:29-2025.10.15  1                        1
 Update       19:16:30-2025.10.15  4                        4
              19:16:37-2025.10.15  9                        9
 Notification -                    0                        0
              -                    0                        0
 Keepalive    19:24:19-2025.10.15  10                       10
              19:24:22-2025.10.15  11                       11
 RouteRefresh -                    0                        0
              -                    0                        0
 Total        -                    15                       15
              -                    21                       21

 Maximum allowed prefix number: 4294967295
 Threshold: 75%
 Authentication type configured: None
 Minimum time between advertisements is 15 seconds
 Optional capabilities:
  Multi-protocol extended capability has been enabled
  Route refresh capability has been enabled
  Extended nexthop encoding has been enabled
 It's route-reflector-client
 Connect-interface has been configured
 Peer preferred value: 0
 Site-of-Origin: Not specified
 Routing policy configured:
 No routing policy is configured

        Peer: 4.4.4.4   Local: 1.1.1.1
        Type: IBGP link
        Peer's description: "Route_Reflector_Clients"
        BGP version 4, remote router ID 4.4.4.4
        Update group ID: 1
        BGP current state: Established, Up for 00h08m12s
        BGP current event: RecvKeepalive
        BGP last state: OpenConfirm
        Port:  Local - 179      Remote - 3266
        Configured: Active Hold Time: 180 sec   Keepalive Time: 60 sec
        Received  : Active Hold Time: 180 sec
        Negotiated: Active Hold Time: 180 sec   Keepalive Time: 60 sec
        Peer optional capabilities:
        Peer supports BGP multi-protocol extension
        Peer supports BGP route refresh capability
        Peer supports BGP route AS4 capability
        Address family IPv4 Unicast: advertised and received

 InQ updates: 0, OutQ updates: 0
 NLRI statistics:
        Rcvd:   UnReach NLRI          0,      Reach NLRI          5
        Sent:   UnReach NLRI          0,      Reach NLRI         70

 Message statistics:
 Msg type     Last rcvd time/      Current rcvd count/      History rcvd count/
              Last sent time       Current sent count       History sent count
 Open         19:16:37-2025.10.15  1                        1
              19:16:37-2025.10.15  1                        1
 Update       19:16:38-2025.10.15  4                        4
              19:16:43-2025.10.15  9                        9
 Notification -                    0                        0
              -                    0                        0
 Keepalive    19:24:30-2025.10.15  10                       10
              19:24:14-2025.10.15  11                       11
 RouteRefresh -                    0                        0
              -                    0                        0
 Total        -                    15                       15
              -                    21                       21

 Maximum allowed prefix number: 4294967295
 Threshold: 75%
 Authentication type configured: None
 Minimum time between advertisements is 15 seconds
 Optional capabilities:
  Multi-protocol extended capability has been enabled
  Route refresh capability has been enabled
  Extended nexthop encoding has been enabled
 It's route-reflector-client
 Connect-interface has been configured
 Peer preferred value: 0
 Site-of-Origin: Not specified
 Routing policy configured:
 No routing policy is configured

        Peer: 10.0.0.13 Local: 1.1.1.1
        Type: EBGP link
        Peer's description: "Reg1-R1"
        BGP version 4, remote router ID 11.11.11.11
        Update group ID: 0
        BGP current state: Established, Up for 12h39m09s
        BGP current event: RecvKeepalive
        BGP last state: OpenConfirm
        Port:  Local - 1088     Remote - 179
        Configured: Active Hold Time: 180 sec   Keepalive Time: 60 sec
        Received  : Active Hold Time: 180 sec
        Negotiated: Active Hold Time: 180 sec   Keepalive Time: 60 sec
        Peer optional capabilities:
        Peer supports BGP multi-protocol extension
        Peer supports BGP route refresh capability
        Peer supports BGP route AS4 capability
        Address family IPv4 Unicast: advertised and received

 InQ updates: 0, OutQ updates: 0
 NLRI statistics:
        Rcvd:   UnReach NLRI          0,      Reach NLRI          2
        Sent:   UnReach NLRI          0,      Reach NLRI         15

 Message statistics:
 Msg type     Last rcvd time/      Current rcvd count/      History rcvd count/
              Last sent time       Current sent count       History sent count
 Open         06:45:47-2025.10.15  1                        1
              06:45:47-2025.10.15  1                        1
 Update       07:02:30-2025.10.15  3                        3
              19:16:37-2025.10.15  11                       11
 Notification -                    0                        0
              -                    0                        0
 Keepalive    19:24:42-2025.10.15  983                      983
              19:24:15-2025.10.15  986                      986
 RouteRefresh -                    0                        0
              -                    0                        0
 Total        -                    987                      987
              -                    998                      998

 Maximum allowed prefix number: 4294967295
 Threshold: 75%
 Authentication type configured: None
 Minimum time between advertisements is 30 seconds
 Optional capabilities:
  Multi-protocol extended capability has been enabled
  Route refresh capability has been enabled
  Extended nexthop encoding has been enabled
 Peer preferred value: 0
 Site-of-Origin: Not specified
 Routing policy configured:
 No routing policy is configured
[PE1]
```   
</details>

<details>
<summary> PE2 </summary>

```   
[PE2]display ip routing-table

Destinations : 30       Routes : 33

Destination/Mask   Proto   Pre Cost        NextHop         Interface
1.1.1.1/32         IS_L1   15  10          10.0.0.0        GE0/0/0
2.2.2.2/32         Direct  0   0           127.0.0.1       Loop0
3.3.3.3/32         IS_L1   15  10          10.0.0.6        GE0/0/2
4.4.4.4/32         IS_L1   15  10          10.0.0.9        GE0/0/1
10.0.0.0/31        Direct  0   0           10.0.0.1        GE0/0/0
10.0.0.1/32        Direct  0   0           127.0.0.1       GE0/0/0
10.0.0.2/31        IS_L1   15  20          10.0.0.0        GE0/0/0
                   IS_L1   15  20          10.0.0.6        GE0/0/2
10.0.0.4/31        IS_L1   15  20          10.0.0.0        GE0/0/0
                   IS_L1   15  20          10.0.0.9        GE0/0/1
10.0.0.6/31        Direct  0   0           10.0.0.7        GE0/0/2
10.0.0.7/32        Direct  0   0           127.0.0.1       GE0/0/2
10.0.0.8/31        Direct  0   0           10.0.0.8        GE0/0/1
10.0.0.8/32        Direct  0   0           127.0.0.1       GE0/0/1
10.0.0.10/31       IS_L1   15  20          10.0.0.6        GE0/0/2
                   IS_L1   15  20          10.0.0.9        GE0/0/1
10.0.0.12/31       IS_L1   15  20          10.0.0.0        GE0/0/0
10.0.0.14/31       Direct  0   0           10.0.0.14       GE0/0/7
10.0.0.14/32       Direct  0   0           127.0.0.1       GE0/0/7
10.0.0.16/31       IS_L1   15  20          10.0.0.6        GE0/0/2
10.0.0.18/31       IS_L1   15  20          10.0.0.9        GE0/0/1
10.0.0.20/31       IS_L1   15  20          10.0.0.6        GE0/0/2
10.0.0.22/31       IS_L1   15  20          10.0.0.9        GE0/0/1
11.11.11.11/32     BGP     255 0           10.0.0.13       GE0/0/0
12.12.12.12/32     BGP     255 0           3.3.3.3         GE0/0/2
21.21.21.21/32     BGP     255 0           10.0.0.15       GE0/0/7
22.22.22.22/32     BGP     255 0           4.4.4.4         GE0/0/1
31.31.31.31/32     BGP     255 0           3.3.3.3         GE0/0/2
32.32.32.32/32     BGP     255 0           4.4.4.4         GE0/0/1
127.0.0.0/8        Direct  0   0           127.0.0.1       InLoop0
127.0.0.1/32       Direct  0   0           127.0.0.1       InLoop0
127.255.255.255/32 Direct  0   0           127.0.0.1       InLoop0
255.255.255.255/32 Direct  0   0           127.0.0.1       InLoop0
[PE2]
[PE2]display ip routing-table protocol bgp

Summary count : 14

BGP Routing table status : <Active>
Summary count : 6

Destination/Mask   Proto   Pre Cost        NextHop         Interface
11.11.11.11/32     BGP     255 0           10.0.0.13       GE0/0/0
12.12.12.12/32     BGP     255 0           3.3.3.3         GE0/0/2
21.21.21.21/32     BGP     255 0           10.0.0.15       GE0/0/7
22.22.22.22/32     BGP     255 0           4.4.4.4         GE0/0/1
31.31.31.31/32     BGP     255 0           3.3.3.3         GE0/0/2
32.32.32.32/32     BGP     255 0           4.4.4.4         GE0/0/1

BGP Routing table status : <Inactive>
Summary count : 8

Destination/Mask   Proto   Pre Cost        NextHop         Interface
1.1.1.1/32         BGP     255 0           1.1.1.1         GE0/0/0
3.3.3.3/32         BGP     255 0           3.3.3.3         GE0/0/2
4.4.4.4/32         BGP     255 0           4.4.4.4         GE0/0/1
10.0.0.12/31       BGP     255 0           1.1.1.1         GE0/0/0
10.0.0.16/31       BGP     255 0           3.3.3.3         GE0/0/2
10.0.0.18/31       BGP     255 0           4.4.4.4         GE0/0/1
10.0.0.20/31       BGP     255 0           3.3.3.3         GE0/0/2
10.0.0.22/31       BGP     255 0           4.4.4.4         GE0/0/1
[PE2]
[PE2]disp bgp peer ipv4

 BGP local router ID: 2.2.2.2
 Local AS number: 65000
 Total number of peers: 2                 Peers in established state: 2

 * - Dynamically created peer
 Peer                    AS  MsgRcvd  MsgSent OutQ  PrefRcv Up/Down  State

 1.1.1.1              65000       30       20    0       13 00:15:09 Established
 10.0.0.15            65002      982      997    0        2 12:37:11 Established
[PE2]
```   
</details>

<details>
<summary> PE3 </summary>

```   
[PE3]display ip routing-table

Destinations : 31       Routes : 34

Destination/Mask   Proto   Pre Cost        NextHop         Interface
1.1.1.1/32         IS_L1   15  10          10.0.0.2        GE0/0/1
2.2.2.2/32         IS_L1   15  10          10.0.0.7        GE0/0/2
3.3.3.3/32         Direct  0   0           127.0.0.1       Loop0
4.4.4.4/32         IS_L1   15  10          10.0.0.11       GE0/0/0
10.0.0.0/31        IS_L1   15  20          10.0.0.2        GE0/0/1
                   IS_L1   15  20          10.0.0.7        GE0/0/2
10.0.0.2/31        Direct  0   0           10.0.0.3        GE0/0/1
10.0.0.3/32        Direct  0   0           127.0.0.1       GE0/0/1
10.0.0.4/31        IS_L1   15  20          10.0.0.2        GE0/0/1
                   IS_L1   15  20          10.0.0.11       GE0/0/0
10.0.0.6/31        Direct  0   0           10.0.0.6        GE0/0/2
10.0.0.6/32        Direct  0   0           127.0.0.1       GE0/0/2
10.0.0.8/31        IS_L1   15  20          10.0.0.7        GE0/0/2
                   IS_L1   15  20          10.0.0.11       GE0/0/0
10.0.0.10/31       Direct  0   0           10.0.0.10       GE0/0/0
10.0.0.10/32       Direct  0   0           127.0.0.1       GE0/0/0
10.0.0.12/31       IS_L1   15  20          10.0.0.2        GE0/0/1
10.0.0.14/31       IS_L1   15  20          10.0.0.7        GE0/0/2
10.0.0.16/31       Direct  0   0           10.0.0.16       GE0/0/7
10.0.0.16/32       Direct  0   0           127.0.0.1       GE0/0/7
10.0.0.18/31       IS_L1   15  20          10.0.0.11       GE0/0/0
10.0.0.20/31       Direct  0   0           10.0.0.20       GE0/0/8
10.0.0.20/32       Direct  0   0           127.0.0.1       GE0/0/8
10.0.0.22/31       IS_L1   15  20          10.0.0.11       GE0/0/0
11.11.11.11/32     BGP     255 0           10.0.0.13       GE0/0/1
12.12.12.12/32     BGP     255 0           10.0.0.17       GE0/0/7
21.21.21.21/32     BGP     255 0           2.2.2.2         GE0/0/2
22.22.22.22/32     BGP     255 0           4.4.4.4         GE0/0/0
31.31.31.31/32     BGP     255 0           10.0.0.21       GE0/0/8
32.32.32.32/32     BGP     255 0           4.4.4.4         GE0/0/0
127.0.0.0/8        Direct  0   0           127.0.0.1       InLoop0
127.0.0.1/32       Direct  0   0           127.0.0.1       InLoop0
127.255.255.255/32 Direct  0   0           127.0.0.1       InLoop0
255.255.255.255/32 Direct  0   0           127.0.0.1       InLoop0
[PE3]
[PE3]display ip routing-table protocol bgp

Summary count : 13

BGP Routing table status : <Active>
Summary count : 6

Destination/Mask   Proto   Pre Cost        NextHop         Interface
11.11.11.11/32     BGP     255 0           10.0.0.13       GE0/0/1
12.12.12.12/32     BGP     255 0           10.0.0.17       GE0/0/7
21.21.21.21/32     BGP     255 0           2.2.2.2         GE0/0/2
22.22.22.22/32     BGP     255 0           4.4.4.4         GE0/0/0
31.31.31.31/32     BGP     255 0           10.0.0.21       GE0/0/8
32.32.32.32/32     BGP     255 0           4.4.4.4         GE0/0/0

BGP Routing table status : <Inactive>
Summary count : 7

Destination/Mask   Proto   Pre Cost        NextHop         Interface
1.1.1.1/32         BGP     255 0           1.1.1.1         GE0/0/1
2.2.2.2/32         BGP     255 0           2.2.2.2         GE0/0/2
4.4.4.4/32         BGP     255 0           4.4.4.4         GE0/0/0
10.0.0.12/31       BGP     255 0           1.1.1.1         GE0/0/1
10.0.0.14/31       BGP     255 0           2.2.2.2         GE0/0/2
10.0.0.18/31       BGP     255 0           4.4.4.4         GE0/0/0
10.0.0.22/31       BGP     255 0           4.4.4.4         GE0/0/0
[PE3]
[PE3]disp bgp peer ipv4

 BGP local router ID: 3.3.3.3
 Local AS number: 65000
 Total number of peers: 3                 Peers in established state: 3

 * - Dynamically created peer
 Peer                    AS  MsgRcvd  MsgSent OutQ  PrefRcv Up/Down  State

 1.1.1.1              65000       30       23    0       11 00:15:07 Established
 10.0.0.17            65001      997      864    0        2 12:45:53 Established
 10.0.0.21            65003      995      914    0        2 12:45:31 Established
[PE3]
```   
</details>

<details>
<summary> PE4 </summary>

```   
[PE4]display ip routing-table

Destinations : 31       Routes : 34

Destination/Mask   Proto   Pre Cost        NextHop         Interface
1.1.1.1/32         IS_L1   15  10          10.0.0.4        GE0/0/2
2.2.2.2/32         IS_L1   15  10          10.0.0.8        GE0/0/1
3.3.3.3/32         IS_L1   15  10          10.0.0.10       GE0/0/0
4.4.4.4/32         Direct  0   0           127.0.0.1       Loop0
10.0.0.0/31        IS_L1   15  20          10.0.0.4        GE0/0/2
                   IS_L1   15  20          10.0.0.8        GE0/0/1
10.0.0.2/31        IS_L1   15  20          10.0.0.4        GE0/0/2
                   IS_L1   15  20          10.0.0.10       GE0/0/0
10.0.0.4/31        Direct  0   0           10.0.0.5        GE0/0/2
10.0.0.5/32        Direct  0   0           127.0.0.1       GE0/0/2
10.0.0.6/31        IS_L1   15  20          10.0.0.8        GE0/0/1
                   IS_L1   15  20          10.0.0.10       GE0/0/0
10.0.0.8/31        Direct  0   0           10.0.0.9        GE0/0/1
10.0.0.9/32        Direct  0   0           127.0.0.1       GE0/0/1
10.0.0.10/31       Direct  0   0           10.0.0.11       GE0/0/0
10.0.0.11/32       Direct  0   0           127.0.0.1       GE0/0/0
10.0.0.12/31       IS_L1   15  20          10.0.0.4        GE0/0/2
10.0.0.14/31       IS_L1   15  20          10.0.0.8        GE0/0/1
10.0.0.16/31       IS_L1   15  20          10.0.0.10       GE0/0/0
10.0.0.18/31       Direct  0   0           10.0.0.18       GE0/0/7
10.0.0.18/32       Direct  0   0           127.0.0.1       GE0/0/7
10.0.0.20/31       IS_L1   15  20          10.0.0.10       GE0/0/0
10.0.0.22/31       Direct  0   0           10.0.0.22       GE0/0/8
10.0.0.22/32       Direct  0   0           127.0.0.1       GE0/0/8
11.11.11.11/32     BGP     255 0           10.0.0.13       GE0/0/2
12.12.12.12/32     BGP     255 0           3.3.3.3         GE0/0/0
21.21.21.21/32     BGP     255 0           2.2.2.2         GE0/0/1
22.22.22.22/32     BGP     255 0           10.0.0.19       GE0/0/7
31.31.31.31/32     BGP     255 0           3.3.3.3         GE0/0/0
32.32.32.32/32     BGP     255 0           10.0.0.23       GE0/0/8
127.0.0.0/8        Direct  0   0           127.0.0.1       InLoop0
127.0.0.1/32       Direct  0   0           127.0.0.1       InLoop0
127.255.255.255/32 Direct  0   0           127.0.0.1       InLoop0
255.255.255.255/32 Direct  0   0           127.0.0.1       InLoop0
[PE4]
[PE4]display ip routing-table protocol bgp

Summary count : 13

BGP Routing table status : <Active>
Summary count : 6

Destination/Mask   Proto   Pre Cost        NextHop         Interface
11.11.11.11/32     BGP     255 0           10.0.0.13       GE0/0/2
12.12.12.12/32     BGP     255 0           3.3.3.3         GE0/0/0
21.21.21.21/32     BGP     255 0           2.2.2.2         GE0/0/1
22.22.22.22/32     BGP     255 0           10.0.0.19       GE0/0/7
31.31.31.31/32     BGP     255 0           3.3.3.3         GE0/0/0
32.32.32.32/32     BGP     255 0           10.0.0.23       GE0/0/8

BGP Routing table status : <Inactive>
Summary count : 7

Destination/Mask   Proto   Pre Cost        NextHop         Interface
1.1.1.1/32         BGP     255 0           1.1.1.1         GE0/0/2
2.2.2.2/32         BGP     255 0           2.2.2.2         GE0/0/1
3.3.3.3/32         BGP     255 0           3.3.3.3         GE0/0/0
10.0.0.12/31       BGP     255 0           1.1.1.1         GE0/0/2
10.0.0.14/31       BGP     255 0           2.2.2.2         GE0/0/1
10.0.0.16/31       BGP     255 0           3.3.3.3         GE0/0/0
10.0.0.20/31       BGP     255 0           3.3.3.3         GE0/0/0
[PE4]
[PE4]disp bgp peer ipv4

 BGP local router ID: 4.4.4.4
 Local AS number: 65000
 Total number of peers: 3                 Peers in established state: 3

 * - Dynamically created peer
 Peer                    AS  MsgRcvd  MsgSent OutQ  PrefRcv Up/Down  State

 1.1.1.1              65000       31       23    0       11 00:15:39 Established
 10.0.0.19            65002      995      913    0        2 12:41:47 Established
 10.0.0.23            65003      992      862    0        2 12:42:34 Established
[PE4]
```   
</details>


### Настройка MPLS
текст

<details>
<summary> PE1 </summary>

```   
```   
</details>

<details>
<summary> PE2 </summary>

```   
```   
</details>

<details>
<summary> PE3 </summary>

```   
```   
</details>

<details>
<summary> PE4 </summary>

```   
```   
</details>


Диагностика:
<details>
<summary> PE1 </summary>

```   
```   
</details>

<details>
<summary> PE2 </summary>

```   
```   
</details>

<details>
<summary> PE3 </summary>

```   
```   
</details>

<details>
<summary> PE4 </summary>

```   
```   
</details>


Проверка сетевой связности.

<details>
<summary> PE1 </summary>

```   
```   
</details>

<details>
<summary> PE2 </summary>

```   
```   
</details>

<details>
<summary> PE3 </summary>
```   
```   
</details>

<details>
<summary> PE4 </summary>

```   
```   
</details>

```   
```   

### Настройка TE

текст

<details>
<summary> PE1 </summary>

```   
```   
</details>

<details>
<summary> PE2 </summary>

```   
```   
</details>

<details>
<summary> PE3 </summary>

```   
```   
</details>

<details>
<summary> PE4 </summary>

```   
```   
</details>


Диагностика:
<details>
<summary> PE1 </summary>

```   
```   
</details>

<details>
<summary> PE2 </summary>

```   
```   
</details>

<details>
<summary> PE3 </summary>

```   
```   
</details>

<details>
<summary> PE4 </summary>

```   
```   
</details>


Проверка сетевой связности.

<details>
<summary> PE1 </summary>

```   
```   
</details>

<details>
<summary> PE2 </summary>

```   
```   
</details>

<details>
<summary> PE3 </summary>

```   
```   
</details>

<details>
<summary> PE4 </summary>

```   
```   
</details>

```   
```   

## Выводы


#### Всякая фигня

Тире:


En-Dash         &ndash;

Em-Dash         &mdash;

Minus Symbol    &minus;


стрелки:

Up arrow (^): &uarr;
Down arrow (v): &darr;
Left arrow (<): &larr;
Right arrow (>): &rarr;
Double headed arrow (-): &harr;
