# Проксирование внешнего хоста через Traefik (Ingress-контроллер)

У меня в домашней сети есть хост с сервисом (audiobookshelf), то удобно его прокинуть через Traefik, чтобы не открывать
на внешний IP лишних адресов, портов, использовать домен и включить SSL, и управлять всем этим через из единой
точки.

В моем случае:
* `<PROXIED-HOST>` -- IP-адрес хоста, где работает сервис, который надо проксировать.
* `<PROXIED-PORT>` -- порт, с которого отвечает сервис.
* `<YOU-DOMAIN-NAME>` -- доменное имя, на которое будет проксировать сервис.

## Пространство имен

Чтобы все было аккуратно и серисы и поды не путались, создадим пространство имен для проксируемого сервиса.
Например, `ab-shelf`.
```bash
sudo ubectl create namespace ab-shelf
```

Проверяем, что пространство создано:
```bash
sudo kubectl get namespace ab-shelf
```

Увидим, что пространство создано и активно:
```text
NAME       STATUS   AGE
ab-shelf   Active   54s
```

## Конфигурация IngressRoute, Service и Endpoints

Для удобства я объединил манифесты в один файл (но можно и по отдельности). Создаем единый манифетст:
```bash
sudo nano ~/k3s/audiobookshelf/audiobookshelf.yaml
```

И вставляем в него следующее содержимое (не забудь заменить `<PROXIED-HOST>`, `<PROXIED-PORT>` и `<YOU-DOMAIN-NAME>`
на свои значения... пространство имен `ab-shelf` и имя сервиса `audiobookshelf` можно тоже поменять на свое, если
у вас другой сервис):
```yaml
# Endpoints для внешнего хоста
apiVersion: v1
kind: Endpoints
metadata:
  name: audiobookshelf
  namespace: ab-shelf    # пространство, например
subsets:
  - addresses:
      - ip: <PROXIED-HOST>
    ports:
      - port: <PROXIED-PORT>
        protocol: TCP
---

# Service для проксируемого хоста (<PROXIED-HOST>:<PROXIED-PORT>)
apiVersion: v1
kind: Service
metadata:
  name: audiobookshelf
  namespace: ab-shelf    # пространство, например
spec:
  ports:
    - port: <PROXIED-PORT>
      targetPort: <PROXIED-PORT>
      protocol: TCP
---

# IngressRoute
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: audiobookshelf
  namespace: ab-shelf    # пространство, например
spec:
  entryPoints:
    - web-custom         # ендпоинт, который "слушает" порт 2055
  routes:
    - match: Host("<YOU-DOMAIN-NAME>")
      kind: Rule
      services:
        - name: audiobookshelf
          port: <PROXIED-PORT>
```

Что тут происходит:
* `Endpoints` -- указываем, что сервис будет проксировать запросы на внешний хост `<PROXIED-HOST>`
  и порт `<PROXIED-PORT>`.  Ендпоинт -- это конечная точка, к которой будет проксироваться запрос. В данном случае
  это IP-адрес и порт, но обычно это имя пода, на который отправляются и который отвечает на запросы.
* `Service` -- создаем сервис, который будет использоваться для проксирования запросов к `Endpoints`. Сервис -- это
  абстракция, которая позволяет упрощать доступ к подам. Он может использоваться для балансировки нагрузки между
  несколькими потоками, которые обрабатывают запросы. В данном случае мы создаем сервис, который будет проксировать
  запросы к `Endpoints` (внешнему хосту).
* `IngressRoute` -- создаем маршрут, который будет проксировать запросы на домен `<YOU-DOMAIN-NAME>`
  к сервису `audiobookshelf` в пространстве `ab-shelf`. Маршрут -- это правило, которое определяет, как обрабатывать
  запросы, поступающие на определенный адрес. В данном случае мы создаем маршрут, который будет направлять запросы
  на домен `<YOU-DOMAIN-NAME>` к сервису `audiobookshelf` в пространстве `ab-shelf`. Внутри маршрута мы указываем,
  что запросы должны обрабатываться сервисом `audiobookshelf` на порту `<PROXIED-PORT>`. 
  В данном случае мы используем `web-custom` как точку входа, которая будет слушать порт 2055.

