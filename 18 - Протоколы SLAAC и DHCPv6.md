## Настройка DHCPv6

#### Топология

![[Pasted image 20220515152132.png]]

![Pics](https://github.com/2lama2/otus_engineer_basic/blob/7c5c20954e18a4eba91e4fafb47af1b3bc416093/pics/Pasted%20image%2020220515152132.png)
#### Таблица адесации
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

1. Создание сети и настройка основных параметров устройства
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
* Настроим на хосте **`Linux1`** сетевой интерфейс для использования протокола IPv6 в режиме автоконфигурации (SLAAC):

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
	R1(config)# ipv6 dhcp pool R1-STATELESS
	```

	- зададим имя DNS-сервера и доменный суффикс:

	```
	R1(config-dhcpv6)# dns-server 2001:db8:acad::254
	
	R1(config-dhcpv6)# domain-name STATELESS.com
	```

	- на интерфейсе, смотрящем в клиентский сегмент, настроим параметры сообщения **`RA`** протокола **`ICMPv6`** так, чтобы клиенты для автоконфигурации IPv6 обращались на DHCP-сервер за дополнительными настройками (устанавим флаг **`O`**):

	```
	R1(config-dhcpv6)# int e0/0
	
	R1(config-if)# ipv6 nd other-config-flag
	
	R1(config-if)# ipv6 dhcp server R1-STATELESS
	
	```

* Проверим вновь полученные параметры автоконфигурации на клиенте **`Linux1`** после перезагрузки:

	1. конфигурация интерфейса в режиме DHCP без сохранения состояния (метод **`auto`**, параметр **`dhcp 1`**):

	```
	root@Linux1:~# cat /etc/network/interfaces
	......
	allow-hotplug ens3
	iface ens3 inet6 auto
	dhcp 1
	accept_ra 1
	```

	2. IPv6-сформирован полученным от маршрутизатора префиксом сети и сгенерированным методом EUI-64 идентификатором интерфейса:

	```
	root@Linux1:~# ip -6 a
	
	1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 state UNKNOWN qlen 1000
	    inet6 ::1/128 scope host 
	       valid_lft forever preferred_lft forever
	2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP qlen 1000
	    inet6 2001:db8:acad:1:250:ff:fe00:700/64 scope global dynamic mngtmpaddr 
	       valid_lft 2591862sec preferred_lft 604662sec
	    inet6 fe80::250:ff:fe00:700/64 scope link 
	       valid_lft forever preferred_lft forever
	```

	3. шлюзом по умолчанию назначен link-local адрес интерфейса маршрутизатора, подключенного к клиентской сети:

	```
	root@Linux1:~# ip -6 r
	
	::1 dev lo proto kernel metric 256 pref medium
	
	2001:db8:acad:1::/64 dev ens3 proto kernel metric 256 expires 2591949sec pref medium
	
	fe80::/64 dev ens3 proto kernel metric 256 pref medium
	
	default via fe80::1 dev ens3 proto ra metric 1024 expires 1749sec hoplimit 64 pref medium
	```

	4. с сервера DHCP получен суффикс доменного имени, а также IPv6-адрес DNS-сервера:

	```
	root@Linux1:~# cat /etc/resolv.conf 

	search STATELESS.com.
	nameserver 2001:db8:acad::254
	```

* Проверим доступность второй клиентской сети командой **PING** до IPv6-адреса маршрутизатора **`R2`**:

	```
	root@Linux1:~# ping6 -c 3 2001:db8:acad:3::1
	
	PING 2001:db8:acad:3::1(2001:db8:acad:3::1) 56 data bytes
	64 bytes from 2001:db8:acad:3::1: icmp_seq=1 ttl=63 time=0.596 ms
	64 bytes from 2001:db8:acad:3::1: icmp_seq=2 ttl=63 time=0.719 ms
	64 bytes from 2001:db8:acad:3::1: icmp_seq=3 ttl=63 time=0.771 ms
	
	--- 2001:db8:acad:3::1 ping statistics ---
	3 packets transmitted, 3 received, 0% packet loss, time 39ms
	rtt min/avg/max/mdev = 0.596/0.695/0.771/0.076 ms
	```

### 4. Настройка и проверка DHCPv6-сервера с отслеживанием состояния на R1
Настроим маршрутизатор **`R1`** для ответа на запросы DHCPv6 из локальной сети за маршрутизатором **`R2`**.

* Создадим пул для второй клиентской сети:

	```
	R1(config)# ipv6 dhcp pool R2-STATEFUL
	```

* Назначим префикс адреса:

	```
	R1(config-dhcpv6)# address prefix 2001:db8:acad:3:aaa::/80
	```

* Зададим имя DNS-сервера и доменный суффикс:

	```
	R1(config-dhcpv6)# dns-server 2001:db8:acad::254
	
	R1(config-dhcpv6)# domain-name STATEFUL.com
	```

* На интерфейсе, подключенном в сеть с роутером **`R2`**, назначим DHCPv6-сервер с пулом R2-STATEFUL:

	```
	R1(config-dhcpv6)# int e0/1
	
	R1(config-if)# ipv6 dhcp server R2-STATEFUL
	```


### 5. Настройка и проверка DHCPv6 Relay на R2
* Настроим на хосте **`Linux2`** сетевой интерфейс для использования протокола IPv6 в режиме получения настроек с DHCPv6-сервера с отслеживанием состояний (метод **`dhcp`**) и автоконфигурации SLAAC и (параметр **`autoconf 1`**):

	```
	root@Linux2:~# cat /etc/network/interfaces
	......
	allow-hotplug ens3
	iface ens3 inet6 dhcp
	accept_ra 1
	autoconf 1
	```

* Проверим полученные настройки:

	```
	root@Linux2:~# ip -6 a
	
	1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 state UNKNOWN qlen 1000
	    inet6 ::1/128 scope host 
	       valid_lft forever preferred_lft forever
	2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP qlen 1000
	    inet6 2001:db8:acad:3:250:ff:fe00:800/64 scope global dynamic mngtmpaddr 
	       valid_lft 2591835sec preferred_lft 604635sec
	    inet6 fe80::250:ff:fe00:800/64 scope link 
	       valid_lft forever preferred_lft forever
	```

	```
	root@Linux2:~# ip -6 r
	
	::1 dev lo proto kernel metric 256 pref medium
	2001:db8:acad:3::/64 dev ens3 proto kernel metric 256 expires 2591924sec pref medium
	fe80::/64 dev ens3 proto kernel metric 256 pref medium
	default via fe80::1 dev ens3 proto ra metric 1024 expires 1724sec hoplimit 64 pref medium
	```

	видим, что адрес хоста сформирован префиксом сети **`2001:db8:acad:3::/64`**, установленном на маршрутизаторе **`R2`** на интерфейсе **`e0/1`** и идентификатор интерфейса сформирован методом EUI-64 - **`250:ff:fe00:800`**. А также шлюзом по умолчанию установлен link-local адрес маршрутизатора **`R2`** - **`fe80::1`**.

* Настроим маршрутизатор **`R2`** в качестве агента DHCPv6-ретрансляции для второй клиентской сети:

	```
	R2(config)# int e0/1
	
	R2(config-if)# ipv6 nd managed-config-flag
	
	R2(config-if)# ipv6 dhcp relay destination fe80::1 Ethernet 0/0
	```

	Командой **`ipv6 nd managed-config-flag`** устанавим флаг **`M`** в сообщених **`RA`** протокола **`ICMPv6`**, чтобы клиенты для автоконфигурации IPv6 обращались на DHCPv6-сервер за всеми настройками, кроме шлюза по умолчанию, который по-прежнему берется клиентом из **`RA`**-сообщения маршрутизатора.

	Командой **`ipv6 dhcp relay destination fe80::1 Ethernet 0/0`** включаем relay-агента на интерфейсе маршрутизатора в локальной сети, который будет пересылать запросы к DHCPv6-серверу из сети клиентов к указанному адресу DHCPv6-сервера (в данном случае - к маршрутизатору **`R1`**, используя его link-local-адрес, через свой интерфейс **`e0/0`**).

* После перезагрузки **`Linux2`** проверим настройки, полученные от DHCPv6-сервера:

	```
	root@Linux2:~# ip -6 a
	
	1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 state UNKNOWN qlen 1000
	    inet6 ::1/128 scope host 
	       valid_lft forever preferred_lft forever
	2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP qlen 1000
	    inet6 2001:db8:acad:3:aaa:b4b7:229e:3916/128 scope global 
	       valid_lft forever preferred_lft forever
	    inet6 2001:db8:acad:3:250:ff:fe00:800/64 scope global dynamic mngtmpaddr 
	       valid_lft 2591945sec preferred_lft 604745sec
	    inet6 fe80::250:ff:fe00:800/64 scope link 
	       valid_lft forever preferred_lft forever
	
	root@Linux2:~# ip -6 r
	
	::1 dev lo proto kernel metric 256 pref medium
	2001:db8:acad:3:aaa:b4b7:229e:3916 dev ens3 proto kernel metric 256 pref medium
	2001:db8:acad:3::/64 dev ens3 proto kernel metric 256 expires 2591938sec pref medium
	fe80::/64 dev ens3 proto kernel metric 256 pref medium
	default via fe80::1 dev ens3 proto ra metric 1024 expires 1738sec hoplimit 64 pref medium
	
	root@Linux2:~# cat /etc/resolv.conf 
	
	search STATEFUL.com.
	nameserver 2001:db8:acad::254
	```

	После того, как мы настроили relay-агента, наш клиент смог получить все параметры (кроме шлюза по умолчанию) от DHCPv6-сервера на маршрутизаторе **`R1`**. А именно: адрес получен с нужным префиксом, настроенным в пуле адресов для данной сети, также получены настройки DNS-сервера и суффикса доменного имени.

	Информацию о выделенных адресах клиентам можно посмотреть с помощью команды:

	```
	R1# sh ipv6 dhcp binding
	
	Client: FE80::250:FF:FE00:800
	  DUID: 000100012A267920005000000800
	  Username : unassigned
	  VRF : default
	  IA NA: IA ID 0x00000800, T1 43200, T2 69120
	    Address: 2001:DB8:ACAD:3:AAA:B4B7:229E:3916
	            preferred lifetime 86400, valid lifetime 172800
	            expires at May 31 2022 08:59 PM (170011 seconds)
	```

* Проверим доступность первой клиентской сети из второй, используя команду **`PING`** до адреса интерфейса маршрутизатора **`R1`** c клиента **`Linux2`**:

	```
	root@Linux2:~# ping6 2001:db8:acad:1::1
	
	PING 2001:db8:acad:1::1(2001:db8:acad:1::1) 56 data bytes
	64 bytes from 2001:db8:acad:1::1: icmp_seq=1 ttl=63 time=0.668 ms
	64 bytes from 2001:db8:acad:1::1: icmp_seq=2 ttl=63 time=0.989 ms
	64 bytes from 2001:db8:acad:1::1: icmp_seq=3 ttl=63 time=0.831 ms
	^C
	--- 2001:db8:acad:1::1 ping statistics ---
	3 packets transmitted, 3 received, 0% packet loss, time 6ms
	rtt min/avg/max/mdev = 0.668/0.829/0.989/0.133 ms
	```
