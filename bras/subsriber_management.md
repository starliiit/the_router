# Subscriber management. Управление подписчиками.

## TheRouter поддерживает следующие функции и технологии, необходимые для управления подпичиками и создания BRAS сервера.

 * L2/L3 connected subscribers
 * vlan per subsriber with ip unnumbered adresses
 * proxy arp
 * traffic shaping (Token bucket filter with extended burst value)
 * DHCP relay
 * redirect subsriber traffic based on multiple routing tables and PBR
 * radius/coa

# Варианты подключений подписчиков

## L2 connected subscribers

<img src="http://therouter.net/images/bras/l2_connected_subsc_overview.png">

Подписчики и раутер подключаются друг к другу через общий L2 широковещательный домен.
На раутере настраивается интерфейс, используемый для создания сессий подписчиков.
Этот интерфейс может использовать любой вариант инкапсуляции: untagged, dot1q, qinq.
В настройках интерфейса должен быть указан флаг l2_subs, разрешающий раутеру создание 
сессий подписчиков на этом интерфейсе. 

	vif add name v_subsc port 0 type qinq svid 2 cvid 121 flags l2_subs

Создание сессии подписчика инициируется входящим или исходящим 
неклассифицированным ip пакетом. Неклассифицированным пакетом считается пакет, ip адрес которого
(ip адрес назначения в случае исходящих пакетов или ip адрес источника для входящих пакетов)
не принадлежит ни одной из авторизованных сессий.

Для авторизации сессии используется radius протокол.
Запрос на авторизацию L2 subsriber ceccии включает следующие атрибуты:

todo

## L3 connected subscribers

<img src="http://therouter.net/images/bras/l3_connected_subsc_overview.png">

L2 трафик подписчиков терминируется на отдельном маршрутизаторе, который в свою очередь, 
подключается к the_router через любой доступный на нем интерфейс.
Этот интерфейс может использовать любой вариант инкапсуляции: untagged, dot1q, qinq.
В настройках интерфейса должен быть указан флаг l3_subs, разрешающий раутеру создание 
сессий подписчиков на этом интерфейсе.

	vif add name v21 port 0 type dot1q cvid 21 flags l3_subs

Создание сессии подписчика инициируется входящим или исходящим 
неклассифицированным ip пакетом. Неклассифицированным пакетом считается пакет, ip адрес которого
(ip адрес назначения в случае исходящих пакетов или ip адрес источника для входящих пакетов)
не принадлежит ни одной из авторизованных сессий.

Для авторизации сессии используется radius протокол.
Запрос на авторизацию L3 subsriber ceccии включает следующие атрибуты:

todo

## Vlan per subscriber

<img src="http://therouter.net/images/bras/vlan_per_subsc_overview.png">

Каждый подписчик подключается к сети через отдельный vlan (dot1q или qinq).
The_router подключается к сети через trunk порт, через который доступен
трафик всех вланов подписчиков. На этом порту the_router'a динамически создаются
интерфейсы (сессии) для каждого подписчика. Создание динамического интерфейса
подписчика происходит только по входящему неклассифицированному пакету. Неклассифицированным
считается пакет, vlan информация из заголовков которого, не соответсвует ни одному авторизованному
динамическому интерфейсу подписчика. Настройки порта the_router, к которому подключены подписчики, 
должны включать флаг dynamic_vif, разрещающий создание динамических интерфейсов подписчиков.

	port 0 mtu 1500 tpid 0x8100 state enabled flags dynamic_vif

### Авторизация создания динамических интерфейсов
Авторизация создания динамических интерфейсов подписчиков (сеcсий) выполняется с помощью radius протокола.
Запрос на авторизацию включает следующие атрибуты:

 * THE_ROUTER_VSA_OUTER_VID - Внешняя влан метка или service vlan id (svid). Задается всегда. Для dot1q интерфейса используется значение 0.
	
 * THE_ROUTER_VSA_INNER_VID - Внутренняя влан метка или customer vlan id (cvid).

 * THE_ROUTER_VSA_PORT_ID - Номер порта TheRouter.

 * Username - для динамических интерфейсов формируется на основе полей portid, svid и сvid
путем их объединения в одну строку с исользованием точки в качестве разделителя.
	
### Ответ на запрос авторизации
Ответ может включать перечисленные ниже атрибуты. В зависимости от наличия того или иного атрибута
после создания интерфейса к нему будут применены соответствующие политики и сервисы.

 * THE_ROUTER_VSA_IP_UNNUMBERED_VIF - наличие этого атрибута в ответе указывает на необходиость
настройки ip адресации созданного интерфейса по принципу ip unnumbered. Т.е. TheRouter добавляет
IPv4 адрес в список адресов интерфейса и создает маршрут к адресу пользователя через этот интерфейс.
Т.е. TheRouter выполняет следующие команды:

	ip addr add <GW_IP>/32 dev <dynamic_vif>
	ip route add <SUB_IP>/32 dev <dynamic_vif> src <GW_IP>
	
Где:
 * GW_IP - адрес TheRouter, одинаковый для всех динамических интерфейсов, использующих общую подсеть IPv4.
 * SUB_IP - адрес пользователя.
	
Адреса GW_IP,SUB_IP определяются атрибутами:
 * THE_ROUTER_VSA_IPV4 задает адрес SUB_IP. В качестве значения принимается целое положительное число.
 * THE_ROUTER_VSA_IPV4_MASK задает маску для SUB_IP. В качестве значения принимается целое положительное число. Например 24.
 * THE_ROUTER_VSA_IPV4_GW задает адрес GW_IP В качестве значения принимается целое положительное число.
	
арибут THE_ROUTER_VSA_IPV4_GW необязателен и может быть рассчитан как первый адрес в сети.
	
 * THE_ROUTER_VSA_INGRESS_CIR, THE_ROUTER_VSA_EGRESS_CIR -
атрибуты, задающие ограничение входящей и исходящей скорости для интерфейса.
В качестве значения ипользуется челое положительное число, задающее скрость в Мбит/c.

 * THE_ROUTER_VSA_PBR - атрибут, контролирующий настройку PBR маршрутизации для входящих пакетов
подписчика. Этот атрибут используется механизмом управления трафиком подписчиков, подробная
информация о котором находится в соответствующем разделе.

# Управление трафиком заблокированных подписчиков

С помощью дополнительной таблицы маршрутизации и правил Policy Based Routing (PBR),
определяющих какую таблицу маршрутизации использовать, TheRouter может
перенаправлять трафик заблокированных подписпичиков по отдельному маршруту.
Такая функция может быть использована, например, для перенаправления заблокированных
пользователей на web сайт, с информацией об учетной информации подписчика и его услугах.

Для использования функции перенаправления необходимо:

 * Создать отдельную таблицу маршрутизации
 * Заполнить созданную таблицу маршрутизации необходимыми маршрутами
 * Создать таблицы для хранения идентификаторов (id) заблокированных подписчиков
 * Создать PBR правила, использующих таблицы с id заблокированных подписчиков
и дополнительную таблицу маршрутизации
 * Иcпользовать PBR Radius атрибуты во время авторизации заблокированных подписчиков
либо использовать механизм CoA, чтобы TheRouter сохранил id заблокированного
подписчика в соответсвующей таблице

Таким образом, TheRouter в момент авторизации заблокированного пользователя
либо с помощью механизма CoA сохранит id заблокированного подписчика в специальную
таблицу. Затем, получив от пользователя входящий пакет, к нему применятся правила
pbr. Если какое-либо правило pbr обнаружит, что инфомация из пакета соответсвует 
одному из id, хранящихся в таблице заблокированных подписчиков, этот пакет будет 
смаршрутизирован согласно правилам таблицы маршрутизации, определяемым этим правилом.

### Дополнительные таблицы маршрутизации

	ip route table add <table name>
	Например,
	
	ip route table add blocked_subsc

## PBR правила

### Описание механизма PBR

PBR механизм хранит таблицу правил PBR.
Правила применяются к каждому входящему пакету и определяют какую таблицу
маршрутизации использовать для него.

Положение правила с таблице определяет его приоритет.
Просматриваются правила в порядке возрастания приоритета. Как только
найдено правило, соответсвующее пакету, для маршрутизации пакета используется
таблица, заданная в правиле, и дальнейший поиск правил pbr прекращается.

### PBR правила для L2/L3 connected подписчиков

Входящий трафик L2/L3 connected подписчиков идентифицируется по IP адресам источника,
поэтому для PBR правил необходимо использовать таблицы IP адресов, хранящие IP адреса
заблокированных подписчиков.

Например, следующая команда добавит правило pbr с приоритетом 10 в общий
список правил pbr. 
	
Правило использует в качестве критерия соответствия таблицу IP адресов
с именем ips1. Все пакеты, ip адреса источников которых, хранятся
в таблице ips1 будут маршрутизироваться с помощью таблицы маршрутизации blocked_subsc.

	ip pbr rule add prio 10 u32set ips1 type "ip" table blocked_subsc

### PBR правила для Vlan per subscriber подписчиков

Входящий трафик Vlan per subscriber подписчиков идентифицируется по vlan идентификаторам и номеру порта
на котором получен пакет.

Поэтому для PBR правил необходимо использовать таблицы L2, хранящие информацию о vlan индетификаторах и портах
заблокированных подписчиков (L2 ID).

Например, следующая команда добавит правило pbr с приоритетом 20 в список правил pbr. 
	
Правило использует в качестве критерия соответствия L2 ID таблицу 
с именем l2s1. Все пакеты, L2 информация которых, хранится
в таблице l2s1 будут маршрутизироваться с помощью таблицы blocked_subsc.

	ip pbr rule add prio 20 u32set l2s1 type "l2" table blocked_subsc

## Таблицы заблокированных подписчиков

### Таблицы с IP адресам заблокированных L2/L3 connected подписчиков
Пример создания таблицы для хранения ip адресов с именем "ips1"

	u32set create ips1 size 4096 bucket_size 16


### Таблицы с L2 идентификаторами заблокированных vlan per subscriber подписчиков
Пример создания таблицы для хранения L2 ID с именем "l2s1"

	u32set create l2s1 size 4096 bucket_size 16

## PBR Radius атрибуты

THE_ROUTER_VSA_PBR - атрибут, контролирующий настройку PBR маршрутизации для входящих пакетов подписчика.
В зависимости от значения атрибута в таблицу, используемую PBR правилами, добавляется или удаляется идентификатор 
динамического интерфейса. Значение 1 атрибута соответсвует операции добавления, 2 - удаления.
Имя таблицы, используемой этим механизмом, задается в конфигурационном файле директивой: 

	subsc u32set init <IP_TABLE_NAME> <L2_TABLE_NAME>

Где:

 * IP_TABLE_NAME - имя таблицы с IP адресами. Она используется для L2/L3 сессий, 
т.к. идентификатор таких сессий - IP адрес пользователя.
	
 * L2_TABLE_NAME - имя таблицы c L2 идентификаторами динамических интерфейсов.

Сами таблицы, правила PBR, использующие их, дополнтельные таблицы маршрутизации 
должны также быть заданы в конфигурационном файле до директивы subsc u32set init.

# Radius
## Общая настройка Radius клиента

	radius_client add server 192.168.3.2
	radius_client add src ip 192.168.3.1
	radius_client set secret "secret"

# Radius/COA

Механизм CoA может быть использован для:
 * изменения скорости подписчика
 * отключения подписчика
 * перенаправления трафика подписчика и перевода подписчика в группу заблокированных

Для этого в СoA могут быть использованы следующие атрибуты:

 * THE_ROUTER_VSA_PBR
 * THE_ROUTER_VSA_INGRESS_CIR
 * THE_ROUTER_VSA_EGRESS_CIR

## Изменение скорости подписчика

### Изменнение скорости подписчика vlan per subscriber

Пример изменения скорости подписчика на порту 0 в влане svid 10, cvid 20

	echo User-Name=0:10:20,User-Password=mypass,Vendor-Specific = "TheRouter,therouter_ingress_cir=101", Vendor-Specific = "TheRouter,therouter_engress_cir=102" | radclient 192.168.3.1:3799 coa secret	

	!! THE_ROUTER_VSA_INGRESS_CIR и THE_ROUTER_VSA_EGRESS_CIR должны быть определены вместе.

### Перенаправление трафика подписчика

	echo User-Name=0:0:130,User-Password=mypass,Vendor-Specific = "TheRouter,therouter_pbr=1" | radclient 192.168.3.1:3799 coa secret

где

therouter_pbr - код действия с pbr таблицей:

	1 - добавить
	2 - удалить

В результате id подписчика с именем '0:0:130' добавится в таблицу l2set, определенную
настройками the_router'a.

### Отключение подписчика


# DHCP relay

	dhcp_relay 192.168.3.2


# Различные команды 

## Просмотр авторизованных сессий подписчиков