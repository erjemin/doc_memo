# Защита кластера с помощью CrowdSec

Вы наверняка использовали (или как минимум слышали) о Fail2Ban. Он очень широко распространён для защиты SSH на хостах,
противодействия сканированию сайтов, легких DDoS-атак и "фонового bot-трафика". Fail2Ban существует с 2004 года
и давно стал стандартом для защиты серверов. Но он слабо подходит для защиты кластеров Kubernetes, так поды
обслуживающие внешний трафик (Ingress-контроллеры Traefik в случае k3s) могут находиться на разных узлах кластера.
Если Fail2Ban заблокирует IP-адрес на одной ноде, то он не сможет защитить другие узлы кластера, так как они ничего
не узнают о блокировках. 

Для защиты распределённых систем (в том числе кластеров Kubernetes) набирает популярность CrowdSec. Это проект
с открытым исходным кодом, который, кроме обмена информацией об атаках между узлами (за периметром), использует
и внешний краудсорсинг (Community Blocklist) для защиты от атак. Он собирает данные о блокировках и позволяет
обмениваться этой информацией между всеми участниками сети (это отключаемая опция, и по умолчанию она отключена).
Таким образом, CrowdSec может не только защитить все узлы кластера (благодаря обмену информацией за периметром), 
о блокировать IP-адреса, еще до их атаки на ваш сервер (если данные IP уже заблокированы другими участниками CrowdSec).
А еще CrowdSec модульный, поддерживает сценарии (http-cms-scanning, sshd-bf и тому-подобное),в 60 раз быстрее
Fail2Ban (он написан на Golang), работает с IPv6 и имеет интеграции с Traefik, Cloudflare, Nginx, k3s, Docker и другими
инструментами. CrowdSec активно растёт в нише DevOps, облаков, контейнеров и кластеров Kubernetes. А еще он не
требовательный по ресурсам (~100 МБ RAM) и подходит для Orange Pi.

## Утановка CrowdSec 

В принципе, СrowdSec можно установить в кластер через Helm. Тогда он сам развернется на всех узлах кластера и это
отличный вариант для защиты Traefik (HTTP-запросы, сценарии http-cms-scanning, http-probing) и контейнеризированных
приложений (в моем случае [Gitea](k3s-migrating-container-from-docker-to-kubernetes.md), [3x-ui](k3s-3xui-pod.md) 
и тому подобного). Но мне нужно защитить еще и SSH самих узлов (узла) кластера. Поэтому план такой:

* Хостовый CrowdSec (на одном или всех узлах кластера) использует тот же Local API (LAPI) через виртуальный IP (VIP)
  Keepalived для получения бан-листа и применяет его к SSH (через Firewall Bouncer) и через тот же LAPI сообщает
  о банах ssh-bt в CrowdSec Agent внутри k3s.
* Кластерный CrowdSec Agent, внутри k3s, анализирует логи Traefik и создаёт решения (decisions) о бане IP.
* Traefik Bouncer в k3s, также подключается к LAPI для защиты HTTP (для git.cube2.ru и других web-приложений).

### CrowdSec на первом узле и защита SSH на хосте

Делаем обновляем список пактов и систему: 
```shell
sudo apt update
sudo apt upgrade
```

Добавляем репозиторий CrowdSec и ключи репозитория:
```shell
curl -s https://packagecloud.io/install/repositories/crowdsec/crowdsec/script.deb.sh | sudo bash
```

Устанавливаем CrowdSec:
```shell
sudo apt install crowdsec
```

Увидим, в число прочего:
```text
...
...
reating /etc/crowdsec/acquis.yaml
INFO[2025-xx-xx xx:xx:xx] crowdsec_wizard: service 'ssh': /var/log/auth.log
INFO[2025-xx-xx xx:xx:xx] crowdsec_wizard: using journald for 'smb'
INFO[2025-xx-xx xx:xx:xx] crowdsec_wizard: service 'linux': /var/log/syslog /var/log/kern.log
Machine 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx' successfully added to the local API.
API credentials written to '/etc/crowdsec/local_api_credentials.yaml'.
Updating hub
Downloading /etc/crowdsec/hub/.index.json
Action plan:
🔄 check & update data files


INFO[2025-05-17 17:56:45] crowdsec_wizard: Installing collection 'crowdsecurity/linux'
downloading parsers:crowdsecurity/syslog-logs
downloading parsers:crowdsecurity/geoip-enrich
downloading https://hub-data.crowdsec.net/mmdb_update/GeoLite2-City.mmdb
downloading https://hub-data.crowdsec.net/mmdb_update/GeoLite2-ASN.mmdb
downloading parsers:crowdsecurity/dateparse-enrich
downloading parsers:crowdsecurity/sshd-logs
downloading scenarios:crowdsecurity/ssh-bf
downloading scenarios:crowdsecurity/ssh-slow-bf
downloading scenarios:crowdsecurity/ssh-cve-2024-6387
downloading scenarios:crowdsecurity/ssh-refused-conn
downloading contexts:crowdsecurity/bf_base
downloading collections:crowdsecurity/sshd
downloading collections:crowdsecurity/linux
enabling parsers:crowdsecurity/syslog-logs
enabling parsers:crowdsecurity/geoip-enrich
enabling parsers:crowdsecurity/dateparse-enrich
enabling parsers:crowdsecurity/sshd-logs
enabling scenarios:crowdsecurity/ssh-bf
enabling scenarios:crowdsecurity/ssh-slow-bf
enabling scenarios:crowdsecurity/ssh-cve-2024-6387
enabling scenarios:crowdsecurity/ssh-refused-conn
enabling contexts:crowdsecurity/bf_base
enabling collections:crowdsecurity/sshd
enabling collections:crowdsecurity/linux
...
...
```

Как видим, CrowdSec сам определил, что у нас есть SSH и Linux (syslog и kern.log). Создан локальный API (LAPI)
м логин/пароль для него записан в `/etc/crowdsec/local_api_credentials.yaml`.

Далее CrowdSec загрузил парсеры, сценарии и коллекции для настройки защиты SSH и Linux.

Проверим, что CrowdSec работает:
```shell
sudo systemctl status crowdsec
```

Увидим что-то вроде:
```text
● crowdsec.service - Crowdsec agent
     Loaded: loaded (/lib/systemd/system/crowdsec.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat xxxx-xx-xx xx:xx:xx XXX; 51min ago
   Main PID: 3357651 (crowdsec)
      Tasks: 14 (limit: 18978)
     Memory: 30.7M
        CPU: 18.233s
     CGroup: /system.slice/crowdsec.service
             ├─3357651 /usr/bin/crowdsec -c /etc/crowdsec/config.yaml
             └─3357715 journalctl --follow -n 0 _SYSTEMD_UNIT=smb.service

Xxx xx xx:xx:xx xxxx systemd[1]: Starting Crowdsec agent...
Xxx xx xx:xx:xx xxxx systemd[1]: Started Crowdsec agent.
```

Проверим версию CrowdSec:
```shell
sudo cscli version
```

Увидим что-то вроде:
```text
version: v1.6.8-debian-pragmatic-arm64-f209766e
Codename: alphaga
BuildDate: 2025-03-25_14:50:57
GoVersion: 1.24.1
Platform: linux
libre2: C++
User-Agent: crowdsec/v1.6.8-debian-pragmatic-arm64-f209766e-linux
...
...
```

Проверим список установленных парсеров:
```shell
sudo cscli parsers list
```

Увидим что-то вроде:
```text
──────────────────────────────────────────────────────────────────────────────────────────────────────────────
 PARSERS                                                                                                      
──────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Name                            📦 Status    Version  Local Path                                             
──────────────────────────────────────────────────────────────────────────────────────────────────────────────
 crowdsecurity/dateparse-enrich  ✔️  enabled  0.2      /etc/crowdsec/parsers/s02-enrich/dateparse-enrich.yaml 
 crowdsecurity/geoip-enrich      ✔️  enabled  0.5      /etc/crowdsec/parsers/s02-enrich/geoip-enrich.yaml     
 crowdsecurity/smb-logs          ✔️  enabled  0.2      /etc/crowdsec/parsers/s01-parse/smb-logs.yaml          
 crowdsecurity/sshd-logs         ✔️  enabled  3.0      /etc/crowdsec/parsers/s01-parse/sshd-logs.yaml         
 crowdsecurity/syslog-logs       ✔️  enabled  0.8      /etc/crowdsec/parsers/s00-raw/syslog-logs.yaml         
 crowdsecurity/whitelists        ✔️  enabled  0.3      /etc/crowdsec/parsers/s02-enrich/whitelists.yaml       
──────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

Как видим `crowdsecurity/sshd-logs` доступны, а значит CrowdSec может парсить логи SSH. Проверим список
установленных коллекций:
```shell
 sudo cscli collections list
```

Увидим что-то вроде:
```text
─────────────────────────────────────────────────────────────────────────────────
 COLLECTIONS                                                                     
─────────────────────────────────────────────────────────────────────────────────
 Name                 📦 Status    Version  Local Path                           
─────────────────────────────────────────────────────────────────────────────────
 crowdsecurity/linux  ✔️  enabled  0.2      /etc/crowdsec/collections/linux.yaml 
 crowdsecurity/smb    ✔️  enabled  0.1      /etc/crowdsec/collections/smb.yaml   
 crowdsecurity/sshd   ✔️  enabled  0.6      /etc/crowdsec/collections/sshd.yaml  
─────────────────────────────────────────────────────────────────────────────────
```

Видим, что `crowdsecurity/sshd` доступны. Проверим список установленных сценариев:
```shell
sudo cscli scenarios list
```

Увидим что-то вроде:
```text
SCENARIOS                                                                                             
───────────────────────────────────────────────────────────────────────────────────────────────────────
 Name                             📦 Status    Version  Local Path                                     
───────────────────────────────────────────────────────────────────────────────────────────────────────
 crowdsecurity/smb-bf             ✔️  enabled  0.2      /etc/crowdsec/scenarios/smb-bf.yaml            
 crowdsecurity/ssh-bf             ✔️  enabled  0.3      /etc/crowdsec/scenarios/ssh-bf.yaml            
 crowdsecurity/ssh-cve-2024-6387  ✔️  enabled  0.2      /etc/crowdsec/scenarios/ssh-cve-2024-6387.yaml 
 crowdsecurity/ssh-refused-conn   ✔️  enabled  0.1      /etc/crowdsec/scenarios/ssh-refused-conn.yaml  
 crowdsecurity/ssh-slow-bf        ✔️  enabled  0.4      /etc/crowdsec/scenarios/ssh-slow-bf.yaml       
───────────────────────────────────────────────────────────────────────────────────────────────────────
```

Сценарии `ssh-bf`, `crowdsecurity/ssh-slow-bf` (брутфорсинг и медленный брутфорсинг SSH),
`crowdsecurity/ssh-cve-2024-6387` (защита от regreSSHion-атак на старые SSH-сервера) и 
crowdsecurity/ssh-refused-conn` (отказ соединения SSH) доступны.

Кстати, обновлять все это богачество (парсеры, сценарии, коллекции и т.п.) можно командой:
```shell
sudo cscli hub update
```

Проверим конфиги CrowdSec, и убедимся, что он анализирует логи SSH:
```shell
sudo cat /etc/crowdsec/acquis.yaml
```

Должны увидеть вот такой блок:
```yaml
filenames:
  - /var/log/auth.log
labels:
  type: syslog
---
```

Если, вдруг, такого блока нет, добавьте его (лучше в начало) и перезапустим CrowdSec. Но обычно все уже настроено.

Проверим, что CrowdSec анализирует логи SSH:
```shell
sudo cscli metrics
```

Увидим что-то вроде:
```text
╭──────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
│ Acquisition Metrics                                                                                              │
├────────────────────────┬────────────┬──────────────┬────────────────┬────────────────────────┬───────────────────┤
│ Source                 │ Lines read │ Lines parsed │ Lines unparsed │ Lines poured to bucket │ Lines whitelisted │
├────────────────────────┼────────────┼──────────────┼────────────────┼────────────────────────┼───────────────────┤
│ file:/var/log/auth.log │ 628        │ -            │ 628            │ -                      │ -                 │
│ file:/var/log/kern.log │ 2.78k      │ -            │ 2.78k          │ -                      │ -                 │
│ file:/var/log/syslog   │ 3.46k      │ -            │ 3.46k          │ -                      │ -                 │
╰────────────────────────┴────────────┴──────────────┴────────────────┴────────────────────────┴───────────────────╯
...
...
```

Как видим, CrowdSec читает `/var/log/auth.log` (логи SSH).

#### Подключаем CrowdSec для обмена данными об атаках

CrowdSec может обмениваться данными об атаках с другими участниками сети. Чтобы это сделать, нужно пойти [на сайт
CrowdSec](https://crowdsec.net/) и зарегистрироваться. После подтверждения регистрации по email, в личном кабинете
в самом низу, увидим строчку команды, типа:
```shell
sudo cscli console enroll -e context хеш-идентификатор-вашего-аккаунта
```

Скопируем эту команду и выполняем в терминале. Увидим что-то вроде:
```text
INFO manual set to true                           
INFO context set to true                          
INFO Enabled manual : Forward manual decisions to the console 
INFO Enabled tainted : Forward alerts from tainted scenarios to the console 
INFO Enabled context : Forward context with alerts to the console 
INFO Watcher successfully enrolled. Visit https://app.crowdsec.net to accept it. 
INFO Please restart crowdsec after accepting the enrollment. 
```

Как видим, нужно перезапустить CrowdSec:
```shell
sudo systemctl restart crowdsec
```

Теперь нужно снова зайти в личный кабинет CrowdSec и подтвердить подключение Security Engine.

Все! Подключение локального CrowdSec к Community Blocklist завершено. В личном кабинете можно посмотреть статистику
(по каждому Security Engine, ведь на один аккаунт можно подключить несколько хостов с CrowdSec) и даже управлять
фильтрами и сценариями (это не точно).

![crowdsec--security-engine-registration.png](../images/crowdsec--security-engine-registration.png)

Проверим, что CrowdSec получает блокировки через Community Blocklist API (CAPI):
```shell
sudo cscli metrics
```

Увидим что-то типа:
```text
...
...
╭──────────────────────────────────────────╮
│ Local API Decisions                      │
├────────────────┬────────┬────────┬───────┤
│ Reason         │ Origin │ Action │ Count │
├────────────────┼────────┼────────┼───────┤
│ generic:scan   │ CAPI   │ ban    │ 3222  │
│ smb:bruteforce │ CAPI   │ ban    │ 427   │
│ ssh:bruteforce │ CAPI   │ ban    │ 10033 │
│ ssh:exploit    │ CAPI   │ ban    │ 1315  │
╰────────────────┴────────┴────────┴───────╯
...
```

Как видим, CrowdSec получает блокировки.

#### Настройка Whitelist (белого списка)

Чтобы не заблокировать себя (случайно) нужно создать в Whitelist (белый список). Например, сделаем `home_whitelist`
(имя списка, таких списков может быть несколько, и
```shell
sudo cscli allowlist create home_whitelist -d 'Мой домашний whitelist'
```

Теперь добавим в него свои домашнюю подсеть или IP-адрес (через пробел можно указать несколько адресов или подсетей):
```shell
sudo cscli allowlist add home_whitelist 192.168.1.0/24 XXX.XXX.XXX.XXX
````

Проверим, что все добавилось:
```shell
sudo cscli allowlist inspect home_whitelist
```

Увидим что-то вроде:
```text
──────────────────────────────────────────────
 Allowlist: home_whitelist                    
──────────────────────────────────────────────
 Name                home_whitelist           
 Description         Мой домашний whitelist   
 Created at          2025-05-17T21:00:13.042Z 
 Updated at          2025-05-17T21:01:29.090Z 
 Managed by Console  no                       
──────────────────────────────────────────────

───────────────────────────────────────────────────────────────
 Value           Comment  Expiration  Created at               
───────────────────────────────────────────────────────────────
 192.168.1.0/24           never       2025-05-17T21:00:13.042Z 
 XXX.XXX.XXX.XXX          never       2025-05-17T21:00:13.042Z 
 XXX.XXX.XXX.XXX          never       2025-05-17T21:00:13.042Z 
 ...
 ...
─────────────────────────────────────────────────────────────── 
```

