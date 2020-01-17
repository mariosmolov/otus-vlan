# otus-vlan
Домашнее задание
строим бонды и вланы
В Office1 в тестовой подсети появляется сервер с доп. интерфесами и адресами в internal сети testLAN:
- testClient1 - 10.10.10.254
- testClient2 - 10.10.10.254
- testServer1- 10.10.10.1 
- testServer2- 10.10.10.1

Изолировать с помощью vlan:
testClient1 <-> testServer1
testClient2 <-> testServer2

Схема сети

![1](https://github.com/mariosmolov/otus-vlan/blob/master/network23.png)


Между centralRouter и inetRouter создать 2 линка (общая inernal сеть) и объединить их с помощью bond-интерфейса, 
проверить работу c отключением сетевых интерфейсов

# Работоспособность

Поднимаем кластер `vagrant up`

Между centralRouter и inetRouter созданы 2 линка и объединины с помощью team-интерфейса team0 в режиме loadbalance
```
[root@centralRouter vagrant]# teamdctl team0 state
setup:
  runner: loadbalance
ports:
  eth1
    link watches:
      link summary: up
      instance[link_watch_0]:
        name: ethtool
        link: up
        down count: 0
  eth2
    link watches:
      link summary: up
      instance[link_watch_0]:
        name: ethtool
        link: up
        down count: 0
```
Отключаем линк
```
[root@centralRouter vagrant]# ip link set dev eth2 down
[root@centralRouter vagrant]# teamdctl team0 state
setup:
  runner: loadbalance
ports:
  eth1
    link watches:
      link summary: up
      instance[link_watch_0]:
        name: ethtool
        link: up
        down count: 0
  eth2
    link watches:
      link summary: down
      instance[link_watch_0]:
        name: ethtool
        link: down
        down count: 1
[root@centralRouter vagrant]# ping 8.8.8.8 -c 2
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=61 time=44.2 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=61 time=45.4 ms

--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 44.270/44.868/45.466/0.598 ms
[root@centralRouter vagrant]# tracepath -n 8.8.8.8 
 1?: [LOCALHOST]                                         pmtu 1500
 1:  192.168.255.1                                         0.639ms 
 1:  192.168.255.1                                         0.457ms 
 2:  no reply
 3:  no reply
^C
```
Проверим изоляцию testClient1 и testServer1 в vlan100
```
[root@testClient1 vagrant]# cat /etc/sysconfig/network-scripts/ifcfg-eth1.100 
VLAN=yes
TYPE=Vlan
PHYSDEV=eth1
VLAN_ID=100
BOOTPROTO=none
IPADDR=10.10.10.1
PREFIX=24
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=no
NAME=eth1.100
DEVICE=eth1.100
ONBOOT=yes
```

Изоляция testClient2 и testServer2 в vlan101
```
[root@testClient2 vagrant]# cat /etc/sysconfig/network-scripts/ifcfg-eth1.101
VLAN=yes
TYPE=Vlan
PHYSDEV=eth1
VLAN_ID=101
BOOTPROTO=none
IPADDR=10.10.10.1
PREFIX=24
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=no
NAME=eth1.101
DEVICE=eth1.101
ONBOOT=yes
```
Проверим доступность сети
```
[root@testClient1 vagrant]# ping 10.10.10.254 -c 3
PING 10.10.10.254 (10.10.10.254) 56(84) bytes of data.
64 bytes from 10.10.10.254: icmp_seq=1 ttl=64 time=0.622 ms
64 bytes from 10.10.10.254: icmp_seq=2 ttl=64 time=0.809 ms
64 bytes from 10.10.10.254: icmp_seq=3 ttl=64 time=0.732 ms

--- 10.10.10.254 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 0.622/0.721/0.809/0.076 ms
[root@testClient1 vagrant]# tracepath -n 10.10.10.254
 1?: [LOCALHOST]                                         pmtu 1500
 1:  10.10.10.254                                          1.465ms reached
 1:  10.10.10.254                                          0.462ms reached
     Resume: pmtu 1500 hops 1 back 1 
[root@testClient2 vagrant]# ping 10.10.10.1 -c 3
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=0.033 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=0.081 ms
64 bytes from 10.10.10.1: icmp_seq=3 ttl=64 time=0.081 ms

--- 10.10.10.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1998ms
rtt min/avg/max/mdev = 0.033/0.065/0.081/0.022 ms
[root@testClient2 vagrant]# tracepath -n 10.10.10.1
 1:  10.10.10.1                                            0.177ms reached
     Resume: pmtu 65535 hops 1 back 1 
```
Созданы два netns
```
[root@officeRouter vagrant]# ip netns
VRF101 (id: 1)
VRF100 (id: 0)
```
Так же созданы vlan100 и vlan101 на VRF100 и VRF101
```
[root@officeRouter vagrant]# ip netns exec VRF100 ip -4 a
5: eth2.100@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000 link-netnsid 0
    inet 10.10.10.2/24 scope global eth2.100
       valid_lft forever preferred_lft forever
7: veth100@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000 link-netnsid 0
    inet 172.29.100.2/30 scope global veth100
       valid_lft forever preferred_lft forever
[root@officeRouter vagrant]# ip netns exec VRF101 ip -4 a
6: eth2.101@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000 link-netnsid 0
    inet 10.10.10.2/24 scope global eth2.101
       valid_lft forever preferred_lft forever
9: veth101@if10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000 link-netnsid 0
    inet 172.29.101.2/30 scope global veth101
       valid_lft forever preferred_lft forever
```
Трансляция сети 10.10.10.0/24 в сеть 10.10.100(101)
```[root@officeRouter vagrant]# ip netns exec VRF100 iptables -nL -t nat
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination         
NETMAP     all  --  0.0.0.0/0            10.10.100.0/24      10.10.10.0/24

Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
NETMAP     all  --  10.10.10.0/24        0.0.0.0/0           10.10.100.0/24
[root@officeRouter vagrant]# ip netns exec VRF101 iptables -nL -t nat
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination         
NETMAP     all  --  0.0.0.0/0            10.10.101.0/24      10.10.10.0/24

Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
NETMAP     all  --  10.10.10.0/24        0.0.0.0/0           10.10.101.0/24
```
И доступ для тест хостов интернет
```
[vagrant@testClient1 ~]$ ping 8.8.8.8 -c 3
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=55 time=48.1 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=55 time=46.0 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=55 time=47.8 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 46.023/47.330/48.162/0.969 ms
[vagrant@testClient1 ~]$ tracepath -n 8.8.8.8
 1?: [LOCALHOST]                                         pmtu 1500
 1:  10.10.10.2                                            0.678ms 
 1:  10.10.10.2                                            0.437ms 
 2:  172.29.100.1                                          0.589ms 
 3:  192.168.255.5                                         0.955ms 
 4:  192.168.255.1                                         1.395ms 
[vagrant@testClient2 ~]$ ping 8.8.8.8 -c 3
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=55 time=48.9 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=55 time=46.2 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=55 time=50.4 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2005ms
rtt min/avg/max/mdev = 46.210/48.539/50.461/1.777 ms
[vagrant@testClient2 ~]$ tracepath -n 8.8.8.8
 1?: [LOCALHOST]                                         pmtu 1500
 1:  10.10.10.2                                            0.709ms 
 1:  10.10.10.2                                            0.602ms 
 2:  172.29.101.1                                          0.526ms 
 3:  192.168.255.5                                         0.913ms 
 4:  192.168.255.1                                         1.820ms 
^C
```
