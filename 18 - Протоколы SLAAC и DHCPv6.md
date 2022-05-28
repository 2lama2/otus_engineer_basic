## Настройка DHCPv6

#### Топология

![[Pasted image 20220515152132.png]]
#### Таблица адресации
<table>
<thead>
  <tr>
    <th>Устройство</th>
    <th>Интерфейс</th>
    <th>IPv6-адрес</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td rowspan="4">R1</td>
    <td rowspan="2">e0/1</td>
    <td>2001:db8:acad:2::1/64</td>
  </tr>
  <tr>
    <td>fe80::1</td>
  </tr>
  <tr>
    <td rowspan="2">e0/0</td>
    <td>2001:db8:acad:1::1/64</td>
  </tr>
  <tr>
    <td>fe80::1</td>
  </tr>
  <tr>
    <td rowspan="4">R2</td>
    <td rowspan="2">e0/0</td>
    <td>2001:db8:acad:2::2/64</td>
  </tr>
  <tr>
    <td>fe80::2</td>
  </tr>
  <tr>
    <td rowspan="2">e0/1</td>
    <td>2001:db8:acad:3::1/64</td>
  </tr>
  <tr>
    <td>fe80::1</td>
  </tr>
  <tr>
    <td>Linux1</td>
    <td>NIC</td>
    <td>DHCP</td>
  </tr>
  <tr>
    <td>Linux2</td>
    <td>NIC</td>
    <td>DHCP</td>
  </tr>
</tbody>
</table>

#### Задачи

1. Создание сети и настройка основных параметров устройства]]
2. Проверка назначения адреса SLAAC от R1
3. Настройка и проверка сервера DHCPv6 без отслеживания состояния на R1
4. Настройка и проверка DHCPv6-сервера с отслеживанием состояния на R1
5. Настройка и проверка DHCPv6 Relay на R2

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

* На каждом роутере включим маршрутизацию протокола IPv6:

	```
	R(config)#ipv6 unicast-routing
	```

* Настроим маршрутизатор **`R1`** и **`R2`** по таблице адресации:

	```
	R1# sh ipv6 int br
	Ethernet0/0            [up/up]
	    FE80::1
	    2001:DB8:ACAD:1::1
	Ethernet0/1            [up/up]
	    FE80::1
	    2001:DB8:ACAD:2::1
	Ethernet0/2            [administratively down/down]
	    unassigned
	Ethernet0/3            [administratively down/down]
	    unassigned
	
	R2# sh ipv6 int br
	Ethernet0/0            [up/up]
	    FE80::2
	    2001:DB8:ACAD:2::2
	Ethernet0/1            [up/up]
	    FE80::1
	    2001:DB8:ACAD:3::1
	Ethernet0/2            [administratively down/down]
	    unassigned
	Ethernet0/3            [administratively down/down]
	    unassigned
	```

* На обоих маршрутизаторах установим маршрутом по умолчанию соответствующий адрес интерфейса противоположного маршрутизатора:

	```
	R1(config)# ipv6 route ::/0 e0/1 fe80::2
	
	R2(config)# ipv6 route ::/0 e0/0 fe80::1
	```

* Проверим, что настройки верны командой **PING** с одного маршрутизатора до другого:

	```
	R2# ping ipv6 fe80::1
	Output Interface: ethernet0/0
	Type escape sequence to abort.
	Sending 5, 100-byte ICMP Echos to FE80::1, timeout is 2 seconds:
	Packet sent with a source address of FE80::2%Ethernet0/0
	!!!!!
	Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
	```

### 2. Проверка назначения адреса SLAAC от R1
* Настроим на хосте **`Linux1`** сетевой интерфейс для использования протокола IPv6 в режиме автосонфигурации (SLAAC):

	```
	root@Linux1:~# cat /etc/network/interfaces
	......
	# The primary network interface
	allow-hotplug ens3
	iface ens3 inet6 auto
	```

* Проверим полученные настройки:

	```
	root@Linux1:~# ip -6 r
	::1 dev lo proto kernel metric 256 pref medium
	2001:db8:acad:1::/64 dev ens3 proto kernel metric 256 expires 2591984sec pref medium
	fe80::/64 dev ens3 proto kernel metric 256 pref medium
	default via fe80::1 dev ens3 proto ra metric 1024 expires 1784sec hoplimit 64 pref medium
	
	root@Linux1:~# ip -6 a
	......
	2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP qlen 1000
	    inet6 2001:db8:acad:1:250:ff:fe00:700/64 scope global dynamic mngtmpaddr 
	       valid_lft 2591827sec preferred_lft 604627sec
	    inet6 fe80::250:ff:fe00:700/64 scope link 
	       valid_lft forever preferred_lft forever
	```

	видим, что адрес хоста сформирован префиксом сети **`2001:db8:acad:1::/64`**, установленном на маршрутизаторе R1 и идентификатор интерфейса сформирован методом EUI-64 - **`250:ff:fe00:700`**. А также шлюзом по умолчанию установлен link-local адрес маршрутизатора - **`fe80::1`**.

### 3. Настройка и проверка сервера DHCPv6 без отслеживания состояния на R1
* Настроим DHCPv6-сервер на маршрутизаторе R1:

	- создадим пул:

	```
	R1(config)#ipv6 dhcp pool R1-STATELESS
	```

	- зададим имя DNS-сервера и доменный суффикс:

	```
	R1(config-dhcpv6)#dns-server 2001:db8:acad::254
	
	R1(config-dhcpv6)#domain-name STATELESS.com
	```

	- на интерфейсе, смотрящем в клиентский сегмент, настроим параметры сообщения **`RA`** протокола **`ICMPv6`** так, чтобы клиенты для автоконфигурации IPv6 обращались на DHCP-сервер за дополнительными настройками (устанавим флаг **`O`**):

	```
	R1(config-dhcpv6)#int e0/0
	
	R1(config-if)#ipv6 nd other-config-flag
	
	R1(config-if)#ipv6 dhcp server R1-STATELESS
	
	```

	* Проверим вновь полученные параметры автоконфигурации на клиенте Linux1 после перезагрузки:

	```
	
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

* Посотрим статистику DHCP-сервера:

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
