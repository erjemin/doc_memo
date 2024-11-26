# Kubernetes на Orange Pi 5 Plus под управлением Ubuntu 20.04 (на других Orange Pi, Raspberry Pi и других SBC тоже должно работать)

## Подготовка

### Установим DBus и Avahi

DBus — система межпроцессного взаимодействия, которая позволяет различным приложениям и службам в системе общаться
друг с другом. DBus часто используется для управления службами, взаимодействия с системными демонами и упрощения
интеграции приложений.

Avahi — это демон и утилиты для работы с mDNS/DNS-SD (Bonjour) и реализация протокола Zeroconf. Avahi тесно
интегрирован с D-Bus и используется для обнаружения устройств и сервисов в локальной сети и предоставляет
автоматическое обнаружение устройств и сервисов в локальной сети. Например, Avahi используется для обнаружения сетевых
принтеров, файловых серверов и других ресурсов без необходимости ручной настройки. Нам avahi понадобится для
обнаружения хостов кластера в локальной сети. 

```shell
sudo apt update
sudo apt install dbus avahi-daemon avahi-utils
```

Запускаем эти сервисы:
```bash
sudo systemctl start dbus
sudo systemctl start avahi-daemon
```

А также включаем автозапуск при загрузке:
```shell
sudo systemctl enable dbus
sudo systemctl enable avahi-daemon
```

#### Возможные проблемы с DBus

При проверке статуса dbus `sudo systemctl status dbus`, случается, выскакивают предупреждения, типа:
```text
Xxx xx xx:xx:xx _xxx-hostname-xxx_ dbus-daemon[909]: Unknown username "whoopsie" in message bus configuration file
```

Это связано с тем, что в конфигурационном файле DBus указаны несуществующий пользователь _whoopsie_. Это системный
пользователь, используемый сервисом Whoopsie, отвечающий за отправку отчётов об ошибках на серверы разработчиков
(Canonical) в системах Ubuntu. Если whoopsie или его конфигурация установлены, но пользователь отсутствует, возникают такие предупреждения.

Я не возражаю против отправки отчётов об ошибках, тем более, что сервис whoopsie у меня не запущен. :) Так что просто
добавлю пользователя _whoopsie_ в систему:
```shell
sudo adduser --system --no-create-home --disabled-login whoopsie
```

Но если внутренний параноик вам шепчет, что это не безопасно, то нужно найти в каких конфигурационных файлах DBus встречается _whoopsie_ `grep -r "whoopsie" /etc/dbus-1/` и закомментировать или удалить соответствующие строки. 

Также, порадовать своего внутреннего параноика, можно отключить отправку отчётов об ошибках в Ubuntu:
```shell
sudo systemctl stop whoopsie
sudo apt-get remove --purge whoopsie
```

После всех упражнений (добавления пользователя _whoopsie_ или, наоборот истребления его, и отключения сервиса
whoopsie) перезагрузим D-Bus:
```shell
sudo systemctl restart dbus
```

Теперь проверкак статуса dbus:
```shell
sudo systemctl status dbus
```

Не должна выдавать предупреждений:
```text
 dbus.service - D-Bus System Message Bus
     Loaded: loaded (/lib/systemd/system/dbus.service; static)
     Active: active (running) since Sat XXXX-XX-XX XX:XX:XX MSK; 1min ago
TriggeredBy: ● dbus.socket
       Docs: man:dbus-daemon(1)
   Main PID: 8437 (dbus-daemon)
      Tasks: 1 (limit: 18675)
     Memory: 616.0K
        CPU: 112ms
     CGroup: /system.slice/dbus.service
             └─8437 @dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation --syslog-only

XXX XX XX:XX:XX _xxx-hostname-xxx_ systemd[1]: Started D-Bus System Message Bus.
```

#### Возможные проблемы с Avahi

При проверке статуса avahi `sudo systemctl status avahi-daemon`, случается, выскакивают предупреждения, типа:
```text
Xxx xx xx:xx:xx _xxx-hostname-xxx_ avahi-daemon[2079]: Failed to parse address 'fe80::1%xxxxxxxx', ignoring.
```

Я не понял как это исправить и почему локальная петля (loopback) для iv6 `fe80::1` -- проблема. Отключение
обслуживания IPv6 для avahi в конфиге `/etc/avahi/avahi-daemon.conf` не помогло. Ставил в нем `use-ipv6=no`,
но предупреждения продолжались. Но, вроде, это не критично, но...

|                                               |
|:----------------------------------------------|
| **СООБЩАЙТЕ, ЕСЛИ ЗНАЕТЕ КАК ЭТО ИСПРАВИТЬ!** |
|                                               |

Пока я нашел следующее решение (по карйне мере у меня сработало, и сработало только если его проделать после всех
предыдущих пунктов по установке `avahi-daemon` вручную). Порядок действий напоминает шаманство:

Запускаем конфигуратор Orange Pi:
```shell
sudo orangepi-config
```

Панель orangepi-config на Orange Pi 5 выглядит так:

![Панель orangepi-config на Orange Pi 5 выглядит так](../images/orange--orange-config.gif)

Выбираем пункт **'System: System and security settings'** и заходим в панель **'System Settings'**. Выбираем в ней пункт
'**Avahi: Announce system in the network**':

![Панель 'System: System and security settings' в Orange Pi 5, выбран пункт 'Avahi: Announce system in the network'](../images/orange--orange-config--system-settings--avahi-01.gif)

Сервис устанавливается.

![Устанавливается и конфигурируется avahi-demon](../images/orange--orange-config--avahi-installing.gif)

Возможно, на панели 'System Setting' вместо пункта 'Avahi: Announce system in the network' будет пункт 'Avahi: Disable
system announcement in the network':

![Устанавливается и конфигурируется avahi-demon](../images/orange--orange-config--system-settings--avahi-02.gif)

Всё равно выбираем его: сначала отключаем avahi-демон; после возвращаемся в '**System Settings**'; повторно выбираем пункт
'**Avahi: Announce system in the network**' и устанавливаем avahi-демон заново через '**Avahi: Announce system in the
network**'... Всё как у настоящих системщиков -- надо "выйти и зайти".

Покидаем orangepi-config (Back и затем Exit) и перезагружаем Orange Pi:
```shell
sudo reboot
```

После перезагрузки предупреждения о проблемах  loopback для iv6 (`fe80::1`) в avahi должны исчезнуть.
```shell
sudo service avahi-daemon status
```

Все чисто. Магия!

------
## Настройка сети

Мой домашний роутер выдает IP-адреса через DHCP. Можно настроить узлы кластера (наши Orange Pi) на статические
IP-адреса, но чтобы DHCP-сервер случайно не выдавал такой адрес другим устройствам, надежнее настроить в DHCP
резервирование IP-адресов для узлов кластера. 

Для этого надо узнать MAC- и IP-адреса Orange Pi. На Ubuntu это можно сделать, например, с помощью команды `ifconfig`.
Увидим что-то вроде этого:
```text
...
...

enP4p65s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.110  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::1e2f:65ff:fe49:3ab0  prefixlen 64  scopeid 0x20<link>
        ether 1c:2f:65:49:3a:b0  txqueuelen 1000  (Ethernet)
        RX packets 656166  bytes 157816045 (157.8 MB)
        RX errors 0  dropped 12472  overruns 0  frame 0
        TX packets 44578  bytes 4805687 (4.8 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        
...
```

MAC-адрес: `ether 1c:2f:65:49:3a:b0`
IP-адрес: `inet 192.168.1.110`

И кстати, на Orange Pi 5 Plus есть два сетевых интерфейса: `enP4p65s0` и `enP3p49s0`. Так что, возможно, надо
зарезервировать IP для обоих интерфейсов.



## Установим Docker и Kubernetes

Для начала надо установить GPG-ключи репозитория Docker и Kubernetes. Установка GPG-ключей для Docker подробна 
описана в [отдельной инструкции](docker/docker-trusted-gpg.md). Для GPG-Kubernetes ключи устанавливаются похожим
образом. Скачиваем GPG-ключ в папку `/etc/apt/trusted.gpg.d/`:
```shell
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/kubernetes-apt-keyring.gpg
```

Добавляем репозиторий Kubernetes (с указанием этого GPG-ключа и ARM-платформы, ведь у нас Orange Pi 5 Plus на ARM):
```shell
echo 'deb [arch=arm64 signed-by=/etc/apt/trusted.gpg.d/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Готово. Теперь обновим список пакетов:
```shell
sudo apt update
```


