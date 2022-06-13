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
Интернет-провайдер выделил компании общедоступное пространство IP-адресов 209.165.200.224/29. Эта сеть используется для обращения к каналу между маршрутизатором ISP (R2) и шлюзом компании (R1). Первый адрес (209.165.200.225) назначается интерфейсу g0/0 на R2, а последний адрес (209.165.200.230) назначается интерфейсу g0/0/0 на R1. Остальные адреса (209.165.200.226-209.165.200.229) будут использоваться для предоставления доступа в Интернет хостам компании. Маршрут по умолчанию используется от R1 до R2. Подключение интернет-провайдера к Интернету смоделировано loopback-адресом на маршрутизаторе интернет-провайдера.

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
	
	ip default-gateway 209.165.200.225
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

2. Создадим пул адресов, в которые будут транслироваться выбранные предыдущим пунктом адреса:

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
1. С Linux2 выполним **`PING`** до адреса **`209.165.200.1`**, находящемся в "Интернет-сегменте" и отобразим NAT-таблицу на **`R1`**:
	```
	R1# sh ip nat translations
	
	Pro Inside global      Inside local       Outside local      Outside global
	icmp 209.165.200.226:580 192.168.1.3:580  209.165.200.1:580  209.165.200.1:580
	--- 209.165.200.226    192.168.1.3        ---                ---
	```
	Вывод команды показал, что произошла динамическая трансляция адресов из внутренней сети во внешний пул адресов.

2. С Linux1 выполним **`PING`** до адреса **`209.165.200.1`**, находящемся в "Интернет-сегменте" и отобразим NAT-таблицу на **`R1`**:

	```
	R1# sh ip nat translations
	
	Pro Inside global      Inside local       Outside local      Outside global
	icmp 209.165.200.227:479 192.168.1.2:479  209.165.200.1:479  209.165.200.1:479
	--- 209.165.200.227    192.168.1.2        ---                ---
	icmp 209.165.200.226:583 192.168.1.3:583  209.165.200.1:583  209.165.200.1:583
	--- 209.165.200.226    192.168.1.3        ---                ---
	```
	Вывод показал, что теперь таблица трансляции содержит записи для обоих внутренних адресов и для второго хоста из пула внешних адресов выбран следующий свободный.

3. Выполним аналогичную предыдущим команду PING на коммутаторах **`S1`** и **`S2`**:

	```
	R1# sh ip nat translations
	
	Pro Inside global      Inside local       Outside local      Outside global
	--- 209.165.200.227    192.168.1.2        ---                ---
	--- 209.165.200.226    192.168.1.3        ---                ---
	--- 209.165.200.228    192.168.1.11       ---                ---
	```
	Вывод показывает, что для трансляции использованы все доступные адреса и команда PING на коммутаторе **`S2`** завершается неудачей:

	```
	S1# ping 209.165.200.1
	
	Type escape sequence to abort.
	Sending 5, 100-byte ICMP Echos to 209.165.200.1, timeout is 2 seconds:
	U.U.U
	Success rate is 0 percent (0/5)
	```

	С помощью следующей команды можно увидеть время резервирования записей для выполненных трансляций:

	```
	R1# sh ip nat translations verbose
	
	Pro Inside global      Inside local       Outside local      Outside global
	......
	--- 209.165.200.226    192.168.1.3        ---                ---
	    create 00:29:15, use 00:00:03 timeout:86400000, left 23:59:56, Map-Id(In): 1,
	    flags:
	none, use_count: 7, entry-id: 1, lc_entries: 0
	......
	```
	Поля **`timeout:86400000, left 23:59:56`** показывают время жизни записи в таблице трансляции.

4. Для очистки таблицы и статистики выполним следующие команды:

	```
	R1# clear ip nat translation *
	
	R1# clear ip nat statistics
	```

### 3. Настройка и проверка PAT для IPv4
* Для настройки механизма PAT удалим настройку NAT (в части привязки списка доступа, пула и службы трансляции) из предыдущего раздела:

	```
	R1(config)# no ip nat inside source list 1 pool PUBLIC_ACCESS
	```

* Задействуем механизм трансляции PAT:

	```
	R1(config)#ip nat inside source list 1 pool PUBLIC_ACCESS overload
	```

* Проведем некоторые тесты:
1. С Linux2 выполним **`PING`** до адреса **`209.165.200.1`**, находящемся в "Интернет-сегменте" и отобразим NAT-таблицу на **`R1`**:

	```
	R1# sh ip nat translations
	
	Pro Inside global      Inside local       Outside local      Outside global
	......
	icmp 209.165.200.228:604 192.168.1.3:604  209.165.200.1:604  209.165.200.1:604
	......
	```

2. С Linux1 выполним **`PING`** до адреса **`209.165.200.1`**, находящемся в "Интернет-сегменте" и отобразим NAT-таблицу на **`R1`**:

	```
	R1# sh ip nat translations
	
	Pro Inside global      Inside local       Outside local      Outside global
	icmp 209.165.200.228:494 192.168.1.2:494  209.165.200.1:494  209.165.200.1:494
	......
	icmp 209.165.200.228:604 192.168.1.3:604  209.165.200.1:604  209.165.200.1:604
	.......
	```

3. Выполним аналогичную предыдущим команду PING на коммутаторах **`S1`** и **`S2`**:

	```
	R1# sh ip nat translations
	
	Pro Inside global      Inside local       Outside local      Outside global
	icmp 209.165.200.228:494 192.168.1.2:494  209.165.200.1:494  209.165.200.1:494
	icmp 209.165.200.228:604 192.168.1.3:604  209.165.200.1:604  209.165.200.1:604
	icmp 209.165.200.228:3 192.168.1.11:3     209.165.200.1:3    209.165.200.1:3
	icmp 209.165.200.228:1 192.168.1.12:1     209.165.200.1:1    209.165.200.1:1
	```
	Вывод показывает, что трансляция происходит в один внешний адрес.

* При текущей настройке в нашем сценарии остались неиспользуемые IPv4-адреса. Для исправления данной ситуации перейдем к PAT с перегрузкой интерфейса.
	1. Выполним очистку статистики и таблицы трансляции командами **`clear ip nat translation *`** и **`clear ip nat statistics`**
	2. Удалим некоторые ранее сделанные настройки:

	```
	R1(config)# no ip nat inside source list 1 pool PUBLIC_ACCESS overload
	
	R1(config)# no ip nat pool PUBLIC_ACCESS
	```

	3. Включим механизм PAT с использованием перегрузки интерфейса:

	```
	R1(config)#ip nat inside source list 1 interface e0/0 overload
	```

	Выполним PING со всех устройств и проверим результат:

	```
	R1# sh ip nat translations
	
	Pro Inside global      Inside local       Outside local      Outside global
	icmp 209.165.200.230:498 192.168.1.2:498  209.165.200.1:498  209.165.200.1:498
	icmp 209.165.200.230:609 192.168.1.3:609  209.165.200.1:609  209.165.200.1:609
	icmp 209.165.200.230:4 192.168.1.11:4     209.165.200.1:4    209.165.200.1:4
	icmp 209.165.200.230:2 192.168.1.12:2     209.165.200.1:2    209.165.200.1:2
	```
	Теперь используется только один адрес внешнего интерфейса маршрутизатора **`R1`**



### 4. Настройка и проверка статического NAT для IPv4
В этой части будет настроена статическая NAT таким образом, чтобы Linux1 был доступен напрямую из Интернета. Linux1 будет доступен из R2 по адресу 209.165.200.229.

* На R1 настроим команду NAT, необходимую для статического сопоставления внутреннего адреса с внешним адресом:

	```
	R1(config)# ip nat inside source static 192.168.1.2 209.165.200.229
	```

* Проверим настройки и протестируем работу static NAT командой PING с маршрутизатора **`R2`**:

	```
	R1#sh ip nat translations
	
	Pro Inside global      Inside local       Outside local      Outside global
	icmp 209.165.200.229:0 192.168.1.2:0      209.165.200.225:0  209.165.200.225:0
	--- 209.165.200.229    192.168.1.2        ---                ---
	```
	Ping проходит, static NAT отработал.