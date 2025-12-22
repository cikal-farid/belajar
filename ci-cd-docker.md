# CI/CD Docker

### Masuk ke folder Harbor dan edit config

```
cd /opt/harbor
sudo nano harbor.yml
```

ubah port menjadi 8081

### Apply config

```
cd /opt/harbor
sudo docker compose down || true
sudo ./prepare
sudo ./install.sh
sudo docker compose up -d
sudo docker compose ps
```

### Verifikasi Harbor

```
curl -I http://192.168.56.43:8081
curl -s -o /dev/null -w "%{http_code}\n" http://192.168.56.43:8081/v2/
```

## Set Docker “insecure registry” untuk Harbor HTTP (VM2)

```
sudo tee /etc/docker/daemon.json <<EOF
{
  "insecure-registries": ["192.168.56.43:8081"]
}
EOF

sudo systemctl restart docker
```

⚠️ PENTING: setelah restart docker, container Harbor bisa ikut stop.\
Naikkan lagi Harbor:

```
cd /opt/harbor
sudo docker compose up -d
sudo docker compose ps
```

Cek insecure registries:

```
docker info | sed -n '/Insecure Registries/,+10p'
```

✅ Harus ada `192.168.56.43:8081`.

### Buat ulang dengan APP\_KEY asli (cara paling aman)

```
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

```
cat .env.dev.compose
```

✅ APP\_KEY harus sudah “base64:xxxxx” bukan placeholder.

## Siapkan HTTPS dev.local (self‑signed) untuk nginx-dev

```
cd ~/three-body-problem-main
mkdir -p certs/dev

openssl req -newkey rsa:2048 -nodes \
  -keyout certs/dev/dev.key \
  -x509 -days 365 \
  -out certs/dev/dev.crt
# Common Name (CN): dev.local

```

Common Name (e.g. server FQDN or YOUR name) \[]:dev.local

```
dev.local
```

## Siapkan Nginx Gateway config (WAJIB) + pastikan file bukan directory

### Buat folder

```
cd ~/three-body-problem-main
mkdir -p devops/nginx
```

### Buat `devops/nginx/dev.conf`

```
cat > devops/nginx/dev.conf <<'EOF'
# devops/nginx/dev.conf (atau file yang berisi server dev.local)

upstream laravel_upstream { server laravel:80; }
upstream go_upstream      { server goapi:8080; }
upstream fe_upstream      { server frontend:80; }

server {
  listen 80;
  listen 443 ssl;
  server_name dev.local;

  ssl_certificate     /etc/nginx/certs/dev.crt;
  ssl_certificate_key /etc/nginx/certs/dev.key;

  # Frontend
  location / {
    proxy_pass http://fe_upstream;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }

  # React memanggil: /api/laravel/api/products  -> laravel menerima /api/products
  location /api/laravel/ {
    proxy_pass http://laravel_upstream/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }

  # React memanggil: /api/go/api/products -> goapi menerima /api/products
  location /api/go/ {
    proxy_pass http://go_upstream/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
EOF
```

### Frontend nginx config

```
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

### Dockerfile Go

```
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

### Dockerfile Laravel (menghindari error composer + artisan)

```
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

### Dockerfile Frontend

```
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

Cek string:

```
grep -R "127\.0\.0\.1:8001\|localhost:8080" -n frontend/src || true
```

Replace ke gateway path:

```
find frontend/src -type f \( -name "*.js" -o -name "*.jsx" -o -name "*.ts" -o -name "*.tsx" \) -print0 \
| xargs -0 sed -i \
  -e 's|http://127.0.0.1:8001/api/products|/api/laravel/api/products|g' \
  -e 's|http://localhost:8080/api/products|/api/go/api/products|g'
```

## Buat file .gitlab-ci.yml

```
nano .gitlab-ci.yml
```

```
stages:
  - build
  - deploy

variables:
  DOCKER_BUILDKIT: "1"
  COMPOSE_DOCKER_CLI_BUILD: "1"

  COMPOSE_PROJECT_NAME: "threebody-dev"
  DEPLOY_DIR: "/home/gitlab-runner/threebody-deploy"

  # DEV only: kalau root mismatch, auto wipe volume mysql_data (DATA HILANG)
  AUTO_RESET_DB_ON_ROOT_MISMATCH: "1"

default:
  tags: ["deploy"]
  before_script:
    - set -euo pipefail
    - docker version
    - docker compose version

# -------------------------
# BUILD + PUSH (Harbor)
# -------------------------
.harbor_login: &harbor_login |
  set -euo pipefail
  : "${HARBOR_URL:?}" "${HARBOR_PROJECT:?}" "${HARBOR_USERNAME:?}" "${HARBOR_PASSWORD:?}"
  echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin
  TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"

build_goapi:
  stage: build
  script:
    - *harbor_login
    - docker build -t "$HARBOR_URL/$HARBOR_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi .
    - docker push "$HARBOR_URL/$HARBOR_PROJECT/goapi:$TAG"

build_laravel:
  stage: build
  retry: 2
  script:
    - *harbor_login
    - docker build -t "$HARBOR_URL/$HARBOR_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel .
    - docker push "$HARBOR_URL/$HARBOR_PROJECT/laravel:$TAG"

build_frontend:
  stage: build
  script:
    - *harbor_login
    - docker build -t "$HARBOR_URL/$HARBOR_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend .
    - docker push "$HARBOR_URL/$HARBOR_PROJECT/frontend:$TAG"

# -------------------------
# DEPLOY DEV (main only) - self heal + auto reset volume jika root mismatch
# -------------------------
deploy_dev:
  stage: deploy
  needs: ["build_goapi", "build_laravel", "build_frontend"]
  resource_group: dev
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - |
      set -euo pipefail

      : "${DEV_MYSQL_ROOT_PASSWORD:?}" "${DEV_MYSQL_PASSWORD:?}" "${DEV_APP_KEY:?}"
      : "${HARBOR_URL:?}" "${HARBOR_PROJECT:?}" "${HARBOR_USERNAME:?}" "${HARBOR_PASSWORD:?}"

      TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"
      mkdir -p "$DEPLOY_DIR"

      # sync deploy files
      if command -v rsync >/dev/null 2>&1; then
        rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY_DIR/"
      else
        rm -rf "$DEPLOY_DIR/devops" "$DEPLOY_DIR/docker-compose.dev.yml" || true
        cp -a docker-compose.dev.yml devops "$DEPLOY_DIR/"
      fi

      # cert (sekali saja)
      mkdir -p "$DEPLOY_DIR/certs/dev"
      if [ ! -s "$DEPLOY_DIR/certs/dev/dev.crt" ] || [ ! -s "$DEPLOY_DIR/certs/dev/dev.key" ]; then
        docker run --rm -v "$DEPLOY_DIR/certs/dev:/out" alpine:3.19 sh -lc '
          set -e
          apk add --no-cache openssl >/dev/null
          openssl req -newkey rsa:2048 -nodes \
            -keyout /out/dev.key \
            -x509 -days 365 \
            -out /out/dev.crt \
            -subj "/CN=dev.local"
        '
      fi

      # env compose
      umask 077
      {
        echo "MYSQL_ROOT_PASSWORD=${DEV_MYSQL_ROOT_PASSWORD}"
        echo "MYSQL_PASSWORD=${DEV_MYSQL_PASSWORD}"
        echo "MYSQL_USER=threebody"
        echo "MYSQL_DATABASE=threebody"
        echo "APP_KEY=${DEV_APP_KEY}"
        echo "HARBOR_URL=${HARBOR_URL}"
        echo "HARBOR_PROJECT=${HARBOR_PROJECT}"
        echo "CI_COMMIT_SHA=${TAG}"
      } > "$DEPLOY_DIR/.env.dev.compose"

      cd "$DEPLOY_DIR"
      echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin

      dc() { docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml "$@"; }
      dc config >/dev/null

      wait_mysql_root_tcp() {
        dc exec -T mysql sh -lc '
          for i in $(seq 1 90); do
            if MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysqladmin ping -h 127.0.0.1 --protocol=tcp -uroot --silent >/dev/null 2>&1; then
              exit 0
            fi
            sleep 2
          done
          exit 1
        '
      }

      wipe_mysql_volume() {
        echo "AUTO RESET: wipe mysql_data volume (DEV ONLY, DATA HILANG)"
        dc down -v --remove-orphans || true
        # extra safety: hapus volume mysql_data berdasarkan label compose
        for v in $(docker volume ls -q \
          --filter label=com.docker.compose.project="$COMPOSE_PROJECT_NAME" \
          --filter label=com.docker.compose.volume=mysql_data); do
          docker volume rm -f "$v" || true
        done
      }

      ensure_db_user() {
        dc exec -T mysql sh -lc '
          set -eu
          SQL="
            CREATE DATABASE IF NOT EXISTS \`${MYSQL_DATABASE}\`;
            CREATE USER IF NOT EXISTS '\''${MYSQL_USER}'\''@'\''%'\'' IDENTIFIED BY '\''${MYSQL_PASSWORD}'\'';
            ALTER USER '\''${MYSQL_USER}'\''@'\''%'\'' IDENTIFIED BY '\''${MYSQL_PASSWORD}'\'';
            GRANT ALL PRIVILEGES ON \`${MYSQL_DATABASE}\`.* TO '\''${MYSQL_USER}'\''@'\''%'\'' ;
            FLUSH PRIVILEGES;
          "
          MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysql -h 127.0.0.1 --protocol=tcp -uroot -e "$SQL"
        '
      }

      # bersih + pull
      dc down --remove-orphans || true
      dc pull

      # 1) naikkan mysql dulu saja
      dc up -d --remove-orphans mysql

      # 2) tunggu mysql FINAL siap (TCP 3306 + root auth)
      if ! wait_mysql_root_tcp; then
        echo "WARNING: Root MySQL gagal login (kemungkinan volume lama beda root password)."
        if [ "${AUTO_RESET_DB_ON_ROOT_MISMATCH:-1}" = "1" ]; then
          wipe_mysql_volume
          dc up -d --remove-orphans mysql
          wait_mysql_root_tcp || {
            echo "ERROR: Setelah auto reset, root MySQL masih gagal login."
            dc logs --no-color mysql | tail -n 200 || true
            exit 1
          }
        else
          echo "ERROR: AUTO_RESET_DB_ON_ROOT_MISMATCH=0 -> stop."
          dc logs --no-color mysql | tail -n 200 || true
          exit 1
        fi
      fi

      # 3) pastikan db/user app selalu benar (ini yang bikin Laravel tidak Access denied)
      ensure_db_user

      # 4) baru naikkan service lainnya
      dc up -d --remove-orphans

      # 5) restart service app biar pasti pick up koneksi DB
      dc restart laravel goapi || true

      # 6) migrate
      dc exec -T laravel php artisan optimize:clear
      dc exec -T laravel php artisan migrate --force

      dc ps
```

## Buat file docker-compose.dev.yml

```
nano docker-compose.dev.yml
```

```
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_ROOT_HOST: "%"
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - devnet
    healthcheck:
      # TCP: hanya sehat kalau mysqld FINAL sudah listen di 3306 (temporary server port 0 tidak akan lolos)
      test: ["CMD-SHELL", "MYSQL_PWD=$$MYSQL_ROOT_PASSWORD mysqladmin ping -h 127.0.0.1 --protocol=tcp -uroot --silent"]
      interval: 5s
      timeout: 5s
      retries: 40
      start_period: 20s
    restart: unless-stopped

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
      DB_DATABASE: ${MYSQL_DATABASE}
      DB_USERNAME: ${MYSQL_USER}
      DB_PASSWORD: ${MYSQL_PASSWORD}
    networks:
      - devnet
    restart: unless-stopped

  goapi:
    image: ${HARBOR_URL}/${HARBOR_PROJECT}/goapi:${CI_COMMIT_SHA}
    depends_on:
      mysql:
        condition: service_healthy
    environment:
      DB_HOST: mysql
      DB_PORT: 3306
      DB_USER: ${MYSQL_USER}
      DB_PASSWORD: ${MYSQL_PASSWORD}
      DB_NAME: ${MYSQL_DATABASE}
    networks:
      - devnet
    restart: unless-stopped

  frontend:
    image: ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${CI_COMMIT_SHA}
    depends_on:
      - laravel
      - goapi
    networks:
      - devnet
    restart: unless-stopped

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
    restart: unless-stopped

volumes:
  mysql_data:

networks:
  devnet:

```

## Register runner gitlab

```
sudo gitlab-runner register \
  --url https://gitlab.com \
  --token glrt-ArpoA9OGuN_vrA_25AdWUm86MQpwOjE5c3Q1NAp0OjMKdTpocWd0ahg.01.1j1mzr2o7 \
  --executor shell
```

jika belum ada token nya di buat dulu token nya seperti gambar dibawah ini DAN PASTIKAN RUNNER ONLINE

<figure><img src=".gitbook/assets/image (814).png" alt=""><figcaption></figcaption></figure>

saya buat nama

```
DevOps_43_Runner
```

dan tags

```
vm2, shell, docker, deploy
```

## Buat variable juga seperti gambar dibawah ini

<figure><img src=".gitbook/assets/image (815).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (816).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (817).png" alt=""><figcaption></figcaption></figure>

```
DEV_APP_KEY
```

```
base64:LUtk3bmVqbELgudTMT0us8hv5JFGlZP5cVAMQBHAkPM=
```

```
DEV_MYSQL_PASSWORD
```

```
UserPassDev123!
```

```
DEV_MYSQL_ROOT_PASSWORD
```

```
RootPassDev123!
```

```
HARBOR_PASSWORD
```

```
Harbor12345
```

```
HARBOR_PROJECT
```

```
threebody
```

```
HARBOR_URL
```

```
192.168.56.43:8081
```

```
HARBOR_USERNAME
```

```
admin
```

```
MYSQL_PASSWORD
```

```
Harbor12345
```

```
MYSQL_ROOT_PASSWORD
```

```
Harbor12345
```

### Tahap akhir

#### Lalu push ke gitlab

disini opsional jika mengalami kendala harap di cek kembali

```
git config --global user.name "cikalfarid"
git config --global user.email "cikalfarid@gmail.com"

git status
git add .
git commit -m "CI: final stable build+deploy (mysql-first + tcp healthcheck)"
```

Buat SSH

```
ssh-keygen -t ed25519 -C "cikalfarid@gmail.com"
cat ~/.ssh/id_ed25519.pub
```

#### paste ssh pub ke ssh nya gitlab di bagian profile gitlab.com

buat file config untuk gitlab di vm2

```
Host gitlab-443
  HostName altssh.gitlab.com
  User git
  Port 443
```

Verifikasi gitlab

```
ssh -T git@gitlab-443
```

Set Origin

```
git remote set-url origin git@gitlab-443:cikalfarid/three-body-problem.git
```

```
git pull --rebase origin main
git status
```

```
git add .
git commit -am "CI: final stable build+deploy (mysql-first + tcp healthcheck)"
git rebase --continue
git push
```
