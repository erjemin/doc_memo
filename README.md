# Памятки по «граблям» и просто полезные заметки, инструкции и самописные скрипты и доки

## Docker
* [Расположение образов Docker](docker/docker-adjasting.md)
* [Установка GPG-ключа для репозиториев Docker](docker/docker-trusted-gpg.md)
* [Контейнер MariaDB/MySQL](docker/docker-mariadb.md)
* [Nginx и letsencrypt в контейнерах](docker/docker-nginx-w-certbot.md) на примере Portainer
* [Контейнер MySQL под Windows 10](docker/docker-mysql-in-windows10.md)
* [Развертывание VPN-сервера на базе MS SSTP](docker/docker-sstp-vpn.md)
* [Развертывание прокси базе Shadowsocks (сервер и клиент)](docker/docker-shadowsocks.md)

## Kubernetes (k3s/k8s)
* [Установка k3s на Orange Pi 5 Plus](raspberry-and-orange-pi/k3s.md)
* [Под с Shadowsocks-клиент](kubernetes/k3s-shadowsocks-client.md) (k3s)

## Python 
* [Устранение проблем при установке Python-коннектора mysqlclient (MySQL/MariaDB)](python/python-mysql.md)
* [Python-скрипт как служба Linux](python/python_as_service.md)

## Orange Pi / Raspberry Pi 
* [Установка (перенос) системы на NVMe или eMMC (для Orange Pi 5 Plus)](raspberry-and-orange-pi/opi5plus-move-system-to-nvme-or-emmc.md)
* [Измерение производительности накопителей](raspberry-and-orange-pi/measuring-performance-storage-devices.md)
* [Установка Docker и Docker Compose](raspberry-and-orange-pi/install-docker-compose.md)
* [Резервное копирование и восстановление](raspberry-and-orange-pi/backup-restore.md) 
* [k8s (кubernetes) на Orange Pi (драфт...)](raspberry-and-orange-pi/k8s.md)
* [k3s (кubernetes) на Orange Pi](raspberry-and-orange-pi/k3s.md)
* 

## Nginx / Apache
* [Ограничение доступа по User-Agent (на примере GPTBot)](nginx/nginx-ban-user-agent.md)
* [Настройка nginx как прямого прокси](nginx/nginx_as_direct_proxy.md)

## Разное
* [Настройка RU-локали в Ubuntu/Debian](misc/set-locale-ru.md)
* [Развертывание Django-приложения (сайта) на VDS-хостинге](misc/deploying-django-site-to-dvs-hosting.md)
* [Сплиттер для разделения логов](misc/splitter-for-logs.md)