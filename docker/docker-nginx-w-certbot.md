# Веб-сервер Nginx с SSL-сертификатами Let's Encrypt в контейнерах Docker

Для удобного переноса сайтов или веб-приложений между серверами, а также для упрощения обновления и обслуживания
веб-сервера nginx, удобно держать его в контейнере Docker. В данной инструкции рассмотрено развертывание веб-сервера
Nginx с SSL-сертификатами Let's Encrypt в контейнерах Docker. В качестве примера сайта используется контейнер Portainer --
отличный инструмент для управления Docker-контейнерами через веб-интерфейс.

### Соглашения:
|                             |                                                                             |
|-----------------------------|----------------------------------------------------------------------------------------| 
| `web`                       | Пользователь от имени которого мы работаем                                             |                                                                                                            
| `/home/web`                 | Домашний каталог.                                                                      |
| `/home/web/docker-data`     | Каталог для хранения данных Docker-контейнеров (место куда монтируют тома контейнеров) |
| `portainer.you.domain.name` | Домен, по которому будет доступен Portainer                                            |
| `email@you.domain.name`     | Email для сертификатов Let's Encrypt                                                   |                                                                                     |
 
Чтобы не случилось путаницы во время копи-паста рекомендуется сразу произвести все замены в тексте.  

## Portainer + nginx в контейнерах Docker

И так, для начала создадим каталог для хранения данных Portainer (можно опустить если вам не нужен Portainer):
```bash
mkdir -p /home/web/docker-data/portainer
```

Теперь создадим каталог для хранения конфигурационных файлов Nginx. Сам Nginx будет сидеть в контейнере, но
конфигурационные файлы, которые он будет использовать, находятся на хосте в каталоге `/home/web/docker-data/nginx/conf.d`:
```bash
mkdir -p /home/web/docker-data/nginx/conf.d
```

Теперь создадим файл конфигурации Nginx, который будет использоваться для проксирования запросов к контейнеру Portainer:
```bash
nano /home/web/docker-data/nginx/conf.d/portainer.conf
```

Вставьте в файл следующее содержимое:
```nginx configuration
server {
    listen 80;
    server_name portainer.you.domain.name;

    location / {
        proxy_pass http://portainer:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Что происходит в этом файле конфигурации:
- `listen 80;` — слушаем порт 80 (обычный HTTP-трафик);
- `server_name portainer.you.domain.name;` — имя хоста, по которому будет доступен Portainer (замените `you.domain.name`
  на ваш домен);
- `proxy_pass http://portainer:9000;` — проксируем запросы в контейнер Portainer, который будет доступен по хосту (имени
  контейнера `portainer`) и порту 9000;
- `proxy_set_header Host $host;` — передаем заголовок `Host` в запросе;
- `proxy_set_header X-Real-IP $remote_addr;` — передаем заголовок `X-Real-IP` в запросе;
- `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;` — передаем заголовок `X-Forwarded-For` в запросе;

Сохраните файл и выйдите из редактора (`Ctrl + X`, затем `Y` для подтверждения).

Теперь создадим файл `docker-compose.yml` для развертывания контейнеров Nginx и Portainer:
```bash
nano /home/web/docker-data/docker-compose.yml
```

Вставьте в файл следующее содержимое:  
```yaml
version: '3'
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    # Гасим порт 9000, чтобы он не светил на хост, а был доступен только во внутри-контейнерной сети
    # ports:
    #   - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /home/web/docker-data/portainer:/data
    restart: always
    networks:
      - web
      # Можно закомментировать строку выше и раскомментировать строки ниже если зачем-то нужен закрепленный IP-адрес
      # web:
      #   ipv4_address: 172.20.0.10     

  nginx:
    image: nginx:latest
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /home/web/docker-data/nginx/conf.d:/etc/nginx/conf.d
    restart: always
    networks:
      - web

networks:
  web:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24  # Подсеть для пользовательской сети
```

Что у нас настроено в этом `docker-compose.yml`:
- `portainer` (у вас его может и не быть, или быть какой-то другой сервис, который вы будете производить):
  - `image: portainer/portainer-ce:latest` — используем образ Portainer Community Edition;
  - `container_name: portainer` — имя контейнера `portainer`;
  - `volumes: ...` — монтируем файлы или тома изнутри контейнера Portainer на хост. В данном случае монтируем: сокет --
     чтобы изнутри контейнера Portainer можно было через Docker API управлять Docker самого хоста (вот так хитро) и
     каталог `/home/web/docker-data/portainer` — чтобы сохранять настройки и данные Portainer между перезапусками; 
  - `restart: always` — автоматически перезапускаем контейнер при его остановке;
  - `networks: ...` — подключаем контейнер к пользовательской (внутри-контейнерной) сети `web`.
- `nginx`:
  - `image: nginx:latest` — используем образ Nginx;
  - `container_name: nginx` — имя контейнера `nginx`;
  - `ports: ...` — пробрасываем порты 80 и 443 на хост;
  - `volumes: ...` — монтируем каталог с конфигурационными файлами Nginx;
  - `restart: always` — автоматически перезапускаем контейнер при его остановке;
  - `networks: ...` — подключаем контейнер к пользовательской (внутри-контейнерной) сети `web`.
- `networks: ...`:
  - `web`:
    - `driver: bridge` — используем драйвер сети `bridge` (по умолчанию);
    - `ipam: ...` — настраиваем IP-адреса для контейнеров внутри сети `web`. В данном случае используем подсеть

Сохраняем файл `docker-compose.yml` и выходим из редактора (`Ctrl + X`, затем `Y` для подтверждения).

Теперь развернем контейнеры Nginx и Portainer:
```bash
cd /home/web/docker-data
docker-compose up -d
```

После того как контейнеры запустятся, можно зайти в веб-интерфейс Portainer по адресу `http://portainer.you.domain.name`.

## Добавляем контейнер Let's Encrypt

Создадим каталог для хранения ключей сертификатов Let's Encrypt и временных файлов для проверки владения доменом:
```bash
mkdir -p /home/web/docker-data/letsencrypt
mkdir -p /home/web/docker-data/letsencrypt/_cert
mkdir -p /home/web/docker-data/letsencrypt/_ownership_check
```

Добавим в `docker-compose.yml` Certbot-контейнер (для получения сертификатов Let's Encrypt):

```yaml
  certbot:
    image: certbot/certbot:latest
    container_name: letsencrypt-certbot
    volumes:
      - /home/web/docker-data/letsencrypt/_cert:/etc/letsencrypt          # Для хранения сертификатов
      - /home/web/docker-data/letsencrypt/_ownership_check:/var/www/html  # Для временных файлов для Let's Encrypt
      - /var/run/docker.sock:/var/run/docker.sock                         # Для управления контейнерами Docker
    networks:
      - web
    entrypoint: "/bin/sh -c 'apk add --no-cache curl && trap exit TERM; while :; do sleep 12h & wait $${!}; certbot renew --deploy-hook /etc/letsencrypt/renewal-hooks/deploy/restart-nginx.sh; done'"
```

*Что тут происходит и зачем нам такие мапинги томов?*

1. Когда certbot запрашивает сертификат у Let's Encrypt, то тот требует подтверждения владения доменом. При работе
   certbot в контейнере самый популярный (и лучший при работе в контейнере) способ подтверждения — это HTTP-проверка.
   Certbot создает временные файлы в каталоге `/var/www/html/letsencrypt` (мы будем указывать этот каталог при инициализации
   сертификата). Сервер же Let's Encrypt перед выдачей сертификата делает HTTP-запрос к этому временному файлу по URL
   (например, в нашем случае по `http://portainer.you.domain.namey/.well-known/acme-challenge/`) и если файл доступен,
   владение доменом считается подтвержденным, и сертификат выдается.

   Таким образом, маппинг `/home/web/docker-data/letsencrypt/_ownership_check:/var/www/html` позволяет certbot создавать 
   временные файлы в каталоге `/home/web/docker-data/letsencrypt/_ownership_check` хоста, и эти файлы nginx сможет 
   "отдать" при проверке со стороны Let's Encrypt.

   Конечно, нам еще придется настроить nginx, чтобы он мог отдавать эти временные файлы. Но об этом чуть позже.

2. Мапинг `/home/web/docker-data/letsencrypt/_cert:/etc/letsencrypt` позволяет certbot сохранять получаемые
   и обновляемые сертификаты Let's Encrypt в каталоге хоста `/home/web/docker-data/letsencrypt/_cert`, чтобы они не 
   пропадали при перезапуске контейнера certbot.

3. Мапинг `/var/run/docker.sock:/var/run/docker.sock` позволяет certbot управлять контейнерами Docker, чтобы он мог
   перезапускать контейнеры (в нашем случае контейнер nginx) в случае обновления сертификатов. В принципе, этот мапинг
   можно не делать, и обойтись хуками certbot, но это сложнее.

4. `entrypoint: ...`:
   - `apk add --no-cache curl` — устанавливаем пакет `curl`, который потребуется для работы с Docker API через сокет;
   - `trap exit TERM;` — устанавливаем обработчик сигнала `TERM`, чтобы контейнер certbot корректно завершал работу;
   - `while :; do sleep 12h & wait $${!}; certbot renew --deploy-hook /etc/letsencrypt/renewal-hooks/deploy/restart-nginx.sh; done`
     — это скрипт, который запускается при старте контейнера certbot. Он запускает certbot в режиме `renew` каждые 12 часов.
     Таким образом, сертификаты будут автоматически провереться каждые 12 часов и обновляться, если это необходимо.

Также нам нужно добавить маппинг тома для сертификатов Let's Encrypt в контейнере `nginx`. Теперь описание этого контейнера
в `docker-compose.yml` будет выглядеть так:
```yaml
    nginx:
    image: nginx:latest
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /home/web/docker-data/nginx/conf.d:/etc/nginx/conf.d
      - /home/web/docker-data/letsencrypt/_cert:/etc/letsencrypt                 # <- этот маппинг для сертификатов
      - /home/web/docker-data/letsencrypt/_ownership_check:/var/www/letsencrypt  # <- этот маппинг для временных файлов
    restart: always
    networks:
      - web
```

*Тут происходит очень похожий маппинг тома для сертификатов Let's Encrypt и временных файлов для проверки
владения, но теперь для контейнера nginx*:

- `volumes: ...`:
  - `/home/web/docker-data/letsencrypt/_cert:/etc/letsencrypt` — маппинг тома для сертификатов Let's Encrypt, чтобы их
    можно было использовать в контейнере nginx;
  - `/home/web/docker-data/letsencrypt/_ownership_check:/var/www/letsencrypt` — маппинг тома временных файлов для
    проверки владения доменом со стороны Let's Encrypt.

Сохраняем файл `docker-compose.yml`.

Теперь нам нужно настроить nginx, чтобы он мог отдавать временные файлы Certbot. Для этого изменим конфигурационный файл
`/home/web/docker-data/nginx/conf.d/portainer.conf` на следующий:
```nginx configuration
server {
    listen 80;
    server_name portainer.you.domain.name;

    location ^~ /.well-known/acme-challenge/ {
        root /var/www/letsencrypt;
        # Или так, с помощью alias:
        # alias /var/www/letsencrypt/.well-known/acme-challenge/;
        # try_files $uri =404;
    }
    
    location / {
        proxy_pass http://portainer:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Как видно, мы добавили новый *location*-блок, который отдает временные файлы Certbot из каталога `/var/www/letsencrypt/`
(а это каталог контейнера `nginx`, который мы ранее замаппили в каталог хоста `/home/web/docker-data/letsencrypt/_ownership_check`).
Он явно указывает, что запросы к `/.well-known/acme-challenge/` не должны идти в прокси, а должны обслуживаться локально.
Используем директиву `location ^~` — она приоритетнее `location /` и такой *location* будет срабатывать даже при
включённом proxy_pass.

**Важно!** После того как сертификаты Let's Encrypt будут получены, не надо удалять этот `location ^~` из конфигурации!
Он нужен для автоматического обновления сертификатов. При обновлении сертификатов certbot будет снова создавать
временные файлы в каталоге `/var/www/letsencrypt/` а Let's Encrypt проверять их доступность. Если nginx не сможет отдать
эти файлы, то обновление сертификатов не произойдет.

Останавливаем docker-compose и запускаем только nginx из него:
```bash
cd /home/web/docker-data
docker-compose down
docker-compose up -d nginx
```

Теперь можно запустить certbot для получения сертификатов Let's Encrypt:
```bash
docker run --rm --name letsencrypt-certbot \
  -v /home/web/docker-data/letsencrypt/_ownership_check:/var/www/html \
  -v /home/web/docker-data/letsencrypt/_cert:/etc/letsencrypt \
  certbot/certbot certonly --webroot \
  -w /var/www/html \
  -d portainer.you.domain.name \
  --email email@you.domain.name \
  --agree-tos --no-eff-email --force-renewal
```

Если все пройдет успешно (должно пройти) мы увидим примерно такой вывод:
```text
certbot certonly --webroot -w /var/www/html -d portainer.you.domain.name --email email@you.domain.name --agree-tos --no-eff-email --force-renewal
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Requesting a certificate for portainer.you.domain.name

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/portainer.you.domain.name/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/portainer.you.domain.name/privkey.pem
This certificate expires on 2025-05-14.
These files will be updated when the certificate renews.
NEXT STEPS:
- The certificate will need to be renewed before it expires. Certbot can automatically renew the certificate in the background, but you may need to take steps to enable that functionality. See https://certbot.org/renewal-setup for instructions.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

Если у вас что-то пойдёт не так, то скорее всего что-то напутано в маппингах томов, или в конфигурации nginx. Вы можете
посмотреть логи certbot (возможно вам придется замаппить каталог логов из контейнера certbot на хост).

Уже хочется проверить, что все работает? Рано! Нам нужно добавить в конфигурацию nginx SSL-сертификаты и настроить
перенаправление с HTTP на HTTPS. Отредактируем конфиг nginx, теперь он будет выглядеть так:
```nginx configuration
server {
    listen 443 ssl;
    server_name portainer.you.domain.name;

    ssl_certificate /etc/letsencrypt/live/portainer.you.domain.name/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/portainer.you.domain.name/privkey.pem;

    # Рекомендуемые SSL настройки
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location ^~ /.well-known/acme-challenge/ {
        root /var/www/letsencrypt;
        # Или так, с помощью alias:
        # alias /var/www/letsencrypt/.well-known/acme-challenge/;
        # try_files $uri =404;
    }

    location / {
        proxy_pass http://portainer:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Перенаправление с HTTP на HTTPS
server {
    listen 80;
    server_name portainer.you.domain.name;
    return 301 https://$host$request_uri;
}
```

Что изменилось в конфигурации:
- Добавлены директивы для SSL для сервера в котором живет прокси на Portainer:
  - `listen 443 ssl;` — теперь сервер слушает порт 443 (HTTPS) и использует SSL;
  - `ssl_certificate ...` и `ssl_certificate_key ...` — указываем пути к сертификату и ключу сертификата Let's Encrypt;
  - `ssl_protocols TLSv1.2 TLSv1.3;` — указываем протоколы SSL/TLS, которые будут использоваться;
  - `ssl_ciphers HIGH:!aNULL:!MD5;` — указываем шифры, которые будут использоваться (`HIGH` → Разрешает только сильные
    шифры, например, AES256, `!aNULL` → Запрещает анонимные шифры, которые не используют аутентификацию и уязвимы
    к MITM-атакам, `!MD5` → Запрещает использование хэша MD5, так как он давно признан небезопасным;
  - `ssl_prefer_server_ciphers on;` — указываем, что сервер предпочтет использовать свои шифры, а не клиентские.
- Добавлен блок `server` для перенаправления с HTTP на HTTPS:
  - `return 301 https://$host$request_uri;` — перенаправляем все запросы с порта 80 на порт 443 c кодом 301 (перемещено
    навсегда).

Сохраним файл и, наконец запустим все контейнеры:
```bash
cd /home/web/docker-data
docker-compose down
docker-compose up -d
```

Теперь можно проверить, что все работает, и зайдите в веб-интерфейс Portainer по адресу `https://portainer.you.domain.name`.

Осталось проверить, что перевыпуск сертификатов будет происходить и мы всё ещё ничего не сломали в мапингах:
```bash
docker exec -it letsencrypt-certbot certbot renew --dry-run -v
```

Должны увидеть примерно такой вывод:
```text
Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Processing /etc/letsencrypt/renewal/portainer.you.domain.name.conf
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Certificate not due for renewal, but simulating renewal for dry run
Plugins selected: Authenticator webroot, Installer None
Simulating renewal of an existing certificate for portainer.you.domain.name
Performing the following challenges:
http-01 challenge for portainer.you.domain.name
Using the webroot path /var/www/html for all unmatched domains.
Waiting for verification...
Cleaning up challenges

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Congratulations, all simulated renewals succeeded: 
  /etc/letsencrypt/live/portainer.you.domain.name/fullchain.pem (success)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

И наконец, при обновлении сертификатов нужно перезапускать контейнер `nginx` чтобы он перподключил новые сертификаты. 
Изнутри контейнера `letsencrypt-certbot` мы не можем управлять контейнерами Docker (и даже ничего не знаем о них),
но можно добавить **хук** в `letsencrypt-certbot`, который будет перезапускать контейнер `nginx` сразу после успешного
обновления сертификатов (и только если обновление прошло успешно)!

Что такое **хук**? Это скрипт, который выполняется в определенный момент жизненного цикла `certbot`. Certbot автоматически
ищет и выполняет скрипты в специальных каталогах, которые и называют **хуками**:
- `/etc/letsencrypt/renewal-hooks/deploy/` – скрипты выполняются после успешного обновления;
- `/etc/letsencrypt/renewal-hooks/pre/` – выполняются до начала обновления;
- `/etc/letsencrypt/renewal-hooks/post/` – выполняются после любой попытки обновления (даже если оно не удалось);

Таким образом наш маппинг `/home/web/docker-data/letsencrypt/_cert:/etc/letsencrypt` обеспечит нам исполнение хуков
внутри контейнера certbot, хотя сами хуки будут лежать на хосте. Добавим скрипт
`/home/web/docker-data/letsencrypt/_cert/renewal-hooks/deploy/restart-nginx.sh`:
```bash
mkdir -p /home/web/docker-data/letsencrypt/_cert/renewal-hooks/deploy
sudo nano /home/web/docker-data/letsencrypt/_cert/renewal-hooks/deploy/restart-nginx.sh
```

Обратите внимание, что хук мы создаем через `sudo`. Все содержимое `/home/web/docker-data/letsencrypt/_cert` было создано
изнутри контейнера `certbot`, и поэтому принадлежит пользователю `root` (`root` из контейнера, но на хосте он превратился
в `root` хоста). Поэтому нам нужно использовать `sudo` чтобы редактировать файл.

Вставим в скрипт следующее содержимое:
```bash
#!/bin/sh
echo "СРАБОТАЛ ХУК \"deploy/restart-nginx.sh\": перезапускаем контейнер nginx"
curl -s -o /dev/null --unix-socket /var/run/docker.sock -X POST http:/v1.41/containers/nginx/restart
```

Что тут важно? Несмотря на то, что скрипт лежит на хосте, он все равно будет исполняться внутри контейнера `letsencrypt-certbot`.
Потому просто так перезапустить контейнер `nginx` не получится. Нам нужно использовать Docker API, чтобы сделать это.
Именно поэтому мы в нашем `docker-compose.yml` замаппили сокет Docker в контейнер `letsencrypt-certbot` и устанавливали `curl`.

Сохраним файл и сделаем его исполняемым:
```bash
sudo chmod +x /home/web/docker-data/letsencrypt/_cert/renewal-hooks/deploy/restart-nginx.sh
```

Теперь можно проверить как все это сработает. Посмотрим на наши контейнеры:
```bash
docker ps
```

И увидим наши три контейнера:
```text
CONTAINER ID   IMAGE                              COMMAND                  CREATED          STATUS                PORTS                                                                      NAMES
8be62353e563   nginx:latest                       "/docker-entrypoint.…"   13 minutes ago   Up 9 minutes          0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp   nginx
6b805c7df486   portainer/portainer-ce:latest      "/portainer"             13 minutes ago   Up 13 minutes         8000/tcp, 9000/tcp, 9443/tcp                                               portainer
dd0b7a683dde   certbot/certbot:latest             "/bin/sh -c 'apk add…"   13 minutes ago   Up 13 minutes         80/tcp, 443/tcp                                                            letsencrypt-certbot
```

Дадим команду в контейнер `letsencrypt-certbot` на принудительное обновление сертификатов:
```bash
docker exec -it letsencrypt-certbot certbot renew --force-renewal
```

Увидим как certbot обновляет сертификаты:
```text
aving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Processing /etc/letsencrypt/renewal/portainer.you.domain.name.conf
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Renewing an existing certificate for portainer.you.domain.name
Hook 'deploy-hook' ran with output:
 СРАБОТАЛ ХУК "deploy/restart-nginx.sh": перезапускаем контейнер nginx

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Congratulations, all renewals succeeded: 
  /etc/letsencrypt/live/portainer.you.domain.name/fullchain.pem (success)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

Теперь можно убедиться, что контейнер nginx перезапустился:
```bash
docker ps
```

И увидим, что STATUS контейнера nginx изменился (uptime сброшен):
```text
8be62353e563   nginx:latest                       "/docker-entrypoint.…"   2 seconds ago    Up 24 seconds         0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp   nginx
6b805c7df486   portainer/portainer-ce:latest      "/portainer"             13 minutes ago   Up 13 minutes         8000/tcp, 9000/tcp, 9443/tcp                                               portainer
dd0b7a683dde   certbot/certbot:latest             "/bin/sh -c 'apk add…"   13 minutes ago   Up 13 minutes         80/tcp, 443/tcp 
```

**Все!** Теперь у нас полностью контейнеризированное решение, без лишних зависимостей на хосте, и при переносе каталога
`~/docker-data` на другой сервер (с Docker + docker-compose) всё должно точно также запуститься.


