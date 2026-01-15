# 5-Angie как веб-сервер

```sh
# C хоста залить файлы на сервер
scp static_site-252831-96c1e2.zip debian-angie-lab-1:/tmp/
```

```sh
# На сервере
mkdir -p /var/www/static_site-252831-96c1e2
unzip -q /tmp/static_site-252831-96c1e2.zip -d /var/www/static_site-252831-96c1e2
touch /etc/angie/http.d/static_site-252831-96c1e2.test.conf

tee /etc/angie/http.d/static_site-252831-96c1e2.test.conf >/dev/null <<'EOF'
# 80 -> 443
server {
    listen 80;
    server_name static_site-252831-96c1e2.test;
    return 301 https://$host$request_uri;
}

# HTTPS
server {
    listen 443 ssl;
    http2 on;

    server_name static_site-252831-96c1e2.test;

    ssl_certificate     /etc/angie/ssl/static_site-252831-96c1e2.test.crt;
    ssl_certificate_key /etc/angie/ssl/static_site-252831-96c1e2.test.key;

    access_log /var/log/angie/static_site-252831-96c1e2.test.access.log;
    error_log  /var/log/angie/static_site-252831-96c1e2.test.error.log;

    root /var/www/static_site-252831-96c1e2;
    index index.html;

    add_header X-Site static-site always;

    location / { try_files $uri $uri/ =404; }
}
EOF

# Генерируем сертификат
DOMAIN="static_site-252831-96c1e2.test"
DAYS=36500

sudo openssl req -x509 -newkey rsa:2048 -nodes -days "$DAYS" \
  -keyout "/etc/angie/ssl/${DOMAIN}.key" \
  -out   "/etc/angie/ssl/${DOMAIN}.crt" \
  -subj  "/CN=${DOMAIN}" \
  -addext "subjectAltName=DNS:${DOMAIN}"

sudo chmod 600 "/etc/angie/ssl/${DOMAIN}.key"

# проверка и перезагрузка
sudo angie -t && sudo systemctl reload angie
```

```sh
# Проверки
IP="127.0.0.1"
curl -sS -I  --resolve "${DOMAIN}:80:${IP}"  "http://${DOMAIN}/"  | sed -n '1,10p'
curl -sS -k -I --resolve "${DOMAIN}:443:${IP}" "https://${DOMAIN}/" | sed -n '1,20p'
curl -sS -k    --resolve "${DOMAIN}:443:${IP}" "https://${DOMAIN}/" | head
```

```sh
# Результат проверок - ОК
HTTP/1.1 301 Moved Permanently
Server: Angie/1.11.1
Date: Wed, 14 Jan 2026 23:46:31 GMT
Content-Type: text/html
Content-Length: 169
Connection: keep-alive
Location: https://static_site-252831-96c1e2.test/

HTTP/2 200
server: Angie/1.11.1
date: Wed, 14 Jan 2026 23:46:31 GMT
content-type: text/html
content-length: 14532
last-modified: Fri, 06 Sep 2024 16:10:23 GMT
etag: "66db296f-38c4"
x-site: static-site
accept-ranges: bytes

<!DOCTYPE HTML>
<!--
        Dimension by HTML5 UP
        html5up.net | @ajlkn
        Free for personal and commercial use under the CCA 3.0 license (html5up.net/license)
-->
<html>
        <head>
                <title>Welcome</title>
                <meta charset="utf-8" />
```

# Для каждой директории создайте location со своими настройками. Отдельно создайте location c регулярным выражением для отдачи картинок jpg, jpeg, png, gif.

```sh
tee /etc/angie/http.d/static_site-252831-96c1e2.test.conf >/dev/null <<'EOF'
# 80 -> 443
server {
    listen 80;
    server_name static_site-252831-96c1e2.test;
    return 301 https://$host$request_uri;
}

# HTTPS
server {
    listen 443 ssl;
    http2 on;

    server_name static_site-252831-96c1e2.test;

    ssl_certificate     /etc/angie/ssl/static_site-252831-96c1e2.test.crt;
    ssl_certificate_key /etc/angie/ssl/static_site-252831-96c1e2.test.key;

    access_log /var/log/angie/static_site-252831-96c1e2.test.access.log;
    error_log  /var/log/angie/static_site-252831-96c1e2.test.error.log;

    root /var/www/static_site-252831-96c1e2;
    index index.html;

    add_header X-Site static_site-252831-96c1e2.test always;

    # Главная и HTML — без кеша
    location = / {
        try_files /index.html =404;
        add_header Cache-Control "no-cache" always;
    }

    location ~* \.html$ {
        try_files $uri =404;
        add_header Cache-Control "no-cache" always;
    }

    # /assets/ — общий
    location /assets/ {
        try_files $uri =404;
        access_log off;
        expires 7d;
        add_header Cache-Control "public, max-age=604800" always;
    }

    location /assets/css/ {
        try_files $uri =404;
        access_log off;
        expires 30d;
        add_header Cache-Control "public, max-age=2592000, immutable" always;
    }

    location /assets/js/ {
        try_files $uri =404;
        access_log off;
        expires 30d;
        add_header Cache-Control "public, max-age=2592000, immutable" always;
    }

    location /assets/fonts/ {
        try_files $uri =404;
        access_log off;
        expires 365d;
        add_header Cache-Control "public, max-age=31536000, immutable" always;
    }

    # исходники — не раздаём
    location /assets/sass/ {
        deny all;
    }

    location /images/ {
        try_files $uri =404;
        access_log off;
        expires 30d;
        add_header Cache-Control "public, max-age=2592000, immutable" always;
    }

    location /error/ {
        try_files $uri =404;
        add_header Cache-Control "no-store" always;
    }

    # regex для картинок (jpg/jpeg/png/gif)
    location ~* \.(?:jpg|jpeg|png|gif)$ {
        try_files $uri =404;
        access_log off;
        expires 30d;
    }
}
EOF

sudo angie -t && sudo systemctl reload angie
```

```sh
# Проверки выражений
DOMAIN="static_site-252831-96c1e2.test"
IP="127.0.0.1"
curl -sS -k -I --resolve "${DOMAIN}:443:${IP}" "https://${DOMAIN}/" | sed -n '1,25p'
curl -sS -k -I --resolve "${DOMAIN}:443:${IP}" "https://${DOMAIN}/images/bg.jpg" | sed -n '1,25p'
```

Результат проверок

```sh
HTTP/2 200
server: Angie/1.11.1
date: Thu, 15 Jan 2026 00:16:22 GMT
content-type: text/html
content-length: 14532
last-modified: Fri, 06 Sep 2024 16:10:23 GMT
etag: "66db296f-38c4"
cache-control: no-cache
accept-ranges: bytes

HTTP/2 200
server: Angie/1.11.1
date: Thu, 15 Jan 2026 00:16:22 GMT
content-type: image/jpeg
content-length: 37864
last-modified: Fri, 06 Sep 2024 15:38:16 GMT
etag: "66db21e8-93e8"
expires: Sat, 14 Feb 2026 00:16:22 GMT
cache-control: max-age=2592000
x-site: static_site-252831-96c1e2.test
accept-ranges: bytes
```

# Задействуйте переменные, определённые с map для работы с location.

```sh
tee /etc/angie/http.d/00-geo-hacks.conf >/dev/null <<'EOF'
# Литералы/хелперы
geo $dollar {
    default "$";
}
EOF
```

У меня на текущий момент ноль представления как использовать map, чтобы это было полезно и удобно. Потому представляю синтетический и надуманный пример, как доказательство работы.

```sh
tee /etc/angie/http.d/00-maps.conf >/dev/null <<'EOF'
# Пример map: классификация запросов по URI
map $uri $bucket {
    default  "page";
    ~^/assets/ "assets";
    ~^/images/ "images";
    ~^/error/  "error";
}
EOF

tee /etc/angie/http.d/00-maps-cache.conf >/dev/null <<'EOF'
map $uri $cc {
    default          "no-cache";

    ~^/error/        "no-store";

    ~^/assets/fonts/ "public, max-age=31536000, immutable";
    ~^/assets/css/   "public, max-age=2592000, immutable";
    ~^/assets/js/    "public, max-age=2592000, immutable";
    ~^/assets/       "public, max-age=604800";

    ~^/images/       "public, max-age=2592000, immutable";

    ~*\.jpg$         "public, max-age=2592000, immutable";
    ~*\.jpeg$        "public, max-age=2592000, immutable";
    ~*\.png$         "public, max-age=2592000, immutable";
    ~*\.gif$         "public, max-age=2592000, immutable";

    ~*\.html$        "no-cache";
}
EOF

tee /etc/angie/http.d/static_site-252831-96c1e2.test.conf >/dev/null <<'EOF'
# 80 -> 443
server {
    listen 80;
    server_name static_site-252831-96c1e2.test;
    return 301 https://$host$request_uri;
}

# HTTPS
server {
    listen 443 ssl;
    http2 on;

    server_name static_site-252831-96c1e2.test;

    ssl_certificate     /etc/angie/ssl/static_site-252831-96c1e2.test.crt;
    ssl_certificate_key /etc/angie/ssl/static_site-252831-96c1e2.test.key;

    access_log /var/log/angie/static_site-252831-96c1e2.test.access.log;
    error_log  /var/log/angie/static_site-252831-96c1e2.test.error.log;

    root /var/www/static_site-252831-96c1e2;
    index index.html;

    add_header X-Site static_site-252831-96c1e2.test always;
    add_header X-Bucket $bucket always;
    add_header Cache-Control $cc always;

    # Главная и HTML - без кеша
    location = / {
        try_files /index.html =404;
    }

    location ~* \.html$ {
        try_files $uri =404;
    }

    # /assets/ - общий
    location /assets/ {
        try_files $uri =404;
        access_log off;
    }

    location /assets/css/ {
        try_files $uri =404;
        access_log off;
    }

    location /assets/js/ {
        try_files $uri =404;
        access_log off;
    }

    location /assets/fonts/ {
        try_files $uri =404;
        access_log off;
    }

    # исходники - не раздаём
    location /assets/sass/ {
        deny all;
    }

    location /images/ {
        try_files $uri =404;
        access_log off;
    }

    location /error/ {
        try_files $uri =404;
    }

    # regex для картинок (jpg/jpeg/png/gif)
    location ~* \.(?:jpg|jpeg|png|gif)$ {
        try_files $uri =404;
        access_log off;
    }
}
EOF

angie -t && systemctl reload angie
```

```sh
# Проверка
DOMAIN="static_site-252831-96c1e2.test"; IP="127.0.0.1"
for path in / /images/bg.jpg /assets/css/ /error/index.html; do
  echo "== $path =="
  curl -sS -k -I --resolve "${DOMAIN}:443:${IP}" "https://${DOMAIN}${path}" \
  | awk -F': ' 'BEGIN{IGNORECASE=1} $1=="cache-control" || $1=="x-bucket"{print}'
done
```

Результат вывода:

```sh
== / ==
x-bucket: page
cache-control: no-cache
== /images/bg.jpg ==
x-bucket: images
cache-control: public, max-age=2592000, immutable
== /assets/css/ ==
x-bucket: assets
cache-control: public, max-age=2592000, immutable
== /error/index.html ==
x-bucket: error
cache-control: no-store
```

# Настройте два вида перенаправлений (301/302 и внутренние).

```sh
tee /etc/angie/http.d/static_site-252831-96c1e2.test.conf >/dev/null <<'EOF'
# 80 -> 443
server {
    listen 80;
    server_name static_site-252831-96c1e2.test;
    return 301 https://$host$request_uri;
}

# HTTPS
server {
    listen 443 ssl;
    http2 on;

    server_name static_site-252831-96c1e2.test;

    ssl_certificate     /etc/angie/ssl/static_site-252831-96c1e2.test.crt;
    ssl_certificate_key /etc/angie/ssl/static_site-252831-96c1e2.test.key;

    access_log /var/log/angie/static_site-252831-96c1e2.test.access.log;
    error_log  /var/log/angie/static_site-252831-96c1e2.test.error.log;

    root /var/www/static_site-252831-96c1e2;
    index index.html;

    add_header X-Site static_site-252831-96c1e2.test always;
    add_header X-Bucket $bucket always;
    add_header Cache-Control $cc always;

    # Главная и HTML - без кеша
    location = / {
        try_files /index.html =404;
    }

    location ~* \.html$ {
        try_files $uri =404;
    }

    # /assets/ - общий
    location /assets/ {
        try_files $uri =404;
        access_log off;
    }

    location /assets/css/ {
        try_files $uri =404;
        access_log off;
    }

    location /assets/js/ {
        try_files $uri =404;
        access_log off;
    }

    location /assets/fonts/ {
        try_files $uri =404;
        access_log off;
    }

    # исходники - не раздаём
    location /assets/sass/ {
        deny all;
    }

    location /images/ {
        try_files $uri =404;
        access_log off;
    }

    location /error/ {
        try_files $uri =404;
    }

    # regex для картинок (jpg/jpeg/png/gif)
    location ~* \.(?:jpg|jpeg|png|gif)$ {
        try_files $uri =404;
        access_log off;
    }

    # 301 Moved Permanently
    location = /go301 {
        return 301 /;
    }

    # 302 Found
    # /go302 -> /error/index.html
    location = /go302 {
        return 302 /error/index.html;
    }

    # Internal Redirect - внутренняя переадресация внутри сервера
    # URL остаётся /internal, но отдадим файл /error/index.html
    # (rewrite + last делает внутренний редирект)
    location = /internal {
        rewrite ^ /error/index.html last;
    }
}
EOF
```

```sh
# Проверки
DOMAIN="static_site-252831-96c1e2.test"; IP="127.0.0.1"

# 301: должен быть Location: / и статус 301
curl -sS -k -I --resolve "${DOMAIN}:443:${IP}" "https://${DOMAIN}/go301" | sed -n '1,10p'

# 302: должен быть Location: /error/index.html и статус 302
curl -sS -k -I --resolve "${DOMAIN}:443:${IP}" "https://${DOMAIN}/go302" | sed -n '1,10p'

# internal: статус 200, но содержимое/etag/last-modified будет как у /error/index.html,
# при этом URL не меняется (curl этого не "видит", но Location не будет)
curl -sS -k -I --resolve "${DOMAIN}:443:${IP}" "https://${DOMAIN}/internal" | sed -n '1,20p'
curl -sS -k -I --resolve "${DOMAIN}:443:${IP}" "https://${DOMAIN}/error/index.html" | sed -n '1,20p'
```

Результат вывода
```sh
HTTP/2 301
server: Angie/1.11.1
date: Thu, 15 Jan 2026 01:56:11 GMT
content-type: text/html
content-length: 169
location: https://static_site-252831-96c1e2.test/
x-site: static_site-252831-96c1e2.test
x-bucket: page
cache-control: no-cache

HTTP/2 302
server: Angie/1.11.1
date: Thu, 15 Jan 2026 01:56:11 GMT
content-type: text/html
content-length: 145
location: https://static_site-252831-96c1e2.test/error/index.html
x-site: static_site-252831-96c1e2.test
x-bucket: page
cache-control: no-cache

HTTP/2 200
server: Angie/1.11.1
date: Thu, 15 Jan 2026 01:56:11 GMT
content-type: text/html
content-length: 1398
last-modified: Fri, 06 Sep 2024 15:38:16 GMT
etag: "66db21e8-576"
x-site: static_site-252831-96c1e2.test
x-bucket: error
cache-control: no-store
accept-ranges: bytes

HTTP/2 200
server: Angie/1.11.1
date: Thu, 15 Jan 2026 01:56:11 GMT
content-type: text/html
content-length: 1398
last-modified: Fri, 06 Sep 2024 15:38:16 GMT
etag: "66db21e8-576"
x-site: static_site-252831-96c1e2.test
x-bucket: error
cache-control: no-store
accept-ranges: bytes
```

# Напишите директиву rewrite для преобразования адресов (301 редирект). Все запросы, которые не заканчиваются на слэш, нужно перенаправить на такой же адрес, но с окончанием на слэш (кроме запросов к статическим файлам).

Добавить перед первым location в HTTPS

```sh
    # 301: trailing slash для "pretty URLs" всем директориям, кроме статики
    location ~ ^(.*/)?[^./]+$ {
        return 301 $uri/$is_args$args;
    }
```

```sh
angie -t && systemctl reload angie
# Проверка
DOMAIN="static_site-252831-96c1e2.test"; IP="127.0.0.1"
curl -sS -k -I --resolve "${DOMAIN}:443:${IP}" "https://${DOMAIN}/about" | sed -n '1,10p'
curl -sS -k -I --resolve "${DOMAIN}:443:${IP}" "https://${DOMAIN}/images/bg.jpg" | sed -n '1,10p'
```

Результат

```sh
HTTP/2 301
server: Angie/1.11.1
date: Thu, 15 Jan 2026 02:08:08 GMT
content-type: text/html
content-length: 169
location: https://static_site-252831-96c1e2.test/about/
x-site: static_site-252831-96c1e2.test
x-bucket: page
cache-control: no-cache

HTTP/2 200
server: Angie/1.11.1
date: Thu, 15 Jan 2026 02:08:08 GMT
content-type: image/jpeg
content-length: 37864
last-modified: Fri, 06 Sep 2024 15:38:16 GMT
etag: "66db21e8-93e8"
x-site: static_site-252831-96c1e2.test
x-bucket: images
cache-control: public, max-age=2592000, immutable
```
# Создайте переменную, которая будет зависеть от заголовка запроса Accept. Если в заголовке есть avif, то она должна иметь значение “.avif”, если есть webp, то “.webp”.

```sh
tee -a /etc/angie/http.d/00-maps.conf >/dev/null <<'EOF'

# Выбор расширения по Accept (avif приоритетнее webp)
map $http_accept $img_ext {
    default "";
    ~*avif  ".avif";
    ~*webp  ".webp";
}
EOF
```

```sh
# Для проверки добавь в HTTPS server
add_header X-Img-Ext $img_ext always;
```

```sh
angie -t && systemctl reload angie
# Проверки
DOMAIN="static_site-252831-96c1e2.test"; IP="127.0.0.1"
curl -sS -k -I --resolve "${DOMAIN}:443:${IP}" -H 'Accept: image/avif,image/webp,*/*' "https://${DOMAIN}/" | grep -i x-img-ext
curl -sS -k -I --resolve "${DOMAIN}:443:${IP}" -H 'Accept: image/webp,*/*' "https://${DOMAIN}/" | grep -i x-img-ext
```

Результаты проверки

```sh
x-img-ext: .avif
x-img-ext: .webp
```
