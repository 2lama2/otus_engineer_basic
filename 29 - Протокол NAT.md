## Настройка NAT для IPv4

![[Pasted image 20220607231421.png]]

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
    <td>209.165.200.230</td>
    <td>255.255.255.248</td>
  </tr>
  <tr>
    <td>e0/1</td>
    <td>192.168.1.1</td>
    <td>255.255.255.0</td>
  </tr>
  <tr>
    <td rowspan="2">R2</td>
    <td>e0/0</td>
    <td>209.165.200.225</td>
    <td>255.255.255.248</td>
  </tr>
  <tr>
    <td>Lo1</td>
    <td>209.165.200.1</td>
    <td>255.255.255.224</td>
  </tr>
  <tr>
    <td>S1</td>
    <td>VLAN 1</td>
    <td>192.168.1.11</td>
    <td>255.255.255.0</td>
  </tr>
  <tr>
    <td>S2</td>
    <td>VLAN 1</td>
    <td>192.168.1.12</td>
    <td>255.255.255.0</td>
  </tr>
  <tr>
    <td>Linux1</td>
    <td>NIC</td>
    <td>192.168.1.2</td>
    <td>255.255.255.0</td>
  </tr>
  <tr>
    <td>Linux2</td>
    <td>NIC</td>
    <td>192.168.1.3</td>
    <td>255.255.255.0</td>
  </tr>
</tbody>
</table>
Подключение к Интернету смоделировано loopback-адресом на маршрутизаторе R2.

#### Задачи

1. Создание сети и настройка основных параметров устройства
2. Настройка и проверка NAT для IPv4
3. Настройка и проверка PAT для IPv4
4. Настройка и проверка статического NAT для IPv4

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

* Настроим маршрутизаторы согласно таблице адресации и для R1 зададим маршрут по умолчанию, указывающим на R2:

	```
	R1# sh run | sec interface
	
	interface Ethernet0/0
	 ip address 209.165.200.230 255.255.255.248
	 duplex auto
	
	interface Ethernet0/1
	 ip address 192.168.1.1 255.255.255.0
	 duplex auto
	
	interface Ethernet0/2
	 no ip address
	 shutdown
	 duplex auto
	
	interface Ethernet0/3
	 no ip address
	 shutdown
	 duplex auto
	
	
	R1# sh run | sec gate
	
	ip default-gateway 209.165.200.255
	```

	```
	R2# sh run | sec interface
	
	interface Loopback1
	 ip address 209.165.200.1 255.255.255.224
	
	interface Ethernet0/0
	 ip address 209.165.200.225 255.255.255.248
	 duplex auto
	
	interface Ethernet0/1
	 no ip address
	 shutdown
	 duplex auto
	
	interface Ethernet0/2
	 no ip address
	 shutdown
	 duplex auto
	
	interface Ethernet0/3
	 no ip address
	 shutdown
	 duplex auto
	```

* Настроим коммутаторы согласно таблице адресации

### 2. Настройка и проверка NAT для IPv4
* Настроим NAT на **`R1`**, используя пул из трех адресов **`209.165.200.226-209.165.200.228`**.

1. Настроим список доступа, определяющий какие адреса будут транслироваться:

	```
	R1(config)# access-list 1 permit 192.168.1.0 0.0.0.255
	```

2. Создадим пул  адресов, в которые будут транслироваться выбранные предыдущим пунктом адреса:

	```
	R1(config)# ip nat pool PUBLIC_ACCESS 209.165.200.226 209.165.200.228 netmask 255.255.255.248
	```

3. Настроим трансляцию адресов, связав ранее созданные ACL и пул адресов с процессом преобразования адресов:

	```
	R1(config)# ip nat inside source list 1 pool PUBLIC_ACCESS
	```

4. Зададим внутренний и внешний интерфейсы:

	```
	R1(config)# int e0/1
	R1(config-if)# ip nat inside
	R1(config-if)# int e0/0
	R1(config-if)# ip nat outside
	```

* Проведем некоторые тесты:
	* С Linux2 выполним **`PING`** до адреса **`209.165.200.1`**, находящемся в "Интернет-сегменте" и отобразим NAT-таблицу на **`R1`**:
	```
		
	```

### 1. Настройка основных параметров устройства
* ффф

	```
	
	```

* aaa

	```
	
	```

### 1. Настройка основных параметров устройства
* ффф

	```
	
	```

* aaa

	```
	
	```

### Вопросы для повторения
* Какой идентификатор подсети в индивидуальном IPv6-адресе **`2001:db8:acad::aaa:1234/64`**?
	> Идентификатор подсети для указанного адреса равен **`0000`**.
