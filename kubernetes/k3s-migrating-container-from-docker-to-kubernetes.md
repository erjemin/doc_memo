# Перенос контейнера из Docker в k3s (на примере Gitea)

Вот эта самая инструкция, котору вы сейчас читаете размещена на моем персональном сервере Gitea. Раньше она размещалась
на домашнем хосте Orange Pi 5 в Docker-контейнере, и проксировалась через nginx c добавлением SSL-сертификата
Let's Encrypt. Мне показалось, что зависимость от одного хоста -- опасненько. И я решил перенести Gitea в кластер,
в котором у меня несколько узлов, и в случае падения одного из них, Gitea продолжит работать на другом узле. К тому
же мне очень хотелось подключить старый Orange Pi 5 тоже к кластеру (ведь для этого нужно установить чистую систему и
[перекомпилировать ядро](../raspberry-and-orange-pi/opi5plus-rebuilding-linux-kernel-for-iscsi.md) 
).

Я решил задокументировать процесс переноса контейнера из Docker в k3s, тем более Gitea был не единственный контейнер,
который нужно было перенести. Возможно мой опыт вам тоже пригодится.


## Перенос данных Docker-контейнера Gitea на узел k3s

Останавливаем докер

Архивируем данные gitea

Переносим данные на узел k3s (в моем случае это opi5plus-1) по SCP:


## Подготовка узла k3s

Создадим пространство имен для gitea в k3s (чтобы все было аккуратно):
```bash
sudo kubectl create namespace gitea
```

Создаем папку для хранения манифестов gitea:
```bash
mkdir -p ~/k3s/gitea
```

## Перемещаем файлы и базу данных SQLite в блочное хранилище k3s

Теперь нам надо перенести данные gitea в k3s в PersistentVolumeClaim (Longhorn). Longhorn -- это блочное хранилище k3s,
которое позволяет создавать и управлять блочными томами в кластере для обеспечения высокой доступности
и отказоустойчивости. Если узел, на котором находится том, выходит из строя, Longhorn автоматически перемещает том
на другой узел и контейнер продолжает работу с томом, как будто ничего не произошло.

Создадим манифест для PersistentVolumeClaim (PVC) и PersistentVolume (PV):
```bash
nano ~/k3s/gitea/longhorn-pvc.yaml
```

Вставляем в него следующее содержимое:
```yaml
piVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitea-pvc     # Имя PVC-хранилища
  namespace: gitea    # Пространство имен gitea
spec:
  accessModes:
    - ReadWriteOnce   # Режим доступа к PVC -- чтение и запись
  storageClassName: longhorn   # Используем Longhorn как класс хранения
  resources:
    requests:
      storage: 10Gi   # Размер под данные Gitea 10 Гб (максимальный объем)
```

Применим манифест:
```bash
sudo kubectl apply -f ~/k3s/gitea/longhorn-pvc.yaml
```

Проверим, что PVC создан и доступен:
```bash
sudo kubectl get pvc -n gitea -o wide
```

Увидим что-то вроде:
```text
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE   VOLUMEMODE
gitea-pvc   Bound    pvc-5a562c67-89b2-48e2-97a6-25035b90729a   10Gi       RWO            longhorn       <unset>                 20h   Filesystem
```

Создадим временный под `gitea-init-data` для переноса данных gitea с хост-узла в хранилище Longhorn:
```bash
nano ~/k3s/gitea/gitea-init-data.yaml
```

Манифест временного пода:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gitea-init-data
  namespace: gitea
spec:
  nodeName: opi5plus-1        # Привязка к хосту opi5plus-1 (на этом узле у нас лежит архив с данными gitea из docker)
  containers:
    - name: init-data
      image: alpine:latest
      command: ["/bin/sh", "-c", "tar -xzf /mnt/gitea-data.tar.gz -C /data && chmod -R 777 /data && ls -la /data && sleep 3600"]
      volumeMounts:
        - name: gitea-data
          mountPath: /data
        - name: tmp-data
          mountPath: /mnt
  volumes:
    - name: gitea-data
      persistentVolumeClaim:
        claimName: gitea-pvc
    - name: tmp-data
      hostPath:
        path: /home/opi/tmp
        type: Directory      # Указываем, что это папка
  restartPolicy: Never
```

Что тут происходит:
- `metadata` — задаёт имя пода `gitea-init-data` и привязывает его к пространству имен `gitea`.
- `nodeName` — гарантирует запуск на нужной ноде (в данном случае `opi5plus-1`).
- `containers` — задаёт контейнер с именем `init-data`, который использует образ `alpine:latest`.
  - `command` — выполняет команду в контейнере:
    - `tar -xzf /mnt/gitea-data.tar.gz -C /data` — распаковывает архив в `/data`.
    - `chmod -R 777 /data` — задаёт права на папку `/data`.
    - `ls -la /data` — выводит содержимое `/data` в логи.
    - `sleep 3600` — держит под живым 1 час, чтобы ты можно было зайти в sh и проверить, что всё распаковалось.
  - `volumeMounts` — монтирует два тома:
    - том и именем `gitea-data` монтируем PVC в `/data`.
    - том и именем `tmp-data` монтирует временную папку на хосте в `/mnt`.
- `volumes` — определяет расположение томов:
    - том `gitea-data` — размещается в PersistentVolumeClaim (PVC) хранилища `gitea-pvc`, которое мы создали ранее.
    - том `tmp-data` — размещается в каталоге хоста `/home/opi/tmp` (как рам, у нас лежит архив с данными gitea
      из docker).
- `restartPolicy: Never` — под не будет перезапускаться, если завершится.

Применим манифест:
```bash
sudo kubectl apply -f ~/k3s/gitea/gitea-init-data.yaml
```

Проверим, что под создан и работает:
```bash
sudo kubectl get pod -n gitea -o wide
```

Увидим что-то вроде:
```text
NAME              READY   STATUS    RESTARTS   AGE   IP           NODE         NOMINATED NODE   READINESS GATES
gitea-init-data   1/1     Running   0          4m    10.42.2.64   opi5plus-1   <none>           <none>
```

Проверим логи пода:
```bash
sudo kubectl logs -n gitea gitea-init-data
```

Увидим что-то вроде:
```text
total 36
drwxrwxrwx    6 1000     1000          4096 Apr 16 15:09 .
drwxr-xr-x    1 root     root          4096 Apr 17 13:01 ..
drwxrwxrwx    5 root     root          4096 Apr 16 15:09 git
drwxrwxrwx   18 root     root          4096 Apr 17 13:02 gitea
drwxrwxrwx    2 root     root         16384 Apr 16 15:27 lost+found
drwxrwxrwx    2 root     root          4096 Apr 17 13:01 ssh
```

Как видим, данные благополучно распаковались в `/data` внутри пода, и это Longhorn PVC `gitea-pvc`. Можно также "зайти"
в под и посмотреть, что там внутри:
```bash
sudo kubectl exec -it -n gitea gitea-init-data -- /bin/sh
```

Внутри пода дать команду, например:
```bash
ls -l /data/gitea/gitea.db
```

И убедиться, что данные gitea распакованы. Увидим что-то вроде:
```text
-rwxrwxrwx    1 root     root       2555904 Apr 16 15:09 /data/gitea/gitea.db
```

База SQLite gitea.db на месте. Выходим из пода:
```bash
exit
```

## Создание пода gitea и подключение к нему хранилища

Теперь нужно создать под с Gitea, подключить к нему PVC `gitea-pvc` и проверить, что данные подцепились.

### Настройка Аффинити -- предпочтительные узлы 

В моем кластере k3s на OrangePi 5 Plus несколько узлов работают на nVME SSD, а некоторые на eMMC. Накопители
SSD быстрее (если интересно, то вот [заметка о тестировании производительности дискового накопителя](../raspberry-and-orange-pi/measuring-performance-storage-devices.md))
и потому если контейнер с Gitea будет работать на SSD, то он будет работать быстрее. Как настроить предпочтение узлов
описано в [заметке о аффинити](k3s-affinate.md), поэтому кратко: присваиваем узлам метки, например `disk=ssd`:
```bash
sudo kubectl label nodes opi5plus-1 disk=ssd
sudo kubectl label nodes opi5plus-3 disk=ssd
```

Проверяем, что метки добавлены:
```bash
sudo kubectl get nodes --show-labels | grep "disk=ssd"
```

Будут показаны только узлы с меткой `disk=ssd`. У каждого узда очень много меток.

### Создание манифеста для развертывания Gitea

Создадим манифест для развертывания Gitea (deployment):
```bash
nano ~/k3s/gitea/gitea-deployment.yaml
```

Вставляем в него следующее содержимое:
```yaml
apiVersion: apps/v1
kind: Deployment                # определяем тип ресурса -- Deployment (развертывание)
metadata:                       # определяем метаданные развертывания
  name: gitea                     # имя развертывания `gitea`
  namespace: gitea                # в пространстве имен `gitea`
  labels:                         
    app: gitea                    
spec:
  replicas: 1                       # количество реплик (подов) в кластере -- 1
  selector:
    matchLabels:
      app: gitea
  template:
    metadata:
      labels:
        app: gitea
    spec:
      affinity:                     # определяем аффинити (предпочтения) для узлов
        nodeAffinity:                 # аффинити для узлов
          preferredDuringSchedulingIgnoredDuringExecution: 
            - weight: 100                 # вес предпочтения -- 100
              preference:                   # предпочтение
                matchExpressions:             # выражения для соответствия
                  - key: disk                   # метка узла `disk`...
                    operator: In                # ...оператор `In` (входит в множество)...
                    values:                     # ...значений...
                      - ssd                       # ...значение `ssd` (т.е. узлы с SSD)
      containers:                   # определяем контейнеры, которые будут запущены в поде
        - name: gitea                 # имя контейнера `gitea`
          image: gitea/gitea:latest     # образ
          env:                          # переменные окружения
            - name: USER_UID              # идентификатор пользователя
              value: "1000"
            - name: USER_GID              # идентификатор группы
              value: "1000"
          ports:                        # определяем порты, которые будут открыты в контейнере
            - containerPort: 3000       # порт 3000
              name: http                  # будет именоваться 'http' (используется в Gitea для веб-интерфейса)
            - containerPort: 22         # порт 22
              name: ssh                   # будет именоваться 'ssh' (используется в Gitea для SSH-доступа)
          volumeMounts:                 # монтируем тома в контейнер
            - name: gitea-data            # том с именем 'gitea-data'...
              mountPath: /data              # в каталог '/data' внутри контейнера
            - name: timezone              # том 'timezone'...
              mountPath: /etc/timezone      # в каталог '/etc/timezone' внутри контейнера
              readOnly: true                # только для чтения
            - name: localtime             # том 'localtime'...
              mountPath: /etc/localtime     # в каталог '/etc/localtime' внутри контейнера
              readOnly: true                # только для чтения
      volumes:                        # определяем тома, которые будут использоваться в поде
        - name: gitea-data            # том с именем 'gitea-data'...
          persistentVolumeClaim:        # используем PersistentVolumeClaim (PVC Longhorn)
            claimName: gitea-pvc          # имя PVC 'gitea-pvc'
        - name: timezone              # том 'timezone'...
          hostPath:                     # использует каталог (или файл) на хосте узла
            path: /etc/timezone           # путь к файлу '/etc/timezone' на хосте
        - name: localtime             # том 'localtime'...
          hostPath:                     # использует каталог (или файл) на хосте узла
            path: /etc/localtime          # путь к файлу '/etc/localtime' на хосте
```

Применим манифест:
```bash
sudo kubectl apply -f ~/k3s/gitea/gitea-deployment.yaml
```

Проверим, что под создан и работает:
```bash
sudo kubectl get pod -n gitea -o wide
```

Увидим что-то вроде:
```text
NAME                     READY   STATUS      RESTARTS   AGE   IP           NODE         NOMINATED NODE   READINESS GATES
gitea-55dfcf4dd9-d99nw   1/1     Running     0          74s   10.42.1.94   opi5plus-3   <none>           <none>
gitea-init-data          0/1     Completed   0          2h    10.42.2.64   opi5plus-1   <none>           <none>
 ```

Как видим:
* под `gitea` работает и запущен на узде `opi5plus-3` (там где у нас SSD). В принципе, аффинити не гарантирует запуск
  пода на узле с SSD, это только предпочтение. Но в данном случае предпочтение сработало.
* под `gitea-init-data` завершился. Это нормально, так как он был создан только для переноса данных gitea в Longhorn
  PVC `gitea-pvc`, и должен быть "жив" только 1 час (времени, которое мы указали в `sleep 3600` в манифесте пода).

Проверим логи пода `gitea`:
```bash
sudo kubectl logs -n gitea deployment/gitea
```

Увидим что-то вроде:
```text
Server listening on :: port 22.
Server listening on 0.0.0.0 port 22.
2025/04/18 17:42:33 cmd/web.go:253:runWeb() [I] Starting Gitea on PID: 17
2025/04/18 17:42:33 cmd/web.go:112:showWebStartupMessage() [I] Gitea version: 1.23.7 built with GNU Make 4.4.1, go1.23.8 : bindata, timetzdata, sqlite, sqlite_unlock_notify
2025/04/18 17:42:33 cmd/web.go:113:showWebStartupMessage() [I] * RunMode: prod
2025/04/18 17:42:33 cmd/web.go:114:showWebStartupMessage() [I] * AppPath: /usr/local/bin/gitea
2025/04/18 17:42:33 cmd/web.go:115:showWebStartupMessage() [I] * WorkPath: /data/gitea
2025/04/18 17:42:33 cmd/web.go:116:showWebStartupMessage() [I] * CustomPath: /data/gitea
2025/04/18 17:42:33 cmd/web.go:117:showWebStartupMessage() [I] * ConfigFile: /data/gitea/conf/app.ini
...
...
...
2025/04/18 17:42:34 cmd/web.go:205:serveInstalled() [I] PING DATABASE sqlite3
...
...
...
2025/04/18 17:42:39 cmd/web.go:315:listen() [I] Listen: http://0.0.0.0:3000
2025/04/18 17:42:39 cmd/web.go:319:listen() [I] AppURL(ROOT_URL): https://git.cube2.ru/
2025/04/18 17:42:39 cmd/web.go:322:listen() [I] LFS server enabled
2025/04/18 17:42:39 ...s/graceful/server.go:50:NewServer() [I] Starting new Web server: tcp:0.0.0.0:3000 on PID: 17
```

Как видим, Gitea запустилась, порт 3000 (HTTP) и 22 (SSH) открыты, тома подключены, база данных SQLite на месте, права
на папку `/data` установлены. Всё работает.

Можно устроить более глубокую проверку, зайдя внутр пода и посмотреть как там всё устроено:
```bash
sudo kubectl exec -n gitea -it deployment/gitea -- /bin/bash
```

Внутри пода проверим, что PVC `gitea-pvc` подключен и данные gitea и база на месте:
```bash
ls -la /data
ls -l /data/gitea/gitea.db
```

Проверим открытые порты в поде:
```bash
netstat -tuln
```

Увидим что порты слушаются:
```text
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      
tcp        0      0 :::3000                 :::*                    LISTEN      
tcp        0      0 :::22                   :::*                    LISTEN
```    

Проверим, что web-интерфейс Gitea доступен:
```bash
curl -v http://localhost:3000
```

Увидим что-то вроде:
```text
 Host localhost:3000 was resolved.
* IPv6: ::1
* IPv4: 127.0.0.1
*   Trying [::1]:3000...
* Connected to localhost (::1) port 3000
* using HTTP/1.x
> GET / HTTP/1.1
> Host: localhost:3000
> User-Agent: curl/8.12.1
> Accept: */*
> 
* Request completely sent off
< HTTP/1.1 303 See Other
< Cache-Control: max-age=0, private, must-revalidate, no-transform
< Content-Type: text/html; charset=utf-8
...
...
...
```

Как видим, web-интерфейс Gitea доступен внутри пода по http на порту 3000. Все работает. Можно выходить из пода:
```bash
exit
```

## Создание сервиса и IngressRoute для доступа к Gitea снаружи



```bash
nano ~/k3s/gitea/gitea-service.yaml
```

Вставляем в него следующее содержимое:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: gitea
  namespace: gitea
spec:
  selector:
    app: gitea
  ports:
    - name: http
      port: 80
      targetPort: 3000
    #- name: ssh
    #  port: 22
    #  targetPort: 222
```
Объяснение:

selector: app: gitea — находит поды из Deployment gitea.
port: 80 — внешний порт сервиса (Traefik будет слать трафик сюда).
targetPort: 3000 — порт контейнера Gitea.

Применим манифест:
```bash
sudo kubectl apply -f ~/k3s/gitea/gitea-service.yaml
```

sudo kubectl get svc -n gitea -o wide
NAME    TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE    SELECTOR
gitea   ClusterIP   10.43.211.8   <none>        80/TCP    115s   app=gitea

nano ~/k3s/gitea/https-redirect-middleware.yaml

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
name: https-redirect
namespace: gitea
spec:
redirectScheme:
    scheme: https
    permanent: true
```

Объяснение:

redirectScheme: scheme: https — перенаправляет запросы на HTTPS.
permanent: true — возвращает 301 (постоянный редирект) для SEO и кэширования.
Размещаем в gitea, чтобы не затрагивать другие сервисы.

Применить:

```bash
sudo kubectl apply -f ~/k3s/gitea/https-redirect-middleware.yaml
```



У нас уже настроена выдача сертификатов Let’s Encrypt в подах cert-managerб cert-manager-cainjector и
cert-manager-webhook, в пронстве имен cert-manager. Это нестандартный способ (см. [заметку о cert-manager](k3s-cert-manager.md)).

Проверим, что cert-manager работает:
```bash
sudo kubectl get pods -n cert-manager
```
Увидим что-то вроде:
```text
NAME                                       READY   STATUS    RESTARTS        AGE   IP           NODE         NOMINATED NODE   READINESS GATES
cert-manager-64478b89d5-p4msl              1/1     Running   2 (5d13h ago)   19d   10.42.1.55   opi5plus-3   <none>           <none>
cert-manager-cainjector-65559df4ff-t7rj4   1/1     Running   5 (5d13h ago)   19d   10.42.1.54   opi5plus-3   <none>           <none>
cert-manager-webhook-544c988c49-zxdxc      1/1     Running   0               19d   10.42.1.56   opi5plus-3   <none>           <none>
```

Поды `cert-manager`, `cainjector`, `webhook` должны быть **Running**.

Проверим наличие ClusterIssuer:
```bash
sudo kubectl get clusterissuer -A -o wide
```

Увидим что-то вроде:
```text
NAME               READY   STATUS                                                 AGE
letsencrypt-prod   True    The ACME account was registered with the ACME server   19d
```

Проверим, что работает и Let's Encrypt знает о нас:
```bash
sudo kubectl describe clusterissuer letsencrypt-prod
```

Увидим что-то вроде:
```text
Name:         letsencrypt-prod
Namespace:    
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         ClusterIssuer
...
...
...
Status:
  Acme:
    Last Private Key Hash:  тут-будет-хэш-вашего-ключа=
    Last Registered Email:  тут-будет-ваш-email
    Uri:                    https://acme-v02.api.letsencrypt.org/acme/acct/тут-будет-id-вашего-аккаунта
  Conditions:
    ...
    ...
    Status:                True
    Type:                  Ready
Events:                    <none>
```

Важно чтобы `Status: Conditions: Ready:` был `True`.



Создадим манифест для получения сертификата Let's Encrypt:
```bash
nano ~/k3s/gitea/gitea-certificate.yaml
```

и вставим в него следующее содержимое:
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: gitea-tls
  namespace: gitea
spec:
  secretName: gitea-tls
  dnsNames:
    - git.cube2.ru
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

secretName: gitea-tls: Сертификат сохраняется в Secret gitea-tls в gitea.
dnsNames: Домен git.cube2.ru.
issuerRef: Ссылается на ClusterIssuer letsencrypt-prod.

Применим манифест:
```bash
sudo kubectl apply -f ~/k3s/gitea/gitea-certificate.yaml
```

Проверим, что секрет создан:
```bash
sudo kubectl get secret -n gitea gitea-tls -o wide
```

Увидим что-то вроде:
```text
NAME        TYPE                DATA   AGE
gitea-tls   kubernetes.io/tls   2      46s
```

Проверим, что сертификат выдан:
```bash
sudo kubectl describe certificate -n gitea gitea-tls
```

Увидим что-то вроде:
```text
Name:         gitea-tls
Namespace:    gitea
...
...
...
Spec:
  Dns Names:
    тут-будет-ваш-домен
  ...
  ...
Status:
  Conditions:
    Last Transition Time:  тут-будет-дата-время-выдачи-сертификата
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               тут-будет-дата-время-окончания-действия-сертификата
  Not Before:              тут-будет-дата-время-начала-действия-сертификата
  Renewal Time:            тут-будет-дата-время-ближайшего-обновления-сертификата
  Revision:                1
Events:
  ...
  ...
```

Ожидается `Status: True` в `Conditions`, свежие даты в `Not After` и `Not Before` и сообщение `Certificate is
 up to date and has not expired` в `Message`.




Создать IngressRoute для HTTPS
Настроим IngressRoute для маршрутизации git.cube2.ru через HTTPS с Let’s Encrypt и редиректом HTTP.
```bash
nano ~/k3s/gitea/gitea-ingressroute.yaml
```

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: gitea-http
  namespace: gitea
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`git.cube2.ru`)
      kind: Rule
      services:
        - name: gitea
          port: 80
      middlewares:
        - name: https-redirect
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: gitea-https
  namespace: gitea
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`git.cube2.ru`)
      kind: Rule
      services:
        - name: gitea
          port: 80
  tls:
    secretName: gitea-tls
    # Если у вас стандартная настройка Traefik, то раскомментируйте следующую строку (а строку выше закомментируйте)
    # certResolver: letsencrypt
```
Объяснение:

Первая IngressRoute (gitea-http):
entryPoints: web — слушает порт 80 (HTTP).
match: Host(git.cube2.ru) — обрабатывает запросы к git.cube2.ru.
services: gitea — направляет трафик в Service gitea (порт 80 → 3000).
middlewares: https-redirect — редирект на HTTPS.
Вторая IngressRoute (gitea-https):
entryPoints: websecure — слушает порт 443 (HTTPS).
tls: certResolver: letsencrypt — включает Let’s Encrypt для автоматического получения сертификата.
Трафик идёт в тот же Service gitea.

Применим манифест:
```bash
sudo kubectl apply -f ~/k3s/gitea/gitea-ingressroute.yaml
```







Можно удалить временный под:
```bash
sudo kubectl delete pod gitea-init-data -n gitea
```
