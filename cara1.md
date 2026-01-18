# 🐳 Docker Multi-Website Deployment

Deploy multiple websites ke satu VPS dengan Docker + Cloudflare.

---

## 🏗️ SETUP AWAL (Sekali Saja)

```bash
# Login ke VPS
ssh root@YOUR_IP

# Install Docker
apt update && apt upgrade -y
curl -fsSL https://get.docker.com -o get-docker.sh && sh get-docker.sh
apt install docker-compose-plugin -y

# Buat struktur & network
mkdir -p /opt/docker-apps/{nginx-proxy,postgres}
docker network create webapps
```

---

## 🐘 SETUP POSTGRESQL (Sekali Saja)

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
      POSTGRES_PASSWORD: YOUR_PASSWORD
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - webapps

volumes:
  postgres_data:

networks:
  webapps:
    external: true
```

```bash
cd /opt/docker-apps/postgres && docker compose up -d
```

---

## 🌐 SETUP NGINX PROXY (Sekali Saja)

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
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - webapps

networks:
  webapps:
    external: true
```

```bash
nano /opt/docker-apps/nginx-proxy/nginx.conf
```

```nginx
events {
    worker_connections 1024;
}

http {
    # TAMBAHKAN DOMAIN BARU DI BAWAH INI
}
```

```bash
cd /opt/docker-apps/nginx-proxy && docker compose up -d
```

---

# 📁 TEMPLATE: Tambah Project Baru

## A. Static HTML (Non-Laravel)

```bash
# 1. Buat folder
mkdir -p /opt/docker-apps/NAMA_PROJECT/src

# 2. Buat Dockerfile
nano /opt/docker-apps/NAMA_PROJECT/Dockerfile
```

```dockerfile
FROM nginx:alpine
COPY src/ /usr/share/nginx/html/
EXPOSE 80
```

```bash
# 3. Buat docker-compose.yml
nano /opt/docker-apps/NAMA_PROJECT/docker-compose.yml
```

```yaml
services:
  app:
    build: .
    container_name: NAMA_PROJECT_app
    restart: always
    networks:
      - webapps

networks:
  webapps:
    external: true
```

```bash
# 4. Clone / upload source code
cd /opt/docker-apps/NAMA_PROJECT/src
git clone https://github.com/USERNAME/REPO.git .

# 5. Build & start
cd /opt/docker-apps/NAMA_PROJECT && docker compose up -d --build
```

---

## B. Laravel Project

```bash
# 1. Buat folder
mkdir -p /opt/docker-apps/NAMA_PROJECT/src

# 2. Buat database
docker exec -it shared_postgres psql -U webadmin -c "CREATE DATABASE db_NAMA_PROJECT;"

# 3. Buat Dockerfile
nano /opt/docker-apps/NAMA_PROJECT/Dockerfile
```

```dockerfile
FROM php:8.2-fpm-alpine

RUN apk add --no-cache nginx supervisor libpng-dev libzip-dev postgresql-dev \
    && docker-php-ext-install pdo pdo_pgsql zip gd bcmath

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer
COPY nginx.conf /etc/nginx/http.d/default.conf
COPY supervisord.ini /etc/supervisor.d/supervisord.ini

WORKDIR /var/www/html
COPY src/ /var/www/html/
RUN composer install --optimize-autoloader --no-dev --no-interaction
RUN chown -R www-data:www-data /var/www/html \
    && chmod -R 755 /var/www/html/storage /var/www/html/bootstrap/cache

EXPOSE 80
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]
```

```bash
# 4. Buat nginx.conf
nano /opt/docker-apps/NAMA_PROJECT/nginx.conf
```

```nginx
server {
    listen 80;
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
}
```

```bash
# 5. Buat supervisord.ini
nano /opt/docker-apps/NAMA_PROJECT/supervisord.ini
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
```

```bash
# 6. Buat docker-compose.yml
nano /opt/docker-apps/NAMA_PROJECT/docker-compose.yml
```

```yaml
services:
  app:
    build: .
    container_name: NAMA_PROJECT_app
    restart: always
    environment:
      APP_ENV: production
      APP_DEBUG: "false"
      APP_URL: https://DOMAIN.com
      DB_CONNECTION: pgsql
      DB_HOST: shared_postgres
      DB_PORT: 5432
      DB_DATABASE: db_NAMA_PROJECT
      DB_USERNAME: webadmin
      DB_PASSWORD: YOUR_PASSWORD
    volumes:
      - ./src/storage:/var/www/html/storage
    networks:
      - webapps

networks:
  webapps:
    external: true
```

```bash
# 7. Clone source code
cd /opt/docker-apps/NAMA_PROJECT/src
git clone https://github.com/USERNAME/REPO.git .

# 8. Build & start
cd /opt/docker-apps/NAMA_PROJECT && docker compose up -d --build

# 9. Setup Laravel
docker exec NAMA_PROJECT_app php artisan key:generate
docker exec NAMA_PROJECT_app php artisan migrate --force
docker exec NAMA_PROJECT_app php artisan storage:link
```

---

## C. Tambah ke Nginx Proxy

```bash
nano /opt/docker-apps/nginx-proxy/nginx.conf
```

**Tambahkan block berikut di dalam `http { }`:**

```nginx
    server {
        listen 80;
        server_name DOMAIN.com www.DOMAIN.com;
        
        location / {
            proxy_pass http://NAMA_PROJECT_app:80;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
```

```bash
# Restart nginx
docker compose -f /opt/docker-apps/nginx-proxy/docker-compose.yml restart
```

---

## D. Setup Cloudflare

1. Login Cloudflare → Pilih domain
2. **DNS** → Tambah A record: `@` → `YOUR_VPS_IP`
3. **SSL/TLS** → Set ke **"Flexible"**

---

# 🔧 Commands Berguna

```bash
# Lihat semua container
docker ps

# Lihat logs
docker logs NAMA_PROJECT_app

# Rebuild setelah update
cd /opt/docker-apps/NAMA_PROJECT && docker compose up -d --build

# Laravel commands
docker exec NAMA_PROJECT_app php artisan cache:clear
docker exec NAMA_PROJECT_app php artisan migrate

# Hapus project
cd /opt/docker-apps/NAMA_PROJECT && docker compose down
rm -rf /opt/docker-apps/NAMA_PROJECT
```

---

# 📋 Checklist Tambah Project Baru

- [ ] Buat folder `/opt/docker-apps/NAMA_PROJECT`
- [ ] Buat database (jika Laravel)
- [ ] Buat file Docker (Dockerfile, docker-compose.yml, dll)
- [ ] Clone/upload source code ke `src/`
- [ ] Build & start container
- [ ] Setup Laravel (key, migrate, storage:link)
- [ ] Tambah domain di nginx.conf
- [ ] Restart nginx proxy
- [ ] Setup DNS di Cloudflare
