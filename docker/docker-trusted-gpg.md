# Установка GPG-ключа для репозиториев Docker

Иногда, при установке Docker, возникает ошибка с ключом GPG. Например, при установке Docker на Ubuntu 20.04. Тогда
при обновлении списка пакета при обновлении командой `sudo apt-get update` выдается сообщение (или подобное):

> W: https://mirrors.aliyun.com/docker-ce/linux/ubuntu/dists/jammy/InRelease: Key is stored in legacy trusted.gpg keyring (/etc/apt/trusted.gpg), see the DEPRECATION section in apt-key(8) for details. ??

Это связано с устаревшей практикой хранения ключей репозиториев в общем файле `/etc/apt/trusted.gpg`, а предупреждение
означает, что ключ GPG для репозитория Docker CE хранится в старом формате, который в будущем будет удален из
APT (Advanced Package Tool). Ранее APT использовал общий файл /etc/apt/trusted.gpg для хранения **всех** ключей
репозиториев. Это устарело из-за соображений безопасности. Новая практика заключается в том, чтобы хранить ключи
в отдельных файлах в директории `/etc/apt/trusted.gpg.d/`.

## Как исправить

Найдём ключ для репозитория:
```shell
apt-key list
```

Ключ, связанный с репозиторием Docker может выглядеть, например, так (и радом будет предупреждение):
```text
Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).
/etc/apt/trusted.gpg
--------------------
pub   rsa4096 2017-02-22 [SCEA]
      9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid           [ unknown] Docker Release (CE deb) <docker@docker.com>
sub   rsa4096 2017-02-22 [S]
```

Удалим ключ из старой связки ключей (keyring):
```shell
sudo apt-key del 0EBFCD88
```

На всякий случай, проверим, что установлены покеты для работы с HTTPS, curl для загрузки ключей по интернет,
ca-certificates для проверки сертификатов и gpg для работы с ключами. Просто установим их (если они не установлены,
то ничего не произойдет):
```shell
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```  
  
На Orange Pi 5 Plus у меня не получилось установить GPG-ключ для Docker нормальным образом через команду (у вас, может, и получится): 
```shell
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/docker-archive-keyring.gpg --keyserver keyserver.ubuntu.com --recv-keys 7EA0A9C3F273FCD8
```
 
И потому я пошел другим путем. Скачал ключ с сайта Docker и установил его вручную:
```shell
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

Проверим, что ключ сохранен:
```shell
ls -l /usr/share/keyrings/docker-archive-keyring.gpg
```

Исправим список репозиториев для Docker. Отроем на редактирование _docker.list_  командой:
```shell
sudo nano /etc/apt/sources.list.d/docker.list
```

У меня он выглядел так:
```text
deb [arch=arm64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu jammy stable
```

Исправим его так, чтобы обращение в репозиторий было через GPG-ключ:
```text
# deb [arch=arm64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu jammy stable
deb [arch=arm64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu jammy stable
```

Обновим список пакетов:
```shell
sudo apt-get update
```

Теперь при обновлении списка пакетов не будет предупреждения о старом ключе GPG!