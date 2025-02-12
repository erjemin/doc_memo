# Веб-сервер Nginx с SSL-сертификатами Let's Encrypt в контейнерах Docker

Для удобного переноса сайтов или веб-приложений между серверами, а также для упрощения обновления и обслуживания
веб-сервера nginx, удобно держать его в контейнере Docker. В данной инструкции рассмотрено развертывание веб-сервера
Nginx с SSL-сертификатами Let's Encrypt в контейнерах Docker. В качестве примера используется контейнер Portainer --
отличный инструмент для управления Docker-контейнерами через веб-интерфейс.

Соглашения: пусть наш пользователь от имени которого мы работаем -- `web`. Таким образом, домашний каталог -- `/home/web`.
Каталог для хранения данных Docker-контейнеров (место куда монтируют тома контейнера) -- `/home/web/docker-data`.

И так, для начала создадим каталог для хранения данных Portainer (можно опустить если вам не нужен Portainer):
```bash
mkdir -p /home/web/docker-data/portainer
```

## Nginx в контейнере Docker 

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
- `listen 80;` -- слушаем порт 80 (обычный HTTP-трафик);
- `server_name portainer.you.domain.name;` -- имя хоста, по которому будет доступен Portainer (замените `you.domain.name`
  на ваш домен);
- `proxy_pass http://portainer:9000;` -- проксируем запросы в контейнер Portainer, который будет доступен по хосту (имени
  контейнера `portainer`) и порту 9000;
- `proxy_set_header Host $host;` -- передаем заголовок `Host` в запросе;
- `proxy_set_header X-Real-IP $remote_addr;` -- передаем заголовок `X-Real-IP` в запросе;
- `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;` -- передаем заголовок `X-Forwarded-For` в запросе;

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
      - /etc/letsencrypt:/etc/letsencrypt
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
  - `image: portainer/portainer-ce:latest` -- используем образ Portainer Community Edition;
  - `container_name: portainer` -- имя контейнера `portainer`;
  - `volumes: ...` -- монтируем файлы или тома изнутри контейнера Portainer на хост. В данном случае монтируем: сокет --
     чтобы изнутри контейнера Portainer можно было управлять Docker, в котором сам же и работает (вот так хитро) и каталог
     `/home/web/docker-data/portainer` -- чтобы сохранять данные Portainer между перезапусками; 
  - `restart: always` -- автоматически перезапускаем контейнер при его остановке;
  - `networks: ...` -- подключаем контейнер к пользовательской (внутри-контейнерной) сети `web`.
- `nginx`:
  - `image: nginx:latest` -- используем образ Nginx;
  - `container_name: nginx` -- имя контейнера `nginx`;
  - `ports: ...` -- пробрасываем порты 80 и 443 на хост;
  - `volumes: ...` -- монтируем каталог с конфигурационными файлами Nginx и каталог с SSL-сертификатами Let's Encrypt;
  - `restart: always` -- автоматически перезапускаем контейнер при его остановке;
  - `networks: ...` -- подключаем контейнер к пользовательской (внутри-контейнерной) сети `web`.
- `networks: ...`:
  - `web`:
    - `driver: bridge` -- используем драйвер сети `bridge` (по умолчанию);
    - `ipam: ...` -- настраиваем IP-адреса для контейнеров внутри сети `web`. В данном случае используем подсеть

Сохраняем файл `docker-compose.yml` и выходим из редактора (`Ctrl + X`, затем `Y` для подтверждения).

Теперь развернем контейнеры Nginx и Portainer:
```bash
cd /home/web/docker-data
docker-compose up -d
```

После того как контейнеры запустятся, можно зайти в веб-интерфейс Portainer по адресу `http://portainer.you.domain.name`.

## Let's Encrypt в контейнере Docker