# Проектирование адресного пространства

### Цели:

1. Собрать схему CLOS
2. Распределить адресное пространство

### Схема сети:

![Network scheme](network_scheme.png)

### Принцип распределения IP адресов:

IP = 10.Dn.Sn.X  
Dn – диапазон в зависимости от номера ЦОДа  
Sn – номер spine  
X – значение по порядку  

IP = 10.**Dn**.Sn.X  
Dn для DCN = [8*(N-1) .. (8*N-1)], где  
N – номер ЦОД (DC), N = [1 .. 32]  

Распределение внутри ЦОДа:  
Lo1 (/16) (8*(N-1))  
Lo2 (/16) (8*(N-1) + 1)  
p2p links (/16) (8*(N-1) + 2)  
резерв (/16) (8*(N-1) + 3)  
services (/16) (8*(N-1) + [4 .. 7])  

##### Пример распределение сетей для DC1:  

Lo1 - 10.**0**.0.0/16  
Lo2 - 10.**1**.0.0/16  
Суммарный для Lo1 и Lo2 – 10.0.0.0/15  

p2p links - 10.**2**.0.0/16  
резерв - 10.**3**.0.0/16  
Суммарный для p2p links и резерва – 10.2.0.0/15  

services -  10.**4**.0.0/16, 10.**5**.0.0/16, 10.**6**.0.0/16, 10.**7**.0.0/16  
Суммарный – 10.4.0.0/14  

##### Пример распределение IP адресов для DC1:  

10.0.1.0/32 – Spine 1 Lo1  
...  
10.0.N.0/32 – Spine N Lo1  

10.1.0.1/32 – Leaf 1 Lo2  
...  
10.1.0.N/32 – Leaf N Lo2  

10.2.2.4/31 – p2p Spine 2 <-> Leaf 3  
10.2.N.2*(M-1)/31 – p2p Spine N <-> Leaf M  
M= [1 .. 128]  

10.4.0.0/14 – services summary  

### IP план:

### Конфигурация на оборудовании Huawei:

<details>
<summary> Spine-1 </summary>

```
#
sysname Spine-1
```

</details>

<details>
<summary> Spine-2 </summary>

</details>

<details>
<summary> Leaf-1 </summary>

</details>

<details>
<summary> Leaf-2 </summary>

</details>

<details>
<summary> Leaf-3 </summary>

</details>

<details>
<summary> Clients 1-4 </summary>

</details>
