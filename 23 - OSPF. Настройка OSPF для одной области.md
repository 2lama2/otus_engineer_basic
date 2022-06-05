## Настройка протокола OSPFv2 для одной области

#### Топология
![[Pasted image 20220603230022.png]]

![Pics](https://github.com/2lama2/otus_engineer_basic/blob/0918bf9f4066fec686c2c7ce890e01e34f66ee18/pics/Pasted%20image%2020220603230022.png)

#### Таблица адресации
<table>
<thead>
  <tr>
    <th>Устройство</th>
    <th>Интерфейс</th>
    <th>IP-адрес</th>
    <th>Маска подсети</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td rowspan="2">R1</td>
    <td>e0/0</td>
    <td>10.53.0.1</td>
    <td>255.255.255.0</td>
  </tr>
  <tr>
    <td>Loopback1</td>
    <td>172.16.1.1</td>
    <td>255.255.255.0</td>
  </tr>
  <tr>
    <td rowspan="2">R2</td>
    <td>e0/0</td>
    <td>10.53.0.2</td>
    <td>255.255.255.0</td>
  </tr>
  <tr>
    <td>Loopback1</td>
    <td>192.168.1.1</td>
    <td>255.255.255.0</td>
  </tr>
</tbody>
</table>
 

#### Задачи

1. Создание сети и настройка основных параметров устройства
2. Настройка и проверка базовой работы протокола  OSPFv2 для одной области
3. Оптимизация и проверка конфигурации OSPFv2 для одной области

---

### 1. Создание сети и настройка основных параметров устройства
* Создадим топологию сети в EVE-NG.

* Выполним базовую настройку маршрутизаторов и коммутаторов по шаблону:

	```
	HOSTNAME# clock set HH:MM:SS DAY MONTH YEAR
	HOSTNAME# configure terminal
	HOSTNAME(config)# hostname ==HOSTNAME==
	
	HOSTNAME(config)# enable secret ==PASSWORD==
	HOSTNAME(config)# service password-encryption
	
	HOSTNAME(config)# banner motd #Warning! Ye've been warned!#
	
	HOSTNAME(config)# no ip domain-lookup
	
	HOSTNAME(config)# line console 0
	HOSTNAME(config-line)# password ==PASSWORD==
	HOSTNAME(config-line)# login
	
	HOSTNAME(config-line)# logging synchronous
	
	HOSTNAME(config)# line vty 0 4
	HOSTNAME(config-line)# password ==PASSWORD==
	HOSTNAME(config-line)# login
	HOSTNAME(config-line)# transport input telnet
	
	HOSTNAME# copy running-config startup-config
	```

### 2. Настройка и проверка базовой работы протокола  OSPFv2 для одной области
* Настроим адреса интерфейсов на каждом маршрутизаторе по таблице адресации

	```
	R1# sh ip interface brief
	
	Interface                  IP-Address      OK? Method Status                Protocol
	Ethernet0/0                10.53.0.1       YES manual up                    up  
	Ethernet0/1                unassigned      YES unset  administratively down down
	Ethernet0/2                unassigned      YES unset  administratively down down
	Ethernet0/3                unassigned      YES unset  administratively down down
	Loopback1                  172.16.1.1      YES manual up                    up  
	```

	```
	R2# sh ip interface brief
	
	Interface                  IP-Address      OK? Method Status                Protocol
	Ethernet0/0                10.53.0.2       YES manual up                    up  
	Ethernet0/1                unassigned      YES unset  administratively down down
	Ethernet0/2                unassigned      YES unset  administratively down down
	Ethernet0/3                unassigned      YES unset  administratively down down
	Loopback1                  192.168.1.1     YES manual up                    up  
	```

* Настроим процесс OSPF на каждом маршрутизаторе, выбрав идентификатор процесса равным 56. Зададим идентификаторы маршрутизатора для протокола OSPF на каждом маршрутизаторе:

	```
	R1(config)# router ospf 56
	R1(config-router)# router-id
	R1(config-router)# router-id 1.1.1.1
	```

	```
	R2(config)# router ospf 56
	R2(config-router)# router-id
	R2(config-router)# router-id 2.2.2.2
	```

* На обоих маршрутизаторах зададим сеть, которая будет участвовать в работе протокола OSPF:

	```
	R(config-router)# network 10.53.0.0 0.0.0.255 area 0
	```

* Для маршрутизатора R2 дополнительно добавим сеть интерфейса Loopback1 также в облась 0:

	```
	R2(config-router)# network 192.168.1.0 0.0.0.255 area 0
	```

* Для проверки того, что маршрутизаторы начали взаимодействовать по протоколу OSPF, выполним команду `sh ip ospf int brief`, которая показывает какие интерфейсы участвуют в работе протокола OSPF:

	```
	R1# sh ip ospf int brief
	
	Interface    PID   Area            IP Address/Mask    Cost  State Nbrs F/C
	Et0/0        56    0               10.53.0.1/24       10    BDR   1/1
	```

	```
	R2# sh ip ospf int brief
	
	Interface    PID   Area            IP Address/Mask    Cost  State Nbrs F/C
	Lo1          56    0               192.168.1.1/24     1     LOOP  0/0
	Et0/0        56    0               10.53.0.2/24       10    DR    1/1
	```

	а также команду `sh ip ospf neighbor`, которая показывает какие маршрутизаторы учствуют в обмене информации о маршрутизаторах и маршрутах в сети:

	```
	R1# sh ip ospf neighbor
	
	Neighbor ID     Pri   State           Dead Time   Address         Interface
	2.2.2.2           1   FULL/DR         00:00:37    10.53.0.2       Ethernet0/0
	```

	```
	R2# sh ip ospf neighbor
	
	Neighbor ID     Pri   State           Dead Time   Address         Interface
	1.1.1.1           1   FULL/BDR        00:00:34    10.53.0.1       Ethernet0/0
	```

	Из вывода команд видно, что маршрутизатор R2 стал назначенным маршрутизатором (DR). Выбор данного маршрутизатора назначенным обусловлен тем, что у него значение идентификатора маршрутизатора (router-id) больше, чем у маршрутизатора R1 и этот критерий является вторым (первостепенным является приоритет, который по умолчанию равен 1 для обоих маршрутизаторов) в работе алгоритма SPF при выборах ролей DR и BDR для маршрутизаторов.

* Команда `sh ip route ospf` позволяет вывести информацию о маршрутах, полученных маршрутизатором от соседей по протоколу OSPF:

	```
	R1# sh ip route ospf
	......
	      192.168.1.0/32 is subnetted, 1 subnets
	O        192.168.1.1 [110/11] via 10.53.0.2, 00:30:29, Ethernet0/0
	```

	Проверим доступность изученного маршрута:

	```
	R1# traceroute 192.168.1.1
	
	Type escape sequence to abort.
	Tracing the route to 192.168.1.1
	VRF info: (vrf in name/id, vrf out name/id)
	  1 10.53.0.2 1 msec *  1 msec
	```


### 3. Оптимизация и проверка конфигурации OSPFv2 для одной области
* Зададим приоритет маршрутизатору R1 с тем, чтобы он стал назначенным маршрутизатором:

	```
	R1(config)# int e0/0
	R1(config-if)# ip ospf priority 50
	```

* На обоих маршрутизаторах поменяем интервал сообщений приветствия на значение 30 сек:

	```
	R(config-if)# ip ospf hello-interval 30
	```

* Настроим на маршрутизаторе R1 маршрутом по умолчанию интерфейс Loopback1 и включим распространение данного маршрута протоколом OSPF:

	```
	R1(config)# ip route 0.0.0.0 0.0.0.0 loopback 1
	
	R1(config-router)#default-information originate
	```

	```
	R2(config-if)#do sh ip route
	
	Gateway of last resort is 10.53.0.1 to network 0.0.0.0
	
	O*E2  0.0.0.0/0 [110/1] via 10.53.0.1, 00:00:35, Ethernet0/0
	......
	```

* Для маршрутизатора R2 переведем обработку сети на интерфейсе Loopback1 в режим point-to-point:

	```
	R2(config)# int lo1
	R2(config-if)# ip ospf network point-to-point
	```

	```
	R2# sh ip ospf int br
	
	Interface    PID   Area            IP Address/Mask    Cost  State Nbrs F/C
	Lo1          56    0               192.168.1.1/24     1     P2P   0/0
	Et0/0        56    0               10.53.0.2/24       10    BDR   1/1
	```

	и также запретим отправлять объявления OSPF в сеть Loopback1:

	```
	R2(config)# router ospf 56
	R2(config-router)# passive-interface loopback 1
	```

	```
	R2(config-router)# do sh ip pro | b ospf
	
	Routing Protocol is "ospf 56"
	  Outgoing update filter list for all interfaces is not set
	  Incoming update filter list for all interfaces is not set
	  Router ID 2.2.2.2
	  Number of areas in this router is 1. 1 normal 0 stub 0 nssa
	  Maximum path: 4
	  Routing for Networks:
	    10.53.0.0 0.0.0.255 area 0
	    192.168.1.0 0.0.0.255 area 0
	  Passive Interface(s):
	    Loopback1
	  Routing Information Sources:
	    Gateway         Distance      Last Update
	    1.1.1.1              110      00:21:25
	  Distance: (default is 110)
	```

* Изменим базовую пропускную способность для маршрутизаторов.

	Выведем текущую конфигурацию:

	```
	R1# sh int e0/0 | i BW
	
	  MTU 1500 bytes, BW 10000 Kbit/sec, DLY 1000 usec,
	
	R1# sh ip ospf | i bandwidth
	
	 Reference bandwidth unit is 100 mbps
	
	R1# sh ip ospf int br
	
	Interface    PID   Area            IP Address/Mask    Cost  State Nbrs F/C
	Et0/0        56    0               10.53.0.1/24       10    DR    1/1
	```

	При значениях скорости интерфейса 10000 Kbit/sec и базовой полосы 100 mbps, стоимость канала равняется 10.

	Изменим базовую пропускную способность:

	```
	R1(config)# router ospf 56
	R1(config-router)# auto-cost reference-bandwidth 10
	```

	Перезапустим процессы OSPF на маршрутизаторах, чтобы изменения вступили в силу:

	```
	R# clear ip ospf process
	```

	Наблюдаем результат: метрика канала стала равна 1:

	```
	R# sh ip ospf | i bandwidth
	
	 Reference bandwidth unit is 10 mbps
	 
	R1# sh ip ospf int br
	
	Interface    PID   Area            IP Address/Mask    Cost  State Nbrs F/C
	Et0/0        56    0               10.53.0.1/24       1     DR    1/1
	
	R2# sh ip ospf int br
	
	Interface    PID   Area            IP Address/Mask    Cost  State Nbrs F/C
	Lo1          56    0               192.168.1.1/24     1     P2P   0/0
	Et0/0        56    0               10.53.0.2/24       1     BDR   1/1
	```

* Проверим итоговое состояние по таблицам маршрутизации:

	```
	R1# sh ip route
	
	Gateway of last resort is 0.0.0.0 to network 0.0.0.0
	
	S*    0.0.0.0/0 is directly connected, Loopback1
	      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
	C        10.53.0.0/24 is directly connected, Ethernet0/0
	L        10.53.0.1/32 is directly connected, Ethernet0/0
	      172.16.0.0/16 is variably subnetted, 2 subnets, 2 masks
	C        172.16.1.0/24 is directly connected, Loopback1
	L        172.16.1.1/32 is directly connected, Loopback1
	O     192.168.1.0/24 [110/2] via 10.53.0.2, 00:01:04, Ethernet0/0
	```

	```
	R2# sh ip route
	
	Gateway of last resort is 10.53.0.1 to network 0.0.0.0
	
	O*E2  0.0.0.0/0 [110/1] via 10.53.0.1, 00:01:22, Ethernet0/0
	      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
	C        10.53.0.0/24 is directly connected, Ethernet0/0
	L        10.53.0.2/32 is directly connected, Ethernet0/0
	      192.168.1.0/24 is variably subnetted, 2 subnets, 2 masks
	C        192.168.1.0/24 is directly connected, Loopback1
	L        192.168.1.1/32 is directly connected, Loopback1
	```

**Вопрос:**
Почему стоимость OSPF для маршрута по умолчанию на R2 отличается от стоимости OSPF в R1 для сети 192.168.1.0/24?
> В таблицах маршрутизации хранится суммарная стоимость каналов на пути к точке назначения. В данном случае маршрут до сети 192.168.1.0/24 проходит два канала: канал между двумя маршрутизаторами со стоимостью 1 и канал на выходе из интерфейса Loopback1 маршрутизатора R2 со стоимостью 1 (что является значением по умолчанию для интерфейсов обратной петли). В случае же с маршрутом по умолчанию для маршрутизатора R2, трафику нужно лишь пройти до интерфейса 10.53.0.1, до которого только идин канал со стоимостью 1.
