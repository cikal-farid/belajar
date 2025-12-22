# DOCKER

Siap. Ini aku buatkan **rangkuman tahapan “VERSI FINAL / NO DRAMA”** dari nol sampai DEV benar‑benar jalan, **termasuk perbaikan untuk kendala yang barusan kamu alami** (`docker-compose.dev.yml` tidak ada, APP\_KEY placeholder tidak terganti, Harbor drop setelah restart docker, dev.conf salah tipe, dll).

> Fokus runbook ini: **DEV environment di VM2 (devops-ci / 192.168.56.43)** sampai bisa hit:
>
> * `https://dev.local` ✅
> * `https://dev.local/api/go/api/products` ✅
> * `https://dev.local/api/laravel/api/products` ✅

```
https://dev.local
```

```
https://dev.local/api/go/api/products
```

```
https://dev.local/api/laravel/api/products
```

***

## 0) Prasyarat Network & Hostname (sekali saja)

### 0.1 IP & hostname

* VM1: `192.168.56.42` → `k8s-master`
* VM2: `192.168.56.43` → `devops-ci`
* VM3: `192.168.56.44` → `monitoring`

Di semua VM, `/etc/hosts`:

```
192.168.56.42 k8s-master
192.168.56.43 devops-ci
192.168.56.44 monitoring
```

### 0.2 Host PC (Windows/Linux) tambahkan hosts

Di host PC (yang buka browser):

```
192.168.56.43 dev.local
```

***

## 1) Setup dasar VM2 (devops-ci)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl vim git ca-certificates gnupg lsb-release
```

***

## 2) Install Docker (VM2)

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
# logout/login lagi
docker ps
```

✅ Kalau `docker ps` tidak error → OK.

***

## 3) Install & Jalankan Harbor (VM2) di PORT 8081 (agar tidak bentrok dengan nginx-dev 80/443)

> Ini mencegah masalah “connection refused” + bentrok port.

#### Install Harbor

#### \[VM2 – devops-ci]

```bash
cd /opt
sudo wget https://github.com/goharbor/harbor/releases/download/v2.10.0/harbor-online-installer-v2.10.0.tgz
sudo tar xzf harbor-online-installer-v2.10.0.tgz
cd harbor
sudo cp harbor.yml.tmpl harbor.yml
sudo nano harbor.yml
```

### 3.1 Masuk ke folder Harbor dan edit config

```bash
cd /opt/harbor
sudo nano harbor.yml
```

Pastikan bagian ini:

```yaml
hostname: 192.168.56.43

http:
  port: 8081

https:
  enabled: false
```

### 3.2 Apply config

```bash
cd /opt/harbor
sudo docker compose down || true
sudo ./prepare
sudo ./install.sh
sudo docker compose up -d
sudo docker compose ps
```

### 3.3 Verifikasi Harbor

```bash
curl -I http://192.168.56.43:8081
curl -s -o /dev/null -w "%{http_code}\n" http://192.168.56.43:8081/v2/
```

✅ Harus:

* `/` → 200
* `/v2/` → 401 (normal, butuh login)

***

## 4) Set Docker “insecure registry” untuk Harbor HTTP (VM2)

```bash
sudo tee /etc/docker/daemon.json <<EOF
{
  "insecure-registries": ["192.168.56.43:8081"]
}
EOF

sudo systemctl restart docker
```

⚠️ PENTING: setelah restart docker, container Harbor bisa ikut stop.\
Naikkan lagi Harbor:

```bash
cd /opt/harbor
sudo docker compose up -d
sudo docker compose ps
```

Cek insecure registries:

```bash
docker info | sed -n '/Insecure Registries/,+10p'
```

✅ Harus ada `192.168.56.43:8081`.

***

## 5) Clone repo aplikasi (VM2)

```bash
cd ~
git clone https://github.com/Rafianda/three-body-problem.git three-body-problem-main
cd ~/three-body-problem-main
```

Verifikasi isi repo:

```bash
ls -la
```

✅ Harus terlihat folder `frontend/`, `go/`, `laravel/`.

***

## 6) Pastikan file `docker-compose.dev.yml` ADA (ini yang bikin kamu error barusan)

Kamu error:

> `open .../docker-compose.dev.yml: no such file or directory`

Artinya file ini tidak ada atau beda nama.

### 6.1 Cek file compose

```bash
cd ~/three-body-problem-main
ls -la docker-compose*.yml
```

#### Kalau `docker-compose.dev.yml` **tidak ada**, buat file ini (FINAL):

```bash
cd ~/three-body-problem-main

cat > docker-compose.dev.yml <<'EOF'
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: threebody
      MYSQL_USER: threebody
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - devnet
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -uroot -p$${MYSQL_ROOT_PASSWORD} --silent"]
      interval: 5s
      timeout: 3s
      retries: 20

  laravel:
    image: ${HARBOR_URL}/${HARBOR_PROJECT}/laravel:${CI_COMMIT_SHA}
    depends_on:
      mysql:
        condition: service_healthy
    environment:
      APP_ENV: local
      APP_DEBUG: "true"
      APP_KEY: ${APP_KEY}
      APP_URL: https://dev.local/api/laravel
      LOG_CHANNEL: stderr

      DB_CONNECTION: mysql
      DB_HOST: mysql
      DB_PORT: 3306
      DB_DATABASE: threebody
      DB_USERNAME: threebody
      DB_PASSWORD: ${MYSQL_PASSWORD}
    networks:
      - devnet

  goapi:
    image: ${HARBOR_URL}/${HARBOR_PROJECT}/goapi:${CI_COMMIT_SHA}
    depends_on:
      mysql:
        condition: service_healthy
    environment:
      DB_HOST: mysql
      DB_PORT: 3306
      DB_USER: threebody
      DB_PASSWORD: ${MYSQL_PASSWORD}
      DB_NAME: threebody
    networks:
      - devnet

  frontend:
    image: ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${CI_COMMIT_SHA}
    depends_on:
      - laravel
      - goapi
    networks:
      - devnet

  nginx-dev:
    image: nginx:alpine
    depends_on:
      - frontend
      - laravel
      - goapi
    volumes:
      - ./devops/nginx/dev.conf:/etc/nginx/conf.d/dev.conf:ro
      - ./certs/dev:/etc/nginx/certs:ro
    ports:
      - "80:80"
      - "443:443"
    networks:
      - devnet

volumes:
  mysql_data:

networks:
  devnet:
EOF
```

✅ Setelah itu, pastikan file ada:

```bash
ls -la docker-compose.dev.yml
```

***

## 7) Buat `.env.dev.compose` (WAJIB) dengan APP\_KEY yang benar (ini juga jebakan kamu)

Kamu sempat bikin:\
`APP_KEY=base64:ISI_RANDOM_NANTI`\
Lalu command `grep ... || echo ...` **tidak mengganti** karena APP\_KEY sudah ada → akhirnya tetap placeholder.

### 7.1 Buat ulang dengan APP\_KEY asli (cara paling aman)

```bash
cd ~/three-body-problem-main
KEY="$(openssl rand -base64 32)"

cat > .env.dev.compose <<EOF
MYSQL_ROOT_PASSWORD=RootPassDev123!
MYSQL_PASSWORD=UserPassDev123!

HARBOR_URL=192.168.56.43:8081
HARBOR_PROJECT=threebody

CI_COMMIT_SHA=local
APP_KEY=base64:${KEY}
EOF
```

Cek:

```bash
cat .env.dev.compose
```

✅ APP\_KEY harus sudah “base64:xxxxx” bukan placeholder.

***

## 8) Siapkan HTTPS dev.local (self‑signed) untuk nginx-dev

```bash
cd ~/three-body-problem-main
mkdir -p certs/dev

openssl req -newkey rsa:2048 -nodes \
  -keyout certs/dev/dev.key \
  -x509 -days 365 \
  -out certs/dev/dev.crt
# Common Name (CN): dev.local
```

***

## 9) Siapkan Nginx Gateway config (WAJIB) + pastikan file bukan directory

### 9.1 Buat folder

```bash
cd ~/three-body-problem-main
mkdir -p devops/nginx
```

### 9.2 Buat `devops/nginx/dev.conf` (FINAL)

```bash
cat > devops/nginx/dev.conf <<'EOF'
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

server {
    listen 80;
    server_name dev.local;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name dev.local;

    ssl_certificate     /etc/nginx/certs/dev.crt;
    ssl_certificate_key /etc/nginx/certs/dev.key;

    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    location / {
        proxy_pass http://frontend:80;
    }

    location ^~ /api/laravel {
        limit_req zone=api_limit burst=20 nodelay;
        rewrite ^/api/laravel/?(.*)$ /$1 break;
        proxy_pass http://laravel:80;
    }

    location ^~ /api/go {
        limit_req zone=api_limit burst=20 nodelay;
        rewrite ^/api/go/?(.*)$ /$1 break;
        proxy_pass http://goapi:8080;
    }
}
EOF
```

✅ Pastikan itu file:

```bash
ls -l devops/nginx/dev.conf
# harus diawali '-' bukan 'd'
```

***

## 10) Dockerfile multistage (kalau dari repo awal belum ada)

> Kalau file ini sudah ada dari pekerjaan kamu sebelumnya, lewati.\
> Kalau belum, buat ini (FINAL).

### 10.1 Frontend nginx config

```bash
cat > devops/nginx/frontend.conf <<'EOF'
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
EOF
```

### 10.2 Dockerfile Go

```bash
mkdir -p devops/docker

cat > devops/docker/Dockerfile.goapi <<'EOF'
# syntax=docker/dockerfile:1
FROM golang:1.22-alpine AS builder
WORKDIR /src
RUN apk add --no-cache git ca-certificates

COPY go/ ./
RUN go mod download

RUN set -eux; \
    MAIN_PKG="$(go list -f '{{if eq .Name "main"}}{{.ImportPath}}{{end}}' ./... | grep -v '^$' | head -n 1)"; \
    echo "MAIN_PKG=$MAIN_PKG"; \
    CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/goapi "$MAIN_PKG"

FROM alpine:3.19
RUN apk add --no-cache ca-certificates
WORKDIR /app
COPY --from=builder /out/goapi /app/goapi
EXPOSE 8080
CMD ["/app/goapi"]
EOF
```

### 10.3 Dockerfile Laravel (menghindari error composer + artisan)

```bash
cat > devops/docker/Dockerfile.laravel <<'EOF'
# syntax=docker/dockerfile:1

FROM php:8.2-cli AS vendor
WORKDIR /app
ENV COMPOSER_ALLOW_SUPERUSER=1

RUN apt-get update \
 && apt-get install -y --no-install-recommends git unzip libzip-dev \
 && docker-php-ext-install zip \
 && rm -rf /var/lib/apt/lists/*

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

COPY laravel/composer.json laravel/composer.lock ./
RUN composer install --no-dev --prefer-dist --no-interaction --no-progress --no-scripts

FROM php:8.2-apache

RUN apt-get update \
 && apt-get install -y --no-install-recommends libzip-dev unzip \
 && docker-php-ext-install pdo pdo_mysql zip \
 && rm -rf /var/lib/apt/lists/* \
 && a2enmod rewrite \
 && echo "ServerName localhost" > /etc/apache2/conf-available/servername.conf \
 && a2enconf servername

WORKDIR /var/www/html
COPY laravel/ ./
COPY --from=vendor /app/vendor ./vendor

RUN sed -i 's|DocumentRoot /var/www/html|DocumentRoot /var/www/html/public|g' /etc/apache2/sites-available/000-default.conf \
 && printf '\n<Directory "/var/www/html/public">\n\tAllowOverride All\n\tRequire all granted\n</Directory>\n' >> /etc/apache2/apache2.conf

RUN chown -R www-data:www-data storage bootstrap/cache || true \
 && chmod -R 775 storage bootstrap/cache || true

EXPOSE 80
CMD ["apache2-foreground"]
EOF
```

### 10.4 Dockerfile Frontend

```bash
cat > devops/docker/Dockerfile.frontend <<'EOF'
# syntax=docker/dockerfile:1
FROM node:20-alpine AS builder
WORKDIR /app

COPY frontend/package*.json ./
RUN if [ -f package-lock.json ]; then npm ci; else npm install; fi

COPY frontend/ ./
RUN npm run build

FROM nginx:alpine
COPY devops/nginx/frontend.conf /etc/nginx/conf.d/default.conf
COPY --from=builder /app/build /usr/share/nginx/html
EXPOSE 80
EOF
```

***

## 11) Fix React supaya tidak hit localhost/127.0.0.1 (biar tidak “Failed to fetch”)

Cek string:

```bash
grep -R "127\.0\.0\.1:8001\|localhost:8080" -n frontend/src || true
```

Replace ke gateway path:

```bash
find frontend/src -type f \( -name "*.js" -o -name "*.jsx" -o -name "*.ts" -o -name "*.tsx" \) -print0 \
| xargs -0 sed -i \
  -e 's|http://127.0.0.1:8001/api/products|/api/laravel/api/products|g' \
  -e 's|http://localhost:8080/api/products|/api/go/api/products|g'
```

***

## 12) Build image → Push ke Harbor → Compose Up

### 12.1 Login Harbor

```bash
docker login 192.168.56.43:8081
```

### BUAT PROJECT DI HARBOR DENGAN NAMA DIBAWAH INI (PRIVATE)

```
threebody
```

### 12.2 Build

```bash
cd ~/three-body-problem-main

docker build -t 192.168.56.43:8081/threebody/goapi:local    -f devops/docker/Dockerfile.goapi .
docker build -t 192.168.56.43:8081/threebody/laravel:local  -f devops/docker/Dockerfile.laravel .
docker build -t 192.168.56.43:8081/threebody/frontend:local -f devops/docker/Dockerfile.frontend .
```

### 12.3 Push

```bash
docker push 192.168.56.43:8081/threebody/goapi:local
docker push 192.168.56.43:8081/threebody/laravel:local
docker push 192.168.56.43:8081/threebody/frontend:local
```

### 12.4 Jalankan compose (gunakan file yang benar!)

```bash
cd ~/three-body-problem-main

docker compose --env-file .env.dev.compose -f docker-compose.dev.yml down --volumes --remove-orphans
docker compose --env-file .env.dev.compose -f docker-compose.dev.yml pull
docker compose --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans
docker compose --env-file .env.dev.compose -f docker-compose.dev.yml ps
```

✅ Harus terlihat: mysql healthy, laravel/goapi/frontend/nginx-dev up.

***

## 13) Laravel migrate (biar table products ada)

```bash
docker exec -it three-body-problem-main-laravel-1 sh -lc 'php artisan migrate --force'
```

***

## 14) Seed data (opsional, untuk demo)

```bash
docker exec -it three-body-problem-main-mysql-1 mysql -u threebody -p'UserPassDev123!' threebody -e "
INSERT INTO products (name, description, price, quantity, category, created_at, updated_at) VALUES
('Laptop Pro 15','High-performance laptop with 16GB RAM and 512GB SSD',1299.99,25,'Electronics',NOW(),NOW()),
('Wireless Headphones','Noise-cancelling wireless headphones with 30h battery life',199.99,50,'Electronics',NOW(),NOW());
"
```

***

## 15) Validasi akhir (harus lolos semua)

### 15.1 Dari VM2 (curl)

```bash
curl -k --resolve dev.local:443:127.0.0.1 https://dev.local/ -I
curl -k --resolve dev.local:443:127.0.0.1 https://dev.local/api/go/api/products
curl -k --resolve dev.local:443:127.0.0.1 https://dev.local/api/laravel/api/products
```

### 15.2 Dari Browser host

* buka `https://dev.local`
* klik **Hit Go API** → success
* klik **Hit Laravel API** → success

***

## 16) “Kumpulan Kendala” yang kamu alami + fix permanennya (ringkas buat README)

1. **invalid reference format** → env variable kosong\
   ✅ Fix: buat `.env.dev.compose` + selalu pakai `--env-file`
2. **Harbor connection refused** → port salah / docker restart\
   ✅ Fix: Harbor di `8081` + setelah restart docker selalu `cd /opt/harbor && sudo docker compose up -d`
3. **Docker pull/push gagal karena HTTPS**\
   ✅ Fix: `/etc/docker/daemon.json` → insecure registries `192.168.56.43:8081`
4. **dev.conf is a directory**\
   ✅ Fix: pastikan `devops/nginx/dev.conf` adalah file (`ls -l` harus `-`)
5. **Go build gagal** (`COPY goapi/`)\
   ✅ Fix: source sebenarnya folder `go/`
6. **Laravel composer lock mismatch**\
   ✅ Fix: vendor stage pakai `php:8.2-cli` + `--no-scripts`
7. **Laravel 500** (APP\_KEY tidak masuk)\
   ✅ Fix: inject `APP_KEY: ${APP_KEY}` di compose + recreate container
8. **Products table doesn’t exist**\
   ✅ Fix: `php artisan migrate --force`
9. **React Failed to fetch** (hit localhost/127.0.0.1 + mixed content)\
   ✅ Fix: frontend pakai path gateway `/api/laravel/api/products` & `/api/go/api/products`
10. **Nginx route tidak match karena slash**\
    ✅ Fix: pakai `location ^~ /api/go` dan rewrite `/?(.*)`

***

### Tambahan: biar kamu tidak keulang error yang barusan

Kalau kamu sampai ketemu lagi:

#### ❌ `docker-compose.dev.yml not found`

Jalankan:

```bash
cd ~/three-body-problem-main
ls -la docker-compose.dev.yml
```

Kalau tidak ada → buat ulang pakai template di Step 6.1.

#### ❌ APP\_KEY masih placeholder

Jalankan:

```bash
cd ~/three-body-problem-main
grep '^APP_KEY=' .env.dev.compose
```

Kalau masih `ISI_RANDOM_NANTI` → buat ulang file `.env.dev.compose` sesuai Step 7.1.

***

Kalau rangkuman ini sudah kamu anggap “final & rapih”, baru kita lanjut ke langkah berikutnya:

**“lanjut buat .gitlab-ci.yml final”** (build/test/push Harbor → deploy dev compose → deploy staging/prod k8s + healthcheck + post-deploy validation + secret manager).
