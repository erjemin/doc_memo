# Резервное копирование и восстановление K3s

У меня все манифесты хранятся в домашнем каталоге в папке `~/k3s`, но сохранение манифестов не обеспечит резервного
копирования (хотя и будет хорошим подспорьем). Но в k3s есть еще настройки развертывания, маршруты, секреты,
данные etcd (базы данных, в котрой хранится и синхронизируется вся информация k3s) и тома блочного хранилища
PersistentVolumeClaims (PVC). Хочется сделать резервную копию всего этого, на случай сбоя и фактора "кривых рук".

Вот скрпит (не забудьте заменить `<secret-password>`, `<NAS-IP>` и `<FOLDER>` на свои значения):
```bash
#!/usr/bin/bash

# МОНТИРУЕМ SAMBA -- Seagate Personal Cloud
mount -t cifs -o username=erjemin,password=<secret-password> //<NAS-IP>/<FOLDER> /media/backup/
echo -e "$(date +'%F %R:%S') - монтируем SAMBA '/media/backup':\n$(date +'%F %R:%S') - =========================\n";
# Проверяем, что на SAMBA-каталоге есть каталог k3s-backup (и создаем его, если нет)
if [ ! -d /media/backup/k3s-backup ]; then
    mkdir -p /media/backup/k3s-backup
fi
echo -e "\n\n$(date +'%F %R:%S') - монтируем SAMBA '/media/backup':\n$(date +'%F %R:%S') - =========================\n" >> /media/backup/k3s-backup/backup.log

cd /home/opi
# Резервное копирование

# ==== 1 Снапшоты etcd (k3s по умолчанию делает снапшоты каждык 12 часов
#        в каталоге /var/lib/rancher/k3s/server/db/snapshots)
/usr/bin/zip -r /media/backup/k3s-backup/etcd--$(date +'%F--%H-%M-%S').zip /var/lib/rancher/k3s/server/db/snapshots/

# ==== 2 Сохранение манифестов
/usr/bin/zip -r /media/backup/k3s-backup/manifests--$(date +'%F--%H-%M-%S').zip /home/opi/k3s/

# ==== 3 Сохранение Секретов
# Получаем список пространств имен
namespaces=$(kubectl get ns -o jsonpath='{.items[*].metadata.name}')
archive="secrets--$(date +'%F--%H-%M-%S').zip"
# Создаем временную директорию 
tmp_dir=$(mktemp -d)
# Перебираем каждое пространство имен
for ns in $namespaces; do
    # kubectl get secret -n "$ns" -o yaml | zip $archive - "secrets-$ns.yaml"
    kubectl get secret -n "$ns" -o yaml  > "secrets-$ns.yaml" && zip $archive "secrets-$ns.yaml" && rm "secrets-$ns.yaml"
done

# ==== 4 Сохранение PVC






# Чистим старые файлы резервных копий
echo -e "$(date +'%F %R:%S') - удаляем старые файлы бекапов старше 14 дней:\n$(date +'%F %R:%S') - =========================\n" >> /media/backup/k3s-backup/backup.log
/usr/bin/find /media/backup/k3s-backup/ -type f -name "*.zip" -mtime +14 -name "backup.log-*" -delete
echo -e "$(date +'%F %R:%S') - удаляем старые файлы бекапов старше 14 дней:\n$(date +'%F %R:%S') - =========================\n"

echo -e "$(date +'%F %R:%S') - ВСЕ РЕЗЕРВНЫЕ КОПИИ K3S В SAMBA:\n$(date +'%F %R:%S') - =========================\n" >> /media/backup/k3s-backup/backup.log
ls -alhc /media/backup/k3s-backup/ >> /media/backup/k3s-backup/backup.log
echo -e "$(date +'%F %R:%S') - ВСЕ РЕЗЕРВНЫЕ КОПИИ K3S В SAMBA:\n$(date +'%F %R:%S') - =========================\n"
ls -alhc /media/backup/k3s-backup/ 

# Отсоединяем SAMBA
echo -e "$(date +'%F %R:%S') - отсоединяем SAMBA\n$(date +'%F %R:%S') - =========================\n" >> /media/backup/k3s-backup/backup.log
# Ротация лога
if [ $(stat -c %s /media/backup/k3s-backup/backup.log) -gt 10485760 ]; then
    mv /media/backup/k3s-backup/backup.log /media/backup/k3s-backup/backup.log-$(date +'%F--%H-%M-%S')
    touch /media/backup/k3s-backup/backup.log
fi
umount /media/backup
```