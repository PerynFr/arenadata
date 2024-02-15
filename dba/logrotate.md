## Создай новый файл конфигурации, например, nginx-logrotate, в директории /etc/logrotate.d/:
```
sudo nano /etc/logrotate.d/nginx
```
## Добавь следующие строки в файл nginx-logrotate:
```
/opt/adcm/log/nginx/access.log {
    monthly
    rotate 12
    compress
    delaycompress
    missingok
    notifempty
    create 0640 nginx nginx
```

monthly: Указывает, что ротация должна выполняться раз в месяц.  
rotate 12: Хранит последние 12 архивных файлов логов.  
compress: Сжимает архивные файлы.  
delaycompress: Задерживает сжатие до следующей ротации.  
missingok: Не выдает ошибку, если лог-файл отсутствует.  
notifempty: Не выполняет ротацию, если лог-файл пуст.  
create 0640 nginx nginx: Создает новый лог-файл с указанными правами доступа и владельцем.  
