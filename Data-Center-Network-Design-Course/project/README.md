# Project

## Проектирование распределенной сети CLOS с поддержкой L2 связности для миграции с классической схемы сети 

(Новиков Александр)

---

Основной целью проекта является проектирование сети в топологии CLOS с использованием VXLAN для последующего перехода с классической трехcтупенчатой топологии.

### Основные задачи проекта
1. Построить топологию приближенную к реальной сетевой инфраструктуре
2. Запланировать распределение IP адресации
3. Настроить underlay сеть 
4. Настроить overlay сеть
5. Обеспечить отказоустойчивость на уровне подключения серверов/серверных корзин
6. Обеспечить L2 связность в рамках одного L2VNI
7. Обеспечить L3 cвязность в рамках разных L2VNI через L3VNI
8. Обеспечить возможность управлять сетевой доступностью сервисов в разных сегментах через внешнее устройство Firewall
9. Реализовать L2 связность между сайтами в разных локациях

### Текущая топология в общем виде
![Текущая топология](https://github.com/anovikov666/otus_learning/blob/main/Data-Center-Network-Design-Course/project/images/current_topology.png)

Схема достаточно классическая с совмещенным уровнем ядра и агрегации, сервера и серверные корзины подключаются либо непосредственно в ядро, либо через промежуточные коммутаторы. Терминация L3 на уровне коммутаторов ядра. Между сайтами есть растянутые L2 линки для обеспечения миграции виртуальных машин и сервисов, для которых нет возможности обеспечить кластеризацию в разных широковещательных сегментах (такие, к сожалению, имеются). Между несколько выделенных оптических волокон и DWDM.
#### Минусы топологии
1. Растянутый между площадками L2 - первый и очевиднейший минус, большой шанс широковещательного шторма в случае петли, отключение избыточных линков.
2. Высокая терминация L3 - терминация L3 на коммутаторах ядра также увеличивает шанс широковещательного шторма.
3. Низкая масштабируемость - количестов портов на коммутаторах ядра конечно. Добавление портовой емкости через добавление дополнительных коммутатор в уровень аггрегации.
4. Устаревшее оборудование - узкие uplink, нет поддержки новых технологий, при росте трафика можно упереться в пропускную способность

### Целевая схема топологии сети 
![Текущая топология](https://github.com/anovikov666/otus_learning/blob/main/Data-Center-Network-Design-Course/project/images/target_topology.png)

На текущей схеме по одному BorderLeaf и всего один Router-Firewall, но в последствии BorderLeaf будет по 2 на каждой площадке, также на DC2 будет свой роутер. Для требуемых задач по тестированию VXLAN фабрики этого будет достаточно.

#### Плюсы топологии CLOS
1. Низкая терминация L3 - Весь L2 заканчивается на Leaf коммутаторах, минимальный шанс широковещательного шторма
2. Нет растянутого между сайтами L2 - вся L2 связность осуществляется через VXLAN, сервисы, которым необходима L2 связность, продолжат работать без угрозы широквещательного шторма
3. Легкая масштабируемость - в случае источения портовой емкости или высокой утилизации аплинков необходимо просто добавить коммутатор либо на уровень LEAF, либо на уровень SPINE.
4. Новое оборудование с более широкими портами под возрастающие нагрузки
#### Минусы топологии
1. Высокая стоимость обородования для создания VXLAN фабрики
2. Конфигурация коммутаторов в случае их большого количества требует автоматизации
3. Большое количество протоколов добавляет сложности в траблшутинге и поддержке
4. Необходимы компетенции в развертывании топологии CLOS

Тестовый стенд развернут на платформе GNS3 на оборудовании Cisco Nexus

##### Планирование адресного пространства фабрики
##### Site1(DC1)

Лупбеки LEAFs резерв 10.0.0.0/24 - из этого диапазона также будем брать secondary для работы VPC

Лупбеки SPINEs резерв 10.0.1.0/24

Пиринги LEAFs резерв 10.10.0.0/16 - для удобства резервируем большую сеть /16

Пиринги в VRF для Firewall резерв 172.16.0.0 - 172.168.7.255

Loopback LEAFS

- DC1-Lo-Leaf1 - ```10.0.0.1/32 (secondary 10.0.0.250)```
- DC1-Lo-Leaf2 - ```10.0.0.2/32 (secondary 10.0.0.250)```
- DC1-Lo-Leaf3 - ```10.0.0.3/32```
- DC1-Lo-Leaf4 - ```10.0.0.4/32```
- DC1-Lo-BLeaf1 - ```10.0.0.5/32```

Loopback SPINES
- DC1-Lo-Spine1 - ```10.0.1.1/32```
- DC1-Lo-Spine2 - ```10.0.1.2/32```


Peering
- DC1-Leaf1-Spine1 - ```10.10.1.0/31```
- DC1-Leaf1-Spine2 - ```10.10.2.0/31```
- DC1-Leaf2-Spine1 - ```10.10.1.2/31```
- DC1-Leaf2-Spine2 - ```10.10.2.2/31```
- DC1-Leaf3-Spine1 - ```10.10.1.4/31```
- DC1-Leaf3-Spine2 - ```10.10.2.4/31```
- DC1-Leaf4-Spine1 - ```10.10.1.6/31```
- DC1-Leaf4-Spine2 - ```10.10.2.6/31```
- DC1-BLeaf1-Spine1 - ```10.10.1.8/31```
- DC1-Bleaf1-Spine2 - ```10.10.2.8/31```

##### Site1(DC2)

Лупбеки LEAFs резерв 10.0.2.0/24 - из этого диапазона также будем брать secondary для работы VPC

Лупбеки SPINEs резерв 10.0.3.0/24

Пиринги LEAFs резерв 10.11.0.0/16 - для удобства резервируем большую сеть /16

Пиринги в VRF для Firewall резерв 172.16.8.0 - 172.168.15.255

Loopback LEAFS
- DC2-Lo-Leaf1 - ```10.0.2.1/32```
- DC2-Lo-Leaf2 - ```10.0.2.2/32```
- DC2-Lo-Spine1 - ```10.0.3.1/32```
- DC2-Lo-Spine2 - ```10.0.3.2/32```
- DC2-Lo-Bleaf1 - ```10.0.2.3/32```
- 
Peering
- DC2-Leaf1-Spine1 - ```10.11.1.0/31```
- DC2-Leaf1-Spine2 - ```10.11.2.0/31```
- DC2-Leaf2-Spine1 - ```10.11.1.2/31```
- DC2-Leaf2-Spine2 - ```10.11.2.2/31```
- DC2-Bleaf1-Spine1 - ```10.11.1.4/31```
- DC2-Bleaf1-Spine2 - ```10.11.2.4/31```

На всех коммутаторах настраиваем Loopback адреса и пиринги в соответствии с планом и обеспечиваем связность по пиринговым линкам
``` Text
Вывод с Leaf1

DC1-LEAF1# show ip int br

IP Interface Status for VRF "default"(1)
Interface            IP Address      Interface Status
Lo0                  10.0.0.1        protocol-up/link-up/admin-up
Eth1/1               10.10.1.0       protocol-up/link-up/admin-up
Eth1/2               10.10.2.0       protocol-up/link-up/admin-up
```

``` Text
Вывод с Spine1

DC1-SPINE1# show ip int br

IP Interface Status for VRF "default"(1)
Interface            IP Address      Interface Status
Lo0                  10.0.1.1        protocol-up/link-up/admin-up
Eth1/1               10.10.1.1       protocol-up/link-up/admin-up
Eth1/2               10.10.1.3       protocol-up/link-up/admin-up
Eth1/3               10.10.1.5       protocol-up/link-up/admin-up
Eth1/4               10.10.1.7       protocol-up/link-up/admin-up
Eth1/5               10.10.1.9       protocol-up/link-up/admin-up
```

```Text
Убедимся, что связность есть пропингуем пиринговый адрес LEEAF1 со SPINE1
DC1-SPINE1# ping 10.10.1.0
PING 10.10.1.0 (10.10.1.0): 56 data bytes
64 bytes from 10.10.1.0: icmp_seq=0 ttl=254 time=5.094 ms
64 bytes from 10.10.1.0: icmp_seq=1 ttl=254 time=4.029 ms
64 bytes from 10.10.1.0: icmp_seq=2 ttl=254 time=3.527 ms
64 bytes from 10.10.1.0: icmp_seq=3 ttl=254 time=6.159 ms
64 bytes from 10.10.1.0: icmp_seq=4 ttl=254 time=3.61 ms
```

Проверяйм что все 4 Leaf доступны со SPINE коммутаторов на первом сайте, и то же самое делаем на второй.

После обеспечения связности необходимо собрать UNDERLAY, который будет анонсировать наши Loopback адреса.

На первом сайте в качестве UNDERLAY используем ISIS c L1 линками, на второй OSPF c одной area

Необходимо включить feature ISIS и OSPF на коммутаторах и запустить процессы ISIS и OSPF 

Начнем с DC1 включаем ISIS, запускаем процесс выделяем уникальный net и включаем ISIS на Loopback адресах и пиринговых адресах

Пример настройки на SPINE1
```Text
feature isis
router isis UNDERLAY
  net 49.0010.0000.0001.0000.00
interface loopback0
  ip address 10.0.1.1/32
  ip router isis UNDERLAY

interface Ethernet1/1
  description to-eth1-leaf1
  no switchport
  ip address 10.10.1.1/31
  isis network point-to-point
  isis circuit-type level-1
  ip router isis UNDERLAY
  no shutdown

interface Ethernet1/2
  description to-eth1-leaf2
  no switchport
  ip address 10.10.1.3/31
  isis network point-to-point
  isis circuit-type level-1
  ip router isis UNDERLAY
  no shutdown
```

Настраиваем на всех коммутаторах и проверяем, что ISIS соседство установилось на всех коммутаторах и  наши лупбеки доступны

Вывод со SPINE1 и SPINE2

```Text
DC1-SPINE1# show isis adjacency
IS-IS process: UNDERLAY VRF: default
IS-IS adjacency database:
Legend: '!': No AF level connectivity in given topology
System ID       SNPA            Level  State  Hold Time  Interface
DC1-LEAF1       N/A             1      UP     00:00:32   Ethernet1/1
DC1-LEAF2       N/A             1      UP     00:00:32   Ethernet1/2
DC1-LEAF3       N/A             1      UP     00:00:27   Ethernet1/3
DC1-LEAF4       N/A             1      UP     00:00:23   Ethernet1/4

DC1-SPINE2# show isis adjacency
IS-IS process: UNDERLAY VRF: default
IS-IS adjacency database:
Legend: '!': No AF level connectivity in given topology
System ID       SNPA            Level  State  Hold Time  Interface
DC1-LEAF1       N/A             1      UP     00:00:30   Ethernet1/1
DC1-LEAF2       N/A             1      UP     00:00:25   Ethernet1/2
DC1-LEAF3       N/A             1      UP     00:00:31   Ethernet1/3
DC1-LEAF4       N/A             1      UP     00:00:30   Ethernet1/4
```
Проверяем анонсируюется ли наши лупбеки

```Text
DC1-SPINE2# show ip route isis-UNDERLAY
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.0.0.1/32, ubest/mbest: 1/0
    *via 10.10.2.0, Eth1/1, [115/41], 1w0d, isis-UNDERLAY, L1
10.0.0.2/32, ubest/mbest: 1/0
    *via 10.10.2.2, Eth1/2, [115/41], 1w0d, isis-UNDERLAY, L1
10.0.0.3/32, ubest/mbest: 1/0
    *via 10.10.2.4, Eth1/3, [115/41], 1w0d, isis-UNDERLAY, L1
10.0.0.4/32, ubest/mbest: 1/0
    *via 10.10.2.6, Eth1/4, [115/41], 1w0d, isis-UNDERLAY, L1
10.0.0.250/32, ubest/mbest: 2/0
    *via 10.10.2.0, Eth1/1, [115/41], 1w0d, isis-UNDERLAY, L1
    *via 10.10.2.2, Eth1/2, [115/41], 1w0d, isis-UNDERLAY, L1
```

Все лупбеки доступны, переходим к DC2. Настраиваем OSPF и проверяем доступность

Пример настройки на SPINE1
```Text
feature ospf
router ospf UNDERLAY
  router-id 10.0.3.1
interface loopback0
  ip address 10.0.3.1/32
  ip router ospf UNDERLAY area 0.0.0.0

interface Ethernet1/1
  description to-eth1-leaf1
  ip address 10.11.1.1/31
  ip ospf network point-to-point
  ip router ospf UNDERLAY area 0.0.0.0
  no shutdown
```
Проверяем доступность, видим что все доступно.

Можно переходить к настройки OVERLAY 





