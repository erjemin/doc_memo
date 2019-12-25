# Как запустить MySQL под Windows 10

Сначала надо перенастроть Dockers в advansed режим и разрешить использовать experimental-контенеры. См. тут: https://stackoverflow.com/questions/48066994/docker-no-matching-manifest-for-windows-amd64-in-the-manifest-list-entries

Дальше:

```
docker run --name mysql -d -e MYSQL_ROOT_PASSWORD=qwaseR12 -e MYSQL_ROOT_HOST=10.10.5.6 -p 3306:3306 mysql:latest
```

важно указать действительный ip на котором работает dockers

Останавливать и удалить контейнер
```
docker stop mysql
docker rm mysql
```