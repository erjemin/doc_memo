# Как запустить MySQL под Windows 10

Сначала надо перенастроть Dockers в advansed режим и разрешить использовать experimental-контенеры. См. тут: https://stackoverflow.com/questions/48066994/docker-no-matching-manifest-for-windows-amd64-in-the-manifest-list-entries

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