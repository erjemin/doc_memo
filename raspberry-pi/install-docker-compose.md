# Установка Docker и Docker Compose на Raspberry Pi / Orange Pi

Раньше для установки Docker на Raspberry нужно было неплохо так поприседать, но эти времена в прошлом.
Скачиваем скрипт установки `get-docker.sh` и запускаем его:
```shell
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
rm get-docker.sh
```

В процессе будет запрошен пароли для sudo-доступа.

Проверяем, что установка прошла успешно:
```shell
docker version
```

Увидим, что-то типа:
```txt
Client: Docker Engine - Community
 Version:           25.0.0-beta.1
 API version:       1.44
 Go version:        go1.21.3
 Git commit:        2b521e4
 Built:             Mon Nov 13 16:50:14 2023
 OS/Arch:           linux/arm64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          25.0.0-beta.1
  API version:      1.44 (minimum version 1.12)
  Go version:       go1.21.3
  Git commit:       6af7d6e
  Built:            Mon Nov 13 16:50:14 2023
  OS/Arch:          linux/arm64
  Experimental:     false
 containerd:
  Version:          1.6.26
  GitCommit:        3dd1e886e55dd695541fdcd67420c2888645a495
 runc:
  Version:          1.1.10
  GitCommit:        v1.1.10-0-g18a0cb0
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```

Проверяем, что установка Docker Compose тоже прошла успешно:
```shell
docker compose version
```

Увидим, что-то типа:
```txt
Docker Compose version v2.23.0
```
