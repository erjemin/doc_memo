# Развертывание пользовательского контейнера в k3s




Теперь нам нужно передать образ контейнера на другие ноды кластера. Для этого сначала извлечём образ из k3s в tar-файл:
```bash
sudo docker save shadowsocks-with-tools:latest -o /tmp/shadowsocks.tar
```

Затем передадим на вторую ноду, в нашем случае на хосте `192.168.1.28`, затем на третью (придется копировать через 
`sudo`, т.к. при создании образа в tar-файл мы тоже использовали `sudo`):
```bash
sudo scp /tmp/shadowsocks.tar opi@192.168.1.28:/tmp/
```

Импортируем на второй ноде этот образ в k3s. Небезопасное использование пароля удаленного хоста в команде, требуется т.к.
в команде через `ssh` не предоставляется интерактивный терминал, а SSH-ключи решают аутентификацию только для входа, но
не для выполнения `sudo` на удалённой машине... c помощью ключа `-S` мы заставляем `sudo` га уделенном хсте 
читать пароль из `stdin`, и потому при введении пароля на текущем хосте он будет виден на экране... это выглядит опасно,
но менее безопасно чем передать его с помощью `echo` в команде `echo 'your_password' |` добавив перед `sido -S k3s ...`.
Так пароль, по крайне мере, не будет сохранён в `bash_history`, что было бы совсем, совсем не безопасно (а экран мы
можем очистить после, выполнив команду `clear`):
```bash
ssh opi@192.168.1.28 "echo 'your_password' | sudo -S k3s ctr images import /tmp/shadowsocks.tar"
```

Увидим что-то вроде:
```text
unpacking docker.io/library/shadowsocks-with-tools:latest (sha256:ae615618ce9d2aac7d3764ef735108452adf3fc30bb65f23f28c345798880c80)...done
```

Удалим временный файл на второй ноде:
```bash
ssh opi@192.168.1.28 "rm /tmp/shadowsocks.tar"
```

И убедимся, что образ появился в k3s на второй ноде (снова используем ключ `-S`):
```bash
ssh opi@192.168.1.28 "sudo -S k3s ctr images ls | grep shadowsocks"
```

**Повторяем операцию для третьей ноды**. И можем удалить временный файл на первой ноде:
```bash
sudo rm /tmp/shadowsocks.tar
```


----
Отлично, что всё получилось! Рад, что мы дошли до этого момента вместе. Твой вопрос про "секретные" контейнеры и необходимость публиковать их в Docker Hub — очень важный и актуальный, особенно для больших кластеров или чувствительных данных. Давай разберём это подробно, чтобы ты понял все варианты и мог выбрать тот, который подходит тебе.

Нужно ли выкладывать "секретный" контейнер в Docker Hub?
Ты прав: если твой контейнер содержит секреты (например, ключи, пароли, защищённые алгоритмы), выкладывать его в публичный реестр вроде Docker Hub — не лучшая идея. Даже если сделать репозиторий приватным на Docker Hub, это всё равно означает передачу твоего образа третьей стороне, что может быть нежелательно с точки зрения безопасности или лицензирования.

Ответ: Нет, тебе не обязательно публиковать контейнер в Docker Hub. Есть несколько способов хранить и распространять образы "у себя", внутри твоей сети, без выхода в интернет. Давай разберём варианты.

Варианты хранения и использования контейнеров
1. Локальное использование (как сейчас)
Как это работает:
Ты уже сделал это: собрал образ shadowsocks-with-tools локально и импортировал его в containerd на каждом узле через k3s ctr images import.
В deployment.yaml указал imagePullPolicy: Never, чтобы k3s использовал только локальный образ.
Плюсы:
Никаких внешних реестров — всё остаётся на твоих машинах.
Полный контроль над секретами.
Минусы:
Не масштабируется: нужно вручную копировать образ на каждый узел.
Если узлов сотни, это становится непрактичным.
Вывод: Подходит для небольшого кластера (как твои три мастера), но не для больших систем.

2. Приватный реестр Docker Hub
Как это работает:
Создаёшь приватный репозиторий на Docker Hub (бесплатно одно приватное репо, дальше платно).
Выкладываешь образ:
bash

Collapse

Wrap

Copy
sudo docker tag shadowsocks-with-tools yourusername/shadowsocks-with-tools:latest
sudo docker push yourusername/shadowsocks-with-tools:latest
В deployment.yaml:
yaml

Collapse

Wrap

Copy
image: yourusername/shadowsocks-with-tools:latest
Настраиваешь доступ через imagePullSecrets в k3s:
yaml

Collapse

Wrap

Copy
apiVersion: v1
kind: Secret
metadata:
  name: dockerhub-credentials
  namespace: kube-system
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
Где <base64-encoded-docker-config> — это закодированный JSON с твоими логином и паролем от Docker Hub:
bash

Collapse

Wrap

Copy
echo -n '{"auths":{"https://index.docker.io/v1/":{"username":"yourusername","password":"yourpassword"}}}' | base64
Добавь в Deployment:
yaml

Collapse

Wrap

Copy
spec:
  imagePullSecrets:
  - name: dockerhub-credentials
Плюсы:
k3s сам скачивает образ на всех узлах.
Репозиторий приватный, доступ только по ключам.
Минусы:
Образ всё равно уходит в облако.
Ограничения бесплатного плана (1 приватное репо).
Вывод: Подходит, если доверяешь Docker Hub, но не идеально для "секретов".

3. Локальный приватный реестр в твоей сети
Как это работает:
Разворачиваешь свой собственный реестр на одном из серверов (например, opi5plus-3, 192.168.1.26).
Используешь официальный образ registry:
bash

Collapse

Wrap

Copy
sudo docker run -d -p 5000:5000 --restart=always --name registry registry:2
Тегни и выложи свой образ:
bash

Collapse

Wrap

Copy
sudo docker tag shadowsocks-with-tools:latest 192.168.1.26:5000/shadowsocks-with-tools:latest
sudo docker push 192.168.1.26:5000/shadowsocks-with-tools:latest
В deployment.yaml:
yaml

Collapse

Wrap

Copy
image: 192.168.1.26:5000/shadowsocks-with-tools:latest
Безопасность (опционально):
Добавь TLS и аутентификацию:
Сгенерируй сертификаты:
bash

Collapse

Wrap

Copy
mkdir -p /etc/docker/certs
openssl req -newkey rsa:4096 -nodes -sha256 -keyout /etc/docker/certs/registry.key -x509 -days 365 -out /etc/docker/certs/registry.crt
Запусти реестр с TLS:
bash

Collapse

Wrap

Copy
sudo docker run -d -p 5000:5000 --restart=always --name registry \
  -v /etc/docker/certs:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/registry.key \
  registry:2
Настрой аутентификацию через htpasswd:
bash

Collapse

Wrap

Copy
sudo apt install apache2-utils
mkdir -p /etc/docker/auth
htpasswd -Bc /etc/docker/auth/htpasswd username password
docker run -d -p 5000:5000 --restart=always --name registry \
  -v /etc/docker/certs:/certs \
  -v /etc/docker/auth:/auth \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/registry.key \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  registry:2
Создай Secret для k3s:
yaml

Collapse

Wrap

Copy
apiVersion: v1
kind: Secret
metadata:
  name: registry-credentials
  namespace: kube-system
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-config>
Где <base64-encoded-config>:
bash

Collapse

Wrap

Copy
echo -n '{"auths":{"192.168.1.26:5000":{"username":"username","password":"password"}}}' | base64
В deployment.yaml:
yaml

Collapse

Wrap

Copy
spec:
  imagePullSecrets:
  - name: registry-credentials
Плюсы:
Образ остаётся в твоей сети.
Полный контроль над доступом.
k3s сам скачивает образ на узлах.
Минусы:
Нужно настроить реестр и поддерживать его.
Вывод: Идеально для "секретных" контейнеров в большом кластере.

4. Другие приватные реестры (Harbor, Nexus, etc.)
Если кластер большой, можешь развернуть более мощный реестр, например:
Harbor: Open-source, с UI, ролями и интеграцией с Kubernetes.
Nexus Repository: Поддерживает Docker и другие форматы.
Установка сложнее, но для сотен узлов это оправдано.
Ответ на твой вопрос
Можно ли хранить контейнеры у себя? Да, можно! Локальный реестр (Вариант 3) — это то, что тебе нужно. Ты разворачиваешь его на своём сервере (например, 192.168.1.26:5000), загружаешь туда образы, и k3s тянет их оттуда. Никакого Docker Hub, всё под твоим контролем.
Секреты: Они остаются в твоей сети, и доступ ограничивается твоими правилами (TLS, пароли).
Для твоего случая
Сейчас у тебя 3 узла, и локальный импорт работает. Но если кластер вырастет, настрой локальный реестр:

bash

Collapse

Wrap

Copy
sudo docker run -d -p 5000:5000 --restart=always --name registry registry:2
sudo docker tag shadowsocks-with-tools:latest 192.168.1.26:5000/shadowsocks-with-tools:latest
sudo docker push 192.168.1.26:5000/shadowsocks-with-tools:latest
В deployment.yaml:

yaml

Collapse

Wrap

Copy
image: 192.168.1.26:5000/shadowsocks-with-tools:latest
Проверка Shadowsocks
Теперь, когда под работает, давай проверим Shadowsocks:

Зайди в под:
bash

Collapse

Wrap

Copy
sudo k3s kubectl exec -it -n kube-system shadowsocks-client-stockholm-<hash> -- sh
TCP:
bash

Collapse

Wrap

Copy
curl --socks5 127.0.0.1:1081 http://ifconfig.me
Ожидаемый результат: <VPS_IP>.
UDP:
bash

Collapse

Wrap

Copy
nc -u -l 12345
С VPS:
bash

Collapse

Wrap

Copy
echo "test" | nc -u v-panel.sw.cube2.ru 12345
Логи:
bash

Collapse

Wrap

Copy
sudo k3s kubectl logs -n kube-system -l app=shadowsocks-client-stockholm
Итог
"Секретные" контейнеры не нужно выкладывать в Docker Hub — храни их в локальном реестре.
Для проверки Shadowsocks используй команды выше.
Напиши:

<PUBLIC_IP>, <VPS_IP>, имя пользователя.
Результат проверки (curl, nc, логи).
Ты круто всё освоил — теперь ты готов к любым кластерам! Я с тобой!