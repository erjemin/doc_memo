# Резервное копирование и восстановление Raspberry Pi (Orange Pi)

**Важно:** *На борту моего микрокомпьютера Orange Pi 5 встроен объемный SSD
(смонтированный в `/home/`). Он используется для создания и промежуточного образа системного
SD-накопителя.*

## Для облака _cloud.mail.ru_ или _360.yandex.ru/disk_

### Установка WebDav и подключения "облаков":

Установим поддержку файловой системы WebDav на наш Orange Pi:
```shell
sudo apt-get install davfs2
```

Включим пользователя `[user]`, от имени которого производится резервное копирование
(обычно **pi** -- для Raspberry Pi, и **orangepi** -- для Orange Pi), в группу `davfs2`:  
```bash
sudo usermod -aG davfs2 [user]
```

**ВАЖНО** *В дальнейших примерах, хранение резервных копий осуществляется в [инфраструктуре
облака mail.ru (о-drive)](https://cloud.mail.ru/).* 

Создадим папку для монтирования "облака" mail.ru:
```shell
sudo mkdir /media/maildisk
```

Дадим права пользователю `[user]` на монтирование "облака":
```shell
sudo chmod 4755 /sbin/mount.davfs
sudo chown -R [user]:[user] /media/maildisk/
```

С 1 января 2022 для подключения "облака" по WebDAV нужно использовать пароль внешнего приложения.
Доступ по обычному паролю закрыт, создать пароль приложения для mail.ru [надо по ссылке](https://help.mail.ru/mail/security/protection/external),
а для Yandex-Disk [тут](https://yandex.ru/support/id/authorization/app-passwords.html#app-passwords__create).

Изменим файл с паролями `/etc/davfs2/secrets` и запишем в него логин и пароль от "облака":
```shell
sudo nano /etc/davfs2/secrets
```

И добавим в конец файла, что-то типа:
```txt
/media/maildisk                 [ваш-логин]@mail.ru             [пароль-внешнего-приложения]
```

К слову сказать, пароль от внешнего приложения через `/etc/davfs2/secrets` срабатывает
не всегда, поэтому я дублирую [пароль-внешнего-приложения] в файл `/media/mailru.pwd`: 
```shell
sudo echo "[пароль-внешнего-приложения]" > /media/mailru.pwd
```

Для возможности не только читать, но и писать в облако, надо отредактировать файл
`/etc/davfs2/davfs2.conf` (расположение файла может варьироваться в зависимости от
используемого дистрибутива), найти в нём строчку, начинающуюся `# use_locks`, и заменить
её на:
```txt 
use_locks 0
```

### Скрипт резервного копирования в "облако" 

Создадим скрипт резервного копирования `backup_orange.sh` (я предпочитаю хранить скрипты
в папке `/home/[user]/scripts/`):
```shell
nano ~/scripts/backup_orange.sh
```

Скрипт резервного копирования (*не забудьте заменить `[user]` и `[login]`*) сохранит
zip-архивы образа flash-накопителя и домашний каталог `/home/` в папку
`/BackUps/orange-pi-home` "облака". Архивы будут разбиты на части по 1 Гб, т.к. облако
не позволяет загружать файлы больше 2 Гб. Старые архивы будут удаляться через 8 дней:
```shell
#!/usr/bin/bash
if [ ! -d /home/[user]/backup ]; then
    mkdir -p /home/[user]/backup
fi

if [ ! -d /media/maildisk/BackUps ]; then
    mkdir -p /media/maildisk/BackUps
    mkdir -p /media/maildisk/BackUps/orange-pi-home
else
    if [ ! -d /media/maildisk/BackUps/orange-pi-home ]; then
        mkdir -p /media/maildisk/BackUps/orange-pi-home
    fi
fi

echo -e "$(date +'%F %R:%S') - делаем образ flash-накопителя на локальный SSD-диск:\n$(date +'%F %R:%S') - =========================\n"  >> /home/[user]/backup/backup.log
echo -e "$(date +'%F %R:%S') - делаем образ flash-накопителя на локальный SSD-диск:\n$(date +'%F %R:%S') - =========================\n"
dd if=/dev/mmcblk1 of=/home/[user]/backup/flash-disk.img >> /home/[user]/backup/backup.log

echo -e "$(date +'%F %R:%S') - ГОТОВО:\n$(date +'%F %R:%S') - =========================\n" >> /home/[user]/backup/backup.log
echo -e "$(date +'%F %R:%S') - ГОТОВО:\n$(date +'%F %R:%S') - =========================\n";
ls -alh /home/[user]/backup/flash-disk.img >> /home/[user]/backup/backup.log
echo -e "$(date +'%F %R:%S') - =========================\n" >> /home/[user]/backup/backup.log
echo -e "$(date +'%F %R:%S') - =========================\n";

echo -e "$(date +'%F %R:%S') - монтируем облако mail.ru в папку '/media/maildisk':\n$(date +'%F %R:%S') - =========================\n" >> /home/[user]/backup/backup.log
echo -e "$(date +'%F %R:%S') - монтируем облако mail.ru в папку '/media/maildisk':\n$(date +'%F %R:%S') - =========================\n";
mount.davfs -o username=[login]@mail.ru https://webdav.cloud.mail.ru /media/maildisk < /media/mailru.pwd

echo -e "\n$(date +'%F %R:%S') - упаковываем образ flash-накопителя в облако mail.ru:\n$(date +'%F %R:%S') - =========================\n" >> /home/[user]/backup/backup.log
echo -e "\n$(date +'%F %R:%S') - упаковываем образ flash-накопителя в облако mail.ru:\n$(date +'%F %R:%S') - =========================\n";
cd /home/[user]/backup/
/bin/zip -s 1024m -9 /media/maildisk/BackUps/orange-pi-home/flash-disk--$(date +'%F').zip flash-disk.img  >> /home/[user]/backup/backup.log
cd /home/[user]/

echo -e "$(date +'%F %R:%S') - ГОТОВО:\n$(date +'%F %R:%S') - =========================\n" >> /home/[user]/backup/backup.log
echo -e "$(date +'%F %R:%S') - ГОТОВО:\n$(date +'%F %R:%S') - =========================\n";
ls -al /media/maildisk/BackUps/orange-pi-home/flash-disk--$(date +'%F').*  >> /home/[user]/backup/backup.log
echo -e "$(date +'%F %R:%S') - =========================\n" >> /home/[user]/backup/backup.log
echo -e "$(date +'%F %R:%S') - =========================\n";

echo -e "$(date +'%F %R:%S') - удаляем образ flash-накопителя с локального SSD-диска:\n$(date +'%F %R:%S') - =========================\n"  >> /home/[user]/backup/backup.log
echo -e "$(date +'%F %R:%S') - удаляем образ flash-накопителя с локального SSD-диска:\n$(date +'%F %R:%S') - =========================\n";
rm -f /home/[user]/backup/flash-disk.img
rm -f /home/[user]/backup/flash-disk--*.*

echo -e "$(date +'%F %R:%S') - упаковываем home-том в облако mail.ru:\n$(date +'%F %R:%S') - =========================\n"  >> /home/[user]/backup/backup.log
echo -e "$(date +'%F %R:%S') - упаковываем home-том в облако mail.ru:\n$(date +'%F %R:%S') - =========================\n";
/bin/zip --symlinks -s 1024m -r9q /media/maildisk/BackUps/orange-pi-home/home-volume--$(date +'%F').zip /home/ -x /home/[user]/backup/*.* >> /home/[user]/backup/backup.log

echo -e "$(date +'%F %R:%S') - ГОТОВО:\n$(date +'%F %R:%S') - =========================\n" >> /home/[user]/backup/backup.log
echo -e "$(date +'%F %R:%S') - ГОТОВО:\n$(date +'%F %R:%S') - =========================\n";
ls -al /media/maildisk/BackUps/orange-pi-home/home-volume--$(date +'%F').*  >> /home/[user]/backup/backup.log
echo -e "$(date +'%F %R:%S') - =========================STOP\n" >> /home/[user]/backup/backup.log
echo -e "$(date +'%F %R:%S') - =========================STOP\n";

echo -e "$(date +'%F %R') - удаляем старые файлы быкапов больше чем за два дня:\n$(date +'%F %R') - =========================\n" >> /home/[user]/backup/backup.log
echo -e "$(date +'%F %R') - удаляем старые файлы быкапов больше чем за два дня:\n$(date +'%F %R') - =========================\n";
/usr/bin/find /media/maildisk/BackUps/orange-pi-home/ -mtime +8 -delete  >> /home/[user]/backup/backup.log

echo -e "$(date +'%F %R') - ВСЕ РЕЗЕРВНЫЕ КОПИИ В ОБЛАКЕ MAIL.RU:\n $(date +'%F %R') =========================\n" >> /home/[user]/backup/backup.log
echo -e "$(date +'%F %R') - ВСЕ РЕЗЕРВНЫЕ КОПИИ В ОБЛАКЕ MAIL.RU:\n $(date +'%F %R') =========================\n";
ls -alhc /media/maildisk/BackUps/orange-pi-home/  >> /home/[user]/backup/backup.log
echo -e "=========================\n" >> /home/[user]/backup/backup.log
echo -e "=========================\n";

echo -e "$(date +'%F %R') - Копируем log-бекапа в облако MAIL.RU:\n $(date +'%F %R') =========================\n" >> /home/[user]/backup/backup.log
echo -e "$(date +'%F %R') - Копируем log-бекапа в облако MAIL.RU:\n $(date +'%F %R') =========================\n";
cp /home/[user]/backup/backup.log /media/maildisk/BackUps/orange-pi-home/

echo -e "$(date +'%F %R') - отсоединяем облако mail.ru\n $(date +'%F %R') =========================\n" >> /home/[user]/backup/backup.log
echo -e "$(date +'%F %R') - отсоединяем облако mail.ru\n $(date +'%F %R') =========================\n";
umount /media/maildisk  >> /home/[user]/backup/backup.log
```


## Для резервного копирования в _SMB-папку_ внутри домашней сети

### Установка Samba клиента:

Скорее всего поддержка **samba** уже включена, но на всякий случай установим её на наш
Orange Pi (если ещё не установлена):
```shell
sudo apt-get install samba samba-client smbclient cifs-utils
```

Создадим папку для монтирования samba-папки:
```shell
sudo mkdir /media/backup
```

Дадим права пользователю `[user]` на запись в samba-папку:
```shell
sudo chown -R 777 /media/backup/
sudo chown -R [user]:[user] /media/backup/
```

### Скрипт резервного копирования в SAMBA-папку внутри домашней сети 

Скрипт резервного копирования (*не забудьте заменить `[ip]`, `[user]` и `[login]` -- ip-адрес NAS
в домашней сети, NAS-логин и NAS-пароль*) сохранит zip-архивы образа flash-накопителя и домашний
каталог `/home/` в папку `/NetBackup` на NAS. Старые архивы будут удаляться через 24 дня:
```shell
#!/usr/bin/bash

echo -e "$(date +'%F %R:%S') - монтируем SAMBA '/media/backup':\n$(date +'%F %R:%S') - =========================\n";
mount -t cifs -o username=[samba-login],password=[smaba-pwd] //192.168.1.50/NetBackup /media/backup/
echo -e "$(date +'%F %R:%S') - монтируем SAMBA '/media/backup':\n$(date +'%F %R:%S') - =========================\n" >> /media/backup/orange-pi-backup/backup.log


if [ ! -d /media/backup/orange-pi-backup ]; then
    mkdir -p /media/backup/orange-pi-backup
fi

if [ ! -d /home/[user]/backup ]; then
    mkdir -p /home/[user]/backup
fi

echo -e "$(date +'%F %R:%S') - делаем образ flash-накопителя на локальный SSD-диск:\n$(date +'%F %R:%S') - =========================\n"  >> /media/backup/orange-pi-backup/backup.log
echo -e "$(date +'%F %R:%S') - делаем образ flash-накопителя на локальный SSD-диск:\n$(date +'%F %R:%S') - =========================\n"
dd if=/dev/mmcblk1 of=/home/[user]/backup/flash-disk.img >> /home/[user]/backup/backup.log

echo -e "$(date +'%F %R:%S') - ГОТОВО:\n$(date +'%F %R:%S') - =========================\n" >> /media/backup/orange-pi-backup/backup.log
echo -e "$(date +'%F %R:%S') - ГОТОВО:\n$(date +'%F %R:%S') - =========================\n";
ls -alh /home/[user]/backup/flash-disk.img >> /media/backup/orange-pi-backup/backup.log
echo -e "$(date +'%F %R:%S') - =========================\n" >> /media/backup/orange-pi-backup/backup.log
echo -e "$(date +'%F %R:%S') - =========================\n";

echo -e "\n$(date +'%F %R:%S') - упаковываем образ flash-накопителя в SAMBA:\n$(date +'%F %R:%S') - =========================\n" >> /media/backup/orange-pi-backup/backup.log
echo -e "\n$(date +'%F %R:%S') - упаковываем образ flash-накопителя в SAMBA:\n$(date +'%F %R:%S') - =========================\n";
cd /home/[user]/backup/
/bin/zip -9 /media/backup/orange-pi-backup/flash-disk--$(date +'%F').zip flash-disk.img  >> /media/backup/orange-pi-backup/backup.log
cd /home/[user]/

echo -e "$(date +'%F %R:%S') - ГОТОВО:\n$(date +'%F %R:%S') - =========================\n" >> /media/backup/orange-pi-backup/backup.log
echo -e "$(date +'%F %R:%S') - ГОТОВО:\n$(date +'%F %R:%S') - =========================\n";
ls -al /media/backup/orange-pi-backup/flash-disk--$(date +'%F').*  >> /media/backup/orange-pi-backup/backup.log
echo -e "$(date +'%F %R:%S') - =========================\n" >> /media/backup/orange-pi-backup/backup.log
echo -e "$(date +'%F %R:%S') - =========================\n";

echo -e "$(date +'%F %R:%S') - удаляем образ flash-накопителя:\n$(date +'%F %R:%S') - =========================\n"  >> /media/backup/orange-pi-backup/backup.log
echo -e "$(date +'%F %R:%S') - удаляем образ flash-накопителя:\n$(date +'%F %R:%S') - =========================\n";
rm -f /home/[user]/backup/flash-disk.img

echo -e "$(date +'%F %R:%S') - упаковываем home-том в SAMBA:\n$(date +'%F %R:%S') - =========================\n"  >> /media/backup/orange-pi-backup/backup.log
echo -e "$(date +'%F %R:%S') - упаковываем home-том в SAMBA:\n$(date +'%F %R:%S') - =========================\n";
/bin/zip --symlinks -r9q /media/backup/orange-pi-backup/home-volum--$(date +'%F').zip /home/ -x /home/[user]/backup/*.* >> /media/backup/orange-pi-backup/backup.log


echo -e "$(date +'%F %R:%S') - ГОТОВО:\n$(date +'%F %R:%S') - =========================\n" >> /media/backup/orange-pi-backup/backup.log
echo -e "$(date +'%F %R:%S') - ГОТОВО:\n$(date +'%F %R:%S') - =========================\n";
ls -al /media/backup/orange-pi-backup/home-volum--$(date +'%F').*  >> /media/backup/orange-pi-backup/backup.log
echo -e "$(date +'%F %R:%S') - =========================STOP\n" >> /media/backup/orange-pi-backup/backup.log
echo -e "$(date +'%F %R:%S') - =========================STOP\n";

echo -e "$(date +'%F %R:%S') - удаляем старые файлы быкапов больше чем за два дня:\n$(date +'%F %R') - =========================\n" >> /media/backup/orange-pi-backup/backup.log
echo -e "$(date +'%F %R:%S') - удаляем старые файлы быкапов больше чем за два дня:\n$(date +'%F %R') - =========================\n";
/usr/bin/find /media/backup/orange-pi-backup/ -type f -name "flash-disk--*.*" -mtime +24 -exec rm {} \;
/usr/bin/find /media/backup/orange-pi-backup/ -type f -name "home-volum--*.*" -mtime +24 -exec rm {} \
echo -e "$(date +'%F %R:%S') - ВСЕ РЕЗЕРВНЫЕ КОПИИ В SAMBA:\n$(date +'%F %R:%S') - =========================\n" >> /media/backup/orange-pi-backup/backup.log
echo -e "$(date +'%F %R:%S') - ВСЕ РЕЗЕРВНЫЕ КОПИИ В SAMBA:\n$(date +'%F %R:%S') - =========================\n";
ls -alhc /media/backup/orange-pi-backup/  >> /media/backup/orange-pi-backup/backup.log
echo -e "$(date +'%F %R:%S') - =========================\n" >> /media/backup/orange-pi-backup/backup.log
echo -e "$(date +'%F %R:%S') - =========================\n";

echo -e "$(date +'%F %R:%S') - отсоединяем SAMBA\n$(date +'%F %R:%S') - =========================\n" >> /media/backup/orange-pi-backup/backup.log
echo -e "$(date +'%F %R:%S') - отсоединяем SAMBA\n$(date +'%F %R:%S') - =========================\n";
umount /media/backup
```

## Регулярное выполнение резервного копирования

Дадим права на выполнение скрипта:
```shell
sudo chmod +x ~/scripts/backup_orange.sh
```

Запускать скрипт будем через `cron` от root-пользователя, поэтому отредактируем `crontab`:
```shell
sudo crontab -e
```

И добавим в конец файла строку (комментарии добавлять не обязательно):
```txt
# +-------------------------------- минуты (0 - 59)
# |     +-------------------------- часы (0 - 23)
# |     |       +------------------ день месяца (1 - 31)
# |     |       |       +---------- месяц (1 - 12)
# |     |       |       |       +-- день недели (0 - 7) (Воскресенье=0 или 7)
# |     |       |       |       |
# m     h       dom     mon     *       команда для исполнения
#--------------------------------------------------------------------------
5       0       *       *       1,3,5   /usr/bin/bash /home/[user]/scripts/backup_orange.sh
```

Скрипт будет запускаться каждый понедельник в 00:05. Таким образом в каждый момент времени
в облаке будет храниться две последних резервных копий (за две предыдущих недели).