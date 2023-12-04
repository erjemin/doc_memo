# Развертывание MariaDB (MySQL) в контейнере Docker

Создаем и запускаем контейнер с MariaDB. Т.к. при перезапуске контейнера данные из базы уничтожаются, то
пробросим из нее том с данными наружу во внешний каталог (том), в данном случае `/home/[user]/docker-data/maria-db/` :
```shell
sudo docker run --name MariaDB_11.1.2 --restart=always \
    -e MYSQL_ROOT_PASSWORD=qwaseR12 \
    -e TZ=Europe/Moscow \
    -p 127.0.0.1:3306:3306 \
    -v /home/[user]/docker-data/maria-db/:/var/lib/mysql \
    -v /etc/timezone:/etc/timezone:ro \
    -v /etc/localtime:/etc/localtime:ro \
    -d mariadb:11.1.2
```

Для docker-compose это будет вот такой `docker-compose.ym`:
```yaml
version: "3.1"
services:
  mariadb:
    container_name: MariaDB_11.1.2
    image: mariadb:11.1.2
    volumes:
      - /home/[user]/docker-data/maria-db:/var/lib/mysql
      # ↓↓↓ это устанавливает часовой пояс как в системе, но сработает только под Linux: 
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    environment:
      - MARIADB_ROOT_PASSWORD=***
      # - MARIADB_AUTO_UPGRADE=yes
      #  ↓↓↓ это устанавливает часовой пояс принудительно, и сработает и под MacOS, и под Windows 
      - TZ=Europe/Moscow
      #  ↓↓↓ всякие оптимизационные параметры устанавливаем вот так:
      - sql-mode=""
      - ft_min_word_len=1
      - wait_timeout=600
      - max_allowed_packet=1G
      - innodb_buffer_pool_size=100M
      - net_read_timeout=3600
      - net_write_timeout=3600
    ports:
      - 127.0.0.1:3306:3306
    command:
      --bind-address=0.0.0.0
      --skip_ssl=true
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_unicode_ci
    restart: always
```

Далее надо создать суперпользователя в базе данных. Для этого надо зайти в CLI (командную строку)
нашего контейнера, выполнив команду (в Docker Desktop под Windows или MacOS -- просто выбираем
контейнер в списке и нажимаем кнопку `CLI` или в новых версиях вкладка `exec`):
```shell
docker exec -it MariaDB_11.1.2 bash
```

После этого внутри контейнера войти в MariaDB:
```shell
mariadb -u root -p
```

Спросит пароль -- просто жмем `Enter`. Создаём супер-пользователя (например: `superdb_user`) командами:
```sql
CREATE USER 'superdb_user'@'%' IDENTIFIED BY 'secret-password';
GRANT ALL PRIVILEGES ON *.* TO 'superdb_user'@'%';
FLUSH PRIVILEGES;
exit;
```

После этого можно выйти из контейнера:
```shell
exit
```

Теперь можно коннектится к базе внутри контейнера точно так же как к базе установленным обычным образом.
