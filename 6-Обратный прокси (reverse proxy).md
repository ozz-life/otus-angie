# 6-Обратный прокси (reverse proxy)

```sh
# C хоста залить файлы на сервер
scp ./angie_configs-224190-5d5c46.conf debian-angie-lab-1:/tmp/
scp ./wordpress-224190-f2258a.conf debian-angie-lab-1:/tmp/
scp ./docker_proj-224190-7f1c31.zip debian-angie-lab-1:/tmp/
scp ./wordress_install-224190-4fb7bc.sh debian-angie-lab-1:/tmp/
```

# Установка wordpress

```sh
sudo apt update
# sudo apt install -y angie
sudo apt install -y mariadb-server
sudo apt install -y php-fpm php-mysql php-curl php-gd php-intl php-mbstring php-soap php-xml php-zip
```

# Wordpress BD

```
sudo mariadb
```

Поменяй пароль и пользователя, при необходимости.

```sql
CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE USER 'wordpress'@'localhost' IDENTIFIED BY 'password';

GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpress'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Скачать и у становить

```sh
cd /tmp
curl -LO https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz

sudo mkdir -p /var/www/wordpress
sudo rsync -a /tmp/wordpress/ /var/www/wordpress/

sudo chown -R www-data:www-data /var/www/wordpress
```

```bash
sudo cp /var/www/wordpress/wp-config-sample.php /var/www/wordpress/wp-config.php
sudo nano /var/www/wordpress/wp-config.php
```

Впиши:

`DB_NAME` = `wordpress`
`DB_USER` = `wordpress`    
`DB_PASSWORD` = password
`DB_HOST` = `localhost`

Вставь туда же соли вместо блока AUTH_KEY

```sh
curl -s https://api.wordpress.org/secret-key/1.1/salt/
```

```sh
# Генерируем сертификаты
sudo mkdir -p /etc/angie/ssl

SITE=wordpress-224190-f2258a.test
DAYS=36500

sudo openssl req -x509 -newkey rsa:2048 -nodes -days "$DAYS" \
  -keyout "/etc/angie/ssl/${SITE}.key" \
  -out   "/etc/angie/ssl/${SITE}.crt" \
  -subj  "/CN=${SITE}" \
  -addext "subjectAltName=DNS:${SITE}"

sudo chmod 600 "/etc/angie/ssl/${SITE}.key"

mkdir /var/www/${SITE}
```

```sh
# Проверить корректность сокета, выбрать корректный. В моём случае это /run/php/php8.4-fpm.sock . Потребуется, чтобы прописать в конфиг angie
ls -la /run/php/
```

```sh
tee /etc/angie/http.d/wordpress-224190-f2258a.conf >/dev/null <<'EOF'
server {
    listen 80;
    server_name wordpress-224190-f2258a.test;
    return 301 https://wordpress-224190-f2258a.test$request_uri;
}

server {
    listen 443 ssl;
    #listen 443 quic reuseport;

    http2 on;
    http3 on;

    server_name wordpress-224190-f2258a.test;

    ssl_certificate     /etc/angie/ssl/wordpress-224190-f2258a.test.crt;
    ssl_certificate_key /etc/angie/ssl/wordpress-224190-f2258a.test.key;

    access_log /var/log/angie/wordpress-224190-f2258a.test.access.log;
    error_log  /var/log/angie/wordpress-224190-f2258a.test.error.log warn;

    rewrite ^/(test.*) /test2/$1 last;

    charset utf-8;

    root /var/www/wordpress-224190-f2258a.test;
    index index.html index.php;

    location ~ /\. {
        deny all;
    }

    location ~ ^/wp-content/cache {
        deny all;
    }

    location ~* /(?:uploads|files)/.*\.php$ {
        deny all;
    }

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location /wp-content {
        add_header Cache-Control "max-age=31536000, public, no-transform, immutable";
    }

    location ~* \.(css|gif|ico|jpeg|jpg|js|png)$ {
        add_header Cache-Control "max-age=31536000, public, no-transform, immutable";
    }

    location ~ \.php$ {
        include fastcgi.conf;
        fastcgi_intercept_errors on;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
        fastcgi_index index.php;
    }
}
EOF
```

```sh
# Проверка
curl -kI https://wordpress-224190-f2258a.test/
curl -kI https://wordpress-224190-f2258a.test/readme.html
cat /var/log/angie/wordpress-224190-f2258a.test.access.log
```

Результат

```sh
# Редирект на установщик wordpress OK
HTTP/2 302
server: Angie/1.11.2
date: Sat, 31 Jan 2026 15:03:27 GMT
content-type: text/html; charset=UTF-8
location: https://wordpress-224190-f2258a.test/wp-admin/install.php
expires: Wed, 11 Jan 1984 05:00:00 GMT
cache-control: no-cache, must-revalidate, max-age=0, no-store, private
x-redirect-by: WordPress

# Статика отдаётся
HTTP/2 200
server: Angie/1.11.2
date: Sat, 31 Jan 2026 15:11:52 GMT
content-type: text/html; charset=utf-8
content-length: 7425
last-modified: Tue, 08 Jul 2025 11:05:40 GMT
etag: "686cfb84-1d01"
accept-ranges: bytes

# В логе ОК
192.168.122.1 - - [31/Jan/2026:18:10:30 +0300] "GET / HTTP/2.0" 302 0 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:147.0) Gecko/20100101 Firefox/147.0"
192.168.122.1 - - [31/Jan/2026:18:10:31 +0300] "GET /wp-admin/install.php HTTP/2.0" 200 13486 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:147.0) Gecko/20100101 Firefox/147.0"
192.168.122.39 - - [31/Jan/2026:18:11:52 +0300] "HEAD /readme.html HTTP/2.0" 200 0 "-" "curl/8.14.1"

# В браузере открылась страница установки wordpress. Устанавливать его дальше для работы с Angie нет необходимости.
```
# Дополнительно Golang

```sh
sudo apt update
sudo apt install -y golang-go
go version

sudo mkdir -p /opt/goapp
cd /opt/goapp

sudo tee /opt/goapp/main.go >/dev/null <<'GO'
package main

import (
	"encoding/json"
	"log"
	"net"
	"net/http"
)

func main() {
	mux := http.NewServeMux()

	mux.HandleFunc("/api/health", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json; charset=utf-8")
		resp := map[string]any{
			"ok":              true,
			"method":          r.Method,
			"path":            r.URL.Path,
			"host":            r.Host,
			"x_forwarded_for": r.Header.Get("X-Forwarded-For"),
			"x_real_ip":       r.Header.Get("X-Real-IP"),
			"x_forwarded_proto": r.Header.Get("X-Forwarded-Proto"),
		}
		_ = json.NewEncoder(w).Encode(resp)
	})

	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "text/html; charset=utf-8")
		w.WriteHeader(http.StatusOK)
		_, _ = w.Write([]byte(`
<h1>Go backend behind Angie</h1>
<ul>
  <li><a href="/api/health">/api/health</a></li>
  <li><a href="/static/hello.txt">/static/hello.txt</a></li>
</ul>
`))
	})

	addr := "127.0.0.1:8081"
	ln, err := net.Listen("tcp", addr)
	if err != nil {
		log.Fatalf("listen %s: %v", addr, err)
	}
	log.Printf("listening on http://%s", addr)
	log.Fatal(http.Serve(ln, mux))
}
GO

sudo go build -o /usr/local/bin/goapp /opt/goapp/main.go
sudo mkdir -p /var/www/goapp/static echo "Hello static from Angie (Go stack)" | sudo tee /var/www/goapp/static/hello.txt >/dev/null
```

```bash
# Systemd
sudo tee /etc/systemd/system/goapp.service >/dev/null <<'UNIT'
[Unit]
Description=Go demo app behind Angie
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/goapp
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
UNIT

sudo systemctl daemon-reload
sudo systemctl enable --now goapp.service
sudo systemctl status goapp.service --no-pager
```

```sh
# Сертификат
sudo mkdir -p /etc/angie/ssl
SITE=goapp.test
DAYS=36500

sudo openssl req -x509 -newkey rsa:2048 -nodes -days "$DAYS" \
  -keyout "/etc/angie/ssl/${SITE}.key" \
  -out   "/etc/angie/ssl/${SITE}.crt" \
  -subj  "/CN=${SITE}" \
  -addext "subjectAltName=DNS:${SITE}"

sudo chmod 600 "/etc/angie/ssl/${SITE}.key"
```

```sh
tee /etc/angie/http.d/goapp.test.conf >/dev/null <<'EOF'
server {
    listen 80;
    server_name goapp.test;
    return 301 https://goapp.test$request_uri;
}

server {
    listen 443 ssl;
    http2 on;

    server_name goapp.test;

    ssl_certificate     /etc/angie/ssl/goapp.test.crt;
    ssl_certificate_key /etc/angie/ssl/goapp.test.key;

    access_log /var/log/angie/goapp.test.access.log;
    error_log  /var/log/angie/goapp.test.error.log warn;

    # Статика (Angie отдаёт сам)
    location ^~ /static/ {
        alias /var/www/goapp/static/;
        try_files $uri =404;
        add_header Cache-Control "public, max-age=2592000, immutable";
        access_log off;
    }

    # Динамика -> Go backend
    location / {
        proxy_pass http://127.0.0.1:8081;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 60s;
    }
}
EOF
```

Проверки

```sh
root@debian-angie-lab-1:/opt/goapp# curl -k https://goapp.test/api/health
{"host":"goapp.test","method":"GET","ok":true,"path":"/api/health","x_forwarded_for":"192.168.122.39","x_forwarded_proto":"https","x_real_ip":"192.168.122.39"}


root@debian-angie-lab-1:/opt/goapp# curl -k https://goapp.test/static/hello.txt
Hello static from Angie (Go stack)


root@debian-angie-lab-1:/opt/goapp# curl -kI https://goapp.test/static/hello.txt
HTTP/2 200
server: Angie/1.11.2
date: Sat, 31 Jan 2026 16:09:06 GMT
content-type: text/plain
content-length: 35
last-modified: Sat, 31 Jan 2026 15:59:40 GMT
etag: "697e26ec-23"
cache-control: public, max-age=2592000, immutable
accept-ranges: bytes

root@debian-angie-lab-1:/opt/goapp# tail -n 50 /var/log/angie/goapp.test.access.log
192.168.122.1 - - [31/Jan/2026:19:05:26 +0300] "GET / HTTP/2.0" 200 155 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:147.0) Gecko/20100101 Firefox/147.0"
192.168.122.1 - - [31/Jan/2026:19:05:26 +0300] "GET /favicon.ico HTTP/2.0" 200 155 "https://goapp.test/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:147.0) Gecko/20100101 Firefox/147.0"
192.168.122.1 - - [31/Jan/2026:19:05:34 +0300] "GET /api/health HTTP/2.0" 200 158 "https://goapp.test/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:147.0) Gecko/20100101 Firefox/147.0"
192.168.122.1 - - [31/Jan/2026:19:05:34 +0300] "GET /favicon.ico HTTP/2.0" 200 155 "https://goapp.test/api/health" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:147.0) Gecko/20100101 Firefox/147.0"
192.168.122.1 - - [31/Jan/2026:19:06:30 +0300] "GET / HTTP/2.0" 502 157 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:147.0) Gecko/20100101 Firefox/147.0"
192.168.122.1 - - [31/Jan/2026:19:06:43 +0300] "GET / HTTP/2.0" 200 155 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:147.0) Gecko/20100101 Firefox/147.0"
192.168.122.39 - - [31/Jan/2026:19:08:24 +0300] "GET /api/health HTTP/2.0" 200 160 "-" "curl/8.14.1"
root@debian-angie-lab-1:/opt/goapp#
```

Итог. Приложение Golang запущено. Angie отдаёт и его и статику.
