# Установка RU-локали в Ubuntu 

Чтобы добавить русскую локаль `ru_RU.UTF-8` в Ubuntu, выполните следующие шаги:

Проверим, доступна ли локаль ru_RU.UTF-8 в системе:
```shell
locale -a | grep ru_RU
```

Если локаль ru_RU.UTF-8 уже есть в списке, значит, она уже установлена. Если её нет, cгенерируем локаль `ru_RU.UTF-8`
и добавим её в систему:
```shell
sudo locale-gen ru_RU.UTF-8
```

Обновим настройку локалей:
```shell
sudo update-locale
```

## Установим локаль как системную по умолчанию. :

Чтобы локаль `ru_RU.UTF-8` была установлена по умолчанию для всей системы, отредактируем файл `/etc/default/locale`:
```shell
sudo nano /etc/default/locale
```

Добавим (или изменим) строки на следующие:
```text
LANG=ru_RU.UTF-8
LC_ALL=ru_RU.UTF-8
```

Сохраним файл и выйдем из редактора (`Ctrl + X`, затем `Y` для подтверждения).

Чтобы изменения вступили в силу, перезагрузим систему или выполним:
```shell
source /etc/default/locale
```

Проверим текущую локаль:
```shell
locale
```

Увидим что-то типа:
```text
LANG=ru_RU.UTF-8
LANGUAGE=
LC_CTYPE="ru_RU.UTF-8"
LC_NUMERIC="ru_RU.UTF-8"
LC_TIME="ru_RU.UTF-8"
LC_COLLATE="ru_RU.UTF-8"
LC_MONETARY="ru_RU.UTF-8"
LC_MESSAGES="ru_RU.UTF-8"
LC_PAPER="ru_RU.UTF-8"
LC_NAME="ru_RU.UTF-8"
LC_ADDRESS="ru_RU.UTF-8"
LC_TELEPHONE="ru_RU.UTF-8"
LC_MEASUREMENT="ru_RU.UTF-8"
LC_IDENTIFICATION="ru_RU.UTF-8"
LC_ALL=ru_RU.UTF-8
```

Локаль `ru_RU.UTF-8` установлена корректно.


