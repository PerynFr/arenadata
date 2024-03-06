[![Alt text](cat.png)](https://docs.arenadata.io/ru/ADQM/current/introduction/intro.html)
## Документация по управлению  
## бэкапами и ресторами кластера ADQM с помощью clickhouse-backup

### Оглавление

- [Тест и сравнение (Clickhouse-backup) с (Clickhouse Backup & Restore)](test_clickhouse_backup.pdf)
- [Установка clickhouse-backup](#установка)
- [Настройка clickhouse-backup](#настройка)
- [Резервное копирование](#резервное-копирование)
- [Восстановление](#восстановление)
- [Дополнение](#дополнение)

---

### Установка:

- Для установки `clickhouse-backup` выполните следующие шаги:

- скачайте пакет из офф. репозитория [проекта](https://github.com/Altinity/clickhouse-backup/releases)  
    в сответствии с архитектурой вашей системы.  
- офлайн установка:
```sh
wget https://github.com/Altinity/clickhouse-backup/releases/download/v2.4.33/clickhouse-backup-2.4.33-1.x86_64.rpm
```
- установите пакет  
```sh
sudo rpm -ivh clickhouse-backup-2.4.33-1.x86_64.rpm
```

### Настройка:


- После установки, настройте параметры подключения к кластеру ClickHouse в файле конфигурации `config.yaml`.  
    и какие схемы будут пропускаться при бэкапе
- создайте файл 
```sh
sudo cp /etc/clickhouse-backup/config.yml.example /etc/clickhouse-backup/config.yml
```
```
clickhouse:
    username: default
    password: ""
    host: localhost
    port: 9000
    disk_mapping: {}
    skip_tables:
        - system.*
        - INFORMATION_SCHEMA.*
        - information_schema.*
        - _temporary_and_external_tables.*
    skip_table_engines: []
    timeout: 5m
    freeze_by_part: false
    freeze_by_part_where: ""
    use_embedded_backup_restore: false
    embedded_backup_disk: ""
    backup_mutations: true
    restore_as_attach: false
    check_parts_columns: true
    secure: false
    skip_verify: false
    sync_replicated_tables: false
    log_sql_queries: true
    config_dir: /etc/clickhouse-server/
    restart_command: exec:systemctl restart clickhouse-server
    ignore_not_exists_error_during_freeze: true
    check_replicas_before_attach: true
    tls_key: ""
    tls_cert: ""
    tls_ca: ""
    max_connections: 2
    debug: false
```

- в секции general укаживается remote_storage и какое количество бэкапов будет хранится в ротации локально и в удаленном хосте (полный бэкап не удаляется если нужен в цепочке)
```
general:
    remote_storage: sftp
    max_file_size: 0
    disable_progress_bar: true
    backups_to_keep_local: 7
    backups_to_keep_remote: 7

```

- в секции sftp укажем ip хоста для удаленного хранения бэкапов address: и дирректорию path:  
    а также способ авторизации по паролю или ключу, concurrency число потоков
```
sftp:
    address: "10.92.36.11"
    timeout: 1m
    username: "backupadmin"
    key: "/var/lib/clickhouse/id_ed25519_backupadmin"
    path: "/backup_upload"  # This path has been created beforehand
    compression_format: gzip
    compression_level: 3
    concurrency: 4
    debug: true
```

- создаем пользователя backupadmin в на хосте куда будут переноситься бэкапы и делаем его владельцем диррректории где будут хранится бэкапы
```sh
sudo chown backupadmin /backup_upload
```
- создаем пользователя в системе backupadmin где расположена база, создаем ключ и копируем публичный ключ на хост где будут хранится бэкапы
```sh
sudo -u backupadmin ssh-keygen -t ed25519 -f /home/backupadmin/.ssh/id_ed25519_backupadmin
ssh-copy-id -i /home/backupadmin/.ssh/id_ed25519_backupadmin.pub backupadmin@10.92.36.11
```
- clickhouse-backup может работать от root или от имени сервисной учетной записи clickhouse  
    логин backupadmin будет запускать скрипт bash через крон, в скрипте будет вызов `sudo -u clickhouse clickhouse-backup`  
    чтобы это работало дадим права sudo
```sh
visudo
backupadmin ALL=(ALL)  NOPASSWD: /bin/clickhouse-backup
```
- скопируем приватный ключ в дирректорию access чтобы он бэкапился вместе с разрешениями в базе  
    и сделаем clickhouse:clickhouse владельцами
```sh
cp /home/backupadmin/.ssh/id_ed25519_backupadmin /opt/lib/clickhouse/access
chown clickhouse:clickhouse /opt/lib/clickhouse/access/id_ed25519_backupadmin
```
в файле `/etc/clickhouse-backup/config.yml` указываем этот путь  
```
sftp:
    username: "backupadmin"
    key: "/var/lib/clickhouse/id_ed25519_backupadmin"
```

#### размещаем
 скрипт в: 
`sudo vim /usr/local/bin/clickhouse-backup-run.sh`  

```bash
#!/bin/bash
# sudo vim /etc/default/clickhouse-backup-run.sh
# (full/incremental local/remote debug) backup parameter
if [[ $1 == "full" ]]; then
    IS_FULL=true
elif [[ $1 == "incremental" ]]; then
    IS_FULL=false
else
    echo "full/incremental not specified" >> /var/log/clickhouse-backup.log 2>&1
    echo "ERROR!!! return $exit_code, see in cat /var/log/clickhouse-backup.log"
    exit 1;
fi
# local/remote backup parameter
if [[ $2 == "local" ]]; then
    CREATE_COMMAND="create"
elif [[ $2 == "remote" ]]; then
    CREATE_COMMAND="create_remote"
else
    echo "local/remote not specified" >> /var/log/clickhouse-backup.log 2>&1
    echo "ERROR!!! return $exit_code, see in cat /var/log/clickhouse-backup.log"
    exit 1;
fi
DATETIME=$(date -u +%Y-%m-%dT%H-%M-%S)
BACKUP_NAME_FULL="auto_full_$DATETIME"
BACKUP_NAME_INCREMENTAL="auto_incremental_$DATETIME"
# monitoring API call, f.e. healthchecks.io
BACKUP_HEALTH_CHECK=$BACKUP_HEALTH_CHECK
if [[ $IS_FULL == true ]]; then
    echo "Starting full ($CREATE_COMMAND) backup" >> /var/log/clickhouse-backup.log 2>&1
    echo "Creating backup $BACKUP_NAME_FULL" >> /var/log/clickhouse-backup.log 2>&1
    if [[ $3 == "debug" ]]; then
        sudo -u clickhouse clickhouse-backup $CREATE_COMMAND $BACKUP_NAME_FULL --backup-rbac --backup-configs >> /var/log/clickhouse-backup.log 2>&1
    else 
        sudo -u clickhouse clickhouse-backup $CREATE_COMMAND $BACKUP_NAME_FULL --backup-rbac --backup-configs
    fi
    exit_code=$?
    if [[ $exit_code != 0 ]]; then
        echo "clickhouse-backup $CREATE_COMMAND $BACKUP_NAME_FULL FAILED and return $exit_code exit code" >> /var/log/clickhouse-backup.log 2>&1
        exit $exit_code
        echo "ERROR!!! return $exit_code, see in cat /var/log/clickhouse-backup.log"
    else
         if [[ $2 == "remote" ]]; then
             sudo -u clickhouse clickhouse-backup delete local $BACKUP_NAME_FULL
             sudo -u clickhouse clickhouse-backup clean
         fi
         echo "SUCCESS!!!" >> /var/log/clickhouse-backup.log 2>&1
    fi
else
    echo "Starting incremental ($CREATE_COMMAND) backup" >> /var/log/clickhouse-backup.log 2>&1
    # for incremental backup, take previous backup name
    # based on dates in auto generated backup names.
    BACKUP_NAME_PREV="$(sudo -u clickhouse clickhouse-backup list $2 | grep -E '^auto_' | tail -n 1 | cut -d " " -f 1)"
    echo "Creating backup $BACKUP_NAME_INCREMENTAL as diff from $BACKUP_NAME_PREV" >> /var/log/clickhouse-backup.log 2>&1
    if [[ $3 == "debug" ]]; then
        sudo -u clickhouse clickhouse-backup $CREATE_COMMAND --diff-from-remote=$BACKUP_NAME_PREV $BACKUP_NAME_INCREMENTAL --backup-rbac --backup-configs >> /var/log/clickhouse-backup.log 2>&1
    else 
        sudo -u clickhouse clickhouse-backup $CREATE_COMMAND --diff-from-remote=$BACKUP_NAME_PREV $BACKUP_NAME_INCREMENTAL --backup-rbac --backup-configs
    fi
    exit_code=$?
    if [[ $exit_code != 0 ]]; then
        echo "clickhouse-backup create $BACKUP_NAME_INCREMENTAL FAILED and return $exit_code exit code" >> /var/log/clickhouse-backup.log 2>&1
        exit $exit_code
        echo "ERROR!!! return $exit_code, see in cat /var/log/clickhouse-backup.log"
    else
         if [[ $2 == "remote" ]]; then
            sudo -u clickhouse clickhouse-backup delete local $BACKUP_NAME_INCREMENTAL
            sudo -u clickhouse clickhouse-backup clean
         fi
         echo "SUCCESS!!!" >> /var/log/clickhouse-backup.log 2>&1
    fi
fi
```
- и даем права на выполнение для backupadmin
```sh
sudo chown backupadmin /usr/local/bin/clickhouse-backup-run.sh
sudo chmod u+x /usr/local/bin/clickhouse-backup-run.sh
```

### Резервное копирование

- для полного локального резервного копирования:
```
/usr/local/bin/clickhouse-backup-run.sh full local
```
***локальное инкрементальное резервное копирование не предусмотрено***

- для полного резервного копирования в удаленное хранилище:
```
/usr/local/bin/clickhouse-backup-run.sh full remote
```
- для инкрементального резервного копирования в удаленное хранилище:
```
/usr/local/bin/clickhouse-backup-run.sh incremental remote
```
- под логином backupadmin
создаем задачи в cron `crontab -e`
```
# Инкрементное резервное копирование каждые 3 часа, кроме 00:00 по воскресеньям.
0 */3 * * 0-6 /usr/local/bin/clickhouse-backup-run.sh incremental remote
# Полное резервное копирование в 00:00 по воскресеньям.
0 0 * * 0 /usr/local/bin/clickhouse-backup-run.sh full remote
```
для записи отладки в лог можно добавить параметр в конце `debug`
```
0 0 * * 0 /usr/local/bin/clickhouse-backup-run.sh full remote debug
```
- смотрим логи
``` 
tail -n 50 -f /var/log/clickhouse-backup.log
```

###  Восстановление

- смотрим историю бэкапов в удаленном хранилище или локальном

```sh
sudo -u clickhouse clickhouse-backup list
sudo -u clickhouse clickhouse-backup list local
sudo -u clickhouse clickhouse-backup list remote
```
- знак "+" обозначает зависимость от предыдущего бэкапа, цепочку формирует скрипт выше  [clickhouse-backup-run.sh](#размещаем)
---
|       name                           |  size    |  date                |  locate |  dependency name                       |  compression    |
|--------------------------------------|----------|----------------------|---------|----------------------------------------|-----------------|
| auto_full_2024-03-05T14-30-01        |  472MiB  |  05/03/2024 14:30:11 |  remote |                                        |  gzip, regular  |
| auto_incremental_2024-03-05T14-35-01 |  1.04MiB |  05/03/2024 14:35:02 |  remote |  +auto_full_2024-03-05T14-30-01        |  gzip, regular  | 
| auto_incremental_2024-03-05T14-40-01 |  1.04MiB |  05/03/2024 14:40:02 |  remote |  +auto_incremental_2024-03-05T14-35-01 |  gzip, regular  |
| auto_incremental_2024-03-05T14-45-01 |  1.04MiB |  05/03/2024 14:45:02 |  remote |  +auto_incremental_2024-03-05T14-40-01 |  gzip, regular  |
| auto_incremental_2024-03-05T14-50-02 |  1.04MiB |  05/03/2024 14:50:03 |  remote |  +auto_incremental_2024-03-05T14-45-01 |  gzip, regular  |
| auto_incremental_2024-03-05T14-55-01 |  1.04MiB |  05/03/2024 14:55:02 |  remote |  +auto_incremental_2024-03-05T14-50-02 |  gzip, regular  |
| auto_incremental_2024-03-05T15-00-01 |  1.04MiB |  05/03/2024 15:00:02 |  remote |  +auto_incremental_2024-03-05T14-55-01 |  gzip, regular  |
---
- выбираем бэкап с нужном датой и запускаем рестор, вся цепочка от полного бэкапа применится автоматически, и будут восстановлены данные,  
    метаданные, права и и конфигурационные файлы
```sh
sudo -u backupadmin clickhouse-backup restore_remote --drop --restore-rbac --configs "auto_incremental_2024-03-05T15-00-01" 
```

- если нужно восстановить отдельные таблицы и поместить их в другую схему для последующего сравнения
```sh
sudo -u backupadmin clickhouse-backup restore_remote \
	-t 'default.foo,default.bar' \
	-m 'default:default2' "auto_incremental_2024-03-05T15-00-01"
``` 

- восстановить только одну таблицу
```sh
sudo -u backupadmin clickhouse-backup restore_remote --drop \
	"auto_full_2024-03-05T14-30-01" -t 'default.foo' 
```

- восстановить только одну базу
```sh
sudo -u clickhouse clickhouse-backup restore_remote \
    "auto_full_2024-03-05T14-30-01" -t 'default.*'
```
### Дополнение

- Для очистки потерянных данных в /var/lib/clickhouse/shadow:
```sh
sudo -u clickhouse clickhouse-backup clean
```
- Удалить бэкап в можно командой, для этого добжны быть разрешения на операцию
```
sudo -u clickhouse clickhouse-backup delete remote "auto_incremental_2024-03-06T09-00-02"
```

- разрешить пустые копии `allow_empty_backups: true`
- Подробнее об инкрементном резервном копировании:
    минимальным элементом приращения для расчета приращения является data part name.
    приращение будет расти, если вы часто используете 
```
OPTIMIZE... FINAL или ALTER.
TABLE ... UPDATE / DELETE
```
- Приращение рассчитывается только на этапе загрузки
    команда Create всегда создает полную резервную копию

- Команды выше создают много новых частей данных для существующих данных.
    части, присутствующие в базовой резервной копии, помечены как обязательные в файле `Metadata/db/table.json`.
    во время загрузки необходимые части будут загружены из базы удаленного резервного копирования на локальный диск.

- ClickHouse-backup создает жесткие ссылки в папке `backup_name/shadow` для завершения работы.

##### [к оглавлению](#оглавление)