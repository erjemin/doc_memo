# Резервное копирование и восстановление K3s

У меня все манифесты хранятся в домашнем каталоге в папке `~/k3s`, но сохранение манифестов не обеспечит резервного
копирования (хотя и будет хорошим подспорьем). Но в k3s есть еще настройки развертывания, маршруты, секреты,
данные etcd (базы данных, в котрой хранится и синхронизируется вся информация k3s) и тома блочного хранилища
PersistentVolumeClaims (PVC). Хочется сделать резервную копию всего этого, на случай сбоя и фактора "кривых рук".

```bash
mkdir -p ~/script
nano ~/script/backup-k3s.sh
```

И вставить туда вот такой скрипт (не забудьте заменить `<secret-password>`, `<NAS-IP>` и `<FOLDER>` на свои значения):
```bash
#!/usr/bin/bash
# Скрипт для резервного копирования компонентов K3s (снапшоты etcd, манифесты, секреты)
# на сетевой ресурс SAMBA.

# --- Конфигурация ---
# Локальная точка монтирования для SAMBA
MOUNT_POINT="/media/backup"
# Сетевой ресурс SAMBA
SAMBA_USER="<USER>"
SAMBA_PASSWORD="<secret-password>" # Лучше использовать файл credentials: credentials=/путь/к/.smbcreds
SAMBA_SHARE="//<NAS-IP>/<FOLDER>"
# Каталог для резервных копий на SAMBA
BACKUP_DIR="${MOUNT_POINT}/k3s-backup"
# Временный файл журнала на SAMBA
LOG_FILE="${BACKUP_DIR}/backup.log"
# Каталог с манифестами K3s
MANIFESTS_DIR="/home/opi/k3s"
# Каталог со снапшотами etcd K3s
ETCD_SNAPSHOT_DIR="/var/lib/rancher/k3s/server/db/snapshots"
# Домашний каталог пользователя (используется для cd)
USER_HOME="/home/opi"
# Сколько дней хранить старые резервные копии
RETENTION_DAYS=14
# Формат даты для имен файлов и записей в журнале
DATE_FORMAT='%F--%H-%M-%S'

# --- Вспомогательные функции ---
# Функция для записи сообщения в журнал и на консоль
log_message() {
    local message="$1"
    local timestamp
    timestamp=$(date +'%F %R:%S')
    # Выводим на консоль и дописываем в файл журнала (если он уже доступен)
    echo -e "${timestamp} - ${message}" | tee -a "${LOG_FILE}"
}

# Функция для вывода разделителя в журнал и на консоль
log_separator() {
    local timestamp
    timestamp=$(date +'%F %R:%S')
    echo -e "${timestamp} - =========================" | tee -a "${LOG_FILE}"
}

# Функция для завершения скрипта и размонтирования SAMBA
cleanup_and_exit() {
    local exit_code=$? # Захватываем код завершения последней команды
    local timestamp
    timestamp=$(date +'%F %R:%S')

    # Логируем код завершения *до* попытки размонтирования, пока лог-файл доступен
    log_message "Скрипт завершился с кодом ${exit_code}."

    # Пытаемся размонтировать SAMBA, если она примонтирована
    if mountpoint -q "${MOUNT_POINT}"; then
        log_message "Размонтирование SAMBA ресурса '${MOUNT_POINT}'..." # Это сообщение еще попадет в лог
        log_separator # И это тоже

        if umount "${MOUNT_POINT}"; then
            # <<< РЕСУРС УСПЕШНО РАЗМОНТИРОВАН >>>
            # Выводим сообщение только в консоль, так как лог-файл уже недоступен
            echo "${timestamp} - SAMBA ресурс успешно размонтирован."
        else
            # Ошибка размонтирования. Лог-файл может быть еще доступен, а может и нет.
            # Надежнее вывести ошибку в консоль. Можно попытаться записать и в лог, но без гарантии.
            echo "${timestamp} - ОШИБКА: Не удалось размонтировать SAMBA ресурс '${MOUNT_POINT}'."
            # При желании, можно раскомментировать следующую строку, чтобы попытаться записать ошибку и в лог:
            # log_message "ОШИБКА: Не удалось размонтировать SAMBA ресурс '${MOUNT_POINT}'."
        fi
    else
         # Ресурс не был примонтирован, лог-файл на нем недоступен
         echo "${timestamp} - SAMBA ресурс '${MOUNT_POINT}' не примонтирован или уже размонтирован."
    fi
    exit "${exit_code}"
}

# Перехватываем сигнал EXIT для запуска функции очистки
trap cleanup_and_exit EXIT

# --- Основной скрипт ---

log_message "Запуск скрипта резервного копирования K3s..."
log_separator

# Проверяем, что скрипт запущен от имени root (нужно для mount, доступа к /var/lib/rancher)
if [[ $EUID -ne 0 ]]; then
   log_message "ОШИБКА: Этот скрипт должен быть запущен от имени root (используй sudo)."
   exit 1
fi

# 1. Подготовка точки монтирования
log_message "Проверка и создание локальной точки монтирования '${MOUNT_POINT}'..."
if [ ! -d "${MOUNT_POINT}" ]; then
    if mkdir -p "${MOUNT_POINT}"; then
        log_message "Точка монтирования '${MOUNT_POINT}' создана."
    else
        log_message "ОШИБКА: Не удалось создать точку монтирования '${MOUNT_POINT}'."
        exit 1
    fi
fi
log_separator

# 2. Монтирование SAMBA ресурса
log_message "Монтирование SAMBA ресурса '${SAMBA_SHARE}' в '${MOUNT_POINT}'..."
# Для безопасности лучше использовать файл credentials: -o credentials=/путь/к/.smbcreds,uid=1000,gid=1000 и т.д.
if ! mount -t cifs -o username="${SAMBA_USER}",password="${SAMBA_PASSWORD}" "${SAMBA_SHARE}" "${MOUNT_POINT}"; then
    log_message "ОШИБКА: Не удалось примонтировать SAMBA ресурс."
    exit 1
fi
log_message "SAMBA ресурс успешно примонтирован."
log_separator

# 3. Подготовка каталога для резервных копий на SAMBA
log_message "Проверка и создание каталога для резервных копий '${BACKUP_DIR}' на SAMBA..."
if [ ! -d "${BACKUP_DIR}" ]; then
    if mkdir -p "${BACKUP_DIR}"; then
        log_message "Каталог для резервных копий '${BACKUP_DIR}' создан."
    else
        log_message "ОШИБКА: Не удалось создать каталог '${BACKUP_DIR}' на SAMBA ресурсе."
        exit 1 # Выходим, так как некуда сохранять резервные копии
    fi
fi
log_separator

# Начинаем полноценное логирование в файл на примонтированном ресурсе
# (Начальные сообщения выводились только в консоль, пока монтирование не было подтверждено)
echo -e "\n\n$(date +'%F %R:%S') - Начало процесса резервного копирования..." >> "${LOG_FILE}"
echo -e "$(date +'%F %R:%S') - =========================" >> "${LOG_FILE}"


# Переходим в домашний каталог пользователя (если нужно для относительных путей, хотя сейчас используются абсолютные)
cd "${USER_HOME}" || { log_message "ОШИБКА: Не удалось перейти в каталог ${USER_HOME}"; exit 1; }

# 4. Резервное копирование снапшотов etcd
log_message "Резервное копирование снапшотов etcd из '${ETCD_SNAPSHOT_DIR}'..."
etcd_backup_file="${BACKUP_DIR}/etcd-------$(date +"${DATE_FORMAT}").zip"
if /usr/bin/zip -r "${etcd_backup_file}" "${ETCD_SNAPSHOT_DIR}"; then
    log_message "Снапшоты etcd сохранены в ${etcd_backup_file}."
else
    log_message "ОШИБКА: Не удалось создать резервную копию снапшотов etcd."
    # Решите, является ли это критической ошибкой или скрипт может продолжаться
fi
log_separator

# 5. Резервное копирование манифестов
log_message "Резервное копирование манифестов из '${MANIFESTS_DIR}'..."
manifests_backup_file="${BACKUP_DIR}/manifests--$(date +"${DATE_FORMAT}").zip"
if /usr/bin/zip -r "${manifests_backup_file}" "${MANIFESTS_DIR}"; then
    log_message "Манифесты сохранены в ${manifests_backup_file}."
else
    log_message "ОШИБКА: Не удалось создать резервную копию манифестов."
fi
log_separator

# 6. Резервное копирование секретов Kubernetes
log_message "Резервное копирование секретов Kubernetes..."
secrets_backup_file="${BACKUP_DIR}/secrets----$(date +"${DATE_FORMAT}").zip"
# Безопасно создаем временный каталог
tmp_secrets_dir=$(mktemp -d -t k8s-secrets-backup-XXXXXX)
if [[ -z "$tmp_secrets_dir" || ! -d "$tmp_secrets_dir" ]]; then
    log_message "ОШИБКА: Не удалось создать временный каталог для резервной копии секретов."
else
    log_message "Создан временный каталог для секретов: ${tmp_secrets_dir}"
    secrets_exported=false
    # Получаем все пространства имен, исключая некоторые системные (при необходимости)
    namespaces=$(kubectl get ns -o jsonpath='{.items[*].metadata.name}')
    # Если нужно, настройте исключаемые из резервного копирования пространства имен
    # namespaces=$(kubectl get ns -o jsonpath='{.items[*].metadata.name}' --field-selector metadata.name!=kube-system,metadata.name!=kube-public,metadata.name!=kube-node-lease,metadata.name!=default,metadata.name!=longhorn-system,metadata.name!=cert-manager)

    for ns in $namespaces; do
        log_message "Экспорт секретов из пространства имен: ${ns}"
        # Определяем путь к файлу вывода во временном каталоге
        secret_file="${tmp_secrets_dir}/secrets-${ns}.yaml"
        # Экспортируем секреты во временный файл
        if kubectl get secret -n "${ns}" -o yaml > "${secret_file}"; then
            # Проверяем, не пустой ли файл (если в namespace нет секретов)
            if [[ -s "${secret_file}" ]]; then
                 log_message "Успешно экспортированы секреты для пространства имен ${ns} в ${secret_file}"
                 secrets_exported=true
            else
                 log_message "В пространстве имен ${ns} нет секретов, пропускаем."
                 rm "${secret_file}" # Удаляем пустой файл
            fi
        else
            log_message "ПРЕДУПРЕЖДЕНИЕ: Не удалось экспортировать секреты для пространства имен ${ns}. Возможно, оно пустое или недоступно."
            # Удаляем файл, если он был создан, но команда завершилась с ошибкой
            [ -f "${secret_file}" ] && rm "${secret_file}"
        fi
    done

    # Архивируем собранные секреты из временного каталога
    if [ "$secrets_exported" = true ]; then
        # Используем флаг -j, чтобы не сохранять структуру временного каталога в архиве
        if /usr/bin/zip -j "${secrets_backup_file}" "${tmp_secrets_dir}"/*; then
             log_message "Секреты сохранены в ${secrets_backup_file}."
        else
             log_message "ОШИБКА: Не удалось заархивировать экспортированные секреты."
        fi
    else
        log_message "Секреты для экспорта не найдены, создание архива пропущено."
    fi

    # Очищаем временный каталог
    log_message "Удаление временного каталога секретов: ${tmp_secrets_dir}"
    rm -rf "${tmp_secrets_dir}"
fi
log_separator


# 7. Резервное копирование PVC (Заглушка - Требуется отдельная стратегия, например, Velero или бэкап Longhorn)
log_message "Секция резервного копирования PVC - Заглушка."
log_message "Примечание: Резервное копирование данных PVC требует специальной стратегии, такой как Velero (velero.io) или встроенные функции резервного копирования Longhorn."
# Пример использования бэкапа Longhorn (концептуально - требует настройки Longhorn):
# longhorn backup create my-pvc-backup --dest s3://my-backup-bucket/longhorn/
log_separator


# 8. Очистка старых резервных копий
log_message "Очистка старых резервных копий старше ${RETENTION_DAYS} дней в '${BACKUP_DIR}'..."
# Ищем и удаляем старые zip-файлы и файлы журналов, соответствующие шаблонам
# Используем -maxdepth 1, чтобы случайно не удалить файлы во вложенных каталогах
deleted_files=$(/usr/bin/find "${BACKUP_DIR}" -maxdepth 1 -type f \( -name "etcd-------*.zip" -o -name "manifests--*.zip" -o -name "secrets----*.zip" -o -name "log-backup-*.log" \) -mtime +"${RETENTION_DAYS}" -print -delete)

if [[ -n "$deleted_files" ]]; then
    log_message "Удалены старые файлы резервных копий:"
    echo "$deleted_files" | while IFS= read -r file; do log_message "  - $file"; done
else
    log_message "Старые файлы резервных копий для удаления не найдены."
fi
log_separator

# 9. Список текущих резервных копий
log_message "Текущие резервные копии в '${BACKUP_DIR}':"
ls -alhcrt "${BACKUP_DIR}" >> "${LOG_FILE}" # Записываем подробный список в журнал
ls -alhcrt "${BACKUP_DIR}" # Показываем список на консоли
log_separator

# 10. Ротация файла журнала
log_message "Ротация файла журнала..."
log_backup_file="${BACKUP_DIR}/log-backup-$(date +"${DATE_FORMAT}").log"
if mv "${LOG_FILE}" "${log_backup_file}"; then
    log_message "Файл журнала перемещен в ${log_backup_file}."
else
    log_message "ОШИБКА: Не удалось переместить файл журнала ${LOG_FILE}."
fi
log_separator

# 11. Размонтирование и выход (Обрабатывается через trap)
log_message "Процесс резервного копирования завершен. Размонтирование произойдет при выходе."
# механизм `trap cleanup_and_exit EXIT` размонтирует SAMBA при выходе из скрипта. 
# и поэтому явный `umount "${MOUNT_POINT}"` не нужен.

# Явный выход с кодом 0, если мы дошли до сюда без ошибок
exit 0
```

Добавим скрипт в системный cron (root):
```bash
sudo crontab -e
```

Например, добавим в cron запуск скрипта каждый день в 2:10:
```text
# Резервное копирование K3S
10 2 * * * /usr/bin/bash /home/opi/script/backup-k3s.sh
```