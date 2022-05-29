## Реализация DHCPv4

#### Топология

![[Pasted image 20220515152132.png]]

![Pics](https://github.com/2lama2/otus_engineer_basic/blob/7c5c20954e18a4eba91e4fafb47af1b3bc416093/pics/Pasted%20image%2020220515152132.png)
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
    <td rowspan="5">R1</td>
    <td>e0/1</td>
    <td>10.0.0.1</td>
    <td>255.255.255.252</td>
    <td rowspan="5">-</td>
  </tr>
  <tr>
    <td>e0/0</td>
    <td>-</td>
    <td>-</td>
  </tr>
  <tr>
    <td>e0/0.100</td>
    <td>192.168.1.1</td>
    <td>255.255.255.192</td>
  </tr>
  <tr>
    <td>e0/0.200</td>
    <td>192.168.1.65</td>
    <td>255.255.255.224</td>
  </tr>
  <tr>
    <td>e0/0.1000</td>
    <td>-</td>
    <td>-</td>
  </tr>
  <tr>
    <td rowspan="2">R2</td>
    <td>e0/0</td>
    <td>10.0.0.2</td>
    <td>255.255.255.252</td>
    <td rowspan="2">-</td>
  </tr>
  <tr>
    <td>e0/1</td>
    <td>192.168.1.97</td>
    <td>255.255.255.240</td>
  </tr>
  <tr>
    <td>S1</td>
    <td>VLAN 200</td>
    <td>192.168.1.66</td>
    <td>255.255.255.224</td>
    <td>192.168.1.65</td>
  </tr>
  <tr>
    <td>S2</td>
    <td>VLAN 1</td>
    <td>192.168.1.98</td>
    <td>255.255.255.240</td>
    <td>192.168.1.97</td>
  </tr>
  <tr>
    <td>Linux1</td>
    <td>NIC</td>
    <td>DHCP</td>
    <td>DHCP</td>
    <td>DHCP</td>
  </tr>
  <tr>
    <td>Linux2</td>
    <td>NIC</td>
    <td>DHCP</td>
    <td>DHCP</td>
    <td>DHCP</td>
  </tr>
</tbody>
</table>

#### Таблица VLAN
| **VLAN** | **Имя** | **Назначенный интерфейс** |
|:---------------|:--------------|:-------------|
| 1             | Нет            | S2: e0/1     |
| 100           | CLIENTS        | S1: e0/0     |
| 200           | MGMT           | S1: VLAN 200 |
| 999           | PARCKING_LOT   | S1: e0/2-3   |
| 1000          | NATIVE         | -            |


#### Задачи

1. Создание сети и настройка основных параметров устройства
2. Настройка и проверка двух серверов DHCPv4 на R1
3. Настройка и проверка DHCP-ретрансляции на R2

---

### 1. Создание сети и настройка основных параметров устройства
* Создание схемы адресации

	Для того, чтобы организовать поддержку 58 хостов в клиентском сегменте за маршрутизатором R1 (VLAN CLIENTS), 28 хостов в управляющем сегменте (VLAN MGMT), а также 12 хостов в клиентском сегменте за маршрутизатором R2 (VLAN CLENTS), разобъём сеть 192.168.1.0/24 на следующие подсети:
	
	* **Подсеть A**: 192.168.1.0 / 26
		- диапазон хостов: 192.168.1.1 - 192.168.1.62
		- маска сети: 255.255.255.192
		- количество хостов: 62
	* **Подсеть B**: 192.168.1.64 / 27
		- диапазон хостов: 192.168.1.65 - 192.168.1.94
		- маска сети: 255.255.255.224
		- количество хостов: 30
	* **Подсеть C**: 192.168.1.96 / 28
		- диапазон хостов: 192.168.1.97 - 192.168.1.110
		- маска сети: 255.255.255.240
		- количество хостов: 14

	Заполним соответствующими значениями таблицу адресации.

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

* Настроим маршрутизатор **`R1`**:

	```
	R1(config-subif)# int e0/0.100
	R1(config-subif)# description CLIENTS
	R1(config-subif)# encapsulation dot1Q 100
	R1(config-subif)# ip address 192.168.1.1 255.255.255.192
	
	R1(config-subif)# int e0/0.200
	R1(config-subif)# description MGMT
	R1(config-subif)# encapsulation dot1Q 200
	R1(config-subif)# ip address 192.168.1.65 255.255.255.224
	
	R1(config-subif)# int e0/0.1000
	R1(config-subif)# encapsulation dot1Q 1000 native
	R1(config-subif)# description NATIVE
	
	R1(config)# int e0/0
	R1(config-if)# no sh
	
	R1(config)# int e0/1
	R1(config-if)# ip address 10.0.0.1 255.255.255.252
	R1(config-if)# no sh
	
	R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.0.2
	
	R1# sh ip int br
	
	Interface                  IP-Address      OK? Method Status                Protocol
	Ethernet0/0                unassigned      YES unset  up                    up  
	Ethernet0/0.100            192.168.1.1     YES manual up                    up  
	Ethernet0/0.200            192.168.1.65    YES manual up                    up  
	Ethernet0/0.1000           unassigned      YES unset  up                    up  
	Ethernet0/1                10.0.0.1        YES manual up                    up  
	Ethernet0/2                unassigned      YES unset  administratively down down
	Ethernet0/3                unassigned      YES unset  administratively down down
	```

* Настроим маршрутизатор **`R2`**:

	```
	R2(config)# int e0/1
	R2(config-if)# ip address 192.168.1.97 255.255.255.240
	R2(config-if)# no sh
	
	R2(config-if)# int e0/0
	R2(config-if)# ip address 10.0.0.2 255.255.255.252
	R2(config)# int e0/0
	
	R2(config)# ip route 0.0.0.0 0.0.0.0 10.0.0.1
	
	R2# sh ip int brief
	
	Interface                  IP-Address      OK? Method Status                Protocol
	Ethernet0/0                10.0.0.2        YES manual up                    up  
	Ethernet0/1                192.168.1.97    YES manual up                    up  
	Ethernet0/2                unassigned      YES unset  administratively down down
	Ethernet0/3                unassigned      YES unset  administratively down down
	```

* Выполним настройки коммутаторов, используя вышеприведенную информацию об адресации и конфигурации VLAN:

	```
	S1# sh run
	......
	!
	interface Ethernet0/0
	 switchport access vlan 100
	 switchport mode access
	!
	interface Ethernet0/1
	 switchport trunk allowed vlan 100,200,1000
	 switchport trunk encapsulation dot1q
	 switchport trunk native vlan 1000
	 switchport mode trunk
	!
	interface Ethernet0/2
	 switchport access vlan 999
	 switchport mode access
	 shutdown
	!
	interface Ethernet0/3
	 switchport access vlan 999
	 switchport mode access
	 shutdown
	!
	interface Vlan200
	 ip address 192.168.1.66 255.255.255.224
	!
	ip default-gateway 192.168.1.65
	......
	
	S1# sh ip int br
	
	Interface              IP-Address      OK? Method Status                Protocol
	Ethernet0/0            unassigned      YES unset  up                    up
	Ethernet0/1            unassigned      YES unset  up                    up
	Ethernet0/2            unassigned      YES unset  administratively down down
	Ethernet0/3            unassigned      YES unset  administratively down down
	Vlan200                192.168.1.66    YES manual up                    up
	
	S1# sh vlan
	
	VLAN Name                             Status    Ports
	---- -------------------------------- --------- -------------------------------
	1    default                          active
	100  CLIENTS                          active    Et0/0
	200  MGMT                             active
	999  PARCKING_LOT                     active    Et0/2, Et0/3
	1000 NATIVE                           active
	
	S1# sh interfaces trunk
	
	Port        Mode             Encapsulation  Status        Native vlan
	Et0/1       on               802.1q         trunking      1000
	
	Port        Vlans allowed on trunk
	Et0/1       100,200,1000
	
	Port        Vlans allowed and active in management domain
	Et0/1       100,200,1000
	
	Port        Vlans in spanning tree forwarding state and not pruned
	Et0/1       100,200,1000
	```

	```
	S2# sh run
	......
	!
	interface Vlan1
	 ip address 192.168.1.98 255.255.255.240
	!
	ip default-gateway 192.168.1.97
	......
	
	S2# sh ip int brief
	Interface              IP-Address      OK? Method Status                Protocol
	Ethernet0/0            unassigned      YES unset  up                    up
	Ethernet0/1            unassigned      YES unset  up                    up
	Ethernet0/2            unassigned      YES unset  administratively down down
	Ethernet0/3            unassigned      YES unset  administratively down down
	Vlan1                  192.168.1.98    YES manual up                    up
	
	S2# sh vlan
	
	VLAN Name                             Status    Ports
	---- -------------------------------- --------- -------------------------------
	1    default                          active    Et0/0, Et0/1, Et0/2, Et0/3
	```



### 2. Настройка и проверка двух серверов DHCPv4 на R1
* Настроим два DHCP-пула для подсети A и C:

	* Создадим пул **`R1_Client_LAN`** со следующими параметрами:
		- имя пула: **R1_Client_LAN**
		- сеть: **192.168.1.0 / 26**
		- шлюз: **192.168.1.1**
		- имя домена: **CCNA-lab.com**
		- время аренды: **2дн 14час 30мин**

	* Создадим пул **`R2_Client_LAN`** со следующими параметрами:
		- имя пула: **R2_Client_LAN**
		- сеть: **192.168.1.96 / 28**
		- шлюз: **192.168.1.97**
		- имя домена: **CCNA-lab.com**
		- время аренды: **2дн 14час 30мин**

	* Исключим из автоматической выдачи первые 5 адресов каждой подсети.

	```
	R1# sh run | section dhcp
	
	ip dhcp excluded-address 192.168.1.1 192.168.1.5
	ip dhcp excluded-address 192.168.1.97 192.168.1.102
	
	ip dhcp pool R1_Client_LAN
	 network 192.168.1.0 255.255.255.192
	 domain-name CCNA-lab.com
	 default-router 192.168.1.1
	 lease 2 12 30
	 
	ip dhcp pool R2_Client_LAN
	 network 192.168.1.96 255.255.255.240
	 domain-name CCNA-lab.com
	 default-router 192.168.1.97
	 lease 2 12 30
	```

* На стороне клиента Linux1 проверим автоконфигурацию сетевого интерфейса и доступность шлюза по умолчанию:

	```
	root@Linux1:~# ip a
	......
	2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
	    link/ether 00:50:00:00:01:00 brd ff:ff:ff:ff:ff:ff
	    inet 192.168.1.6/26 brd 192.168.1.63 scope global dynamic ens3
	       valid_lft 217758sec preferred_lft 217758sec
	......	
	root@Linux1:~# ping 192.168.1.1
	PING 192.168.1.1 (192.168.1.1) 56(84) bytes of data.
	64 bytes from 192.168.1.1: icmp_seq=1 ttl=255 time=0.648 ms
	64 bytes from 192.168.1.1: icmp_seq=2 ttl=255 time=1.10 ms
	^C
	--- 192.168.1.1 ping statistics ---
	2 packets transmitted, 2 received, 0% packet loss, time 10ms
	rtt min/avg/max/mdev = 0.648/0.872/1.096/0.224 ms
	```

	Ожидаемо ip-адрес получен динамически из пула для подсети A. Шлюз по умолчанию также доступен.

### 3. Настройка и проверка DHCP-ретрансляции на R2
* Настроим ретрансляцию запросов DHCP из подсети C на DHCP-сервер с адресом 10.0.0.1 с помощью команды **`ip helper-address`**:

	```
	R2# sh run | sec interface
	......
	interface Ethernet0/1
	 ip address 192.168.1.97 255.255.255.240
	 ip helper-address 10.0.0.1
	......
	```

* На стороне клиента Linux2 проверим автоконфигурацию сетевого интерфейса и доступность шлюза по умолчанию:

	```
	root@Linux2:~# ip r
	
	default via 192.168.1.97 dev ens3 
	192.168.1.96/28 dev ens3 proto kernel scope link src 192.168.1.103 
	
	root@Linux2:~# ip a
	......
	2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
	    link/ether 00:50:00:00:02:00 brd ff:ff:ff:ff:ff:ff
	    inet 192.168.1.103/28 brd 192.168.1.111 scope global dynamic ens3
	       valid_lft 132473sec preferred_lft 132473sec
	......
	root@Linux2:~# ping 192.168.1.97
	
	PING 192.168.1.97 (192.168.1.97) 56(84) bytes of data.
	64 bytes from 192.168.1.97: icmp_seq=1 ttl=255 time=0.437 ms
	64 bytes from 192.168.1.97: icmp_seq=2 ttl=255 time=0.654 ms
	64 bytes from 192.168.1.97: icmp_seq=3 ttl=255 time=0.725 ms
	^C
	--- 192.168.1.97 ping statistics ---
	3 packets transmitted, 3 received, 0% packet loss, time 42ms
	rtt min/avg/max/mdev = 0.437/0.605/0.725/0.124 ms
	```

	Здесь также интерфейс получил адрес автоматически из своего пула подсети C. Шлюз доступен.

* Посмотрим статистику DHCP-сервера:

	```
	R1# sh ip dhcp server statistics
	
	Memory usage         50283
	Address pools        2
	Database agents      0
	Automatic bindings   2
	Manual bindings      0
	Expired bindings     0
	Malformed messages   0
	Secure arp entries   0
	
	Message              Received
	BOOTREQUEST          0
	DHCPDISCOVER         3
	DHCPREQUEST          4
	DHCPDECLINE          0
	DHCPRELEASE          1
	DHCPINFORM           0
	
	Message              Sent
	BOOTREPLY            0
	DHCPOFFER            3
	DHCPACK              4
	DHCPNAK              0
	```

	и выданные в аренду IP-адреса:

	```
	R1#sh ip dhcp binding
	Bindings from all pools not associated with VRF:
	IP address          Client-ID/              Lease expiration        Type
	                    Hardware address/
	                    User name
	192.168.1.6         0050.0000.0100          May 19 2022 08:04 AM    Automatic
	192.168.1.103       0050.0000.0200          May 18 2022 11:56 AM    Automatic
	```
