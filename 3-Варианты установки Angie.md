# 3.1 Установка на Debian из официального репозитория

```sh
apt-get update
apt-get install -y ca-certificates curl
curl -o /etc/apt/trusted.gpg.d/angie-signing.gpg \
            https://angie.software/keys/angie-signing.gpg
echo "deb https://download.angie.software/angie/$(. /etc/os-release && echo "$ID/$VERSION_ID $VERSION_CODENAME") main" \
    | sudo tee /etc/apt/sources.list.d/angie.list > /dev/null
apt-get update
apt-get install -y angie angie-module-zstd
```

Результат:

```sh
root@debian-angie-lab-1:~# angie -v
Angie version: Angie/1.11.0
root@debian-angie-lab-1:~# angie -V
Angie version: Angie/1.11.0
nginx version: nginx/1.29.3
built on Wed, 24 Dec 2025 10:16:08 GMT
built with OpenSSL 3.5.4 30 Sep 2025
TLS SNI support enabled
configure arguments: --prefix=/etc/angie --conf-path=/etc/angie/angie.conf --error-log-path=/var/log/angie/error.log --http-log-path=/var/log/angie/access.log --lock-path=/run/angie.lock --modules-path=/usr/lib/angie/modules --pid-path=/run/angie.pid --sbin-path=/usr/sbin/angie --http-acme-client-path=/var/lib/angie/acme --http-client-body-temp-path=/var/cache/angie/client_temp --http-fastcgi-temp-path=/var/cache/angie/fastcgi_temp --http-proxy-temp-path=/var/cache/angie/proxy_temp --http-scgi-temp-path=/var/cache/angie/scgi_temp --http-uwsgi-temp-path=/var/cache/angie/uwsgi_temp --user=angie --group=angie --with-file-aio --with-http_acme_module --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-http_v3_module --with-mail --with-mail_ssl_module --with-stream --with-stream_acme_module --with-stream_mqtt_preread_module --with-stream_rdp_preread_module --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-threads --feature-cache=../angie-feature-cache --with-ld-opt='-Wl,-z,relro -Wl,-z,now'
```

# 3.2 Установка Angie PRO

```sh
# C хоста залить файлы на сервер
scp angie.license-629136-99249f.zip angie_repo-629136-6d5c6c.zip debian-angie-lab-1:/tmp/
```

```sh
# На сервере
apt install unzip
mkdir -p /etc/ssl/angie/
mkdir -p /etc/angie/
mv /tmp/angie_repo-629136-6d5c6c.zip /etc/ssl/angie/
mv /tmp/angie.license-629136-99249f.zip /etc/angie/
# распаковать с перезаписью и удалить архивы
cd /etc/ssl/angie
unzip -o angie_repo-629136-6d5c6c.zip
rm -f angie_repo-629136-6d5c6c.zip
cd /etc/angie/
unzip -o angie.license-629136-99249f.zip
rm -f angie.license-629136-99249f.zip
# Дальше по стандартной инструкции с сайта
chown -R _apt:nogroup /etc/ssl/angie/
apt-get update
apt-get install -y apt-transport-https lsb-release \
               ca-certificates curl gnupg2
curl -o /etc/apt/trusted.gpg.d/angie-signing.gpg \
            https://angie.software/keys/angie-signing.gpg
echo "deb https://download.angie.software/angie-pro/$(. /etc/os-release && echo "$ID/$VERSION_ID $VERSION_CODENAME") main" \
    | sudo tee /etc/apt/sources.list.d/angie.list > /dev/null
cat > /etc/apt/apt.conf.d/90download-angie <<'EOF'
Acquire::https::download.angie.software::Verify-Peer "true";
Acquire::https::download.angie.software::Verify-Host "true";
Acquire::https::download.angie.software::SslCert     "/etc/ssl/angie/angie-repo.crt";
Acquire::https::download.angie.software::SslKey      "/etc/ssl/angie/angie-repo.key";
EOF
chmod 0644 /etc/apt/apt.conf.d/90download-angie
ls -la /etc/apt/apt.conf.d/90download-angie
sed -n '1,120p' /etc/apt/apt.conf.d/90download-angie
apt-get update
apt-get install -y angie-pro
# Проверка лицензии
angie -t
# Не забудь изменить /etc/angie/angie.conf на ограничения worker_processes 1; worker_connections 256;
```

Результат:

```sh
root@debian-angie-lab-1:/etc/angie# angie -t
angie: 
angie: Valid license found:
angie:   - owner: CN=Angie Client License / Лицензия Angie PRO
angie:   - period: Nov 17 21:00:00 2025 GMT .. Jul 18 20:59:59 2026 GMT
angie: 
angie: Limitations:
angie:   - worker_processes: 2
angie:   - worker_connections: 256
angie: 
angie: the configuration file /etc/angie/angie.conf syntax is ok
angie: configuration file /etc/angie/angie.conf test is successful
root@debian-angie-lab-1:/etc/angie# angie -v
Angie version: Angie/1.11.0 (PRO)
root@debian-angie-lab-1:/etc/angie# angie -V
Angie version: Angie/1.11.0 (PRO)
nginx version: nginx/1.29.3
built on Wed, 24 Dec 2025 12:52:15 GMT
built with OpenSSL 3.5.4 30 Sep 2025
TLS SNI support enabled
configure arguments: --prefix=/etc/angie --build=PRO --conf-path=/etc/angie/angie.conf --error-log-path=/var/log/angie/error.log --http-log-path=/var/log/angie/access.log --lock-path=/run/angie.lock --modules-path=/usr/lib/angie/modules --pid-path=/run/angie.pid --sbin-path=/usr/sbin/angie --http-acme-client-path=/var/lib/angie/acme --http-client-body-temp-path=/var/cache/angie/client_temp --http-fastcgi-temp-path=/var/cache/angie/fastcgi_temp --http-proxy-temp-path=/var/cache/angie/proxy_temp --http-scgi-temp-path=/var/cache/angie/scgi_temp --http-uwsgi-temp-path=/var/cache/angie/uwsgi_temp --user=angie --group=angie --with-file-aio --with-http_acme_module --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-http_v3_module --with-mail --with-mail_ssl_module --with-stream --with-stream_acme_module --with-stream_mqtt_preread_module --with-stream_rdp_preread_module --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-threads --with-license --feature-cache=../angie-feature-cache --with-cc-opt='-DDEFAULT_WORKER_CONNECTIONS_LIMIT=256 -DDEFAULT_WORKER_PROCESSES_LIMIT=1' --with-ld-opt='-Wl,-z,relro -Wl,-z,now'
root@debian-angie-lab-1:/etc/angie# 
```

Заметки на будущее, в Angie PRO podman

```sh
# Скорее всего придётся так же пробросить
LICENSE_FILE="/etc/ssl/angie/license.pem"
 -v "${LICENSE_FILE}:/etc/angie/license.pem:ro" \
```

# 3.3 - Debian Docker/Podman

```sh
apt install podman crun
# добавь пользователя в группу podman, для администрирования вне root. Сейчас не нужно

# Хостовые пути
HTML_DIR="/usr/share/angie/html" # или /usr/share/angie/html
CONF_DIR="/etc/angie"
LOG_DIR="/var/log/angie"
# Пути внутри контейнера
HTML_IN="/usr/share/angie/html"
CONF_IN="/etc/angie"
LOG_IN="/var/log/angie"
HTTP_PORT=8800
mkdir -p "$HTML_DIR" "$CONF_DIR" "$LOG_DIR"

podman run --name angie \
  -v "${HTML_DIR}:${HTML_IN}:ro" \
  -v "${CONF_DIR}:${CONF_IN}:ro" \
  -v "${LOG_DIR}:${LOG_IN}" \
  -p "${HTTP_PORT}:80" -d docker.angie.software/angie:latest
```

Результат ОК.

### Podman hint

```sh
# Выключить и удалить
podman stop angie && podman rm angie
# Директория с конфигурацией в контейнере 
podman exec -it angie sh
ls /etc/angie 
```

# Дополнительные задания не выполнялись

**Дополнительное задание** - для самых любознательных (необязательно к выполнению).

1. Соберите Angie из исходных кодов с собственной OpenSSL библиотекой.
2. Пересоберите пакет на выбор (RPM, DEB) с изменением набора модулей.
3. Создайте собственный Docker-образ Angie, в котором будет установлен и подключен сторонний модуль из репозитория Angie.
