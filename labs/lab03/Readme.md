# Настройка ISIS для Underlay сети

### Цели

1. Настройка IP адресации на топологии CLOS
2. Настройка ISIS для Underlay сети на оборудовании Huawei
3. Настройка BFD для ISIS
4. Проверка наличия IP связанности между устройствами в ISIS домене

### Схема сети

![Network scheme](network_scheme3.png)

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

### Настройка ISIS

Поднимаем ISIS на устройстве и анонсируем подсеть Lo1 в OSPF домен. В качестве router id используем IP с интерфейса Loopback1.

Пример настройки:

    router id 10.0.0.1
    #
    ospf 1
      suppress-reachability
      area 0.0.0.0
        network 10.0.0.0 0.0.255.255

*suppress-reachability* - убирает анонсы p2p IP адресации из OSPF домена.

Для топологии CLOS хорошо подходит OSPF в режиме работы *point-to-point*. 

*point-to-point* - Используется для указания двухточечного типа сети, когда один канал соединяет только два маршрутизатора. И поэтому нет необходимости в ручном конфигурировании соседних устройств. Для каждого логического канала при реализации двухточечных соединений используется своя подсеть. В данном типе сети OSPF выборы назначенных маршрутизаторов не осуществляются, а установка отношений смежности осуществляется автоматически с помощью периодической рассылки hello пакетов. Данный тип сети применяется, когда требуется установить отношения смежности только между маршрутизаторами.
 
Поднимаем OSPF на всех интерфейсах Leaf <-> Spine.

Пример настройки интерфейса:

    interface GE1/0/0
      ospf network-type p2p
      ospf enable 1 area 0.0.0.0

### Проверка наличия IP связанности

Для примера проверим работу OSPF и IP связность на устройстве Leaf-1

Проверям наличие OSPF соседей:

```
<Leaf-1>display ospf peer brief
OSPF Process 1 with Router ID 10.0.0.1
                   Peer Statistic Information
Total number of peer(s): 2
 Peer(s) in full state: 2
-----------------------------------------------------------------------------
 Area Id         Interface                  Neighbor id          State
 0.0.0.0         GE1/0/0                    10.0.1.0             Full
 0.0.0.0         GE1/0/1                    10.0.2.0             Full
-----------------------------------------------------------------------------
```

Проверяем наличие необходимых машрутов в таблице маршрутизации:

```
<Leaf-1>display ip routing-table | include OSPF
Proto: Protocol        Pre: Preference
Route Flags: R - relay, D - download to fib, T - to vpn-instance, B - black hole route
------------------------------------------------------------------------------
Routing Table : _public_
         Destinations : 17       Routes : 19

Destination/Mask    Proto   Pre  Cost        Flags NextHop         Interface

       10.0.0.2/32  OSPF    10   2             D   10.2.2.0        GE1/0/1
                    OSPF    10   2             D   10.2.1.0        GE1/0/0
       10.0.0.3/32  OSPF    10   2             D   10.2.2.0        GE1/0/1
                    OSPF    10   2             D   10.2.1.0        GE1/0/0
       10.0.1.0/32  OSPF    10   1             D   10.2.1.0        GE1/0/0
       10.0.2.0/32  OSPF    10   1             D   10.2.2.0        GE1/0/1
```

Проверяем доступность Spin'ов:

```
<Leaf-1>ping -a 10.0.0.1 10.0.1.0
  PING 10.0.1.0: 56  data bytes, press CTRL_C to break
    Reply from 10.0.1.0: bytes=56 Sequence=1 ttl=255 time=34 ms
    Reply from 10.0.1.0: bytes=56 Sequence=2 ttl=255 time=4 ms
    Reply from 10.0.1.0: bytes=56 Sequence=3 ttl=255 time=4 ms
    Reply from 10.0.1.0: bytes=56 Sequence=4 ttl=255 time=4 ms
    Reply from 10.0.1.0: bytes=56 Sequence=5 ttl=255 time=3 ms

  --- 10.0.1.0 ping statistics ---
    5 packet(s) transmitted
    5 packet(s) received
    0.00% packet loss
    round-trip min/avg/max = 3/9/34 ms

<Leaf-1>ping -a 10.0.0.1 10.0.2.0
  PING 10.0.2.0: 56  data bytes, press CTRL_C to break
    Reply from 10.0.2.0: bytes=56 Sequence=1 ttl=255 time=21 ms
    Reply from 10.0.2.0: bytes=56 Sequence=2 ttl=255 time=4 ms
    Reply from 10.0.2.0: bytes=56 Sequence=3 ttl=255 time=4 ms
    Reply from 10.0.2.0: bytes=56 Sequence=4 ttl=255 time=6 ms
    Reply from 10.0.2.0: bytes=56 Sequence=5 ttl=255 time=4 ms

  --- 10.0.2.0 ping statistics ---
    5 packet(s) transmitted
    5 packet(s) received
    0.00% packet loss
    round-trip min/avg/max = 4/7/21 ms
```

Проверяем доступность Leaf'ов:

```
<Leaf-1>
<Leaf-1>ping -a 10.0.0.1 10.0.0.2
  PING 10.0.0.2: 56  data bytes, press CTRL_C to break
    Reply from 10.0.0.2: bytes=56 Sequence=1 ttl=254 time=24 ms
    Reply from 10.0.0.2: bytes=56 Sequence=2 ttl=254 time=6 ms
    Reply from 10.0.0.2: bytes=56 Sequence=3 ttl=254 time=7 ms
    Reply from 10.0.0.2: bytes=56 Sequence=4 ttl=254 time=7 ms
    Reply from 10.0.0.2: bytes=56 Sequence=5 ttl=254 time=5 ms

  --- 10.0.0.2 ping statistics ---
    5 packet(s) transmitted
    5 packet(s) received
    0.00% packet loss
    round-trip min/avg/max = 5/9/24 ms

<Leaf-1>ping -a 10.0.0.1 10.0.0.3
  PING 10.0.0.3: 56  data bytes, press CTRL_C to break
    Reply from 10.0.0.3: bytes=56 Sequence=1 ttl=254 time=26 ms
    Reply from 10.0.0.3: bytes=56 Sequence=2 ttl=254 time=7 ms
    Reply from 10.0.0.3: bytes=56 Sequence=3 ttl=254 time=6 ms
    Reply from 10.0.0.3: bytes=56 Sequence=4 ttl=254 time=9 ms
    Reply from 10.0.0.3: bytes=56 Sequence=5 ttl=254 time=6 ms

  --- 10.0.0.3 ping statistics ---
    5 packet(s) transmitted
    5 packet(s) received
    0.00% packet loss
    round-trip min/avg/max = 6/10/26 ms
```

### Конфигурация на оборудовании Huawei

<details>
<summary> Spine-1 </summary>

```
<Spine-1>display current-configuration
!Software Version V200R005C10SPC607B607
!Last configuration was updated at 2024-05-29 19:32:26+00:00
!Last configuration was saved at 2024-05-29 19:34:30+00:00
#
sysname Spine-1
#
router id 10.0.1.0
#
interface GE1/0/0
 undo portswitch
 description to Leaf-1
 undo shutdown
 ip address 10.2.1.0 255.255.255.254
 ospf network-type p2p
 ospf enable 1 area 0.0.0.0
#
interface GE1/0/1
 undo portswitch
 description to Leaf-2
 undo shutdown
 ip address 10.2.1.2 255.255.255.254
 ospf network-type p2p
 ospf enable 1 area 0.0.0.0
#
interface GE1/0/2
 undo portswitch
 description to Leaf-3
 undo shutdown
 ip address 10.2.1.4 255.255.255.254
 ospf network-type p2p
 ospf enable 1 area 0.0.0.0
#
interface LoopBack1
 description Underlay
 ip address 10.0.1.0 255.255.255.255
#
interface LoopBack2
 description Overlay
 ip address 10.1.1.0 255.255.255.255
#
ospf 1
 suppress-reachability
 area 0.0.0.0
  network 10.0.0.0 0.0.255.255
#
```

</details>

<details>
<summary> Spine-2 </summary>

```
<Spine-2>display current-configuration
!Software Version V200R005C10SPC607B607
!Last configuration was updated at 2024-05-29 19:31:18+00:00
!Last configuration was saved at 2024-05-29 19:33:46+00:00
#
sysname Spine-2
#
router id 10.0.2.0
#
interface GE1/0/0
 undo portswitch
 description to Leaf-1
 undo shutdown
 ip address 10.2.2.0 255.255.255.254
 ospf network-type p2p
 ospf enable 1 area 0.0.0.0
#
interface GE1/0/1
 undo portswitch
 description to Leaf-2
 undo shutdown
 ip address 10.2.2.2 255.255.255.254
 ospf network-type p2p
 ospf enable 1 area 0.0.0.0
#
interface GE1/0/2
 undo portswitch
 description to Leaf-3
 undo shutdown
 ip address 10.2.2.4 255.255.255.254
 ospf network-type p2p
 ospf enable 1 area 0.0.0.0
#
interface LoopBack1
 description Underlay
 ip address 10.0.2.0 255.255.255.255
#
interface LoopBack2
 description Overlay
 ip address 10.1.2.0 255.255.255.255
#
ospf 1
 suppress-reachability
 area 0.0.0.0
  network 10.0.0.0 0.0.255.255
#
```

</details>

<details>
<summary> Leaf-1 </summary>

```
<Leaf-1>display current-configuration
!Software Version V200R005C10SPC607B607
!Last configuration was updated at 2024-05-29 19:29:38+00:00
!Last configuration was saved at 2024-05-29 19:49:42+00:00
#
sysname Leaf-1
#
router id 10.0.0.1
#
interface GE1/0/0
 undo portswitch
 description to Spine-1
 undo shutdown
 ip address 10.2.1.1 255.255.255.254
 ospf network-type p2p
 ospf enable 1 area 0.0.0.0
#
interface GE1/0/1
 undo portswitch
 description to Spine-2
 undo shutdown
 ip address 10.2.2.1 255.255.255.254
 ospf network-type p2p
 ospf enable 1 area 0.0.0.0
#
interface GE1/0/9
 undo portswitch
 description to Client-1
 undo shutdown
 ip address 10.4.0.1 255.255.255.192
#
interface LoopBack1
 description Underlay
 ip address 10.0.0.1 255.255.255.255
#
interface LoopBack2
 description Overlay
 ip address 10.1.0.1 255.255.255.255
#
ospf 1
 suppress-reachability
 area 0.0.0.0
  network 10.0.0.0 0.0.255.255
#
```

</details>

<details>
<summary> Leaf-2 </summary>

```
<Leaf-2>display current-configuration
!Software Version V200R005C10SPC607B607
!Last configuration was updated at 2024-05-29 19:35:59+00:00
!Last configuration was saved at 2024-05-29 19:38:00+00:00
#
sysname Leaf-2
#
router id 10.0.0.2
#
interface GE1/0/0
 undo portswitch
 description to Spine-1
 undo shutdown
 ip address 10.2.1.3 255.255.255.254
 ospf network-type p2p
 ospf enable 1 area 0.0.0.0
#
interface GE1/0/1
 undo portswitch
 description to Spine-2
 undo shutdown
 ip address 10.2.2.3 255.255.255.254
 ospf network-type p2p
 ospf enable 1 area 0.0.0.0
#
interface GE1/0/9
 undo portswitch
 description to Client-2
 undo shutdown
 ip address 10.4.0.65 255.255.255.192
#
interface LoopBack1
 description Underlay
 ip address 10.0.0.2 255.255.255.255
#
interface LoopBack2
 description Overlay
 ip address 10.1.0.2 255.255.255.255
#
ospf 1
 suppress-reachability
 area 0.0.0.0
  network 10.0.0.0 0.0.255.255
#
```

</details>

<details>
<summary> Leaf-3 </summary>

```
<Leaf-3>display current-configuration
!Software Version V200R005C10SPC607B607
!Last configuration was updated at 2024-05-29 19:39:44+00:00
!Last configuration was saved at 2024-05-29 19:40:14+00:00
#
sysname Leaf-3
#
router id 10.0.0.3
#
interface GE1/0/0
 undo portswitch
 description to Spine-1
 undo shutdown
 ip address 10.2.1.5 255.255.255.254
 ospf network-type p2p
 ospf enable 1 area 0.0.0.0
#
interface GE1/0/1
 undo portswitch
 description to Spine-2
 undo shutdown
 ip address 10.2.2.5 255.255.255.254
 ospf network-type p2p
 ospf enable 1 area 0.0.0.0
#
interface GE1/0/8
 undo portswitch
 description to Client-3
 undo shutdown
 ip address 10.4.0.129 255.255.255.192
#
interface GE1/0/9
 undo portswitch
 description to Client-4
 undo shutdown
 ip address 10.4.0.193 255.255.255.192
#
interface LoopBack1
 description Underlay
 ip address 10.0.0.3 255.255.255.255
#
interface LoopBack2
 description Overlay
 ip address 10.1.0.3 255.255.255.255
#
ospf 1
 suppress-reachability
 area 0.0.0.0
  network 10.0.0.0 0.0.255.255
#
```

</details>
