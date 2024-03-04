[![Alt text](cat.png)](https://docs.arenadata.io/ru/ADQM/current/introduction/intro.html)
## Документация по управлению бэкапами и ресторами кластера ADQM с помощью clickhouse-backup

### Оглавление

0. [Тест и сравнение clickhouse-backup с другими](test_clickhouse_backup.pdf)
1. [Установка clickhouse-backup](#установка)
2. [Настройка clickhouse-backup](#настройка)
3. [Резервное копирование и восстановление](#резервное-копирование-и-восстановление)

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

## Резервное копирование и восстановление





[![Alt text](kisspng.png)](https://docs.arenadata.io/ru/ADQM/current/introduction/intro.html)