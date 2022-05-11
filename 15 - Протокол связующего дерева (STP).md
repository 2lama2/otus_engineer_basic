## Развертывание коммутируемой сети с резервными каналами

#### Топология

![[Pasted image 20220508120131.png]]

![pics](https://github.com/2lama2/otus_engineer_basic/blob/4bcde14a674123dc046b6e91a2cc5c762ab8b899/pics/Pasted%20image%2020220503225650.png)
#### Таблица адресации

| **Устройство** | **Интерфейс** | **IP-адрес** | **Маска подсети** |
|:---------------|:--------------|:-------------|:------------------|
| **S1**             | VLAN 1        | 192.168.1.1  | 255.255.255.0     |
| **S2**             | VLAN 1        | 192.168.1.2  | 255.255.255.0     |
| **S3**             | VLAN 1        | 192.168.1.3  | 255.255.255.0     |

#### Задачи

1. Создание сети и настройка основных параметров устройства
2. Выбор корневого моста
3. Наблюдение за процессом выбора протоколом STP порта, исходя из стоимости портов
4. Наблюдение за процессом выбора протоколом STP порта, исходя из приоритета портов

---

### 1. Создание сети и настройка основных параметров устройства
* Создадим модель сети в **EVE-NG** и выполним базовую настройку коммутаторов **`S1`**,  **`S2`** и **`S3`** по следующему шаблону:

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

* Проверим связь между коммутаторами:

	S1 \==**PING**\==> \[192.168.1.2] S2
	S1 \==**PING**\==> \[192.168.1.3] S3
	S2 \==**PING**\==> \[192.168.1.3] S3

	```
	S1# ping 192.168.1.2
	Type escape sequence to abort.
	Sending 5, 100-byte ICMP Echos to 192.168.1.2, timeout is 2 seconds:
	.!!!!
	Success rate is 80 percent (4/5), round-trip min/avg/max = 1/2/4 ms
	
	S1# ping 192.168.1.3
	Type escape sequence to abort.
	Sending 5, 100-byte ICMP Echos to 192.168.1.3, timeout is 2 seconds:
	.!!!!
	Success rate is 80 percent (4/5), round-trip min/avg/max = 1/1/1 ms
	
	S2# ping 192.168.1.3
	Type escape sequence to abort.
	Sending 5, 100-byte ICMP Echos to 192.168.1.3, timeout is 2 seconds:
	.!!!!
	Success rate is 80 percent (4/5), round-trip min/avg/max = 1/1/1 ms
	```
	
### 2. Выбор корневого моста
* Для каждого коммутатора отключим все порты:

	```
	S(config)# interface range e0/0-3
	```

* переведем задействованные порты в режим магистрали:

	```
	S(config-if-range)# switchport trunk encapsulation dot1q
	S(config-if-range)# switchport mode trunk
	```

* включим те порты, которые объединят наши коммутаторы в кольцо:

	**S1{Et0/0**, **Et0/3}**; **S2{Et0/0**, **Et0/2}**; **S3{Et0/0**, **Et0/3}**:

	```
	S1(config)# int r e0/0, e0/3
	S1(config-if-range)# no sh
	S2(config)# int r e0/0, e0/2
	S2(config-if-range)# no sh
	S3(config)# int r e0/0, e0/3
	S3(config-if-range)# no sh
	```

* Выполним команду **`show spanning-tree`** для каждого коммутатора:

	```
	S1#sh spanning-tree
	
	VLAN0001
	  Spanning tree enabled protocol ieee
	  Root ID    Priority    32769
	             Address     aabb.cc00.1000
	             This bridge is the root
	             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
	
	  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
	             Address     aabb.cc00.1000
	             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
	             Aging Time  300 sec
	
	Interface           Role Sts Cost      Prio.Nbr Type
	------------------- ---- --- --------- -------- --------------------------------
	Et0/0               Desg FWD 100       128.1    P2p
	Et0/3               Desg FWD 100       128.4    P2p
	```

	```
	S2#sh spanning-tree
	
	VLAN0001
	  Spanning tree enabled protocol ieee
	  Root ID    Priority    32769
	             Address     aabb.cc00.1000
	             Cost        100
	             Port        1 (Ethernet0/0)
	             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
	
	  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
	             Address     aabb.cc00.2000
	             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
	             Aging Time  300 sec
	
	Interface           Role Sts Cost      Prio.Nbr Type
	------------------- ---- --- --------- -------- --------------------------------
	Et0/0               Root FWD 100       128.1    P2p
	Et0/2               Desg FWD 100       128.3    P2p
	```

	```
	S3#sh spanning-tree
	
	VLAN0001
	  Spanning tree enabled protocol ieee
	  Root ID    Priority    32769
	             Address     aabb.cc00.1000
	             Cost        100
	             Port        4 (Ethernet0/3)
	             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
	
	  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
	             Address     aabb.cc00.3000
	             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
	             Aging Time  300 sec
	
	Interface           Role Sts Cost      Prio.Nbr Type
	------------------- ---- --- --------- -------- --------------------------------
	Et0/0               Altn BLK 100       128.1    P2p
	Et0/3               Root FWD 100       128.4    P2p
	```

	По результатам выполнения команд заполним следующую таблицу:

| Коммутатор | Инт1: Роль/Сост | Инт2: Роль/Сост | MAC           |
 |------------|-----------------|-----------------|---------------|
 | S1         |Et0/0: Desg/FWD  |Et0/3: Desg/FWD  |aabb.cc00.1000 |
 | S2         |Et0/0: Root/FWD  |Et0/2: Desg/FWD  |aabb.cc00.2000 |
 | S3         |Et0/0: Altn/BLK  |Et0/3: Root/FWD  |aabb.cc00.3000 |

В данном случае корневым мостом является коммутатор **`S1`**. 

> Приоритет идентификатора моста рассчитывается путем сложения значений приоритета и расширенного идентификатора системы. Расширенным идентификатором системы является номер сети VLAN. В нашем случае все три коммутатора имеют равные значения приоритета идентификатора моста (32769 = 32768 + 1, где приоритет по умолчанию = 32768, номер сети VLAN = 1); следовательно, коммутатор с самым низким значением MAC-адреса становится корневым мостом.

Корневыми портами являются: **S2{Et0/0}** и **S3{Et0/3}**.

Назначенными портами являются: **S1{Et0/0, Et0/3}** и **S2{Et0/2}**.

В качестве альтернативного порта выбран: **S3{Et0/0}**.

> Альтернативным данный порт выбран алгоритмом STA потому, что, имея одинаковую стоимость пути до корневого моста с портом **S2{Et0/2}**, он имеет больший mac-адрес и уступает роль назначенного порта вышеуказанному **S2{Et0/2}**, а значит становится альтернативным портом.

### 3. Наблюдение за процессом выбора протоколом STP порта, исходя из стоимости портов
* Уменьшим стоимость корневого порта коммутатора **S3{Et0/3}** на единицу:

	```
	S3(config)# interface e0/3
	S3(config-if)# spanning-tree cost 99
	```

	Проверим изменения, которые произвел алгоритм STA:

	```
	S2# sh spanning-tree
	
	VLAN0001
	  Spanning tree enabled protocol ieee
	  Root ID    Priority    32769
	             Address     aabb.cc00.1000
	             Cost        100
	             Port        1 (Ethernet0/0)
	             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
	
	  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
	             Address     aabb.cc00.2000
	             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
	             Aging Time  300 sec
	
	Interface           Role Sts Cost      Prio.Nbr Type
	------------------- ---- --- --------- -------- --------------------------------
	Et0/0               Root FWD 100       128.1    P2p
	Et0/2               Altn BLK 100       128.3    P2p
	```

	```
	S3# sh spanning-tree
	
	VLAN0001
	  Spanning tree enabled protocol ieee
	  Root ID    Priority    32769
	             Address     aabb.cc00.1000
	             Cost        99
	             Port        4 (Ethernet0/3)
	             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
	
	  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
	             Address     aabb.cc00.3000
	             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
	             Aging Time  300 sec
	
	Interface           Role Sts Cost      Prio.Nbr Type
	------------------- ---- --- --------- -------- --------------------------------
	Et0/0               Desg FWD 100       128.1    P2p
	Et0/3               Root FWD 99        128.4    P2p
	```

	В результате видим, что STA заблокировал порт **S2{Et0/2}** и выбрал назначенным порт **S3{Et0/0}** в связи с тем, что стоимость пути до корневого моста через порт **S3{Et0/0}** стала меньше на единицу по сравнению со стоимостью пути через порт **S2{Et0/2}**.

* Вернем прежнее значение стоимости порта **S3{Et0/3}**:

	```
	S3(config)# interface e0/3
	S3(config-if)# no spanning-tree cost 99
	```

	STA в результате изменений привел систему в начальное состояние.

	```
	S2#sh spanning-tree
	
	VLAN0001
	  Spanning tree enabled protocol ieee
	  Root ID    Priority    32769
	             Address     aabb.cc00.1000
	             Cost        100
	             Port        1 (Ethernet0/0)
	             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
	
	  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
	             Address     aabb.cc00.2000
	             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
	             Aging Time  15  sec
	
	Interface           Role Sts Cost      Prio.Nbr Type
	------------------- ---- --- --------- -------- --------------------------------
	Et0/0               Root FWD 100       128.1    P2p
	Et0/2               Desg FWD 100       128.3    P2p
	```

	```
	S3#sh spanning-tree
	
	VLAN0001
	  Spanning tree enabled protocol ieee
	  Root ID    Priority    32769
	             Address     aabb.cc00.1000
	             Cost        100
	             Port        4 (Ethernet0/3)
	             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
	
	  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
	             Address     aabb.cc00.3000
	             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
	             Aging Time  15  sec
	
	Interface           Role Sts Cost      Prio.Nbr Type
	------------------- ---- --- --------- -------- --------------------------------
	Et0/0               Altn BLK 100       128.1    P2p
	Et0/3               Root FWD 100       128.4    P2p
	```

### 4. Наблюдение за процессом выбора протоколом STP порта, исходя из приоритета портов
* Добавим дополнительные связи между коммутаторами, включив порты **S1{Et0/1**, **Et0/2}**; **S2{Et0/1**, **Et0/3}**; **S3{Et0/1, **Et0/2}**:

	```
	S1(config)# int r e0/1, e0/2
	S1(config-if-range)# no sh
	
	S2(config)# int r e0/1, e0/3
	S2(config-if-range)# no sh
	
	S3(config)# int r e0/1, e0/2
	S3(config-if-range)# no sh
	```

	Изучим работу алгоритма STA при дополнительных избыточных связях:

	```
	S1# sh spanning-tree
	
	VLAN0001
	  Spanning tree enabled protocol ieee
	  Root ID    Priority    32769
	             Address     aabb.cc00.1000
	             This bridge is the root
	             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
	
	  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
	             Address     aabb.cc00.1000
	             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
	             Aging Time  300 sec
	
	Interface           Role Sts Cost      Prio.Nbr Type
	------------------- ---- --- --------- -------- --------------------------------
	Et0/0               Desg FWD 100       128.1    P2p
	Et0/1               Desg FWD 100       128.2    P2p
	Et0/2               Desg FWD 100       128.3    P2p
	Et0/3               Desg FWD 100       128.4    P2p
	```

	```
	S2# sh spanning-tree
	
	VLAN0001
	  Spanning tree enabled protocol ieee
	  Root ID    Priority    32769
	             Address     aabb.cc00.1000
	             Cost        100
	             Port        1 (Ethernet0/0)
	             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
	
	  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
	             Address     aabb.cc00.2000
	             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
	             Aging Time  300 sec
	
	Interface           Role Sts Cost      Prio.Nbr Type
	------------------- ---- --- --------- -------- --------------------------------
	Et0/0               Root FWD 100       128.1    P2p
	Et0/1               Altn BLK 100       128.2    P2p
	Et0/2               Desg FWD 100       128.3    P2p
	Et0/3               Desg FWD 100       128.4    P2p
	```

	```
	S3# sh spanning-tree
	
	VLAN0001
	  Spanning tree enabled protocol ieee
	  Root ID    Priority    32769
	             Address     aabb.cc00.1000
	             Cost        100
	             Port        3 (Ethernet0/2)
	             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
	
	  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
	             Address     aabb.cc00.3000
	             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
	             Aging Time  300 sec
	
	Interface           Role Sts Cost      Prio.Nbr Type
	------------------- ---- --- --------- -------- --------------------------------
	Et0/0               Altn BLK 100       128.1    P2p
	Et0/1               Altn BLK 100       128.2    P2p
	Et0/2               Root FWD 100       128.3    P2p
	Et0/3               Altn BLK 100       128.4    P2p
	```

	В результате работы STA корневыми были выбраны порты **S2{Et0/0}** и **S3{Et0/2}**.
	> Данные порты были выбраны корневыми т.к., если стоимости портов равны (а они равны), процесс сравнивает BID. Если BID равны (а они равны), для определения корневого моста используются приоритеты портов. Значение приоритета по умолчанию — 128. STA объединяет приоритет порта с номером порта, чтобы разорвать связи. Наиболее низкие значения являются предпочтительными. Таким образом в соответствиии с приоритетами и номерами рассматриваемых портов были выбраны вышеуказанные.


### Вопросы для повторения
---
* Какое значение протокол STP использует первым после выбора корневого моста, чтобы определить выбор порта?
	> Первым проверяется стоимость портов.

* Если первое значение на двух портах одинаково, какое следующее значение будет использовать протокол STP при выборе порта?
	> Далее проверяется Bridge ID.
* Если оба значения на двух портах равны, каким будет следующее значение, которое использует протокол STP при выборе порта?
	> Следующее значение для сравнения - приоритет порта (128 по умолканию) и номер порта.
