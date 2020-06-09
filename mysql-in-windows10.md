# Как запустить MySQL под Windows 10

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

Дальше:

```
docker run --name mysql -d -e MYSQL_ROOT_PASSWORD=this_is_password -e MYSQL_ROOT_HOST=10.10.5.6 -p 3306:3306 mysql:latest
```

Важно указать действительный ip на котором работает dockers

```bash
docker inspect mysql
```

Останавливать, удалить контейнер и удалить образ контейнера:
```
docker stop mysql
docker rm mysql
docker rmi mysql
```