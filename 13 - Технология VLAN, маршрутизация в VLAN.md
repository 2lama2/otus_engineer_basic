## Внедрение маршрутизации между виртуальными локальными сетями

#### Топология
![[Pasted image 20220503225650.png]]
[Pics](https://github.com/2lama2/otus_engineer_basic/blob/4bcde14a674123dc046b6e91a2cc5c762ab8b899/pics/Pasted%20image%2020220503225650.png)
#### Таблица адресации

<table>
<thead>
  <tr>
    <th>Устройство</th>
    <th>Интерфейс</th>
    <th>IP-адрес</th>
    <th>Маска подсети</th>
    <th>Шлюз по умолчанию</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td rowspan="4">R1</td>
    <td>e0/0.10</td>
    <td>192.168.10.1</td>
    <td>255.255.255.0</td>
    <td rowspan="4">-</td>
  </tr>
  <tr>
    <td>e0/0.20</td>
    <td>192.168.20.1</td>
    <td>255.255.255.0</td>
  </tr>
  <tr>
    <td>e0/0.30</td>
    <td>192.168.30.1</td>
    <td>255.255.225.0</td>
  </tr>
  <tr>
    <td>e0/0.1000</td>
    <td>-</td>
    <td>-</td>
  </tr>
  <tr>
    <td>S1</td>
    <td>VLAN 10</td>
    <td>192.168.10.11</td>
    <td>255.255.255.0</td>
    <td>192.168.10.1</td>
  </tr>
  <tr>
    <td>S2</td>
    <td>VLAN 10</td>
    <td>192.168.10.12</td>
    <td>255.255.255.0</td>
    <td>192.168.10.1</td>
  </tr>
  <tr>
    <td>VPC1</td>
    <td>NIC</td>
    <td>192.168.20.3</td>
    <td>255.255.255.0</td>
    <td>192.168.20.1</td>
  </tr>
  <tr>
    <td>VPC2</td>
    <td>NIC</td>
    <td>192.168.30.3</td>
    <td>255.255.255.0</td>
    <td>192.168.30.1</td>
  </tr>
</tbody>
</table>

#### Таблица VLAN

<table>
<thead>
  <tr>
    <th>VLAN</th>
    <th>Имя</th>
    <th>Назначенный интерфейс</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td>10</td>
    <td>Управление</td>
    <td>S1: VLAN 10<br>S2: VLAN 10</td>
  </tr>
  <tr>
    <td>20</td>
    <td>Sales</td>
    <td>S1: E0/2</td>
  </tr>
  <tr>
    <td>30</td>
    <td>Operations</td>
    <td>S2: E0/1</td>
  </tr>
  <tr>
    <td>999</td>
    <td>Parking_Lot</td>
    <td>S1: E0/3<br>S2: E0/2-3</td>
  </tr>
  <tr>
    <td>1000</td>
    <td>Native</td>
    <td>-</td>
  </tr>
</tbody>
</table>

#### Задачи

1. Создание сети и настройка основных параметров устройства
2. Создание сетей VLAN и назначение портов коммутатора
3. Настройка транка 802.1Q между коммутаторами
4. Настройка маршрутизации между сетями VLAN
5. Проверка, что маршрутизация между VLAN работает

---

### 1. Создание сети и настройка основных параметров устройства
* Создадим модель сети в **EVE-NG** и выполним базовую настройку маршрутизатора **`R1`** и коммутаторов **`S1`** и **`S2`** по следующему шаблону:

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

	Также назначим IP-адреса виртуальным машинам **`VPC1`** и **`VPC2`** из таблицы.

### 2. Создание сетей VLAN и назначение портов коммутатора
* Для коммутатора **`S1`** создадим необходимые VLAN'ы и дадим им имена:

	```
	S1(config)# vlan 10
	S1(config-vlan)# name MGMT
	S1(config-vlan)# vlan 20
	S1(config-vlan)# name SALES
	S1(config-vlan)# vlan 30
	S1(config-vlan)# name OPERATIONS
	S1(config-vlan)# vlan 999
	S1(config-vlan)# name PARKING_LOT
	S1(config-vlan)# vlan 1000
	S1(config-vlan)# name NATIVE
	```

	Настроим интерфейс управления и шлюз по умолчанию, используя информацию об IP-адресе в таблице адресации:

	```
	S1(config)# interface vlan 10
	S1(config-if)# ip address 192.168.10.11 255.255.255.0
	S1(config-if)# no shutdown
	S1(config-if)# exit
	S1(config)# ip default-gateway 192.168.10.1
	```

	Назначим все неиспользуемые порты коммутатора (в данном случае **`E0/3`**) VLAN PARKING_LOT, переведем их в режим ACCESS и выключим:

	```
	S1(config)# interface e0/3
	S1(config-if)# switchport mode access
	S1(config-if)# switchport access vlan 999
	S1(config-if)# shutdown
	```

* Настроим остальные порты коммутатора по таблице VLAN:

	```
	S1(config-if)# int e0/2
	S1(config-if)# switchport mode access
	S1(config-if)# switchport access vlan 20
	```

	Убедимся в правильности настройки портов:

	```
	S1# sh vlan brief
		
	VLAN Name                             Status    Ports
	---- -------------------------------- --------- -------------------------------
	1    default                          active    Et0/0, Et0/1
	10   MGMT                             active
	20   SALES                            active    Et0/2
	30   OPERATIONS                       active
	999  PARKING_LOT                      active    Et0/3
	1000 NATIVE                           active
	1002 fddi-default                     act/unsup
	1003 token-ring-default               act/unsup
	1004 fddinet-default                  act/unsup
	1005 trnet-default                    act/unsup
	```

* Аналогичным образом настроим коммутатор **`S2`**:

	```
	S2(config-if)# do sh vlan br
	
	VLAN Name                             Status    Ports
	---- -------------------------------- --------- -------------------------------
	1    default                          active    Et0/0
	10   MGMT                             active
	20   SALES                            active
	30   OPERATIONS                       active    Et0/1
	999  PARKING_LOT                      active    Et0/2, Et0/3
	1000 NATIVE                           active
	1002 fddi-default                     act/unsup
	1003 token-ring-default               act/unsup
	1004 fddinet-default                  act/unsup
	1005 trnet-default                    act/unsup

	```
	
### 3. Настройка транка 802.1Q между коммутаторами
* Проведем настройку магистрального подключения между коммутаторами на стороне коммутатора **`S1`** :

	Выберем интерфейс и укажем использовать тегирование VLAN'ов по стандарту **IEEE 802.1q**:

	```
	S1(config)# interface e0/1
	S1(config-if)# switchport trunk encapsulation dot1q
	```

	Включим режим магистрали и назначим номер собственной (native) VLAN:

	```
	S1(config-if)# switchport mode trunk
	S1(config-if)# switchport trunk native vlan 1000
	```

	Следующей командой разрешим проходить по магистрали трафику с тэгом VLAN 10, 20, 30 и 1000:

	```
	S1(config-if)# switchport trunk allowed vlan 10,20,30,1000
	```

	Проверим правильность сделанных настроек с помощью команды:

	```
	S1(config-if)# do sh int trunk
	
	Port        Mode             Encapsulation  Status        Native vlan
	Et0/1       on               802.1q         trunking      1000
	
	Port        Vlans allowed on trunk
	Et0/1       10,20,30,1000
	
	Port        Vlans allowed and active in management domain
	Et0/1       10,20,30,1000
	
	Port        Vlans in spanning tree forwarding state and not pruned
	Et0/1       10,20,30,1000
	```

* Аналогичным образом настроим магистральный интерфейс коммутатора **`S2`**, обращенный к коммутатору **`S1`**:

	```
	S2# sh interfaces trunk
	
	Port        Mode             Encapsulation  Status        Native vlan
	Et0/0       on               802.1q         trunking      1000
	
	Port        Vlans allowed on trunk
	Et0/0       10,20,30,1000
	
	Port        Vlans allowed and active in management domain
	Et0/0       1,10,20,30,1000
	
	Port        Vlans in spanning tree forwarding state and not pruned
	Et0/0       1,10,20,30,1000
	```

* Настроим магистральный интерфейс на коммутаторе **`S1`**, обращенный к маршрутизатору **`R1`**:

	```
	S1(config)# int e0/0
	S1(config-if)# switchport trunk encapsulation dot1q
	S1(config-if)# switchport mode trunk
	S1(config-if)# switchport trunk native vlan 1000
	S1(config-if)# switchport trunk allowed vlan 10,20,30,1000
	
	S1(config-if)# do sh int trunk
	
	Port        Mode             Encapsulation  Status        Native vlan
	Et0/0       on               802.1q         trunking      1000
	Et0/1       on               802.1q         trunking      1000
	
	Port        Vlans allowed on trunk
	Et0/0       10,20,30,1000
	Et0/1       10,20,30,1000
	
	Port        Vlans allowed and active in management domain
	Et0/0       10,20,30,1000
	Et0/1       10,20,30,1000
	
	Port        Vlans in spanning tree forwarding state and not pruned
	Et0/0       10,20,30,1000
	Et0/1       10,20,30,1000
	```


### 4. Настройка маршрутизации между сетями VLAN
* Настроим маршрутизатор **`R1`** по таблице адресации:

	```
	R1# sh run
	Building configuration...
	
	interface Ethernet0/0
	 no ip address
	 duplex auto
	!
	interface Ethernet0/0.10
	 description MGMT
	 encapsulation dot1Q 10
	 ip address 192.168.10.1 255.255.255.0
	!
	interface Ethernet0/0.20
	 description SALES
	 encapsulation dot1Q 20
	 ip address 192.168.20.1 255.255.255.0
	!
	interface Ethernet0/0.30
	 description OPERATIONS
	 encapsulation dot1Q 30
	 ip address 192.168.30.1 255.255.255.0
	!
	interface Ethernet0/0.1000
	 description NATIVE
	 encapsulation dot1Q 1000 native
	```

	Убедимся, что интерфейсы работают:

	```
	R1# sh ip interface brief
	Interface                  IP-Address      OK? Method Status                Protocol
	Ethernet0/0                unassigned      YES NVRAM  up                    up  
	Ethernet0/0.10             192.168.10.1    YES NVRAM  up                    up  
	Ethernet0/0.20             192.168.20.1    YES NVRAM  up                    up  
	Ethernet0/0.30             192.168.30.1    YES NVRAM  up                    up  
	Ethernet0/0.1000           unassigned      YES unset  up                    up  
	Ethernet0/1                unassigned      YES NVRAM  administratively down down
	Ethernet0/2                unassigned      YES NVRAM  administratively down down
	Ethernet0/3                unassigned      YES NVRAM  administratively down down
	```

### 5. Маршрутизация между VLAN

* Проверим, что маршрутизация выполняется верно:

	VPC1 \==**PING**\==> \[192.168.20.1] R1

	```
	VPC1> ping 192.168.20.1
	
	84 bytes from 192.168.20.1 icmp_seq=1 ttl=255 time=0.364 ms
	84 bytes from 192.168.20.1 icmp_seq=2 ttl=255 time=0.701 ms
	84 bytes from 192.168.20.1 icmp_seq=3 ttl=255 time=0.845 ms
	84 bytes from 192.168.20.1 icmp_seq=4 ttl=255 time=0.592 ms
	84 bytes from 192.168.20.1 icmp_seq=5 ttl=255 time=0.750 ms
	```

	VPC1 \==**PING**\==> \[192.168.30.3] VPC2

	```
	VPC1> ping 192.168.30.3
	84 bytes from 192.168.30.3 icmp_seq=1 ttl=63 time=2.392 ms
	84 bytes from 192.168.30.3 icmp_seq=2 ttl=63 time=1.285 ms
	84 bytes from 192.168.30.3 icmp_seq=3 ttl=63 time=0.849 ms
	84 bytes from 192.168.30.3 icmp_seq=4 ttl=63 time=4.601 ms
	84 bytes from 192.168.30.3 icmp_seq=5 ttl=63 time=1.037 ms
	```

	VPC1 \==**PING**\==> \[192.168.10.12] S2
	[Здесь не удалось победить **EVE-NG**.
	В **Packet Tracer** все отрабатывает - Ping идет до интерфейса коммутатора]

	```
	VPC1> ping 192.168.10.12
	192.168.10.12 icmp_seq=1 timeout
	192.168.10.12 icmp_seq=2 timeout
	192.168.10.12 icmp_seq=3 timeout
	192.168.10.12 icmp_seq=4 timeout
	192.168.10.12 icmp_seq=5 timeout
	```

	VPC2 \==**TRACE**\==> \[192.168.20.3] VPC1

	```
	VPC2> trace 192.168.20.3
	trace to 192.168.20.3, 8 hops max, press Ctrl+C to stop
	 1   192.168.30.1   0.795 ms  0.433 ms  0.573 ms
	 2   *192.168.20.3   2.354 ms (ICMP type:3, code:3, Destination port unreachable)
	```

Таким образом, конфигурация проверена на работоспособность. Такая схема имеет специальное название Router on a stick (ROS).
