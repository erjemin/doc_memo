# PostgeSQL в K3s

Самый простой способ установить PostgreSQL в k3s -- просто создать один под со своим PVC и ConfigMap для
конфигурирования. Достаточно для тестов и небольших приложений, и доступность будет обеспечена на приемлемом уровне.


## Доступность порта 5432 (не обязательно)

Мне для разработки и тестирования на моем рабочем компе (вне k3s) нужно, чтобы PostgreSQL был доступен снаружи кластера
(внутри домашней сети). Это можно сделать двумя способами:
1. Через `NodePort`: сервиса Kubernetes, который открывает указанный порт на всех узлах (нодах) кластера. Порты, 
   обычно, находятся в диапазоне 30000–32767, и NodePort проксирует на нужный порт внутри кластера, таким
   образом, направляя запросы к нужным подам. Это решение удобно, так как довольно просто в настройке
   и не требует Traefik, но порт будет открыт на всех нодах и будет потеряно централизованное управление
   маршрутизацией, которое даёт Traefik (например, HostSNI или TLS termination) и, возможно, потребуется внешний
   балансировщик нагрузки, если нод несколько.
2. Через `IngressRouteTCP` в Traefik: это более гибкий способ, который позволяет использовать маршрутизации трафика,
   включая поддержку HostSNI (например с доступом через домен с сертификатами (TLS termination). 
   Маршрутизацией на порта внутри кластер в нужный контейнер обеспечивает Traefik.

В продакшене делать порт доступным вне кластера вообще не обязательно. Внутри кластера порт всегда доступен
в контейнере PostgreSQL, и любые приложения илди сервисы кластера смогут обращаться к нему по имени
(`имя_контейнера:порт`).

Мне, для моих задач, больше подходит второй вариант (с `IngressRouteTCP`), так как я уже использую Traefik
для маршрутизации. Но для того чтобы "вывесить порт" наружу кластера, нужно изменить статические манифесты,
и настроить Traefik для прослушивания и маршрутизации дополнительных портов. Нам понадобится `Helm Chart` 
(для конфигурации Traefik), и добавления нового `entryPoint` для порта 5432.

У меня уже есть манифест `Helm Chart` (Helm -- это такой пакетный менеджер для Kubernetes), в котором [настраивал 
доступ и открывал порт 2222 для SSH внутрь контейнер Gitea](k3s-migrating-container-from-docker-to-kubernetes.md).
Добавлю туда и 5432-порт для PostgreSQL:
```yaml
# ~/k3s/traefik/traefik.yaml 
# "Статичные" манифесты, которые применяются при старте Traefik 

# Helm-манифест: установка нестандартных праметров Treafik
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    additionalArguments:
      # ...
      # ...
      - --entrypoints.postgres.address=:5432    # Добавляем entryPoint для PostgreSQL (5432 порт)

    ports:
      # ...
      # ...
      postgres:
        port: 5432              # Внутренний порт в контейнере Traefik
        expose:
          default: true         # Доступен по умолчанию
        exposedPort: 5432       # Порт на хосте (включая VIP с Keepalived)
        protocol: TCP           # Это TCP трафик

---
...
...
```

После применения:
```shell
kubectl apply -f ~/k3s/traefik/traefik.yaml
```

Traefik перезапустится и настройки вступят в силу.


## Создание манифестов для PostgreSQL

Теперь создадим манифесты для PostgreSQL. Они будут включать:
1. Пространство имён `postgresql` (если его нет).
2. PersistentVolumeClaim (PVC) для хранения данных PostgreSQL.
3. ConfigMap (их будет несколько) для кастомизации настроек PostgreSQL.
4. Deployment для PostgreSQL с монтированием PVC и ConfigMap-ов.
5. Секрет для хранения пароля PostgreSQL (тоже может быть несколько, если нужно).
6. Service для внутреннего доступа к PostgreSQL.
7. IngressRouteTCP для внешнего доступа к PostgreSQL по порту 5432.

```yaml
# ~/k3s/postgresql/postgresql.yaml
# Все манифесты для PostgreSQL (без наворотов вроде репликаций, высокой доступности через Patroni и т.д.)

# 1. Манифест создания пространства имён `postgresql`. Если оно уже есть — kubectl apply ничего не изменит
apiVersion: v1
kind: Namespace
metadata:
  name: postgresql      # Имя пространства имён для PostgreSQL


---
# 2. Манифест PVC (Longhorn) -- том в блочном хранилище с данными (базами) PostgreSQL
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data       # Имя PVC-хранилища
  namespace: postgresql     # в пространстве имен `postgresql`
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn    # Используем Longhorn как класс хранения
  resources:
    requests:
      storage: 2Gi   # Запрашиваемое хранилище 2 Гб (позже можно изменить на большее)


---
# 3. Манифесты ConfigMap конфига ... 
# 3.1. Этот ConfigMap будет подключаться в `/etc/postgresql/postgresql.conf` в контейнере PostgreSQL.
#      При инициализации, после initdb, создается дефолтный `postgresql.conf` внутри $PGDATA,
#      каталоге с данными (в нашем случае в `/var/lib/postgresql/db/postgresql.conf`). Если
#      смонтировать `postgresql.conf` сразу в каталог $PGDATA, инициализация не произойдет,
#      так как при инициализации требуется именно *ЧИСТЫЙ* каталог. И, вот прям неожиданно,
#      в этом конфиге по умолчанию нет `include_dir = ...`, а значит, никакие кастомные
#      конфиги из каталога `/etc/postgresql/conf.d` не будут подключаться, и придется для
#      изменения настроек ВСЕГДА править `postgresql.conf` вручную после инициализации, уже
#      внутри пода (или командами внутри `psql`), что не очень удобно (1) и не оставляет следов, 
#      так как кто все эти изменения будет документировать (2)?
#      И так, ConfigMap `postgresql-init-config` монтируется в `/etc/postgresql/postgresql.conf`,
#      а затем через `args` и `POSTGRES_INITDB_ARGS` подцепляем его в хвост дефолтного конфига. 
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgresql-init-config
  namespace: postgresql
data:
  # ВАЖНО! Используйте одинарные кавычки для строковых значений. Двойные кавычки приводят к ошибкам!
  postgresql.conf: |-
    listen_addresses = '*'
    dynamic_shared_memory_type = posix
    max_wal_size = 1GB
    min_wal_size = 80MB
    datestyle = 'iso, mdy'
    lc_messages = 'en_US.utf8'
    lc_monetary = 'en_US.utf8'
    lc_numeric = 'en_US.utf8'
    lc_time = 'en_US.utf8'
    default_text_search_config = 'pg_catalog.russian'
    include_dir = '/etc/postgresql/conf.d'

---
# 3.2. Манифесты ConfigMap для кастомизации (или один ConfigMap с несколькими ключами) будет 
#      смонтирован в контейнер PostgreSQL, в каталог `/etc/postgresql/conf.d`. Конфиги в этом 
#      каталоге будут автоматически "подхвачены" при запуске и могут добавлять и переопределять
#      настройки. Они применяются последовательно (в алфавитно-литеральном порядке) и потому
#      имя (или ключ) монтирования важен, так как него зависит порядок применения кастомизации
#      (конфиг O1-xxx будет применён раньше, чем O2-xxx и т.д.).
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-custom-config-dir
  namespace: postgresql     # в пространстве имен `postgresql`
data:
  # Пример кастомного конфига PostgreSQL
  # ВАЖНО! Используйте одинарные кавычки для строковых значений. Двойные кавычки приводят к ошибкам!
  01-custom.conf: |-
    max_connections = 120
    shared_buffers = 256MB
    log_min_duration_statement = 1000
    work_mem = 6MB
  # Еще один кастомный конфиг (устанавливает часовой пояс) 
  # ВАЖНО! Используйте одинарные кавычки для строковых значений. Двойные кавычки приводят к ошибкам!
  02-timezone.conf: |-
    log_timezone = 'Europe/Moscow'
    timezone = 'Europe/Moscow'


---
# 4. Deployment: PostgreSQL
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: postgresql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16    # Используем официальный образ PostgreSQL версии 16
          ports:
            - containerPort: 5432   # Порт PostgreSQL внутри контейнера
          # Дополнительный аргумент... он подключает наш `postgresql.conf` (из ConfigMap), смонтированный
          #  в контейнер в каталог `/etc/postgresql/` (через ConfigMap) в хвост конфига по умолчанию.
          args: ["-c", "config_file=/etc/postgresql/postgresql.conf"]
          env:
            - name: POSTGRES_INITDB_ARGS    # То же самое, что и выше...
              value: "--set=config_file=/etc/postgresql/postgresql.conf"
            - name: POSTGRES_PASSWORD   # По умолчанию postgres создаёт пользователя `postgres` c 
              # паролем, который можно задать через переменную окружения POSTGRES_PASSWORD.
              # Если переменную с паролем не указать, то контейнер не стартует и завершит работу с ошибкой.
              # Так что POSTGRES_PASSWORD -- обязательная переменная.
              valueFrom:
                secretKeyRef:             # Получаем пароль из секрета:
                  name: postgres-secret     # Имя секрета
                  key: password             # Ключ в секрете, содержащий пароль
            # - name: POSTGRES_USER       # В принципе, имя пользователя по умолчанию тоже можно изменить
            #   value: postgres
            - name: POSTGRES_DB           # Имя базы данных, которая будет создана при запуске (по умолчанию: `postgres`)
              value: postgres
            - name: PGDATA                # Каталог, где PostgreSQL будет хранить свои данные
              value: /var/lib/postgresql/db
            - name: POSTGRES_CONFIG_DIR   # Каталог, где PostgreSQL будет искать кастомные конфигурационные файлы
              value: /etc/postgresql/conf.d # ...и туда мы будем монтирвовать конфиги через ConfigMap
            - name: TZ                    # Часовой пояс, который будет в поде
              value: Europe/Moscow
          volumeMounts:       # Монтируем тома:
            - name: data                                        # Данные PostgreSQL в Longhorn (PVC)
              mountPath: /var/lib/postgresql/                   # в каталог внутри контейнера (он не совпадает с PGDATA, т.к.
                                                                #    PVC Longhorn всегда создает каталог `lost+found` в новом томе,
                                                                #    а PostgreSQL _нужен_ пустой каталог для инициализации)
            - name: config
              mountPath: /etc/postgresql/postgresql.conf
              subPath: postgresql.conf
              # mountPath: /var/lib/postgresql/config
            #- name: config
            #  mountPath: /docker-entrypoint-initdb.d/postgresql.conf
            #  subPath: postgresql.conf
            - name: custom-configs                           # Кастомные конифиги PostgreSQL из ConfigMap
              mountPath: /etc/postgresql/conf.d                 # в каталог внутри контейнера
          # Проверка "живости" контейнера (Liveness Probe). Kubernetes будет выполнять периодически команду 'command'.
          # Если она начнет возвращать ошибку (например, если PostgreSQL "завис"), то kubelet перезапустит pod.
          livenessProbe:
            exec:
              command: ["pg_isready", "-U", "postgres"]  # Проверяем доступность PostgreSQL для пользователя 'postgres'
            initialDelaySeconds: 15     # Подождать 15 секунд после запуска контейнера перед первой проверкой
            periodSeconds: 25           # Повторять проверку каждые 25 секунд
            timeoutSeconds: 3           # Считаем ошибкой, если команда не ответила за 3 секунды
            failureThreshold: 5         # Подряд 5 неудач — и pod будет перезапущен
          # Проверка "готовности" контейнера (Readiness Probe). Kubernetes использует `command`, чтобы понять: можно ли
          # направлять трафик в этот pod. Если проверка не прошла -> pod "не готов" -> не получает запросы от сервиса.  
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "postgres"]
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
      volumes:            # Используемые тома:
        - name: data        # Том PVC (Longhorn) для хранения данных
          persistentVolumeClaim:
            claimName: postgres-data
        - name: config
          configMap:
            name: postgresql-init-config
        - name: custom-configs          # Конфиги (ключи) из ConfigMap для кастомизации PostgreSQL
          configMap:
            name: postgres-custom-config-dir

---
# 5. Секреты PostgreSQL
#   Следует отметить, что секреты в Kubernetes хранятся в виде base64-строк, и это не шифрование, а просто кодирование.
#   Так что то, что он указан в манифесте в открытом виде, не увеличит уязвимость (все равно все можно получить из
#   команды `kubectl get secret postgres-secret -o yaml`).
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret     # имя секрета
  namespace: postgresql     # в пространстве имен `postgresql`
type: Opaque              # Тип данных секрета 
stringData:               # Используем stringData для удобства
  password: xXxXxXxX...   #  ключ: значение (тут укажем пароль)

---
# 6. Service: внутренний доступ к контейнеру PostgreSQL
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: postgresql
spec:
  selector:
    app: postgres
  ports:
    - name: pg
      protocol: TCP
      port: 5432
      targetPort: 5432

---
# 7. IngressRouteTCP: внешний доступ на pg.local (через порт 5432, а доменное имя, чтоб не указывать VIP-адрес)
apiVersion: traefik.io/v1alpha1
kind: IngressRouteTCP
metadata:
  name: postgres
  namespace: postgresql
spec:
  entryPoints:
    - postgres  # используем entryPoint, который мы добавили в HelmChart для Traefik (5432 порт)
  routes:
    - match: HostSNI("*")    # доменное имя, по которому будет доступен сервис
      services:
        - name: postgres            # через имя сервиса обращаемся к контейнеру PostgreSQL
          port: 5432                # порт внутри контейнера, на который будет направлен трафик
```

Все "тонкие моменты", а их не мало, разъяснены в комментариях.

Еще интересного, чего нет в комментариях: В `IngressRouteTCP` пришлось указать `HostSNI("*")` (без имени
домена). Это потому, что `HostSNI()` в `IngressRouteTCP` работает только при TLS (SSL) соединении. SNI (Server
Name Indication) — это часть _TLS handshake_. Когда клиент подключается по TCP без TLS, не передаёт никакое
имя хоста — просто IP и порт. Соответственно, Traefik не может сопоставить имя pg.local с каким-либо маршрутом,
потому что имя не приходит. Именно по этому, если нет TLS, то инструкция `HostSNI("pg.local")` не сработает,
а `HostSNI("*")` просто матчит все входящие TCP без проверки имени. В принципе, это нормально для домашней
сети, ведь порт 5432 не виден из внешнего в интернет. Но в "злом" продакшн или для критических данных между
базой PostgreSQL и приложением лучше использовать TLS, чтобы защитить данные от перехвата; и тогда понадобится
TLS; и тогда можно будет использовать `HostSNI("pg.local")` для маршрутизации.

Примерим манифесты:
```shell
kubectl apply -f ~/k3s/postgresql/postgresql.yaml
```

Посмотреть пароль к базе данных можно так (когда секрет уже создан):
```shell
kubectl get secret -n postgresql postgres-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

Посмотреть логи развертывания так:
```shell
kubectl logs -n postgresql deployment/postgres
```

Покажут много интересного, и все настройки будут "по умолчанию". Настройки нашего, примонтированного и подключенного
через `config_file = ...` конфига, не покажут! 

В конце будет:
```text
PostgreSQL init process complete; ready for start up.

2025-05-25 08:00:10.025 UTC [1] LOG:  starting PostgreSQL 16.9 (Debian 16.9-1.pgdg120+1) on aarch64-unknown-linux-gnu, compiled by gcc (Debian 12.2.0-14) 12.2.0, 64-bit
2025-05-25 08:00:10.026 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2025-05-25 08:00:10.026 UTC [1] LOG:  listening on IPv6 address "::", port 5432
2025-05-25 08:00:10.040 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2025-05-25 08:00:10.069 UTC [63] LOG:  database system was shut down at 2025-05-25 08:00:09 UTC
2025-05-25 08:00:10.095 UTC [1] LOG:  database system is ready to accept connections
```

Все. Можно подключаться к базе данных.


## Проверки

Посмотреть статус пода PostgreSQL:
```shell
kubectl get pods -n postgresql -o wide
```

Увидим что-то вроде:
```text
NAME                        READY   STATUS    RESTARTS   AGE   IP           NODE   NOMINATED NODE   READINESS GATES
postgres-xxxxxxxxxx-xxxxx   1/1     Running   0          46m   10.42.0.44   opi5   <none>           <none>
```

Зайти внутрь пода (взять имя пода можно из вывода предыдущей команды):
```shell
kubectl exec -it -n postgresql postgres-xxxxxxxxxx-xxxxx  -- /bin/bash
```

Если что-то не развернулось, то удалять в следующей последовательности (сначала deployment, потом PVC, потом под,
хотя он сам удалится при удалении deployment):

```shell
kubectl delete deployment postgres -n postgresql
kubectl delete pvc postgres-data -n postgresql
kubectl delete pod -l app=postgres -n postgresql
```


## Настройки

Теперь, благодаря нашим ConfiMap с кастомными конфигами, можно легко менять настройки PostgreSQL. Только не забывать
после изменения конфигов, делать им `kubectl apply ...`, что перезапустит под PostgreSQL, и изменения вступят
в силу. Еще можно через `kubectl rollout restart deployment postgres -n postgresql`, но так никакие изменения
в ConfigMap не будут применены, только изменения в самом Deployment.

Еще из необходимого, что пока нельзя сделать через манифесты, то через веб-интерфейс Longhorn установить для тома
с базой `Data Locality` в `best-effort`, чтобы Longhorn держал одну из реплик тома на ноде, где запущен контейнер.
Это ускорить работу PostgreSQL, так как не будет сетевых задержек при обращении к данным.


## Подключения

Для подключения к базе данных PostgreSQL можно использовать любой клиент или коннектор, поддерживающий PostgreSQL.
А для работы из командной строки с внешнего хоста (и использование `psql`) установить клиент PostgreSQL.

### Для MacOS

Через `Homebrew`:
```shell
brew install libpq
```

Можно после добавить `libpq` в `PATH`, или использовать полный путь `/opt/homebrew/opt/libpq/bin/psql`, например у меня,
через хост `pg.local` (доменное имя, которое указывает на VIP-адрес с Keepalived):
```shell
/opt/homebrew/opt/libpq/bin/psql -h pg.local -U postgres -d postgres
```

или сразу с указанием пароля:
```shell
PGPASSWORD=xXxXxXxXx /opt/homebrew/opt/libpq/bin/psql -h pg.local -U postgres -d postgres
```

### Для Linux

Для Debian/Ubuntu:
```shell
sudo apt install postgresql-client
```

### Для Windows

Не знаю, наверное можно использовать [PostgreSQL для Windows](https://www.postgresql.org/download/windows/).



## Конфиг PostgreSQL по умолчанию (на всякий случай, и на русском)


```apacheconf
# -----------------------------
# Конфигурационный файл PostgreSQL
# -----------------------------

# Этот файл состоит из строк вида:
#
#   name = value
#
# (Знак "=" необязателен.) Можно использовать пробелы. Комментарии начинаются с
# "#" в любом месте строки. Полный список имен параметров и допустимых значений
# можно найти в документации PostgreSQL.
#
# Закомментированные настройки в этом файле показывают значения по умолчанию.
# Повторное закомментирование настройки НЕ возвращает её к значению по умолчанию;
# нужно перезагрузить сервер.
#
# Этот файл читается при запуске сервера и при получении сервером сигнала SIGHUP.
# Если вы редактируете файл на работающей системе, нужно отправить SIGHUP серверу,
# выполнить "pg_ctl reload" или запустить "SELECT pg_reload_conf()". Некоторые
# параметры, отмеченные ниже, требуют полной остановки и перезапуска сервера для
# вступления в силу.
#
# Любой параметр можно также передать через командную строку сервера, например,
# "postgres -c log_connections=on". Некоторые параметры можно менять во время
# работы с помощью SQL-команды "SET".
#
# Единицы измерения памяти:  B  = байты         Единицы времени:  us  = микросекунды
#                            kB = килобайты                       ms  = миллисекунды
#                            MB = мегабайты                       s   = секунды
#                            GB = гигабайты                       min = минуты
#                            TB = терабайты                       h   = часы
#                                                                 d   = дни


#------------------------------------------------------------------------------
# РАСПОЛОЖЕНИЕ ФАЙЛОВ
#------------------------------------------------------------------------------

# Значения по умолчанию для этих переменных задаются через опцию командной строки -D
# или переменную окружения PGDATA, здесь обозначенную как ConfigDir.

#data_directory = 'ConfigDir'           # использовать данные в другом каталоге
                                        # (изменение требует перезапуска)
#hba_file = 'ConfigDir/pg_hba.conf'     # файл аутентификации на основе хоста
                                        # (изменение требует перезапуска)
#ident_file = 'ConfigDir/pg_ident.conf' # файл конфигурации ident
                                        # (изменение требует перезапуска)

# Если external_pid_file явно не задан, дополнительный PID-файл не создаётся.
#external_pid_file = ''                 # записать дополнительный PID-файл
                                        # (изменение требует перезапуска)


#------------------------------------------------------------------------------
# ПОДКЛЮЧЕНИЯ И АУТЕНТИФИКАЦИЯ
#------------------------------------------------------------------------------

# - Настройки подключений -

listen_addresses = '*'
                                        # список адресов через запятую;
                                        # по умолчанию 'localhost'; '*' для всех
                                        # (изменение требует перезапуска)
#port = 5432                            # (изменение требует перезапуска)
max_connections = 100                   # (изменение требует перезапуска)
#reserved_connections = 0               # (изменение требует перезапуска)
#superuser_reserved_connections = 3     # (изменение требует перезапуска)
#unix_socket_directories = '/var/run/postgresql' # список каталогов через запятую
                                        # (изменение требует перезапуска)
#unix_socket_group = ''                 # (изменение требует перезапуска)
#unix_socket_permissions = 0777         # начинать с 0 для восьмеричной нотации
                                        # (изменение требует перезапуска)
#bonjour = off                          # анонсировать сервер через Bonjour
                                        # (изменение требует перезапуска)
#bonjour_name = ''                      # по умолчанию имя компьютера
                                        # (изменение требует перезапуска)

# - Настройки TCP -
# подробности в "man tcp"

#tcp_keepalives_idle = 0                # TCP_KEEPIDLE, в секундах;
                                        # 0 выбирает системное значение по умолчанию
#tcp_keepalives_interval = 0            # TCP_KEEPINTVL, в секундах;
                                        # 0 выбирает системное значение по умолчанию
#tcp_keepalives_count = 0               # TCP_KEEPCNT;
                                        # 0 выбирает системное значение по умолчанию
#tcp_user_timeout = 0                   # TCP_USER_TIMEOUT, в миллисекундах;
                                        # 0 выбирает системное значение по умолчанию

#client_connection_check_interval = 0   # интервал между проверками отключения клиента
                                        # во время выполнения запросов;
                                        # 0 для отключения проверки

# - Аутентификация -

#authentication_timeout = 1min          # от 1 сек до 600 сек
#password_encryption = scram-sha-256    # scram-sha-256 или md5
#scram_iterations = 4096
#db_user_namespace = off

# GSSAPI с использованием Kerberos
#krb_server_keyfile = 'FILE:${sysconfdir}/krb5.keytab'
#krb_caseins_users = off
#gss_accept_delegation = off

# - SSL -

#ssl = off
#ssl_ca_file = ''
#ssl_cert_file = 'server.crt'
#ssl_crl_file = ''
#ssl_crl_dir = ''
#ssl_key_file = 'server.key'
#ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL' # разрешённые SSL-шифры
#ssl_prefer_server_ciphers = on
#ssl_ecdh_curve = 'prime256v1'
#ssl_min_protocol_version = 'TLSv1.2'
#ssl_max_protocol_version = ''
#ssl_dh_params_file = ''
#ssl_passphrase_command = ''
#ssl_passphrase_command_supports_reload = off


#------------------------------------------------------------------------------
# ИСПОЛЬЗОВАНИЕ РЕСУРСОВ (кроме WAL)
#------------------------------------------------------------------------------

# - Память -

shared_buffers = 128MB                  # минимум 128 кБ
                                        # (изменение требует перезапуска)
#huge_pages = try                       # on, off или try
                                        # (изменение требует перезапуска)
#huge_page_size = 0                     # ноль для системного значения по умолчанию
                                        # (изменение требует перезапуска)
#temp_buffers = 8MB                     # минимум 800 кБ
#max_prepared_transactions = 0          # ноль отключает функцию
                                        # (изменение требует перезапуска)
# Предупреждение: не рекомендуется устанавливать max_prepared_transactions ненулевым,
# если вы не планируете активно использовать подготовленные транзакции.
#work_mem = 4MB                         # минимум 64 кБ
#hash_mem_multiplier = 2.0              # множитель от 1 до 1000.0 для work_mem хэш-таблиц
#maintenance_work_mem = 64MB            # минимум 1 МБ
#autovacuum_work_mem = -1               # минимум 1 МБ или -1 для использования maintenance_work_mem
#logical_decoding_work_mem = 64MB       # минимум 64 кБ
#max_stack_depth = 2MB                  # минимум 100 кБ
#shared_memory_type = mmap              # по умолчанию первый поддерживаемый системой вариант:
                                        #   mmap
                                        #   sysv
                                        #   windows
                                        # (изменение требует перезапуска)
dynamic_shared_memory_type = posix      # по умолчанию обычно первый поддерживаемый системой вариант:
                                        #   posix
                                        #   sysv
                                        #   windows
                                        #   mmap
                                        # (изменение требует перезапуска)
#min_dynamic_shared_memory = 0MB        # (изменение требует перезапуска)
#vacuum_buffer_usage_limit = 256kB      # размер кольцевого буфера стратегии доступа для vacuum и analyze;
                                        # 0 отключает стратегию доступа к буферу vacuum;
                                        # диапазон от 128 кБ до 16 ГБ

# - Диск -

#temp_file_limit = -1                   # ограничивает пространство временных файлов на процесс
                                        # в килобайтах, -1 для отсутствия лимита

# - Ресурсы ядра -

#max_files_per_process = 1000           # минимум 64
                                        # (изменение требует перезапуска)

# - Задержка на основе стоимости vacuum -

#vacuum_cost_delay = 0                  # от 0 до 100 миллисекунд (0 отключает)
#vacuum_cost_page_hit = 1               # от 0 до 10000 кредитов
#vacuum_cost_page_miss = 2              # от 0 до 10000 кредитов
#vacuum_cost_page_dirty = 20            # от 0 до 10000 кредитов
#vacuum_cost_limit = 200                # от 1 до 10000 кредитов

# - Фоновый писатель -

#bgwriter_delay = 200ms                 # от 10 до 10000 мс между циклами
#bgwriter_lru_maxpages = 100            # максимум записываемых буферов за цикл, 0 отключает
#bgwriter_lru_multiplier = 2.0          # множитель от 0 до 10.0 для сканируемых буферов за цикл
#bgwriter_flush_after = 512kB           # измеряется в страницах, 0 отключает

# - Асинхронное поведение -

#backend_flush_after = 0                # измеряется в страницах, 0 отключает
#effective_io_concurrency = 1           # от 1 до 1000; 0 отключает предварительную выборку
#maintenance_io_concurrency = 10        # от 1 до 1000; 0 отключает предварительную выборку
#max_worker_processes = 8               # (изменение требует перезапуска)
#max_parallel_workers_per_gather = 2    # ограничено max_parallel_workers
#max_parallel_maintenance_workers = 2   # ограничено max_parallel_workers
#max_parallel_workers = 8               # количество max_worker_processes, которые
                                        # могут использоваться в параллельных операциях
#parallel_leader_participation = on
#old_snapshot_threshold = -1            # от 1 мин до 60 дней; -1 отключает; 0 немедленно
                                        # (изменение требует перезапуска)


#------------------------------------------------------------------------------
# ЖУРНАЛ ПРЕДВАРИТЕЛЬНОЙ ЗАПИСИ (WAL)
#------------------------------------------------------------------------------

# - Настройки -

#wal_level = replica                    # minimal, replica или logical
                                        # (изменение требует перезапуска)
#fsync = on                             # сбрасывать данные на диск для защиты от сбоев
                                        # (отключение может привести к
                                        # невосстановимому повреждению данных)
#synchronous_commit = on                # уровень синхронизации;
                                        # off, local, remote_write, remote_apply или on
#wal_sync_method = fsync                # по умолчанию первый поддерживаемый системой вариант:
                                        #   open_datasync
                                        #   fdatasync (по умолчанию на Linux и FreeBSD)
                                        #   fsync
                                        #   fsync_writethrough
                                        #   open_sync
#full_page_writes = on                  # восстанавливать после частичных записей страниц
#wal_log_hints = off                    # также выполнять полные записи страниц для некритических обновлений
                                        # (изменение требует перезапуска)
#wal_compression = off                  # включает сжатие полных записей страниц;
                                        # off, pglz, lz4, zstd или on
#wal_init_zero = on                     # заполнять новые WAL-файлы нулями
#wal_recycle = on                       # переиспользовать WAL-файлы
#wal_buffers = -1                       # минимум 32 кБ, -1 устанавливает на основе shared_buffers
                                        # (изменение требует перезапуска)
#wal_writer_delay = 200ms               # от 1 до 10000 миллисекунд
#wal_writer_flush_after = 1MB           # измеряется в страницах, 0 отключает
#wal_skip_threshold = 2MB

#commit_delay = 0                       # диапазон от 0 до 100000, в микросекундах
#commit_siblings = 5                    # диапазон от 1 до 1000

# - Контрольные точки -

#checkpoint_timeout = 5min              # диапазон от 30 сек до 1 дня
#checkpoint_completion_target = 0.9     # целевая длительность контрольной точки, от 0.0 до 1.0
#checkpoint_flush_after = 256kB         # измеряется в страницах, 0 отключает
#checkpoint_warning = 30s               # 0 отключает
max_wal_size = 1GB
min_wal_size = 80MB

# - Предварительная выборка во время восстановления -

#recovery_prefetch = try                # предварительно выбирать страницы, указанные в WAL?
#wal_decode_buffer_size = 512kB         # окно предугадывания для предварительной выборки
                                        # (изменение требует перезапуска)

# - Архивирование -

#archive_mode = off             # включает архивирование; off, on или always
                                # (изменение требует перезапуска)
#archive_library = ''           # библиотека для архивирования WAL-файла
                                # (пустая строка означает использование archive_command)
#archive_command = ''           # команда для архивирования WAL-файла
                                # заполнители: %p = путь к файлу для архивации
                                #               %f = только имя файла
                                # например: 'test ! -f /mnt/server/archivedir/%f && cp %p /mnt/server/archivedir/%f'
#archive_timeout = 0            # принудительное переключение WAL-файла после
                                # указанного количества секунд; 0 отключает

# - Восстановление из архива -

# Эти параметры используются только в режиме восстановления.

#restore_command = ''           # команда для восстановления архивированного WAL-файла
                                # заполнители: %p = путь к файлу для восстановления
                                #               %f = только имя файла
                                # например: 'cp /mnt/server/archivedir/%f %p'
#archive_cleanup_command = ''   # команда, выполняемая при каждом restartpoint
#recovery_end_command = ''      # команда, выполняемая при завершении восстановления

# - Цель восстановления -

# Устанавливайте эти параметры только при целевом восстановлении.

#recovery_target = ''           # 'immediate' для завершения восстановления, как только
                                # достигнуто согласованное состояние
                                # (изменение требует перезапуска)
#recovery_target_name = ''      # именованная точка восстановления, до которой будет
                                # выполняться восстановление
                                # (изменение требует перезапуска)
#recovery_target_time = ''      # временная метка, до которой будет выполняться восстановление
                                # (изменение требует перезапуска)
#recovery_target_xid = ''       # ID транзакции, до которой будет выполняться восстановление
                                # (изменение требует перезапуска)
#recovery_target_lsn = ''       # WAL LSN, до которого будет выполняться восстановление
                                # (изменение требует перезапуска)
#recovery_target_inclusive = on # Указывает, останавливаться ли:
                                # сразу после указанной цели восстановления (on)
                                # или перед целью восстановления (off)
                                # (изменение требует перезапуска)
#recovery_target_timeline = 'latest'    # 'current', 'latest' или ID таймлайна
                                # (изменение требует перезапуска)
#recovery_target_action = 'pause'       # 'pause', 'promote', 'shutdown'
                                # (изменение требует перезапуска)


#------------------------------------------------------------------------------
# РЕПЛИКАЦИЯ
#------------------------------------------------------------------------------

# - Отправляющие серверы -

# Устанавливайте на первичном сервере и на любом резервном, отправляющем данные репликации.

#max_wal_senders = 10           # максимальное количество процессов walsender
                                # (изменение требует перезапуска)
#max_replication_slots = 10     # максимальное количество слотов репликации
                                # (изменение требует перезапуска)
#wal_keep_size = 0              # в мегабайтах; 0 отключает
#max_slot_wal_keep_size = -1    # в мегабайтах; -1 отключает
#wal_sender_timeout = 60s       # в миллисекундах; 0 отключает
#track_commit_timestamp = off   # собирать временные метки фиксации транзакций
                                # (изменение требует перезапуска)

# - Первичный сервер -

# Эти настройки игнорируются на резервном сервере.

#synchronous_standby_names = '' # резервные серверы для синхронной репликации
                                # метод выбора синхронных резервов, количество синхронных резервов
                                # и список application_name резервных серверов через запятую
                                # '*' = все

# - Резервные серверы -

# Эти настройки игнорируются на первичном сервере.

#primary_conninfo = ''                  # строка подключения к отправляющему серверу
#primary_slot_name = ''                 # слот репликации на отправляющем сервере
#hot_standby = on                       # "off" запрещает запросы во время восстановления
                                        # (изменение требует перезапуска)
#max_standby_archive_delay = 30s        # максимальная задержка перед отменой запросов
                                        # при чтении WAL из архива;
                                        # -1 допускает бесконечную задержку
#max_standby_streaming_delay = 30s      # максимальная задержка перед отменой запросов
                                        # при чтении потокового WAL;
                                        # -1 допускает бесконечную задержку
#wal_receiver_create_temp_slot = off    # создавать временный слот, если primary_slot_name
                                        # не задан
#wal_receiver_status_interval = 10s     # отправлять ответы не реже, чем с этим интервалом
                                        # 0 отключает
#hot_standby_feedback = off             # отправлять информацию с резерва для предотвращения
                                        # конфликтов запросов
#wal_receiver_timeout = 60s             # время ожидания получателем связи от первичного
                                        # в миллисекундах; 0 отключает
#wal_retrieve_retry_interval = 5s       # время ожидания перед повторной попыткой
                                        # получения WAL после неудачи
#recovery_min_apply_delay = 0           # минимальная задержка для применения изменений во время восстановления

# - Подписчики -

# Эти настройки игнорируются на издателе.

#max_logical_replication_workers = 4    # берётся из max_worker_processes
                                        # (изменение требует перезапуска)
#max_sync_workers_per_subscription = 2  # берётся из max_logical_replication_workers
#max_parallel_apply_workers_per_subscription = 2        # берётся из max_logical_replication_workers


#------------------------------------------------------------------------------
# ОПТИМИЗАЦИЯ ЗАПРОСОВ
#------------------------------------------------------------------------------

# - Конфигурация методов планировщика -

#enable_async_append = on
#enable_bitmapscan = on
#enable_gathermerge = on
#enable_hashagg = on
#enable_hashjoin = on
#enable_incremental_sort = on
#enable_indexscan = on
#enable_indexonlyscan = on
#enable_material = on
#enable_memoize = on
#enable_mergejoin = on
#enable_nestloop = on
#enable_parallel_append = on
#enable_parallel_hash = on
#enable_partition_pruning = on
#enable_partitionwise_join = off
#enable_partitionwise_aggregate = off
#enable_presorted_aggregate = on
#enable_seqscan = on
#enable_sort = on
#enable_tidscan = on

# - Константы стоимости планировщика -

#seq_page_cost = 1.0                    # измеряется в произвольной шкале
#random_page_cost = 4.0                 # та же шкала, что выше
#cpu_tuple_cost = 0.01                  # та же шкала, что выше
#cpu_index_tuple_cost = 0.005           # та же шкала, что выше
#cpu_operator_cost = 0.0025             # та же шкала, что выше
#parallel_setup_cost = 1000.0           # та же шкала, что выше
#parallel_tuple_cost = 0.1              # та же шкала, что выше
#min_parallel_table_scan_size = 8MB
#min_parallel_index_scan_size = 512kB
#effective_cache_size = 4GB

#jit_above_cost = 100000                # выполнять JIT-компиляцию, если доступно
                                        # и запрос дороже этого значения;
                                        # -1 отключает
#jit_inline_above_cost = 500000         # встраивать небольшие функции, если запрос
                                        # дороже этого значения; -1 отключает
#jit_optimize_above_cost = 500000       # использовать дорогие JIT-оптимизации, если
                                        # запрос дороже этого значения;
                                        # -1 отключает

# - Генетический оптимизатор запросов -

#geqo = on
#geqo_threshold = 12
#geqo_effort = 5                        # диапазон от 1 до 10
#geqo_pool_size = 0                     # выбирает значение по умолчанию на основе effort
#geqo_generations = 0                   # выбирает значение по умолчанию на основе effort
#geqo_selection_bias = 2.0              # диапазон от 1.5 до 2.0
#geqo_seed = 0.0                        # диапазон от 0.0 до 1.0

# - Другие опции планировщика -

#default_statistics_target = 100        # диапазон от 1 до 10000
#constraint_exclusion = partition       # on, off или partition
#cursor_tuple_fraction = 0.1            # диапазон от 0.0 до 1.0
#from_collapse_limit = 8
#jit = on                               # разрешить JIT-компиляцию
#join_collapse_limit = 8                # 1 отключает свёртку явных
                                        # JOIN-предложений
#plan_cache_mode = auto                 # auto, force_generic_plan или
                                        # force_custom_plan
#recursive_worktable_factor = 10.0      # диапазон от 0.001 до 1000000


#------------------------------------------------------------------------------
# ОТЧЁТЫ И ЛОГИРОВАНИЕ
#------------------------------------------------------------------------------

# - Куда логировать -

#log_destination = 'stderr'              # Допустимые значения — комбинации
                                        # stderr, csvlog, в зависимости от платформы
                                        # csvlog требует включения logging_collector

# Используется при логировании в stderr:
#logging_collector = off                 # Включить сбор stderr, jsonlog и
                                        # csvlog в файлы логов. Требуется включить
                                        # для csvlog и jsonlog.
                                        # (изменение требует перезапуска)

# Используются только если включён logging_collector:
#log_directory = 'log'                  # каталог для записи файлов логов,
                                        # может быть абсолютным или относительным к PGDATA
#log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log' # шаблон имени файла лога,
                                        # может включать escapes strftime()
#log_file_mode = 0600                   # режим создания файлов логов,
                                        # начинать с 0 для восьмеричной нотации
#log_rotation_age = 1d                  # Автоматическая ротация логов произойдёт
                                        # после этого времени. 0 отключает.
#log_rotation_size = 10MB              # Автоматическая ротация логов произойдёт
                                        # после этого объёма вывода лога.
                                        # 0 отключает.
#log_traveled_on_rotation = off         # Если включено, существующий файл лога с
                                        # тем же именем, что и новый, будет обрезан,
                                        # а не дополнен.
                                        # Но такая обрезка происходит только при
                                        # ротации по времени, не при перезапуске
                                        # # или ротации по размеру. По умолчанию off,
                                        #
                                        # что означает добавление в существующие файлы
                                        # во всех случаях.

# Используются при логировании в syslog:
#syslog_facility = 'LOCAL0'
#syslog_ident = 'postgres'
#syslog_sequence_numbers = on
#syslog_split_messages = on

# Используется только при логировании в eventlog (Windows):
# (изменение требует перезапуска)
#event_source = 'PostgreSQL'

# - Когда логировать -

#log_min_messages = warning              # Значения в порядке уменьшения детализации:
                                         #   debug5
                                         #   debug4
                                         #   debug3
                                         #   debug2
                                         #   debug1
                                         #   info
                                         #   notice
                                         #   warning
                                         #   error
                                         #   log
                                         #   fatal
                                         #   panic

#log_min_error_statement = error         # Значения в порядке уменьшения детализации:
                                         #   debug5
                                         # ... (аналогично log_min_messages)
                                         #   panic (эффективно отключает)

#log_min_duration = -1                  # -1 отключает, 0 логирует все операционные действия
                                        # и их длительность, >0
                                        # операторов только для
                                        # только для операторов, выполняющихся не менее этого числа
                                        # миллисекунд

#log_min_duration_sample = -1           # -1 отключает, 0 логирует выборку операторов
                                        # и их длительность, >0 логирует только выборку
                                        # операторов, выполняющихся не менее этого числа
                                        # миллисекунд;
                                        # доля выборки определяется log_statement_sample_rate

#log_statement_sample_rate = 1.0        # доля логируемых операторов, превышающих
                                        # log_min_duration_sample, для записи;
                                        # 1.0 логирует все такие операторы, 0.0 никогда не логирует

#log_transaction_sample_rate = 0.0      # доля транзакций, чьи операторы
                                        # логируются независимо от их длительности;
                                        # 1.0 логирует все операторы из всех транзакций, 0.0 никогда не логирует

#log_startup_progress_interval = 10s    # Время между обновлениями прогресса для
                                        # длительных операций.
                                        # 0 отключает функцию, >0 указывает
                                        # интервал в миллисекундах.

# - Какие логировать данные -

#debug_print_parse = off
#debug_print_rewritten = off
#debug_print_plan = off
#debug_pretty_print = on
#log_autovacuum_min_duration = 10min    # логировать активность автоваакума;
                                        # -1 отключает, 0 логирует все действия и
                                        # их длительность, >0 логирует только
                                        # действия, выполняющиеся не менее
                                        # указанного числа миллисекунд.
#log_checkpoints = on
#log_connections = off
#log_disconnections = off
#log_duration = off
#log_error_verbosity = default          # краткие, сообщения по умолчанию или подробные
#log_hostname = off
#log_line_prefix = '%m [%p] '           # специальные значения:
                                        # %a = имя приложения
                                        # %u = имя пользователя
                                        # %d = имя базы данных
                                        # %r = удалённый хост и порт
                                        # %h = удалённый хост
                                        # %b = тип приложения
                                        # %p = ID процесса
                                        # %P = ID процесса лидера параллельной группы
                                        # %t = временная метка без миллисекунд
                                        # %m = временная метка с миллисекундами
                                        # %n = временная метка с миллисекундами (в формате Unix epoch)
                                        # %Q = ID запроса (0, если нет или не вычислен)
                                        # %i = тег команды
                                        # %e = состояние SQL
                                        # %c = ID сессии
                                        # %l = номер строки сессии
                                        # %s = временная метка начала сессии
                                        # %v = виртуальный ID транзакции
                                        # %x = ID транзакции (0, если нет)
                                        # %q = остановиться здесь в процессах
                                        #      не связанных с сессией
                                        # %% = '%'
                                        # например: '<%u%%%d> '
#log_lock_waits = off                   # логировать ожидания блокировок >= deadlock_timeout
#log_recovery_conflict_waits = off      # логировать ожидания конфликтов восстановления резерва
                                        # >= deadlock_timeout
#log_parameter_max_length = -1          # при логировании операторов ограничивать
                                        # значения параметров привязки до N байт;
                                        # -1 означает печатать полностью, 0 отключает
#log_parameter_max_length_on_error = 0  # при логировании ошибки ограничивать
                                        # значения параметров привязки до N байт;
                                        # -1 означает печатать полностью, 0 отключает
#log_statement = 'none'                 # none, ddl, mod, all
#log_replication_commands = off
#log_temp_files = -1                    # логировать временные файлы, равные или больше
                                        # указанного размера в килобайтах;
                                        # -1 отключает, 0 логирует все временные файлы
log_timezone = 'Etc/UTC'

# - Название процесса -

#cluster_name = ''                      # добавляется в заголовки процессов, если не пусто
                                        # (изменение требует перезапуска)
#update_process_title = on


#------------------------------------------------------------------------------
# СТАТИСТИКА
# - Кумулятивная статистика запросов и индексов -

#track_activities = on
#track_activity_query_size = 1024       # (изменение требует перезапуска)
#track_counts = on
#track_io_timing = off
#track_wal_io_timing = off
#track_functions = none                 # none, pl, all
#stats_fetch_consistency = cache        # cache, none, snapshot

# - Мониторинг -

#compute_query_id = auto
#log_statement_stats = off
#log_parser_stats = off
#log_planner_stats = off
#log_executor_stats = off


#------------------------------------------------------------------------------
# АВТОВАКУУМ
#------------------------------------------------------------------------------

#autonomous = on                        # Включить подпроцесс автоваума?
                                        # 'on' требует, чтобы также был включен track_counts
                                        #
#autovacuum_max_workers = 3             # Максимальное количество подпроцессоров автоваума
                                        # (изменение требует перезапуска)
#autovacuum_naptime = 1min              # время между запусками автоваума
#autovacuum_vacuum_threshold = 50       # минимальное количество обновлений строк перед
                                        # вакуумом
#autovacuum_vacuum_insert_threshold = 1000   # минимальное количество вставок строк
                                             # перед вакуумом; -1 отключает вакуумы по вставкам
                                             #  вакуумы
#autovacuum_analyze_threshold = 50      # минимальное количество обновлений строк перед
                                        # анализом
#autovacuum_vacuum_scale_factor = 0.2   # доля размера таблицы перед вакуумом
#autovacuum_vacuum_insert_scale_factor = 0.2    # доля вставок относительно размера таблицы
                                                # перед вставкой вакуума
#autovacuum_analyze_scale_factor = 0.1  # доля размера таблицы перед анализом
#autovacuum_freeze_max_age = 200000000  # максимальный возраст XID
                                        # перед принудительным вакуумом
                                        # (изменение требует перезапуска)
                                        # (перезапускается изменение)
#autovacuum_multixact_freeze_max_age = 400000000        # максимальный возраст мультикса
                                                        # перед принудительным вакуумом
                                                        # (применением требует перезапускается)
                                                        # (перезапускается) 
#autovacuum_vacuum_cost_delay = 2ms     # задержка по умолчанию для стоимости автоваума
                                        # автова, в миллисекундах
                                        # -1 означает использование vacuum_cost_delay;
                                        # -1
                                        #
#autovacuum_cost_limit = -1             # лимит стоимости по умолчанию для
                                        # автоваума, -1 означает использование
                                        # vacuum_cost_limit лимита


#------------------------------------------------------------------------------
# НАСТРОЙКИ ПО ДКЛКЛЕНТА ПО ПОДКЛЮЧЕНИЯМ
#------------------------------------------------------------------------------


# - Поведение операторов -

#client_min_messages = notice           # Значения в порядке уменьшения детализации:
                                        #   debug5
                                        #   debug4
                                        #   debug3
                                        #   debug2
                                        #   debug1
                                        #   log
                                        #   notice
                                        #   warning
                                        #   error
                                        #
#search_path = '"$uuser"', 'ppublic'        # имена публичных схем
#row_security = on
#default_table_access_method = 'hheap'      # Метод доступа к таблицам по умолчанию
#default_tablespace = ''                    # имя табличного пространства, '' использует значение по умолчанию
#default_toast_compression = 'pglz'     # 'pglz' или 'lz4' для
#temp_tablespaces = ''                  # список имен табличных пространств, '' использует только
#                                       # табличное пространство по умолчанию
                                        # только по умолчанию
#check_function_bodies = 0on             # проверять тела функций проверки
#default_transaction_isolation = 'read-only'    # изоляция транзакции по умолчанию
#default_transaction_read_only = 0off           # только чтение
#default_transaction_deferrable = off           # отложенные транзакции
#session_replication_role = 'oorigin'           # роль репликации сессии
#statement_timeout = ''                         # в миллисекундах, 0 отключает
#lock_timeout = 0                               # в миллисекундах; миллисекундах, 0 отключает
#idle_in_transaction_session_timeout = 0        # в миллисекундах; миллисекунд
#idle_session_timeout = 0                       # в миллисекундах, 0; 0 отключается
#vacuum_freeze_table_age = 150000000            # возраст заморозки таблицы
#freeze_min_age = 50000000                   
#vacuum_failsafe = 0                            # vaccoom; 0 откладывает
#vacuum_multixact_freeze_table_age = 150000000  # возраст заморозки мультикса
#multixact_freeze_min_age = 5000                #0
# 0vacuum_failsafe_age = 0                      # возраст
#bytea_output = 'hhex'                         # hex, вывод format
#xmlbinary = 'base64'
#xmloption = 'ccontent'
#gin_pending_list_limit = 4MB
#createrole_self_grant = ''                 # предоставление прав самому себе
                                            # set и/или inherit

# - Локализация и форматирование -

datestyle = 'iso, mdy'
#intervalstyle = 'postgres'
timezone = 'Etc/UTC'
#timezone_abbreviations = 'Default'     # Выберите набор доступных сокращений для часового пояса
                                        # сокращений часового пояса. В настоящее время:
                                        #   Default
                                        #   Australia (историческое использование)
                                        #   India
                                        # Вы можете создать собственный файл в
                                        # share/timezonesets/.
#extra_float_digits = 12                # мин -15, макс 3; любое значение >0
                                        # фактически выбирает точный режим вывода >0
#client_encoding = sql_ascii            # на самом факеле, по умолчанию кодировка базы данных
                                        # кодирование

# Эти настройки инициализируются initdb с помощью, но их можно изменить.
lc_messages = 'en_USA.utf8'             # Локализация локаль для системных сообщений об ошибках
                                        # строк сообщений
lc_monetary = 'en_USA.utf8'             # локализация локаль для денежного форматирования
lc_numeric = 'en_USA.utf8'              # UTF-8 для форматирования чисел
lc_time = 'en_USA.utf8'                 # форматирование времени локаль
                                        # формата

#icu_validation_level = warning         # сообщать об ошибках валидации локали ICU локали на указанном уровне
                                        # уровне сообщений

# Конфигурация по умолчанию для текстового поиска
default_text_search_config = 'pg_catalog.english'

# - Предварительная загрузка библиотек -

#local_preload_libraries = ''
#session_preload_libraries = ''
#shared_preload_libraries = ''           # (change requires shared library preload)
#jit_provider = 'llvmjit'                # Библиотека JIT для использования

# - Прочие значения по умолчанию -

#dynamic_library_path = '$ldir'          # путь к файлу библиотеки
#extension_destdir = ''                  # добавлять путь при загрузке расширений и
                                         # shared objects (добавлено Debian)
#gin_fuzzy_search_limit = 0


#------------------------------------------------------------------------------
# УПРАВЛЕНИЕ БЛОКИРОВКАМИ
#------------------------------------------------------------------------------


#deadlock_timeout = 1s                  # таймаут взаимоблокировки
#max_locks_per_transaction = 64         # минимум 10
                                        # (изменение требует перезапуска)
#max_pred_locks_per_transaction = 64    # максимум предсказанных блокировок
                                        # минимум 10
                                        # (изменение требует перезапуска)
# max_pred_locks_per_relation = -2      # максимум предсказанных блокировок на отношение (отрицательное значение mean)
                                        # (max_pred_locks_per_transaction / -max_pred_locks_per_relation) - 1
#max_pred_locks_per_page = 2            # min 0
                                        


#------------------------------------------------------------------------------
# СОВМЕСТИМОСТЬ ВЕРСИЙ И ПЛАТФОРМ
#------------------------------------------------------------------------------


# - Предыдущие версии PostgreSQL -

#array_nulls = on                       # включение нулевых массивов
#backslash_quote = safe_encoding        # on, off или safe_encoding
#escape_string_warning = on
#lo_compat_privileges = off
#quote_all_identifiers = off
#standard_conforming_strings = on
#synchronize_seqscans = on

# - Прочие платформы или клиенты -

#transform_null_equals = off           # преобразование NULL равно


# - Обработка ошибок
#------------------------------------------------------------------------------

#exit_on_error = off                    # Пререрывать сессию при любой ошибке
#restart_after_crash = on               # Перезапускать сервер после сбоя
#data_sync_retry = off                  # Попытаться синхронизировать данные после сбоя
                                        # (change requires restart)
#recovery_init_sync_method = fsync      # метод синхронизации: fsync, syncfs (Linux 5.8+)



#------------------------------------------------------------------------------
# ВКЛючение КОНФИГ ФАЙЛОВ
#------------------------------------------------------------------------------

# Эти параметры позволяют загружать настройки из файлов, кроме
# postgresql.conf по умолчанию. Это директивы, а не присваивания,
# поэтому их можно использовать несколько раз.

#include_directories = '...'            # включать файлы, заканчивающиеся на '.conf', из
                                        # каталога, например, 'conf.d'
#include_if_exists = '...'              # включать только если файлы существуют
#include = '...'                        # включение файла


#------------------------------------------------------------------------------
# ПОЛЬЗОВАТЕЛЬСКИЕ НАСТРОЙКИ
#------------------------------------------------------------------------------

# Добавить настройки для расширений здесь
```