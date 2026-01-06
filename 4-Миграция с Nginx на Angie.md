# Подготовка nginx. 

```sh
# C хоста залить файлы на сервер
scp nginx_conf.tar-252831-0242cd.gz debian-angie-lab-1:/tmp/
```

```sh
# Уже на хосте
cd /tmp
# Распаковку я делал копированием через mc в /etc/nginx
nginx -t && systemctl reload nginx
# Получаем ошибку
2025/12/31 16:02:25 [emerg] 1394#1394: unknown directive "brotli_static" in /etc/nginx/nginx.conf:59
nginx: configuration file /etc/nginx/nginx.conf test failed
#
apt update
apt install -y libnginx-mod-http-brotli-filter libnginx-mod-http-brotli-static
ls -la /etc/nginx/modules-enabled/ | grep -i brotli || true
nginx -t && systemctl reload nginx
```

Результат

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

В вебе 502 Bad Gateway. Поведение изменилось. Но мне на этом моменте хотелось бы видеть любую заглушку под сайт, чтобы видеть, что всё сделано корректно.

В качестве доказательства, что перенос архива сделан корректно

```sh
root@debian-angie-lab-1:/tmp# nginx -T
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
# configuration file /etc/nginx/nginx.conf:
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
	worker_connections 768;
	# multi_accept on;
}

http {

	##
	# Basic Settings
	##

	sendfile on;
	tcp_nopush on;
	types_hash_max_size 2048;
	# server_tokens off;

	# server_names_hash_bucket_size 64;
	# server_name_in_redirect off;

	include /etc/nginx/mime.types;
	default_type application/octet-stream;

	proxy_cache_valid 1m;
	proxy_cache_key $scheme$host$request_uri;
	proxy_cache_path /var/www/cache levels=1:2 keys_zone=one:10m inactive=48h max_size=800m;

	##
	# SSL Settings
	##

	ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
	ssl_prefer_server_ciphers on;

	##
	# Logging Settings
	##

	access_log /var/log/nginx/access.log;
	error_log /var/log/nginx/error.log;

	##
	# Gzip Settings
	##

	gzip on;

	gzip_vary on;
	gzip_proxied any;
	gzip_comp_level 6;
	gzip_buffers 16 8k;
	gzip_http_version 1.1;
	gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    brotli_static		on;
    brotli				on;
    brotli_comp_level	5;
    brotli_types		text/plain text/css text/xml application/javascript application/json image/x-icon image/svg+xml;

    #zstd 			on;
    #zstd_min_length 	256;
    #zstd_comp_level 	5;
    #zstd_static 		on;
    #zstd_types			text/plain text/css text/xml application/javascript application/json image/x-icon image/svg+xml;

    map $http_accept $webp_suffix {
        "~*webp"  ".webp";
    }

    map $http_accept $avif_suffix {
        "~*avif"  ".avif";
        "~*webp"  ".webp";
    }

    map $msie $cache_control {
      default "max-age=31536000, public, no-transform, immutable";
        "1"     "max-age=31536000, private, no-transform, immutable";
    }

    map $msie $vary_header {
	default "Accept";
	"1"     "";
    }


	##
	# Virtual Host Configs
	##

	include /etc/nginx/conf.d/*.conf;
	include /etc/nginx/sites-enabled/*;
}


#mail {
#	# See sample authentication script at:
#	# http://wiki.nginx.org/ImapAuthenticateWithApachePhpScript
#
#	# auth_http localhost/auth.php;
#	# pop3_capabilities "TOP" "USER";
#	# imap_capabilities "IMAP4rev1" "UIDPLUS";
#
#	server {
#		listen     localhost:110;
#		protocol   pop3;
#		proxy      on;
#	}
#
#	server {
#		listen     localhost:143;
#		protocol   imap;
#		proxy      on;
#	}
#}

# configuration file /etc/nginx/modules-enabled/50-mod-http-brotli-filter.conf:
load_module modules/ngx_http_brotli_filter_module.so;

# configuration file /etc/nginx/modules-enabled/50-mod-http-brotli-static.conf:
load_module modules/ngx_http_brotli_static_module.so;

# configuration file /etc/nginx/mime.types:

types {
    text/html                             html htm shtml;
    text/css                              css;
    text/xml                              xml;
    image/gif                             gif;
    image/jpeg                            jpeg jpg;
    application/javascript                js;
    application/atom+xml                  atom;
    application/rss+xml                   rss;

    text/mathml                           mml;
    text/plain                            txt;
    text/vnd.sun.j2me.app-descriptor      jad;
    text/vnd.wap.wml                      wml;
    text/x-component                      htc;

    image/png                             png;
    image/tiff                            tif tiff;
    image/vnd.wap.wbmp                    wbmp;
    image/x-icon                          ico;
    image/x-jng                           jng;
    image/x-ms-bmp                        bmp;
    image/svg+xml                         svg svgz;
    image/webp                            webp;

    application/font-woff                 woff;
    application/java-archive              jar war ear;
    application/json                      json;
    application/mac-binhex40              hqx;
    application/msword                    doc;
    application/pdf                       pdf;
    application/postscript                ps eps ai;
    application/rtf                       rtf;
    application/vnd.apple.mpegurl         m3u8;
    application/vnd.ms-excel              xls;
    application/vnd.ms-fontobject         eot;
    application/vnd.ms-powerpoint         ppt;
    application/vnd.wap.wmlc              wmlc;
    application/vnd.google-earth.kml+xml  kml;
    application/vnd.google-earth.kmz      kmz;
    application/x-7z-compressed           7z;
    application/x-cocoa                   cco;
    application/x-java-archive-diff       jardiff;
    application/x-java-jnlp-file          jnlp;
    application/x-makeself                run;
    application/x-perl                    pl pm;
    application/x-pilot                   prc pdb;
    application/x-rar-compressed          rar;
    application/x-redhat-package-manager  rpm;
    application/x-sea                     sea;
    application/x-shockwave-flash         swf;
    application/x-stuffit                 sit;
    application/x-tcl                     tcl tk;
    application/x-x509-ca-cert            der pem crt;
    application/x-xpinstall               xpi;
    application/xhtml+xml                 xhtml;
    application/xspf+xml                  xspf;
    application/zip                       zip;

    application/octet-stream              bin exe dll;
    application/octet-stream              deb;
    application/octet-stream              dmg;
    application/octet-stream              iso img;
    application/octet-stream              msi msp msm;

    application/vnd.openxmlformats-officedocument.wordprocessingml.document    docx;
    application/vnd.openxmlformats-officedocument.spreadsheetml.sheet          xlsx;
    application/vnd.openxmlformats-officedocument.presentationml.presentation  pptx;

    audio/midi                            mid midi kar;
    audio/mpeg                            mp3;
    audio/ogg                             ogg;
    audio/x-m4a                           m4a;
    audio/x-realaudio                     ra;

    video/3gpp                            3gpp 3gp;
    video/mp2t                            ts;
    video/mp4                             mp4;
    video/mpeg                            mpeg mpg;
    video/quicktime                       mov;
    video/webm                            webm;
    video/x-flv                           flv;
    video/x-m4v                           m4v;
    video/x-mng                           mng;
    video/x-ms-asf                        asx asf;
    video/x-ms-wmv                        wmv;
    video/x-msvideo                       avi;
}

# configuration file /etc/nginx/sites-enabled/default:
upstream backend {
    #hash $request_uri;
    server 127.0.0.1:9000 weight=2;
    server 127.0.0.1:9001 weight=5;
    server 127.0.0.1:9002 weight=8;
    server 127.0.0.1:9003 weight=4 fail_timeout=5s;
}

server {
	listen 80 default_server;
	listen [::]:80 default_server;

	root /var/www/html;

	index index.html index.htm index.nginx-debian.html;

	server_name _;
	
	access_log /var/log/nginx/default.log;

	location / {
		#proxy_cache one;
	    proxy_cache_valid 200 1h;
	    proxy_cache_lock on;
	    proxy_cache_use_stale updating error timeout invalid_header http_500 http_502 http_504;
	    proxy_cache_background_update on;
		#proxy_pass	http://127.0.0.1:8008;
		proxy_pass	http://backend;
		#try_files $uri $uri/ =404;
	}

	location /stat {
		include         uwsgi_params;
		uwsgi_pass      127.0.0.1:81;
	}

  # Static files location
  location ~* \.(jpg|jpeg|gif|png)$ {
        include /etc/nginx/static-avif.conf;
  }

  # Static files location
  location ~* \.(ttf|eot|svg|woff|woff2|css|js|json|ico|zip|tgz|gz|rar|bz2|doc|docx|xlsx|pptx|xls|exe|pdf|ppt|txt|tar|mid|midi|wav|bmp|rtf|avi|swf|flv|mp3|mp4|fla)$ {
		expires max;
        include /etc/nginx/static.conf;
  }


	location ~ \.php$ {
		include snippets/fastcgi-php.conf;
		# With php-fpm (or other unix sockets):
		fastcgi_pass unix:/run/php/php8.1-fpm.sock;
	#	# With php-cgi (or other tcp sockets):
	#	fastcgi_pass 127.0.0.1:9000;
	}
}

# configuration file /etc/nginx/uwsgi_params:

uwsgi_param  QUERY_STRING       $query_string;
uwsgi_param  REQUEST_METHOD     $request_method;
uwsgi_param  CONTENT_TYPE       $content_type;
uwsgi_param  CONTENT_LENGTH     $content_length;

uwsgi_param  REQUEST_URI        $request_uri;
uwsgi_param  PATH_INFO          $document_uri;
uwsgi_param  DOCUMENT_ROOT      $document_root;
uwsgi_param  SERVER_PROTOCOL    $server_protocol;
uwsgi_param  REQUEST_SCHEME     $scheme;
uwsgi_param  HTTPS              $https if_not_empty;

uwsgi_param  REMOTE_ADDR        $remote_addr;
uwsgi_param  REMOTE_PORT        $remote_port;
uwsgi_param  SERVER_PORT        $server_port;
uwsgi_param  SERVER_NAME        $server_name;

# configuration file /etc/nginx/static-avif.conf:
add_header Vary $vary_header;
add_header Cache-Control $cache_control;
try_files $uri$avif_suffix $uri$webp_suffix $uri =404;

# configuration file /etc/nginx/static.conf:
add_header Cache-Control "max-age=31536000, public, no-transform, immutable";

# configuration file /etc/nginx/snippets/fastcgi-php.conf:
# regex to split $uri to $fastcgi_script_name and $fastcgi_path
fastcgi_split_path_info ^(.+?\.php)(/.*)$;

# Check that the PHP script exists before passing it
try_files $fastcgi_script_name =404;

# Bypass the fact that try_files resets $fastcgi_path_info
# see: http://trac.nginx.org/nginx/ticket/321
set $path_info $fastcgi_path_info;
fastcgi_param PATH_INFO $path_info;

fastcgi_index index.php;
include fastcgi.conf;

# configuration file /etc/nginx/fastcgi.conf:

fastcgi_param  SCRIPT_FILENAME    $document_root$fastcgi_script_name;
fastcgi_param  QUERY_STRING       $query_string;
fastcgi_param  REQUEST_METHOD     $request_method;
fastcgi_param  CONTENT_TYPE       $content_type;
fastcgi_param  CONTENT_LENGTH     $content_length;

fastcgi_param  SCRIPT_NAME        $fastcgi_script_name;
fastcgi_param  REQUEST_URI        $request_uri;
fastcgi_param  DOCUMENT_URI       $document_uri;
fastcgi_param  DOCUMENT_ROOT      $document_root;
fastcgi_param  SERVER_PROTOCOL    $server_protocol;
fastcgi_param  REQUEST_SCHEME     $scheme;
fastcgi_param  HTTPS              $https if_not_empty;

fastcgi_param  GATEWAY_INTERFACE  CGI/1.1;
fastcgi_param  SERVER_SOFTWARE    nginx/$nginx_version;

fastcgi_param  REMOTE_ADDR        $remote_addr;
fastcgi_param  REMOTE_PORT        $remote_port;
fastcgi_param  REMOTE_USER        $remote_user;
fastcgi_param  SERVER_ADDR        $server_addr;
fastcgi_param  SERVER_PORT        $server_port;
fastcgi_param  SERVER_NAME        $server_name;

# PHP only, required if PHP was built with --enable-force-cgi-redirect
fastcgi_param  REDIRECT_STATUS    200;
```

# fail

Я несколько дней размышлял как это всё же сдать, с учётом того что заготовка не полная и проверить корректность переноса на этом тестовом конфиге нельзя, кроме как прохождения angie -t с настроенным модулем brotli.

Мне не нравится предложенный конфиг, я бы не хотел переносить подобный на работе, он запутанный. Хочется видеть более понятный. Потому было решено сделать чистую настройку с нуля, со стандартными путями.

Дополнительно: автоматизируйте преобразование конфигов с помощью bash или другого скриптового языка. WAT? А это точно выполнимая задача? Мне кажется, если бы она была выполнима, то этот скрипт был бы частью функционала angie. А на сайте не была бы опубликована странная пошаговая инструкция. :(

# new config

Этого конфига мне почти хватило, чтобы выполнить рабочую задачу на работе по переносе наших двух wiki сайтов с nginx на angie. Потому прикладываю его, как выполнение этого ДЗ. В текущих формулировках оно для меня не выполнимо.

Конфиг специально создан максимально упрощённым и понятным, но доведённым до рабочего состояния. 

```sh
root@debian-angie-lab-1:~# angie -T
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
# configuration file /etc/angie/angie.conf:
user  angie;
worker_processes  1;
worker_rlimit_nofile 65536;

error_log  /var/log/angie/error.log notice;
pid        /run/angie.pid;

events {
    worker_connections  256;
}


http {
    include       /etc/angie/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    log_format extended '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" rt="$request_time" '
                        '"$http_user_agent" "$http_x_forwarded_for" '
                        'h="$host" sn="$server_name" ru="$request_uri" u="$uri" '
                        'ucs="$upstream_cache_status" ua="$upstream_addr" us="$upstream_status" '
                        'uct="$upstream_connect_time" urt="$upstream_response_time"';

    access_log  /var/log/angie/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/angie/http.d/*.conf;
}

#stream {
#    include /etc/angie/stream.d/*.conf;
#}

# configuration file /etc/angie/mime.types:

types {
    text/html                                        html htm shtml;
    text/css                                         css;
    text/xml                                         xml;
    image/gif                                        gif;
    image/jpeg                                       jpeg jpg;
    application/javascript                           js;
    application/atom+xml                             atom;
    application/rss+xml                              rss;

    text/mathml                                      mml;
    text/plain                                       txt;
    text/vnd.sun.j2me.app-descriptor                 jad;
    text/vnd.wap.wml                                 wml;
    text/x-component                                 htc;

    image/avif                                       avif;
    image/bmp                                        bmp;
    image/png                                        png;
    image/svg+xml                                    svg svgz;
    image/tiff                                       tif tiff;
    image/vnd.wap.wbmp                               wbmp;
    image/webp                                       webp;
    image/x-icon                                     ico;
    image/x-jng                                      jng;

    font/woff                                        woff;
    font/woff2                                       woff2;

    application/java-archive                         jar war ear;
    application/json                                 json;
    application/mac-binhex40                         hqx;
    application/msword                               doc;
    application/pdf                                  pdf;
    application/postscript                           ps eps ai;
    application/rtf                                  rtf;
    application/vnd.apple.mpegurl                    m3u8;
    application/vnd.debian.binary-package            deb udeb;
    application/vnd.google-earth.kml+xml             kml;
    application/vnd.google-earth.kmz                 kmz;
    application/vnd.ms-excel                         xls;
    application/vnd.ms-fontobject                    eot;
    application/vnd.ms-powerpoint                    ppt;
    application/vnd.oasis.opendocument.graphics      odg;
    application/vnd.oasis.opendocument.presentation  odp;
    application/vnd.oasis.opendocument.spreadsheet   ods;
    application/vnd.oasis.opendocument.text          odt;
    application/vnd.openxmlformats-officedocument.presentationml.presentation
                                                     pptx;
    application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
                                                     xlsx;
    application/vnd.openxmlformats-officedocument.wordprocessingml.document
                                                     docx;
    application/vnd.rar                              rar;
    application/vnd.wap.wmlc                         wmlc;
    application/wasm                                 wasm;
    application/x-7z-compressed                      7z;
    application/x-cocoa                              cco;
    application/x-java-archive-diff                  jardiff;
    application/x-java-jnlp-file                     jnlp;
    application/x-makeself                           run;
    application/x-perl                               pl pm;
    application/x-pilot                              prc pdb;
    application/x-redhat-package-manager             rpm;
    application/x-sea                                sea;
    application/x-shockwave-flash                    swf;
    application/x-stuffit                            sit;
    application/x-tcl                                tcl tk;
    application/x-x509-ca-cert                       der pem crt;
    application/x-xpinstall                          xpi;
    application/xhtml+xml                            xhtml;
    application/xspf+xml                             xspf;
    application/zip                                  zip;

    application/octet-stream                         bin exe dll;
    application/octet-stream                         dmg;
    application/octet-stream                         iso img;
    application/octet-stream                         msi msp msm;

    audio/midi                                       mid midi kar;
    audio/mpeg                                       mp3;
    audio/ogg                                        ogg;
    audio/x-m4a                                      m4a;
    audio/x-realaudio                                ra;

    video/3gpp                                       3gpp 3gp;
    video/mp2t                                       ts;
    video/mp4                                        mp4;
    video/mpeg                                       mpeg mpg;
    video/quicktime                                  mov;
    video/webm                                       webm;
    video/x-flv                                      flv;
    video/x-m4v                                      m4v;
    video/x-mng                                      mng;
    video/x-ms-asf                                   asx asf;
    video/x-ms-wmv                                   wmv;
    video/x-msvideo                                  avi;
}

# configuration file /etc/angie/http.d/00-catchall.conf:
# Catch-all для HTTP
server {
    listen 80 default_server;
    server_name _;
    return 444;
}

# Catch-all для HTTPS
server {
    listen 443 ssl default_server;
    http2 on;
    server_name _;

    ssl_certificate     /etc/angie/ssl/default.local.crt;
    ssl_certificate_key /etc/angie/ssl/default.local.key;

    return 421;
}

# configuration file /etc/angie/http.d/site1.test.conf:
# 80 -> 443
server {
    listen 80;
    server_name site1.test;
    return 301 https://$host$request_uri;
}

# HTTPS
server {
    listen 443 ssl;
    http2 on;

    server_name site1.test;

    ssl_certificate     /etc/angie/ssl/site1.test.crt;
    ssl_certificate_key /etc/angie/ssl/site1.test.key;

    access_log /var/log/angie/site1.test.access.log;
    error_log  /var/log/angie/site1.test.error.log;

    root /var/www/site1.test;
    index index.html;

    add_header X-Site site1 always;

    location / { try_files $uri $uri/ =404; }
}

# configuration file /etc/angie/http.d/site2.test.conf:
server {
    listen 80;
    server_name site2.test;

    root /var/www/site2.test;
    index index.html;

    add_header X-Site site2 always;

    location / { try_files $uri $uri/ =404; }
}

root@debian-angie-lab-1:~# 
```

# Рекомендации по улучшению ДЗ

- Необходима более очевидная заглушка сайта. 