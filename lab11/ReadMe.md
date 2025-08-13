# Лабораторная работа №11 "BGP управление анонсами/фильтрация"

## Задание

1. Настроить фильтрацию в офисе Москва так, чтобы не появилось транзитного трафика(As-path).
2. Настроить фильтрацию в офисе С.-Петербург так, чтобы не появилось транзитного трафика(Prefix-list).
3. Настроить провайдера Киторн так, чтобы в офис Москва отдавался только маршрут по умолчанию.
4. Настроить провайдера Ламас так, чтобы в офис Москва отдавался только маршрут по умолчанию и префикс офиса С.-Петербург.
5. Все сети в лабораторной работе должны иметь IP связность.


Схема сети представлена на рисунке ниже:

![Topology-with-IP.jpg](./img/Topology-with-IP.jpg)


## Решение

### Настройка

#### Москва

Для предотвращения транзитного трафика в Москве необходимо анонсировать соседям только сети принадлежащие AS 1001.
Для этого настраивается ip as-path access list. Фильтруем с его помощью анонсы в сторону R22 (111.111.111.1) на R14:
```
router bgp 1001
 ...
 neighbor 111.111.111.1 filter-list 1 out
exit
ip as-path access-list 1 permit ^$
end
```
и на R15:
```
router bgp 1001
 ...
 neighbor 131.131.131.1 filter-list 1 out
exit
ip as-path access-list 1 permit ^$
end
```
#### Санкт-Петербург

Настрою фильтрацию в офисе С.-Петербург так, чтобы не появилось транзитного трафика, используя Prefix-list.

R18:
```
router bgp 2042
 ...
 neighbor 152.152.152.1 prefix-list TRANSITE_OFF out
 neighbor 152.152.152.5 prefix-list TRANSITE_OFF out
exit 
ip prefix-list TRANSITE_OFF seq 5 permit 10.78.0.0/16
end
```
#### Киторн

Настраиваю провайдера Киторн так, чтобы в офис Москва отдавался только маршрут по умолчанию.
R22:
```
router bgp 101
...
 neighbor 111.111.111.2 default-originate
 neighbor 111.111.111.2 prefix-list DEFAULT_MSK out
exit
ip prefix-list DEFAULT_MSK seq 5 permit 0.0.0.0/0
end
```

#### Ламас

Настраиваю провайдера Ламас так, чтобы в офис Москва отдавался только маршрут по умолчанию и префикс офиса С.-Петербург.
R21:
```
router bgp 301
 
 ...
 neighbor 131.131.131.2 default-originate
 neighbor 131.131.131.2 filter-list 1 out
exit
ip as-path access-list 1 permit 2042$
end
```

### Проверка

#### Москва

До настройки:

```
R14#sh ip bgp neighbors 111.111.111.1 advertised-routes
BGP table version is 8, local router ID is 10.77.0.14
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  10.77.0.0/16     0.0.0.0                  0         32768 i
 *>i 10.78.0.0/16     10.77.0.15               0    200      0 301 520 2042 i
 *>i 10.152.1.0/24    10.77.0.15               0    200      0 301 520 i
 *>i 111.111.111.0/24 10.77.0.15               0    200      0 301 101 i
 *>i 131.131.131.0/24 10.77.0.15               0    200      0 301 i
```
После настройки:
```
R14#sh ip bgp neighbors 111.111.111.1 advertised-routes
BGP table version is 8, local router ID is 10.77.0.14
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  10.77.0.0/16     0.0.0.0                  0         32768 i

Total number of prefixes 1


R15#sh ip bgp neighbors 131.131.131.1 advertised-routes
BGP table version is 32, local router ID is 10.77.0.15
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  10.77.0.0/16     0.0.0.0                  0         32768 i

Total number of prefixes 1

```

#### Санкт-Петербург

До настройки:

```
R18#sh ip bgp neighbors 152.152.152.1 advertised-routes
BGP table version is 108, local router ID is 10.78.0.18
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  10.77.0.0/16     152.152.152.1                          0 520 301 1001 i
 *>  10.78.0.0/16     0.0.0.0                  0         32768 i
 *>  10.152.1.0/24    152.152.152.5            0             0 520 i
 *>  111.111.111.0/24 152.152.152.5                          0 520 101 i
 *>  131.131.131.0/24 152.152.152.1                          0 520 301 i

Total number of prefixes 5
R18#sh ip bgp neighbors 152.152.152.5 advertised-routes
BGP table version is 108, local router ID is 10.78.0.18
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  10.77.0.0/16     152.152.152.1                          0 520 301 1001 i
 *>  10.78.0.0/16     0.0.0.0                  0         32768 i
 *>  10.152.1.0/24    152.152.152.5            0             0 520 i
 *>  111.111.111.0/24 152.152.152.5                          0 520 101 i
 *>  131.131.131.0/24 152.152.152.1                          0 520 301 i

Total number of prefixes 5
```

После настройки:

```
R18#sh ip bgp neighbors 152.152.152.5 advertised-routes
BGP table version is 108, local router ID is 10.78.0.18
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  10.78.0.0/16     0.0.0.0                  0         32768 i

Total number of prefixes 1
R18#sh ip bgp neighbors 152.152.152.1 advertised-routes
BGP table version is 108, local router ID is 10.78.0.18
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  10.78.0.0/16     0.0.0.0                  0         32768 i

Total number of prefixes 1
```


#### Киторн
До настройки:

```
R22#
R22#sh ip bgp neighbors 111.111.111.2 advertised-routes
BGP table version is 53, local router ID is 10.111.0.22
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  10.77.0.0/16     111.111.111.6                          0 301 1001 i
 *>  10.78.0.0/16     111.111.111.10                         0 520 2042 i
 *>  10.152.1.0/24    111.111.111.10                         0 520 i
 *>  111.111.111.0/24 0.0.0.0                  0         32768 i
 *>  131.131.131.0/24 111.111.111.6            0             0 301 i

Total number of prefixes 5
```



После настрогйки:

```
R22#
*Aug 10 17:33:24.300: %SYS-5-CONFIG_I: Configured from console by console
R22#sh ip bgp neighbors 111.111.111.2 advertised-routes

Total number of prefixes 0
R22#sh ip bgp neighbors 111.111.111.2 advertised-routes

Total number of prefixes 0
```


Маршруты на R14:

```
R14>sh ip bgp ipv4 unicast
BGP table version is 28, local router ID is 10.77.0.14
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  0.0.0.0          111.111.111.1                          0 101 i
 * i 10.77.0.0/16     10.77.0.15               0    100      0 i
 *>                   0.0.0.0                  0         32768 i
 *>i 10.78.0.0/16     10.77.0.15               0    200      0 301 520 2042 i
 *>i 10.152.1.0/24    10.77.0.15               0    200      0 301 520 i
 *>  111.111.111.0/30 0.0.0.0                  0         32768 ?
 *>i 111.111.111.0/24 10.77.0.15               0    200      0 301 101 i
 *>i 131.131.131.0/30 10.77.0.15               0    100      0 ?
 *>i 131.131.131.0/24 10.77.0.15               0    200      0 301 i
```


#### Ламас

Маршруты на R15, пришедшие от R21 (131.131.131.1):

До настройки:

```
R15#sh ip bgp ipv4 unicast
BGP table version is 28, local router ID is 10.77.0.15
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 r>i 0.0.0.0          10.77.0.14               0    100      0 101 i
 * i 10.77.0.0/16     10.77.0.14               0    100      0 i
 *>                   0.0.0.0                  0         32768 i
 *>  10.78.0.0/16     131.131.131.1                 200      0 301 520 2042 i
 *>  10.152.1.0/24    131.131.131.1                 200      0 301 520 i
 *>i 111.111.111.0/30 10.77.0.14               0    100      0 ?
 *>  111.111.111.0/24 131.131.131.1                 200      0 301 101 i
 *>  131.131.131.0/30 0.0.0.0                  0         32768 ?
 *>  131.131.131.0/24 131.131.131.1            0    200      0 301 i
```


После настройки:

```
R15#sh ip bgp ipv4 unicast
BGP table version is 32, local router ID is 10.77.0.15
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  0.0.0.0          131.131.131.1                 200      0 301 i
 * i 10.77.0.0/16     10.77.0.14               0    100      0 i
 *>                   0.0.0.0                  0         32768 i
 *>  10.78.0.0/16     131.131.131.1                 200      0 301 520 2042 i
 *>i 111.111.111.0/30 10.77.0.14               0    100      0 ?
 *>  131.131.131.0/30 0.0.0.0                  0         32768 ?
```



Конфигурационныe файлы можно найти по [ссылке](./cfg).
