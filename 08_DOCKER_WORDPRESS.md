# Создание контейнера wordpress

Для общего понимания сделаем небольшое ревью задачи, разбив её на подзадачу.

Сначала выпишем список того, что нам нужно для контейнера. Это:

- php с плагинами для работы wordpress
- php-fpm для связи с nginx
- сам wordpress. Просто так, чтобы было.

Для настройки нам потребуется выполнить следующие действия:

- установить через Dockerfile php со всеми плагинами
- установить через Dockerfile все необходимые программы
- скачать и положить в /var/www сам вордпресс, так же через Dockerfile
- подсунуть в контейнер правильный конфиг fastcgi (www.conf)
- запустить в контейнере fastcgi через сокет php-fpm
- добавить все необходимые разделы в docker-compose
- установить порядок запуска контейнеров
- добавить раздел с wordpress контейнеру с nginx
- тестить чтобы всё работало

## Шаг 1. Настройка Dockerfile

Итак, мы переходим к настройке wordpress.  Действуем всё так же: берём за основу последний alpine и накатываем на него нужный нам софт.

Но накатываем по-умному, указав актуальную на сегодня версию php. На момент создания гайда (2022) это php 8, если с 2022 года прошло много времени, нужно зайти на [официальный сайт php](https://www.php.net/ "официальный сайт php") и посмотреть, не вышла ли более новая версия.

Поэтому версию PHP я укажу в переменной - аргументе командной строки. Задаёт переменную инструкция ARG.

Сначала перечислим базовые компоненты: это php, на котором и работает наш wordpress, php-fpm для взаимодействия с nginx и php-mysqli для взаимодействия с mariadb:

```
FROM alpine:latest
ARG PHP_VERSION=8
RUN apk update && apk upgrade && apk add --no-cache \
    php${PHP_VERSION} \
    php${PHP_VERSION}-fpm \
    php${PHP_VERSION}-mysqli
```

Теперь обратимся к [документации wordpress](https://make.wordpress.org/hosting/handbook/server-environment/ "официальная документация wordpress") и посмотрим,что ещё нам понадобится.

Для полноценной работы нашего wordpress-а не поскупимся и загрузим все обязательные модули, опустив модули кэширования и дополнительные. Так же загрузим пакет wget, нужный для скачивания самого wordpress, и пакет unzip для разархивирования архива со скачанным wordpress:

```
FROM alpine:latest
ARG PHP_VERSION=8
RUN apk update && apk upgrade && apk add --no-cache \
    php${PHP_VERSION} \
    php${PHP_VERSION}-fpm \
    php${PHP_VERSION}-mysqli \
    php${PHP_VERSION}-json \
    php${PHP_VERSION}-curl \
    php${PHP_VERSION}-dom \
    php${PHP_VERSION}-exif \
    php${PHP_VERSION}-fileinfo \
    php${PHP_VERSION}-mbstring \
    php${PHP_VERSION}-openssl \
    php${PHP_VERSION}-xml \
    php${PHP_VERSION}-zip \
    wget \
    unzip
```

Далее исправим нужный нам конфиг - конфиг www.conf, чтобы наш fastcgi слушал все соединения по порту 9000 (путь /etc/php8/php-fpm.d/ зависит от установленной версии php!):

```
FROM alpine:latest
ARG PHP_VERSION=8
RUN apk update && apk upgrade && apk add --no-cache \
    php${PHP_VERSION} \
    php${PHP_VERSION}-fpm \
    php${PHP_VERSION}-mysqli \
    php${PHP_VERSION}-json \
    php${PHP_VERSION}-curl \
    php${PHP_VERSION}-dom \
    php${PHP_VERSION}-exif \
    php${PHP_VERSION}-fileinfo \
    php${PHP_VERSION}-mbstring \
    php${PHP_VERSION}-openssl \
    php${PHP_VERSION}-xml \
    php${PHP_VERSION}-zip \
    wget \
	  unzip \
    sed -i "s|listen = 127.0.0.1:9000|listen = 9000|g" \
    /etc/php8/php-fpm.d/www.conf \
    sed -i "s|;listen.owner = nobody|listen.owner = nobody|g" \
    /etc/php8/php-fpm.d/www.conf \
    sed -i "s|;listen.group = nobody|listen.group = nobody|g" \
    /etc/php8/php-fpm.d/www.conf \
    && rm -f /var/cache/apk/*
```

Принцип тот же, что и в предыдущем гайде. Меняем три строчки конфига sed-ом.

Последней командой мы очищаем кэш установленных модулей.

Далее нам надо скачать wordpress и разархивировать его по пути /var/www. Для удобства сделаем этот путь рабочим командой WORKDIR:

```
FROM alpine:latest
ARG PHP_VERSION=8
RUN apk update && apk upgrade && apk add --no-cache \
    php${PHP_VERSION} \
    php${PHP_VERSION}-fpm \
    php${PHP_VERSION}-mysqli \
    php${PHP_VERSION}-json \
    php${PHP_VERSION}-curl \
    php${PHP_VERSION}-dom \
    php${PHP_VERSION}-exif \
    php${PHP_VERSION}-fileinfo \
    php${PHP_VERSION}-mbstring \
    php${PHP_VERSION}-openssl \
    php${PHP_VERSION}-xml \
    php${PHP_VERSION}-zip \
    wget \
    unzip && \
    sed -i "s|listen = 127.0.0.1:9000|listen = 9000|g" \
      /etc/php8/php-fpm.d/www.conf && \
    sed -i "s|;listen.owner = nobody|listen.owner = nobody|g" \
      /etc/php8/php-fpm.d/www.conf && \
    sed -i "s|;listen.group = nobody|listen.group = nobody|g" \
      /etc/php8/php-fpm.d/www.conf && \
    rm -f /var/cache/apk/*
WORKDIR /var/www
RUN wget https://wordpress.org/latest.zip && \
    unzip latest.zip && \
    cp -rf wordpress/* . && \
    rm -rf wordpress latest.zip
CMD ["/usr/sbin/php-fpm8", "-F"]
```

В последней инструкции RUN мы загрузили wget-ом последнюю версию wordpress, разархивировали её и удалили все исходные файлы.

CMD же запускает наш установленный php-fpm (внимание: версия должна соответствовать установленной!)

## Шаг 2. Конфигурация docker-compose

Теперь добавим в наш docker-compose секцию с wordpress.

Для начала пропишем следующее:

```
  wordpress:
    build:
      context: .
      dockerfile: requirements/wordpress/Dockerfile
#    depends_on:
#      - mariadb
    restart: unless-stopped
```

Директива depends_on означает, что wordpress зависит от mariadb и не запустится, пока контейнер с базой данных не соберётся. Самым "шустрым" из наших контейнеров будет nginx - ввиду малого веса он соберётся и запустится первым. А вот база и CMS собираются примерно равное время, и чтобы не случилась, что wordpress начинает устанавливаться на ещё не развёрнутую базу потребуется указать эту зависимость.

Но пока мы тестируем сам wordpress, mariadb ещё не развёрнута, потому оставим, но просто закомментируем эту зависимость.

Далее укажем директорию, в которой развернётся наш wordpress, и имя контейнера:

```
  wordpress:
    build:
      context: .
      dockerfile: requirements/wordpress/Dockerfile
#    depends_on:
#      - mariadb
    restart: unless-stopped
    container_name: wordpress
```

## Шаг 3. Создание разделов

У nginx и wordpress должен быть общий раздел для обмена данными. Можно примонтировать туда и туда одну и ту же папку, но для удобства создадим раздел, указав путь к его папке:

```
volumes:
  wordpress:
    name: wp-volume
    driver_opts:
      o: bind
      type: none
      device: /home/${USER}/wordpress
```

Так же создадим папку wordpress в домашнем каталоге:

``mkdir ~/wordpress``

Теперь добавим этот раздел ко всем контейнерам, которые от него зависят. Таким образом вся наша конфигурация будет выглядеть так:

```
version: '3'

services:
  nginx:
    build:
      context: .
      dockerfile: requirements/nginx/Dockerfile
    container_name: nginx
    ports:
      - "443:443"
    volumes:
      - ./requirements/nginx/conf/:/etc/nginx/conf.d/
      - ./requirements/nginx/tools:/etc/nginx/ssl/
      - wp-volume:/var/www/
    restart: unless-stopped

  wordpress:
    build:
      context: .
      dockerfile: requirements/wordpress/Dockerfile
#    depends_on:
#      - mariadb
    restart: unless-stopped
    volumes:
      - wp-volume:/var/www/
    container_name: wordpress

volumes:
  wp-volume:
    driver_opts:
      o: bind
      type: none
      device: /home/${USER}/wordpress
```

## Шаг 4. Создание файла конфигурации worpdress

Нам нужно будет скопировать в папку wordpress-а конфигурационный файл, который соединит нас с контейнером базы данных.

Создадим этот файл в папке tools:

``nano srcs/requirements/wordpress/conf/wp-config-create.sh``

Вставим в него следующее содержимое:

```
#!bin/sh

if [ ! -f "/var/www/wp-config.php" ]; then

        cat << EOF > /var/www/wp-config.php
<?php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wpuser' );
define( 'DB_PASSWORD', 'wppass' );
define( 'DB_HOST', 'mariadb' );
define( 'DB_CHARSET', 'utf8' );
define( 'DB_COLLATE', '' );
$table_prefix = 'wp_';
define( 'WP_DEBUG', false );
if ( ! defined( 'ABSPATH' ) ) {
	define( 'ABSPATH', __DIR__ . '/' );
}
require_once ABSPATH . 'wp-settings.php';
EOF

fi
```

## Шаг 5. Изменение конфигурации nginx

Нам необходимо изменить конфигурацию nginx-а чтобы тот обрабатывал только php-файлы. Для этого удалим из конфига все index.html.

Для полного счастья нам осталось раскомментировать блок nginx-а, обрабатывающий php, чтобы наш nginx.conf выглядел следующим образом:

```
server {
    listen      80;
    listen      443 ssl;
    server_name  <your_nickname>.42.fr www.<your_nickname>.42.fr;
    root    /var/www/;
    index index.php;
#   if ($scheme = 'http') {
#       return 301 https://<your_nickname>.42.fr$request_uri;
#   }
    ssl_certificate     /etc/nginx/ssl/<your_nickname>.42.fr.crt;
    ssl_certificate_key /etc/nginx/ssl/<your_nickname>.42.fr.key;
    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    keepalive_timeout 70;
    location / {
        try_files $uri /index.php?$args;
        add_header Last-Modified $date_gmt;
        add_header Cache-Control 'no-store, no-cache';
        if_modified_since off;
        expires off;
        etag off;
    }
    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass wordpress:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```

Вот теперь вроде бы всё, наша конфигурация готова к запуску.

``docker exec -it wordpress ps aux | grep 'php'``

Вывод должен быть следующим:

```
    1 root      0:00 {php-fpm8} php-fpm: master process (/etc/php8/php-fpm.conf
    9 nobody    0:00 {php-fpm8} php-fpm: pool www
   10 nobody    0:00 {php-fpm8} php-fpm: pool www
```

``docker exec -it wordpress php -v``

```
PHP 8.0.22 (cli) (built: Aug  5 2022 23:54:32) ( NTS )
Copyright (c) The PHP Group
Zend Engine v4.0.22, Copyright (c) Zend Technologies
```

``docker exec -it wordpress php -m``

```
[PHP Modules]
Core
curl
date
dom
exif
fileinfo
filter
hash
json
libxml
mbstring
mysqli
mysqlnd
openssl
pcre
readline
Reflection
SPL
standard
xml
zip
zlib

[Zend Modules]
```

...и вуаля! (как любят говорить наши французские друзья)

![настройка wordpress](media/work_wp.png)

И вот, когда вы успешно запустили вордпресс, где-то в Париже возрадовался один разработичк...

![настройка mariadb](media/docker_mariadb/step_6.jpeg)

## Шаг 6. Изменение Makefile

Так же не забываем копировать наш Makefile. Его придётся немного изменить, потому как docker-compose у нас лежит по пути srcs:

```
name = inception
all:
	@printf "Launch configuration ${name}...\n"
	@docker-compose -f ./srcs/docker-compose.yml up -d

down:
	@printf "Stopping configuration ${name}...\n"
	@docker-compose -f ./srcs/docker-compose.yml down

re:
	@printf "Rebuild configuration ${name}...\n"
	@docker-compose -f ./srcs/docker-compose.yml up -d --build

clean: down
	@printf "Cleaning configuration ${name}...\n"
	@docker system prune --a

fclean:
	@printf "Total clean of all configurations docker\n"
	@docker stop $$(docker ps -qa)
	@docker system prune --all --force --volumes
	@docker network prune --force
	@docker volume prune --force

.PHONY	: all down re clean fclean
```

Перед сохранением в облако советую сделать make fclean.
