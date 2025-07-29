# Проксирование внешнего хоста через Traefik (Ingress-контроллер)

У меня в домашней сети есть хост с web-сервисом (audiobookshelf), а стандартные web-порты (80 для HTTP и 443 для HTTPS)
на домашнем роутере перенаправлены в кластер k3S (через keepalived). Таким образом, если прокинуть http-трафик этого
хоста через Traefik, то можно будет получить доступ к этому сервису через доменное имя и SSL-сертификат от
Let’s Encrypt.

Для удобства я поместил все манифесты в один файл (вы можете оформить из и как отдельные файлы). Так как хотелось бы
описать как делаются универсальные манифесты, которые можно использовать для проксирования любого сервиса, то
я заменил в нем конкретные значения на "заглушки. Можно взять этот манифест и просто заменить в нем значения на
свои:

* `<PROXIED-HOST>` -- IP-адрес хоста, где работает сервис, который надо проксировать.
* `<PROXIED-PORT>` -- порт, с которого отвечает сервис.
* `<YOU-DOMAIN-NAME>` -- доменное имя, на которое будет проксировать сервис.
* `<NAME-SPACE>` -- пространство имен кластера, в котором будет создан сервис, маршруты, секреты и все необходимое
  для проксирования. Пространство имен -- это логическая группа ресурсов в кластере Kubernetes, которая позволяет
  организовать и изолировать ресурсы. 
* `<SERVICE-NAME>` -- имя сервиса, который будет проксироваться. Это имя, для простоты, будем использоваться
* и в маршрутах, и сертификатах, и в секрете...

## Пространство имен

Чтобы все было аккуратно и сервисы и поды не путались, создадим пространство имен для проксирования конкретного хоста.
Мо хост относится к <SERVICE-NAME>, поэтому назову пространство имен `<NAME-SPACE>`
Например, `<NAME-SPACE>`.
```bash
sudo ubectl create namespace <NAME-SPACE>
```

Проверяем, что пространство создано:
```bash
sudo kubectl get namespace <NAME-SPACE>
```

Увидим, что пространство создано и активно:
```text
NAME           STATUS   AGE
<NAME-SPACE>   Active   54s
```

## Конфигурация всего

Для удобства я объединил манифесты в один файл (но можно и по отдельности). Создаем единый манифест:
```bash
sudo nano ~/k3s/<SERVICE-NAME>/<SERVICE-NAME>.yaml
```

Он состоит из следующих частей:
* `Endpoints` -- указывает на конечную точку, в которую будет проксироваться трафик. В данном случае это
  IP-адрес и порт, на который будет проксироваться запрос.
* `Service` -- создает сервис, который будет использоваться для проксирования запросов к `Endpoints`.
* `Certificate` -- создает сертификат для домена `<YOU-DOMAIN-NAME>`, который будет использоваться для
  шифрования трафика. Сертификат запрашивается через `cert-manager`, который автоматически обновляет его по мере
  необходимости.
* `Middleware` -- создает промежуточное ПО, которое будет использоваться для обработки запросов. В данном случае
  это редирект с HTTP на HTTPS и исключение редиректа для ACME challenge (механизма внешней проверки владения доменом
  со стороны Let’s Encrypt).
* `IngressRoute` -- создает маршрут, который будет использоваться для проксирования запросов к сервису.
  В данном случае это маршруты для HTTP и HTTPS, которые будут обрабатывать запросы на домен `<YOU-DOMAIN-NAME>`.
  Также создается маршрут для ACME challenge, который позволяет cert-manager пройти проверку через порт 80.

Вставляем в манифест следующее содержимое (не забудьте заменить `<PROXIED-HOST>`, `<PROXIED-PORT>`, `<YOU-DOMAIN-NAME>`,
 `<NAME-SPACE>` и `<SERVICE-NAME>` на свои значения):
```yaml
# Endpoints для внешнего хоста <SERVICE-NAME>
# Задаёт IP и порт внешнего сервера, так как <SERVICE-NAME> внешний хост для k3s
apiVersion: v1
kind: Endpoints
metadata:
  name: <SERVICE-NAME>
  namespace: <NAME-SPACE>  # Namespace для <SERVICE-NAME>
subsets:  # Прямо в корне, без spec
  - addresses:
      - ip: <PROXIED-HOST>  # IP Synology, где работает <SERVICE-NAME>
    ports:
      - port: <PROXIED-PORT>  # Порт Synology (HTTP)
        protocol: TCP

---
# Service для маршрутизации трафика от Traefik к внешнему хосту
# Связывает IngressRoute с Endpoints
apiVersion: v1
kind: Service
metadata:
  name: <SERVICE-NAME>
  namespace: <NAME-SPACE>
spec:
  ports:
    - port: <PROXIED-PORT>  # Порт сервиса, на который Traefik отправляет трафик
      targetPort: <PROXIED-PORT>  # Порт на Synology
      protocol: TCP

---
# Middleware для редиректа HTTP → HTTPS
# Применяется к HTTP-запросам для перенаправления на HTTPS
apiVersion: traefik.io/v1alpha1               # версия Traefik v34.2.1+up34.2.0 (Traefik v3.3.6)
# apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: https-redirect
  namespace: <NAME-SPACE>
spec:
  redirectScheme:
    scheme: https  # Перенаправлять на HTTPS
    permanent: true  # Код 301 (постоянный редирект)

---
# Middleware для исключения редиректа на ACME challenge
# Позволяет Let’s Encrypt проверять /.well-known/acme-challenge без HTTPS
apiVersion: traefik.io/v1alpha1               # версия Traefik v34.2.1+up34.2.0 (Traefik v3.3.6)
# apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: no-https-redirect
  namespace: <NAME-SPACE>
spec:
  stripPrefix:
    prefixes:
      - /.well-known/acme-challenge  # Убрать префикс для ACME

---
# IngressRoute для HTTP (порт 80) с редиректом на HTTPS
# Обрабатывает HTTP-запросы и перенаправляет их
apiVersion: traefik.io/v1alpha1               # версия Traefik v34.2.1+up34.2.0 (Traefik v3.3.6)
# apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: <SERVICE-NAME>-http
  namespace: <NAME-SPACE>
spec:
  entryPoints:
    - web  # Порт 80 (стандартный HTTP)
  routes:
    - match: Host("<YOU-DOMAIN-NAME>")  # Запросы к <YOU-DOMAIN-NAME>
      kind: Rule
      services:
        - name: <SERVICE-NAME>  # Сервис <SERVICE-NAME>
          port: <PROXIED-PORT>  # Порт сервиса
      middlewares:
        - name: https-redirect  # Редирект на HTTPS

---
# IngressRoute для HTTPS (порт 443)
# Обрабатывает HTTPS-запросы с TLS
apiVersion: traefik.io/v1alpha1               # версия Traefik v34.2.1+up34.2.0 (Traefik v3.3.6)
# apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: <SERVICE-NAME>-https
  namespace: <NAME-SPACE>
spec:
  entryPoints:
    - websecure  # Порт 443 (HTTPS)
  routes:
    - match: Host("<YOU-DOMAIN-NAME>")
      kind: Rule
      services:
        - name: <SERVICE-NAME>
          port: <PROXIED-PORT>
  tls:
    secretName: <SERVICE-NAME>-tls  # Сертификат от cert-manager

---
# IngressRoute для HTTP-01 challenge (Let’s Encrypt)
# Позволяет cert-manager пройти проверку через порт 80
apiVersion: traefik.io/v1alpha1               # версия Traefik v34.2.1+up34.2.0 (Traefik v3.3.6)
# apiVersion: traefik.containo.us/v1alpha1    # старая версия Traefik 
kind: IngressRoute
metadata:
  name: <SERVICE-NAME>-acme
  namespace: <NAME-SPACE>
spec:
  entryPoints:
    - web  # Порт 80
  routes:
    - match: Host("<YOU-DOMAIN-NAME>") && PathPrefix("/.well-known/acme-challenge")
      kind: Rule
      services:
        - name: <SERVICE-NAME>
          port: <PROXIED-PORT>
      middlewares:
        - name: no-https-redirect  # Не редиректить ACME

---
# Certificate для TLS-сертификата от Let’s Encrypt
# Запрашивает сертификат для <YOU-DOMAIN-NAME> через cert-manager
# ВАЖНО: cert-manager должен быть установлен в кластере
# ВАЖНО: если манифесты принимаются (apply) последовательно, то манифест с сертификатом должен быть последним для
#        избежания исчерпания лимитов Let’s Encrypt (пять запросов в неделю)
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: <SERVICE-NAME>-tls
  namespace: <NAME-SPACE>
spec:
  secretName: <SERVICE-NAME>-tls  # Имя секрета для сертификата
  dnsNames:
    - <YOU-DOMAIN-NAME>  # Домен для сертификата
  issuerRef:
    name: letsencrypt-prod  # ClusterIssuer для Let’s Encrypt
    kind: ClusterIssuer
```

**ВАЖНО**: В манифесте используется letsencrypt-prod для получения сертификата от Let’s Encrypt. Это нестандартный 
ClusterIssuer cert-manager, создание которого описано в [документации](https://cert-manager.io/docs/usage/ingress/#tls-termination)
и [отдельной инструкции](k3s-custom-container-deployment.md#создание-clusterissuer)
(возможно, вам нужно будет создать его отдельно). Если вы используете другой ClusterIssuer, то замените letsencrypt-prod
на имя вашего ClusterIssuer в секции `issuerRef` в манифесте.

## Применяем манифест
```bash
sudo kubectl apply -f ~/k3s/<SERVICE-NAME>/<SERVICE-NAME>.yaml
```

Проверяем, что все ресурсы создались:
```bash
sudo kubectl get all -n <NAME-SPACE>
sudo kubectl get ingressroute -n <NAME-SPACE>
sudo kubectl get middleware -n <NAME-SPACE>
sudo kubectl get service -n <NAME-SPACE>
sudo kubectl get endpoints -n <NAME-SPACE>
```

Увидим что-то вроде:
```text
NAME                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)               AGE
service/<SERVICE-NAME>   ClusterIP   10.43.152.59   <none>        <PROXIED-PORT>/TCP   3h
```

Для IngressRoute:
```text
NAME                   AGE
<SERVICE-NAME>         3h
<SERVICE-NAME>-acme    1h
<SERVICE-NAME>-http    1h
<SERVICE-NAME>-https   1h
```

Для Middleware:
```text
NAME                AGE
https-redirect      1h
no-https-redirect   1h
```

Для Service:
```text
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)              AGE
<SERVICE-NAME>   ClusterIP   10.43.152.59   <none>        <PROXIED-PORT>/TCP   3h
```

Для Endpoints:
```text
NAME             ENDPOINTS                       AGE
<SERVICE-NAME>   <PROXIED-HOST>:<PROXIED-PORT>   3h
```

Проверяем, что сертификат создан:
```bash
sudo kubectl describe certificate -n <NAME-SPACE> <SERVICE-NAME>-tls
sudo kubectl get secret -n <NAME-SPACE>  <SERVICE-NAME>-tls
```

## UPD

Почему-то SSL-сертификат Let's Encrypt через положенный срок не перевыпустился. Для начала выяснилось, что на 
моем роутере перестал работать NAT-loopback, и внутрення проверка Certbot перестала работать. Похоже роутер получил
обновление прошивки, и все сломалось. В результате DNS кластера (coredns) разрешал домен во внешний IP-адрес роутера,
а не во внутренний IP-адрес кластера (VIP через keepalived), а без NAT-loopback запросы к домену не доходили до
кластера.

Как чинил:

Во-первых, руками поменял на всех узлах кластера (включая роутер) в файле `/etc/resolv.conf` IP-адрес DNS.
Обновление испортило и DHCP-сервер на роутере, и он стал раздавать свой IP-адреса как DNS-сервер, вместо моего
домшнего DNS-сервера (работат на Synology NAS).

Во-вторых, в кластере k3s обновил конфигурацию coredns:
```bash
kubectl edit configmap -n kube-system coredns
```

Отчего-то там было навернуты хосты 'hosts /etc/coredns/NodeHosts` внутри `.:53 ` блока, и это мешало. В результате
получалось следующее:
```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
        import /etc/coredns/custom/*.override
    }
    import /etc/coredns/custom/*.server
...
...
...
```

После сохранения конфигурации манифет coredns (т.к. его редактируем пряма в кластере) применяется автоматически.
После пеускаем coredns: 
```bash
kubectl rollout restart deployment -n kube-system coredns
```

Проверяем логи:
```bash
kubectl logs -n kube-system -l k8s-app=kube-dns
```

Проверяем статусы подов CoreDNS:
```shell
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
```

Поды должны быть в статусе `Running`:
```text
NAME                       READY   STATUS    RESTARTS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES
coredns-646d9c4784-wsbbg   1/1     Running   0          58m   10.42.2.127   opi5plus-1   <none>           <none>
coredns-646d9c4784-z5zqv   1/1     Running   0          58m   10.42.0.125   opi5         <none>           <none>
```

И наконец, в-третьих, обновил сертификат Let's Encrypt. Сначала удалил старый сертификат и секрет:
```bash
kubectl delete certificate -n <NAME-SPACE> <SERVICE-NAME>-tls
kubectl delete secret -n <NAME-SPACE> <SERVICE-NAME>-tls
```

И снова применил манифест с сертификатом:
```bash
kubectl apply -f ~/k3s/<SERVICE-NAME>/<SERVICE-NAME>.yaml
```

Проверяем, что сертификат обновился:
```bash
kubectl describe certificate -n <NAME-SPACE> <SERVICE-NAME>-tls
kubectl get secret -n <NAME-SPACE>  <SERVICE-NAME>-tls
```




