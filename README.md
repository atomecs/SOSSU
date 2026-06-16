# SOSSU

Топология:
Наверху расположена машина ISP – это маршрутизатор, один интерфейс выходит в интернет, другой интерфейс идет в левую сеть ISP-HQ, другой интерфейс ISP идет в правую сеть ISP-BR. Эти интерфейсы связываются с роутерами HQ-RTR и BR-RTR соответственно.
Левая сеть: на входе роутер HQ-RTR, он имеет еще один интерфейс вниз в сеть HQ-NET и связывается с коммутатором HQ-SW, этот коммутатор HQ-SW также имеет интерфейсы и связывается с сетью SRV-NET с сервером HQ-SRV, и интерфейс в сеть CLI-NET с клиентом HQ-CLI.
Правая сеть: есть BR-RTR, нижний интерфейс идет в сеть FW-NET с межсетевым экраном BR-FW, который стоит на входе в правую сеть. Далее BR-FW имеет интерфейс, который ведет в сеть BR-NET, которая связывает сервер BR-SRV и клиент BR-CLI.
документация эко-роутер - https://docs.ecorouter.ru/EcoRouter-UserGuide.pdf


Настройка:
ЗАДАНИЕ 1, 12 - имена и системное время 
Альт - hostnamectl set-hostname isp.au-team.irpo; 
timedatectl set-timezone Europe/Moscow 

EcoRouter - hostname hq-rtr.au-team.irpo;
		ntp timezone UTC+3

На Ideco на этапе настройки

ЗАДАНИЕ 2. ISP, ЗАДАНИЕ 8

Чтобы ISP получал адрес по dhcp
В файле  /etc/net/ifaces/<интерфейс в интернет>/options

TYPE=eth
CONFIG_WIRELESS=no 
BOOTPROTO=dhcp
SYSTEMD_BOOTPROTO=dhcp4
CONFIG_IPV4=yes
DISABLED=no
NM_CONTROLLED=yes
SYSTEMD_CONTROLLED=no
ONBOOT=yes
DHCP_TIMEOUT=7

для альтов для получения адресов по dhcp команда - dhcpd

В файле /etc/net/sysctl.conf 
net.ipv4.ip_forward = 1
Но лучше: sysctl -w net.ipv4.ip_forward = 1

Для динамической сетевой трансляции можно использовать iptables. 
apt-get install iptables 
iptables –t nat –A POSTROUTING –o <ИМЯ_ВНЕШНЕГО_ИНТЕРФЕЙСА> –j MASQUERADE 
iptables-save >> /etc/sysconfig/iptables 
systemctl enable --now iptables 

 Как проверить? 
sysctl net.ipv4.ip_forward 
iptables –t nat –L –n –v 

Настройка ECO
Это прописываем на двух ECO
Заранее выбираем порт в сторону ISP - по умолчанию ge0

настройка
conf
interface <имя интерфейса>
ip address 172.16.1.2/28 (или другой)
exit
port ge0
service-instance 1 (номер любой, главное чтобы не соответсвовал вланам другим)
encapsulation untagged
connect ip interface <имя интерфейса>

это настройка интерфейса, так же надо настроить шлюзы
conf
ip route 0.0.0.0/0 172.16.1.1

ЗАДАНИЕ 3. Локальные учетные записи

useradd sshuser -u 2026
passwd sshuser

Для редактирования sudo можно воспользоваться командой visudo или явно открыть файл /etc/sudoers в  текстовом редакторе vim или nano, после чего следует найти и  раскомментировать строку WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL. 
gpasswd -a sshuser wheel

	ECO:
configure terminal
username net_admin
password P@ssw0rd
(если нужно назначить роль):
role admin
Доступные роли: 
admin — права администратора; 
helpdesk — привилегия поддержки; 
noc — привилегии оператора. 

ЗАДАНИЕ 4. Настройте коммутацию в сегменте HQ 
настройка eco
найти порт в сторону устройств (условно ge2 - у себя смотрите сами)
conf
interface ge2.100
ip address 10.10.100.1/27
ex
interface ge2.200
ip address 10.10.200.1/28
ex
interface ge2.999
ip address 10.10.30.1/29
ex
port ge2
service-instance 100
encapsulation dot1q 100 exact
rewrite pop 1
connect ip interface ge2.100
ex
service-instance 200
encapsulation dot1q 200 exact
rewrite pop 1
connect ip interface ge2.200
ex
service-instance 999
encapsulation dot1q 999 exact
rewrite pop 1
connect ip interface ge2.999
результатом должны быть поднятые интерфейсы
проверить show ip interface brief


аналогично и для друго устройства
ip address 10.20.10.1/30
exit
port ge1
service-instance 1 (номер любой, главное чтобы не соответсвовал вланам другим)
encapsulation untagged
connect ip interface <имя интерфейса>
СУТЬ В ЧЕМ - это работа с вланами, дальше на устройства адрес должен прилететь по dhcp, в процессе проверки по статике с обычным интерфейсов ниче не работало, получилось только с вланами - как их нормально настроить - я не ебу
Либо в настройках самого подключения - но как это сделать хзхзхз
На сервере HQ-SRV:
ip link add link ens3 name ens3.100 type vlan id 100
ip link set ens3.100 up
ip addr add 10.10.100.2/28 dev ens3.100
ip route del default via 10.10.100.1 dev ens3
ip route add default via 10.10.100.1 dev ens3.100

ЗАДАНИЕ 5. 
nano /etc/openssh/sshd_config 

MaxAuthTries 2 
Banner /etc/ssh/banner (Включаем баннер - по умолчанию строка закомментирована)

echo "Authorized access only" > /etc/ssh/banner
systemctl restart sshd 

Задание 6. ГРЕ 
на hq
conf
interface tunnel.0
ip address 10.10.10.1/30
ip tunnel 172.16.1.2 172.16.2.2 mode gre

на br
conf
interface tunnel.0
ip address 10.10.10.2/30
ip tunnel 172.16.2.2 172.16.1.2 mode gre

проверить пингом простым

ЗАДАНИЕ 7. OSPF
на hq
conf
#router ospf 1
passive-interface default
no passive-interface tunnel.0
network 10.10.10.0/30 area 0
network 10.10.100.0/27 area 0
network 10.10.200.0/28 area 0
network 10.10.30.0/29 area 0
area 0 authentication
ex
interface tunnel.0
ip ospf authentication-key P@ssw0rd
ex

	на br
ecorouter(config)#router ospf 1
ecorouter(config-router)#passive-interface default
ecorouter(config-router)#no passive-interface tunnel.0
ecorouter(config-router)#network 10.10.10.0/30 area 0
ecorouter(config-router)#network 10.20.10.0/30 area 0
ecorouter(config-router)#network 10.20.20.0/28 area 0
ecorouter(config-router)#network 10.20.30.0/29 area 0
ecorouter(config-router)#area 0 authentication
ecorouter(config-router)#ex
ecorouter(config)#interface tunnel.0
ecorouter(config-if-tunnel)#ip ospf authentication-key P@ssw0rd
ecorouter(config-if-tunnel)#ex

show ospf ?neighbors?
ЗАДАНИЕ 8. Динамическая трансляция:
определить названия интерфейсов в сторону isp (в нашем случае to_isp) и все остальные
interface to_isp
ip nat outside
ex
interface ge2.100
ip nat inside
ex
и это для всех - ge2.200 ge2.999 - ip nat inside

ip nat pool HQ 10.10.100.1-10.10.200.254
ip nat source dynamic inside-to-outside pool HQ overload interface  to_isp

на всех конечных устройствах:
проверить пинг до маршрутизатора
проверить маршрут по умолчанию до маршрутизатора - именно влан интерфейса - ip route add default via 10.10.100.1 dev ens3.100

ЗАДАНИЕ 9. DHCP 
ВАЖНО - смотри на интерфейсы - тут наоборот интерфейсы и адреса, везде надо менять на противоположный
hq-rtr.au-team.irpo(config)#ip pool HQ-cli
hq-rtr.au-team.irpo(config-ip-pool)#range 10.10.200.2-10.10.200.14
hq-rtr.au-team.irpo(config-ip-pool-range)#ex
hq-rtr.au-team.irpo(config-ip-pool)#ex
hq-rtr.au-team.irpo(config)#dhcp-server 1
hq-rtr.au-team.irpo(config-dhcp-server)#pool HQ-cli 1
hq-rtr.au-team.irpo(config-dhcp-server-pool)#mask 255.255.255.240
hq-rtr.au-team.irpo(config-dhcp-server-pool)#gateway 10.10.200.1
hq-rtr.au-team.irpo(config-dhcp-server-pool)#dns 10.10.100.2
hq-rtr.au-team.irpo(config-dhcp-server-pool)#domain-name au-team.irpo
hq-rtr.au-team.irpo(config-dhcp-server-pool)#ex
hq-rtr.au-team.irpo(config-dhcp-server)#exit
hq-rtr.au-team.irpo(config)#interface ge2.200
hq-rtr.au-team.irpo(config-if)#dhcp-server 1
Для того чтобы прилетел ip нужно создавать сабинтерфейс, в настройках самого интерфейса надо проставить dhcp (acc) (роляет это или нет я не знаю, но я делал так) и создать интерфейс командой

опять же смотри на вланы

ip link add link ens3 name ens3.200 type vlan id 200
ip link set ens3.200 up

ну и выдача происходит по команде dhcpd или dhclient - мб reboot

ЗАДАНИЕ 10. Настройте пользователей BR-FW:
Если там не будет лицензии:
/usr/share/ideco/license-backend/hwid - меняем 1 цифру
systemctl restart ideco-license-backend.service

Создайте пользователей mgmt и srv
перейти в модуль «Пользователи», затем в раздел «Учётные записи»;  нажать пункт «Добавить пользователя»; 
заполнить следующие поля: (2 лаба, 3 часть)

Для авторизации пользователя по IP-адресу необходимо выполнить действия: - создать пользователя для авторизации по IP в Ideco NGFW; - перейти в раздел Пользователи -> Учетные записи -> Карточка пользователя -> IP и MAC авторизация или Пользователи -> Авторизация -> IP и MAC авторизация: (ЛАБА 3)         

ЗАДАНИЕ 11. DNS на HQ-SRV
apt-get update && apt-get install bind -y 
nano /var/lib/bind/etc/options.conf 
# Слушать на всех сетевых интерфейсах IPv4
   listen-on { any; };
   # Отключаем IPv6, если он не используется
   listen-on-v6 { none; };
allow-query { any; };
forwarders {94.232.137.104; };


создание зон - 
nano /etc/bind/local.conf 
zone "au-team.irpo" { 
type master; 
file "au-team.irpo"; 
}; 


# Обратная зона (IP -> имена) для сети 10.10.100.0/27 (HQ-SRV, HQ-RTR) zone "100.10.10.in-addr.arpa" { 
type master; 
file "100.10.10.in-addr.arpa"; 
}; 


zone "200.10.10.in-addr.arpa" { 
type master; 
file "200.10.10.in-addr.arpa"; 
}; 

создадим файлы зон
[root@host-15 ~]# cd /var/lib/bind/etc/zone/
[root@host-15 zone]# cp empty au-team.irpo
[root@host-15 zone]# cp empty 100.10.10.in-addr.arpa
[root@host-15 zone]# cp empty 200.10.10.in-addr.arpa


[root@host-15 zone]# cat au-team.irpo 
; BIND reverse data file for empty rfc1918 zone
;
; DO NOT EDIT THIS FILE - it is used for multiple zones.
; Instead, copy it and use that copy.
;
$TTL	1D
@	IN	SOA	localhost. root.localhost. (
				2026052501	; serial
				12H		; refresh
				1H		; retry
				1W		; expire
				1H		; ncache
			)
	IN	NS	localhost.
# А-записи для инфраструктуры (Соответствие Имя -> IP)
hq-srv  IN  A   10.10.100.2
hq-rtr  IN  A   10.10.100.1
hq-cli  IN  A   10.10.200.2

# Записи для правого офиса BR и ISP
br-rtr  IN  A   172.16.2.2
br-fw   IN  A   10.20.10.2
br-srv  IN  A   10.20.20.2
br-cli  IN  A   10.20.30.2

[root@host-15 zone]# cat 100.10.10.in-addr.arpa
; BIND reverse data file for empty rfc1918 zone
;
; DO NOT EDIT THIS FILE - it is used for multiple zones.
; Instead, copy it and use that copy.
;
$TTL	1D
@	IN	SOA	localhost. root.localhost. (
				2026052500	; serial
				12H		; refresh
				1H		; retry
				1W		; expire
				1H		; ncache
			)
	IN	NS	localhost.
1   IN  PTR hq-rtr.au-team.irpo.
2   IN  PTR hq-srv.au-team.irpo.

[root@host-15 zone]# cat 200.10.10.in-addr.arpa
; BIND reverse data file for empty rfc1918 zone
;
; DO NOT EDIT THIS FILE - it is used for multiple zones.
; Instead, copy it and use that copy.
;
$TTL	1D
@	IN	SOA	localhost. root.localhost. (
				2026052500	; serial
				12H		; refresh
				1H		; retry
				1W		; expire
				1H		; ncache
			)
	IN	NS	localhost.
2   IN  PTR hq-cli.au-team.irpo.

Проверяем конфигурационный файл:
named-checkconf /var/lib/bind/etc/local.conf
# 1. Генерируем ключ сразу в chroot-директорию Альта 
rndc-confgen > /var/lib/bind/etc/rndc.key 
sed -i '6,$d' /var/lib/bind/etc/rndc.key 
chgrp -R named /var/lib/bind/etc/zone/

для проверки
[root@host-15 zone]# named-checkconf 
[root@host-15 zone]# named-checkconf -z

systemctl restart bind

проверка
nslookup hq-rtr.au-team.irpo 10.10.100.2 
nslookup 10.10.100.1 10.10.100.2 




