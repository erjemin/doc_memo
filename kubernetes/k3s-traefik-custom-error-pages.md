# Кастомная страница ошибки 404 (и других) в Traefik

Страницы ошибок Traefik по умолчанию выглядят скучно. Это даже не страницы, а просто текстовые сообщения. Например,
404 выглядит как `404 page not found`. Это позволяет Traefik быть лёгким и быстрым.

Если хочется сделать страницы статусоы 4xx и 5xx более привлекательными, то кастомизация страниц ошибок — отличная идея!
Для каждого HTTP-сервиса внутри можно сделать свои страницы ошибок на уровне приложения (как например, на Gitea,
на которой ты сейчас сидишь). И это наиболее правильный способ. Но если http-запрос не привязан ни к какому сервису,
и Traefik не знает куда его отправить, то он выдаёт свои страницы ошибок. Например, при обращении по IP. 


Traefik позволяет кастомизировать страницы ошибок через **middleware** типа `errors`, который перенаправляет запросы
с определёнными кодами ошибок (например, 404) на кастомный сервис, возвращающий нужную html-страницу. И все это
излишество в `k3s` нужно настраивать глобально, с помощью `Middleware` и `IngressRoute`, и применять ко всем маршрутам.

Чтобы кастомная страница 404 работала для всех запросов в кластере, нужно:

# Создать сервис, который возвращает кастомную страницу (например, контейнер с Nginx или простой HTTP-сервер).
# Настроить middleware `errors` для перехвата ошибок 404.
# Применить middleware глобально через `IngressRoute` или конфигурацию Traefik.

Самый простой подход — развернуть лёгкий контейнер (например, Nginx) с HTML-файлом для страницы 404 и настроить
Traefik для перенаправления ошибок на этот контейнер. Ну или, как альтернатива, использовать внешний сервис (по URL),
но это сложнее для глобальной настройки, и создаст зависимость от этого URL.

#### 2. План действий
- Создать кастомную страницу 404 (HTML-файл).
- Развернуть контейнер с Nginx, который будет отдавать эту страницу.
- Настроить Traefik middleware `errors` для перехвата 404.
- Применить middleware глобально для всех маршрутов в `k3s`.
- Проверить результат.

---

### Настройка кастомной страницы 404

#### 1. Создать кастомную страницу 404
- Создай HTML-файл для страницы 404 на ноде `opi5`:
  ```bash
  mkdir -p ~/k3s/error-pages
  cat > ~/k3s/error-pages/404.html <<EOF
  <!DOCTYPE html>
  <html>
  <head>
      <title>404 Not Found</title>
      <style>
          body { font-family: Arial, sans-serif; text-align: center; padding: 50px; }
          h1 { color: #ff5555; }
          p { font-size: 18px; }
      </style>
  </head>
  <body>
      <h1>404 - Page Not Found</h1>
      <p>Oops! Looks like you're lost in the void.</p>
  </body>
  </html>
EOF
  ```

#### 2. Развернуть Nginx для отдачи страницы
- Создай манифест для Nginx, который будет отдавать `404.html`:
  ```bash
  cat > ~/k3s/error-pages/error-pages.yaml <<EOF
  apiVersion: v1
  kind: Namespace
  metadata:
    name: error-pages
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: error-pages
    namespace: error-pages
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: error-pages
    template:
      metadata:
        labels:
          app: error-pages
      spec:
        containers:
        - name: nginx
          image: nginx:alpine
          ports:
          - containerPort: 80
          volumeMounts:
          - name: error-pages
            mountPath: /usr/share/nginx/html
        volumes:
        - name: error-pages
          configMap:
            name: error-pages
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: error-pages
    namespace: error-pages
  spec:
    selector:
      app: error-pages
    ports:
    - port: 80
      targetPort: 80
  ---
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: error-pages
    namespace: error-pages
  data:
    404.html: |
      <!DOCTYPE html>
      <html>
      <head>
          <title>404 Not Found</title>
          <style>
              body { font-family: Arial, sans-serif; text-align: center; padding: 50px; }
              h1 { color: #ff5555; }
              p { font-size: 18px; }
          </style>
      </head>
      <body>
          <h1>404 - Page Not Found</h1>
          <p>Oops! Looks like you're lost in the void.</p>
      </body>
      </html>
EOF
  ```

- Примени:
  ```bash
  kubectl apply -f ~/k3s/error-pages/error-pages.yaml
  ```

- Проверь поды и сервис:
  ```bash
  kubectl get pod -n error-pages
  kubectl get svc -n error-pages
  ```

#### 3. Настроить Traefik middleware для ошибок
- Создай манифест для middleware `errors`:
  ```bash
  cat > ~/k3s/traefik/error-middleware.yaml <<EOF
  apiVersion: traefik.io/v1alpha1
  kind: Middleware
  metadata:
    name: error-pages
    namespace: kube-system
  spec:
    errors:
      status:
        - "404"
      service:
        name: error-pages
        namespace: error-pages
        port: 80
      query: /404.html
EOF
  ```

- Примени:
  ```bash
  kubectl apply -f ~/k3s/traefik/error-middleware.yaml
  ```

- Проверь:
  ```bash
  kubectl get middleware -n kube-system
  ```

#### 4. Применить middleware глобально
- В `k3s` дефолтный Traefik обрабатывает маршруты через `IngressRoute` или `Ingress`. Чтобы middleware применялся ко всем маршрутам, нужно либо:
  - Добавить middleware к каждому `IngressRoute` вручную.
  - Настроить Traefik для глобального применения middleware через `defaultMiddlewares`.

- Для простоты создадим глобальный `IngressRoute` для всех доменов:
  ```bash
  cat > ~/k3s/traefik/global-error-route.yaml <<EOF
  apiVersion: traefik.io/v1alpha1
  kind: IngressRoute
  metadata:
    name: global-error-route
    namespace: kube-system
  spec:
    entryPoints:
      - web
      - websecure
    routes:
      - match: HostRegexp(`{host:.+}`)
        kind: Rule
        services:
          - name: noop
            namespace: kube-system
            port: 9999
        middlewares:
          - name: error-pages
            namespace: kube-system
EOF
  ```

- Примени:
  ```bash
  kubectl apply -f ~/k3s/traefik/global-error-route.yaml
  ```

- **Примечание**:
  - Сервис `noop:9999` — это заглушка, так как `IngressRoute` требует сервис, но middleware `errors` перехватит 404 до обращения к сервису.
  - Это обеспечивает, что любой запрос с кодом 404 (для любого домена) будет перенаправлен на `error-pages`.

#### 5. Проверить кастомную страницу 404
- Попробуй открыть несуществующий путь:
  ```bash
  curl -v https://git.cube2.ru/nonexistent
  ```

- Ожидаемый ответ:
  - Код: `404 Not Found`.
  - HTML-страница:
    ```html
    <h1>404 - Page Not Found</h1>
    <p>Oops! Looks like you're lost in the void.</p>
    ```

- Проверь для другого домена (например, `Bitwarden`):
  ```bash
  curl -v https://<bitwarden-domain>/nonexistent
  ```

- Проверь логи Traefik:
  ```bash
  kubectl logs -n kube-system -l app.kubernetes.io/name=traefik | tail -n 20
  ```

#### 6. (Опционально) Настроить другие ошибки
- Чтобы добавить кастомные страницы для других кодов (например, 403, 500), обнови middleware:
  ```bash
  cat > ~/k3s/traefik/error-middleware.yaml <<EOF
  apiVersion: traefik.io/v1alpha1
  kind: Middleware
  metadata:
    name: error-pages
    namespace: kube-system
  spec:
    errors:
      status:
        - "403"
        - "404"
        - "500-503"
      service:
        name: error-pages
        namespace: error-pages
        port: 80
      query: /{status}.html
EOF
  ```

- Создай дополнительные файлы (`403.html`, `500.html`) в `ConfigMap`:
  ```bash
  kubectl edit configmap -n error-pages error-pages
  ```
  - Добавь, например:
    ```yaml
    403.html: |
      <!DOCTYPE html>
      <html>
      <head><title>403 Forbidden</title></head>
      <body><h1>403 - Forbidden</h1><p>Access denied!</p></body>
      </html>
    ```

- Примени:
  ```bash
  kubectl apply -f ~/k3s/traefik/error-middleware.yaml
  kubectl delete pod -n error-pages -l app=error-pages
  ```

---

### Ответ на твой вопрос
> По умолчанию Traefik выдаёт скучные текстовые страницы для 404 и других ошибок. Как сделать кастомные 404 для всего `k3s` без привязки к домену? Где они лежат, и есть ли простой способ их переопределить?

- **Где лежат дефолтные страницы**:
  - Они встроены в бинарник Traefik и генерируются как текст (не HTML), например: `404 page not found`.
  - Физически их нет в виде файлов в кластере.

- **Почему не HTML**:
  - Traefik использует текстовые ответы для минимизации ресурсов.

- **Как переопределить**:
  - Использовать middleware `errors`, который перенаправляет ошибки (например, 404) на кастомный сервис с HTML-страницей.
  - Развернуть контейнер (например, Nginx) с кастомной страницей.
  - Настроить глобальный `IngressRoute` для применения middleware ко всем доменам.

- **Простой способ**:
  1. Создать `ConfigMap` с HTML-файлом (`404.html`).
  2. Развернуть Nginx в namespace `error-pages` для отдачи страницы.
  3. Настроить middleware `errors` для перехвата 404.
  4. Применить middleware через глобальный `IngressRoute`.

- **Для всего `k3s`**:
  - Глобальный `IngressRoute` с `HostRegexp` перехватывает все запросы и применяет middleware `errors` для ошибок 404.

---

### Рекомендации
1. Создать и применить страницу 404:
   ```bash
   kubectl apply -f ~/k3s/error-pages/error-pages.yaml
   ```

2. Настроить middleware:
   ```bash
   kubectl apply -f ~/k3s/traefik/error-middleware.yaml
   ```

3. Применить глобальный маршрут:
   ```bash
   kubectl apply -f ~/k3s/traefik/global-error-route.yaml
   ```

4. Проверить:
   ```bash
   curl -v https://git.cube2.ru/nonexistent
   curl -v https://<bitwarden-domain>/nonexistent
   ```

5. Проверить логи:
   ```bash
   kubectl logs -n kube-system -l app.kubernetes.io/name=traefik | tail -n 20
   ```

6. (Опционально) Добавить другие ошибки:
   - Обновить `ConfigMap` и middleware для 403, 500 и т.д.

---

### Итог
Дефолтные страницы ошибок Traefik — это встроенные текстовые ответы, которые можно переопределить с помощью middleware `errors` и кастомного сервиса (например, Nginx с HTML-страницей). Для глобальной настройки в `k3s` мы развернули контейнер с `404.html`, настроили middleware для перехвата ошибок 404, и применили его ко всем доменам через `IngressRoute` с `HostRegexp`. Это простой и универсальный способ сделать страницы ошибок яркими и весёлыми! 😄 Теперь твои 404 будут выглядеть стильно, и ты можешь добавить такие же для других ошибок.

**Действия**:
1. Применить:
   ```bash
   kubectl apply -f ~/k3s/error-pages/error-pages.yaml
   kubectl apply -f ~/k3s/traefik/error-middleware.yaml
   kubectl apply -f ~/k3s/traefik/global-error-route.yaml
   ```
2. Проверить:
   ```bash
   curl -v https://git.cube2.ru/nonexistent
   ```

**Напиши**:
1. Получилась ли кастомная страница 404? (`curl -v https://git.cube2.ru/nonexistent`)
2. Работает ли для других доменов? (`curl -v https://<bitwarden-domain>/nonexistent`)
3. Хочешь настроить страницы для других ошибок (403, 500)?

Теперь можно расслабиться и наслаждаться яркими страницами ошибок! 🚀