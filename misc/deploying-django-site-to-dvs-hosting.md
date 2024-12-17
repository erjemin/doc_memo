# Инструкция по развёртыванию Django-проекта на VDS-хостинге (_виртуальная машина_) — Django/MariaDB/uWSGI/nginx 

## 1: Создание пользователя

Изначально есть только root-доступ. Если мы залогированы под **root** (например, через KVM), то следует создать
пользователя от имени  которого мы будем осуществлять все действия (позже root-доступ будет закрыт).
Создадим пользователя (например **your_username**, но можно и другое имя) и дадим ему права администратора.

```shell
sudo useradd -c 'WEB-user' -m your_username
```

Присвоим ему пароль:
```shell
sudo passwd your_username
```

Дадим ему права работать от имени root. Для этого откроем на редактирование конфигурационный файл `/etc/sudoers`:
```shell
nano /etc/sudoers
```

и после строки `root ALL=(ALL:ALL) ALL` добавим в него строку:
```editorconfig
your_username     ALL=(ALL:ALL) ALL
```

Сохраняем конфигурационный файл `Ctrl+O` и `Enter`, а выходим из редактора `Ctrl+X`.

**ВАЖНО**: Если по какой-то причине файл `/etc/sudoers` пустой — это означает, что служба sudoers не установлена. 
Её нужно установить: `apt-get install sudo`.

Разлогируемся оз под root.
```shell
logout
```

Теперь можно залогироваться от имени пользователя **your_username**. 

Установим командную оболочку bash для пользователя:
```shell
chsh -s /bin/bash
```

## 2: Настройка SSH и окружения клиента

### Ограничиваем доступ root и повышение безопасности SSH

После завершения перезагрузки произведённой в конце предыдущего этапа мы можем заходить на наш сервер по SSH под новым, только что созданным, пользователем [user] и паролем [user_pwd]. Теперь следует сделать небольшие изменения в файле `/etc/ssh/sshd_config` — конфигурации ssh-доступа:

```shell
sudo nano /etc/ssh/sshd_config
```

Полное описание настроек sshd_config [находится на сайте Ubuntu](https://help.ubuntu.ru/wiki/ssh). Там содержатся 
вполне разумные рекомендации по увеличению безопасности. Откроем на редактирование конфигурационный файл ` 
/etc/ssh/sshd_config`:
```shell
sudo nano /etc/ssh/sshd_config
```

И изменим порт на котором будет отвечать SSH (например на 2002):
```ini
Port 2002
```

Так же дописываем в конце следующие две строки в которых и запрещаем ssh-вход 
пользователя **root** и разрешаем доступ нашему пользователю **your_username**:
```
DenyUsers root
AllowUsers your_username
```

Сохраняем конфигурационный файл `Ctrl+O` и `Enter`, а выходим из редактора `Ctrl+X`.

Чтобы настройки подействовали нужно перезапустить ssh-сервис:
```shell
sudo service ssh restart
```

Проверим, что сервис корректно перезапустился:
```shell
sudo service sshd status
```

Увидим, что все работает:
```text
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/lib/systemd/system/ssh.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2022-05-25 01:01:24 MSK; 2 days ago
       Docs: man:sshd(8)
             man:sshd_config(5)
   Main PID: 647 (sshd)
      Tasks: 1 (limit: 1066)
     Memory: 5.8M
     CGroup: /system.slice/ssh.service
             └─647 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
```

Разлогируемся — `logout`. Входим ещё раз и проверяем, что теперь пользователя root больше в систему по SSH не пускают.
Логируется вновь созданным пользователем `your_username`.

### Упрощение идентификации сервера при входе

В конфигурационном файле `/etc/ssh/sshd_config` есть специальный параметр (`Banner`), указывающий на файл который будет
отображён при логировании (по умолчанию `/etc/ssh_banner`). Его отображение упрощает идентификацию сервера при входе:
вам будет сложнее спутать этот сервер с другими, особенно когда работа идёт одновременно с терминалами нескольких
серверов. Но этот баннер отображается не только при входе через SSH, но и по SFTP. Существует более радикальное
решение, отображающееся при входе по только по KVM и SSH. И оно упростит идентификацию даже если мы работаем
с KVM-консоли. Это файл `/etc/motd` (Message Of The Day).

Очистим `/etc/motd`, и поместим туда какой-нибудь красивый ASCII-арт текст из ASCII-генератора. :
```shell
sudo nano /etc/motd
```

Вот несколько ссылок на онлайн-генераторы текстового ASCII-арта:
* [Текст-арт 1](https://fsymbols.com/text-art/).
* [Текст-арт 2](https://patorjk.com/software/taag/).
* [Коллекция ASCII-арт 1](https://www.asciiart.website/).
* [Коллекция ASCII-арт 2](https://www.textartcopy.com/ascii-art.html).

Например, вот такой баннер:
```
██████╗░░░░░░██╗░█████╗░███╗░░██╗░██████╗░░█████╗
██╔══██╗░░░░░██║██╔══██╗████╗░██║██╔════╝░██╔══██╗
██║░░██║░░░░░██║███████║██╔██╗██║██║░░██╗░██║░░██║
██║░░██║██╗░░██║██╔══██║██║╚████║██║░░╚██╗██║░░██║
██████╔╝╚█████╔╝██║░░██║██║░╚███║╚██████╔╝╚█████╔╝
╚═════╝░░╚════╝░╚═╝░░╚═╝╚═╝░░╚══╝░╚═════╝░░╚════╝ Project
```
Сохраняем баннер `Ctrl+O` и `Enter`, а выходим из редактора `Ctrl+X`.

### Раскрашиваем оболочку bash

Чтобы сделать красочно-разноцветным нашу командную строку (это тоже помогает идентифицировать сервер... например
красный цвет — это сервер продакшена, а зелёный — тестового), откроем на редактирование файл настроек оболочки bash
пользователя `.bashrc`:
```shell
nano ~/.bashrc
```
Находим там строку `#force_color_prompt=yes` раскомментируем её (и удаляем в ней #). И чтобы совсем расцветить,
находим блок:

```bash
if [ "$color_prompt" = yes ]; then
    PS1='${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
else
    PS1='${debian_chroot:+($debian_chroot)}\u@\h:\w\$ '
fi
```

И меняем на блок
```bash
if [ "$color_prompt" = yes ]; then
    PS1='${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u\[\033[01;33m\]@\[\033[01;32m\]\h\[\033[00m\]:\[\033[00;34m\]\w\[\033[00m\]\$ '
else
    PS1='${debian_chroot:+($debian_chroot)}\u@\h:\w\$ '
fi
```

Раскраска будет такая:
* `\[\033[01;32m\]` — CSI-код (ESC-последовательность) устанавливает цвет текста: `01` — жирный шрифт, `32` — зелёный цвет. Другие цвеита:
  * `30` — черный
  * `31` — красный
  * `32` — зелёный
  * `33` — жёлтый
  * `34` — синий
  * `35` — пурпурный
  * `36` — голубой
  * `37` — белый
  Правила раскраски и управлением текста через CSI-коды см. [статью в Wikipedia](https://ru.wikipedia.org/wiki/Управляющие_последовательности_ANSI). 
* `\u` — имя пользователя
* `\[\033[01;33m\]` — CSI-код устанавливает цвет текста: `01` — жирный шрифт, `33` — жёлтый цвет
* `@` — символ `@`
* `\[\033[01;32m\]` — CSI-код устанавливает цвет текста: `01` — жирный шрифт, `32` — зелёный цвет
* `\h` — имя хоста
* `\[\033[00m\]` — CSI-код сбрасывает цвет текста
* `:` — символ двоеточие
* `\[\033[00;34m\]` — CSI-код устанавливает цвет текста: `00` — обычный шрифт, `34` — синий цвет
* `\w` — текущий рабочий каталог
* `\[\033[00m\]` — символ ESC сбрасывает цвет текста
* `\$` — символ `$` или `#` в зависимости от того, является ли пользователь администратором
* ` ` — пробел

Всё! Перелогируемся чтобы настройки подействовали.

```shell
logout
```

## 3. Настраиваем службу времени (необязательно)

Устанавливаем службу точного времени `ntp`:
```bash
sudo apt-get install ntp
```

 Откроем на редактирование конфигурационный файл `/etc/ntp.conf`:
```shell
sudo nano /etc/ntp.conf
```

Заменяем буржуйские адреса серверов точного времени на отечественные: 
```editorconfig
# Specify one or more NTP servers.

# Use servers from the NTP Pool Project. Approved by Ubuntu Technical Board
# on 2011-02-08 (LP: #104525). See http://www.pool.ntp.org/join.html for
# more information.
pool ntp.msk-ix.ru iburst prefer
pool 194.190.168.1 iburst
pool 2001:6d0:ffd4::1 
# pool ntp.ix.ru
# pool 1.ubuntu.pool.ntp.org iburst
# pool 2.ubuntu.pool.ntp.org iburst
# pool 3.ubuntu.pool.ntp.org iburst

# Use Ubuntu's ntp server as a fallback.
pool ntp.msk-ix.ru
```

Сохраняем баннер `Ctrl+O` и `Enter`, а выходим из редактора `Ctrl+X`.

Чтобы настройки подействовали сервис точного времени нужно перезапустить:
```shell
sudo service ntp restart
```

Проверим, что сервис корректно перезапустился:
```shell
sudo service ntp status
```

Увидим, что все ок:
```text
● ntp.service - Network Time Service
     Loaded: loaded (/lib/systemd/system/ntp.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2022-05-25 01:01:24 MSK; 2 days ago
       Docs: man:ntpd(8)
   Main PID: 642 (ntpd)
      Tasks: 2 (limit: 1066)
     Memory: 1.5M
     CGroup: /system.slice/ntp.service
             └─642 /usr/sbin/ntpd -p /var/run/ntpd.pid -g -u 112:119
```

Проверить какой сервер точного времени ближе, задержки между серверами и т.п.:
```shell
ntpq -p
```

Увидим что-то типа:
```text
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 ntp.msk-ix.ru   .POOL.          16 p    -   64    0    0.000    0.000   0.000
 194.190.168.1   .POOL.          16 p    -   64    0    0.000    0.000   0.000
 2001:6d0:ffd4:: .POOL.          16 p    -   64    0    0.000    0.000   0.000
 ntp.ix.ru       .POOL.          16 p    -   64    0    0.000    0.000   0.000
*ntp.ix.ru       .GPS.            1 u  450 1024  377    2.442   -0.254   0.117
```

Немного дополнительных сведений о настройках NTP [можно найти в заметке о k8s на Orange Pi)[../raspberry-and-orange-pi/k8s.md#настойка-времени].

## 4. Настраиваем брандмауэр для использования только с допустимыми внешними интерфейсами (защита портов)

Устанавливаем iptables для управления IP-соединения и iptables-persistent для сохранения конфигураций настроенных
соединений и их автоматического подключения после перегрузке компьютера.
```shell
sudo -S apt-get install iptables
```

Далее создадим в домашней папке bash-скрипт `prepare-iptable.sh`:
```shell
nano ~/prepare-iptable.sh
```

...и вставим в него нижеследующий текст (исправьте если необходимо, следуя комментариям в скрипте):
```shell
#!/bin/bash

# You have to set the rules of your firewall on your server only with the
# services used outside the VM.

# Вы должны установить правила брандмауэра на своем сервере только
# с сервисами, используемыми вне виртуальной машины.

echo "Сетевые настройки: разрешения трафика и портов";
echo "";
echo "ЭТОТ СКРИПТ НУЖНО ЗАПУСКАТЬ С ПРАВАМИ АДМИНИСТРАТОРА (ИЗ ПОД SUDO).";
echo "ОБАЗАТЕЛЬНО ИЗ KVM-КОНСОЛИ! ИЗ ПОД SSH МОЖЕТ (скорре всего нет) СЛОМАТЬ ВАШУ ВИРТАЛКУ!";
echo "===================================================================";
echo "";
read -p "Хотите чтобы скрипт сделал это (возможно, и сломает)? (Y/N):" -n 1 -r
echo
if [[ ! $REPLY =~ ^[Yy]$ ]]
then
    exit 1
fi

# Установим пакет iptables (если он уже установлен, попытка повторной установки "проскочит")
sudo -S apt-get install iptables

# ПРИСВАИВАЕМ ПЕРМЕННЫЕ:
# $IPT     = "/usr/sbin/iptables"
# $WAN     = текущий внешний сетевой интерфейс виртуалки
# $WAN_IP  = текущий IP V4 назначенный на внешний сетевой интерфейс виртуалки
# $WAN_IP6 = текущий IP V6 назначенный на внешний сетевой интерфейс виртуалки
export IPT="$(sudo -S which iptables)"
export WAN="$(ip add | grep 'BROADCAST' |  awk '{print $2}' | cut -d ':' -f 1)"
export WAN_IP="$(ip ad | grep 'inet ' | awk '(NR == 2)' | awk '{print $2}' | cut -d '/' -f 1)"
export WAN_IP6="$(ip ad | grep 'inet6 ' | awk '(NR == 2)' | awk '{print $2}' | cut -d '/' -f 1)"

echo "-------------------ТЕКУЩИЕ ПРАВИЛА--------------------";
$IPT -L -v -n

# СБРАСЫВАЕМ старые правила:
$IPT -F
$IPT -X

# БАЗОВЫЕ правила (политика по умолчанию) — все обрубаем!
$IPT -P INPUT DROP
$IPT -P OUTPUT DROP
$IPT -P FORWARD DROP

# Разрешаем локальный траффик для loopback (для localhost)
$IPT -A INPUT -i lo -j ACCEPT
$IPT -A OUTPUT -o lo -j ACCEPT

# Разрешаем пинги
# $IPT -A INPUT -p icmp --icmp-type echo-reply -j ACCEPT
# $IPT -A INPUT -p icmp --icmp-type destination-unreachable -j ACCEPT
# $IPT -A INPUT -p icmp --icmp-type time-exceeded -j ACCEPT
# $IPT -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# Разрешаем исходящие соединения с самого сервера
$IPT -A OUTPUT -o $WAN -j ACCEPT

# Состояние ESTABLISHED говорит о том, что это не первый пакет в соединении.
# Пропускать все уже инициированные соединения, а также дочерние от них.
# Так мы разрешим использование текущего порта на который настроен и
# открыт, в настощий момент, SSH а еще доступ к репозиториям для получения
# новых пакетов и их обновлений.
$IPT -A INPUT -p all -m state --state ESTABLISHED,RELATED -j ACCEPT
# Разрешить новые, а так же уже инициированные и их дочерние соединения
$IPT -A OUTPUT -p all -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
# Разрешить форвардинг для уже инициированных и их дочерних соединений
$IPT -A FORWARD -p all -m state --state ESTABLISHED,RELATED -j ACCEPT

# Включаем фрагментацию пакетов. Необходимо из-за разных значений MTU
$IPT -I FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu

# Отбрасывать все пакеты, которые не могут быть идентифицированы
# и поэтому не могут иметь определенного статуса.
$IPT -A INPUT -m state --state INVALID -j DROP
$IPT -A FORWARD -m state --state INVALID -j DROP

# Приводит к связыванию системных ресурсов, так что реальный
# обмен данными становится не возможным, обрубаем
$IPT -A INPUT -p tcp ! --syn -m state --state NEW -j DROP
$IPT -A OUTPUT -p tcp ! --syn -m state --state NEW -j DROP

# Открываем только нужные порты на вход:
# !!! указать свой порт, который вы указали для SSH ранее !!!)
$IPT -A INPUT -i $WAN -p tcp --dport 2002 -j ACCEPT
# порт для your_username сервера (для http)
$IPT -A INPUT -i $WAN -p tcp --dport 80 -j ACCEPT
# порт для your_username сервера (для https)
$IPT -A INPUT -i $WAN -p tcp --dport 443 -j ACCEPT

## Логирование
## Все что не разрешено, но ломится отправим в цепочку undef
# $IPT -N undef_in
# $IPT -N undef_out
# $IPT -N undef_fw
# $IPT -A INPUT -j undef_in
# $IPT -A OUTPUT -j undef_out
# $IPT -A FORWARD -j undef_fw

# Логируем все из undef
# $IPT -A undef_in -j LOG --log-level info --log-prefix "-- IN — DROP "
# $IPT -A undef_in -j DROP
# $IPT -A undef_out -j LOG --log-level info --log-prefix "-- OUT — DROP "
# $IPT -A undef_out -j DROP
# $IPT -A undef_fw -j LOG --log-level info --log-prefix "-- FW — DROP "
# $IPT -A undef_fw -j DROP

# Записываем правила
$IPT-save  > $IPT.rules

# Выводим правила на экран:
echo "----------ПРАВИЛА ДЛЯ TCP-ПОРТОВ УСТАНОВЛЕНЫ----------";

cat $IPT.rules

echo "---------------------И ПРИМЕНЕНЫ----------------------";

$IPT -L -v -n

echo "------------------------------------------------------";
echo "";
echo "Правила сохранены в файле: $IPT.rules";
echo "";
echo "Для восстановления правил запустите скрипт повторно или";
echo "исполните команду:";
echo "";
echo "sudo $IPT-restore < $IPT.rules";

echo "";
echo "";
echo " _._     _,-'\"\"\`-._";
echo "(,-.\`._,'(       |\\\`-/|";
echo "    \`-.-' \\ )-\`( , o o)";
echo "          \`-    \\\`_\`\"'-  Mi-mi-mi... Ok!";
echo "";
```

Сохраняем скрипт `Ctrl+O` и `Enter`, а выходим из редактора `Ctrl+X`.

Далее нужно зайти на виртуалку через KVM-консоль и выполнить:
```shell
sudo bash ~/prepare-iptable.sh
```
После выполнения скрипта все внешние соединения с виртуалкой будут разорваны. После перелогирования через ssh можно
будет увидеть, что новые настройки сетевой фильтрации вступили в силу:
```shell
sudo iptables -L -v -n
```

Увидим примерно следующее:
```text
Chain INPUT (policy DROP 150 packets, 5768 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   28  2204 ACCEPT     all  —  lo     *       0.0.0.0/0            0.0.0.0/0           
   81  7202 ACCEPT     all  —  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
    2    84 DROP       all  —  *      *       0.0.0.0/0            0.0.0.0/0            state INVALID
    8   608 DROP       tcp  —  *      *       0.0.0.0/0            0.0.0.0/0            tcp flags:!0x17/0x02 state NEW
    1    52 ACCEPT     tcp  —  eth0   *       0.0.0.0/0            0.0.0.0/0            tcp dpt:2002
    0     0 ACCEPT     tcp  —  eth0   *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80
    0     0 ACCEPT     tcp  —  eth0   *       0.0.0.0/0            0.0.0.0/0            tcp dpt:443

Chain FORWARD (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 TCPMSS     tcp  —  *      *       0.0.0.0/0            0.0.0.0/0            tcp flags:0x06/0x02 TCPMSS clamp to PMTU
    0     0 ACCEPT     all  —  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
    0     0 DROP       all  —  *      *       0.0.0.0/0            0.0.0.0/0            state INVALID

Chain OUTPUT (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   28  2204 ACCEPT     all  —  *      lo      0.0.0.0/0            0.0.0.0/0           
   69 10113 ACCEPT     all  —  *      eth0    0.0.0.0/0            0.0.0.0/0           
    0     0 ACCEPT     all  —  *      *       0.0.0.0/0            0.0.0.0/0            state NEW,RELATED,ESTABLISHED
    0     0 DROP       tcp  —  *      *       0.0.0.0/0            0.0.0.0/0            tcp flags:!0x17/0x02 state NEW
```

Чтобы сохранить настройки и чтобы они автоматически вступали в силу в 
случае перезагрузки установим пакет `iptables-persistent`:
```shell
sudo -S apt-get install iptables-persistent
```

Будет запрошено сохранить ли и восстанавливать настройки в случае 
перезагрузки для IP-V4 и IP-V6 — оба раза отвечаем `Y`. 

В будущем, если будут меняться правила фильтрации, то сохранить новые настройки можно командой `sudo 
netfilter-persistent save`, а восстановить командой `sudo netfilter-persistent start`. Для детальных
разъяснений [см. по ссылке](https://losst.ru/kak-sohranit-pravila-iptables).

## 5. Установим защиту DoS (брутфорса) по ssh

DoS и DDoS-атака — это агрессивное внешнее воздействие на сервер, проводимое с целью перегрузить его запросами, 
доведения его до отказа или брутфорс-взлома (brute force — грубая сила, например, подбор паролей перебором и т.п.). 

Если атака проводится с одиночного компьютера — ее называют DoS (Denial of Service), если из нескольких, 
распределенных в сети компьютеров — DDoS (Distributed Denial of Service).

Защиту от DoS (включая медленные атаки и нежелатеольное сканирование web-сервера) обеспечивает пакет `fail2ban`.

Установим его (уже должен быть установлен):
````shell
sudo apt-get install fail2ban
````

Для его настройки изменим конфигурационный файл `/etc/fail2ban/jail.local`:
```shell
sudo nano /etc/fail2ban/jail.local
```
И поместим туда следующее (поменяйте `xxx.xxx.xxx.xxx` на свои IP-адрес, с которых будете подключаться):
```editorconfig
# == новый конфиг /etc/fail2ban/jail.local =====================
[DEFAULT]
#email, на который присылать уведомления                             
destemail       = root 
# подключить прафила бана из '/etc/fail2ban/action.d/iptables-multiport.conf'
banaction       = iptables-multiport 
# исключаем из потенциального бана ip машины с которых будем сами подключаться
ignoreip        = xxx.xxx.xxx.xxx xxx.xxx.xxx.xxx xxx.xxx.xxx.xxx 

#### правила для SSH ####                             
[sshd]
enabled = true 
port = ssh,2002 
# filter — подключить правила фильтрации из '/etc/fail2ban/filter.d/sshd.conf'
filter  = sshd
# logpath — какой лог наблюдаем (на тот случай, если он не по умолчанию)
logpath = /var/log/auth.log 
# bantime — время (секунды) на которое баним. Например, на сутки — 60*60*24=86400
bantime = 86400 
# maxretry — число попыток (1) и получаешь бан!                            
maxretry = 1 
# findtime — определяет длительность интервала в секундах, за которое
# событие должно повториться определённое количество раз, после чего санкции
# вступят в силу. Если специально не определить этот параметр, то будет
# установлено значение по умолчанию равное 600 (10 минут). Проблема в том,
# что ботнеты, участвующие в «медленном брутфорсе», умеют обманывать
# стандартное значение. Иначе говоря, при maxretry равным 6,
# атакующий может проверить 5 паролей, затем выждать 10 минут,
# проверить ещё 5 паролей, повторять это снова и снова, и его IP забанен
# не будет. В целом, это не угроза, но всё же лучше жестно банить таких ботов
findtime = 7200

# СТАРЫЙ КОНФИГ ПО УМОЛЧАНИЮ
# [DEFAULT]
# bantime = 600
# findtime = 60
# maxretry = 6
# banaction = iptables-multiport
# [sshd]
# enabled = true
```
Сохраняем конфигурационный файл `Ctrl+O` и `Enter`, а выходим из редактора `Ctrl+X`.

Перезапустим fail2ban:
```shell
sudo service fail2ban restart
```

Проверим статус:
```shell
sudo service fail2ban status
```

Увидим, что все работает: 
```text
● fail2ban.service - Fail2Ban Service
     Loaded: loaded (/lib/systemd/system/fail2ban.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2022-05-17 14:33:22 MSK; 1h 53min ago
       Docs: man:fail2ban(1)
    Process: 7635 ExecStartPre=/bin/mkdir -p /run/fail2ban (code=exited, status=0/SUCCESS)
   Main PID: 7640 (f2b/server)
      Tasks: 5 (limit: 1066)
     Memory: 11.0M
     CGroup: /system.slice/fail2ban.service
             └─7640 /usr/bin/python3 /usr/bin/fail2ban-server -xf start

May 17 14:33:22 vm2203242538 systemd[1]: Starting Fail2Ban Service...
May 17 14:33:22 vm2203242538 systemd[1]: Started Fail2Ban Service.
May 17 14:33:22 vm2203242538 fail2ban-server[7640]: Server ready
```

## 5. Установка и настройка СУБД (MariaDB), создание пользователей

Этот пункт можно пропустить, если в вашем проекте используется `sqlite3`. Для `PostgreSQL` (или других SQL-серверов
нужно будет провести аналогичные действия для соответствующего SQL-сервера).

Следуем [инструкциям](https://www.digitalocean.com/community/tutorials/how-to-install-mariadb-on-ubuntu-20-04),
и у нас получится примерно такой порядок действий:

### Установка

Устанавливаем сервер MariaDB (полный аналог MySQL) и пакет MariaDB-клиента для разработчиков (чтобы после корректно
установился MySQL-коннектор для Python):
```shell
sudo apt install mariadb-server libmysqlclient-dev
```

Отредактируем конфигурационный файл, чтобы сменить кодовые таблицы в будущих базах с `utf8mb4` (со всякими глупыми 
эможи) на православную `utf8` (на самом деле, в последних версиях это уже не срабатывает): 
```shell
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Найдём в конфигурационном файле строки:
```toml
character-set-server = utf8mb4
collation-server = utf8mb4_general_ci
```

и заменим на:
```toml
character-set-server    = utf8
collation-server        = utf8_general_ci
```
Сохраняем конфигурационный файл `Ctrl+O` и `Enter`, а выходим из редактора `Ctrl+X`.

Перезапустим систему управления базы данных:
```shell
sudo service mysql restart
```

Проверим статус:
```shell
sudo service mysql status
```

Если все сделали правильно, должны увидеть что-то типа:
```text
● mariadb.service - MariaDB 10.3.32 database server
     Loaded: loaded (/lib/systemd/system/mariadb.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2022-01-20 14:58:12 MSK; 5s ago
       Docs: man:mysqld(8)
             https://mariadb.com/kb/en/library/systemd/
    Process: 7057 ExecStartPre=/usr/bin/install -m 755 -o mysql -g root -d /var/run/mysqld (code=exited, status=0/SUCCESS)
    Process: 7068 ExecStartPre=/bin/sh -c systemctl unset-environment _WSREP_START_POSITION (code=exited, status=0/SUCCESS)
    Process: 7070 ExecStartPre=/bin/sh -c [ ! -e /usr/bin/galera_recovery ] && VAR= ||   VAR=`cd /usr/bin/..; /usr/bin/galera_recovery`; [ $? >
    Process: 7149 ExecStartPost=/bin/sh -c systemctl unset-environment _WSREP_START_POSITION (code=exited, status=0/SUCCESS)
    Process: 7151 ExecStartPost=/etc/mysql/debian-start (code=exited, status=0/SUCCESS)
   Main PID: 7118 (mysqld)
     Status: "Taking your SQL requests now..."
      Tasks: 31 (limit: 1037)
     Memory: 83.0M
     CGroup: /system.slice/mariadb.service
             └─7118 /usr/sbin/mysqld
```

### Создание пользователей

Для этого запустим mysql с супер-правами `root`:
```shell
sudo mysql
```

Создаём пользователя `your_username`, для полного доступа ко всему (аналог супер-пользователя) и зададим ему
пароль и дадим полные привилегии:
```mysql
CREATE USER 'your_username'@'localhost' IDENTIFIED BY 'очень-секретный-пароль';
GRANT ALL PRIVILEGES ON *.* TO 'your_username'@'localhost';
```

Создаем базу данных нашего проекта `django_prj` для нашего сайта:
```mysql
CREATE DATABASE django_prj DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
```

И создадим пользователя `project_username`, пароль для него и дадим ему права работать только с базой сайта:
```mysql
CREATE USER 'project_username'@'localhost' IDENTIFIED BY 'другой-очень-секретный-пароль';
GRANT ALL PRIVILEGES ON django_prj.* TO 'project_username'@'localhost';
```

Проверим, что установлена правильная часовая зона времени:
```mysql
SHOW VARIABLES LIKE '%time_zone%';
```

Типа, она должна быть `MSK`:
```
+------------------+--------+
| Variable_name    | Value  |
+------------------+--------+
| system_time_zone | MSK    |
| time_zone        | SYSTEM |
+------------------+--------+
2 rows in set (0.001 sec)
```

Выходим из MariaDB:
```mysql
quit;
```

переносим базу

## 6. ПОДГОТОВКА WEB-СЕРВЕРА NGINX

Устанавливаем nginx
```shell
sudo apt-get install nginx
```

На всякий случай (как выяснилось позже), лучше поставить nginx plus. В нем есть динамические модули. И один их 
таких динамических модулей — **geoip** — может понадобиться для ограничения посещения сайта только пользователями 
России. [Смотрим инструкцию по установке nginx plus](https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-plus/).  

Убедиться, что он запущен и работает корректно можно командой:
```shell
sudo service nginx status
```

Результат должен быть примерно таким:
```text
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2022-01-20 18:45:16 MSK; 56s ago
       Docs: man:nginx(8)
   Main PID: 2200 (nginx)
      Tasks: 2 (limit: 1037)
     Memory: 4.6M
     CGroup: /system.slice/nginx.service
             ├─2200 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
             └─2201 nginx: worker process
```

Зайдем по ip нашего сервера c внешнего браузера, или `curl` через localhost:
```shell
curl -I http://localhost/
```

Получим ответ:
```
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Tue, 03 May 2022 18:16:46 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 03 May 2022 12:49:44 GMT
Connection: keep-alive
Vary: Accept-Encoding
ETag: "627124e8-264"
Accept-Ranges: bytes
```

### Настройка GZIP

По умолчанию модуль GZIP установлен в NGINX. Он осуществляет сжатие данных 
«на лету», они отправляются nginx, и, после получения, распаковываются 
браузерами (само-собой теми, которые поддерживают такую возможность). При 
этом между веб-сервером и браузером передается меньшее данных, что хорошо
для медленных подключения (но не для сервера, так как он тратит ресурсы на
сжатие). Сжатие использует ресурсы сервера, поэтому лучше всего сжимать только
те файлы, которые хорошо «сжимаются» (а еще лучше, если ещё и могут кэшироваться на 
стороне браузера). Текстовые файлы (html, javascript, css и т.п.) сжимаются хорошо,
JPEG или PNG уже сжаты по своей природе и никакого результата при сжатии GZIP можно не 
ожидать.

Настроем модуль GZIP в NGINX. Для этого открываем на редактирование 
конфигурационный файл nginx `/etc/nginx/nginx.conf`:
```shell
sudo nano /etc/nginx/nginx.conf
```

Находим в нем кусочек касающийся GZIP:
```
        ##
        # Gzip Settings
        ##
```

и вставляем туда следующее (там будет закомментированный блок, можно использовать его):
```nginx configuration
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_disable "msie6";
    gzip_vary on;
    gzip_min_length 512;
    gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript application/javascript application/vnd.ms-fontobject application/x-font-ttf font/opentype image/svg+xml image/x-icon;
```

Если интересно, расшифруем что тут происходит:
* `gzip_proxied any` — сжимать данные ответов для proxy-серверов.
* `gzip_comp_level 6` — устанавливаем, степень сжатия файлов. Чем выше 
число, тем выше уровень сжатия и использование ресурсов. уровень сжатия, 1 
- минимальное, 9 - максимальное.
* `gzip_buffers 16 8k` — задаёт число и размер буферов, в которые будет 
сжиматься ответ. По умолчанию размер одного буфера равен размеру страницы. 
В зависимости от платформы это или 4K, или 8K.
* `gzip_http_version 1.1` — директива используется для ограничения сжатия 
gzip для браузеров, поддерживающих протокол HTTP/1.1. Если браузер не 
поддерживает его, вероятно, что он не поддерживает и gzip.
* `gzip_disable "msie6"` — исключаем IE6 из браузеров, которые будут 
получать сжатые файлы (этот древний браузер не поддерживает GZIP).
* `gzip_vary on` — включает добавление в ответ заголовка __"Vary: 
Accept-Encoding"__  (для IE4-6 это приведёт к не кешированию данных из-за 
бага).
* `gzip_min_length 512` — сообщаем NGINX не сжимать файлы размером менее 
512 байт.
* `gzip_types` — отображает все типы MIME, которые будут сжаты. В этом 
случае список включает страницы HTML, таблицы стилей CSS, файлы Javascript 
и JSON, файлы XML, иконки (BMP-изображения очень даже сжимаются), 
изображения SVG и веб-шрифты.

Сохраняем конфигурационный файл `Ctrl+O` и `Enter`, а выходим из редактора 
`Ctrl+X` и перезагружаем nginx:
```shell
sudo service nginx restart
```

#### Проверка новой конфигурации

Выполним тестовый запрос:
```shell
curl -H "Accept-Encoding: gzip" -I http://localhost/
```

Мы получим, например, вот такой ответ:
```
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Tue, 03 May 2022 18:16:53 GMT
Content-Type: text/html
Last-Modified: Tue, 03 May 2022 12:49:44 GMT
Connection: keep-alive
Vary: Accept-Encoding
ETag: W/"627124e8-264"
Content-Encoding: gzip
```

Как видим, сжатие включилось.

## 7. Развёртывание окружения проекта

### Настройка виртуального окружения проекта

Установим python с набором для разработчиков (для сборки коннектора к СУБД MariaDB), пакетный менеджер и утилиту для 
создания виртуального окружения python: 
```shell
sudo apt-get install python3-pip python3-virtualenv python3-dev
```

Проверим, что установлена нужная нам версия Python:
```
python3 -V
```

Теперь создадим папку для нашего Django-проекта (например `~/django_project`) и развернём в ней 
виртуальное окружение, указав, что в нем нужно использовать необходимую для проекта версию Python
(например 3.8 из `/usr/bin/python3.8`). Будет создан каталог, с файлами виртуального 
окружение (версия Python, установщик пакетов pip, wheel, setuptools а, в 
будущем, и все пакеты, батарейки, свистелки и хрюкалки нашего проекта).
```shell
cd $HOME
mkdir -p django_project
```

Чтобы для нашего проекта "заморозить" версию Python и все необходимые пакеты, и избежать возможного
конфликта при обновлении системного Python или с пакетами установленными в других проектах создадим
виртуальное окружение в папке нашего сайта (`$HOME/django_project`):
```shell
virtualenv -p /usr/bin/python3.8 $HOME/django_project/env
```

Активируем созданное виртуальное окружение:
```shell
source $HOME/django_project/env/bin/activate
```

Проверить, что теперь мы работаем в виртуальном окружении можно дав команды:
```shell
python -V
pip -V
```
Увидим 
> pip 20.0.2 from /home/your_username/django_project/env/lib/python3.8/site-packages/pip (python 3.8)

Можно обновить менеджер пакетов `pip` по последней версии (сперва получим его с помощью `curl` в папку tmp,
а после запустим установщик через Python)
```shell
curl https://bootstrap.pypa.io/get-pip.py > ~/tmp/get-pip.py
python3 ~/tmp/get-pip.py
```
Попробуем снова
```shell
pip -V
```
Увидим 
> pip 22.0.4 from /home/your_username/django_project/env/lib/python3.8/site-packages/pip (python 3.8)

ФУУ!!

## 8. Установка пакетов необходимых проекту

Точный состав пакетов, обычно, находится в файле `requarement.txt` и чтобы установить все пакеты из этого файла
перейти в папку проекта (в место расположения `requarement.txt`) и выполните команду:
```shell
pip install -r requirements.txt
```
  
Также можно установить пакеты по одному. Как приме,р приведу список пакетов
одного своего древнего проекта:

| Пакет                             | Версия | Назначение                                                                                                                     | Зависимости                                                                                                                                                                   |
|-----------------------------------|--------|--------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| django                            | 3.2.11 | Фреймворк Django                                                                                                               | притащит с собой пакеты: __asgiref-3.4.1__, __pytz-2021.3__ и __sqlparse-0.4.2__                                                                                              |                                                                                            
| mysqlclient                       | 2.1.0  | Коннектор MySQL                                                                                                                | нет                                                                                                                                                                           |                                                                                                                                                                         
| django-filer                      | 2.0.2  | Система управления медиа-файлами с фишками подготовки ресайз-картинок, превьюшек и прочими плюшками                            | притащит с собой пакеты: __Unidecode-1.1.2__, __django-js-asset-1.2.2__, __django-mptt-0.12.0__, __django-polymorphic-3.0.0__, __easy-thumbnails-2.7.1__ и __pillow-8.2.0__   | 
| django-ckeditor                   | 6.2.0  | Wysiwyg-редактор (ckeditor) для админки                                                                                        | нет                                                                                                                                                                           |                                                                                                                          
| django-ckeditor-filebrowser-filer | 0.3.0  | Плугин для дружбы Wysiwyg-редквтора (ckeditor) и django-filer                                                                  | нет, все зависимости уже притащил django-filer                                                                                                                                |                                                                                                                         
| pytils-safe                       | 0.3.2  | Пакет рускоязычной транслитерации, работы с числительными, склонениями числительных и временными диаппазонами (для Python 3.x) | нет                                                                                                                                                                           |                                                                                                                         
| urllib3                           | 1.26.8 | пакет для работы с web-запросами (проекту этот пакет нужен для работы с API внешний HTML-типографов)                           | нет                                                                                                                                                                           | 

Все эти пакеты устанавливаются в виртуальное окружение с помощью пакетного 
менеджера `pip` в последовательности:

```shell
pip install django=3.2.11
pip install mysqlclient=2.1.0
pip install django-filer=2.0.2
pip install django-ckeditor=6.2.0
pip install django-ckeditor-filebrowser-filer=0.3.0
pip install pytils-safe=0.3.2
pip install urllib3=1.26.8
```

### Разворачиваем и тестируем проект

Создаём папки для хранения сокета и логов:
```shell
mkdir -p $HOME/django_project/logs
mkdir -p $HOME/django_project/socket
```

Переносим проект... например архивируем проект. __Само-собой это надо делать там, где проект находится
в разработке и тесте, не на виртуалке__. Например, 
так:
```shell
zip -9 -r django_project.zip public/ django_project/ config/
```

Копируем архив по ssh на нашу виртуалку по адресу `XXX.XXX.XXX.XXX`:
```shell
scp -P 2002 django_project.zip your_username@XXX.XXX.XXX.XXX:~/django_project
```

Возвращаемся на нашу виртуалку хостера и разархивируем:
```shell
cd ~/django_project
unzip django_project.zip
```

__Внимание__, на разных серверах расположение папок проекта может отличаться. При архивации илм после распаковки, 
возможно, нам потребуется переместить некоторые папки. Мы должны получить следующую структуру проекта:
```text
+---public
|   +---media
|   L---static
+---django_project
|   +---django_project
|   +---templates
|   L---web
+---logs
+---config
+---env
L---socket
```

Папки __logs__, __config__, __env__ и __socket__ на данном этапе могут отсутствовать. Мы создадим их позже:

Теперь можно произвести перенос статических файлов админки и батареек в папку для web-статики:
```shell
source ~/django_project/env/bin/activate
cd ~/django_project/django_project
python manage.py collectstatic
```

Теперь произведем подготовку нашей базы под проект. Для этого произведем миграции:
```shell
python manage.py makemigrations
python manage.py migrate
```

Миграция создаст все необходимые таблицы в базе данных и сделает файлы миграций `django_prj.django_migration`.

Теперь можно произвести перенос базы с dev-проекта (или с другого рабочего сервера) в базу нашей виртуалки. 
Следует учесть, что переносить таблицу `django_prj.django_migration` не следует (мы произвели чистую миграцию, и 
если мы перезапишем эту таблицу, то с высокой вероятностью сломаем наш проект).

Настало время проверить, что наше web-приложение Django работает.

Временно откроем порт _8080_ через `iptables`:
```shell
sudo iptables -A INPUT -i eth0 -p tcp --dport 8080 -j ACCEPT
```

Запустим приложение с dev-режиме на нашем IP и этом порту:
```shell
python manage.py runserver 109.70.24.15:8080
```

Если мы обратимся из браузера по адресу __http://109.70.24.15:8080__, то увидим как на нашей виртуалке "бежит лог". 
В браузере будет отображаться что-то похожее на наш проект (с тем отличием, что не будет работать статика... т.е. 
мы не увидим картинок, не будут подгружены стили, JavaScript и прочая статика). 

Завершаем работу web-сервера разработчика, нажав `Control+C`.

## 8. Конфигурируем nginx под наш проект
-----------

Создаем и правим конфиг `~/django_project/config/django_project.conf`:
```shell
nano ~/django_project/config/django_project.conf
```

Помещаем в него следующее (измените `your_username` на своё имя пользователя и прочее на свои данные):
```nginx configuration
#  САЙТ DJANGO_PROJECT
#  ==  Конфикурационный файл nginx django_project.conf

# Описываем апстрим-потоки которые должен подключить Nginx
# Для каждого сайта надо настроить свйо поток, со своим уникальным именем.
# Если будете настраивать несколько python (django) сайтов - измените название upstream

upstream django_project-uwsgi { 
    # расположение файла Unix-сокет для взаимодействие с uwsgi
    server unix:///home/your_username/django_project/socket/django_project.sock;
    # также можно использовать веб-сокет (порт) для взаимодействие с uwsgi. Но это медленнее
    # server 127.0.0.1:8001; # для взаимодействия с uwsgi через веб-порт
}

# конфигурируем сервер
server {
    listen          80;
    server_name     YOUR_SERVER_NAME;   # имя сервера
    charset         utf-8;              # кодировка по умолчанию
    access_log  /home/your_username/django_project/logs/django_project-access.log;    # логи с доступом
    error_log   /home/your_username/django_project/logs/django_project-error.log;     # логи с ошибками
    client_max_body_size 100M; # максимальный объем файла для загрузки на сайт (max upload size)
    error_page  404  /404.html;
    error_page  500  /500.html;

    location /media	{ alias	/home/your_username/django_project/public/media;  }	# Расположение media-файлов Django
    location /static	{ alias	/home/your_username/django_project/public/static; }	# Расположение static-файлов Django

    location /favicon.ico	{ root	/home/your_username/django_project/public; }	# Расположение favicon.ico
    location /favicon.gif	{ root	/home/your_username/django_project/public; }	# Расположение favicon
    location /favicon.png	{ root	/home/your_username/django_project/public; }	# Расположение favicon
    location /favicon.svg	{ root	/home/your_username/django_project/public; }	# Расположение favicon
    # location /robots.txt	{ root	/home/your_username/django_project/public; }	# Расположение robots.txt
    # location /author.txt	{ root	/home/your_username/django_project/public; }	# Расположение author.txt
    
    # Определяем расположение страниц ошибок 404 и 500
    location = /404.html	{
        root	/home/your_username/django_project/django_project/templates/404.html;
        internal;
    }
    location = /500.html	{
        root	/home/your_username/django_project/django_project/templates/500.html;
        internal;
    }
    
    # Определяем расположение файлов robots.txt, author.txt и прочих статичных файлов (xml, html, htm и txt)
    location ~ \.(xml|html|htm|txt)$	{
        root  /home/your_username/django_project/public;        # Расположение статичных *.xml, *.html и *.txt
      }

    location / {
        uwsgi_pass           django_project-uwsgi;         # upstream обрабатывающий обращений
        include              uwsgi_params;               # конфигурационный файл uwsgi;
        uwsgi_read_timeout   1800;     # некоторые запросы на стабеньких машинках очень долго обрабатываются.
        uwsgi_send_timeout   200;      # на всякий случай время записи в сокет 
        }
}
```

Делаем симлинк (символическую ссылку) этого конфигурационного файла в папку конфигов сайтов nginx:
```shell
sudo ln -s ~/django_project/config/django_project.conf /etc/nginx/sites-enabled/
```

Протестируем конфигурацию:
```shell
sudo nginx -t
```

Если все ок, можем "мягко" перезапустить nginx:
```shell
sudo nginx -s reload
```

Или "по-жёсткому":
```shell
sudo service nginx restart
```

Теперь если мы из браузера обратимся по адресу какого-нибудь статического файла (_http://YOUR_SERVER_NAME/robots.txt_, _http://YOUR_SERVER_NAME/favicon.ico_, 
_http://YOUR_SERVER_NAME/favicon.png_ или _http://YOUR_SERVER_NAME/favicon.svg_) то увидим соответсвующий контент.

## 9. НАСТРОЙКА uWSGI

uWSGI — это "прокладка" между веб-сервером и приложением Django. Он принимает через сокет http-запросы от
веб-сервера nginx, обрабатывает их и передает приложению Django. Приложение Django обрабатывает запрос и
отправляет ответ обратно через uWSGI в nginx, который в свою очередь отправляет его клиенту.

И uWSGI, и Python-плагин uWSGI уже должен быть установлен. Но если он не установлены, или для проверки, инсталляция 
производится из так:
```shell
sudo apt-get install uwsgi uwsgi-plugins-all
```

Проверим, что uWSGI работает:
```shell
sudo service uwsgi status
```

Увидим примерно следующее:
```text
● uwsgi.service - LSB: Start/stop uWSGI server instance(s)
     Loaded: loaded (/etc/init.d/uwsgi; generated)
     Active: active (exited) since Thu 2022-01-20 20:36:20 MSK; 2min 57s ago
       Docs: man:systemd-sysv-generator(8)
      Tasks: 0 (limit: 1037)
     Memory: 0B
     CGroup: /system.slice/uwsgi.service
```

Создаём и правим ini-конфиг `~/django_project/config/django_project.ini` для uWSGI:
```shell
nano ~/django_project/config/django_project.conf
```

Помещаем туда следующее:
```editorconfig
# ===  Конфикурационный файл uwsgi django_project.ini
[uwsgi]

# НАСТРОЙКИ ДЛЯ DJANGO
# Корневая папка проекта (полный путь)
chdir           = /home/your_username/django_project/django_project
# Django wsgi файл django_project/wsgi.py записываем так:
module          = django_project.wsgi
# полный путь к виртуальному окружению
home            = /home/your_username/django_project/env
# полный путь к файлу сокета
socket          = /home/your_username/django_project/socket/django_project.sock
# Исходящие сообщения в лог
daemonize       = /home/your_username/django_project/logs/django_project_uwsgi.log

# ЗАГАДОЧНЫЕ НАСТРОЙКИ, ПО ИДЕЕ ОНИ НУЖНЫ, НО И БЕЗ НИХ ВСЁ РАБОТАЕТ
# расположение wsgi.py
wsgi-file       = /home/your_username/django_project/django_project/django_project/wsgi.py
# расположение виртуального окружения (как оно работает если этот параметр не указан, не ясно)
virtualenv      = /home/your_username/django_project/env
# имя файла при изменении которого происходит авторестарт приложения
# (когда этого параметра нет, то гичего не авторестартится, но с ним все рестартится.
# Cтоит изменить любой Python-исходник проекта, как изменения сразу вступают в силу.
touch-reload    = /home/your_username/django_project/logs/touch-reload
# авторестарт приложения при изменении файлов проекта через 5-секунд
py-autoreload	= 5

#  НАСТРОЙКИ ОБЩИЕ
# быть master-процессом
master          = true
# максимальное количество процессов (воркеров) uWSGI
processes       = 1
# если uWSGI установлен как сервис через apt-get то нужно установить еще плугин:
# sudo apt-get install uwsgi-plugin-python
# и добавить в этот конфиг: `plugin = python` или `plugin = python3`
plugin          = python3
# права доступа к файлу сокета. По умолчанию должно хватать 664. Но чтоб наверняка, ставим 666
chmod-socket    = 666
# очищать окружение от служебных файлов uwsgi по завершению
vacuum          = true
# количество секунд после которых подвисший процес будет перезапущен
# Так как некоторе скрипты требуют изрядно времени (особенно полная переиндексация) то ставим значение побольще
harakiri      = 2600
# В общем случае, при некотых значениях harakiri логах uWSGI может вываливаться предупреждение:
# WARNING: you have enabled harakiri without post buffering. Slow upload could be rejected on post-unbuffered webservers
# можно оставить harakiri закоментированным, но нам нужно 900 и на него не ругается. Ругается на 30.

# разрешаем многопоточность
enable-threads  = true
vacuum          = true
thunder-lock    = true
max-requests    = 500

# пользователь и группа пользователей от имени которых запускать uWSGI
# указываем www-data: к этой группе относится nginz, и ранее мы включили в эту группу нашего [user]
# uid             = nginx
# gid             = nginx
# uid             = www-data
# gid             = www-data
uid             = your_username
gid             = your_username

print           = ---------------- Запущен uWSGI для django_project ----------------
```

**ВАЖНО**: *Если вы скопируете этот файл из системы под Microsoft Windows, то может ничего не заработать. В Windows
строчки разделяются символом `CRLF` (перевод строки и возврат каретки), а в Linux — только `LF` (перевод строки).
Конфиг uWSGI чувствителен к этому. Поэтому, если вы скопировали этот файл из Windows, то перед запуском uWSGI
его надо преобразовать в формат Linux. Для этого воспользуйтесь утилитой `dos2unix`. Или вставьте содержимое
файла в редакторе `nano` и пересохраните его.*

Делаем симлинк нашего конфигурационного файла в папку конфигов uwsgi:
```shell
sudo ln -s ~/django_project/config/django_project.ini /etc/uwsgi/apps-enabled/
```

Не забываем перезапустить uwsgi:
```shell
sudo service uwsgi restart
```

Еще раз проверим, что uWSGI работает:
```shell
sudo service uwsgi status
```

Все. Сайт развёрнут. Далее можно проводить улучшайзинг и усиливать защиту:

## 10. Нагрузочное тестирование (нужен dockers)

Нагрузочное тестирование будем проводить при помощи Яндекс.Танк. Он будет "обстреливать" наш проект запросами. 

Для запускать Яндекс.Танк проще всего использовать Dockers. Т.о. Dockers должен быть установлен на машину, с которой 
будем "обстреливать" сайт (и это не виртуалка хостинга... "обстреливать" самих себя бессмысленно).

Создаем каталог  `yandex.tank` (в моем случае `M:/VM/yandex.tank`) и в нем помещаем файл `load.yaml` следующего 
содержания (не забываем корректировать `YOUR_SERVER_NAME` и список `uris` под свой проект... их и будет
"обстреливать" Яндекс.Танк): 
```yaml
overload:
  enabled: true
  package: yandextank.plugins.DataUploader
  token_file: "token.txt"
phantom:
  address: YOUR_SERVER_NAME:80
  header_http: "1.1"
  headers:
    - "[Host: YOUR_SERVER_NAME]"
    - "[Connection: close]"
  uris:
    - /
    - /url1/target1
    - /url2/target2
    - /url3/target3
    - /url4/target4
    - /url5/target5
  load_profile:
    load_type: rps
    # schedule: line(5, 5000, 3m)
    # schedule: const(1,30s) line(1,1000,2m) const(10000,5m)
    schedule: line(10,5000,2m) const(100,1m)
  ssl: true
autostop:
  autostop:
    - http(5xx,10%,5s)
console:
  enabled: true
telegraf:
  enabled: false
```

Так же создаем пустой файл `token.txt` в той же папке. __ВНИМАНИЕ__ файл должен быть в кодировке ANSI (хоть он и пустой,
но это важно).

Теперь можем запустить Яндекс.Танк:
```shell
docker run --rm -v M:/VM/yandex.tank:/var/loadtest -it direvius/yandex-tank
```

После запуска Яндекс.Танк будет "обстреливать" наш сайт. После окончания теста, Яндекс.Танк выдаст отчет о
нагрузке на сайт. В соответствии с этим отчетом можно спланировать какой сервер будет достаточен для работы сайта.

## 11. SSL

Хорошая [инструкция по установке SSL от Let's Encript](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-22-04).

Если же у нас самоподписанный сертификат или покупной (например *AlphaSSL-Wildcard*), то создадим папку
для хранения SSL-сертификтов:
```shell
mkdir -p ~/django_project/ssl
```

Скопируем в нее файлы сертификатов:
* `AlphaSSL-Wildcard-certificate--XXXXX.crt` — сертификат.
* `AlphaSSL-Wildcard-certificate-signing-request--XXXXX.csr` — зашифрованный запрос на выпуск сертификата  
  (Certificate Signing Request).
* `AlphaSSL-Wildcard-private-key.key` — приватный ключ. 

Открываем на редактирование созданный нами ранее конфигурационный файл nginx:
```shell
nano ~/django_project/config/django_project.conf
```

И добавляем в него блоки редиректа: 
```nginx configuration
# конфигурируем 302-редирект с YOUR_SERVER_NAME
server {
    listen 80;                                         # слушает 80 порт
    listen 443 ssl;                                    # слушает 443 порт (ssl)
    server_name YOUR_SERVER_NAME;                      # доменное имя сайта
    return 302 https://YOUR_SERVER_NAME$request_uri;   # и редиректим всё
}

# конфигурируем 301-редирект с 109.70.24.15:80
server {
    listen 80;                                     # слушает 80 порт
    server_name YOUR_SERVER_NAME;                      # доменное имя сайта
    return 301 https://$host$request_uri;   # и редиректим всё
}
```

А в конфигурации нашего сайта надо вносим следующие изменения: 
```nginx configuration
server {
    listen	443 ssl http2;
    server_name YOUR_SERVER_NAME;		# доменное имя сайта
    ssl_certificate         /home/your_username/django_project/ssl/AlphaSSL-Wildcard-certificate--XXXXX.crt;
    ssl_trusted_certificate /home/your_username/django_project/ssl/AlphaSSL-Wildcard-certificate--XXXXX.crt;
    ssl_certificate_key     /home/your_username/django_project/ssl/AlphaSSL-Wildcard-private-key.key;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
    ssl_prefer_server_ciphers on;
    # ssl_stapling on;
    resolver 77.88.8.8;
    ssl_session_cache builtin:1000 shared:SSL:25m;
    keepalive_requests 200;
    # keepalive_time 1h;
    keepalive_timeout 75s;
    # ...
    # ...
    # ...
    }
```

Для более оптимальной нагрузки на uwsgi в блок `upstream` имеет смысл добавить:
```nginx configuration
    keepalive_requests 200;
```

А в блок `location … {uwsgi_pass …}` вставить (но это не точно):
```nginx configuration
    fastcgi_keep_conn on;
    proxy_set_header Host $host;
```

Еще про настройку ssl можно почитать [здесь](https://losst.ru/nastrojka-ssl-v-nginx-s-lets-encrypt) или
[здесь](https://masterhost.ru/support/faq/ssl/install). Инструкция как настроить бесплатный ssl-сертификат
приведена [здесь](https://help.reg.ru/hc/ru/articles/4408046821009--Как-настроить-SSL-сертификат-на-Nginx).

## 12. Блокируем и баним зловредов (бот-сканеры)

### Ограничения числа запросов в nginx (не обязательно, но вдруг)

Настроим nginx на ограничения числа запросов c одного IP-адреса с помощью модуля Limit Req Module.

Открываем файл `/etc/nginx/nginx.conf` (конфиг для всех сайтов на сервере):
```shell
sudo nano /etc/nginx/nginx.conf
```

И в блок `http {…}` добавляем строку:
```nginx configuration
http {
    ...
    limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;
    ...
}
```

где:
* `$binary_remote_addr` — переменная Nginx содержащая IP с которого пришёл запрос;
* `zone=one` — имя зоны. Если настройка делается в нескольких vhost файлах, имена должны быть разные;
* `10m` — размер зоны. 1m может содержать 16000 состояний, т.е. 16000 уникальных IP-адресов;
* `rate=1r/s` — разрешено 1 запрос в секунду. Так же можно указать количество запросов в минуту (30r/m — 30 запросов в минуту)

Затем открываем на редактирование конфигурационный файл nginx для нашего сайта:
```shell
nano ~/django_project/config/django_project.conf
```

И в блок `server {…}`, добавим строку:
```nginx configuration
        limit_req zone=one burst=20 nodelay;
```

Где:
* `one` — имя зоны настроеной в /etc/nginx/nginx.conf (для всех сайтов сервера) в блоке `http {…}`;
* `burst` — максимальный всплеск активности, можно регулировать до какого значения запросов в секунду может быть всплеск запросов;
* `nodelay` — незамедлительно, при достижении лимита подключений, выдавать код 503 (Service Unavailable) для этого IP.

Строку `limit_req zone=one burst=5 nodelay;` можно добавить как непосредственно в блок `server {…}`, и тогда будет ограничиваться число запросов ко всем файлам сайта, так и в расположенные в нем блоки `location … {…}`. Целесообразнее ограничить число запросов к `uwsgi_pass`, т.к. он отвечает только за странички сайта (без статики... статика может быть запрощена клиентом довольна, т.к. одна страничка может содержать внутри себя много встроенных картинок, стилей и скриптов).

Таким образом наш блок `location … {…}` будет выглядеть так: 
```nginx configuration
    location / {
        uwsgi_pass           django_project-uwsgi;         # upstream обрабатывающий обращений
        include              uwsgi_params;               # конфигурационный файл uwsgi;
        proxy_set_header Host $host;
        limit_req zone=one burst=5 nodelay;
        fastcgi_keep_conn    on;
        uwsgi_read_timeout   1800;
        uwsgi_send_timeout   200; 
        }
```

### Баним ботов и подозрительную активность

Настроим fail2ban для выявления ботов, которые ищут скрипты, дампы баз, логи, ключи и тому подобное, анализируя
error-лог. Создаем конфиг-файл фильтра `/etc/fail2ban/filter.d/nginx-noscript.conf`:
```shell
sudo nano /etc/fail2ban/filter.d/nginx-noscript.conf
```

Со следующим содержанием (добавьте в `failregex` все расширения файлов, которые не должны запрашиваться):
```editorconfig
[Definition]
failregex = .*client: <HOST>.*GET.*(\.php|\.asp|\.aspx|\.exe|\.pl|\.cgi|\.scgi|\.log|\.sql|\.jsp|\.csv|\.sh|\.key|\.py|\.pyc|\.asmx|\.asax)
# failregex = ^<HOST> -.*GET.*(\.php|\.asp|\.aspx|\.exe|\.pl|\.cgi|\.scgi|\.log|\.sql|\.jsp|\.csv|\.sh|\.key|\.py|\.pyc|\.asmx|\.asax)
ignoreregex =
```

Проверим, что фильтр написан верно, проверив его на нашем error-логе:
```shell
fail2ban-regex ~/django_project/logs/django_project-error.log /etc/fail2ban/filter.d/nginx-noscript.conf
```

Создаем еще одни фильтр для выявления подозрительной активности (ищут адинки популярных cms, log-и, ключи и тому
подобное) анализируя access лог. Создадим правило `/etc/fail2ban/filter.d/nginx-manual.conf`:
```shell
sudo nano /etc/fail2ban/filter.d/nginx-manual.conf
```

Со следующим содержанием (добавьте свои правила если необходимо, это обычные регулярные выражения):
```editorconfig
[Definition]
failregex = ^<HOST> -.*GET.*wp-content/
            ^<HOST> -.*GET.*wp-admin/.*
            ^<HOST> -.*GET.*administrator/
            ^<HOST> -.*GET.*user/register/
            ^<HOST> -.*GET.*bitrix/admin/
            ^<HOST> -.*GET.*minify/
            ^<HOST> -.*GET.*netcat/
            ^<HOST> -.*GET.*koobooCMS/
            ^<HOST> -.*GET.*apanel/
            ^<HOST> -.*GET.*netcat/
            ^<HOST> -.*GET*/\.git/.*
            ^<HOST> -.*GET*/\.env/.*
            ^<HOST> -.*GET*/cms/.*
            ^<HOST> -.*GET*/\.well-known/.*
            ^<HOST> -.*GET*/\.ssh/.*
            ^<HOST> -.*GET*/\.bash_history/.*
            ^<HOST> -.*GET.*(\.sql|\.php|\.mdb|\.db|\.yml|\.cgi|\.scgi|\.log|\.sql|\.jsp|\.csv|\.sh|\.key|\.py|\.pyc|\.asmx|\.asax))
ignoreregex =
```

Снова проверим, что фильтр написан верно:
```shell
fail2ban-regex ~/django_project/logs/django_project-access.log /etc/fail2ban/filter.d/nginx-manual.conf
```

Далее, редактируем файл `/etc/fail2ban/jail.local` для настройки параметров подключения и работы фильтра:
```shell
sudo nano /etc/fail2ban/jail.local
```

Дописываем строки:
```editorconfig
# ФИЛЬТР nginx-noscript 
[nginx-noscript]
enabled  = true
filter   = nginx-noscript
port     = http,https
action = iptables-multiport[name=NoAuthFailures, port="http,https"]
logpath  = /var/log/nginx/*error*.log
           /home/your_username/django_project/logs/*error*.log
bantime  = 86400
maxretry = 6
findtime = 7200

# ФИЛЬТР manual 
[nginx-manual]
enabled  = true
filter   = nginx-manual
port     = http,https
action = iptables-multiport[name=NoAuthFailures, port="http,https"]
logpath  = /var/log/nginx/*access*.log
           /home/your_username/django_project/logs/*access*.log
bantime  = 86400
maxretry = 6
findtime = 7200
```

Перезапустим fail2ban:
```shell
sudo service fail2ban restart
```

Чтобы посмотреть информацию по заблокированным ip по нашему фильтру используем команду:
```shell
sudo fail2ban-client status nginx-noscript
sudo fail2ban-client status nginx-manual
```

Чтобы разбанить IP (например 8.8.8.8) можно воспользоваться командой:
```shell
sudo fail2ban-client set nginx-noscript unbanip 8.8.8.8
```

## 13. Ротация логов

Обычно процесс ротации логов в системе уже установлен и работает. Но так как мы настроили логирование вручную, в каталое
`~/django_project/logs/`, то нам надо настроить ротацию логов для этой папки самостоятельно (зато может задать свои
правила ротации).

На всякий случай установим пакет `logrotate` (если он еще не установлен):
```shell
sudo apt-get install logrotate
```

Для настройки ротации логов нашего сайта нужно отредактировать конфигурационный файл ротации для nginx `/etc/logrotate.d/nginx`
```shell
sudo nano /etc/logrotate.d/nginx
```

И добавить в него (в конце) следующее:
```editorconfig
/home/your_username/django_project/logs/*.log {
        daily
        missingok
        rotate 14
        compress
        delaycompress
        notifempty
        create 0644 www-data adm
        sharedscripts
        prerotate
                if [ -d /etc/logrotate.d/httpd-prerotate ]; then \
                        run-parts /etc/logrotate.d/httpd-prerotate; \
                fi \
        endscript
        postrotate
                invoke-rc.d nginx rotate >/dev/null 2>&1
        endscript
}
```

После чего процесс ротации надо перезапустить:
```shell
sudo service logrotate restart
```

Убедимся, что ротация работает:
```shell
sudo service logrotate status
```

Увидим:
```text
● logrotate.service - Rotate log files
     Loaded: loaded (/lib/systemd/system/logrotate.service; static; vendor preset: enabled)
     Active: inactive (dead) since Wed 2022-05-18 16:47:06 MSK; 4s ago
TriggeredBy: ● logrotate.timer
       Docs: man:logrotate(8)
             man:logrotate.conf(5)
    Process: 20860 ExecStart=/usr/sbin/logrotate /etc/logrotate.conf (code=exited, status=0/SUCCESS)
   Main PID: 20860 (code=exited, status=0/SUCCESS)
```

## 14. Геограничения и болкировка отдельных юзер-агентов

Если у вас есть необходимость ограничить доступ к сайту по стр��нам, то можно воспользоваться модулем `ngx_http_geoip_module`. 
Инструкции:
* https://stackoverflow.com/questions/62213884/how-install-the-geoip2-module-on-a-nginx-running-in-a-production-environment
* https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-plus/
* https://docs.nginx.com/nginx/admin-guide/dynamic-modules/geoip2/
* https://nginx.org/ru/docs/http/ngx_http_geoip_module.html
* https://dev.maxmind.com/?lang=en

Если есть желание заблокировать доступ к сайту отдельным юзер-агентам, то можно воспользоваться [отдельной инструкцией, 
на примере GPTBot](../nginx/nginx-ban-user-agent.md)
