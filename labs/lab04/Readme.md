# Настройка iBGP для Underlay сети

### Цели

1. Настройка IP адресации на топологии CLOS
2. Настройка iBGP для Underlay сети на оборудовании Huawei
3. Тюнинг iBGP
4. Проверка наличия IP связанности между устройствами в iBGP домене

### Схема сети

![Network scheme](../lab02/network_scheme2.png)

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
||GE1/0/0|10.2.1.1|255.255.255.254
||GE1/0/1|10.2.2.1|255.255.255.254
||GE1/0/9|10.4.0.1|255.255.255.192
Leaf-2|Lo1|10.0.0.2|255.255.255.255
||Lo2|10.1.0.2|255.255.255.255
||GE1/0/0|10.2.1.3|255.255.255.254
||GE1/0/1|10.2.2.3|255.255.255.254
||GE1/0/9|10.4.0.65|255.255.255.192
Leaf-3|Lo1|10.0.0.3|255.255.255.255
||Lo2|10.1.0.3|255.255.255.255
||GE1/0/0|10.2.1.5|255.255.255.254
||GE1/0/1|10.2.2.5|255.255.255.254
||GE1/0/8|10.4.0.129|255.255.255.192
||GE1/0/9|10.4.0.193|255.255.255.192
Client-1|eth0|10.4.0.2|255.255.255.192
Client-2|eth0|10.4.0.66|255.255.255.192
Client-3|eth0|10.4.0.130|255.255.255.192
Client-4|eth0|10.4.0.194|255.255.255.192

### Настройка IP адресации

Настраиваем IP адресацию на всех интерфейсах согласно IP плана и схемы сети.

Пример настройки для физического интерфейса:

    interface GE1/0/0
      undo portswitch
      description to Leaf-1
      undo shutdown
      ip address 10.2.1.0 255.255.255.254

Пример настройки для Loopback интерфейса:

    interface LoopBack1
      description Underlay
      ip address 10.0.0.1 255.255.255.255

### Настройка iBGP

Поднимаем iBGP на устройстве. В качестве router id используем IP с интерфейса Loopback1.

Пример настройки:

    bgp 65000
     router-id 10.0.0.1

Для минимизации настроек будем использовать BGP Peer Group при настройке BGP соседей. BGP Peer Group позволяет задавать единые настройки для всех соседей в группе. При этом на Spine лучше использовать Dynamic BGP Peer Group, что позволеят не менять настройки Spine при добавление новых Leaf. Но к сожалению тестовый стенд не позволяет настроить динамические группы, поэтому будем использовать обычные группы как на Leaf, так и на Spine.
 
Пример настройки iBGP соседства:

```
 group SPINES internal
 peer 10.2.1.0 as-number 65000
 peer 10.2.1.0 group SPINES
 peer 10.2.2.0 as-number 65000
 peer 10.2.2.0 group SPINES
 #
 ipv4-family unicast
  peer SPINES enable
  peer 10.2.1.0 enable
  peer 10.2.1.0 group SPINES
  peer 10.2.2.0 enable
  peer 10.2.2.0 group SPINES
```

Анонсируем IP-адреса Lo1 интрефесов и стыковочные p2p-сети в BGP:

      network 10.0.0.1 255.255.255.255
      network 10.2.1.0 255.255.255.254
      network 10.2.2.0 255.255.255.254

Так как у нас нет связей между Leaf'ами, то необходимо на Spine настроить механизм Route Reflector. Это повзолит Leaf'ам получать все маршруты с других Leaf. 

Пример настройки на Spine:

    bgp 65000
      #
      ipv4-family unicast
        peer LEAVES reflect-client

### Тюнинг iBGP

Для уменьшения времени сходимости уменьшим значения таймеров:

    bgp 65000
     peer LEAVES timer keepalive 3 hold 9

*Таймеры достаточно настроить только на одной из сторон. Для минимизации настроек сделем это на Spine'ах*

И настроим BFD:

    bfd
    #
    bgp 65000
      peer LEAVES bfd enable

*К сожалению после настрйоки BFD тестовый стенд начинает вести себя некорректно, поэтому в итоговой конфигурации от BFD пришлось отказаться.*

На топологии CLOS отличной идей будет настройка BGP Load Balancing для балансировки нагрузки между интерфейсами и Spin'ами.

Пример настройки BGP Load Balancing:

```
bgp 65000
 #
 ipv4-family unicast
  maximum load-balancing ibgp 2
```

### Проверка наличия IP связанности

Для примера проверим работу iBGP и IP связность на устройстве Leaf-1

Проверям наличие iBGP соседей:

```
<Leaf-1>display bgp peer
 BGP local router ID        : 10.0.0.1
 Local AS number            : 65000
 Total number of peers      : 2
 Peers in established state : 2

  Peer            V          AS  MsgRcvd  MsgSent  OutQ  Up/Down       State  PrefRcv
  10.2.1.0        4       65000     6236     6215     0 11:00:15 Established        8
  10.2.2.0        4       65000     4347     4348     0 02:27:44 Established        8
```

Проверяем наличие необходимых машрутов в таблице маршрутизации:

```
<Leaf-1>display ip routing-table | include BGP
Proto: Protocol        Pre: Preference
Route Flags: R - relay, D - download to fib, T - to vpn-instance, B - black hole route
------------------------------------------------------------------------------
Routing Table : _public_
         Destinations : 21       Routes : 27

Destination/Mask    Proto   Pre  Cost        Flags NextHop         Interface

       10.0.0.2/32  IBGP    255  0             RD  10.2.1.3        GE1/0/0
                    IBGP    255  0             RD  10.2.2.3        GE1/0/1
       10.0.0.3/32  IBGP    255  0             RD  10.2.1.5        GE1/0/0
                    IBGP    255  0             RD  10.2.2.5        GE1/0/1
       10.0.1.0/32  IBGP    255  0             RD  10.2.1.0        GE1/0/0
       10.0.2.0/32  IBGP    255  0             RD  10.2.2.0        GE1/0/1
       10.2.1.2/31  IBGP    255  0             RD  10.2.1.0        GE1/0/0
                    IBGP    255  0             RD  10.2.2.3        GE1/0/1
       10.2.1.4/31  IBGP    255  0             RD  10.2.1.0        GE1/0/0
                    IBGP    255  0             RD  10.2.2.5        GE1/0/1
       10.2.2.2/31  IBGP    255  0             RD  10.2.2.0        GE1/0/1
                    IBGP    255  0             RD  10.2.1.3        GE1/0/0
       10.2.2.4/31  IBGP    255  0             RD  10.2.2.0        GE1/0/1
                    IBGP    255  0             RD  10.2.1.5        GE1/0/0
```

Проверяем доступность Spin'ов:

```
<Leaf-1>ping 10.0.1.0
  PING 10.0.1.0: 56  data bytes, press CTRL_C to break
    Reply from 10.0.1.0: bytes=56 Sequence=1 ttl=255 time=30 ms
    Reply from 10.0.1.0: bytes=56 Sequence=2 ttl=255 time=4 ms
    Reply from 10.0.1.0: bytes=56 Sequence=3 ttl=255 time=5 ms
    Reply from 10.0.1.0: bytes=56 Sequence=4 ttl=255 time=5 ms
    Reply from 10.0.1.0: bytes=56 Sequence=5 ttl=255 time=6 ms

  --- 10.0.1.0 ping statistics ---
    5 packet(s) transmitted
    5 packet(s) received
    0.00% packet loss
    round-trip min/avg/max = 4/10/30 ms

<Leaf-1>ping 10.0.2.0
  PING 10.0.2.0: 56  data bytes, press CTRL_C to break
    Reply from 10.0.2.0: bytes=56 Sequence=1 ttl=255 time=18 ms
    Reply from 10.0.2.0: bytes=56 Sequence=2 ttl=255 time=4 ms
    Reply from 10.0.2.0: bytes=56 Sequence=3 ttl=255 time=4 ms
    Reply from 10.0.2.0: bytes=56 Sequence=4 ttl=255 time=3 ms
    Reply from 10.0.2.0: bytes=56 Sequence=5 ttl=255 time=7 ms

  --- 10.0.2.0 ping statistics ---
    5 packet(s) transmitted
    5 packet(s) received
    0.00% packet loss
    round-trip min/avg/max = 3/7/18 ms
```

Проверяем доступность Leaf'ов:

```
<Leaf-1>ping 10.0.0.2
  PING 10.0.0.2: 56  data bytes, press CTRL_C to break
    Reply from 10.0.0.2: bytes=56 Sequence=1 ttl=254 time=33 ms
    Reply from 10.0.0.2: bytes=56 Sequence=2 ttl=254 time=7 ms
    Reply from 10.0.0.2: bytes=56 Sequence=3 ttl=254 time=7 ms
    Reply from 10.0.0.2: bytes=56 Sequence=4 ttl=254 time=7 ms
    Reply from 10.0.0.2: bytes=56 Sequence=5 ttl=254 time=6 ms

  --- 10.0.0.2 ping statistics ---
    5 packet(s) transmitted
    5 packet(s) received
    0.00% packet loss
    round-trip min/avg/max = 6/12/33 ms

<Leaf-1>ping 10.0.0.3
  PING 10.0.0.3: 56  data bytes, press CTRL_C to break
    Reply from 10.0.0.3: bytes=56 Sequence=1 ttl=254 time=17 ms
    Reply from 10.0.0.3: bytes=56 Sequence=2 ttl=254 time=7 ms
    Reply from 10.0.0.3: bytes=56 Sequence=3 ttl=254 time=8 ms
    Reply from 10.0.0.3: bytes=56 Sequence=4 ttl=254 time=9 ms
    Reply from 10.0.0.3: bytes=56 Sequence=5 ttl=254 time=7 ms

  --- 10.0.0.3 ping statistics ---
    5 packet(s) transmitted
    5 packet(s) received
    0.00% packet loss
    round-trip min/avg/max = 7/9/17 ms
```

### Конфигурация на оборудовании Huawei

<details>
<summary> Spine-1 </summary>

```
<Spine-1>display current-configuration
!Software Version V200R005C10SPC607B607
!Last configuration was updated at 2024-06-25 20:13:03+00:00
!Last configuration was saved at 2024-06-25 20:19:30+00:00
#
sysname Spine-1
#
interface GE1/0/0
 undo portswitch
 description to Leaf-1
 undo shutdown
 ip address 10.2.1.0 255.255.255.254
#
interface GE1/0/1
 undo portswitch
 description to Leaf-2
 undo shutdown
 ip address 10.2.1.2 255.255.255.254
#
interface GE1/0/2
 undo portswitch
 description to Leaf-3
 undo shutdown
 ip address 10.2.1.4 255.255.255.254
#
interface LoopBack1
 description Underlay
 ip address 10.0.1.0 255.255.255.255
#
interface LoopBack2
 description Overlay
 ip address 10.1.1.0 255.255.255.255
#
bgp 65000
 router-id 10.0.1.0
 group LEAVES internal
 peer LEAVES timer keepalive 3 hold 9
 peer 10.2.1.1 as-number 65000
 peer 10.2.1.1 group LEAVES
 peer 10.2.1.3 as-number 65000
 peer 10.2.1.3 group LEAVES
 peer 10.2.1.5 as-number 65000
 peer 10.2.1.5 group LEAVES
 #
 ipv4-family unicast
  network 10.0.1.0 255.255.255.255
  network 10.2.1.0 255.255.255.254
  network 10.2.1.2 255.255.255.254
  network 10.2.1.4 255.255.255.254
  peer LEAVES enable
  peer LEAVES reflect-client
  peer 10.2.1.1 enable
  peer 10.2.1.1 group LEAVES
  peer 10.2.1.3 enable
  peer 10.2.1.3 group LEAVES
  peer 10.2.1.5 enable
  peer 10.2.1.5 group LEAVES
#
```

</details>

<details>
<summary> Spine-2 </summary>

```
<Spine-2>display current-configuration
!Software Version V200R005C10SPC607B607
!Last configuration was updated at 2024-06-25 20:15:56+00:00
!Last configuration was saved at 2024-06-25 20:19:04+00:00
#
sysname Spine-2
#
interface GE1/0/0
 undo portswitch
 description to Leaf-1
 undo shutdown
 ip address 10.2.2.0 255.255.255.254
#
interface GE1/0/1
 undo portswitch
 description to Leaf-2
 undo shutdown
 ip address 10.2.2.2 255.255.255.254
#
interface GE1/0/2
 undo portswitch
 description to Leaf-3
 undo shutdown
 ip address 10.2.2.4 255.255.255.254
#
interface LoopBack1
 description Underlay
 ip address 10.0.2.0 255.255.255.255
#
interface LoopBack2
 description Overlay
 ip address 10.1.2.0 255.255.255.255
#
bgp 65000
 router-id 10.0.2.0
 group LEAVES internal
 peer LEAVES timer keepalive 3 hold 9
 peer 10.2.2.1 as-number 65000
 peer 10.2.2.1 group LEAVES
 peer 10.2.2.3 as-number 65000
 peer 10.2.2.3 group LEAVES
 peer 10.2.2.5 as-number 65000
 peer 10.2.2.5 group LEAVES
 #
 ipv4-family unicast
  network 10.0.2.0 255.255.255.255
  network 10.2.2.0 255.255.255.254
  network 10.2.2.2 255.255.255.254
  network 10.2.2.4 255.255.255.254
  peer LEAVES enable
  peer LEAVES reflect-client
  peer 10.2.2.1 enable
  peer 10.2.2.1 group LEAVES
  peer 10.2.2.3 enable
  peer 10.2.2.3 group LEAVES
  peer 10.2.2.5 enable
  peer 10.2.2.5 group LEAVES
#
```

</details>

<details>
<summary> Leaf-1 </summary>

```
<Leaf-1>display current-configuration
!Software Version V200R005C10SPC607B607
!Last configuration was updated at 2024-06-25 20:08:33+00:00
!Last configuration was saved at 2024-06-25 20:21:58+00:00
#
sysname Leaf-1
#
interface GE1/0/0
 undo portswitch
 description to Spine-1
 undo shutdown
 ip address 10.2.1.1 255.255.255.254
#
interface GE1/0/1
 undo portswitch
 description to Spine-2
 undo shutdown
 ip address 10.2.2.1 255.255.255.254
#
interface LoopBack1
 description Underlay
 ip address 10.0.0.1 255.255.255.255
#
interface LoopBack2
 description Overlay
 ip address 10.1.0.1 255.255.255.255
#
bgp 65000
 router-id 10.0.0.1
 group SPINES internal
 peer 10.2.1.0 as-number 65000
 peer 10.2.1.0 group SPINES
 peer 10.2.2.0 as-number 65000
 peer 10.2.2.0 group SPINES
 #
 ipv4-family unicast
  network 10.0.0.1 255.255.255.255
  network 10.2.1.0 255.255.255.254
  network 10.2.2.0 255.255.255.254
  maximum load-balancing ibgp 2
  peer SPINES enable
  peer 10.2.1.0 enable
  peer 10.2.1.0 group SPINES
  peer 10.2.2.0 enable
  peer 10.2.2.0 group SPINES
#
```

</details>

<details>
<summary> Leaf-2 </summary>

```
<Leaf-2>display current-configuration
!Software Version V200R005C10SPC607B607
!Last configuration was updated at 2024-06-25 20:20:58+00:00
!Last configuration was saved at 2024-06-25 20:21:14+00:00
#
sysname Leaf-2
#
interface GE1/0/0
 undo portswitch
 description to Spine-1
 undo shutdown
 ip address 10.2.1.3 255.255.255.254
#
interface GE1/0/1
 undo portswitch
 description to Spine-2
 undo shutdown
 ip address 10.2.2.3 255.255.255.254
#
interface LoopBack1
 description Underlay
 ip address 10.0.0.2 255.255.255.255
#
interface LoopBack2
 description Overlay
 ip address 10.1.0.2 255.255.255.255
#
bgp 65000
 router-id 10.0.0.2
 group SPINES internal
 peer 10.2.1.2 as-number 65000
 peer 10.2.1.2 group SPINES
 peer 10.2.2.2 as-number 65000
 peer 10.2.2.2 group SPINES
 #
 ipv4-family unicast
  network 10.0.0.2 255.255.255.255
  network 10.2.1.2 255.255.255.254
  network 10.2.2.2 255.255.255.254
  maximum load-balancing ibgp 2
  peer SPINES enable
  peer 10.2.1.2 enable
  peer 10.2.1.2 group SPINES
  peer 10.2.2.2 enable
  peer 10.2.2.2 group SPINES
```

</details>

<details>
<summary> Leaf-3 </summary>

```
<Leaf-3>display current-configuration
!Software Version V200R005C10SPC607B607
!Last configuration was updated at 2024-06-25 20:20:09+00:00
!Last configuration was saved at 2024-06-25 20:20:38+00:00
#
sysname Leaf-3
#
interface GE1/0/0
 undo portswitch
 description to Spine-1
 undo shutdown
 ip address 10.2.1.5 255.255.255.254
#
interface GE1/0/1
 undo portswitch
 description to Spine-2
 undo shutdown
 ip address 10.2.2.5 255.255.255.254
#
interface LoopBack1
 description Underlay
 ip address 10.0.0.3 255.255.255.255
#
interface LoopBack2
 description Overlay
 ip address 10.1.0.3 255.255.255.255
#
bgp 65000
 router-id 10.0.0.3
 group SPINES internal
 peer 10.2.1.4 as-number 65000
 peer 10.2.1.4 group SPINES
 peer 10.2.2.4 as-number 65000
 peer 10.2.2.4 group SPINES
 #
 ipv4-family unicast
  network 10.0.0.3 255.255.255.255
  network 10.2.1.4 255.255.255.254
  network 10.2.2.4 255.255.255.254
  maximum load-balancing ibgp 2
  peer SPINES enable
  peer 10.2.1.4 enable
  peer 10.2.1.4 group SPINES
  peer 10.2.2.4 enable
  peer 10.2.2.4 group SPINES
```

</details>
