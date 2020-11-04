# Стенд для проектной работы

Цель: Создание рабочего проекта
с развертыванием нескольких виртуальных машин, который
должен отвечать следующим требованиям:

-   ~~включен https~~
-   основная инфраструктура в DMZ зоне
-   файрволл на входе
-   ~~сбор метрик и настроенный алертинг~~
-   везде включен selinux
-   организован централизованный сбор логов
-   настроен бэкап

* * *

Стенд запускается командой `vagrant up`  
Сайт доступен по адресу [http://192.168.110.100](<>)  
Не настроен https (пример настройки есть в неиспользуемой роли nginx ниже)  
Не настроен сбор метрик и алертинг  

* * *

## Создаем виртуальные машины:

### firewall

-   настроим форвардинг (net.ipv4.ip_forward=1)  
    `echo "net.ipv4.conf.all.forwarding=1" >> /etc/sysctl.d/01-forwarding.conf`  
    `echo "net.ipv4.ip_forward=1" >> /etc/sysctl.d/01-forwarding.conf`  
    `sysctl -p /etc/sysctl.d/01-forwarding.conf`  

-   настроим маскарадинг на external интерфейсе (для выхода в инет из локальной сети)  
    `iptables -t nat -A POSTROUTING ! -d 192.168.0.0/16 -o eth0 -j MASQUERADE`

-   раскидаем сетевые интерфейсы по зонам: external (eth0), internal (eth1), dmz (eth2):  
    `firewall-cmd --zone=external --change-interface=eth0`  
    `firewall-cmd --zone=internal --change-interface=eth1`  
    `firewall-cmd --zone=dmz --change-interface=eth2`  
-   на external интерфейсе разрешим вход по http ~~и https~~  
    `firewall-cmd --zone=external --add-service=http`  
-   настроим маскарадинг  
    `firewall-cmd --zone=external --add-masquerade`  
-   настроим проброс http ~~и https~~ на сервер frontend (nginx)  
    `firewall-cmd --zone=external --add-forward-port=port=80:proto=tcp:toport=80:toaddr=192.168.255.2`  

#### Настройка логов

Настройка производится ролью **log**, которая описана ниже

#### Настройка бэкапа

Настройка производится ролью **backup**, которая описана ниже

* * *

### mysql сервер

#### Установка и настройка софта

Устанавливаем percona:  
`yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm`  
`yum install Percona-Server-server-57 MySQL-python`  

Запускаем mysqld  
`systemctl enable --now mysqld`  

Копируем конфигурационные файлы в каталог /etc/my.cnf.d/  

Меняем временный пароль пользователя mysql root:  
Текущий временный пароль:  
`cat /var/log/mysqld.log | grep 'root@localhost:' | awk '{print $11}'`  

`mysql -u root -p`  

    mysql> update user set Password=PASSWORD('новый пароль') WHERE User='root';
    mysql> flush privileges;
    mysql> exit

Создаем базу wordpress и пользователя базы:  
`mysql -u root -p`

    mysql> create database wordpress;
    mysql> create user 'wpuser'@'192.168.110.102' identified by 'пароль пользователя';
    mysql> grant all privileges on wordpress.* to 'wpuser'@'192.168.110.102' identified by 'пароль пользователя';
    mysql> flush privileges;
    mysql> exit

#### Настройка логов

Настройка производится ролью **log**, которая описана ниже

#### Настройка бэкапа

Настройка производится ролью **backup**, которая описана ниже  
Также добавим в бэкап базу wordpress (ежедневно в 23 часа в каталог /var/backup)  
`crontab -l | { cat; echo "* 23 * * * mysqldump wordpress > "/var/backup/database.sql" --set-gtid-purged=OFF"; } | crontab -`

* * *

## Apache (wordpress)

### Установка и настройка софта

`yum install httpd php-fpm`  
`http://rpms.remirepo.net/enterprise/remi-release-7.rpm` # php72  
`yum install yum-config-manager`  
`yum-config-manager --enable remi-php72`  
`yum install php-cli php-mysql php-json php-opcache php-mbstring php-xml php-gd php-curl wget nano 'python2-cryptography'`  

Меняем порт apache:  
`cat /etc/httpd/conf/httpd.conf`  

    Listen 8080

Создаем директорию для сайта и для логов:  
`mkdir -p /var/www/html/otus-project.local/log`  
`chown -R apache:apache /var/www/html/otus-project.local/`  

Создаем конфигурационный файл для сайта:  
`cat /etc/httpd/conf.d/otus-project.local.conf`  

    <VirtualHost *:8080>
        ServerName www.otus-project.local
        ServerAlias otus-project.local
        DocumentRoot /var/www/html/otus-project.local/wordpress
        ErrorLog /var/www/html/otus-project.local/log/error.log
        CustomLog /var/www/html/otus-project.local/log/requests.log combined
    </VirtualHost>

Создаем файл конфигурации php-fpm:  
`cat /etc/php-fpm/www.conf`  

    [www]
    ...
    listen = /run/php-fpm.sock
    listen.owner = apache
    listen.group = apache
    ...

Настроим apache для работы php-fpm:  
`cat /etc/httpd/conf.modules.d/00-mpm.conf`  

    LoadModule mpm_event_module modules/mod_mpm_event.so

`cat /etc/httpd/conf.d/php.conf`  

    <Proxy "unix:/var/run/php-fpm/default.sock|fcgi://php-fpm">
        	ProxySet disablereuse=off
    </Proxy>
    <FilesMatch \.php$>
    	SetHandler proxy:fcgi://php-fpm
    </FilesMatch>
    AddType text/html .php
    DirectoryIndex index.php

Стартуем сервисы:  
`systemctl enable --now httpd`  
`systemctl enable --now php-fpm`  

#### Установка WordPress

Скачиваем wordpress:  
`cd /var/www/html/otus-project.local`  
`wget https://wordpress.org/latest.tar.gz`  
`tar -xvf latest.tar.gz`  

Настройка подключения к БД:  
`cat /var/www/html/otus-project.local/wordpress/wp-config.php`  

    define('DB_NAME', 'wordpress');
    define('DB_USER', 'wpuser');
    define('DB_PASSWORD', 'пароль пользвателя');
    define('DB_HOST', '192.168.110.136');

#### Selinux и права доступа

`chown -R apache:apache /var/www/html/otus-project.local/`  

`chcon -t httpd_sys_content_t /var/www/html/otus-project.local/wordpress` # доступ только чтение  
`chcon -t httpd_sys_rw_content_t /var/www/html/otus-project.local/wordpress/wp-config.php` # разрешим запись в файл настроек  
`chcon -t httpd_sys_rw_content_t /var/www/html/otus-project.local/wordpress/wp-content -R`  # и разрешим запись в каталог wp-content  
`chcon -t httpd_log_t /var/www/html/otus-project.local/log -R`

`setsebool -P httpd_can_network_connect 1`  
`setsebool -P httpd_can_network_connect_db 1`  
`setsebool -P httpd_unified 1`  
`firewall-cmd --zone=internal --add-port=8080/tcp --permanent`  

Для настройки reverse-proxy добавим маршрут на frontend (nginx):  
`cat /etc/sysconfig/network-script/route-eth1`  

    192.168.255.0/3 via 192.168.110.100  

#### Настройка логов

Настройка производится ролью **log**, которая описана ниже

#### Настройка бэкапа

Настройка производится ролью **backup**, которая описана ниже  
Также добавим в бэкап каталог html (ежедневно в 23 часа в каталог /var/backup)  
`crontab -l | { cat; echo "_ 23 _ \* \* rsync -r /var/www/html/otus-project.local/ /var/backup/"; } | crontab -`  

* * *

### Роль log

Настраивается на каждой машине. Отправляет все логи на машину log  
Проверка логов: `journalctl -D /var/log/journal/remote --follow`

#### Настроим отправку логов на удаленную машину

Устанавливаем journal-gateway:  
`yum install systemd-journal-gateway`  
Создаем директорию для логов:  
`mkdir /var/log/journal`  
Настройки для journal-upload:  
`cat /etc/systemd/journal-upload.conf`  

    [Upload]
    URL=http://192.168.110.103:19532

После этого перезапускаем journald и стартуем journald-upload:  
`systemctl restart journald`  
`systemctl enable --now systemd-journal-upload`  

* * *

### Роль backup

Со всех машин бэкапятся каталоги /etc на удаленный сервер. На некоторых машинах настроен бэкап в локальный каталог /var/backup, который позже также бэкапится на удаленный сервер.  

Устанавливаем borgbackup на все машины:  
`yum install -y borgbackup`  

На сервере создаем пользователя borg:  
`useradd -m borg`  
`echo password | passwd borg --stdin`  

Разрешаем вход ssh по паролю:  
`sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config`  
`systemctl restart sshd`

На клиентах настраиваем вход по ssh-ключам и копируем ключ на сервер:  
`ssh-keygen -b 2048 -t rsa -q -N '' -f ~/.ssh/id_rsa`  
`ssh-copy-id borg@192.168.110.104`

Инициализируем с клиента репозитории для бэкапов (для каждой машины свой):  
`borg init -e repokey borg@192.168.110.104:/var/backup/имя машины`  

Создадим скрипт для автоматического создания резервных копий  
`cat /root/backup.sh`  

    #!/bin/bash
    export BORG_RSH="ssh -i /root/.ssh/id_rsa"
    export BORG_REPO=ssh://borg@192.168.110.104/var/backup/имя машины
    export BORG_PASSPHRASE='password'
    LOG="/var/log/borg_backup.log"
    [ -f "$LOG" ] || touch "$LOG"
    exec &> >(tee -i ""$LOG")
    exec 2>&1
    echo "Starting backup"
    borg create --verbose --list --stats ::'{now:%Y-%m-%d_%H:%M:%S}' /etc /var/backup                            

    echo "Pruning repository"
    borg prune --list --keep-daily 90 --keep-monthly 12

Добавим ротацию лога /var/log/borg_backup.log:  
`cat /etc/logrotate.d/borg_backup.conf`  

    /var/log/borg_backup.log {
      rotate 5
      missingok
      notifempty
      compress
      size 1M
      daily
      create 0644 root root
      postrotate
        service rsyslog restart > /dev/null
      endscript
    }

Добавим задачу в cron:  
`crontab -l | { cat; echo "* 1 * * * /root/backup.sh"; } | crontab -`   

* * *

### Роль routing

Используется для настройки сетевых интерфейсов и файрвола на всех машинах  

-   отключен маршрут по умолчанию на вагрант
-   настроен маршрут по умолчанию на firewall
-   запускается файрволл
-   зона по умолчанию internal

* * *

### ~~nginx сервер (бэкенд)~~

#### Настраивался изначально, затем заменил на apache

#### Установка и настройка софта и wordpress

Устанавливаем и запускаем nginx  
`yum install nginx`  
`systemctl enable --now nginx`  

Устанавливаем репозиторий и модули php72 (и некоторый софт)  
`yum install http://rpms.remirepo.net/enterprise/remi-release-7.rpm`  
`yum install yum-config-manager`  
`yum-config-manager --enable remi-php72`  
`yum install php-cli php-fpm php-mysql php-json php-opcache php-mbstring php-xml php-gd php-curl wget nano`  

Создаем файл конфигурации php-fpm:  
`cat /etc/php-fpm/www.conf`  

    [www]
    user = nginx
    group = nginx
    listen = /run/php-fpm.sock
    listen.owner = nginx
    listen.group = nginx  

Копируем файлы конфигураци nginx (созданы на <https://nginxconfig.io/>)  

Создаем симлинк sites-available to sites-enabled  
`ln -s /etc/nginx/sites-available /etc/nginx/sites-enabled`  

Проверим и перечитаем конфигурацию nginx:  
`nginx -t`  
`systemctl reload nginx`  
Загружаем и распаковываем wordpress  
`cd /tmp`  
`wget https://wordpress.org/latest.tar.gz`  
`tar -xvf latest.tar.gz`  
`mv ./wordpress/* /var/www/html/otus-project.local/wordpress`  

Поправим права доступа:  
`chown -R nginx:nginx /var/www/html/otus-project.local/wordpress`  
Исправим права доступа selinux:  
`chcon -t httpd_sys_content_t /var/www/html/otus-project.local/wordpress` # доступ только чтение  
`chcon -t httpd_sys_rw_content_t /var/www/html/otus-project.local/wordpress/wp-config.php` # разрешим запись в файл настроек  
`chcon -t httpd_sys_rw_content_t /var/www/html/otus-project.local/wordpress/wp-content -R`  # и разрешим запись в каталог wp-content  

Ещё пара правил для php-fpm  
`setsebool -P httpd_can_network_connect`  
`setsebool -P httpd_can_network_connect_db`  

#### Настройка ssl для сайта:

Сертификат самоподписанный, в браузере необходимо разрешить открытие сайта  
Создадим директорию /etc/ssl/private и настроим права на неё  
`mkdir /etc/ssl/private`  
`chmod 700 /etc/ssl/private`  
Создаем сертификаты:  
`openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt`

Создадим dhparm:  
`openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048`  

#### Настройка аудита

Установим audisp-remote  
`yum install audisp-remote`  

Конфигурация audisp-remote:  
`cat /etc/audisp/audisp-remote.conf`  

    remote_server = 192.168.110.103 # сервер логов
    port = 60
    transport = tcp

`cat /etc/audisp/plugins.d/au-remote.conf`  

    active = yes
    direction = out
    path = /sbin/audisp-remote
    type = always
    format = string

Правила аудита:  
`cat /etc/audit/rules.d/nginx.rules`

    -w /etc/nginx/ -k nginx-conf

#### Настройка логов

Настройка производится ролью **log**, которая описана ниже

#### Настройка бэкапа

Настройка производится ролью **backup**, которая описана ниже  
Также добавим в бэкап каталог html (ежедневно в 23 часа в каталог /var/backup)  
`crontab -l | { cat; echo "* 23 * * * rsync -r /var/www/html/otus-project.local/ /var/backup/"; } | crontab -`

### Спасибо за проверку!

P.S. По факту львиную долю времени заняла борьба с косяками и багами вагранта и ансибла.  
Вручную такой стенд поднимается на раз-два.  
