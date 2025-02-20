# Настройка nginx как прямого прокси

Собственно, прямой прокси — это прокси, который просто перенаправляет запросы на другой сервер. Очень полезно,
когда у вас внутри сети есть один компьютер, который виден из интернет (DMZ или через проброс портов), и мы хотим
перенаправить внешние запросы на другие сервера внутри сети. Заодно можно настроить SSL-терминацию.

На примере AudioBookShelf, который должен быть доступен снаружи по адресу `some.you.site` у нас будет вот такой конфиг:
```nginx configuration
# config for AudioBookShelf [some.you.site]

server {
    server_name         [some.you.site];          # доменное имя сайта
    charset             utf-8;                  # кодировка по умолчанию

    access_log  /home/orangepi/web-data/audiobookshelf/logs/audiobookshelf-access.log;    # логи с доступом
    error_log   /home/orangepi/web-data/audiobookshelf/logs/audiobookshelf-error.log;     # логи с ошибками

    client_max_body_size        512M;           # максимальный объем файла для загрузки на сайт (max upload size)
    # listen    80;                                                             # managed by Certbot
    listen      443 ssl http2;                                                  # managed by Certbot
    ssl_certificate     /etc/letsencrypt/live/[some.you.site]/fullchain.pem;      # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/[some.you.site]/privkey.pem;        # managed by Certbot
    include             /etc/letsencrypt/options-ssl-nginx.conf;                # managed by Certbot
    ssl_dhparam         /etc/letsencrypt/ssl-dhparams.pem;                      # managed by Certbot

    location /favicon.ico       { root  /home/orangepi/web-data/audiobookshelf/html; }   # Расположение favicon.ico
    location /favicon.png       { root  /home/orangepi/web-data/audiobookshelf/html; }   # Расположение favicon.png
    location /robots.txt        { root  /home/orangepi/web-data/audiobookshelf/html; }   # robots.txt (dissalow all)

    location / {
        proxy_pass http://[какой-то-ip]:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Nginx-Proxy true;
        # proxy_redirect off;
        proxy_set_header X-Scheme $scheme;
        add_header Strict-Transport-Security "max-age=15552000; includeSubDomains" always;
    }
    # location / {
    #   index index.html;
    # }
}

server {
    if ($host = [some.you.site]) { return 301 https://$host$request_uri; }                    # managed by Certbot
    server_name         [some.you.site];
    listen 80;
    return 404; # managed by Certbot
}
```

## Проксирование на host если nginx находится внутри Docker 

Если [nginx находится внутри Docker или Docker Compose](../docker/docker-nginx-w-certbot.md), то он сможет увидеть 
только свои контейнерные IP-адреса и хосты. Если таким nginx нужно проксировать на сам хост, то в конфиге nginx
нужно указать:
```nginx configuration
        proxy_pass http://host.docker.internal:xxxx;
```

* `host.docker.internal` -- это специальный DNS-имя, которое указывает на хост, на котором запущен Docker.

Начиная с Docker 20.10.0+ сам контейнер надо запускать с дополнительным параметром `--add-host=host.docker.internal:host-gateway`.
```shell
docker run --add-host=host.docker.internal:host-gateway ...
```

Или добавить дополнительную инструкцию `extra_hosts: "host.docker.internal:host-gateway"` в `docker-compose.yml` при
использовании Docker Compose:
```yaml
services:
  nginx:
    image: nginx:latest
    extra_hosts:
      - "host.docker.internal:host-gateway"
    ports:
      - "80:80"
...
...
```
