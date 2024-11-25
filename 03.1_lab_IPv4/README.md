# Lab - Implement DHCPv4

### Topology

![topology](lab_05_topology.png)

### Addressing Table

![address_table](lab_05_addressing.png)


### Vlan Table

![vlan_table](lab_05_vlans.png)

# Цели

+ **Часть 1: Построение сети и настройка базовых параметров устройств**
+ **Часть 2: Настройка и проверка двух DHCPv4-серверов на R1**
+ **Часть 3: Настройка и проверка DHCP Relay на R2**

### **Предыстория / Сценарий**

Протокол динамической конфигурации хостов **(DHCP)** — это сетевой протокол, который позволяет сетевым администраторам управлять и автоматизировать назначение IP-адресов. Без **DHCP** для **IPv4** администратору приходится вручную назначать и настраивать IP-адреса, предпочтительные DNS-серверы и шлюзы по умолчанию. По мере роста сети это становится административной проблемой, особенно когда устройства перемещаются из одной внутренней сети в другую.

В данном сценарии компания увеличила свои масштабы, и сетевые администраторы больше не могут вручную назначать IP-адреса устройствам. **Наша задача** — настроить маршрутизатор **R1** для назначения IPv4-адресов в двух разных подсетях.

### Необходимые ресурсы

Я сделал эту лабораторную работу в EVE-NG, и в ней я использовал следующие устройства:

+ 2x Cisco vIOS Router
+ 2x Cisco vIOS Switch
+ 2x Virtual PC (VPCS)

## Instruction

## - **Часть 1: Построение сети и настройка базовых параметров устройств**

В **части 1** мы создадим топологию сети и настроим базовые параметры на хостах **PC-A**, **PC-B** и коммутаторах.

**Шаг 1**: Установим схему адресации

Разделим сеть **192.168.1.0/24** на подсети, чтобы удовлетворить следующие требования:

**a.** Одна подсеть, **«Подсеть A»**, поддерживающая **58 хостов** (клиентский VLAN на R1).

**Подсеть A:**
```
192.168.1.0/26 (.1 - 63)
```
Запишим первый IP-адрес в таблице адресации для R1 G0/1.100. Запишим второй IP-адрес в таблице адресации для S1 VLAN 200 и укажем соответствующий шлюз по умолчанию.

**b.** Одна подсеть, **«Подсеть B»**, поддерживающая **28 хостов** (Management VLAN на R1).

**Подсеть B:**
```
192.168.1.64/27 (.65-.95)
```
Запишим первый IP-адрес в таблице адресации для **R1 G0/1.200**. Запишим второй IP-адрес в таблице адресации для **S1 VLAN 1** и укажем соответствующий шлюз по умолчанию.

**c.** Одна подсеть, **«Подсеть C»**, поддерживающая **12 хостов** (клиентская сеть на **R2**).

**Подсеть C:**
```
192.168.1.96/28 (.97-.111)
```
Запишим первый IP-адрес в таблице адресации для **R2 G0/1**.

## **Шаг 2: Подключите сеть согласно топологии**

Подключите устройства, как показано на диаграмме топологии, и соедините их кабелями.

## **Шаг 3: Настройте базовые параметры для каждого маршрутизатора**

**a.** Присвойте маршрутизатору имя устройства.
```
hostname R1
```
**b.** Отключите поиск DNS, чтобы предотвратить попытки маршрутизатора переводить неправильно введенные команды, как будто это имена хостов.

```
no ip domain-lookup
```

**c.** Установите class как привилегированный EXEC пароль (шифрованный).

```
R1(config)# enable secret class
```

**d.** Установите cisco как пароль консоли и включите авторизацию входа.

```
R1(config)# line console 0
R1(config-line)# password cisco
R1(config-line)# login
```

**e.** Установите cisco как пароль VTY и включите авторизацию входа.

```
R1(config)# line vty 0 4
R1(config-line)# password cisco
R1(config-line)# login
```

**f.** Зашифруйте пароли в открытом виде.

```
R1(config)# service password-encryption
```

**g.** Создайте баннер, предупреждающий всех, кто получает доступ к устройству, что несанкционированный доступ запрещен.

```
R1(config)# banner motd $ Authorized Users Only! $
```

**h.** Сохраните текущую конфигурацию (running configuration) в файл начальной конфигурации (startup configuration).

```
R1# copy running-config startup-config
```

**i.** Установите на маршрутизаторе текущее время и дату.

```
R1# clock set 15:30:00 27 Aug 2019
```

## **Шаг 4: Настройка межвлановой маршрутизации на R1**

**a.** Активируйте интерфейс G0/0/1 на маршрутизаторе.

```
R1(config)# interface g0/1
R1(config-if)# no shutdown
R1(config-if)# exit
```

**b.** Настройте подинтерфейсы для каждого VLAN в соответствии с таблицей адресации IP. Все подинтерфейсы используют инкапсуляцию 802.1Q и получают первый доступный адрес из пула IP-адресов, который вы рассчитали. Убедитесь, что подинтерфейс для нативного VLAN не имеет назначенного IP-адреса. Добавьте описание для каждого подинтерфейса.

```
R1(config)# interface g0/1.100
R1(config-subif)# description Client VLAN
R1(config-subif)# encapsulation dot1q 100
R1(config-subif)# ip address 192.168.1.1 255.255.255.192
R1(config-subif)# interface g0/1.200
R1(config-subif)# encapsulation dot1q 200
R1(config-subif)# description Management VLAN
R1(config-subif)# ip address 192.168.1.65 255.255.255.224
R1(config-subif)# interface g0/1.1000
R1(config-subif)# encapsulation dot1q 1000 native
R1(config-subif)# description Native VLAN
```

**c.** Проверьте, что подинтерфейсы находятся в рабочем состоянии.

```
 show ip interface brief
```
```
R1#sho ip interface brief
Interface                  IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0         unassigned      YES unset  administratively down down
GigabitEthernet0/1         unassigned      YES unset  up                    up
GigabitEthernet0/1.100     192.168.1.1     YES manual up                    up
GigabitEthernet0/1.200     192.168.1.65    YES manual up                    up
GigabitEthernet0/1.1000    unassigned      YES unset  up                    up
GigabitEthernet0/2         unassigned      YES unset  administratively down down
GigabitEthernet0/3         unassigned      YES unset  administratively down down
```

## **Шаг 5: Настройте G0/0/1 на R2, затем G0/0/0 и статическую маршрутизацию для обоих маршрутизаторов**

Шаг 5: Настройте **G0/1** на **R2**, затем **G0/0** и статическую маршрутизацию для обоих маршрутизаторов

**a.** Настройте G0/1 на R2 с первым IP-адресом из подсети C, который вы рассчитали ранее.

```
R2(config)#int g0/1
R2(config-if)#ip add 192.168.1.97 255.255.255.240
R2(config-if)#no shut
R2(config-if)#
```

**b.** Настройте интерфейс **G0/0** для каждого маршрутизатора в соответствии с таблицей адресации **IP**, приведённой выше.
```
R1(config)#interface g0/0
R1(config-if)#ip address 10.0.0.1 255.255.255.252
R1(config-if)#no shutdown
R1(config-if)#
```
```
R2(config)#interface g0/0
R2(config-if)#ip address 10.0.0.2 255.255.255.0
R2(config-if)#no shutdown
R2(config-if)#
```

**c.** Настройте маршрут по умолчанию на каждом маршрутизаторе, указывающий на **IP-адрес** интерфейса **G0/0** другого маршрутизатора.

```
R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.0.2
R2(config)# ip route 0.0.0.0 0.0.0.0 10.0.0.1
```

**d.** Проверьте, что статическая маршрутизация работает, отправив ping на адрес **G0/0/1** маршрутизатора **R2** с маршрутизатора **R1**.

```
R1# ping 192.168.1.97
```

**e.** Сохраните текущую конфигурацию **(running configuration)** в файл начальной конфигурации **(startup configuration)**.

```
R1# copy running-config startup-config
```

## **Шаг 6: Настройка базовых параметров для каждого коммутатора.**

**a**. Присвойте имя устройству (коммутатору).

```
switch(config)# hostname S1
```

**b.** Отключите поиск **DNS (DNS Lookup)**, чтобы предотвратить попытки маршрутизатора переводить неправильно введенные команды, как будто это имена хостов.

```
S1(config)# no ip domain-lookup
```


**c.** Установите class как привилегированный EXEC пароль (шифрованный).

```
S1(config)# enable secret class
```

**d.** Установите cisco как пароль консоли и включите авторизацию входа.
```
S1(config)# line console 0
S1(config-line)# password cisco
S1(config-line)# login
```

**e.** Установите cisco как пароль VTY и включите авторизацию входа.
```
S1(config)# line vty 0 4
S1(config-line)# password cisco
S1(config-line)# login
```

**f.** Зашифруйте пароли в открытом виде.
```
S1(config)# service password-encryption
```

**g.** Создайте баннер, предупреждающий всех, кто получает доступ к устройству, что несанкционированный доступ запрещен.
```
S1(config)# banner motd $ Authorized Users Only! $
```

**h.** Сохраните текущую конфигурацию (running configuration) в файл начальной конфигурации (startup configuration).
```
S1(config)# exit
S1# copy running-config startup-config
```

**i.** Установите на коммутаторе текущее время и дату.
```
S1# clock set 15:30:00 27 Aug 2019
```

## **Шаг 7: Создание VLAN на S1.**

**Примечание**: На **S2** настроены только базовые параметры.

**a.** Создайте и назовите необходимые **VLAN** на коммутаторе S1 согласно таблице выше.
```
S1(config)# vlan 100
S1(config-vlan)# name Clients
S1(config-vlan)# vlan 200
S1(config-vlan)# name Management
S1(config-vlan)# vlan 999
S1(config-vlan)# name Parking_Lot
S1(config-vlan)# vlan 1000
S1(config-vlan)# name Native
S1(config-vlan)# exit
```


**b.** Настройте и активируйте интерфейс управления на **S1 (VLAN 200)**, используя второй IP-адрес из подсети, рассчитанной ранее. Дополнительно настройте шлюз по умолчанию на **S1**.
```
S1(config)# interface vlan 200
S1(config-if)# ip address 192.168.1.66 255.255.255.224
S1(config-if)# no shutdown
S1(config-if)# exit
S1(config)# ip default-gateway 192.168.1.65
```


**c.** Настройте и активируйте интерфейс управления на **S2** (VLAN 1), используя второй IP-адрес из подсети, рассчитанной ранее. Дополнительно настройте шлюз по умолчанию на **S2**.
```
S2(config)# interface vlan 1
S2(config-if)# ip address 192.168.1.98 255.255.255.240
S2(config-if)# no shutdown
S2(config-if)# exit
S2(config)# ip default-gateway 192.168.1.97
```

**d.** Назначьте все неиспользуемые порты на **S1** в **VLAN Parking_Lot**, настройте их в статическом режиме доступа и административно деактивируйте их. На **S2** административно деактивируйте все неиспользуемые порты.

**Примечание:** Команда **interface range** будет полезна для выполнения этой задачи с минимальным количеством команд.
```
S1(config)#int range g0/0-2,g1/0-2
S1(config-if-range)# switchport mode access
S1(config-if-range)# switchport access vlan 999
S1(config-if-range)# shutdown
S1(config-if-range)# exit

S1(config)#int range g0/0-2,g1/0-2
S2(config-if-range)# switchport mode access
S2(config-if-range)# shutdown
S2(config-if-range)# exit
```
## **Шаг 8: Назначение VLAN для соответствующих интерфейсов коммутатора.**

**a.** Назначьте используемые порты в соответствующие **VLAN** (указанные в таблице VLAN выше) и настройте их в статическом режиме доступа.
```
S1(config)#interface g1/3
S1(config-if)# switchport mode access
S1(config-if)# switchport access vlan 100
```


**b.** Проверим, что **VLAN** назначены на правильные интерфейсы.

```
S1#show vlan br

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Gi0/3
100  Clients                          active    Gi1/3
200  Management                       active
999  Parking_Lot                      active    Gi0/0, Gi0/1, Gi0/2, Gi1/0
                                                Gi1/1, Gi1/2
1000 Native                           active
1002 fddi-default                     act/unsup
1003 token-ring-default               act/unsup
1004 fddinet-default                  act/unsup
1005 trnet-default                    act/unsup
S1#

```

Почему интерфейс **G0/3** указан в **VLAN 1**?

Порт **G0/3** находится в **VLAN** по умолчанию и не был настроен как магистральный канал **(802.1Q trunk)**.

## **Шаг 9: Ручная настройка интерфейса F0/5 на S1 как магистрального канала (802.1Q trunk).**

**a.** Измените режим порта на интерфейсе, чтобы принудительно включить режим trunk.

**b.** В рамках конфигурации trunk установите нативный VLAN на 1000.

**c.** Также в рамках конфигурации trunk укажите, что VLAN 100, 200 и 1000 разрешены для прохождения через trunk.

**d.** Сохраните текущую конфигурацию (running configuration) в файл начальной конфигурации (startup configuration).

**e.** Проверьте статус trunk.

Какой IP-адрес будет у ПК, если они подключены к сети через DHCP на данный момент?
ПК автоматически настроят себе IP-адрес из диапазона Automatic Private IP Address (APIPA) в формате 169.254.x.x.