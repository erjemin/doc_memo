# Подключение менеджера сертификатов (cert-manager) Let's Encrypt к k3s

№ Установка cert-manager

Установим `cert-manager` из официального манифеста GitHub. Это автоматически добавит все необходимые компоненты,
включая CRD (Custom Resource Definitions) и контроллеры:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.3/cert-manager.yaml
```

Это установит cert-manager версии 1.15.3 (предварительно надо проверь актуальную версию на GitHub cert-manager).
Компоненты развернутся в namespace cert-manager.

Проверим установку:
```bash
kubectl get pods --namespace cert-manager
```

Увидим, что все поды в статусе `Running`:
```text
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-778cb8f45-gxtpf               1/1     Running   0          22m
cert-manager-cainjector-6d5854cdd8-w4d8x   1/1     Running   0          22m
cert-manager-webhook-78b57fb6bb-jljdx      1/1     Running   0          22m
```

Что это за поды:
- `cert-manager` — основной контроллер cert-manager, который управляет созданием и обновлением сертификатов.
- `cert-manager-cainjector` — контроллер, который автоматически добавляет CA-сертификаты в секреты и ресурсы Kubernetes.
- `cert-manager-webhook` — контроллер, который обрабатывает запросы на валидацию и мутацию ресурсов cert-manager.

Проверим, что Custom Resource Definitions (CRDs) для cert-manager созданы:
```bash
kubectl get crd | grep cert-manager
```

Должны быть `certificates.cert-manager.io`, `issuers.cert-manager.io`, `clusterissuers.cert-manager.io   и другие:
```text
certificaterequests.cert-manager.io         2025-04-27T16:29:56Z
certificates.cert-manager.io                2025-04-27T16:29:56Z
challenges.acme.cert-manager.io             2025-04-27T16:29:56Z
clusterissuers.cert-manager.io              2025-04-27T16:29:56Z
issuers.cert-manager.io                     2025-04-27T16:29:56Z
orders.acme.cert-manager.io                 2025-04-27T16:29:57Z
```

# Создание ClusterIssuer (Эмитент сертификатов для всего кластера)

Создадим `ClusterIssuer` для Let's Encrypt. Это позволит cert-manager автоматически запрашивать и обновлять сертификаты.
Разместим его в отдельном файле `~/k3s/cert-manager/letsencrypt-clusterissuer.yaml`:
```bash
mkdir -p ~/k3s/cert-manager
nano ~/k3s/cert-manager/letsencrypt-clusterissuer.yaml
```

И вставим в него следующий код (не забудьте заменить `<my@email.com>` на свой email):
```yaml
apiVersion: cert-manager.io/v1  # Версия API для cert-manager, v1 — текущая стабильная.
kind: ClusterIssuer             # Тип ресурса. ClusterIssuer — глобальный объект для выдачи сертификатов
                                # всему кластеру (в отличие от Issuer, который ограничен namespace).
metadata:
  name: letsencrypt-prod        # Имя `ClusterIssuer`, на него будем ссылаться в манифестах (например, в Certificate).
spec:
  acme:                         # Используется ACME-протокол (Automated Certificate Management Environment) Let's Encrypt.
    server: https://acme-v02.api.letsencrypt.org/directory  # URL сервера Let's Encrypt (боевой).
    email: <my@email.com>       # Логин-клиент Let's Encrypt (раньше так же использовался для уведомлений).
    privateKeySecretRef:
      name: letsencrypt-prod    # Имя секрета, где будет храниться приватные ключи для ACME-аккаунта.
    solvers:                    # Список способов "решения" ACME-задач для подтверждения владения доменом.
    - http01:                   # Используется HTTP-01 challenge (подтверждение через HTTP-запросы).
        ingress:
          class: traefik        # Указывает, что ingress-контроллер Traefik будет обрабатывать HTTP-01 challenge
                                # (создает временные маршруты для `/.well-known/acme-challenge`) для подтверждения
                                # владения доменом.
```

Применим манифест:
```bash
kubectl apply -f ~/k3s/cert-manager/letsencrypt-clusterissuer.yaml
```

Проверим, что `ClusterIssuer` создан:
```bash
kubectl get clusterissuer
```

Увидим что-то вроде:
```text
NAME               READY   AGE
letsencrypt-prod   True    27s
```

Проверим статус:
```bash
kubectl describe clusterissuer letsencrypt-prod
```

Увидим что-то вроде:
```text
...
...
Status:
  Acme:
    Last Private Key Hash:  тут-хранится-хэш-приватного-ключа
    Last Registered Email:  <my@email.com>
    Uri:                    https://acme-v02.api.letsencrypt.org/acme/acct/2365771147
  Conditions:
    Last Transition Time:  2025-04-27T17:00:56Z
    Message:               The ACME account was registered with the ACME server
    Observed Generation:   1
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
...
```

Должны быть `Status: True` и `Reason: ACMEAccountRegistered`, что означает, что `ClusterIssuer` успешно зарегистрирован
и готов к использованию.

# Создание сертификата

Эта общая часть, объясняющая как создавать манифесты для выпуска сертификатов.

В своем манифесте, например, в манифесте пода, создается блок (данный кусочек взят из манифеста пода
[Gitea](k3s-migrating-container-from-docker-to-kubernetes.md) и он находится в пространстве имен `gitea`):
```yaml
---
# Манифест для получения сертификата от Let's Encrypt
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: gitea-tls     # Имя сертификата `gitea-tls`
  namespace: gitea    # В пространстве имен `gitea`
spec:
  secretName: gitea-tls   # Имя секрета для сертификата
  dnsNames:                   # Доменные имена для сертификата...
    - git.cube2.ru
  issuerRef:                  # Эмитент сертификата (Issuer)
    name: letsencrypt-prod      # Имя ClusterIssuer, который отвечает за получение сертификата: `letsencrypt-prod`
    kind: ClusterIssuer         # Тип эмитента
```

После применения манифеста, можно проверить статус сертификата:
```bash
kubectl get certificate -A
```

Увидим что-то вроде:
```text
NAMESPACE   NAME                 READY   SECRET               AGE
ab-shelf    audiobookshelf-tls   False   audiobookshelf-tls   90m
gitea       gitea-tls            True    gitea-tls            66s
```

Тут можно заметить, что сертификат `audiobookshelf-tls` не готов (False), а `gitea-tls` готов (True).

Как проверить статус сертификата и причину его неготовности (на примере `audiobookshelf-tls`):
```bash
kubectl describe certificate audiobookshelf-tls -n ab-shelf
```

Увидим что-то вроде:
```text
Name:         audiobookshelf-tls
Namespace:    ab-shelf
...
...
...
Spec:
  Dns Names:
    <тут-будет-доменное-имя-для-которого-выдается-сертификат>
  Issuer Ref:
    Kind:       ClusterIssuer
    Name:       letsencrypt-prod
  Secret Name:  audiobookshelf-tls
Status:
  Conditions:
    Last Transition Time:    2025-04-27T17:13:32Z
    Message:                 The certificate request has failed to complete and will be retried: Failed to wait for order resource "audiobookshelf-tls-1-722111375" to become ready: order is in "errored" state: Failed to create Order: 429 urn:ietf:params:acme:error:rateLimited: too many certificates (5) already issued for this exact set of domains in the last 168h0m0s, retry after 2025-04-28 19:30:09 UTC: see https://letsencrypt.org/docs/rate-limits/#new-certificates-per-exact-set-of-hostnames
    Observed Generation:     1
    Reason:                  Failed
    Status:                  False
    Type:                    Issuing
    Last Transition Time:    2025-04-27T17:13:29Z
    Message:                 Issuing certificate as Secret does not exist
    Observed Generation:     1
    Reason:                  DoesNotExist
    Status:                  False
    Type:                    Ready
...
...
...
```

Как видно, причина неготовности сертификата `audiobookshelf-tls` в том, что уже превышен лимит на количество сертификатов
(5) для данного доменного имени за последние 168 часов (7 дней). Указано время после которого можно повторить запрос.

По идее запрос на повторный сертификат будет отправлен автоматически, но это может произойти спустя несколько часов
после разрешенного времени (учтите что время указывается в UTC, делайте поправку по своему локальному времени).

Чтобы ускорить процесс, можно удалить сертификат и создать его заново (на примере `audiobookshelf-tls`):
```bash
kubectl delete certificate audiobookshelf-tls -n ab-shelf
```

А затем повторно принять манифест, в котором у вас находится `kind: Certificate`

| Заметка                                                                                                                                                                                                                |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Пока Let's Encrypt не выдал сертификат, Traefik будет работать по SSL (https) на самоподписанном сертификате. Можно открыть анонимное окно браузера, согласится с предупреждениями безопасности и пользоваться сайтом. |
|                                                                                                                                                                                                                        |

Когда же все хорошо  (на примере `gitea-tls` для домены `git.cube2.ru`, сайт которого вы сейчас читаете):
```bash
kubectl describe certificate gitea-tls -n gitea
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
    git.cube2.ru
  Issuer Ref:
    Kind:       ClusterIssuer
    Name:       letsencrypt-prod
  Secret Name:  gitea-tls
Status:
  Conditions:
    Last Transition Time:  2025-04-27T18:43:02Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2025-07-26T17:44:29Z
  Not Before:              2025-04-27T17:44:30Z
  Renewal Time:            2025-06-26T17:44:29Z
...
...
...
```

Видим что `Status: True`, `Reason: Ready`, а также время дату/время с которого сертификат действителен
(время `Not Before`) и до которого он действителен (время `Not After`), а также дату/время, когда сертификат
будет автоматически обновлен (время `Renewal Time`). **ВАЖНО**: _время указывается в UTC, делайте поправку на свой
часовой пояс_.