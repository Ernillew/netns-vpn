**netns-vpn readme**
----------------

English version
---------------

Comming soon

Russian version
---------------

В репозитории представлены файлы для запуска OpenVPN в выделенном network namespace при помощи systemd.
Работоспособность опробована на Debian Sid и Ubuntu 16.04.1 LTS.

Подразумевается, что OpenVPN уже установлен и его конфиг-файл лежит в /etc/openvpn, а имя конфиг-файла имеет вид NAME.conf, где NAME — используемое вами имя.

Для работы скопируйте файлы из репозитория в их места на файловой системе, исправьте значение переменной VPN_SERVER в vpn.env на IP-адрес вашего VPN-сервера и можете добавлять OpenVPN в загрузку командой

    $ sudo systemctl enable openvpn-ns@NAME.service

где NAME все то же имя вашего конфиг-файла OpenVPN. После этого можно запускать

    $ sudo systemctl start openvpn-ns@NAME.service

OpenVPN запуститься в выделенном network namespace по имени vpn.

Командой

    $ sudo ip netns exec vpn curl https://ip.lindon.pw

вы сможете проверить, что внутри netns теперь есть VPN и вы ходите с адреса своего сервера.
После чего командой типа

    $ dbus-launch sudo ip netns exec vpn su - USERNAME -c firefox

где USERNAME — имя вашего пользователя, можно запускать браузер который будет ходить через VPN, в то время, как вся остальная система будет ходить напрямую, как и прежде.

При написании конфигов использованы материалы расположенные по адресам https://schnouki.net/posts/2014/12/12/openvpn-for-a-single-application-on-linux/ и https://www.linux.org.ru/forum/admin/11591881.

Отдельная благодарность Сергею Воронову сподвигнувшему меня сделать конфиги
