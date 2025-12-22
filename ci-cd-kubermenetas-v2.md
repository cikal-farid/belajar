# CI/CD KuberMenetas V2

Oke, aku rapikan semuanya dari awal, disesuaikan sama requirement tugas dan skenario 3 VM kamu. Anggap ini “versi final” tahapan 0–16-mu yang sudah diperbaiki + dilengkapi (CI/CD, Harbor, K8s, HTTPS, rate‑limit, secret manager, logging, DRC, env dev–staging–prod).

***

### 0. Topologi, Role VM & Environment

**Jaringan**

* Gunakan 2 NIC per VM:
  * **NAT** → buat akses internet (apt, curl, akses GitLab, GitHub, dll).
  * **Host‑only** → network 192.168.56.0/24 untuk komunikasi antar VM.
* Pastikan ketiga VM bisa `ping` satu sama lain di IP 192.168.56.x.

**VM & Peran**

> Nama VM di hypervisor boleh: `DevOps_42`, `DevOps_43`, `DevOps_44`.\
> Hostname OS di dalam Ubuntu kita pakai yang fungsional.

* **VM1**
  * IP (host‑only): `192.168.56.42`
  * VM name: `DevOps_42`
  * Hostname OS: `k8s-master`
  * RAM: **6 GB**
  * Peran:
    * Single‑node Kubernetes cluster (staging & prod → pakai namespace).
    * Menjalankan MySQL untuk staging/prod (di K8s).
* **VM2**
  * IP: `192.168.56.43`
  * Hostname: `devops-ci`
  * RAM: **2 GB**
  * Peran:
    * **GitLab Runner** (CI/CD).
    * Docker + Docker Compose untuk **environment dev**.
    * **Harbor** (registry image).
* **VM3**
  * IP: `192.168.56.44`
  * Hostname: `monitoring`
  * RAM: **2 GB**
  * Peran:
    * Stack **logging & monitoring**: Loki + Promtail + Grafana.
    * Dipakai juga sebagai node referensi untuk DRC demo/dokumentasi.

**Environment Mapping**

* **dev** → Docker Compose di VM2 (`devops-ci`).
* **staging** → namespace `threebody-staging` di K8s (VM1).
* **prod** → namespace `threebody-prod` di K8s (VM1).
* Semua image → **dibuild sekali di CI**, dipush ke Harbor, lalu dipakai di dev/staging/prod (prod **ambil image dari dev/CI**, sesuai requirement).

***

### 1. Setup Dasar (Semua VM)

Di **VM1, VM2, VM3**:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl vim git ca-certificates gnupg lsb-release
```

Set hostname:

```bash
# VM1
echo "k8s-master" | sudo tee /etc/hostname
sudo hostnamectl set-hostname k8s-master

# VM2
echo "devops-ci" | sudo tee /etc/hostname
sudo hostnamectl set-hostname devops-ci

# VM3
echo "monitoring" | sudo tee /etc/hostname
sudo hostnamectl set-hostname monitoring
```

Edit `/etc/hosts` di **semua VM**:

```bash
sudo nano /etc/hosts
```

Tambahkan baris:

```
192.168.56.42   k8s-master
192.168.56.43   devops-ci
192.168.56.44   monitoring
```

***

### 2. Install Docker (Semua VM) + Siapkan untuk Harbor

Di semua VM:

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
sudo systemctl enable docker
```

Logout/login, lalu cek:

```bash
docker ps
```

#### 2.1. Mark Harbor sebagai Insecure Registry (karena pakai HTTP:80)

Supaya semua host bisa push/pull image HTTP ke Harbor:

Di **VM1, VM2, VM3**:

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<EOF
{
  "insecure-registries": [
    "192.168.56.43"
  ]
}
EOF

sudo systemctl restart docker
```

> Nanti `HARBOR_URL` = `192.168.56.43` (tanpa `:80`) supaya konsisten di Docker & K8s.

***

### 3. Install Harbor Registry di VM2 (`devops-ci`)

Di **VM2 – devops-ci**:

```bash
cd /opt
sudo wget https://github.com/goharbor/harbor/releases/download/v2.10.0/harbor-online-installer-v2.10.0.tgz
sudo tar xzf harbor-online-installer-v2.10.0.tgz
cd harbor
sudo cp harbor.yml.tmpl harbor.yml
sudo nano harbor.yml
```

Ubah minimal:

```yaml
hostname: 192.168.56.43

# gunakan http dulu
http:
  port: 80

https:
  enabled: false
```

Lalu:

```bash
sudo ./prepare
sudo ./install.sh
```

Akses dari PC host: `http://192.168.56.43`\
Login default: `admin / Harbor12345` (atau sesuai config).

* Buat **project** baru di Harbor:\
  Nama: `threebody` (private disarankan).

Kita akan pakai format image:

* `192.168.56.43/threebody/frontend:<tag>`
* `192.168.56.43/threebody/goapi:<tag>`
* `192.168.56.43/threebody/laravel:<tag>`

***

### 4. Setup GitLab Runner di VM2 (Executor: shell)

> **Perbaikan penting** dari versi sebelumnya:\
> Pakai **executor `shell`**, supaya job `deploy_dev`, `deploy_staging`, `deploy_prod` bisa langsung menjalankan `docker` & `kubectl` di host VM2 dan container hasil deploy tetap hidup.

#### 4.1. Install GitLab Runner

Di **VM2 – devops-ci**:

```bash
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt install gitlab-runner -y
```

Tambahkan user `gitlab-runner` ke grup docker:

```bash
sudo usermod -aG docker gitlab-runner
sudo systemctl restart gitlab-runner
```

#### 4.2. Register Runner ke GitLab Project

Di GitLab.com → buka project fork kamu → **Settings → CI/CD → Runners** → ambil **registration token**.

Di **VM2**:

```bash
sudo gitlab-runner register
```

Isi:

* `GitLab instance URL:` → `https://gitlab.com/`
* `Registration token:` → token dari project.
* `Description:` → `devops-ci-shell`
* `Tags:` → boleh `shell,docker,k8s` (boleh lebih dari satu).
* `Executor:` → **`shell`**

Cek:

```bash
sudo gitlab-runner status
```

Di GitLab UI, runner `devops-ci-shell` harus muncul dan aktif.

***

### 5. Clone Repo Aplikasi (Dev Env) di VM2

Di **VM2 – devops-ci** (pakai user biasa, bukan root):

```bash
cd ~
git clone https://github.com/Rafianda/three-body-problem.git three-body-problem-main
cd three-body-problem-main
```

Direkomendasikan: nanti di `.gitlab-ci.yml` kita **tidak clone lagi** karena GitLab akan otomatis checkout repo ke `$CI_PROJECT_DIR`.

***

### 6. Docker Compose Dev + Nginx + HTTPS + Rate Limiting

#### 6.1. File `docker-compose.dev.yml` (Root Project di VM2)

Pastikan mirip seperti ini (sesuaikan path jika di repo beda):

```yaml
version: "3.9"

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

  laravel:
    image: ${HARBOR_URL}/${HARBOR_PROJECT}/laravel:${CI_COMMIT_SHA:-local}
    depends_on:
      - mysql
    environment:
      APP_ENV: local
      APP_DEBUG: "true"
      DB_CONNECTION: mysql
      DB_HOST: mysql
      DB_PORT: 3306
      DB_DATABASE: threebody
      DB_USERNAME: threebody
      DB_PASSWORD: ${MYSQL_PASSWORD}
    networks:
      - devnet

  goapi:
    image: ${HARBOR_URL}/${HARBOR_PROJECT}/goapi:${CI_COMMIT_SHA:-local}
    depends_on:
      - mysql
    environment:
      DB_HOST: mysql
      DB_PORT: 3306
      DB_USER: threebody
      DB_PASSWORD: ${MYSQL_PASSWORD}
      DB_NAME: threebody
    networks:
      - devnet

  frontend:
    image: ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${CI_COMMIT_SHA:-local}
    depends_on:
      - laravel
      - goapi
    environment:
      REACT_APP_LARAVEL_URL: https://dev.local/api/laravel
      REACT_APP_GO_URL: https://dev.local/api/go
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
```

> Perhatikan: `CI_COMMIT_SHA:-local` → kalau jalan di CI, dia pakai commit SHA; kalau jalan manual di VM, dia fallback ke `local`.

#### 6.2. Pastikan `devops/nginx/dev.conf` itu **file**, bukan directory

Di **VM2**:

```bash
cd ~/three-body-problem-main

rm -rf devops/nginx/dev.conf         # kalau sebelumnya berupa folder
nano devops/nginx/dev.conf
```

Isi (rate‑limit + HTTPS dev):

```nginx
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

server {
    listen 80;
    listen 443 ssl;
    server_name dev.local;

    ssl_certificate     /etc/nginx/certs/dev.crt;
    ssl_certificate_key /etc/nginx/certs/dev.key;

    # Frontend React
    location / {
        proxy_pass http://frontend:80;
    }

    # Laravel API
    location /api/laravel/ {
        limit_req zone=api_limit burst=20 nodelay;
        rewrite ^/api/laravel/(.*)$ /$1 break;
        proxy_pass http://laravel:80;
    }

    # Go API
    location /api/go/ {
        limit_req zone=api_limit burst=20 nodelay;
        rewrite ^/api/go/(.*)$ /$1 break;
        proxy_pass http://goapi:8080;
    }
}
```

Cek:

```bash
ls -l devops/nginx/dev.conf
# Harus keluar: '-rw-r--r--' di kolom paling kiri (bukan 'd')
```

#### 6.3. Self‑Signed Certificate untuk `dev.local`

Di **VM2**:

```bash
cd ~/three-body-problem-main

mkdir -p certs/dev
openssl req -newkey rsa:2048 -nodes -keyout certs/dev/dev.key -x509 -days 365 -out certs/dev/dev.crt
```

Saat prompt:

* `Common Name` → isi: **`dev.local`**

Di **PC host** (bukan di VM):

Edit `C:\Windows\System32\drivers\etc\hosts` (Windows) atau `/etc/hosts` (Linux/Mac):

```
192.168.56.43   dev.local
```

***

### 7. Jalankan Environment Dev (Manual Check)

Di **VM2**:

```bash
cd ~/three-body-problem-main

docker compose -f docker-compose.dev.yml down --volumes --remove-orphans
docker compose -f docker-compose.dev.yml up -d --build --remove-orphans

docker compose -f docker-compose.dev.yml ps
```

Harus terlihat:

* `mysql`, `laravel`, `goapi`, `frontend`, `nginx-dev` → **Up**.

Cek log nginx:

```bash
docker logs three-body-problem-main-nginx-dev-1
# Tidak boleh ada error "dev.conf is a directory"
```

Di browser (PC host):

* `https://dev.local` → frontend React.
* `https://dev.local/api/laravel/...` → API Laravel.
* `https://dev.local/api/go/...` → API Go.

Kalau 3 ini OK → environment dev sudah sehat.

***

### 8. Laravel di Container: .env, Permission, Apache (Kalau belum dirapikan di Dockerfile)

> Idealnya semua ini kamu **pindahkan ke Dockerfile Laravel** dan commit ke repo. Tapi untuk cepat, bisa manual dulu seperti di bawah.

#### 8.1. Buat `.env` Laravel di host lalu copy ke container

Di **VM2**:

```bash
cd ~/three-body-problem-main

touch .env
nano .env
```

Isi minimal:

```dotenv
APP_NAME=Laravel
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=https://dev.local/api/laravel

LOG_CHANNEL=stack

DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=threebody
DB_USERNAME=threebody
DB_PASSWORD=<ISI SAMA DENGAN MYSQL_PASSWORD>
```

Copy ke container Laravel:

```bash
docker cp .env three-body-problem-main-laravel-1:/var/www/html/.env
```

#### 8.2. Permission + APP\_KEY

```bash
docker exec -it three-body-problem-main-laravel-1 /bin/sh
```

Di dalam container:

```sh
ls -l /var/www/html/.env

chmod 664 /var/www/html/.env
chown www-data:www-data /var/www/html/.env

php artisan key:generate
# Application key set successfully.

chmod -R 775 /var/www/html/storage /var/www/html/bootstrap/cache
chown -R www-data:www-data /var/www/html/storage /var/www/html/bootstrap/cache
```

#### 8.3. Fix Apache VirtualHost (Laravel 403 → point ke `public`)

Masih di dalam container:

```sh
apt-get update && apt-get install -y nano
nano /etc/apache2/sites-available/000-default.conf
```

Ubah jadi:

```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html/public

    <Directory /var/www/html/public>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Aktifkan mod\_rewrite & restart Apache:

```sh
a2enmod rewrite
service apache2 restart
exit
```

Sekarang **403 Forbidden** harus hilang.

> Next level (disarankan): masukkan konfigurasi ini ke Dockerfile Laravel supaya tidak perlu edit manual di tiap container.

***

### 9. Secret Manager: GitLab CI Variables + K8s Secrets

#### 9.1. GitLab CI Variables (untuk dev build & pipeline)

Di GitLab project → **Settings → CI/CD → Variables**:

Tambahkan (Protected/Masked sesuai kebutuhan):

* `MYSQL_ROOT_PASSWORD`
* `MYSQL_PASSWORD`
* `HARBOR_URL` → `192.168.56.43`
* `HARBOR_PROJECT` → `threebody`
* `HARBOR_USERNAME` → user Login Harbor
* `HARBOR_PASSWORD` → password Harbor
* Opsional:
  * `APP_KEY` (kalau mau diinject via env).
  * `KUBE_CONFIG` → isi file `admin.conf` dari K8s (base64 atau raw, nanti kita pakai di CI).

#### 9.2. K8s Secret (untuk staging & prod DB / APP)

Di **VM1 – k8s-master**:

```bash
kubectl create namespace threebody-staging
kubectl create namespace threebody-prod
```

Contoh buat DB secret untuk staging:

```bash
kubectl create secret generic db-credentials \
  --from-literal=MYSQL_ROOT_PASSWORD=<isi> \
  --from-literal=MYSQL_PASSWORD=<isi> \
  --from-literal=DB_USER=threebody \
  --from-literal=DB_NAME=threebody \
  -n threebody-staging
```

Copy untuk prod:

```bash
kubectl create secret generic db-credentials \
  --from-literal=MYSQL_ROOT_PASSWORD=<isi> \
  --from-literal=MYSQL_PASSWORD=<isi> \
  --from-literal=DB_USER=threebody \
  --from-literal=DB_NAME=threebody \
  -n threebody-prod
```

***

### 10. GitLab CI/CD: Build → Test → Push → Deploy Dev → Deploy Staging/Prod → Health Check

> Kita pakai **runner shell di VM2** yang sudah punya `docker` dan nanti `kubectl`.\
> Pipeline stages:
>
> * `test`
> * `build`
> * `deploy_dev`
> * `deploy_staging`
> * `deploy_prod`
> * `post_check`

Di root repo (di GitLab), buat file **`.gitlab-ci.yml`** kira‑kira seperti ini (sesuaikan path service):

```yaml
stages:
  - test
  - build
  - deploy_dev
  - deploy_staging
  - deploy_prod
  - post_check

variables:
  IMAGE_TAG: "$CI_COMMIT_SHA"

# ====== TEST STAGE ======

test_go:
  stage: test
  script:
    - echo "Test Go API"
    - docker run --rm -v "$CI_PROJECT_DIR/goapi":/app -w /app golang:1.22-alpine go test ./...
  rules:
    - if: '$CI_COMMIT_BRANCH'

test_frontend:
  stage: test
  script:
    - echo "Test React (kalau ada test)"
    - docker run --rm -v "$CI_PROJECT_DIR/frontend":/app -w /app node:20-alpine sh -c "npm install && npm test || echo 'skip test'"
  rules:
    - if: '$CI_COMMIT_BRANCH'

test_laravel:
  stage: test
  script:
    - echo "Test Laravel (phpunit kalau ada)"
    - docker run --rm -v "$CI_PROJECT_DIR/laravel":/app -w /app php:8.2-cli sh -c "php -v; php artisan test || echo 'skip test'"
  rules:
    - if: '$CI_COMMIT_BRANCH'

# ====== BUILD & PUSH IMAGES ======

build_and_push:
  stage: build
  needs:
    - test_go
    - test_frontend
    - test_laravel
  script:
    - echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin

    # build multistage images (Dockerfile path silakan sesuaikan)
    - docker build -t "$HARBOR_URL/$HARBOR_PROJECT/frontend:$IMAGE_TAG"  -f devops/docker/Dockerfile.frontend .
    - docker build -t "$HARBOR_URL/$HARBOR_PROJECT/goapi:$IMAGE_TAG"      -f devops/docker/Dockerfile.goapi .
    - docker build -t "$HARBOR_URL/$HARBOR_PROJECT/laravel:$IMAGE_TAG"    -f devops/docker/Dockerfile.laravel .

    # push ke Harbor
    - docker push "$HARBOR_URL/$HARBOR_PROJECT/frontend:$IMAGE_TAG"
    - docker push "$HARBOR_URL/$HARBOR_PROJECT/goapi:$IMAGE_TAG"
    - docker push "$HARBOR_URL/$HARBOR_PROJECT/laravel:$IMAGE_TAG"
  only:
    - main

# ====== DEPLOY DEV (Docker Compose di VM2) ======

deploy_dev:
  stage: deploy_dev
  needs:
    - build_and_push
  script:
    - echo "Deploy dev (Docker Compose)"
    - cd "$CI_PROJECT_DIR"
    # variabel untuk docker compose
    - export HARBOR_URL="$HARBOR_URL"
    - export HARBOR_PROJECT="$HARBOR_PROJECT"
    - export CI_COMMIT_SHA="$IMAGE_TAG"
    - export MYSQL_ROOT_PASSWORD="$MYSQL_ROOT_PASSWORD"
    - export MYSQL_PASSWORD="$MYSQL_PASSWORD"

    - docker compose -f docker-compose.dev.yml pull
    - docker compose -f docker-compose.dev.yml up -d --remove-orphans
    - docker compose -f docker-compose.dev.yml ps

    # health check simple
    - echo "Health check https://localhost (nginx-dev)"
    - sleep 10
    - curl -k --retry 5 --retry-delay 5 --fail https://localhost || (docker compose -f docker-compose.dev.yml logs nginx-dev && exit 1)
  environment:
    name: dev
    url: https://dev.local
  only:
    - main

# ====== DEPLOY STAGING (K8s, namespace threebody-staging) ======

deploy_staging:
  stage: deploy_staging
  needs:
    - build_and_push
  script:
    - echo "Deploy ke Kubernetes STAGING"
    - export KUBECONFIG="$KUBECONFIG_PATH"
    # apply manifest pertama kali
    - kubectl apply -f k8s/staging/ -n threebody-staging || true

    # rolling update image ke tag baru
    - kubectl -n threebody-staging set image deployment/frontend frontend="$HARBOR_URL/$HARBOR_PROJECT/frontend:$IMAGE_TAG" --record=true
    - kubectl -n threebody-staging set image deployment/goapi goapi="$HARBOR_URL/$HARBOR_PROJECT/goapi:$IMAGE_TAG" --record=true
    - kubectl -n threebody-staging set image deployment/laravel laravel="$HARBOR_URL/$HARBOR_PROJECT/laravel:$IMAGE_TAG" --record=true

    # tunggu rollout
    - kubectl -n threebody-staging rollout status deployment/frontend --timeout=180s
    - kubectl -n threebody-staging rollout status deployment/goapi --timeout=180s
    - kubectl -n threebody-staging rollout status deployment/laravel --timeout=180s
  environment:
    name: staging
    url: https://staging.threebody.local
  when: manual
  only:
    - main

# ====== DEPLOY PROD (K8s, namespace threebody-prod) ======

deploy_prod:
  stage: deploy_prod
  needs:
    - build_and_push
  script:
    - echo "Deploy ke Kubernetes PROD"
    - export KUBECONFIG="$KUBECONFIG_PATH"
    - kubectl apply -f k8s/prod/ -n threebody-prod || true

    - kubectl -n threebody-prod set image deployment/frontend frontend="$HARBOR_URL/$HARBOR_PROJECT/frontend:$IMAGE_TAG" --record=true
    - kubectl -n threebody-prod set image deployment/goapi goapi="$HARBOR_URL/$HARBOR_PROJECT/goapi:$IMAGE_TAG" --record=true
    - kubectl -n threebody-prod set image deployment/laravel laravel="$HARBOR_URL/$HARBOR_PROJECT/laravel:$IMAGE_TAG" --record=true

    - kubectl -n threebody-prod rollout status deployment/frontend --timeout=180s
    - kubectl -n threebody-prod rollout status deployment/goapi --timeout=180s
    - kubectl -n threebody-prod rollout status deployment/laravel --timeout=180s
  environment:
    name: prod
    url: https://prod.threebody.local
  when: manual
  only:
    - tags

# ====== POST DEPLOYMENT CHECK PROD ======

post_check_prod:
  stage: post_check
  needs:
    - deploy_prod
  script:
    - export KUBECONFIG="$KUBECONFIG_PATH"
    - kubectl -n threebody-prod get pods -o wide
    - kubectl -n threebody-prod get ingress
    - echo "Health check endpoint prod"
    - curl -k --retry 5 --retry-delay 5 --fail https://prod.threebody.local/healthz || exit 1
  when: manual
  environment:
    name: prod
```

**Catatan penting:**

* `KUBECONFIG_PATH` → kamu isi di GitLab Variables, misal `/home/gitlab-runner/.kube/config`.
* Pastikan di VM2 file tersebut ada & berisi config dari cluster di VM1 (lihat langkah K8s di bawah).

***

### 11. Multi‑Stage Dockerfile (Custom Image di Harbor)

Minimal kamu punya 3 Dockerfile multistage (lokasi bisa di `devops/docker/`):

#### 11.1. Go API – `devops/docker/Dockerfile.goapi`

```dockerfile
# Builder
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY goapi/ ./
RUN go mod download
RUN go build -o goapi .

# Runner
FROM alpine:3.19
WORKDIR /app
COPY --from=builder /app/goapi .
EXPOSE 8080
CMD ["./goapi"]
```

#### 11.2. Frontend React – `devops/docker/Dockerfile.frontend`

```dockerfile
# Builder
FROM node:20-alpine AS builder
WORKDIR /app
COPY frontend/package*.json ./
RUN npm install
COPY frontend/ ./
RUN npm run build

# Runner (Nginx)
FROM nginx:alpine
COPY devops/nginx/frontend.conf /etc/nginx/conf.d/default.conf
COPY --from=builder /app/build /usr/share/nginx/html
EXPOSE 80
```

`devops/nginx/frontend.conf` bisa super simple (proxy asset statis).

#### 11.3. Laravel – `devops/docker/Dockerfile.laravel`

```dockerfile
# Builder: install composer deps
FROM composer:2 AS vendor
WORKDIR /app
COPY laravel/composer.json laravel/composer.lock ./
RUN composer install --no-dev --prefer-dist --no-interaction --no-progress

# Runtime: PHP + Apache
FROM php:8.2-apache

# enable extension + rewrite
RUN docker-php-ext-install pdo pdo_mysql
RUN a2enmod rewrite

WORKDIR /var/www/html

# copy source
COPY laravel/ ./
COPY --from=vendor /app/vendor ./vendor

# set docroot ke public
RUN sed -i 's|DocumentRoot /var/www/html|DocumentRoot /var/www/html/public|g' /etc/apache2/sites-available/000-default.conf \
 && printf '\n<Directory "/var/www/html/public">\n\tAllowOverride All\n\tRequire all granted\n</Directory>\n' >> /etc/apache2/apache2.conf

# permission
RUN chown -R www-data:www-data storage bootstrap/cache \
 && chmod -R 775 storage bootstrap/cache

EXPOSE 80
CMD ["apache2-foreground"]
```

> Dengan ini, sebagian besar step manual di bagian 8 sebenarnya bisa dihapus (cukup injek `.env` via mount / secret / env var).

***

### 12. Setup Kubernetes di VM1 (`k8s-master`)

#### 12.1. Disable Swap & Install Kube Tools

Di **VM1 – k8s-master**:

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

#### 12.2. Init Cluster

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

Set kubeconfig untuk user biasa:

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Install CNI (Flannel):

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

Hilangkan taint (supaya control-plane bisa jalanin workload):

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

#### 12.3. Bikin Namespace & Secret Harbor

Sudah kita buat namespace di langkah 9.2. Tambah secret image pull:

```bash
kubectl create secret docker-registry harbor-creds \
  --docker-server=192.168.56.43 \
  --docker-username=<harbor-username> \
  --docker-password=<harbor-password> \
  --docker-email=<email> \
  -n threebody-staging

kubectl create secret docker-registry harbor-creds \
  --docker-server=192.168.56.43 \
  --docker-username=<harbor-username> \
  --docker-password=<harbor-password> \
  --docker-email=<email> \
  -n threebody-prod
```

#### 12.4. Copy kubeconfig ke VM2 (untuk CI)

Di **VM1**:

```bash
sudo cat /etc/kubernetes/admin.conf
```

Copy isi ke **VM2**:

```bash
sudo mkdir -p /home/gitlab-runner/.kube
sudo nano /home/gitlab-runner/.kube/config
# paste isi admin.conf
sudo chown -R gitlab-runner:gitlab-runner /home/gitlab-runner/.kube
```

Di GitLab Variables:

* `KUBECONFIG_PATH` → `/home/gitlab-runner/.kube/config`

***

### 13. Manifest K8s Staging & Prod (Deployment + Service + Ingress HTTPS + Rate Limit)

Struktur direktori di repo:

```
k8s/
  staging/
    mysql.yaml
    laravel.yaml
    goapi.yaml
    frontend.yaml
    ingress.yaml
  prod/
    mysql.yaml
    laravel.yaml
    goapi.yaml
    frontend.yaml
    ingress.yaml
```

#### 13.1. Contoh `laravel.yaml` (staging)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel
  namespace: threebody-staging
spec:
  replicas: 1
  selector:
    matchLabels:
      app: laravel
  template:
    metadata:
      labels:
        app: laravel
    spec:
      imagePullSecrets:
        - name: harbor-creds
      containers:
        - name: laravel
          image: 192.168.56.43/threebody/laravel:${IMAGE_TAG} # di CI akan di-set via set image
          ports:
            - containerPort: 80
          env:
            - name: APP_ENV
              value: "staging"
            - name: DB_HOST
              value: "mysql"
            - name: DB_PORT
              value: "3306"
            - name: DB_CONNECTION
              value: "mysql"
            - name: DB_DATABASE
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: DB_NAME
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: DB_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: MYSQL_PASSWORD
          readinessProbe:
            httpGet:
              path: /health
              port: 80
            initialDelaySeconds: 20
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 80
            initialDelaySeconds: 30
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: laravel
  namespace: threebody-staging
spec:
  selector:
    app: laravel
  ports:
    - port: 80
      targetPort: 80
```

> `path: /health` → buat route health sederhana di Laravel.

`goapi`, `frontend`, dan `mysql` mirip (service type `ClusterIP`).

#### 13.2. Ingress Controller + Ingress Resource

Install **Nginx Ingress Controller** di VM1:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/baremetal/deploy.yaml
```

Contoh `k8s/staging/ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: threebody-staging-ingress
  namespace: threebody-staging
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "2"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - staging.threebody.local
      secretName: staging-tls
  rules:
    - host: staging.threebody.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
          - path: /api/laravel
            pathType: Prefix
            backend:
              service:
                name: laravel
                port:
                  number: 80
          - path: /api/go
            pathType: Prefix
            backend:
              service:
                name: goapi
                port:
                  number: 8080
```

Buat TLS secret staging (self‑signed) di VM1:

```bash
openssl req -newkey rsa:2048 -nodes -keyout staging.key -x509 -days 365 -out staging.crt
# CN isi: staging.threebody.local

kubectl create secret tls staging-tls \
  --cert=staging.crt \
  --key=staging.key \
  -n threebody-staging
```

Di PC host, tambahkan ke `/etc/hosts`:

```
192.168.56.42   staging.threebody.local
192.168.56.42   prod.threebody.local
```

Ingress prod sama, hanya beda namespace, secret name (`prod-tls`), dan host (`prod.threebody.local`).

***

### 14. Logging & Monitoring di VM3 (`monitoring`)

#### 14.1. Stack Loki + Grafana (Docker Compose)

Di **VM3**:

```bash
mkdir -p ~/logging-stack
cd ~/logging-stack
nano docker-compose.yml
```

Isi contoh singkat:

```yaml
version: "3.9"

services:
  loki:
    image: grafana/loki:2.9.0
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - ./loki-config.yml:/etc/loki/local-config.yaml

  grafana:
    image: grafana/grafana:10.0.0
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    depends_on:
      - loki
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  grafana-data:
```

`loki-config.yml` bisa pakai config default dari docs Loki (boleh minimal dulu).

Jalankan:

```bash
docker compose up -d
```

Akses Grafana: `http://192.168.56.44:3000` → tambah **Loki** sebagai data source.

#### 14.2. Promtail di VM1 & VM2

Untuk mengirim log container **nginx-dev, laravel, goapi, frontend** dan log pod K8s:

*   Install Promtail di VM1 dan VM2 (pakai docker atau binary). Contoh docker di VM2:

    ```bash
    mkdir -p ~/promtail
    cd ~/promtail
    nano promtail-config.yml
    ```

    Isi utama:

    ```yaml
    server:
      http_listen_port: 9080
      grpc_listen_port: 0

    positions:
      filename: /tmp/positions.yaml

    clients:
      - url: http://192.168.56.44:3100/loki/api/v1/push

    scrape_configs:
      - job_name: docker
        static_configs:
          - targets:
              - localhost
            labels:
              job: docker-logs
              __path__: /var/lib/docker/containers/*/*-json.log
    ```

    Jalankan:

    ```bash
    docker run -d \
      --name=promtail \
      -v /var/log:/var/log \
      -v /var/lib/docker/containers:/var/lib/docker/containers \
      -v $(pwd)/promtail-config.yml:/etc/promtail/config.yml \
      grafana/promtail:2.9.0 \
        -config.file=/etc/promtail/config.yml
    ```
* Setup serupa di VM1 (K8s) → log pod via file log container.

Di Grafana:

* Buat dashboard:
  * Nginx (filter label `job = docker-logs` dan `container_name =~ "nginx|nginx-dev"`).
  * Laravel, Go API, frontend, pod K8s (filter label lain).

***

### 15. DRC (Disaster Recovery Concept) – untuk Dokumentasi Sistem Design

Di **README.md** atau dokumen terpisah (`docs/system-design-drc.md`), tulis hal-hal berikut:

1. **Arsitektur Tingkat Tinggi**
   * Diagram:
     * GitLab.com → Runner (VM2) → Harbor (VM2).
     * Dev (docker-compose di VM2).
     * Staging/Prod (K8s di VM1).
     * Monitoring/logging (VM3).
   * Sebutkan dependency utama:
     * Registry Harbor.
     * MySQL (dev di Docker, staging/prod di K8s).
     * K8s control plane.
2. **Infra as Code**
   * Semua:
     * `Dockerfile` (multistage).
     * `docker-compose.dev.yml`.
     * `devops/nginx/*`.
     * Manifest Kubernetes (`k8s/staging`, `k8s/prod`).
     * `.gitlab-ci.yml`.
   * Disimpan di Git: memudahkan re-provision di DRC site.
3. **Backup & Restore**
   * **Database**:
     * Jadwal `mysqldump` (cron job) dari MySQL (di K8s dan dev).
     * Simpan hasil backup di storage terpisah (manual: folder di VM3 atau external disk).
   * **Harbor**:
     * Backup database Harbor + folder data (volume `harbor`).
   * **Config K8s**:
     * Simpan `admin.conf`, manifest `k8s/*` di repo.
   * **Restore Scenario**:
     * Jika VM1 down:
       * Bring up node baru (VM baru).
       * Install Docker + Kubeadm.
       * Re-init cluster atau join node baru + restore etcd (kalau diset).
       * Restore MySQL dari dump.
       * Jalankan `kubectl apply -f k8s/prod`.
     * Jika VM2 down:
       * Install Docker + GitLab Runner + Harbor di VM lain.
       * Restore data Harbor.
       * Runner re-register ke GitLab.
     * Jika VM3 down:
       * Deploy ulang stack logging; tidak mengganggu jalannya aplikasi (loss log sementara masih ok dalam scope tes).
4. **Security & Secret**
   * Gunakan GitLab Variables (encrypted di GitLab).
   * Gunakan K8s Secret untuk DB credential.
   * Jangan commit password dan APP\_KEY ke repo.
5. **RTO & RPO (secara konseptual)**
   * RTO: berapa lama target pemulihan service (misal 1–2 jam untuk lab).
   * RPO: seberapa banyak data yang boleh hilang (misal ≤ 1 jam berdasarkan interval backup).

***

### 16. Ringkas Alur Akhir (Checklist)

1. **VM & Network** siap (NAT + 192.168.56.0/24, 3 VM).
2. Base **Ubuntu + Docker** jalan di semua VM.
3. **Harbor** running di VM2 → project `threebody`.
4. **GitLab Runner (shell)** di VM2, terhubung ke GitLab.com.
5. Repo `three-body-problem` sudah punya:
   * `Dockerfile` multistage tiap service.
   * `docker-compose.dev.yml`.
   * `devops/nginx/dev.conf` (rate‑limit + HTTPS).
   * `k8s/staging/*` dan `k8s/prod/*`.
   * `.gitlab-ci.yml` seperti di atas.
6. Environment **dev** (Docker Compose di VM2) bisa di‑curl:
   * `https://dev.local`
   * `https://dev.local/api/laravel`
   * `https://dev.local/api/go`
7. Environment **staging/prod** (K8s di VM1) jalan:
   * `https://staging.threebody.local`
   * `https://prod.threebody.local`
   * Health check jalan via CI.
8. **Logging & monitoring** aktif di VM3 (Grafana+Loki) dan Promtail mengirim log dari VM1 & VM2.
9. Dokumentasi **system design + DRC** sudah tertulis di README / docs.

Kalau kamu mau, langkah berikutnya bisa kamu lakukan:

* Commit versi final `Dockerfile`, `docker-compose.dev.yml`, `devops/nginx/dev.conf`, `k8s/*`, `.gitlab-ci.yml`, dan screenshot:
  * Harbor project & image.
  * GitLab pipeline sukses (build, deploy\_dev, deploy\_staging/prod).
  * Tampilan Grafana.
  * Aplikasi jalan di dev/staging/prod.

Tinggal kamu adapt sedikit path direktori sesuai isi repo kamu, tapi alur dan tahapan di atas sudah disesuaikan supaya **tidak tabrakan** dengan requirement, dan meminimalisir error yang sebelumnya kamu alami (dev.conf directory, .env Laravel, permission, 403, deployment yang tidak persistent, dsb).
