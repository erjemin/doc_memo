# Развертывание сервера VPN на базе MS SSTP в контейнере Docker

SSTP (Secure Socket Tunneling Protocol) - это VPN-туннель, обеспечивающий механизм передачи трафика PPP через SSL/TLS.
Это довольно простой VPN (а значит прост в обнаружении), но его преимуществом является то, что он разработан Microsoft,
и его клиент встроен в Windows поддерживается прямо из коробки в AltLinux. SSTP использует TCP-порт 443 (можно
переназначить если на сервере уже работают сайты, использующие SSL по порту 443) и обеспечивает безопасность на
транспортном уровне с согласованием ключей, шифрованием и проверкой целостности трафика.

## Выпуск ключей (сертификата)

Для начала нужно выдать сертификат, который будет использован для шифрования соединения. Нам также понадобится
**домен**, через который мы будем подключаться к серверу. Домен должен быть обязательно направлен на ip адрес сервера
в DNS,

Создадим на сервере пару ключей `server_key.pem` и `server_cert.pem` (необходимо заменить слово **[MYDOMAIN]** в команде
на наш домен, а **[EMAIL]** на наш, или вообще произвольный, email):

```bash
openssl req -x509 -nodes \
   -newkey rsa:4096 \
   -keyout server_key.pem \
   -out server_cert.pem \
   -days 1825 \
   -subj "/C=XX/ST=XX/L=XX/O=XX/OU=XX/CN=[MYDOMAIN]/emailAddress=[EMAIL]"
``` 

В текущем каталоге появятся два файла: `server_key.pem` и `server_cert.pem`. Первый из них -- это закрытый ключ,
второй -- сертификат, который мы будем использовать для шифрования трафика. 

## Запуск сервера

Загружаем образ softethervpn с DockerHub:
```bash
docker pull fernandezcuesta/softethervpn
```

Это контейнер сделанный из [официального мастер-репозитория SoftEtherVPN](https://github.com/SoftEtherVPN/SoftEtherVPN)
-- универсального VPN поддерживающего помимо `MS-SSTP` протоколы `SSL-VPN (HTTPS)`,  `WireGuard`,  `OpenVPN`,
`IPsec`, `L2TP`, `L2TPv3` и `EtherIP`.

Запускаем контейнер сконфигурировав его под **SSTP** и передавая ранее созданные сертификаты.:
```bash
sudo docker run \
      --cap-add NET_ADMIN
       -p 22:443/tcp
       -e SSTP_ENABLED=1
       -e USERNAME=[USER]
       -e PASSWORD=[USER_PASS]
       -e SERVER_PWD=[SERVER_PASS]
       -e CERT="$(cat server_cert.pem)"
       -e KEY="$(cat server_key.pem)" 
       -d --name=sstp-vpn --restart=always fernandezcuesta/softethervpn
```

Обратите внимание, что так как в нашем случае на сервере работают сайты использующие SSL, то порт `443` занят
и мы переназначаем его на `22`. Наш VPN-тоннель будет мимикрировать не под SSL а под SSH. И, конечно, SSH не должен
использовать 22 порт, а тоже должен быть переназначен (впрочем перевести SSH по порт со значением больше 1024 полезно
и с точки зрения безопасности).

Так же замените:
* `[SERVER_PASS]` -- пароль сервера, должен быть минимум 12 символов, включающий цифры, большие и маленькие буквы.
* `[USER]` -- имя пользователя, которое будет использоваться для подключения к VPN-серверу.
* `[USER_PASS]` -- пароль пользователя, должен быть минимум 12 символов, включающий цифры, большие и маленькие буквы.

Проверим, что наш контейнер запущен и работает:
```bash
docker ps
```

Вывод должен быть примерно таким:
```bash
CONTAINER ID   IMAGE                                  COMMAND                  CREATED        STATUS       PORTS                                                                                  NAMES
...
...
d00d6f42694e   fernandezcuesta/softethervpn           "/entrypoint.sh /usr…"   8 hours ago    Up 8 hours   500/udp, 1194/udp, 4500/udp, 1701/tcp, 0.0.0.0:22->443/tcp, :::22->443/tcp             sstp-server
...
```

Для docker-compose это будет примерно вот такой `docker-compose.yml`:
```yaml
ersion: "3.7"
services:
  sstp-vpn:
    image: fernandezcuesta/softethervpn
    container_name: 'sstp-server'
    restart: 'always'
    ports:
      - 22:443/tcp
    cap_add:
      - NET_ADMIN
    environment:
      - SSTP_ENABLED=1
      - USERNAME=[USER]
      - PASSWORD=[USER_PASS]
      - SERVER_PWD=[SERVER_PASS]
      - CERT=-----BEGIN CERTIFICATE-----MIIF2zCCA8OgAwIBAgIU.....ZnOLc-----END CERTIFICATE-----
      - KEY=-----BEGIN PRIVATE KEY-----MIIJQgIBADANBg...6i4xupFQ==-----END PRIVATE KEY-----
```

Обратите внимание, что придется вставить сертификат и ключ из файла... и вставить в одну строку, иначе docker-compose будет ругаться.

Запускаем docker-compose:
```bash
sudo docker-compose up -d
```

### Создание дополнительного пользователя (не обязательно)

Если понадобится создать несколько подключений к серверу, то нужно выполнить следующие две команды для каждого пользователя (подключения):
```bash
docker exec -it sstp-vpn ./vpncmd [MYDOMAIN] /SERVER /PASSWORD:"$SERVER_PASS" /ADMINHUB:DEFAULT /CSV /CMD UserCreate [USER2] /GROUP:none /REALNAME:none /NOTE:none
docker exec -it sstp-vpn ./vpncmd [MYDOMAIN] /SERVER /PASSWORD:"$SERVER_PASS" /ADMINHUB:DEFAULT /CSV /CMD UserPasswordSet [USER2] /PASSWORD:[USER2_PASS]
``` 

Где:
* `[MYDOMAIN]` -- заменить на доменное имя, которе указывали ранее при выдаче сертификата.
* `[USER2]` -- имя нового соединения (пользователя)
* `[USER2_PASS]` -- пароль нового соединения (пользователя)

### Дополнительные настройки (не обязательно)

Рекомендуется увеличить максимальный размер буфера. Эта команда увеличит максимальный размер буфера приема примерно до 2.5МБ:
```bash
sudo sysctl -w net.core.rmem_max=2500000
```

## Подключение клиента

