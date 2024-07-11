# Настройка VxLAN. L3 VNI

### Цели

1. Настройка маршрутизации в рамках Overlay между клиентами для VxLAN
2. Проверка наличия IP связанности между клиентами

### Схема сети

![Network scheme](../lab05/network_scheme5.png)

Базовые настройки сети Underlay, а так же BGP EVPN были рассмотрены в рамках лабораторной работы №5. Воспользуемся ими в данной работе.

### IP план

Device|Interface|IP Address|Subnet Mask
---|---|---|---
Spine-1|Lo1|10.0.1.0|255.255.255.255
||Lo2|10.1.1.0|255.255.255.255
||GE1/0/0|10.2.1.0|255.255.255.254
||GE1/0/1|10.2.1.2|255.255.255.254
||GE1/0/2|10.2.1.4|255.255.255.254
Spine-2|Lo1|10.0.2.0|255.255.255.255
||Lo2|10.1.2.0|255.255.255.255
||GE1/0/0|10.2.2.0|255.255.255.254
||GE1/0/1|10.2.2.2|255.255.255.254
||GE1/0/2|10.2.2.4|255.255.255.254
Leaf-1|Lo1|10.0.0.1|255.255.255.255
||Lo2|10.1.0.1|255.255.255.255
||Eth1|10.2.1.1|255.255.255.254
||Eth2|10.2.2.1|255.255.255.254
||Vlanif100|10.4.0.1|255.255.255.192
Leaf-2|Lo1|10.0.0.2|255.255.255.255
||Lo2|10.1.0.2|255.255.255.255
||Eth1|10.2.1.3|255.255.255.254
||Eth2|10.2.2.3|255.255.255.254
||Vlanif200|10.4.0.65|255.255.255.192
Leaf-3|Lo1|10.0.0.3|255.255.255.255
||Lo2|10.1.0.3|255.255.255.255
||GE1/0/0|10.2.1.5|255.255.255.254
||GE1/0/1|10.2.2.5|255.255.255.254
Client-1|eth0|10.4.0.2|255.255.255.192
Client-2|eth0|10.4.0.66|255.255.255.192

### Схемы маршрутизации трафика между SVI

Существуют 3 основных схемы маршрутизации трафика между SVI.

**Centralized Routing**

В данной реализации появляется единая точка выхода из подсети, это может быть отдельный маршрутизатор или коммутатор L3. Данный дизайн выглядит хорошо только в том случае, когда взаимодействие между разными подсетями минимальны, т.к. весь трафик между подсетями пойдет через данное устройство.

**Asymmetric IRB**

Очень схожая по настройке реализация с Symmetric IRB (о которой ниже), но с одним исключением. Вся маршрутизация происходит на первом же VTEP, и последующим устройствам остаётся только отдать пакет по L2. При такой реализации необходимо, чтобы на каждом Leaf были все VNI+Vlan, что является платой за отказ от L3VNI.

**Symmetric IRB**

При симметричной модели для всех L3 коммуникаций у нас создается отдельная сущность L3VNI, с которой ассоциируется VLAN. Именно данный функционал избавляет нас от необходимости иметь все VNI на каждом VTEP. Остановимся на данном варианте как наиболее простом и универсальном.

### Настройка маршрутизации в рамках Overlay между клиентами для VxLAN

#### Настройка SVI

На Leaf-1 создадим VLAN 100 и разрешим его в сторону Client-1. На Leaf-2 создадим VLAN 200 и разрешим его в сторону Client-2.

Пример настройки для Leaf-1:

    vlan 100
    !
    interface Ethernet8
       description to Client-1
       switchport access vlan 100

Создаем VNI 10100 и добавляем в него VLAN 100 на Leaf-1. Создаем VNI 10200 и добавляем в него VLAN 200 на Leaf-2.

Пример настройки для Leaf-1:

    interface Vxlan1
       vxlan vlan 100 vni 10100
    !
    router bgp 65000
       vlan 100
          rd 10.1.0.1:10100
          route-target both 65000:10100
          redistribute learned

Команда *redistribute learned* отвечает за то, что как только коммутатор узнает mac address через data plane, он его анонсирует через BGP.

Настраиваем SVI на Leaf-1 и Leaf-2 для каждого из клиентов.

Пример настройки для Leaf-1:

    ip virtual-router mac-address 00:66:00:00:00:00
    !
    interface Vlan100
       ip address virtual 10.4.0.1/26

Для каждого VNI настраиваем единый виртуальный IP-адрес и единый MAC для работы фукционала Anycast Gateway. Это позволит "бесшовно" мигрировать хостам между Leaf'ми.

Т.к. Client-1 и Client-2 находятся в разных VNI, то между ними отсутствует IP связност. Прверим это:

```
Client-1> ping 10.4.0.66

*10.4.0.1 icmp_seq=1 ttl=64 time=17.459 ms (ICMP type:3, code:0, Destination network unreachable)
*10.4.0.1 icmp_seq=2 ttl=64 time=13.882 ms (ICMP type:3, code:0, Destination network unreachable)
*10.4.0.1 icmp_seq=3 ttl=64 time=8.812 ms (ICMP type:3, code:0, Destination network unreachable)
*10.4.0.1 icmp_seq=4 ttl=64 time=11.673 ms (ICMP type:3, code:0, Destination network unreachable)
*10.4.0.1 icmp_seq=5 ttl=64 time=9.904 ms (ICMP type:3, code:0, Destination network unreachable)
```

#### Настройка VxLAN L3 VNI

Создадим общий L3VNI (VRF), назначим ему VNI 10000 и анонсируем его BGP EVPN.

Пример настройки для Leaf-1:

    vrf instance VxLAN
    !
    interface Vxlan1
       vxlan vrf VxLAN vni 10000
    !
    ip routing vrf VxLAN
    !
    router bgp 65000
       !
       vrf VxLAN
          rd 10.1.0.1:10000
          route-target import evpn 65000:10000
          route-target export evpn 65000:10000
          redistribute connected

Команда *redistribute connected* отвечает анонс всех direct connected в vrf VxLAN сетей через BGP.

Добавляем SVI в L3VNI:

    interface Vlan100
       vrf VxLAN

### Проверка наличия IP связанности между клиентами

Проверяем наличия IP связанности между клиентами:

```
Client-1> ping 10.4.0.66

84 bytes from 10.4.0.66 icmp_seq=1 ttl=62 time=295.753 ms
84 bytes from 10.4.0.66 icmp_seq=2 ttl=62 time=33.995 ms
84 bytes from 10.4.0.66 icmp_seq=3 ttl=62 time=43.738 ms
84 bytes from 10.4.0.66 icmp_seq=4 ttl=62 time=41.042 ms
84 bytes from 10.4.0.66 icmp_seq=5 ttl=62 time=43.701 ms
```

Так же на Leaf видим, что появились машруты type 5

    Leaf-1(config)#show bgp evpn route-type ip-prefix ipv4
    BGP routing table information for VRF default
    Router identifier 10.1.0.1, local AS number 65000
    Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                        c - Contributing to ECMP, % - Pending BGP convergence
    Origin codes: i - IGP, e - EGP, ? - incomplete
    AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop
    
              Network                Next Hop              Metric  LocPref Weight  Path
     * >      RD: 10.1.0.1:10000 ip-prefix 10.4.0.0/26
                                     -                     -       -       0       i
     * >      RD: 10.1.0.2:10000 ip-prefix 10.4.0.64/26
                                     10.0.0.2              -       100     0       i Or-ID: 10.1.0.2 C-LST: 10.1.1.0
     *        RD: 10.1.0.2:10000 ip-prefix 10.4.0.64/26
                                     10.0.0.2              -       100     0       i Or-ID: 10.1.0.2 C-LST: 10.1.2.0

### Конфигурация на оборудовании Huawei/Arista

<details>
<summary> Spine-1 </summary>

```
<Spine-1>display current-configuration
!Software Version V200R005C10SPC607B607
!Last configuration was updated at 2024-07-10 07:47:19+00:00 by SYSTEM automatically
!Last configuration was saved at 2024-07-10 16:23:18+00:00
#
sysname Spine-1
#
evpn-overlay enable
#
isis 1
 cost-style wide
 network-entity 49.0010.0100.0000.1000.00
#
interface GE1/0/0
 undo portswitch
 description to Leaf-1
 undo shutdown
 ip address 10.2.1.0 255.255.255.254
 isis enable 1
 isis circuit-type p2p
#
interface GE1/0/1
 undo portswitch
 description to Leaf-2
 undo shutdown
 ip address 10.2.1.2 255.255.255.254
 isis enable 1
 isis circuit-type p2p
#
interface GE1/0/2
 undo portswitch
 description to Leaf-3
 undo shutdown
 ip address 10.2.1.4 255.255.255.254
 isis enable 1
 isis circuit-type p2p
#
interface LoopBack1
 description Underlay
 ip address 10.0.1.0 255.255.255.255
 isis enable 1
#
interface LoopBack2
 description Overlay
 ip address 10.1.1.0 255.255.255.255
#
bgp 65000
 router-id 10.1.1.0
 group LEAVES internal
 peer LEAVES connect-interface LoopBack1
 peer 10.0.0.1 as-number 65000
 peer 10.0.0.1 group LEAVES
 peer 10.0.0.2 as-number 65000
 peer 10.0.0.2 group LEAVES
 peer 10.0.0.3 as-number 65000
 peer 10.0.0.3 group LEAVES
 #
 ipv4-family unicast
  undo peer LEAVES enable
  undo peer 10.0.0.1 enable
  undo peer 10.0.0.2 enable
  undo peer 10.0.0.3 enable
 #
 l2vpn-family evpn
  undo policy vpn-target
  peer LEAVES enable
  peer LEAVES reflect-client
  peer 10.0.0.1 enable
  peer 10.0.0.1 group LEAVES
  peer 10.0.0.2 enable
  peer 10.0.0.2 group LEAVES
  peer 10.0.0.3 enable
  peer 10.0.0.3 group LEAVES
#
```

</details>

<details>
<summary> Spine-2 </summary>

```
<Spine-2>display current-configuration
!Software Version V200R005C10SPC607B607
!Last configuration was updated at 2024-07-10 07:47:21+00:00 by SYSTEM automatically
!Last configuration was saved at 2024-07-09 14:59:15+00:00
#
sysname Spine-2
#
evpn-overlay enable
#
isis 1
 cost-style wide
 network-entity 49.0010.0100.0000.2000.00
#
interface GE1/0/0
 undo portswitch
 description to Leaf-1
 undo shutdown
 ip address 10.2.2.0 255.255.255.254
 isis enable 1
 isis circuit-type p2p
#
interface GE1/0/1
 undo portswitch
 description to Leaf-2
 undo shutdown
 ip address 10.2.2.2 255.255.255.254
 isis enable 1
 isis circuit-type p2p
#
interface GE1/0/2
 undo portswitch
 description to Leaf-3
 undo shutdown
 ip address 10.2.2.4 255.255.255.254
 isis enable 1
 isis circuit-type p2p
#
interface LoopBack1
 description Underlay
 ip address 10.0.2.0 255.255.255.255
 isis enable 1
#
interface LoopBack2
 description Overlay
 ip address 10.1.2.0 255.255.255.255
#
bgp 65000
 router-id 10.1.2.0
 group LEAVES internal
 peer LEAVES connect-interface LoopBack1
 peer 10.0.0.1 as-number 65000
 peer 10.0.0.1 group LEAVES
 peer 10.0.0.2 as-number 65000
 peer 10.0.0.2 group LEAVES
 peer 10.0.0.3 as-number 65000
 peer 10.0.0.3 group LEAVES
 #
 ipv4-family unicast
  undo peer LEAVES enable
  undo peer 10.0.0.1 enable
  undo peer 10.0.0.2 enable
  undo peer 10.0.0.3 enable
 #
 l2vpn-family evpn
  undo policy vpn-target
  peer LEAVES enable
  peer LEAVES reflect-client
  peer 10.0.0.1 enable
  peer 10.0.0.1 group LEAVES
  peer 10.0.0.2 enable
  peer 10.0.0.2 group LEAVES
  peer 10.0.0.3 enable
  peer 10.0.0.3 group LEAVES
#
```

</details>

<details>
<summary> Leaf-1 </summary>

```
Leaf-1(config)#show running-config
! Command: show running-config
! device: Leaf-1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
service routing protocols model multi-agent
!
hostname Leaf-1
!
vlan 100
!
vrf instance VxLAN
!
interface Ethernet1
   description to Spine-1
   no switchport
   ip address 10.2.1.1/31
   isis enable 1
   isis network point-to-point
!
interface Ethernet2
   description to Spine-2
   no switchport
   ip address 10.2.2.1/31
   isis enable 1
   isis network point-to-point
!
interface Ethernet8
   description to Client-1
   switchport access vlan 100
!
interface Loopback1
   description Underlay
   ip address 10.0.0.1/32
   isis enable 1
!
interface Loopback2
   description Overlay
   ip address 10.1.0.1/32
!
interface Vlan100
   vrf VxLAN
   ip address virtual 10.4.0.1/26
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 100 vni 10100
   vxlan vrf VxLAN vni 10000
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:66:00:00:00:00
!
ip routing
ip routing vrf VxLAN
!
router bgp 65000
   router-id 10.1.0.1
   no bgp default ipv4-unicast
   neighbor SPINES peer group
   neighbor SPINES remote-as 65000
   neighbor SPINES update-source Loopback1
   neighbor SPINES send-community extended
   neighbor 10.0.1.0 peer group SPINES
   neighbor 10.0.2.0 peer group SPINES
   !
   vlan 100
      rd 10.1.0.1:10100
      route-target both 65000:10100
      redistribute learned
   !
   address-family evpn
      neighbor SPINES activate
   !
   vrf VxLAN
      rd 10.1.0.1:10000
      route-target import evpn 65000:10000
      route-target export evpn 65000:10000
      redistribute connected
!
router isis 1
   net 49.0010.0100.0000.0001.00
   is-type level-1
   !
   address-family ipv4 unicast
!
end
```

</details>

<details>
<summary> Leaf-2 </summary>

```
Leaf-2(config)#show running-config
! Command: show running-config
! device: Leaf-2 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
service routing protocols model multi-agent
!
hostname Leaf-2
!
vlan 200
!
vrf instance VxLAN
!
interface Ethernet1
   description to Spine-1
   no switchport
   ip address 10.2.1.3/31
   isis enable 1
   isis network point-to-point
!
interface Ethernet2
   description to Spine-2
   no switchport
   ip address 10.2.2.3/31
   isis enable 1
   isis network point-to-point
!
interface Ethernet8
   description to Client-2
   switchport access vlan 200
!
interface Loopback1
   description Underlay
   ip address 10.0.0.2/32
   isis enable 1
!
interface Loopback2
   description Overlay
   ip address 10.1.0.2/32
!
interface Vlan200
   vrf VxLAN
   ip address virtual 10.4.0.65/26
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 200 vni 10200
   vxlan vrf VxLAN vni 10000
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:66:00:00:00:00
!
ip routing
ip routing vrf VxLAN
!
router bgp 65000
   router-id 10.1.0.2
   no bgp default ipv4-unicast
   neighbor SPINES peer group
   neighbor SPINES remote-as 65000
   neighbor SPINES update-source Loopback1
   neighbor SPINES send-community extended
   neighbor 10.0.1.0 peer group SPINES
   neighbor 10.0.2.0 peer group SPINES
   !
   vlan 200
      rd 10.1.0.2:10200
      route-target both 65000:10200
      redistribute learned
   !
   address-family evpn
      neighbor SPINES activate
   !
   vrf VxLAN
      rd 10.1.0.2:10000
      route-target import evpn 65000:10000
      route-target export evpn 65000:10000
      redistribute connected
!
router isis 1
   net 49.0010.0100.0000.0002.00
   is-type level-1
   !
   address-family ipv4 unicast
!
end
```

</details>

<details>
<summary> Leaf-3 </summary>

```

```

</details>
