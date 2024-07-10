# Настройка VxLAN. L2 VNI

### Цели

1. Настройка IP адресации на топологии CLOS
2. Настройка ISIS для Underlay сети на оборудовании Huawei/Arista
3. Настройка Overlay на основе VxLAN EVPN для L2 связанности между клиентами
4. Проверка наличия IP связанности между клиентами

### Схема сети

![Network scheme](network_scheme5.png)

Ввиду того, что на оборудовании Huawei в тестовом стенде не отрабатывает BGP EVPN control plane на Leaf устройствах, было принято решение Leaf заменить на Arista. Устройства Spine оставляем на Huawei.

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
Leaf-2|Lo1|10.0.0.2|255.255.255.255
||Lo2|10.1.0.2|255.255.255.255
||Eth1|10.2.1.3|255.255.255.254
||Eth2|10.2.2.3|255.255.255.254
Leaf-3|Lo1|10.0.0.3|255.255.255.255
||Lo2|10.1.0.3|255.255.255.255
||GE1/0/0|10.2.1.5|255.255.255.254
||GE1/0/1|10.2.2.5|255.255.255.254
Client-1|eth0|10.4.0.2|255.255.255.192
Client-2|eth0|10.4.0.3|255.255.255.192

### Настройка IP адресации

Настраиваем IP адресацию на всех интерфейсах согласно IP плана и схемы сети.

Пример настройки для физического интерфейса на Huawei:

    interface GE1/0/0
      undo portswitch
      description to Leaf-1
      undo shutdown
      ip address 10.2.1.0 255.255.255.254

Пример настройки для физического интерфейса на Arista:

    interface Ethernet1
       description to Spine-1
       no switchport
       ip address 10.2.1.1/31

Пример настройки для Loopback интерфейса на Huawei:

    interface LoopBack1
      description Underlay
      ip address 10.0.0.1 255.255.255.255

Пример настройки для Loopback интерфейса на Arista:

    interface Loopback1
       description Underlay
       ip address 10.0.0.1/32

### Настройка ISIS для Underlay

На устройствах типа Leaf настраиваем L1, на устройствах типа Spine настраиваем L1/L2 (L2 оставляем для возможных расширений в будущем).

Поднимаем ISIS на устройстве. 

Пример настройки для Leaf (Arista):

    ip routing
    !
    router isis 1
       net 49.0010.0100.0000.0001.00
       is-type level-1
       !
       address-family ipv4 unicast

Пример настройки для Spine (Huawei):

    isis 1
     cost-style wide
     network-entity 49.0010.0100.0000.1000.00

Для топологии CLOS хорошо подходит ISIS в режиме работы *point-to-point*. 

Поднимаем ISIS на всех интерфейсах Leaf <-> Spine и Loopback1.

Пример настройки физического интерфейса Huawei:

    interface GE1/0/0
      isis enable 1
      isis circuit-type p2p

Пример настройки физического интерфейса Arista:

    interface Ethernet1
       isis enable 1
       isis network point-to-point

Пример настройки Loopback интерфейса Huawei/Arista:

    interface LoopBack1
      isis enable 1

### Настройка Overlay на основе VxLAN EVPN для L2 связанности между клиентами

#### Настройка BGP EVPN Overlay между Leaf'ми и Spine'ми

Для передачи control plane информации в Overlay будем использовать iBGP. Для начала настроим BGP соседство Leaf <> Spine. В качестве Router ID будем использовать IP-адреса Lo2 интерфейсов. Все соединения будем поднимать с Lo1 интерфейсов.

Пример настройки BGP на Spine (Huawei):

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

*к сожалению Huawei на тестовом стенде не позволяет использовать динамеческие группы, поэтому используем обычные.*

Пример настройки BGP на Leaf (Arista):

    router bgp 65000
       router-id 10.1.0.1
       no bgp default ipv4-unicast
       neighbor SPINES peer group
       neighbor SPINES remote-as 65000
       neighbor SPINES update-source Loopback1
       neighbor SPINES send-community extended
       neighbor 10.0.1.0 peer group SPINES
       neighbor 10.0.2.0 peer group SPINES

Добавляем поддержку семейства адресов evpn в BGP для обмена EVPN маршрутами.

Пример настройки на Spine (Huawei):

    evpn-overlay enable
    #
    bgp 65000
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

Пример настройки на Leaf (Arista):

    service routing protocols model multi-agent
    !
    router bgp 65000
       address-family evpn
          neighbor SPINES activate

Проверим на Leaf, что соседства поднялись:

    Leaf-1#show bgp evpn summary
    BGP summary information for VRF default
    Router identifier 10.1.0.1, local AS number 65000
    Neighbor Status Codes: m - Under maintenance
      Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
      10.0.1.0 4 65000            440       445    0    0 06:11:15 Estab   2      2
      10.0.2.0 4 65000            441       445    0    0 06:11:10 Estab   2      2

#### Настройка VxLAN L2 VNI

Перед началом настройки введем немного терминологии:

VTEP — Vitual Tunnel End Point, устройство на котором начинается или заканчивается VxLAN тоннель. В нашей топологии все Leaf коммутаторы являются VTEP.

VNI — Virtual Network Index — идентификатор сети в рамках VxLAN. Можно провести аналогию с VLAN. Однако есть некоторые отличия. При использовании фабрики, VLAN становятся уникальными только в рамках одного Leaf коммутатора и не передаются по сети. Но с каждым VLAN может быть проассоциирован номер VNI, который уже передается по сети.

Пример настройки VTEP:

    interface Vxlan1
       vxlan source-interface Loopback1
       vxlan udp-port 4789
       vxlan learn-restrict any
    
На Leaf-1 создадим VLAN 100 и разрешим его в сторону Client-1. На Leaf-2 создадим VLAN 200 и разрешим его в сторону Client-2.

Пример настройки для Leaf-1:

    vlan 100
    !
    interface Ethernet8
       description to Client-1
       switchport access vlan 100
    !

Создаем VNI 10100 и добавляем в него VLAN 100 на Leaf-1 и VLAN 200 на Leaf-2.

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

Проверям, что устройства обменялись type 3 маршрутами:

    Leaf-1#show bgp evpn route-type imet
    BGP routing table information for VRF default
    Router identifier 10.1.0.1, local AS number 65000
    Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                        c - Contributing to ECMP, % - Pending BGP convergence
    Origin codes: i - IGP, e - EGP, ? - incomplete
    AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop
    
              Network                Next Hop              Metric  LocPref Weight  Path
     * >      RD: 10.1.0.1:10100 imet 10.0.0.1
                                     -                     -       -       0       i
     * >Ec    RD: 10.1.0.2:10100 imet 10.0.0.2
                                     10.0.0.2              -       100     0       i Or-ID: 10.1.0.2 C-LST: 10.1.1.0
     *  ec    RD: 10.1.0.2:10100 imet 10.0.0.2
                                     10.0.0.2              -       100     0       i Or-ID: 10.1.0.2 C-LST: 10.1.2.0

Видим 2 машрута, так как у нас два пути между Leaf'ми.

При этом type 2 у нас пока нет:

    Leaf-1#show bgp evpn route-type mac-ip
    BGP routing table information for VRF default
    Router identifier 10.1.0.1, local AS number 65000
    Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                        c - Contributing to ECMP, % - Pending BGP convergence
    Origin codes: i - IGP, e - EGP, ? - incomplete
    AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop
    
              Network                Next Hop              Metric  LocPref Weight  Path
    Leaf-1#

### Проверка наличия IP связанности между клиентами

Проверяем наличия IP связанности между клиентами:

```
Client-1> ping 10.4.0.3

84 bytes from 10.4.0.3 icmp_seq=1 ttl=64 time=207.418 ms
84 bytes from 10.4.0.3 icmp_seq=2 ttl=64 time=38.311 ms
84 bytes from 10.4.0.3 icmp_seq=3 ttl=64 time=46.263 ms
84 bytes from 10.4.0.3 icmp_seq=4 ttl=64 time=34.521 ms
84 bytes from 10.4.0.3 icmp_seq=5 ttl=64 time=31.419 ms
```

Так же на Leaf видим, что появились машруты type 2

    Leaf-1#show bgp evpn route-type mac-ip
    BGP routing table information for VRF default
    Router identifier 10.1.0.1, local AS number 65000
    Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                        c - Contributing to ECMP, % - Pending BGP convergence
    Origin codes: i - IGP, e - EGP, ? - incomplete
    AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop
    
              Network                Next Hop              Metric  LocPref Weight  Path
     * >      RD: 10.1.0.1:10100 mac-ip 0050.7966.6806
                                     -                     -       -       0       i
     * >Ec    RD: 10.1.0.2:10100 mac-ip 0050.7966.6807
                                     10.0.0.2              -       100     0       i Or-ID: 10.1.0.2 C-LST: 10.1.2.0
     *  ec    RD: 10.1.0.2:10100 mac-ip 0050.7966.6807
                                     10.0.0.2              -       100     0       i Or-ID: 10.1.0.2 C-LST: 10.1.1.0

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
Leaf-1#show running-config
! Command: show running-config
! device: Leaf-1 (vEOS-lab, EOS-4.29.2F)
!
service routing protocols model multi-agent
!
hostname Leaf-1
!
vlan 100
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
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 100 vni 10100
   vxlan learn-restrict any
!
ip routing
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
Leaf-2#show running-config
! Command: show running-config
! device: Leaf-2 (vEOS-lab, EOS-4.29.2F)
!
service routing protocols model multi-agent
!
hostname Leaf-2
!
vlan 200
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
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 200 vni 10100
   vxlan learn-restrict any
!
ip routing
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
      rd 10.1.0.2:10100
      route-target both 65000:10100
      redistribute learned
   !
   address-family evpn
      neighbor SPINES activate
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
