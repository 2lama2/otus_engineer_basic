## Базовая настройка коммутатора
#### 1. Создание сети и проверка настроек коммутатора по умолчанию
##### a. Создаем сетевую топологию в EVE-NG
![[pics/Pasted image 20220423183710.png]]
##### Таблица адресации
**Устройство** | **Интерфейс** | **IP-адрес/префикс**
:---:|:---:|:---:
 SW | VLAN1 | 192.168.1.2/24
VPC | NIC | 192.168.1.10/24

Для первоначальной настройки коммутатора необходимо использовать консольное подключение, т.к. по умолчанию доступ к коммутатору по сети не настроен. Для эмуляции консольного подключения к устройствам в EVE-NG диспользуется ПО Putty.

##### b. Проверка настройки коммутатора по умолчанию

* В EVE-NG используем следующий образ операционной системы коммутатора:

>Cisco IOS Software, Linux Software (I86BI_LINUXL2-IPBASEK9-M), Experimental Version 15.2(20170809:194209) [dstivers-aug9_2017-high_iron_cts 101]

* Перейдем в привилегированный режим **`EXEC`** с помощью комманды **`enable`** и выведем информацию из текущей конфигурации об интерфейсах коммутатора:

>Switch#**show running-config | section interface**
>interface Ethernet0/0
>interface Ethernet0/1
>interface Ethernet0/2
>interface Ethernet0/3

а также о линиях управления и их настройках по умолчанию:

>Switch#**show run | section line**
>line con 0
> logging synchronous
>line aux 0
>line vty 0 4

* При настройках коммутатора по умолчанию или при сбросе к заводским настройкам, файл параметров загрузочной конфигурации **`startup-config`** будет пустым, в чем убедимся, выполнив команду:

>Switch#**show startup-config**
>startup-config is not present

* В EVE-NG для выбранного образа коммутатора по умолчанию не настраивается виртуальный интерфейс коммутатора. Поэтому для примера рассмотрим настройки по умолчанию в ПО Packet Tracer (далее PT) для коммутатора 2960:

>Switch#**show run | section interface**
>interface Vlan1
> no ip address
> shutdown

Конфигурация говорит нам, что создан интерфейс **`Vlan1`**. Он находится в выключенном состоянии и ему не назначен IP-адрес:

>Switch#**show ip interface vlan 1**
>Vlan1 is administratively down, line protocol is down
>  Internet protocol processing disabled

Посмотрим дополнительную информацию об интерфейсе **`Vlan1`**:

>Switch#**show interface vlan 1**
>Vlan1 is administratively down, line protocol is down
>  Hardware is *__CPU Interface__*, address is *__0001.6469.b22e__* (bia 0001.6469.b22e)
>  ..............

Данный интерфейс является виртуальным и ему назначен определенный MAC-адрес.

* При подключении нового устройства в порт коммутатора на консоли наблюдаем информационные сообщения об изменении статуса порта:

>Switch#
>%LINK-5-CHANGED: Interface FastEthernet0/6, changed state to up
>%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/6, changed state to up

- Чтобы узнать информацию о версии ОС коммутатора, выполним команду **`show version`:**

Switch | Ports | Model | SW Version | SW Image
:------:|:-----:|:-----:|:----------:|:----------:
1 | 26 | WS-C2960-24TT-L| 15.0(2)SE4 | C2960-LANBASEK9-M
>......................
>Base ethernet **MAC Address** : 00:01:64:69:B2:2E

* Чтобы узнать информацию о конкретном интерфейсе используем следующую команду:

>Switch#**show interfaces f0/6**
>FastEthernet0/6 is** **up**, line protocol is **up (connected)**
>  Hardware is Lance, address is 0060.5ca9.5106 (bia 0060.5ca9.5106)
>  ....................
>  **Full-duplex**, **100Mb/s**

В данном случае порт включен, работает в режиме full-duplex на скорости 100Mb/s. Если порт нужно отключить, то используется команда **`shutdown`**. Из выключенного состояния во включенное порт переводится командой **`no shutdown`**.

* Изучим настройки по умолчанию для сети VLAN 1:

>Switch#**show vlan name default**

VLAN | Name | Status | Ports
:----:|:------:|:-------:|:------:
1 | default | active | Fa0/1, Fa0/2, Fa0/3, Fa0/4
 | | | | Fa0/5, Fa0/6, Fa0/7, Fa0/8
 | | | | Fa0/9, Fa0/10, Fa0/11, Fa0/12
 | | | | Fa0/13, Fa0/14, Fa0/15, Fa0/16
 | | | | Fa0/17, Fa0/18, Fa0/19, Fa0/20
 | | | | Fa0/21, Fa0/22, Fa0/23, Fa0/24
 | | | | Gig0/1, Gig0/2

VLAN | Type | SAID | MTU | Parent | RingNo | BridgeNo | Stp | BrdgMode | Trans1 | Trans2
:----:|:-----:|:----:|:---:|:----:|:-----:|:----:|:---:|:----:|:-----:|:----:
1 | enet | 100001 | 1500 | - | - | - | - | - | 0 | 0

Из вывода команды следует, что сетью по умолчанию является сеть VLAN 1 и она имеет имя **`default`**. Данная сеть является сетью Ethernet, является активной и включает в себя все порты коммутатора.

* Просмотреть содержимое каталога или устройства хранения данных можно с помощь команды **`show DIRNAME`**, где DIRNAME - это имя устройства или каталог. Например, посмотреть содержимое флэш-памяти можно командой:

>Switch#**show flash**
>Directory of flash:/
>1 -rw- 4670455 `<no date>` **2960-lanbasek9-mz.150-2.SE4.bin**

На данном устройстве хранится образ ОС CISCO IOS.

#### 2. Настройка базовых параметров сетевых устройств
Настройку будем проводить в EVE-NG.

##### 1. Настройка базовых параметров коммутатора
* Перейдем в привилегированный режим выполнения команд EXEC, затем перейдем в режим глобальной конфигурации:

>Switch>**enable**
>Switch#**configure terminal**
>Switch(config)#

* Далее выполним базовые команды настройки коммутатора:

>Switch(config)#**no ip domain-lookup**
>Switch(config)#**hostname S1**
>S1(config)#**service password-encryption**
>S1(config)#**enable secret class**
>S1(config)#**banner motd #**
**Unauthorized access is strictly prohibited. #**

* Назначим виртуальному интерфейсу коммутатора IP-адрес для возможности подключения к коммутатору и конфигурирования его по сети:

>S1(config)#**interface vlan 1**
>S1(config)#**ip address 192.168.1.2 255.255.255.0**

* Настроим каналы виртуального соединения (vty) для удаленного управления, чтобы коммутатор разрешил доступ через Telnet. Если не настроить пароль VTY, будет невозможно подключиться к коммутатору по протоколу Telnet.

>S1(config)#**line vty 0 4**
>S1(config-line)#**password cisco**
>S1(config-line)#**login**
>S1(config-line)#**transport input telnet**

* Настроим парольный доступ к консольному подключению:

>S1(config)#**line console 0**
>S1(config-line)#**password cisco**
>S1(config-line)#**login**

Для удобства работы в консоли включим параметр:

>S1(config-line)#**logging synchronous**

* Сохраним конфигураию копированием текущей (running) в загрузочную (startup):

>S1#**copy running-config startup-config**

##### 2. Настройка IP-адреса на компьютере PC-A

>VPCS> **ip 192.168.1.10 255.255.255.0**
>VPCS> **show ip**
>NAME        : VPCS[1]
>IP/MASK     : 192.168.1.10/24
>GATEWAY     : 255.255.255.0
>DNS         :
>MAC         : 00:50:79:66:68:02

#### 3. Проверка сетевых подключений
##### 1. Отображение конфигурации коммутатора

>S1#show running-config
Building configuration...
Current configuration : 1078 bytes
!
! Last configuration change at 22:24:13 EET Sat Apr 23 2022
!
version 15.2
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption
service compress-config
!
hostname S1
!
boot-start-marker
boot-end-marker
!
!
enable secret 5 $1$ZWl1$Ghu2SSzgaaCbCV.hKuC800
!
no aaa new-model
clock timezone EET 2 0
!
!
!
!
!
no ipv6 cef
!
!
!
!
!
no ip domain-lookup
!
!
ip cef
!
!
!
spanning-tree mode pvst
spanning-tree extend system-id
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!
interface Ethernet0/0
!
interface Ethernet0/1
!
interface Ethernet0/2
!
interface Ethernet0/3
!
interface Vlan1
 ip address 192.168.1.2 255.255.255.0
!
ip forward-protocol nd
!
no ip http server
ip ssh server algorithm encryption aes128-ctr aes192-ctr aes256-ctr
ip ssh client algorithm encryption aes128-ctr aes192-ctr aes256-ctr
!
!
!
!
!
!
control-plane
!
banner motd ^C
Uauthorized access is strictly prohibited. ^C
!
line con 0
 password 7 104D000A0618
 logging synchronous
 login
line aux 0
line vty 0 4
 password 7 0822455D0A16
 login
 transport input telnet
!
!
!
end

Проверим параметры VLAN 1:

>S1#**show interfaces vlan 1**
>Vlan1 **is up**, line protocol **is up**
>  Hardware is Ethernet SVI, address is aabb.cc80.1000 (bia aabb.cc80.1000)
>  Internet address is 192.168.1.2/24
>  MTU 1500 bytes, **BW 1000000 Kbit/sec**, DLY 10 usec,
>      reliability 255/255, txload 1/255, rxload 1/255
>  Encapsulation ARPA, loopback not set
>  Keepalive not supported
>  ARP type: ARPA, ARP Timeout 04:00:00

##### 2. Проверка сквозного соединения с помощью ЭХО-запроса

>VPCS> **ping 192.168.1.10**
>
>192.168.1.10 icmp_seq=1 ttl=64 time=0.001 ms
>192.168.1.10 icmp_seq=2 ttl=64 time=0.001 ms
>192.168.1.10 icmp_seq=3 ttl=64 time=0.001 ms
>192.168.1.10 icmp_seq=4 ttl=64 time=0.001 ms
>192.168.1.10 icmp_seq=5 ttl=64 time=0.001 ms

>VPCS> ping 192.168.1.2
>
>84 bytes from 192.168.1.2 icmp_seq=1 ttl=255 time=0.317 ms
>84 bytes from 192.168.1.2 icmp_seq=2 ttl=255 time=0.345 ms
>84 bytes from 192.168.1.2 icmp_seq=3 ttl=255 time=0.315 ms
>84 bytes from 192.168.1.2 icmp_seq=4 ttl=255 time=0.434 ms
>84 bytes from 192.168.1.2 icmp_seq=5 ttl=255 time=0.485 ms

##### 3. Проверка доступности коммутатора  по сети
Проверим доступность коммутатора по протоколу telnet, добавив еще одну виртуальную машину с адресом 192.168.1.20 в топологию EVE-NG:

![[Pasted image 20220424000328.png]]

## Вопросы для повторения
1. Зачем необходимо настраивать пароль VTY для коммутатора?
	>Без задания пароля для виртуальных линий невозможно включить сервисы TELNET или SSH.
2. Что нужно сделать, чтобы пароли не отправлялись в незашифрованном виде?
	>Для предотвращения передачи пароля в открытом виде при удаленном подключении к комутатору, необходимо использовать службу SSH. Службу TELNET не разрешать для использования.

