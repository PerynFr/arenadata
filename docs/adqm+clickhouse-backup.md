[![Alt text](cat.png)](https://docs.arenadata.io/ru/ADQM/current/introduction/intro.html)
## Документация по управлению бэкапами и ресторами кластера ADQM с помощью clickhouse-backup

### Оглавление

- [Тест и сравнение clickhouse-backup с другими](test_clickhouse_backup.pdf)
- [Установка clickhouse-backup](#установка)
- [Настройка clickhouse-backup](#настройка)
- [Резервное копирование](#резервное-копирование)
- [Восстановление](#восстановление)

---

### Установка:

Для установки `clickhouse-backup` выполните следующие шаги:

скачайте пакет из офф. репозитория [проекта](https://github.com/Altinity/clickhouse-backup/releases)  
в сответствии с архитектурой вашей системы.  
офлайн установка:
```sh
wget https://github.com/Altinity/clickhouse-backup/releases/download/v2.4.33/clickhouse-backup-2.4.33-1.x86_64.rpm
```
установите пакет  
```sh
sudo rpm -ivh clickhouse-backup-2.4.33-1.x86_64.rpm
```

### Настройка:


После установки, настройте параметры подключения к кластеру ClickHouse в файле конфигурации `config.yaml`.  
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

в секции general укаживается remote_storage и какое количество бэкапов будет хранится в ротации локально и в удаленном хосте (полный бэкап не удаляется если нужен в цепочке)
```
general:
    remote_storage: sftp
    max_file_size: 0
    disable_progress_bar: true
    backups_to_keep_local: 7
    backups_to_keep_remote: 7

```

в секции sftp укажем ip хоста для удаленного хранения бэкапов address: и дирректорию path:  
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

создаем пользователя backupadmin в на хосте куда будут переноситься бэкапы и делаем его владельцем диррректории где будут хранится бэкапы
```sh
sudo chown backupadmin /backup_upload
```
создаем пользователя в системе backupadmin где расположена база, создаем ключ и копируем публичный ключ на хост где будут хранится бэкапы
```sh
sudo -u backupadmin ssh-keygen -t ed25519 -f /home/backupadmin/.ssh/id_ed25519_backupadmin
ssh-copy-id -i /home/backupadmin/.ssh/id_ed25519_backupadmin.pub backupadmin@10.92.36.11
```
clickhouse-backup может работать от root или от имени сервисной учетной записи clickhouse  
логин backupadmin будет запускать скрипт bash через крон, в скрипте будет вызов `sudo -u clickhouse clickhouse-backup`  
чтобы это работало дадим права sudo
```sh
visudo
backupadmin ALL=(ALL)  NOPASSWD: /bin/clickhouse-backup
```
скопируем приватный ключ в дирректорию access чтобы он бэкапился вместе с разрешениями в базе  
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
# full/incremental backup parameter
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
    sudo -u clickhouse clickhouse-backup $CREATE_COMMAND $BACKUP_NAME_FULL --backup-rbac --backup-configs
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
    sudo -u clickhouse clickhouse-backup $CREATE_COMMAND --diff-from-remote=$BACKUP_NAME_PREV $BACKUP_NAME_INCREMENTAL --backup-rbac --backup-configs
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
и даем права на выполнение для backupadmin
```sh
sudo chown backupadmin /usr/local/bin/clickhouse-backup-run.sh
sudo chmod u+x /usr/local/bin/clickhouse-backup-run.sh
```

### Резервное копирование

для полного локального резервного копирования:
```
/usr/local/bin/clickhouse-backup-run.sh full local
```
***локальное инкрементальное резервное копирование не предусмотрено***

для полного резервного копирования в удаленное хранилище:
```
/usr/local/bin/clickhouse-backup-run.sh full remote
```
для инкрементального резервного копирования в удаленное хранилище:
```
/usr/local/bin/clickhouse-backup-run.sh incremental remote
```
под логином backupadmin
создаем задачи в cron `crontab -e`
```
# Инкрементное резервное копирование каждые 3 часа, кроме 00:00 по воскресеньям.
0 */3 * * 0-6 /usr/local/bin/clickhouse-backup-run.sh incremental remote
# Полное резервное копирование в 00:00 по воскресеньям.
0 0 * * 0 /usr/local/bin/clickhouse-backup-run.sh full remote
```
смотрим логи
``` 
tail -n 50 -f /var/log/clickhouse-backup.log
```

###  Восстановление

смотрим историю бэкапов в удаленном хранилище

```sh
sudo -u backupadmin clickhouse-backup list remote
```
пример, + обозначает зависимость от предыдущего бэкапа, цепочку формирует скрипт выше  [clickhouse-backup-run.sh](#размещаем)

|--------------------------------------|------------|----------------------|---------|----------------------------------------|-----------------|
| auto_full_2024-03-05T14-30-01        |  472.71MiB |  05/03/2024 14:30:11 |  remote |                                        |   gzip, regular |
| auto_incremental_2024-03-05T14-35-01 |  1.04MiB   |  05/03/2024 14:35:02 |  remote |  +auto_full_2024-03-05T14-30-01        |   gzip, regular | 
| auto_incremental_2024-03-05T14-40-01 |  1.04MiB   |  05/03/2024 14:40:02 |  remote |  +auto_incremental_2024-03-05T14-35-01 |   gzip, regular |
| auto_incremental_2024-03-05T14-45-01 |  1.04MiB   |  05/03/2024 14:45:02 |  remote |  +auto_incremental_2024-03-05T14-40-01 |  gzip, regular  |
| auto_incremental_2024-03-05T14-50-02 |  1.04MiB   |  05/03/2024 14:50:03 |  remote |  +auto_incremental_2024-03-05T14-45-01 |  gzip, regular  |
| auto_incremental_2024-03-05T14-55-01 |  1.04MiB   |  05/03/2024 14:55:02 |  remote |  +auto_incremental_2024-03-05T14-50-02 |  gzip, regular  |
| auto_incremental_2024-03-05T15-00-01 |  1.04MiB   |  05/03/2024 15:00:02 |  remote |  +auto_incremental_2024-03-05T14-55-01 |  gzip, regular  |

