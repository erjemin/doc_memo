# Развертывание k3s на  Orange Pi 

K3s — это облегчённая версия Kubernetes, созданная для слабых или малых серверов (Raspberry Pi, Orange Pi,
IoT-устройства, edge-серверы и т.п.). Для кластера из нескольких Orange Pi он предпочтительнее, так как:

* K3S менее требователен к ресурсам (Полный k8s на ARM может сожрать 1-2 ГБ только на управление кластером,
  а k3s занимает ~500 МБ.
* K3s проще устанавливать и обновлять. Shell-скрипт с [https://get.k3s.io](get.k3s.io) все сделает сам, и не нужно 
  погружаться сложные настройки kubeadm. Обычный Kubernetes состоит из множества компонентов: kube-apiserver,
  kube-controller-manager, kube-scheduler, kubelet на каждой ноде, kube-proxy, etcd и т.д. В K3s всё это 
  упаковано в один бинарник.
* Всё работает "из коробки" благодаря встроенному Flannel (CNI) и не надо вручную настраивать Calico, Weave, Cilium. 
* В отличие от "классического" Kubernetes (например, kubeadm), где мастер-узлы по умолчанию изолированы от рабочих нагрузок с помощью taint'ов (например, NoSchedule), k3s не добавляет такие ограничения автоматически. Это значит:
* Для моего проекта особо важно, что из коробки мастер-узел(ы)) в k3s является "гибридным" и выполняет одновременно 
  функции управления (control-plane) и может запускать обычные поды, как воркер. Компоненты управления (API-сервер,
  контроллеры, etcd) работают как системные сервисы, а для пользовательских подов используется тот же kubelet,
  что и на воркерах.

Но, есть у k3s  и минус для конкретно моего случая -- распределенная база **etcd**, в которой хранится состояния
кластера, нод и подов, в нем заменена SQLite. Это круто для маленьких компьютеров: экономно по памяти и другим ресурсам,
и, что главное, никак не сказывается на производительности (пока узлов меньше 50-80), но означает, что в кластере k3s
может быть только одна мастер-нода. Если мастер-нода упадет, её некому будет заменить и весь кластер умрет.
Мне же надо, чтобы как миниум две (а лучше все) ноды могли быть мастерами, так что я буду делать k3s-кластер
с использованием *etcd*.

### Важное предупреждение

k3s -- это не упрощенная мини-версия Kubernetes, здесь все компоненты упакованы в один бинарник, а значит намного
проще не только добавлять узлы, но и удалять их. Так что если что-то пойдет не так с настройкой узла, просто удалите
и начните заново. Удаление k3s с узла:
```bash 
sudo /usr/local/bin/k3s-uninstall.sh  # На мастерах
sudo /usr/local/bin/k3s-agent-uninstall.sh  # На воркере
```

## Установка k3s на первом узле (мастер)

Некоторые требования к узлам:
* На всех Orange Pi установлена одинаковая версия Ubuntu (например, 22.04 или 24.04).
* Статические IP-адреса узлов (или зрезервированные под MAC-адреса IP в DHCP).
* На уздах открыты порты 6443 (для API), 2379-2380 (для etcd) и 10250 (для kubelet).

 
Установливаем первый мастер:
```bash
curl -sfL https://get.k3s.io | sh -s - server --cluster-init --tls-san=192.168.1.27
```

Здесь:
*  `server` -- значение по умолчанию, устанавливает узел k3s в режиме *мастер* (control-plane). В этом режиме узел
   будет запускать все компоненты управления Kubernetes: API-сервер, контроллер-менеджер, планировщик (scheduler).
   Такой узел отвечает за управление  кластером и может также выполнять рабочие нагрузки (workloads), если
   не настроены ограничения (taints). Если бы мы указали `agent` -- был бы установлен узел k3s в режиме *воркер*-узла.
* `--cluster-init` -- добавляет поддержку высокой доступности (HA -- High Availability) через встроенный `etcd`. Это
   значит, что узел инициализирует новый кластер и готов к тому, чтобы другие мастер-узлы могли к нему подключиться
   (для создания HA-конфигурации).
* `--tls-san=192.168.1.27` -- добавляет IP 192.168.1.27 в сертификаты API-сервера, чтобы другие узлы и клиенты
  могли обращаться к нему по этому адресу.

Проверим, что все k3s запущен:
```bash
sudo service k3s status
```

Увидим что-то типа:
```text
● k3s.service - Lightweight Kubernetes
     Loaded: loaded (/etc/systemd/system/k3s.service; enabled; vendor preset: enabled)
     Active: active (running) since ...
...
...
```

Посмотрим сколько нод в кластере:
```bash
sudo kubectl get nodes
```

И, та-да! Увидим одну ноду:
```text
NAME         STATUS   ROLES                       AGE   VERSION
opi5plus-2   Ready    control-plane,etcd,master   31m   v1.31.5+k3s1
```

Как видим, узел `opi5plus-2` готов к работе и выполняет роли *control-plane*, *etcd* и *master*.


А что там внутри? Посмотрим на поды:
```bash
sudo kubectl get pods -A
```

Целых семь подов (минималистичная установка k3s):
```text
NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE
kube-system   coredns-ccb96694c-tfjwj                   1/1     Running     0          13m
kube-system   helm-install-traefik-crd-bdbgd            0/1     Completed   0          13m
kube-system   helm-install-traefik-mlztm                0/1     Completed   1          13m
kube-system   local-path-provisioner-5cf85fd84d-jwz5n   1/1     Running     0          13m
kube-system   metrics-server-5985cbc9d7-n9dwz           1/1     Running     0          13m
kube-system   svclb-traefik-4f8c2580-jddgz              2/2     Running     0          12m
kube-system   traefik-5d45fc8cc9-t5d58                  1/1     Running     0          12m
```

Тут статус X/Y в выводе kubectl get pods показывает:
* Y — сколько контейнеров должно быть в поде (по спецификации).
* X — сколько из них сейчас работает (running).

Представлены следующие поды:
1. `coredns` — это DNS-сервер для кластера. Он отвечает за разрешение имен внутри Kubernetes (например, чтобы поды
   могли обращаться друг к другу по именам сервисов вроде my-service.default.svc.cluster.local).
2. `helm-install-traefik-crd` -- это временный под (Job), который устанавливает Custom Resource Definitions (CRD)
   для *Traefik* — ingress-контроллера, встроенного в k3s. CRD нужны для управления ingress-ресурсами
   (маршрутизацией HTTP/HTTPS). Этот под — одноразовая задача (Job), а не постоянный сервис. Он запустился, выполнил 
   работу (установил CRD) и завершился. Статус "*Completed*" значит, что он больше не работает.
3. `helm-install-traefik` -- ещё один Job, который устанавливает сам Traefik через Helm-чарт. Этот под развернул 
   основной Traefik-под и завершился.
4. `local-path-provisioner` -- компонент для автоматического создания локальных Persistent Volumes (PV) на узлах. Он
   позволяет подам запрашивать хранилище (например, через PersistentVolumeClaim) без сложной настройки NFS или внешних
   хранилищ. В k3s это встроено для простоты.
5. `metrics-server` -- собирает данные об использовании ресурсов (CPU, память) подов и узлов. Это нужно для команд
   вроде `kubectl top` или для Horizontal Pod Autoscaler (HPA). Установку метрик можно отключить при запуске k3s
   флагом `--disable=metrics-server`.
6. `svclb-traefik` - это под для балансировки нагрузки (Service Load Balancer) для Traefik. В k3s нет встроенного
   облачного балансировщика (как в AWS/GCP), поэтому *svclb* эмулирует его на уровне узла, перенаправляя трафик
   к сервисам типа LoadBalancer. У нас два таких контейнера:
   * один для самой логики балансировки;
   * другой для мониторинга или дополнительной функциональности (например, *keepalived* или аналога) и это зависит
     от реализации в k3s.
7. `traefik` -- сам Traefik, ingress-контроллер, который обрабатывает HTTP/HTTPS трафик кластера и маршрутизирует
    его к соответствующим подам (с динамической конфигурацией нашим) и сервисам по правилам Ingress. Traefik в k3s
    установлен по умолчанию, но его можно отключить при запуске k3s флагом `--disable=traefik` (не будет ни *traefik*,
    ни *svclb*, ни связанных *Helm Jobs*).

Обратите внимание, что, например, под `coredns` получил имя `coredns-ccb96694c-tfjwj`. Имена подов (Pod Names)
в Kubernetes генерируются автоматически на основе правил, чтобы каждый под в кластере имел уникальное имя.
Структура имени -- `<имя-приложения>-<хеш-ревизии>-<случайный-суффикс>`. Впрочем, `<хеш-ревизии>` может отсутствовать,
если под не имеет контроллера репликации (например, Job или CronJob).

Можно проверить, что API нашего узла (кластера) отвечает:
```bash
curl -k https://192.168.1.27
```

Здесь ключ `-k` означает, что мы не проверяем сертификаты (нам важно только, что сервер отвечает). Должны получить
Unauthorized JSON-ответ от API. Что-то вроде:
```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}
```

## Подключение второго узла (мастер)

Для начала, на первой ноде получим токен для подключения нового узла к кластеру:
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Вывод будет что-то вроде `K10...::server:longrandomstring`. Это и есть токен, который нужно будет использовать.

Теперь на втором Orange Pi (например, с IP 192.168.1.28) можно запустить второй мастер-узел (вставим токен
из предыдущего шага):
```bash
curl -sfL https://get.k3s.io | sh -s - server --server https://192.168.1.27:6443 --token <ТОКЕН> --tls-san=192.168.1.28
```
Здесь ключи:
* `--server https://192.168.1.27:6443` -- указывает на API мастер-узла, чтобы наш новый узел мог подключиться к кластеру.
* `--token` — токен аутентификации из предыдущего шага.
* `--tls-san=192.168.1.28` -- добавляет IP нашего второго мастера в сертификаты (для будущих подключений).

Проверим какие теперь ноды в кластере:
```bash
sudo k3s kubectl get nodes
```

Теперь увидим две ноды:
```text
NAME         STATUS   ROLES                       AGE    VERSION
opi5plus-2   Ready    control-plane,etcd,master   2h     v1.31.5+k3s1
opi5plus-3   Ready    control-plane,etcd,master   110s   v1.31.5+k3s1
```

Проверим поды кластера и посмотрим на каких нодах они запущены:
```bash
sudo k3s kubectl get pods -A -o wide
```

И увидим, что на второй ноде запустились те же поды, что и на первой:
```text
NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE   IP          NODE         NOMINATED NODE   READINESS GATES
kube-system   coredns-ccb96694c-tfjwj                   1/1     Running     0          2h    10.42.0.4   opi5plus-2   <none>           <none>
kube-system   helm-install-traefik-crd-bdbgd            0/1     Completed   0          2h    <none>      opi5plus-2   <none>           <none>
kube-system   helm-install-traefik-mlztm                0/1     Completed   1          2h    <none>      opi5plus-2   <none>           <none>
kube-system   local-path-provisioner-5cf85fd84d-jwz5n   1/1     Running     0          2h    10.42.0.3   opi5plus-2   <none>           <none>
kube-system   metrics-server-5985cbc9d7-n9dwz           1/1     Running     0          2h    10.42.0.2   opi5plus-2   <none>           <none>
kube-system   svclb-traefik-4f8c2580-jddgz              2/2     Running     0          2h    10.42.0.7   opi5plus-2   <none>           <none>
kube-system   svclb-traefik-4f8c2580-xzt5d              2/2     Running     0          2m35s 10.42.1.2   opi5plus-3   <none>           <none>
kube-system   traefik-5d45fc8cc9-t5d58                  1/1     Running     0          2h    10.42.0.8   opi5plus-2   <none>           <none>
```

Как видим, у нас появился еще один `svclb-traefik` на второй ноде. Это под -- Service Load Balancer (SLB) для Traefik.
Он эмулирует облачный балансировщик нагрузки (типа AWS ELB), которого нет в локальном окружении вроде Orange Pi.
SLB перенаправляет внешний трафик (например, на порты 80/443) к сервисам типа LoadBalancer внутри кластера.

## Подключение третьего узла (воркера)

Добавление третьего узда в качестве воркера (рабочего узла) мы сделаем временно. Во-первых, чтобы показать как это
делается, а во-вторых, чтобы показать как удалять узел и с какими особенностями это связано. И наконец, в-третьих,
объяснить что такое кворум и почему важно, чтобы в кластере было нечетное количество мастер-узлов.

И так, подключение рабочего узла даже проще, чем мастера. Выполним на нашем новом узле:
```bash
curl -sfL https://get.k3s.io | sh -s - agent --server https://192.168.1.10:6443 --token <ТОКЕН>
```

Здесь ключ:
* `agent` -- устанавливает узел в режиме воркера (worker). Это значит, что узел будет выполнять рабочие нагрузки
  (поды), но не будет управлять кластером (без *control-plane*, *master* и на нем нет реплики *etcd*).

Посмотрим на ноды (команда выполняется на одном из мастер-узлов):
```bash
sudo k3s kubectl get nodes
```

Теперь у нас три ноды, и все они имеют статус *Ready*:
```text
NAME         STATUS   ROLES                       AGE   VERSION
opi5plus-1   Ready    <none>                      96s   v1.31.5+k3s1
opi5plus-2   Ready    control-plane,etcd,master   3h    v1.31.5+k3s1
opi5plus-3   Ready    control-plane,etcd,master   2h    v1.31.5+k3s1
```

Новая нода `opi5plus-1` готова к работе и не имеет ролей, а только выполняет рабочие нагрузки (поды).

Посмотрим на поды:
```bash
opi@opi5plus-2:~$ sudo k3s kubectl get pods -n kube-system -o wide
```

И увидим, что на новом воркере (opi5plus-1) запустился под балансировщика `svclb-traefik`:
```text
NAME                                      READY   STATUS      RESTARTS   AGE   IP          NODE         NOMINATED NODE   READINESS GATES
coredns-ccb96694c-tfjwj                   1/1     Running     0          3h    10.42.0.4   opi5plus-2   <none>           <none>
helm-install-traefik-crd-bdbgd            0/1     Completed   0          3h    <none>      opi5plus-2   <none>           <none>
helm-install-traefik-mlztm                0/1     Completed   1          3h    <none>      opi5plus-2   <none>           <none>
local-path-provisioner-5cf85fd84d-jwz5n   1/1     Running     0          3h    10.42.0.3   opi5plus-2   <none>           <none>
metrics-server-5985cbc9d7-n9dwz           1/1     Running     0          3h    10.42.0.2   opi5plus-2   <none>           <none>
svclb-traefik-4f8c2580-4q7dj              3/3     Running     0          92s   10.42.2.2   opi5plus-1   <none>           <none>
svclb-traefik-4f8c2580-h7b9c              3/3     Running     0          2h    10.42.0.9   opi5plus-2   <none>           <none>
svclb-traefik-4f8c2580-qmzf6              3/3     Running     0          2h    10.42.1.5   opi5plus-3   <none>           <none>
traefik-6c979cd89d-98fk8                  1/1     Running     0          1h    10.42.1.6   opi5plus-3   <none>           <none>
```

Посмотрим состояние сервисов в кластере:
```bash
sudo k3s kubectl get service -n kube-system
```

Увидим, что сервис *traefik* доступен на всех нодах:
```text
NAME             TYPE           CLUSTER-IP      EXTERNAL-IP                              PORT(S)                                     AGE
kube-dns         ClusterIP      10.43.0.10      <none>                                   53/UDP,53/TCP,9153/TCP                      3d
metrics-server   ClusterIP      10.43.248.208   <none>                                   443/TCP                                     3d
traefik          LoadBalancer   10.43.164.48    192.168.1.26,192.168.1.27,192.168.1.28   80:31941/TCP,443:30329/TCP,9000:32185/TCP   3d
```

Можем проверить доступность панели `Traefik` через браузер через IP-адрес нового узла и (в нашем случае `http://192.168.1.26:9000/dashboard/#/`)
и увидим, что балаансировщик работает и перенаправляет трафик и с ноды воркера.

Что ж, теперь у нас есть кластер k3s с тремя нодами: двумя мастерами и одним воркером. Но, как я уже говорил, это не
идеальная конфигурация, так как у нас четное количество мастер-узлов. 