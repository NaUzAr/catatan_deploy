# 🐳 Panduan Deploy Multi-Website Laravel dengan Docker

Panduan lengkap untuk deploy **multiple Laravel websites** ke **satu VPS** menggunakan **Docker**.

## 📋 Overview Arsitektur

```
                         ┌─────────────────────┐
         Internet ──────▶│   Nginx Proxy       │ :80/:443
                         │   (Reverse Proxy)   │
                         └──────────┬──────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              ▼                                           ▼
    ┌──────────────────┐                      ┌──────────────────┐
    │  forlizz.online  │                      │ smartagri.web.id │
    │    Container     │                      │    Container     │
    │  (PHP + Nginx)   │                      │  (PHP + Nginx)   │
    └────────┬─────────┘                      └────────┬─────────┘
             │                                          │
             └────────────────┬─────────────────────────┘
                              ▼
                    ┌──────────────────┐
                    │    PostgreSQL    │
                    │    Container     │
                    └──────────────────┘
```

---

## 📁 Struktur Folder di VPS

```
/opt/docker-apps/
├── nginx-proxy/           # Reverse proxy untuk semua domain
│   ├── docker-compose.yml
│   └── nginx.conf
├── forlizz/               # Project 1: forlizz.online
│   ├── docker-compose.yml
│   ├── Dockerfile
│   ├── nginx.conf
│   └── src/               # Laravel source code
├── smartagri/             # Project 2: smartagri.web.id
│   ├── docker-compose.yml
│   ├── Dockerfile
│   ├── nginx.conf
│   └── src/               # Laravel source code
└── postgres/              # Shared database
    └── docker-compose.yml
```

---

## 🔧 BAGIAN 1: Setup VPS

### 1.1 Install Docker & Docker Compose

```bash
# Login ke VPS
ssh root@IP_VPS

# Update system
apt update && apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Install Docker Compose
apt install docker-compose-plugin -y

# Verify installation
docker --version
docker compose version
```

### 1.2 Buat Struktur Folder

```bash
mkdir -p /opt/docker-apps/{nginx-proxy,forlizz/src,smartagri/src,postgres}
cd /opt/docker-apps
```

---

## 🗄️ BAGIAN 2: Setup PostgreSQL

### 2.1 docker-compose.yml untuk PostgreSQL

```bash
nano /opt/docker-apps/postgres/docker-compose.yml
```

```yaml
services:
  postgres:
    image: postgres:15-alpine
    container_name: shared_postgres
    restart: always
    environment:
      POSTGRES_USER: webadmin
      POSTGRES_PASSWORD: password_kuat_anda
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-databases.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - webapps
    ports:
      - "5432:5432"  # Optional: untuk akses dari luar

volumes:
  postgres_data:

networks:
  webapps:
    external: true
```

### 2.2 Init Script untuk Multiple Databases

```bash
nano /opt/docker-apps/postgres/init-databases.sql
```

```sql
-- Database untuk forlizz.online
CREATE DATABASE db_forlizz;

-- Database untuk smartagri.web.id
CREATE DATABASE db_smartagri;

-- Grant permissions
GRANT ALL PRIVILEGES ON DATABASE db_forlizz TO webadmin;
GRANT ALL PRIVILEGES ON DATABASE db_smartagri TO webadmin;
```

### 2.3 Buat Network & Start PostgreSQL

```bash
# Buat shared network
docker network create webapps

# Start PostgreSQL
cd /opt/docker-apps/postgres
docker compose up -d
```

---

## 🌐 BAGIAN 3: Setup Nginx Reverse Proxy

### 3.1 docker-compose.yml

```bash
nano /opt/docker-apps/nginx-proxy/docker-compose.yml
```

```yaml
services:
  nginx-proxy:
    image: nginx:alpine
    container_name: nginx_proxy
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
      - ./html:/usr/share/nginx/html:ro
      - certbot_data:/var/www/certbot:ro
    networks:
      - webapps
    depends_on:
      - certbot

  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - ./certs:/etc/letsencrypt
      - certbot_data:/var/www/certbot
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"

volumes:
  certbot_data:

networks:
  webapps:
    external: true
```

### 3.2 nginx.conf

```bash
nano /opt/docker-apps/nginx-proxy/nginx.conf
```

```nginx
events {
    worker_connections 1024;
}

http {
    # Logging
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    # ========================================
    # forlizz.online
    # ========================================
    server {
        listen 80;
        server_name forlizz.online www.forlizz.online;

        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

        location / {
            return 301 https://$host$request_uri;
        }
    }

    server {
        listen 443 ssl;
        server_name forlizz.online www.forlizz.online;

        ssl_certificate /etc/nginx/certs/live/forlizz.online/fullchain.pem;
        ssl_certificate_key /etc/nginx/certs/live/forlizz.online/privkey.pem;

        location / {
            proxy_pass http://forlizz_app:80;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    # ========================================
    # smartagri.web.id
    # ========================================
    server {
        listen 80;
        server_name smartagri.web.id www.smartagri.web.id;

        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

        location / {
            return 301 https://$host$request_uri;
        }
    }

    server {
        listen 443 ssl;
        server_name smartagri.web.id www.smartagri.web.id;

        ssl_certificate /etc/nginx/certs/live/smartagri.web.id/fullchain.pem;
        ssl_certificate_key /etc/nginx/certs/live/smartagri.web.id/privkey.pem;

        location / {
            proxy_pass http://smartagri_app:80;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    # ========================================
    # TEMPLATE: Tambah domain baru
    # Copy block di bawah dan ganti:
    # - newdomain.com
    # - newapp_container
    # ========================================
    # server {
    #     listen 80;
    #     server_name newdomain.com www.newdomain.com;
    #     location /.well-known/acme-challenge/ { root /var/www/certbot; }
    #     location / { return 301 https://$host$request_uri; }
    # }
    # server {
    #     listen 443 ssl;
    #     server_name newdomain.com www.newdomain.com;
    #     ssl_certificate /etc/nginx/certs/live/newdomain.com/fullchain.pem;
    #     ssl_certificate_key /etc/nginx/certs/live/newdomain.com/privkey.pem;
    #     location / {
    #         proxy_pass http://newapp_container:80;
    #         proxy_set_header Host $host;
    #         proxy_set_header X-Real-IP $remote_addr;
    #         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    #         proxy_set_header X-Forwarded-Proto $scheme;
    #     }
    # }
}
```

---

## 📦 BAGIAN 4: Setup Laravel Container

### 4.1 Dockerfile (Sama untuk semua project)

```bash
nano /opt/docker-apps/forlizz/Dockerfile
```

```dockerfile
FROM php:8.4-fpm-alpine

# Install dependencies
RUN apk add --no-cache \
    nginx \
    supervisor \
    libpng-dev \
    libzip-dev \
    postgresql-dev \
    && docker-php-ext-install pdo pdo_pgsql zip gd bcmath

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Nginx config
COPY nginx.conf /etc/nginx/http.d/default.conf

# Supervisor config
RUN mkdir -p /etc/supervisor.d
COPY supervisord.ini /etc/supervisor.d/supervisord.ini

# Set working directory
WORKDIR /var/www/html

# Copy source code
COPY src/ /var/www/html/

# Install dependencies
RUN composer install --optimize-autoloader --no-dev

# Set permissions
RUN chown -R www-data:www-data /var/www/html \
    && chmod -R 755 /var/www/html/storage \
    && chmod -R 755 /var/www/html/bootstrap/cache

EXPOSE 80

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]
```

### 4.2 nginx.conf (Internal)

```bash
nano /opt/docker-apps/forlizz/nginx.conf
```

```nginx
server {
    listen 80;
    server_name _;
    root /var/www/html/public;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

### 4.3 supervisord.ini

```bash
nano /opt/docker-apps/forlizz/supervisord.ini
```

```ini
[supervisord]
nodaemon=true

[program:php-fpm]
command=/usr/local/sbin/php-fpm
autostart=true
autorestart=true

[program:nginx]
command=/usr/sbin/nginx -g "daemon off;"
autostart=true
autorestart=true

[program:laravel-worker]
command=php /var/www/html/artisan queue:work --sleep=3 --tries=3
autostart=true
autorestart=true
user=www-data
numprocs=2
process_name=%(program_name)s_%(process_num)02d
```

### 4.4 docker-compose.yml untuk forlizz

```bash
nano /opt/docker-apps/forlizz/docker-compose.yml
```

```yaml
services:
  app:
    build: .
    container_name: forlizz_app
    restart: always
    environment:
      APP_ENV: production
      APP_DEBUG: "false"
      APP_URL: https://forlizz.online
      DB_CONNECTION: pgsql
      DB_HOST: shared_postgres
      DB_PORT: 5432
      DB_DATABASE: db_forlizz
      DB_USERNAME: webadmin
      DB_PASSWORD: password_kuat_anda
    volumes:
      - ./src/storage:/var/www/html/storage
    networks:
      - webapps

networks:
  webapps:
    external: true
```

### 4.5 Copy untuk smartagri

```bash
# Copy semua file
cp /opt/docker-apps/forlizz/Dockerfile /opt/docker-apps/smartagri/
cp /opt/docker-apps/forlizz/nginx.conf /opt/docker-apps/smartagri/
cp /opt/docker-apps/forlizz/supervisord.ini /opt/docker-apps/smartagri/

# Buat docker-compose.yml dengan perubahan
nano /opt/docker-apps/smartagri/docker-compose.yml
```

```yaml
services:
  app:
    build: .
    container_name: smartagri_app
    restart: always
    environment:
      APP_ENV: production
      APP_DEBUG: "false"
      APP_URL: https://smartagri.web.id
      DB_CONNECTION: pgsql
      DB_HOST: shared_postgres
      DB_PORT: 5432
      DB_DATABASE: db_smartagri
      DB_USERNAME: webadmin
      DB_PASSWORD: password_kuat_anda
    volumes:
      - ./src/storage:/var/www/html/storage
    networks:
      - webapps

networks:
  webapps:
    external: true
```

---

## 🚀 BAGIAN 5: Upload & Deploy

### 5.1 Upload Source Code

**Dari Windows (PowerShell):**

```powershell
# Upload forlizz (project belajar)
scp -r "D:\01. Program\00. OTW JADI WEB DEV\Beljar\laravel\belajar\*" root@IP_VPS:/opt/docker-apps/forlizz/src/

# Upload smartagri (project web2)
scp -r "D:\01. Program\00. OTW JADI WEB DEV\Beljar\laravel\web2\*" root@IP_VPS:/opt/docker-apps/smartagri/src/
```

### 5.2 Start Semua Services

```bash
# Start PostgreSQL
cd /opt/docker-apps/postgres
docker compose up -d

# Build & Start forlizz
cd /opt/docker-apps/forlizz
docker compose up -d --build

# Build & Start smartagri
cd /opt/docker-apps/smartagri
docker compose up -d --build

# Start Nginx Proxy (setelah SSL setup)
cd /opt/docker-apps/nginx-proxy
docker compose up -d
```

### 5.3 Run Migrations

```bash
# forlizz
docker exec forlizz_app php artisan key:generate
docker exec forlizz_app php artisan migrate --force
docker exec forlizz_app php artisan storage:link

# smartagri
docker exec smartagri_app php artisan key:generate
docker exec smartagri_app php artisan migrate --force
docker exec smartagri_app php artisan storage:link
```

---

## 🔒 BAGIAN 6: Setup SSL

### 6.1 Persiapan: Nginx HTTP-Only Config

Sebelum generate SSL, gunakan config nginx HTTP-only dulu:

```nginx
events {
    worker_connections 1024;
}
http {
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    # forlizz.online (HTTP only - sementara)
    server {
        listen 80;
        server_name forlizz.online www.forlizz.online;
        location /.well-known/acme-challenge/ { root /usr/share/nginx/html; }
        location / {
            proxy_pass http://forlizz_app:80;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }

    # smartagri.web.id (HTTP only - sementara)
    server {
        listen 80;
        server_name smartagri.web.id www.smartagri.web.id;
        location /.well-known/acme-challenge/ { root /usr/share/nginx/html; }
        location / {
            proxy_pass http://smartagri_app:80;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

### 6.2 Buat Folder & Restart Nginx

```bash
mkdir -p /opt/docker-apps/nginx-proxy/html/.well-known/acme-challenge
cd /opt/docker-apps/nginx-proxy
docker compose restart nginx-proxy
```

### 6.3 Generate SSL Certificates

```bash
# SSL untuk domain 1
docker run -it --rm \
  -v /opt/docker-apps/nginx-proxy/certs:/etc/letsencrypt \
  -v /opt/docker-apps/nginx-proxy/html:/usr/share/nginx/html \
  --network webapps \
  certbot/certbot certonly --webroot \
  -w /usr/share/nginx/html \
  -d forlizz.online -d www.forlizz.online \
  --email your@email.com --agree-tos --no-eff-email

# SSL untuk domain 2
docker run -it --rm \
  -v /opt/docker-apps/nginx-proxy/certs:/etc/letsencrypt \
  -v /opt/docker-apps/nginx-proxy/html:/usr/share/nginx/html \
  --network webapps \
  certbot/certbot certonly --webroot \
  -w /usr/share/nginx/html \
  -d smartagri.web.id -d www.smartagri.web.id \
  --email your@email.com --agree-tos --no-eff-email
```

### 6.4 Update Nginx Config dengan HTTPS

Setelah SSL berhasil, update nginx.conf:

```nginx
events {
    worker_connections 1024;
}
http {
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    # forlizz.online - HTTP redirect to HTTPS
    server {
        listen 80;
        server_name forlizz.online www.forlizz.online;
        location /.well-known/acme-challenge/ { root /usr/share/nginx/html; }
        location / { return 301 https://$host$request_uri; }
    }
    
    # forlizz.online - HTTPS
    server {
        listen 443 ssl;
        server_name forlizz.online www.forlizz.online;
        ssl_certificate /etc/nginx/certs/live/forlizz.online/fullchain.pem;
        ssl_certificate_key /etc/nginx/certs/live/forlizz.online/privkey.pem;
        location / {
            proxy_pass http://forlizz_app:80;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    # smartagri.web.id - HTTP redirect
    server {
        listen 80;
        server_name smartagri.web.id www.smartagri.web.id;
        location /.well-known/acme-challenge/ { root /usr/share/nginx/html; }
        location / { return 301 https://$host$request_uri; }
    }
    
    # smartagri.web.id - HTTPS
    server {
        listen 443 ssl;
        server_name smartagri.web.id www.smartagri.web.id;
        ssl_certificate /etc/nginx/certs/live/smartagri.web.id/fullchain.pem;
        ssl_certificate_key /etc/nginx/certs/live/smartagri.web.id/privkey.pem;
        location / {
            proxy_pass http://smartagri_app:80;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

### 6.5 Restart Nginx Proxy

```bash
docker compose restart nginx-proxy
```

---

## ➕ BAGIAN 7: Menambah Website Baru

### 7.1 Buat Database Baru

```bash
docker exec -it shared_postgres psql -U webadmin -c "CREATE DATABASE db_newsite;"
```

### 7.2 Buat Folder & Copy Template

```bash
mkdir -p /opt/docker-apps/newsite/src

# Copy files
cp /opt/docker-apps/forlizz/Dockerfile /opt/docker-apps/newsite/
cp /opt/docker-apps/forlizz/nginx.conf /opt/docker-apps/newsite/
cp /opt/docker-apps/forlizz/supervisord.ini /opt/docker-apps/newsite/
```

### 7.3 Buat docker-compose.yml

```bash
nano /opt/docker-apps/newsite/docker-compose.yml
```

```yaml
services:
  app:
    build: .
    container_name: newsite_app
    restart: always
    environment:
      APP_ENV: production
      APP_DEBUG: "false"
      APP_URL: https://newdomain.com
      DB_CONNECTION: pgsql
      DB_HOST: shared_postgres
      DB_PORT: 5432
      DB_DATABASE: db_newsite
      DB_USERNAME: webadmin
      DB_PASSWORD: password_kuat_anda
    volumes:
      - ./src/storage:/var/www/html/storage
    networks:
      - webapps

networks:
  webapps:
    external: true
```

### 7.4 Update Nginx Proxy

Edit `/opt/docker-apps/nginx-proxy/nginx.conf` dan tambahkan block baru (lihat template di file).

### 7.5 Upload, Build & Deploy

```bash
# Upload source
scp -r /path/to/project/* root@IP_VPS:/opt/docker-apps/newsite/src/

# Build & start
cd /opt/docker-apps/newsite
docker compose up -d --build

# Run migrations
docker exec newsite_app php artisan key:generate
docker exec newsite_app php artisan migrate --force

# Generate SSL
docker run -it --rm \
  -v /opt/docker-apps/nginx-proxy/certs:/etc/letsencrypt \
  certbot/certbot certonly --webroot \
  -w /var/www/certbot \
  -d newdomain.com -d www.newdomain.com

# Restart nginx
docker compose -f /opt/docker-apps/nginx-proxy/docker-compose.yml restart
```

---

## 🆘 BAGIAN 8: Troubleshooting

### 8.1 Error: 403 Forbidden

**Penyebab:** Permission folder tidak benar.

```bash
# Fix permission
docker exec forlizz_app chmod -R 755 /var/www/html/public
docker exec forlizz_app chown -R www-data:www-data /var/www/html/public
docker exec forlizz_app chown -R www-data:www-data /var/www/html/storage
```

### 8.2 Error: .env file not found / APP_KEY missing

**Penyebab:** File .env tidak ter-copy atau APP_KEY belum di-set.

```bash
# Buat .env manual (HARUS ada APP_KEY=)
docker exec forlizz_app sh -c 'echo "APP_NAME=Laravel
APP_ENV=production
APP_KEY=
APP_DEBUG=false
APP_URL=http://yourdomain.com
DB_CONNECTION=pgsql
DB_HOST=shared_postgres
DB_PORT=5432
DB_DATABASE=db_name
DB_USERNAME=webadmin
DB_PASSWORD=password_kuat_anda" > .env'

# Generate key
docker exec forlizz_app php artisan key:generate
```

### 8.3 Error: SSL Certificate not found

**Penyebab:** Certbot belum generate SSL.

**Solusi sementara:** Gunakan HTTP only dulu di nginx.conf:

```nginx
server {
    listen 80;
    server_name yourdomain.com;
    
    location / {
        proxy_pass http://yourapp_container:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 8.4 Error: Container keeps restarting

```bash
# Cek logs
docker logs container_name --tail 50

# Cek status
docker ps -a
```

### 8.5 Test Koneksi Antar Container

```bash
# Test dari nginx proxy ke app
docker exec nginx_proxy curl http://forlizz_app:80

# Cek network
docker network inspect webapps
```

---

## 🛠️ Commands Berguna

```bash
# Lihat semua containers
docker ps

# Lihat logs
docker logs forlizz_app
docker logs smartagri_app

# Masuk ke container
docker exec -it forlizz_app sh

# Rebuild setelah update code
docker compose up -d --build

# Clear Laravel cache
docker exec forlizz_app php artisan cache:clear
docker exec forlizz_app php artisan config:clear
docker exec forlizz_app php artisan view:clear
```

---

## ⚖️ Perbandingan: Docker vs Native

| Aspek | Docker | Native |
|-------|--------|--------|
| **Setup awal** | Lebih kompleks | Lebih sederhana |
| **Isolasi** | Setiap app terisolasi | Shared environment |
| **Tambah app baru** | Copy template | Manual config |
| **Update PHP** | Per container | Semua app sekaligus |
| **Resource usage** | Lebih tinggi | Lebih efisien |
| **Scaling** | Mudah | Manual |
| **Rollback** | Image versioning | Git only |

**Rekomendasi:**
- **Docker** → Jika sering deploy, butuh isolasi, atau tim banyak
- **Native** → Jika VPS resource terbatas atau prefer simplicity
