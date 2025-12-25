Tham khảo source image bitnami: https://raw.githubusercontent.com/bitnami/containers/main/bitnami/wordpress/docker-compose.yml

* Dùng Compose v3.4+ `docker compose version`
* WordPress + MariaDB chạy luôn bằng 1 lệnh `docker-compose up -d`
* Không dùng Apache, nhẹ hơn → dùng **PHP-FPM + Nginx**
* Dữ liệu lưu **trực tiếp trong thư mục hiện tại**, dễ backup

---

### **Cấu trúc thư mục**

Giả sử bạn tạo một thư mục dự án, ví dụ `mywordpress`:

```
wordpress/
├─ docker-compose-wordpress.yml
├─ wordpress_data/     # dữ liệu WordPress
├─ mariadb_data/       # dữ liệu MariaDB
├─ nginx/
│   └─ default.conf    # config Nginx
│   └─ docker-compose-nginx.yml    # compose Nginx + Modsecurity
├─ caddy/
│   ├─ Dockerfile          # build xcaddy với plugin
│   ├─ Caddyfile           # cấu hình Caddy
│   └─ waf/
│       ├─ config.conf
│       ├─ coreruleset/
│       └─ crs-setup-custom.conf
```

---
## 1️⃣ CẤU HÌNH WORDPRESS AND MARIADB

### **File Compose WordPress + MariaDB**
File compose WordPress + MariaDB `docker-compose-wordpress.yml`
Nội dung:
```yaml
version: '3.8'

services:
  mariadb:
    image: mariadb:10.11
    container_name: mariadb
    environment:
      MYSQL_ROOT_PASSWORD: root@123
      MYSQL_DATABASE: wordpressdb
      MYSQL_USER: wp_user
      MYSQL_PASSWORD: admin@123
    volumes:
      - ./mariadb_data:/var/lib/mysql
    networks:
      - wp-network

  wordpress:
    image: wordpress:php8.2-fpm-alpine
    container_name: wordpress
    environment:
      WORDPRESS_DB_HOST: mariadb:3306
      WORDPRESS_DB_NAME: wordpressdb
      WORDPRESS_DB_USER: wp_user
      WORDPRESS_DB_PASSWORD: admin@123
    volumes:
      - ./wordpress_data:/var/www/html
    networks:
      - wp-network
    depends_on:
      - mariadb
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

networks:
  wp-network:
    name: wp-network
```
> Lưu ý: Mình đặt tên network cố định wp-network để Nginx có thể connect từ Compose khác.

Chạy:
```bash
docker compose -f docker-compose-wordpress.yml up -d
```

---

## 2️⃣ CÂU HÌNH NGINX + MODSECURITY

### **Compose Nginx riêng (có ModSecurity)**
File docker-compose-nginx.yml:
```yaml
version: '3.8'

services:
  nginx:
    image: owasp/modsecurity:nginx-alpine
    container_name: nginx
    ports:
      - "8080:80"
    volumes:
      - ./wordpress_data:/var/www/html  # chia sẻ dữ liệu với WordPress
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
      - ./nginx/modsecurity.conf:/etc/modsecurity/modsecurity.conf
      - ./nginx/rules:/etc/modsecurity/rules
    networks:
      - wp-network
    depends_on:
      - wordpress
      condition: service_healthy

networks:
  wp-network:
    external: true
```

Giải thích:
- external: true → dùng network wp-network đã tạo từ Compose WordPress.
- Container Nginx sẽ kết nối đến service wordpress bằng hostname wordpress:9000.
- Bạn có thể thêm ModSecurity rules, configs, log… thoải mái.

### File Nginx config `nginx/default.conf`
```nginx
server {
    listen 80;
    server_name localhost;

    root /var/www/html;
    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass wordpress:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

### ModSecurity cơ bản
File: `nginx/modsecurity.conf`

```text
SecRuleEngine On
SecRequestBodyAccess On
SecResponseBodyAccess Off
SecRule REQUEST_HEADERS:User-Agent "BadBot" "id:1000001,deny,log,msg:'Blocked BadBot'"
```

Thư mục `nginx/rules/` có thể để trống hoặc thêm rule OWASP CRS sau này.

---

### 4️⃣ Cách chạy

1. Ở thư mục `mywordpress`, chạy compose WordPress + MariaDB::

```bash
docker compose -f docker-compose.yml up -d
```
2. Chạy Nginx (có ModSecurity) sau:

```bash
docker compose -f docker-compose-nginx.yml up -d
```
* Trình duyệt truy cập: `http://localhost:8080`
* Dữ liệu WordPress nằm trong `wordpress_data/`
* Dữ liệu MariaDB nằm trong `mariadb_data/`
* Không dùng Apache, nhẹ, boot nhanh


## 3️⃣ CẤU HÌNH CADDY + PLUGIN + CORROZA MODSECURITY CRS

### Dockerfile 
Dockerfile Caddy tùy chỉnh
File: `caddy/Dockerfile`

```Dockerfile
# Build xcaddy với plugin
FROM caddy:2-builder AS builder

RUN xcaddy build \
    --with github.com/greenpau/caddy-security@latest \
    --with github.com/mholt/caddy-dynamicdns@latest \
    --with github.com/mholt/caddy-ratelimit@latest \
    --with github.com/corazawaf/coraza-caddy/v2

# Copy binary ra final image
FROM caddy:2-alpine
COPY --from=builder /usr/bin/caddy /usr/bin/caddy

# Copy waf config và caddyfile
COPY waf/ /etc/caddy/waf/
COPY Caddyfile /etc/caddy/Caddyfile

# Set working dir
WORKDIR /etc/caddy
```

### Caddyfile
Caddyfile ví dụ dùng WAF
```caddyfile
#{
#    order coraza_waf first
#}
keycloak.vinahost.cloud {
#       coraza_waf {
#                include /etc/caddy/waf/config.conf
#                #include /etc/caddy/waf/coreruleset/crs-setup.conf.example
#                include /etc/caddy/waf/crs-setup-custom.conf
#               include /etc/caddy/waf/coreruleset/rules/*.conf
#
#
#               directives `
#                       SecAction "id:1,pass,log"
#                       SecRule ARGS "(?i).*" "id:900001,phase:2,log,pass,msg:'Log GET query string'"
#                       SecRule REQUEST_URI "/testp1" "id:2, deny, log, phase:1"
#                               SecRule REQUEST_URI "/testp3" "id:4, deny, log, phase:3"
#                       # Block XML-RPC brute-force
#                       SecRule REQUEST_URI "/xmlrpc.php" "id:900002,phase:2,deny,log,msg:'Block XML-RPC'"
#                       # Block common WP exploits (RCE, LFI)
#                       SecRule ARGS "@rx (eval|base64_decode|system|exec|shell_exec)" "id:900003,phase:2,deny,log,msg:'Potential WP RCE'"
#                       # Block sensitive paths
#                       SecRule REQUEST_URI "@rx /(wp-config\.php|\.htaccess|wp-admin/install\.php)" "id:900004,phase:1,deny,log,msg:'Block sensitive WP paths'"
#                       SecRule REQUEST_URI "@beginsWith /wp-json/" "id:100001,phase:1,pass,nolog,ctl:ruleEngine=Off"
#
#
#
#               `
#       }
        root * /var/www/html
        encode zstd gzip
        php_fastcgi wordpress:9000
        file_server
        tls tuyennp.tts@vinahost.com {
                protocols tls1.2 tls1.3
        }
#       header {
#               Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
#               X-Content-Type-Options "nosniff"
#               X-Frame-Options "DENY"
#               Referrer-Policy "no-referrer-when-downgrade"
#               X-XSS-Protection "1; mode=block"
#               Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com; img-src 'self' data: https:; object-src 'none'; frame-ancestors 'none'; base-uri 'self';"
#
#       }
#       @xmlrpc path /xmlrpc.php
#       respond @xmlrpc 403
#
#       @forbidden {
#               path /readme.html
#               path /wp-config.php
#               path *.ini
#               path *.env
#               path *.sql
#               path *.bak
#       }
#       respond @forbidden 403
#
#       @uploads_php path /wp-content/uploads/*.php
#       respond @uploads_php 403
#
#
#       @login {
#               path /wp-login.php
#       }
#       rate_limit @login {
#               zone login {
#                       key {remote_ip}
#                       events 10
#                       window 2m
#                       burst 5
#               }
#       }

        log {
            output file /var/log/caddy/example.access.log {
                roll_size 10mb
                roll_keep 5
                roll_keep_for 720h
            }
        }
}
```

### docker-compose
File: `caddy/docker-compose-caddy.yml`
```yaml
version: '3.8'

services:
  caddy:
    build: .
    container_name: caddy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ../wordpress_data:/var/www/html   # share WordPress data
      - ./waf:/etc/caddy/waf              # waf config
      - ./Caddyfile:/etc/caddy/Caddyfile
      - ./log:/var/log/caddy  # mount folder log ra host
    networks:
      - wp-network
    depends_on:
      - wordpress

networks:
  wp-network:
    external: true
```

### Chạy stack

#### 1. Start WordPress + MariaDB:

```bash
docker compose -f docker-compose-wordpress.yml up -d
```

#### 2. Start Caddy:

```bash
docker compose -f caddy/docker-compose-caddy.yml up -d --build
```

---

### ✅ Lưu ý:

* Plugin Coraza WAF sẽ load từ `/etc/caddy/waf`.
* Nếu muốn custom CRS rules, bạn chỉnh `crs-setup-custom.conf` hoặc thêm rules vào `coreruleset/rules/`.
* Volume `../wordpress_data:/var/www/html` để Caddy đọc PHP + static files.
* Port 443 + TLS setup trong Caddyfile; dev/test có thể dùng `:80` trước.

---