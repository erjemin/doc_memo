# Контейнер с MySQL под Windows 10
 
Сначала надо перенастроть Dockers в advansed режим и разрешить использовать experimental-контенеры. Для этого
* Щелкнуть в трее на значок **Docker** правой кнопокй мыши.
* Выбрать пункт _Settings_
* В блоке _Docker Engine_ в конфигурационный файл добавить `"experimental": true` (в старых версиях вкладка называется _Deamon_). Например получится вот так:
 ```json
{
  "registry-mirrors": [],
  "insecure-registries": [],
  "debug": true,
  "experimental": true
}
```
* Во вкладке _Command line_ включить _Enable experimental features_ (в старых версиях, ту же настроку включаем во вкладке _Deamon_, надо включить _Advanced_)


## MySQL внутри docker-контейнера 

```bash
docker run --name mysql -d \
    -e MYSQL_ROOT_PASSWORD=this_is_password \
    -e MYSQL_ROOT_HOST=10.10.5.6 \
    -p 3306:3306 \
    mysql:latest
```

Важно указать действительный ip на котором работает dockers, иначе невозможно будет коннектиться в MySQL внутри контейнера.

```bash
docker inspect mysql
```

Останавливать, удалить контейнер и удалить образ контейнера:
```bash
docker stop mysql
docker rm mysql
docker rmi mysql
```

### Сохраняем данные базы при перезапуске контейнера

Чтобы при перезапуске контенера данные из базы сохранялись, а не уничтожались вместе с контейнером, их нужно хранить во внешнем для контенера каталоге хост-машины, и заставить контейнер монтировать эту внешнюю для него папку внутрь себя. Каталог можно создать эту папку вручную:
```bash
mkdir mysql_data
```

или командой doсker:
```bash
docker volume create --name mysql_data
```

Запуск контейнера с монтированием этой папки выглядит так:
 
 ```bash
docker run --name mysql -d \
   -e MYSQL_ROOT_PASSWORD=this_is_password \
   -e MYSQL_ROOT_HOST=10.10.5.6 \
   -p 3306:3306 \
   -v mysql_data:/var/lib/mysql \
   mysql:latest
```


## MySQL внутри сборки docker-compose

В последних сборках MySQL Server (8.0.20) изменена процедура авторизации и подключение клиентов осуществяетчя через SSL-шифрование (SHA2 или SHA256) Это может затруднить работу старых приложений (или просто лень создавать и обмениваться ключами шифрования). Для того чтобы переключить механизм авторизации в режим предыдущих версий (_native_mysql_password_) в конфигурационный файл MySQL `/etc/mysql/my.cnf`, в блок `[mysqld]` нужно добавить строчки:
```buildoutcfg
bind-address    = 0.0.0.0
skip_ssl        = true
```            
            
Внутри контейнера эти настройки можно менять через yml-файлы для **docker-compose** (возможно можно и без docker-compose, но я не разобрался). Создадим в корне диска с контенерами dockers папку `docker-compose` и создадим в ней файл `docker-compose.yml` со следующим содержанием:
```yaml
version: "2.1"
services:
   database:
      container_name: mysql8_v1
      image: mysql:latest
      volumes:
         # - "d:/docker-compose/mysql/cfg:/etc/mysql"
         - "d:/docker-compose/mysql/data:/var/lib/mysql"
      environment:
         - "MYSQL_ROOT_PASSWORD=this_is_password"
         - "MYSQL_ROOT_HOST=10.10.5.6"
      ports:
         - "3306:3306"
      command: --bind-address="0.0.0.0" --skip_ssl="true"
```

Обратите внимание, данные базы внутри контейнера будут хранится в примонтированном каталоге `d:/docker-compose/mysql/data` (каталог надо создать заранее)

Запуск контейнероной сборки:
```bash
docker-compose up -d
```

ключ `-d` -- detached mode, т.е. запуск в фоновом режиме (терминал хост-машины не получает информацию на вход и не отображает вывод).

Остановка контейнерной сборки:
```bash
docker-compose down
```

