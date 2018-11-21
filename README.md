# Настройка сервера для проекта (Nginx, PHP-FPM, Elasticsearch, RabbitMQ)

## Подготовка сервера

1. ОС: CentOS 7
2. Сервер для анализа и поиска данных: Elasticsearch
3. Сервер очередей: RabbitMQ
4. Веб сервер: Nginx + PHP7 FPM
5. Framework для импорта и отрисовки данных: Yii2

### Установка ОС CentOS 7

Скачиваем ОС с официального сайта https://www.centos.org/ 

Устанавливаем систему с минимальным набором программ и настраиваем сеть. 

Добавляем репозиторий Remi. 
В папку `/etc/yum.repos.d/` добавляем файл с настройками репозитария:

```
[remi]
name=Les RPM de remi pour Enterprise Linux $releasever - $basearch
#baseurl=http://rpms.famillecollet.com/enterprise/$releasever/remi/$basearch/
mirrorlist=http://rpms.famillecollet.com/enterprise/$releasever/remi/mirror
enabled=1
priority=10
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-remi
failovermethod=priority
```

После добавляем ключ Remi:
```bash
root# rpm --import http://rpms.famillecollet.com/RPM-GPG-KEY-remi
```

Обновляем систему и устанавливаем сопутствующий набор программ:

```bash
root# yum install epel-release -y
root# yum update -y
root# yum groupinstall 'Development tools' -y
root# yum install gcc gcc-c++ glibc-devel make ncurses-devel openssl-devel autoconf java-1.8.0-openjdk-devel git wget wxBase.x86_64
```

Установка Java JDK:

```bash
root# yum install java-1.8.0-openjdk -y
```

На этом подготовка ОС завершена.

### Установка Elasticsearch

Проверяем установлена ли Java:

```bash
root# java -version
root# echo $JAVA_HOME
```

Скачиваем и устанавливаем ключ:

```bash
root# rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

В папку `/etc/yum.repos.d/` добавляем файл с настройками репозитария:

```bash
[elasticsearch-5.x]
name=Elasticsearch repository for 5.x packages
baseurl=https://artifacts.elastic.co/packages/5.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```

Установка Elasticsearch:

```bash
root# yum install elasticsearch -y
```


Настраиваем Firewall, открываем порты для Elasticsearch

```bash
root# firewall-cmd --permanent --add-port=9200/tcp
root# firewall-cmd --permanent --add-port=9300/tcp
root# firewall-cmd --reload
```

Запускаем Elasticsearch и добавляем в автозагрузку

```bash
root# systemctl start elasticsearch
root# systemctl enable elasticsearch
root# systemctl status elasticsearch
```

### Установка RabbitMQ

Установка Erlang

```bash
root# wget http://packages.erlang-solutions.com/erlang-solutions-1.0-1.noarch.rpm
root# rpm -Uvh erlang-solutions-1.0-1.noarch.rpm
root# yum update
root# yum install erlang
```

Установка RabbitMQ

```bash
root# wget https://www.rabbitmq.com/releases/rabbitmq-server/v3.6.10/rabbitmq-server-3.6.10-1.el7.noarch.rpm
root# rpm --import https://www.rabbitmq.com/rabbitmq-release-signing-key.asc
root# yum install rabbitmq-server-3.6.10-1.el7.noarch.rpm
```

Настраиваем Firewall, открываем порты для RabbitMQ

```bash
root# firewall-cmd --permanent --add-port=4369/tcp
root# firewall-cmd --permanent --add-port=25672/tcp
root# firewall-cmd --permanent --add-port=5671-5672/tcp
root# firewall-cmd --permanent --add-port=15672/tcp
root# firewall-cmd --permanent --add-port=61613-61614/tcp
root# firewall-cmd --permanent --add-port=8883/tcp
root# firewall-cmd --reload
```

Запускаем RabbitMQ и добавляем в автозагрузку

```bash
root# systemctl start rabbitmq-server
root# systemctl enable rabbitmq-server
root# rabbitmqctl status
```

Активация консоли управления RabbitMQ

```bash
root# rabbitmq-plugins enable rabbitmq_management
root# chown -R rabbitmq:rabbitmq /var/lib/rabbitmq/
```

После этого консоль управления RabbitMQ становиться доступным по адресу:

http://ip-address:15672/

Добавляем пользователя

```bash
root# rabbitmqctl add_user mqadmin mqadmin
root# rabbitmqctl set_user_tags mqadmin administrator
root# rabbitmqctl set_permissions -p / mqadmin ".*" ".*" ".*"
```

### Установка Веб сервера: Nginx + PHP7 FPM

Установка Nginx:

```bash
root# wget http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
root# rpm -Uvh nginx-release-centos-7-0.el7.ngx.noarch.rpm
root# yum install nginx -y
root# systemctl enable nginx
```

Установка PHP-FPM

```bash
root# yum install php70-php php70-php-cli php70-php-fpm php70-php-bcmath php70-php-devel php70-php-gd php70-php-json php70-php-mbstring php70-php-mcrypt php70-php-opcache php70-php-pecl-amqp php70-php-pecl-event -y
```

Запускаем PHP-FPM
```bash
systemctl enable php70-php-fpm
```

Отключаем SELinux:

В файле `/etc/selinux/config` ставим параметр **SELINUX=disabled**, после выполняес команду:

```bash
root# setenforce 0
```

Настраиваем Nginx и PHP-FPM для совместной работы

PHP-FPM:

Редактируем файл `/etc/opt/remi/php70/php-fpm.d/www.conf` *(не забудьте создать пользователя и группу www-data)*

```
listen = 127.0.0.1:9000

user = www-data
group = www-data
```

Создаем файл `/etc/nginx/conf.d/php-fpm.conf`

```
upstream php-fpm {
    server 127.0.0.1:9000;
    #server unix:/var/run/php-fpm/www.sock;
}
```

Создаем файл для проекта `/etc/nginx/conf.d/project.conf`

```
server {
    listen 80 default_server;
 
    root /home/project;
    index index.php index.html index.htm;
 
    server_name _;
 
    location / {
	index index.php index.html index.htm;
        try_files	$uri $uri/	=404;
    }
 
    location ~ \.php$ {
      try_files $uri =404;
      fastcgi_split_path_info ^(.+\.php)(/.+)$;
      fastcgi_pass php-fpm;
      fastcgi_index index.php;
      fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
      include fastcgi_params;
    }
 
}
```

Создаем папку с проектом `/home/project` 

*(у вас может быть любая папка для проекта, главное не забудьте изменить путь в настройках Nginx)*

В папке проекта создадит файл `index.php` для теста

```
<?php
     phpinfo();
?>
```

Перезапускам Nginx и PHP-FPM

```bash
root# systemctl restart nginx
root# systemctl restart php70-php-fpm
```

Проверяем работу Nginx: http://ip-address/
