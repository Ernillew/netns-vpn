## Запуск отдельных приложений через OpenVPN без контейнеров и виртуализации

Оригинал статьи написан для публикации на сайте habrahabr.ru и находится [тут](https://habrahabr.ru/post/310646/ "тут"). Копия на sohabr находится [тут](https://sohabr.net/habr/post/310646/ "тут").

Как-то одним прекрасным утром я рассказывал в телеграмме бывшему другу и коллеге о том, что такое network namespaces в Linux и с чем его едят. Коллега восхитился, так же, как я, в свое время, а мне пришла в голову, что надо не костылить скриптом, как я делал до этого, а автоматизировать запуск отдельного network namespace и OpenVPN в нем. Так как я использую Debian Sid и Ubuntu 16.04 LTS автоматизацию я себе сделал в виде юнитов systemd, но об этом в конце статьи. После того, как я рассказал еще одному человеку, на этот раз далекому от IT, о возможности запускать только одно приложение, например браузер, под VPN, а остальные, как и прежде, он сказал «Только ради этого стоит перейти на Linux на компе», а я решил написать статью-инструкцию, как это сделать.

О том, что такое network namespaces в Linux написано много, но для тех кто не знает я кратко процитирую попавшееся под руку описание на русском языке:

«В linux относительно давно появилась такая замечательная вещь, как неймспейсы (namespaces). Основное применение данной технологии — контейнерная виртуализация, но и на маршрутизаторе можно придумать много разных применений, так как среди неймспейсов есть «network namespaces». Network namespaces позволяют в рамках одной машины в каждом неймспейсе иметь:

- свой набор таблиц маршрутизации (а их 2^31-1 шт)
- свою arp-таблицу
- свои правила iptables
- свои устройства (а значит и qdisc + class'ы tc)»

А теперь перейдем к теме нашей статьи.

Скрипт, для ручного поднятия network namespace и запуска в нем OpenVPN с комментированием:
```bash
#!/bin/bash
sudo ip netns add vpn # создаем namespace по имени vpn
sudo ip netns exec vpn ip addr add 127.0.0.1/8 dev lo # создаем в нем интерфейс lo
sudo ip netns exec vpn ip link set lo up # поднимаем loopback-интерфейс в netns
sudo ip link add vpn0 type veth peer name vpn1 # добавляем в системе виртуальный интерфейс, через который будет netns общаться с внешним миром
sudo ip link set vpn0 up # поднимаем созданный интерфейс
sudo ip link set vpn1 netns vpn up # создаем в netns интерфейс для общения во внешнем мире
sudo ip addr add 10.10.10.1/24 dev vpn0 # добавляем адрес на интерфейсе в системе
sudo ip netns exec vpn ip addr add 10.10.10.2/24 dev vpn1 # добавляем адрес на интерфейсе в netns
sudo ip netns exec vpn ip route add VPN_IP via 10.10.10.1 dev vpn1 # добавляем маршрут до VPN-сервера(замените VPN_IP на адрес вашего сервера)
sudo ip netns exec vpn ip route add default via 10.10.10.254 dev vpn1 # Добавляем адрес которого у нас нет в сети в качестве гейтвея, что бы OpenVPN мог заменить при запуске гейтвей на свой(если не будет никакого гейта, то OpenVPN не сможет назначить гейт и пакеты во вне у нас ходить через него не будут) 
sudo iptables -A INPUT ! -i vpn0 -s 10.10.10.0/24 -j DROP 
sudo iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o en+ -j MASQUERADE # настраиваем маскарадинг, замените en+ на wl+ если вы используете wifi-сетевуху для подключения к сети
sudo sysctl  -q net.ipv4.ip_forward=1 # разрешаем форвардинг пакетов
sudo mkdir -p /etc/netns/vpn # создаем директорию в которой у нас будет лежать resolv.conf для нашего netns 
echo "nameserver 8.8.8.8" |sudo tee /etc/netns/vpn/resolv.conf # прописываем гугловый 8.8.8.8 в качестве ДНСа
sudo ip netns exec vpn /usr/sbin/openvpn --daemon --writepid /run/openvpn/vpn.pid --cd /etc/openvpn/ --config vpn.conf  # Запускаем OpenVPN с конфиг-файлом /etc/openvpn/vpn.conf внутри netns
```
Выполнив этот скрипт мы можем при помощи команды:
```bash
$ sudo ip netns exec vpn curl http://ifconfig.me
```
Убедиться, что внутри netns у нас поднят OpenVPN и в сеть внутри netns мы выходим через наш OpenVPN. Теперь командой:
```bash
$ sudo ip netns exec vpn su - USER_NAME -c firefox
```
мы можем запустить браузер и получить браузер работающий через VPN, в то время, как вся остальная система у нас работает, как прежде и все остальное ходить напрямую(в команде замените USER_NAME на имя вашего пользователя). Пример запуска браузера приведен исходя из того, что на своем десктопе пользователь имеет sudo. Если кто-то может подсказать, как использовать ip netns exec без sudo буду признателен.

Аналогично браузеру вы можете запускать IM-клиенты, торрент-клиенты и все остальное. В случае, если firefox ругается при запуске, что не может подключиться к dbus поставьте в команде его запуска dbus-launch перед sudo.

Скрипт для остановки нашего netns:
```bash
#!/bin/bash
sudo ip netns pids vpn | xargs -rd'\n' sudo kill
sudo rm -rf /etc/netns/vpn
sudo sysctl -q net.ipv4.ip_forward=0
sudo iptables -D INPUT ! -i vpn0 -s 10.10.10.0/24 -j DROP
sudo iptables -t nat -D POSTROUTING -s 10.10.10.0/24 -o en+ -j MASQUERADE
sudo ip link del vpn0
sudo ip netns delete vpn
```
Юниты для systemd поднимающие все указанное на автомате при загрузке. Юнит для netns:
```
[Unit]
Description=Network namespace for VPN
After=syslog.target network.target
StopWhenUnneeded=true
RefuseManualStart=true
RefuseManualStop=true
 
[Service]
EnvironmentFile=/etc/netns/vpn.env
Type=oneshot
RemainAfterExit=true
ExecStart=/bin/ip netns add vpn
ExecStart=/bin/ip netns exec vpn ip addr add 127.0.0.1/8 dev lo
ExecStart=/bin/ip netns exec vpn ip link set lo up
ExecStart=/bin/ip link add vpn0 type veth peer name vpn1
ExecStart=/bin/ip link set vpn0 up
ExecStart=/bin/ip link set vpn1 netns vpn up
ExecStart=/bin/ip addr add ${NETWORK}.1/24 dev vpn0
ExecStart=/bin/ip netns exec vpn ip addr add ${NETWORK}.2/24 dev vpn1
ExecStart=/bin/ip netns exec vpn ip route add ${VPN_SERVER} via ${NETWORK}.1 dev vpn1
ExecStart=/bin/ip netns exec vpn ip route add default via ${NETWORK}.254 dev vpn1 
ExecStart=/sbin/iptables -A INPUT ! -i vpn0 -s ${NETWORK}.0/24 -j DROP
ExecStart=/sbin/iptables -t nat -A POSTROUTING -s ${NETWORK}.0/24 -o wl+ -j MASQUERADE
ExecStart=/sbin/sysctl -q net.ipv4.ip_forward=1
ExecStart=/bin/mkdir -p /etc/netns/vpn
ExecStart=/bin/sh -c "echo 'nameserver 8.8.8.8' > /etc/netns/vpn/resolv.conf"
 
ExecStop=/bin/rm -rf /etc/netns/vpn
ExecStop=/sbin/sysctl -q net.ipv4.ip_forward=0
ExecStop=/sbin/iptables -D INPUT ! -i vpn0 -s ${NETWORK}.0/24 -j DROP
ExecStop=/sbin/iptables -t nat -D POSTROUTING -s ${NETWORK}.0/24 -o wl+ -j MASQUERADE
ExecStop=/bin/ip link del vpn0
ExecStop=/bin/ip netns delete vpn
 
[Install]
WantedBy=multi-user.target
```
Юнит для OpenVPN:
```
[Unit]
Description=OpenVPN inside network namespace
Requires=vpnns.service
After=syslog.target network.target vpn-ns.service
 
[Service]
PrivateTmp=true
Type=forking
PIDFile=/var/run/openvpn/%i.pid
ExecStart=/bin/ip netns exec vpn /usr/sbin/openvpn --daemon --writepid /var/run/openvpn/%i.pid --cd /etc/openvpn/ --config %i.conf
 
[Install]
WantedBy=multi-user.target
```
И файл с переменными в котором я задаю адрес ВПН-сервера и сети используемой netns:
```bash
VPN_SERVER=1.1.1.1 # Change IP to your OpenVPN-server IP
NETWORK=10.10.10
```
Скопировав файлы для systemd в их места на файловой системе командой

```bash
$ sudo systemctl enable openvpn-ns@NAME.service
```
где NAME все то же имя вашего конфиг-файла OpenVPN. После этого можно запускать
```bash
$ sudo systemctl start openvpn-ns@NAME.service
```
OpenVPN запуститься в выделенном network namespace по имени vpn.
Командой
```bash
$ sudo ip netns exec vpn curl http://ifconfig.me
```
вы сможете проверить, что внутри netns теперь есть VPN и вы ходите с адреса своего сервера.

[Юниты systemd на github](https://github.com/Ernillew/netns-vpn "Юниты systemd на github").

Скрипты [vpnns_up.sh](https://gist.github.com/Ernillew/aa0a13e738d2165878111801c5144d18 "vpnns_up.sh") и[ vpnns_down.sh](https://gist.github.com/Ernillew/8b1d9f410806a56f374d5183c304ffcf " vpnns_down.sh") на gist.gihub.com

При подготовке статьи мне помогли две ссылки:

[http://schnouki.net/posts/2014/12/12/openvpn-for-a-single-application-on-linux](http://schnouki.net/posts/2014/12/12/openvpn-for-a-single-application-on-linux)

[https://www.linux.org.ru/forum/admin/11591881](https://www.linux.org.ru/forum/admin/11591881)

Отдельная благодарность Сергею Воронову ака Рэйсту, после разговора с которым я и решил сделать конфиги и написать статью.

P.S. Запуск VPN в выделенном network namespace может использоваться не только, как обход цензуры, но и для рабочих целей, я после того, как сделал юнит для OpenVPN сделал себе еще аналогичный поднимающий VPN до сети клиента, что бы можно было запускать с доступом к его сети только отдельные приложения.