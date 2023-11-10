# Развертывание прокси базе Shadowsocks (сервер и клиент)

Shadowsocks -- это не VPN, это -- защищенный прокси-сервер Socks5, он перенаправляет трафик через сервер, но
не предоставляют полную анонимность, не защищает весь интернет-трафик, а только трафик приложений настроенных
на этот прокси.

Несмотря на то, что Shadowsocks это приложение с открытым кодом и его просмотрело много людей, он не проходил
официального аудита безопасности, а значит гарантий полной безопасности при его использовании нет. 

Документация по Shadowsocks [в репозитории GitHub](https://github.com/shadowsocks/shadowsocks-libev/).

## Сервер

Загружаем образ shadowsocks с DockerHub:
```bash
docker pull shadowsocks/shadowsocks-libev
```

Запускаем контейнер:
```bash
sudo docker run \
      -e PASSWORD=very#knotty^password! \
      -e TZ=Europe/Istanbul
      -e METHOD=aes-256-cfb \
      -p 8391:8388/tcp \
      -p 8391:8388/udp \
      -d --restart=always --name=ss01-server shadowsocks/shadowsocks-libev:latest
```

Для docker-compose это будет примерно вот такой `docker-compose.yml`:
```yaml
version: "3.7"
services:
  ss-server:
    image: shadowsocks/shadowsocks-libev:latest
    container_name: 'ss02-server'
    environment:
      - TZ=Europe/Moscow
      - PASSWORD=another~yet#knotty^password!
      - METHOD=aes-256-cfb
      - ARGS=--fast-open
    ports:
      - "8390:8388/tcp"
      - "8390:8388/udp"
    restart: unless-stopped
```

Важными при развёртывании является, пожалуй, только параметр `PASSWORD` и порты (в нашем случае сервер shadowsocks будет
виден в интернет по порту `8391`) . Также можно определить следующие параметры:
* `SERVER_ADDR` -- IP или домен для привязки, по умолчанию `0.0.0.0`.
* `SERVER_ADDR_IPV6` -- адрес IPv6 для привязки, по умолчанию `::0`.
* `METHOD` -- метод шифрования, по умолчанию `aes-256-gcm`. Так же поддерживаются `c4-md5`, `aes-128-gcm`,
              `aes-192-gcm`, `aes-128-cfb`, `aes-192-cfb`, `aes-256-cfb`, `aes-128-ctr`, `aes-192-ctr`, `aes-256-ctr`,
              `camellia-128-cfb`, `camellia-192-cfb`, `camellia-256-cfb`, `bf-cfb`, `chacha20-ietf-poly1305`,
              `xchacha20-ietf-poly1305`, `salsa20`, `chacha20` и `chacha20-ietf`. Из этого списка особого внимание
              заслуживают **chacha20-ietf-poly1305** так это шифрование поддерживает [Outline VPN](https://getoutline.org/).
* `TIMEOUT` -- по умолчанию `300`.
* `DNS_ADDRS` -- DNS-серверы для перенаправления запросов поиска NS, по умолчанию: `8.8.8.8,8.8.4.4`.
* `TZ` -- часовой пояс, по умолчанию `UTC`.
* `ARGS` -- дополнительные аргументы, поддерживаемые, `ss-server`, например, для запуска в режиме `--fast-open` или
  с подключёнными плагинами, типа relay-сервера `2ray`.

## Клиент

Ссылки на загрузку различных клиентов Shadowsocks с графическим интерфейсом можно [на сайте Shadowsocks](https://shadowsocks.org/doc/getting-started.html#gui-clients).
Ему нужно будет указать IP-адрес сервера, порт, пароль и метод шифрования. Например, вот такой конфиг для клиента:
```json
{
    "server": "11.22.33.44",
    "server_port": 8391,
    "local_address": "0.0.0.0",
    "local_port": 1080,
    "password": "very#knotty^password!",
    "timeout": 600,
    "method": "aes-256-gcm"
}
```

Но можно и воспользоваться Docker, развернуть клиент в контейнере и работать с ним как локальным Socks5-прокси (другие
пользователи локальной сети тоже могут работать через этот Socks5).

Загружаем образ shadowsocks-клиента:
```bash
docker pull littleqz/shadowsocks-client
```

Запускаем контейнер:
```bash
sudo docker run \
      -e SERVER=11.22.33.44 \
      -e SERVER_PORT=8391 \
      -e LOCAL_PORT=1080 \
      -e PASSWORD=very#knotty^password! \
      -e METHOD=aes-256-cfb \
      -p 1080:1080 \
      -d --restart=always --name=ss_proxy_izmir littleqz/shadowsocks-client
```

Теперь если в настройках браузера или другого приложения указать в качестве прокси-сервера `localhost` и порт `1080`,
то весь трафик будет перенаправляться через прокси-сервер. Так же можно указать в качестве прокси-сервера IP-адрес
компьютера, где запущен контейнер с прокси-сервером, и тогда другие пользователи локальной сети тоже смогут
использовать этот прокси-сервер.

Подобным образом можно запустить несколько контейнеров с shadowsocks-клиентами, например, для разных стран, настроив 
их на разные порты и переключаться между ними.

Для docker-compose это будет примерно вот такой `docker-compose.yml` (настроено сразу на два клиента):
```yaml
version: "3.7"
services:
   ss_proxy_izmir:
      image: littleqz/shadowsocks-client
      container_name: ss_proxy_izmir
      environment:
         - SERVER=11.22.33.44
         - SERVER_PORT=8391
         - LOCAL_PORT=1080
         - PASSWORD=another~yet#knotty^password!
         - TIMEOUT=60
         - METHOD=aes-256-cfb
      # expose:
         # - 1080
      ports:
         - 1080:1080
      restart: always

   ss_proxy_msk:
      image: littleqz/shadowsocks-client
      container_name: ss_proxy_msk
      environment:
         - SERVER=55.66.77.88
         - SERVER_PORT=8391
         - LOCAL_PORT=1081
         - PASSWORD=very#knotty^password!
         - TIMEOUT=60
         - METHOD=aes-256-cfb
      ports:
         - 1081:1081
      restart: always
```

