# Развертывание Metabase в K3s

Metabase ([metabase.com](https://www.metabase.com/)) — это мощный и дружелюбный инструмент BI (Business Intelligence).
Он работает через веб-интерфейс, позволяет подключаться к реальным БД, делать запросы (SQL и визуальный конструктор),
строить графики, дашборды, отчёты (и отправлять эти отчеты по email-рассылки) и многое другое.

Metabase немного похож на [Power BI](https://www.microsoft.com/en-us/power-platform/products/power-bi/) или
[Tableau](https://www.tableau.com/), но Open-Source. 

Поддерживаются основные СУБД: PostgreSQL, MySQL/MariaDB, SQLite, ClickHouse, MongoDB, Microsoft SQL Server, BigQuery
и многие другие.

Metabase — монолит (Java JAR), и работает на сервере (или любом компьютере) как один процесс Java Virtual Machine.
Для хранения своей конфигурации использует встроенную базу данных H2, но его можно подключить и внешней СУБД
(например, PostgreSQL или MariaDB). И главное, для меня, он поддерживает ARM64? (а значит заработает [на моем k3s
на базе Orange Pi 5](../raspberry-and-orange-pi/k3s.md)).


## Подготовка базы данных

В deployment-манифесте Metabase будут указаны параметры подключения к PostgreSQL. Он у меня тоже развернут как под k3s
(см.: [развертывание PostgeSQL в K3s](k3s-postresql.md).

Создадим пользователя PostgreSQL, базу данных и права пользователя для Metabase. Это можно сделать через `psql` или
любой другой клиент PostgreSQL. Нужно выполнить следующие SQL-команды (не забудьте заменить пароль на свой):
```sql
CREATE USER metabase_user WITH ENCRYPTED PASSWORD 'очень-секретный-пароль-123!!';
CREATE DATABASE metabase OWNER metabase_user;
GRANT ALL PRIVILEGES ON DATABASE metabase TO metabase_user;
GRANT ALL ON SCHEMA public TO metabase_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO metabase_user;
```

Что здесь происходит:
- Создаем пользователя `metabase_user` с паролем `очень-секретный-пароль-123!!`.
- Создаем базу данных `metabase` и назначаем владельцем этого пользователя.
- Предоставляем все привилегии на базу данных `metabase` этому пользователю (можно не делать, т.к. мы уже указали владельца).
- Схема public создаётся автоматически в базе, но чтобы metabase_user мог работать с таблицами предоставляем все права
  на схему `public` базы данных (это стандартная схема PostgreSQL).:
- Чтобы пользователь мог создавать таблицы, функции и прочее в схеме...

# Манифесты для развертывания Metabase в K3s

У меня Metabase будет доступен по адресу `http://mb.local` (через VIP-адрес Keeepalive). Замените доменное имя на свое,
и не забудьте настроить DNS или файл `/etc/hosts`.

Для развертывания Metabase в k3s нам понадобятся следующие манифесты Kubernetes (не забудьте поменять пароль
пользователя PostgreSQL на свой):
```yaml
# ~/k3s/metabase/metabase.yaml
# Все манифесты для Metabase в k3s

# 1. Namespace: создаём пространство имен `metabase`
apiVersion: v1
kind: Namespace
metadata:
  name: metabase

---
# 2. Secret: храним пароль к PostgreSQL в Kubernetes-секрете (это безопаснее, чем указывать его прямо в Deployment)
apiVersion: v1
kind: Secret
metadata:
  name: metabase-db-secret
  namespace: metabase
type: Opaque
stringData:
  MB_DB_PASS: 'очень-секретный-пароль-123!!'  # Пароль. Не закодирован, но kubectl хранит его в base64.

---
# 3. PVC: том для временных данных Metabase
# Metabase хранит всё важное в PostgreSQL, но PVC-том нужен для кеша, логов и временных файлов
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: metabase-data
  namespace: metabase
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 512Mi  # Достаточно для большинства задач

---
# 4. Deployment: контейнер с Metabase (+указываем переменные окружения для подключения к PostgreSQL)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metabase
  namespace: metabase
spec:
  replicas: 1
  selector:
    matchLabels:
      app: metabase
  template:
    metadata:
      labels:
        app: metabase
    spec:
      containers:
        - name: metabase
          image: metabase/metabase:latest  # ARM64-совместимый образ
          ports:
            - containerPort: 3000  # Стандартный порт Metabase внутри контейнера
          env:
            - name: MB_DB_TYPE
              value: postgres
            - name: MB_DB_DBNAME
              value: metabase
            - name: MB_DB_PORT
              value: "5432"  # В кавычках: безопаснее в YAML
            - name: MB_DB_USER
              value: metabase_user
            - name: MB_DB_HOST
              value: postgres.postgresql.svc.cluster.local  # Берем из Service-монифеста PostgreSQL (или хост `pg.local`, доступный внутри сети)
            - name: MB_DB_PASS
              valueFrom:
                secretKeyRef:
                  name: metabase-db-secret  # Секрет, созданный выше
                  key: MB_DB_PASS
            - name: JAVA_TIMEZONE
              value: Europe/Moscow
          volumeMounts:
            - name: metabase-storage
              mountPath: /metabase-data  # Временные файлы, кеши, логи
      volumes:
        - name: metabase-storage
          persistentVolumeClaim:
            claimName: metabase-data

---
# 5. Сервис: внутренняя точка доступа (не публикуется наружу)
apiVersion: v1
kind: Service
metadata:
  name: metabase
  namespace: metabase
spec:
  selector:
    app: metabase
  ports:
    - port: 80
      targetPort: 3000  # Проксируем внешний порт 80 → контейнерный 3000
  type: ClusterIP

---
# 6. IngressRoute: Traefik CRD для публикации Metabase по адресу http://mb.local
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: metabase
  namespace: metabase
spec:
  entryPoints:
    - web  # HTTP-порт Traefik (обычно 80)
  routes:
    - match: Host("mb.local")
      kind: Rule
      services:
        - name: metabase
          port: 80
```

Применим манифесты командой:
```bash
kubectl apply -f ~/k3s/metabase/metabase.yaml
```

Все должно заработать, и Metabase будет доступен по адресу `http://mb.local` (или по тому доменному имени, которое
вы указали). Если что-то не так, то скорее всего проблемы с подключением к PostgreSQL. Удалите базу, пользователя,
k3s-деплоймент, и создайте заново. Диагностировать причину неполадок можно посмотрев логи пода:
```bash
kubectl -n metabase logs deploy/metabase
```

