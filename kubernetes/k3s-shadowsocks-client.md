# Создаём под с Shadowsocks

Для каждого VPN-сервера (локации) нужен отдельный клиентский под, который создаст SOCKS5-прокси внутри кластера. 
Другие поды будут подключаться к тому или иному SOCKS5-прокси в зависимости от их назначения.

Все конфиги и манифесты K3S хранит в `etcd` и распространится по всем нодам кластера. Но создавать и вносить изменения
непосредственно в `etcd` не удобно. Намного удобнее k3s-конфиги передавать через ConfigMap. К тому же это позволяет
иметь копии конфигов на каком-нибудь хосте и делать резервные копии и восстанавливать их в случае необходимости.

Не принципиально, где хранить конфиги и манифесты, так как после с помощью `kubectl` они будут загружены в k3s. Но
лучше хранить их в одном месте, чтобы не искать по всему кластеру, где же они хранятся.

Предлагаемая структура каталогов для хранения конфигураций и манифестов Kubernetes:
```text
~/k3s/
├── vpn/                       # Все VPN-клиенты
│   ├── client-shadowsocks--moscow/       # Локация Москва
│   │   ├── config.yaml                     # ConfigMap для Shadowsocks
│   │   └── deployment.yaml                 # Deployment для Shadowsocks
│   ├── client-shadowsocks--stockholm/    # Локация Стокгольм
│   │   ├── config.yaml
│   │   └── deployment.yaml
│   └── cclient-shadowsocks--izmir/       # Локация Измир
│       ├── config.yaml
│       └── deployment.yaml
├── ...
└── ...
```

Создаем файл `config.yaml` для первого Shadowsocks-клиента (Москва):
```bash
nano ~/k3s/vpn/client-shadowsocks--moscow/config.yaml
```

И вставляем в него следующее:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: shadowsocks-client-moscow
  namespace: kube-system    # Ставим в kube-system, чтобы было системно
data:
  config.json: |
    {
      "server": "<IP_ИЛИ_ИМЯ_СЕРВЕРА>",
      "server_port": <ПОРТ>,
      "local_address": "127.0.0.1",
      "local_port": 1081,
      "password": "<PASSWORD_FOR_SHADOWSOCKS_CLIENT>",
      "method": "chacha20-ietf-poly1305",
      "mode": "tcp_and_udp"
    }
```

Что тут происходит:
- `apiVersion: v1` — версия API Kubernetes.
- `kind: ConfigMap` — это способ хранить конфиги внутри k3s.
- `metadata:` — метаданные о конфиге.
  - `name:` — имя конфигурации.
  - `namespace:` — пространство имен, в котором будет храниться конфигурация. Мы используем `kube-system`, чтобы сделать его системным.
- `data:` — данные конфигурации.
  - `config.json:` — имя файла, в который будет записан конфиг.
  - `|` — говорит, что дальше будет многострочный текст.
  - `{...}` — Собственно JSON-конфигурация нашего Shadowsocks-клиента.
    - `server` и `server_port` — адрес и порт нашего VPS.
    - `local_address` и `local_port` — где будет SOCKS5 внутри кластера.
    - `password` и `method` — пароль и метод шифрования. Метод шифрования `chacha20-ietf-poly1305` -- используется,
      например, VPN-сервисом Outline. Получить пароль для Outline можно с помощью base64 декодирования ключа.
      Структура строки подключения `ss://<ПАРОЛЬ_КОДИРОВАННЫЙ_В_BASE64>@<IP_ИЛИ_ИМЯ_СЕРВЕРА>:<ПОРТ>?type=tcp#<ИМЯ-КЛИЕНТА>`
    - `mode: tcp_and_udp` — включает поддержку TCP и UDP.

Применим ConfigMap:
```bash
sudo k3s kubectl apply -f /home/<ПОЛЬЗОВАТЕЛЬ>/k3s/vpn/client-shadowsocks--moscow/config.yaml
```

Важно указывать полный путь к файлу, а не от домашнего каталога `~\`. Запуская `kubectl` из под `sudo` (или от имени
`root`), мы исполняем команды `k3s` от имени другого пользователя, а не от имени текущего.

Когда выполним команду, то увидим что-то вроде:
```text
configmap/shadowsocks-config-moscow created
```

Теперь создадим `Deployment` для Shadowsocks-клиента. Создаём файл `deployment.yaml`:
```bash
nano ~/k3s/vpn/client-shadowsocks--moscow/deployment.yaml
```

И вставляем в него следующее:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shadowsocks-client-moscow     # Уникальное имя (должно совпадать с именем в config.yaml для ConfigMap)
  namespace: kube-system              # В системном пространстве
spec:
  replicas: 1
  selector:
    matchLabels:
      app: shadowsocks-client-moscow
  template:
    metadata:
      labels:
        app: shadowsocks-client-moscow
    spec:
      containers:
      - name: shadowsocks-client
        image: shadowsocks/shadowsocks-libev:latest   # Официальный образ
        command: ["ss-local"]                         # Запускаем клиент
        args:
          - "-c"                                      # Указываем конфиг
          - "/etc/shadowsocks/config.json"            # Путь внутри контейнер
        volumeMounts:
        - name: config-volume
          mountPath: /etc/shadowsocks                 # Монтируем ConfigMap
        ports:
        - containerPort: 1081                         # Открываем порт SOCKS5 (TCP)
          protocol: TCP
        - containerPort: 1081                         # Открываем порт SOCKS5 (UDP)
          protocol: UDP
        securityContext:
          privileged: true                            # Нужно для работы с сетью
      volumes:
      - name: config-volume
        configMap:
          name: shadowsocks-client-moscow             # Связываем с ConfigMap
```

Объяснение:
* `Pod` — это простейший объект в k3s, запускающий один контейнер.
* `image` — официальный образ Shadowsocks.
* `command` и `args` — запускают `ss-local` с конфигом из `ConfigMap`.
* `volumeMounts` — подключают `config.json` из `ConfigMap` в контейнер.
* `ports` — открываем 1080/TCP и 1080/UDP для SOCKS5.
* `privileged: true` — даёт права для работы с сетью (в k3s это иногда нужно).

Применим под:
```bash
sudo k3s kubectl apply -f /home/opi/k3s/vpn/client-shadowsocks--moscow/deployment.yaml
```

### Проверка

Проверяем, что под запустился, посмотрев статус:
```
sudo k3s kubectl get pods -n kube-system
```

Увидим что-то типа:
```text
NAME                                            READY   STATUS              RESTARTS         AGE
...
...
shadowsocks-client-moscow-54d64bf5f4-trb6p      1/1     Running             0                24m
...
```

Можно проверь логи:
```bash
sudo k3s kubectl logs -n kube-system shadowsocks-client-stockholm-54d64bf5f4-trb6p
```

Увидим, что клиент shadowsocks запустился:
```text
 2025-03-09 09:48:24 INFO: initializing ciphers... chacha20-ietf-poly1305
 2025-03-09 09:48:24 INFO: listening at 127.0.0.1:1081
 2025-03-09 09:48:24 INFO: udprelay enabled
```

Запустился, но не подключился. Подключение произойдет при отправке первых пакетов через соединение. Для этого нужно
зайти в под и запросить что-нибудь через `curl`. Но на поде нет `curl`, поэтому что по умолчанию образ контейнера
shadowsocks-клиента минималистичен и в нём нет ничего лишнего. Нам придется собрать свой образ с `curl`. Создадим
файл `Dockerfile` для сборки образа (да, сам Kubernetes не умеет собирать образы, для этого нужен Docker):
```bash
nano k3s/vpn/client-shadowsocks--moscow/Dockerfile
```

И вставим в него следующее:
```dockerfile
FROM shadowsocks/shadowsocks-libev:latest
USER root
RUN apk update && apk add curl netcat-openbsd
```

Что тут происходит:
* `FROM` — базовый образ, от которого мы будем отталкиваться.
* `USER root` — переключаемся на пользователя root, чтобы иметь возможность устанавливать пакеты.
* `RUN` — выполнить команду в контейнере. В данном случае обновляем пакеты (`apk update`) и устанавливаем `curl` и
  `netcat-openbsd` (`apk add curl netcat-openbsd`).

Cоберём образ:
```bash
sudo docker build -t shadowsocks-with-tools:latest ~/k3s/vpn/client-shadowsocks--moscow/
```

Увидим, что образ собрался:
```text
[+] Building 1.4s (6/6) FINISHED                                                                                                             docker:default
 => [internal] load build definition from Dockerfile                                                                                                   0.0s
 => => transferring dockerfile: 135B                                                                                                                   0.0s
 => [internal] load metadata for docker.io/shadowsocks/shadowsocks-libev:latest                                                                        1.4s
 => [internal] load .dockerignore                                                                                                                      0.0s
 => => transferring context: 2B                                                                                                                        0.0s
 => [1/2] FROM docker.io/shadowsocks/shadowsocks-libev:latest@sha256:124d1bff89bf9e6be19d3843fdcd40c5f26524a7931c8accc5560a88d0a42374                  0.0s
 => CACHED [2/2] RUN apk update && apk add curl netcat-openbsd                                                                                         0.0s
 => exporting to image                                                                                                                                 0.0s
 => => exporting layers                                                                                                                                0.0s
 => => writing image sha256:5708432467bcac4a0015cd97dbca968e9b69af06da192018169fff18673ed13f                                                           0.0s
 => => naming to docker.io/library/shadowsocks-with-tools:latest
 ```

Перенесем полученный образ в k3s с помощью `ctr` (containerd CLI):
```bash
sudo docker save shadowsocks-with-tools:latest | sudo k3s ctr images import -
```

Здесь:
* `docker save` — экспортирует образ в tar-формат.
* `k3s ctr` — вызывает ctr внутри k3s.
* `images import -`  — импортирует образ из stdin.

Увидим что-то вроде:
```text
unpacking docker.io/library/shadowsocks-with-tools:latest (sha256:ae615618ce9d2aac7d3764ef735108452adf3fc30bb65f23f28c345798880c80)...done
```

Проверим, что образ появился в k3s:
```bash
sudo k3s ctr images ls | grep shadowsocks
```

Увидим что-то типа:
```text
...
docker.io/library/shadowsocks-with-tools:latest     application/vnd.oci.image.manifest.v1+json      sha256:...  22.5 MiB  linux/arm64           io.cri-containerd.image=managed
...
...                                 
```

Теперь нам нужно передать образ контейнера на другие ноды кластера. Как это сделать есть заметка "[Развертывание
пользовательского контейнера в k3s](k3s-custom-container-deployment.md)"


Когда наш контейнер окажется на всех нодах, изменим `deployment.yaml` Shadowsocks-клиента, чтобы использовать наш
новый образ. Закомментируем строку `image: shadowsocks/shadowsocks-libev:latest` и вставим две строки после неё
(обратите внимание на заметки): 
```yaml
...
    spec:
      containers:
      - name: shadowsocks-client
        # image: shadowsocks/shadowsocks-libev:latest
        image: shadowsocks-with-tools  # Без :latest, чтобы k3s не "ходил" за контейнером в реестр (например, DockerHub)
        imagePullPolicy: Never         # Только локальный образ, не тянуть из реестра
        ...
...
```

Уберём старый под из deployment и удалим сам под из k3s:
```bash
sudo k3s kubectl delete deployment -n kube-system shadowsocks-client-stockholm
sudo k3s kubectl delete pod -n kube-system -l app=shadowsocks-client-stockholm --force --grace-period=0
```

Запустим новый под с нашим новым образом:
```bash
sudo k3s kubectl apply -f ~/k3s/vpn/client-shadowsocks--stockholm/deployment.yaml
```

Проверим, что под запустился, посмотрев статус:
```bash
sudo k3s kubectl get pods -n kube-system
```

Увидим что-то типа:
```text
NAME                                            READY   STATUS              RESTARTS       AGE
...
shadowsocks-client-moscow-6cf7b956b8-mtsg4      1/1     Running             0              9s
...
```

#### Проверка работы Shadowsocks

Посмотрим логи пода с Shadowsocks-клиентом:
```bash
sudo k3s kubectl logs -n kube-system -l app=shadowsocks-client-moscow
```

Увидим, что клиент shadowsocks запустился:
```text
 2025-03-14 21:01:59 INFO: initializing ciphers... chacha20-ietf-poly1305
 2025-03-14 21:01:59 INFO: listening at 127.0.0.1:1081
 2025-03-14 21:01:59 INFO: udprelay enabled
 2025-03-14 21:01:59 INFO: running from root user
```

Проверим TCP-соединение. Зайдём в под:
```bash
sudo k3s kubectl exec -it -n kube-system shadowsocks-client-moscow-<hash> -- sh
```

И выполним внутри пода команду:
```bash
curl --socks5 127.0.0.1:1081 http://ifconfig.me
curl -k --socks5 127.0.0.1:1081 https://ifconfig.me
```

`ifconfig.me` -- это публичный сервис, который показывает IP-адрес, с которого к нему пришёл запрос. В первом случае
проверяем http-соединение, а во втором — https. Ожидаемый результат: `<VPS_IP>` (IP-адрес нашего VPS).

Выходим из пода:
```bash
exit
```

Проверим логи еще раз:
```bash
sudo k3s kubectl logs -n kube-system shadowsocks-client-moscow-<hash>
```

Увидим, что клиент shadowsocks отработал:
```text
 2025-03-14 21:01:59 INFO: running from root user
 2025-03-14 21:03:01 INFO: connection from 127.0.0.1:55226
 2025-03-14 21:03:01 INFO: connect to 34.160.111.145:80
 2025-03-14 21:03:01 INFO: remote: <VPS_IP>:56553
 2025-03-14 21:03:10 INFO: connection from 127.0.0.1:33382
 2025-03-14 21:03:10 INFO: connect to 34.160.111.145:443
 2025-03-14 21:03:10 INFO: remote: <VPS_IP>:56553
 ```

## Изменение конфигурации

Кстати, если нам понадобится внести изменения в конфиг, то можно просто отредактировать файл и применить его снова.
Старые данные автоматически заменятся на новые. "Умная" команда `kubectl apply` сравнивает текущий объект в k3s
(в `etcd`) с тем, что указан в файле. Если объект уже существует (по `metadata.name` и `namespace`), он обновляется.
Если объекта нет, он создаётся.

При применении изменений конфигурации увидим что они применились:
```text
configmap/shadowsocks-config-moscow configured
```

Если же мы захотим удалить конфиг, то выполним (в нашем случае мы удаляем конфиг `metadata:name` для
`shadowsocks-config-moscow`:
```bash
sudo k3s kubectl delete configmap shadowsocks-config-moscow -n kube-system
```

Как изменения дойдут до пода когда `ConfigMap` обновился? Никак! Под (например, наш shadowsocks-client-moscow)
уже запущен с предыдущими настройками, находится в памяти, работает и никак сам не обновится, пока мы его не
перезапустим.

Один из вариантов обновления пода — это удалить его, и `Deployment` сам создаст и поднимет новый под с обновлённым
`ConfigMap`. Удаляем под:
```bash
sudo k3s kubectl delete pod -n kube-system -l app=shadowsocks-client-moscow
```
где `-l` — это фильтр по меткам. В данном случае мы удаляем под с меткой `app=shadowsocks-client-moscow`.

Проверяем:
```bash
sudo k3s kubectl get pods -n kube-system
```
