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
ssh root@IP_VPS
apt update && apt upgrade -y
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
apt install docker-compose-plugin -y
```

### 1.2 Buat Struktur Folder

```bash
mkdir -p /opt/docker-apps/{nginx-proxy,forlizz/src,smartagri/src,postgres}
docker network create webapps
```

---

## 🗄️ BAGIAN 2: Setup PostgreSQL

File: `/opt/docker-apps/postgres/docker-compose.yml`

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
      - "5432:5432"

volumes:
  postgres_data:

networks:
  webapps:
    external: true
```

File: `/opt/docker-apps/postgres/init-databases.sql`

```sql
CREATE DATABASE db_forlizz;
CREATE DATABASE db_smartagri;
GRANT ALL PRIVILEGES ON DATABASE db_forlizz TO webadmin;
GRANT ALL PRIVILEGES ON DATABASE db_smartagri TO webadmin;
```

```bash
cd /opt/docker-apps/postgres && docker compose up -d
```

---

## 📦 BAGIAN 3: Setup Laravel Container

### Dockerfile

```dockerfile
FROM php:8.4-fpm-alpine

RUN apk add --no-cache nginx supervisor libpng-dev libzip-dev postgresql-dev \
    && docker-php-ext-install pdo pdo_pgsql zip gd bcmath

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer
COPY nginx.conf /etc/nginx/http.d/default.conf
RUN mkdir -p /etc/supervisor.d
COPY supervisord.ini /etc/supervisor.d/supervisord.ini

WORKDIR /var/www/html
COPY src/ /var/www/html/
RUN composer install --optimize-autoloader --no-dev
RUN chown -R www-data:www-data /var/www/html \
    && chmod -R 755 /var/www/html/storage \
    && chmod -R 755 /var/www/html/bootstrap/cache

EXPOSE 80
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]
```

### nginx.conf (Internal)

```nginx
server {
    listen 80;
    server_name _;
    root /var/www/html/public;
    index index.php;
    location / { try_files $uri $uri/ /index.php?$query_string; }
    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

### docker-compose.yml

```yaml
services:
  app:
    build: .
    container_name: forlizz_app  # atau smartagri_app
    restart: always
    environment:
      APP_ENV: production
      APP_DEBUG: "false"
      APP_URL: https://forlizz.online
      DB_CONNECTION: pgsql
      DB_HOST: shared_postgres
      DB_DATABASE: db_forlizz
      DB_USERNAME: webadmin
      DB_PASSWORD: password_kuat_anda
    networks:
      - webapps

networks:
  webapps:
    external: true
```

---

## 🔒 BAGIAN 4: Setup SSL

### 4.1 Nginx HTTP-Only Config (Sementara)

```nginx
events { worker_connections 1024; }
http {
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
}
```

### 4.2 Generate SSL

```bash
mkdir -p /opt/docker-apps/nginx-proxy/html/.well-known/acme-challenge

docker run -it --rm \
  -v /opt/docker-apps/nginx-proxy/certs:/etc/letsencrypt \
  -v /opt/docker-apps/nginx-proxy/html:/usr/share/nginx/html \
  --network webapps \
  certbot/certbot certonly --webroot \
  -w /usr/share/nginx/html \
  -d yourdomain.com -d www.yourdomain.com \
  --email your@email.com --agree-tos --no-eff-email
```

### 4.3 Nginx HTTPS Config

```nginx
events { worker_connections 1024; }
http {
    server {
        listen 80;
        server_name forlizz.online www.forlizz.online;
        location /.well-known/acme-challenge/ { root /usr/share/nginx/html; }
        location / { return 301 https://$host$request_uri; }
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
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

---

## 🆘 Troubleshooting

### 403 Forbidden
```bash
docker exec app_name chmod -R 755 /var/www/html/public
docker exec app_name chown -R www-data:www-data /var/www/html/public
```

### .env / APP_KEY Missing
```bash
docker exec app_name sh -c 'echo "APP_NAME=Laravel
APP_ENV=production
APP_KEY=
APP_DEBUG=false
DB_CONNECTION=pgsql
DB_HOST=shared_postgres
DB_DATABASE=db_name
DB_USERNAME=webadmin
DB_PASSWORD=password" > .env'

docker exec app_name php artisan key:generate
```

### Container Restart Loop
```bash
docker logs container_name --tail 50
```

---

## 🛠️ Commands Berguna

```bash
docker ps                                    # Lihat containers
docker logs app_name                         # Lihat logs
docker exec -it app_name sh                  # Masuk container
docker compose up -d --build                 # Rebuild container
docker exec app_name php artisan cache:clear # Clear cache
```
