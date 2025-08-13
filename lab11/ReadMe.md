# ������������ ������ �11 "BGP ���������� ��������/����������"

## �������

1. ��������� ���������� � ����� ������ ���, ����� �� ��������� ����������� �������(As-path).
2. ��������� ���������� � ����� �.-��������� ���, ����� �� ��������� ����������� �������(Prefix-list).
3. ��������� ���������� ������ ���, ����� � ���� ������ ��������� ������ ������� �� ���������.
4. ��������� ���������� ����� ���, ����� � ���� ������ ��������� ������ ������� �� ��������� � ������� ����� �.-���������.
5. ��� ���� � ������������ ������ ������ ����� IP ���������.


����� ���� ������������ �� ������� ����:

![Topology-with-IP.jpg](./img/Topology-with-IP.jpg)


## �������

### ���������

#### ������

��� �������������� ����������� ������� � ������ ���������� ������������ ������� ������ ���� ������������� AS 1001.
��� ����� ������������� ip as-path access list. ��������� � ��� ������� ������ � ������� R22 (111.111.111.1) �� R14:
```
router bgp 1001
 ...
 neighbor 111.111.111.1 filter-list 1 out
exit
ip as-path access-list 1 permit ^$
end
```
� �� R15:
```
router bgp 1001
 ...
 neighbor 131.131.131.1 filter-list 1 out
exit
ip as-path access-list 1 permit ^$
end
```
#### �����-���������

������� ���������� � ����� �.-��������� ���, ����� �� ��������� ����������� �������, ��������� Prefix-list.

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
#### ������

���������� ���������� ������ ���, ����� � ���� ������ ��������� ������ ������� �� ���������.
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

#### �����

���������� ���������� ����� ���, ����� � ���� ������ ��������� ������ ������� �� ��������� � ������� ����� �.-���������.
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

### ��������

#### ������

�� ���������:

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
����� ���������:
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

#### �����-���������

�� ���������:

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

����� ���������:

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


#### ������
�� ���������:

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



����� ����������:

```
R22#
*Aug 10 17:33:24.300: %SYS-5-CONFIG_I: Configured from console by console
R22#sh ip bgp neighbors 111.111.111.2 advertised-routes

Total number of prefixes 0
R22#sh ip bgp neighbors 111.111.111.2 advertised-routes

Total number of prefixes 0
```


�������� �� R14:

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


#### �����

�������� �� R15, ��������� �� R21 (131.131.131.1):

�� ���������:

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


����� ���������:

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



���������������e ����� ����� ����� �� [������](./cfg).
