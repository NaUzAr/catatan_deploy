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

## 📡 SETUP MQTT BROKER (Sekali Saja)

```bash
mkdir -p /opt/docker-apps/mqtt/{config,data,log}
nano /opt/docker-apps/mqtt/docker-compose.yml
```

```yaml
services:
  mqtt:
    image: eclipse-mosquitto:2
    container_name: mqtt_broker
    restart: always
    ports:
      - "1883:1883"   # MQTT
      - "9001:9001"   # WebSocket (opsional)
    volumes:
      - ./config/mosquitto.conf:/mosquitto/config/mosquitto.conf
      - ./data:/mosquitto/data
      - ./log:/mosquitto/log
    networks:
      - webapps

networks:
  webapps:
    external: true
```

```bash
nano /opt/docker-apps/mqtt/config/mosquitto.conf
```

```
# Mosquitto Configuration
listener 1883
listener 9001
protocol websockets

# Authentication (opsional, untuk production)
allow_anonymous true

# Persistence
persistence true
persistence_location /mosquitto/data/

# Logging
log_dest file /mosquitto/log/mosquitto.log
log_dest stdout
```

```bash
# Start MQTT Broker
cd /opt/docker-apps/mqtt && docker compose up -d

# Test koneksi
docker exec mqtt_broker mosquitto_pub -t "test" -m "hello"
docker exec mqtt_broker mosquitto_sub -t "test" -C 1
```

### Koneksi MQTT dari Device/Laravel

```
Host: 203.194.115.76 (atau smartagri.web.id)
Port: 1883
WebSocket Port: 9001
Username: (kosong jika allow_anonymous true)
Password: (kosong jika allow_anonymous true)
```

### MQTT dengan Username & Password (Recommended)

**1. Buat password file:**
```bash
# Masuk ke container
docker exec -it mqtt_broker sh

# Buat user (ganti USERNAME dengan nama user)
mosquitto_passwd -c /mosquitto/config/password.txt USERNAME
# Ketik password 2x, lalu exit
exit
```

**2. Update mosquitto.conf:**
```bash
nano /opt/docker-apps/mqtt/config/mosquitto.conf
```

```
# Mosquitto Configuration
listener 1883
listener 9001
protocol websockets

# Authentication - DENGAN PASSWORD
allow_anonymous false
password_file /mosquitto/config/password.txt

# Persistence
persistence true
persistence_location /mosquitto/data/

# Logging
log_dest file /mosquitto/log/mosquitto.log
log_dest stdout
```

**3. Restart MQTT:**
```bash
cd /opt/docker-apps/mqtt && docker compose restart

# Test dengan auth
docker exec mqtt_broker mosquitto_pub -t "test" -m "hello" -u USERNAME -P PASSWORD
```

**Contoh koneksi ESP32:**
```cpp
const char* mqtt_server = "203.194.115.76";
const char* mqtt_user = "USERNAME";
const char* mqtt_pass = "PASSWORD";

client.connect("ESP32Client", mqtt_user, mqtt_pass);
```

### Firewall untuk MQTT

```bash
ufw allow 1883/tcp   # MQTT
ufw allow 9001/tcp   # WebSocket (opsional)
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
    # Logging (untuk debug & monitoring)
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    # Gzip compression (website lebih cepat)
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    # ===== DOMAIN 1: forlizz.online =====
    server {
        listen 80;
        server_name forlizz.online www.forlizz.online;
        
        location / {
            proxy_pass http://forlizz_app:80;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    # ===== DOMAIN 2: smartagri.web.id =====
    server {
        listen 80;
        server_name smartagri.web.id www.smartagri.web.id;
        
        location / {
            proxy_pass http://smartagri_app:80;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    # ===== TEMPLATE: Copy untuk domain baru =====
    # server {
    #     listen 80;
    #     server_name DOMAIN.com www.DOMAIN.com;
    #     
    #     location / {
    #         proxy_pass http://CONTAINER_NAME:80;
    #         proxy_set_header Host $host;
    #         proxy_set_header X-Real-IP $remote_addr;
    #         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    #         proxy_set_header X-Forwarded-Proto $scheme;
    #     }
    # }
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

---

# 🔌 PostgreSQL Remote Access (Opsional)

Jika ingin akses database dari server/device lain.

## 1. Update docker-compose.yml

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
    ports:
      - "5432:5432"    # Tambahkan ini untuk remote access
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

## 2. Whitelist IP dengan Firewall

```bash
# Izinkan hanya IP tertentu
ufw allow from 182.8.225.79 to any port 5432    # Contoh IP 1
ufw allow from 103.xxx.xxx.xxx to any port 5432 # Contoh IP 2

# Reload firewall
ufw reload

# Cek status
ufw status
```

## 3. Koneksi dari Device Lain

```
Host: YOUR_VPS_IP
Port: 5432
Database: db_smartagri
Username: webadmin
Password: YOUR_PASSWORD
```

**Contoh di Laravel .env:**
```env
DB_CONNECTION=pgsql
DB_HOST=203.194.115.76
DB_PORT=5432
DB_DATABASE=db_smartagri
DB_USERNAME=webadmin
DB_PASSWORD=YOUR_PASSWORD
```

---

# 🆘 Troubleshooting

## Error: Cloudflare 521 "Web server is down"

**Penyebab:** Cloudflare tidak bisa connect ke server.

**Solusi:**
```bash
# Cek container running
docker ps

# Cek nginx proxy logs
docker logs nginx_proxy --tail 20

# Test akses lokal
curl -I http://localhost

# Cek firewall
ufw status
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 22/tcp
ufw enable
```

---

## Error: "host not found in upstream" di Nginx

**Penyebab:** Container yang di-reference di nginx.conf belum running.

**Solusi:**
1. Cek nama container di docker-compose.yml (`container_name`)
2. Pastikan container running: `docker ps`
3. Update nginx.conf dengan nama container yang benar
4. Restart nginx: `docker compose -f /opt/docker-apps/nginx-proxy/docker-compose.yml restart`

---

## Error: 500 Internal Server Error

**Penyebab umum:**

### 1. APP_KEY kosong
```bash
docker exec CONTAINER_NAME php artisan key:generate --force
```

### 2. .env salah (DB_HOST masih 127.0.0.1)
```bash
# Update .env di container
docker exec -it CONTAINER_NAME sh -c 'cat > .env << EOF
APP_NAME=SmartAgri
APP_ENV=production
APP_KEY=base64:YOUR_KEY_HERE
APP_DEBUG=false
APP_URL=https://DOMAIN.com

DB_CONNECTION=pgsql
DB_HOST=shared_postgres
DB_PORT=5432
DB_DATABASE=db_NAME
DB_USERNAME=webadmin
DB_PASSWORD=YOUR_PASSWORD

SESSION_DRIVER=file
CACHE_STORE=file
QUEUE_CONNECTION=sync
EOF'

docker exec CONTAINER_NAME php artisan config:clear
docker exec CONTAINER_NAME php artisan cache:clear
```

### 3. Database belum dibuat
```bash
docker exec shared_postgres psql -U webadmin -c "CREATE DATABASE db_NAME;"
docker exec CONTAINER_NAME php artisan migrate --force
```

### 4. Permission storage folder
```bash
docker exec CONTAINER_NAME chmod -R 777 storage
docker exec CONTAINER_NAME chmod -R 777 bootstrap/cache
```

---

## Error: "composer.json not found" saat build

**Penyebab:** Git clone membuat subfolder tambahan.

**Solusi:**
```bash
# Cek isi folder src
ls -la /opt/docker-apps/PROJECT/src/

# Jika ada subfolder, pindahkan isinya
mv /opt/docker-apps/PROJECT/src/SUBFOLDER/* /opt/docker-apps/PROJECT/src/
rm -rf /opt/docker-apps/PROJECT/src/SUBFOLDER

# Atau clone dengan benar (pakai titik di akhir)
cd /opt/docker-apps/PROJECT/src
rm -rf *
git clone https://github.com/USER/REPO.git .
```

---

## Error: PHP version mismatch

**Penyebab:** composer.lock butuh PHP version berbeda dari Dockerfile.

**Solusi:**
```bash
# Update Dockerfile ke PHP yang sesuai
nano /opt/docker-apps/PROJECT/Dockerfile
# Ganti: FROM php:8.2-fpm-alpine → FROM php:8.4-fpm-alpine

# Atau hapus composer.lock
rm /opt/docker-apps/PROJECT/src/composer.lock

# Rebuild
docker compose up -d --build
```

---

## Error: vendor folder corrupt

**Penyebab:** vendor folder dari development ikut ter-upload.

**Solusi - Update Dockerfile:**
```dockerfile
RUN rm -rf vendor composer.lock && composer install --optimize-autoloader --no-dev --no-interaction
```

---

## Cloudflare SSL Setup

1. Login **Cloudflare Dashboard** → Pilih domain
2. **DNS** → Tambah A record: `@` → `YOUR_VPS_IP` (Proxy: ON ☁️)
3. **SSL/TLS** → **Overview** → Pilih **"Flexible"**
4. Tunggu 1-5 menit untuk propagate
5. Jika masih error, **Caching** → **Purge Everything**

---

## Firewall Setup

```bash
# Enable UFW
ufw allow 22/tcp    # SSH
ufw allow 80/tcp    # HTTP
ufw allow 443/tcp   # HTTPS
ufw allow 5432/tcp  # PostgreSQL (jika remote access)
ufw enable
ufw status
```
