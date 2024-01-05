# Устранение возникшей проблемы с установкой mysqlclient (подключение к MySQL/MariaDB) 

Обычным порядком установка осуществляется командой:
```bash
pip install mysqlclient
```

Но иногда возникают проблемы.

## Проблемы под Apple Mac OS

Если mysqlclient не устанавливается, то возможно нет БД-коннектора в системе.
**Обычно такое происходит если сама СУБД размещена в контейнере Docker** (при установке СУБД
в саму систему клиент «прилетает» автоматически).  

Для MariaDB проблема решается временной установкой `mariadb-connector-c` и созданием
символической ссылки на `mysql_config`:
```bash
brew install mariadb-connector-c
sudo ln -s /usr/local/opt/mariadb-connector-c/bin/mariadb_config /usr/local/bin/mysql_config
```

Затем активизируем виртуальное окружение нашего проекта, устанавливаем mysqlclient и выходим
из виртуального окружения:
```bash
source ~/path-to-you-prj-enveroment/bin/activate
pip install mysqlclient
deactivate
```

Затем удаляем симлинк на коннектор и отключаем его:
```bash
sudo rm /usr/local/bin/mysql_config
brew unlink mariadb-connector-c
```

Аналогичным образом решается проблема и с MySQL. Отличается только устанавливаемый клиент:
```bash
brew install mysql-client
sudo ln -s /opt/homebrew/opt/mysql-client/bin/mysql_config /usr/local/bin/mysql_config

source ~/path-to-you-prj-enveroment/bin/activate
pip install mysqlclient
deactivate

rm /usr/local/bin/mysql_config
brew unlink mysql-client
```

### Проблемы с MYSQLCLIENT_CFLAGS и MYSQLCLIENT_LDFLAGS

Может возникнуть и под Linux, и под Windows, и под Mac OS (например, в моём случае это случилось при обновлении
операционной системы). При установке mysqlclient выдается что-то типа такого сообщения:
```text
      ...
      ...
      Exception: Can not find valid pkg-config name.
      Specify MYSQLCLIENT_CFLAGS and MYSQLCLIENT_LDFLAGS env vars manually
      [end of output]
  
  note: This error originates from a subprocess, and is likely not a problem with pip.
error: subprocess-exited-with-error

× Getting requirements to build wheel did not run successfully.
│ exit code: 1
╰─> See above for output.

note: This error originates from a subprocess, and is likely not a problem with pip.
```

То есть не заданы переменные окружения MYSQLCLIENT_CFLAGS и MYSQLCLIENT_LDFLAGS. Придется
их задать вручную в терминале:

1. Определим расположение заголовочных файлов и библиотек MySQL (или MariaDB). Размещение
   этих файлов отличаются в зависимости от способов установки MySQL и операционной системы. 
   Используем команду `mysql_config` для получения этой информации. Запустим команду `mysql_config --cflags`,
   чтобы получить расположение заголовочных файлов, и команду `mysql_config --libs`, чтобы найти расположение
   библиотек. Запишем то что вывели эти обе команды.
2. Установим переменную окружения `MYSQLCLIENT_CFLAGS`. Выполним команду: `export MYSQLCLIENT_CFLAGS="-I"`,
   заменив в ней все внутри кавычек (`-I`) путем, который получили с помощью `mysql_config --cflags`.
   Например, у меня получилось вот так: `export MYSQLCLIENT_CFLAGS="-I/usr/local/opt/mysql-client/include/mysql"`.
3. Установим переменную окружения `MYSQLCLIENT_LDFLAGS`. Выполним команду: `export MYSQLCLIENT_LDFLAGS="-L"`,
    заменив в ней все внутри кавычек (`-L`) путем, который получили с помощью `mysql_config --libs`.
    Например, у меня получилось вот так: `export MYSQLCLIENT_LDFLAGS="-L/opt/homebrew/opt/mysql-client/lib 
    -lmysqlclient -lz  -lzstd -L/opt/homebrew/lib -lssl -lcrypto -lresolv"`.
   
Теперь, установка mysqlclient должна пройти успешно:
```bash
pip install mysqlclient
```

Получим что-то типа:
```text
Collecting mysqlclient
  Using cached mysqlclient-2.2.0.tar.gz (89 kB)
  Installing build dependencies ... done
  Getting requirements to build wheel ... done
  Installing backend dependencies ... done
  Preparing metadata (pyproject.toml) ... done
Building wheels for collected packages: mysqlclient
  Building wheel for mysqlclient (pyproject.toml) ... done
  Created wheel for mysqlclient: filename=mysqlclient-2.2.0-cp310-cp310-macosx_10_9_universal2.whl size=96614 sha256=02b525c4e2ca7901bfa3c196eeee0becbe149d2379f8cd03c95225178114b6d6
  Stored in directory: /Users/[user]/Library/Caches/pip/wheels/a4/f8/fd/0399687c0abd03c10c975ed56c692fcd3d0fb80440b5a661f1
Successfully built mysqlclient
Installing collected packages: mysqlclient
Successfully installed mysqlclient-2.2.0
```

### Проблемы с clang и библиотекой zstd

После обновления на macOS Sonoma 14.2.1 (23C71) снова возникла проблема с установкой mysqlclient.
Выводилось что-то типа:
```txt
 ...
 ...
 clang -bundle -undefined dynamic_lookup -arch arm64 -arch x86_64 -g build/temp.macosx-10.9-universal2-cpython-310/src/MySQLdb/_mysql.o -o build/lib.macosx-10.9-universal2-cpython-310/MySQLdb/_mysql.cpython-310-darwin.so -L/opt/homebrew/opt/mysql-client/lib -lmysqlclient -lz -lzstd -lssl -lcrypto -lresolv
      ld: library 'zstd' not found
      clang: error: linker command failed with exit code 1 (use -v to see invocation)
      error: command '/usr/bin/clang' failed with exit code 1
```

Для решения сначала пришлось установить библиотеку zstd и переустановить sqlclient [по инструкции
из официального репозитория](https://github.com/PyMySQL/mysqlclient/blob/main/README.md#macos-homebrew):  
```bash
brew install zstd mysql-client pkg-config
export PKG_CONFIG_PATH="$(brew --prefix)/opt/mysql-client/lib/pkgconfig"
```

Затем снова установить значения переменных окружения `MYSQLCLIENT_CFLAGS` и `MYSQLCLIENT_LDFLAGS`. Но сначала
узнать через `pkg-config` чему они равны:
```bash
pkg-config --cflags mysqlclient
pkg-config --libs mysqlclient
```

В первом случае я получил `-I/opt/homebrew/Cellar/mysql-client/8.2.0/include/mysql`, а во втором
`-L/opt/homebrew/Cellar/mysql-client/8.2.0/lib -lmysqlclient`. У вас может быть другие значения.

После установки соответсвующий переменных окружения:
```bash
export MYSQLCLIENT_CFLAGS="-I/opt/homebrew/Cellar/mysql-client/8.2.0/include/mysql"
export MYSQLCLIENT_LDFLAGS="-L/opt/homebrew/Cellar/mysql-client/8.2.0/lib -lmysqlclient"
```

И только после этого удалось успешно установить питоновский mysqlclient.
