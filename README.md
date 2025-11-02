# Установка **Seafile**-сервера

*Seafile - личное облачное хранилище.*

---

Предыдущая статья [Установка VPN сервера](https://github.com/margazun/vpn-xray-server/)
о том, как арендовать в Нидерландах **vds**, установить **Xray** сервер,
настроить клиентов на Android, Windows, OpenWRT (домашний роутер).

На этом же сервере будем устанавливать файловое хранилище [Seafile](https://www.seafile.com/en/home/).

Seafile будет работать в docker-контейнере.

* Операционная система Ubuntu 24.04.2 LTS
* На сервере уже установлены:
    - nginx-full
    - certbot
    - xray сервер

### Что необходимо

* А-запись у регистратора, которая указывает на ip vds-сервера.
  Что-то вроде такого, для домена example.com A seafile -> 275.134.95.8.
  Создаются записи в разделе управления зоной.

* Немного времени

---

### Содержание

1. [Получение сертификатов для сайта](#ssl-cert)
1. [Установка Docker](#docker-install)
1. [Установка Seafile](#seafile-setup)
    - [Подготовка compose-файла](#seafile-docker-setup)
    - [Первый запуск](#seafile-docker-first-start)
    - [Настройка](#seafile-settings)
    - [Создание unit-файла](#seafile-unit)
1. [Настройка xray](#xray-settings)
1. [Настройка Nginx](#nginx-settings)
    - [Изменение nginx.conf](#nginx-settings-conf)
    - [Создание конфигурации для seafile](#nginx-conf-seafile)
    - [Активация конфигурации](#nginx-conf-seafile-activate)
    - [Настройка unit-файла](#nginx-unit)
1. [Создание скрипта для перезапуска Nginx и Seafile](#script-restart-nginx-sefile)
1. [Перезапуск Nginx и Seafile после перевыпуска сертификатов](#restart-nginx-sefile-after-cert)
1. [Активация юнитов](#unit-activate)
1. [Перезапуск Xray, Nginx и Seafile](#seafile-install)
1. [Перезапуск сервера и финальная проверка](#server-reboot)

---

<a name="ssl-cert">

## Получение сертификатов для сайта

* Получим для нашего домена **example.com** ssl-ключи, заменив <seafile.example.com> своим именем

```
sudo certbot certonly --standalone --preferred-challenges http -d seafile.example.com -d www.seafile.example.com
```


<a name="docker-install">

## Установка Docker

Обновляем систему

```
sudo apt update && sudo apt upgrade -y
```

Если не установлены, добавляем пакеты

```
sudo apt install apt-transport-https curl -y
```

Импортируем ключ Docker репозитория

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

Добавляем официальный репозиторий Docker в систему

```
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Обновляем список пакетов

```
sudo apt update
```

Устанавливаем Docker

```
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

Проверяем, что после установки Docker уже работает

```
sudo systemctl is-active docker
```

Должны получить ответ **active**

Проверим работу Docker

```
sudo docker run hello-world
```

В случае успешной установки должны получить примерно такой вывод:
```
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
e6590344b1a5: Pull complete
Digest: sha256:dd01f97f252193ae3210da231b1dca0cffab4aadb3566692d6730bf93f123a48
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

Сделаем так, чтобы Docker можно было запускать без **sudo**

```
sudo groupadd docker && sudo usermod -aG docker $USER && newgrp docker
```

<a name="seafile-setup">

## Установка Seafile

<a name="seafile-docker-setup">

### Подготовка docker-файла

* Создадим необходимые директории и перейдем в **/opt/seafile**

```
sudo mkdir /opt/seafile && sudo mkdir /opt/seafile-data && sudo mkdir /opt/seafile-data/mysql && cd /opt/seafile
```

* Создадим файл ``docker-compose.yml``

```
sudo touch docker-compose.yml
```

В любом редакторе внесем в него, заменив <seafile.example.com> своим именем и задав свои значения:

- MYSQL_ROOT_PASSWORD
- MYSQL_PASSWORD
- DB_ROOT_PASSWD
- DB_PASSWD
- SEAFILE_ADMIN_EMAIL
- SEAFILE_ADMIN_PASSWORD
- SEAFILE_SERVER_HOSTNAME


```
services:
  db:
    image: mariadb:10.5
    container_name: seafile-mysql
    environment:
      MYSQL_ROOT_PASSWORD: DbAdminPassword
      MYSQL_USER: seafile
      MYSQL_PASSWORD: DbUserPassword
      MYSQL_DATABASE: seafile_db
    volumes:
      - /opt/seafile-data/mysql:/var/lib/mysql
    restart: always

  memcached:
    image: memcached:1.6-alpine
    container_name: seafile-memcached
    restart: always

  seafile:
    image: seafileltd/seafile-mc:latest
    container_name: seafile
    ports:
      - "8081:80"
    environment:
      DB_HOST: db
      DB_ROOT_PASSWD: DbAdminPassword
      DB_USER: seafile
      DB_PASSWD: DbUserPassword
      TIME_ZONE: Asia/Novosibirsk
      SEAFILE_SERVER_LETSENCRYPT: "false"
      SEAFILE_SERVER_HOSTNAME: seafile.example.com
      SEAFILE_ADMIN_EMAIL: admin@example.com
      SEAFILE_ADMIN_PASSWORD: AdminPassword
      SEAFILE_MEMCACHED_HOST: memcached
      SEAFILE_MEMCACHED_PORT: 11211
    volumes:
      - /opt/seafile-data:/shared
    depends_on:
      - db
      - memcached
    restart: always

volumes:
  seafile-data:
    driver: local
  seafile-mysql-data:
    driver: local
```

<a name="seafile-docker-first-start">

### Первый запуск **Seafile**

Запустим наши контейнеры

```
docker compose up -d
```

Дождемся успешного запуска контейнеров и после этого остановим их

```
docker compose down
```

<a name="seafile-settings">

### Настройка **Seafile**

После первого запуска в директории **/opt** должна появиться директория
**seafile-data**.

* Изменим файл ``/opt/seafile-data/nginx/conf/seafile.nginx.conf`` в любом редакторе, заменив <seafile.example.com> своим именем

```
# -*- mode: nginx -*-
# Auto generated at 05/19/2025 10:28:25
server {
listen 80;
server_name seafile.example.com;

    client_max_body_size 10m;

    location / {
        proxy_pass http://127.0.0.1:8000/;
        proxy_read_timeout 310s;
        proxy_set_header Host $http_host;
        proxy_set_header Forwarded "for=$remote_addr;proto=$scheme";
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Connection "";
        proxy_http_version 1.1;

        client_max_body_size 0;
        access_log      /var/log/nginx/seahub.access.log seafileformat;
        error_log       /var/log/nginx/seahub.error.log;
    }

    location /seafhttp {
        rewrite ^/seafhttp(.*)$ $1 break;
        proxy_pass http://127.0.0.1:8082;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        client_max_body_size 0;
        proxy_connect_timeout  36000s;
        proxy_read_timeout  36000s;
        proxy_request_buffering off;
        access_log      /var/log/nginx/seafhttp.access.log seafileformat;
        error_log       /var/log/nginx/seafhttp.error.log;
    }

    location /notification/ping {
        proxy_pass http://127.0.0.1:8083/ping;
        access_log      /var/log/nginx/notification.access.log seafileformat;
        error_log       /var/log/nginx/notification.error.log;
    }

    location /notification {
        proxy_pass http://127.0.0.1:8083/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        access_log      /var/log/nginx/notification.access.log seafileformat;
        error_log       /var/log/nginx/notification.error.log;
    }

    location /seafdav {
        proxy_pass         http://127.0.0.1:8080;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Host $server_name;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_read_timeout  1200s;
        client_max_body_size 0;

        access_log      /var/log/nginx/seafdav.access.log seafileformat;
        error_log       /var/log/nginx/seafdav.error.log;
    }

    location /media {
        root /opt/seafile/seafile-server-latest/seahub;
    }

}
```


* Изменим файл ``/opt/seafile-data/seafile/conf/seahub_settings.py`` в любом редакторе, заменив <seafile.example.com> своим именем

```
...
SERVICE_URL = "https://seafile.example.com"
...
FILE_SERVER_ROOT = "https://seafile.example.com/seafhttp"
CSRF_TRUSTED_ORIGINS = ['https://seafile.example.com', 'https://www.seafile.example.com']
ENABLE_HTTPS = True
MAX_UPLOAD_SIZE = 0
...
```
<a name="seafile-unit">

### Создание unit-файл

* Создадим юнит для запуска контейнеров

```
sudo touch /etc/systemd/system/seafile-docker.service
```

* Изменим его в любом редакторе

```
[Unit]
Description=Seafile via Docker Compose
Requires=docker.service
After=network-online.target docker.service docker.socket
Wants=network-online.target
Before=nginx.service

[Service]
Type=oneshot
RemainAfterExit=true
WorkingDirectory=/opt/seafile
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
ExecStartPost=/bin/bash -c 'for i in {1..30}; do \
  if docker ps --format "{{.Names}}" | grep -q seafile; then \
    echo "Seafile container is up."; exit 0; \
  fi; \
  echo "Waiting for Seafile container to start ($i/30)..."; sleep 2; \
done; \
echo "Warning: Seafile container not detected after 60s."'

Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

<a name="xray-settings">

## Настройка Xray

Настроим **xray** так, чтобы он для протокола **vless** слушал порт
8443, вместо 443.

Отредактируем файл ``/opt/xray/config.json`` изменим порт на 8443

```
..
"inbounds": [
  ...
  {
    "port": 8443,
    "protocol": "vless",
    ...
  },
  ...
]
```

<a name="nginx-settings">

## Настройка Nginx

Настроим **nginx** так, чтобы трафик, который предназначается для **seafile**
перенаправлялся в запущенный docker-контейнер.

<a name="nginx-settings-conf">

* Внесем изменения в файл ``/etc/nginx/nginx.conf``,
  заменив <seafile.example.com> своим именем

```
user www-data;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;

events {
	worker_connections 768;
}

http {
	sendfile on;
	tcp_nopush on;
	types_hash_max_size 2048;
	include /etc/nginx/mime.types;
	default_type application/octet-stream;
	ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
	ssl_prefer_server_ciphers on;
	access_log /var/log/nginx/access.log;
	gzip on;
	include /etc/nginx/conf.d/*.conf;
	include /etc/nginx/sites-enabled/*;
 
  client_body_timeout 12;
  client_header_timeout 12;
  keepalive_timeout 15;
  send_timeout 10;
}

stream {
  upstream xray_backend {
    server 127.0.0.1:8443;
  }
  
  upstream domain_backend {
    server 127.0.0.1:444;
  }
  
  map $ssl_preread_server_name $backend {
    seafile.example.com              domain_backend;
    www.seafile.example.com          domain_backend;
    default                          xray_backend;
  }
  
  server {
    listen 443;
    proxy_pass $backend;
    ssl_preread on;
  }
}
```

<a name="nginx-conf-seafile">

* Создадим файл конфигурационный файл **seafile.conf**

```
sudo touch /etc/nginx/sites-available/seafile.conf
```

В любом редакторе внесем в него, заменив <seafile.example.com> своим именем

```
server {
    listen 444 ssl http2;
    server_name seafile.example.com www.seafile.example.com;

    ssl_certificate /etc/letsencrypt/live/seafile.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/seafile.example.com/privkey.pem;

    ssl_session_timeout 5m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
    ssl_prefer_server_ciphers on;

    proxy_set_header X-Forwarded-For $remote_addr;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
    client_max_body_size 0;

    location / {
        proxy_pass http://127.0.0.1:8081;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port 443;
    }
}
```

<a name="nginx-conf-seafile-activate">

* Активируем конфигурацию

```
sudo ln -s /etc/nginx/sites-available/seafile.conf /etc/nginx/sites-enabled/seafile.conf
```

<a name="nginx-unit">

* Настроим **nginx** так, чтобы он запускался только после запуска **seafile**

Создадим директорию ``/etc/systemd/system/nginx.service.d``

```
sudo mkdir /etc/systemd/system/nginx.service.d/
```

Создадим файл ```/etc/systemd/system/nginx.service.d/override.conf```

```
sudo touch /etc/systemd/system/nginx.service.d/override.conf
```

Добавим в него в любом редакторе

```
[Unit]
After=network-online.target docker.service docker.socket seafile-docker.service
Wants=network-online.target docker.service docker.socket seafile-docker.service

[Service]
ExecStartPre=/bin/bash -c 'for i in {1..60}; do \
  if systemctl is-active --quiet docker && docker ps --format "{{.Names}}" | grep -q seafile; then \
    echo "Seafile is up, starting nginx..."; exit 0; \
  fi; \
  echo "[$i/60] Waiting for Docker and Seafile..."; sleep 2; \
done; \
echo "Warning: Seafile not detected, starting nginx anyway"; exit 0'

Restart=on-failure
RestartSec=5s
ExecStartPost=/bin/bash -c ' \
  (sleep 180; \
   if systemctl is-active --quiet docker && docker ps --format "{{.Names}}" | grep -q seafile; then \
     echo "Seafile detected later — restarting nginx"; \
     systemctl restart nginx; \
   fi) & \
'
```

<a name="script-restart-nginx-sefile">

## Создание скрипта для перезапуска **Nginx** и **Seafile**

* Создадим скрипт для перезапуска наших контейнеров и **nginx**

```
sudo touch /usr/local/bin/restart-seafile-nginx.sh
```

Изменим его в любом редакторе

```
#!/bin/bash

LOGFILE="/var/log/seafile/deploy-hook.log"
echo "$(date '+%Y-%m-%d %H:%M:%S') Deploy hook started" >> "$LOGFILE"

# Перезапускаем контейнер Seafile
docker restart seafile >> "$LOGFILE" 2>&1

# Перезапускаем nginx (если он снаружи контейнера)
systemctl restart nginx >> "$LOGFILE" 2>&1

echo "$(date '+%Y-%m-%d %H:%M:%S') Seafile container and nginx restarted" >> "$LOGFILE"
```

* Сделаем его исполняемым

```
sudo chmod +x /usr/local/bin/restart-seafile-nginx.sh
```

<a name="restart-nginx-sefile-after-cert">

## Перезапуск **Nginx** и **Seafile** после перевыпуска сертификатов

Изменим файл ``/etc/letsencrypt/renewal/seafile.example.com.conf``
в любом редакторе. Добавим строку

```
renew_hook = /usr/local/bin/restart-seafile-nginx.sh
```

<a name="unit-activate">

## Активация unit-файлов

```
sudo systemctl daemon-reload
```

```
sudo systemctl enable seafile-docker
```

```
sudo systemctl enable nginx
```

<a name="services-restart">

## Перезапуск **Xray**, **Nginx** и **Seafile**

```
sudo systemctl restart xray
```

```
sudo systemctl restart seafile-docker
```

```
sudo systemctl restart nginx
```

<a name="server-reboot">

## Перезапуск сервера и финальная проверка

* Перезапустим сервер

```
sudo reboot now
```

* После перезагрузки сервера и повторного подключения проверим статус сервисов

```
sudo systemctl status seafile-docker
```

```
systemctl status nginx
```

Оба сервиса должны иметь статус **active (running)**

Все готово! Можно заходить браузером на страницу https://seafile.example.com и
настраивать свое личное облачное хранилище))

**ВАЖНО** - заходить можно только по **https**-протоколу, по http доступа не будет.