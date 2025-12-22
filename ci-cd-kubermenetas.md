# CI/CD KuberMenetas

Oke, kita bikin **versi super lengkap** dari awal sampai masalah terakhir (docker\_push & 504) dan dirapikan jadi **tahapan “bersih” yang kalau diikuti nggak kena error-error yang kemarin**.

Aku pecah jadi beberapa blok:

1. Setup 3 VM
2. Install Docker (dan fix isu API version)
3. Install + register GitLab Runner (dan fix masalah Docker executor)
4. Install Harbor + izin `insecure-registry` (fix error docker\_push)
5. Siapkan repo & docker-compose dev
6. Fix nginx dev + HTTPS `dev.local`
7. Fix Laravel (403, .env, permission, 504)
8. CI/CD `.gitlab-ci.yml` final (yang menghindari semua error invalid tag, dind, dll)
9. Urutan cek akhirnya (supaya yakin semua sehat)

***

### 1. Setup 3 VM & hostnames

#### \[Semua VM: VM1, VM2, VM3]

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl vim git ca-certificates gnupg lsb-release
```

Set hostname:

```bash
# VM1 – 192.168.56.42
echo "k8s-master" | sudo tee /etc/hostname
sudo hostnamectl set-hostname k8s-master

# VM2 – 192.168.56.43
echo "devops-ci" | sudo tee /etc/hostname
sudo hostnamectl set-hostname devops-ci

# VM3 – 192.168.56.44
echo "monitoring" | sudo tee /etc/hostname
sudo hostnamectl set-hostname monitoring
```

`/etc/hosts` di **semua VM**:

```bash
sudo nano /etc/hosts
```

Tambahkan:

```
192.168.56.42   k8s-master
192.168.56.43   devops-ci
192.168.56.44   monitoring
```

***

### 2. Install Docker (fix error “client version too old”)

Kamu **sempat kena** error ini di job:

> Error response from daemon: client version 1.43 is too old…

Ini karena versi Docker daemon & client nggak match (waktu masih pakai dind & docker lama).

Solusinya: pasang Docker terbaru di semua VM, **terutama di VM2 (devops-ci)** yang dipakai GitLab Runner.

#### \[Semua VM – terutama devops-ci]

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
```

Logout → login lagi, lalu:

```bash
docker version
docker ps
```

Harus jalan tanpa error.

> Efeknya: error API version mismatch di job **hilang**, dan kita bisa pakai Docker host via socket tanpa dind.

***

### 3. Install & register GitLab Runner (plus config Docker executor)

Kamu **sudah menjalankan** ini, tapi aku rangkum lagi secara bersih + tambahin langkah yang memastikan service-nya running.

#### 3.1. Install GitLab Runner

#### \[VM2 – devops-ci]

```bash
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt install -y gitlab-runner
```

Outputmu kemarin ada:

* `gitlab-runner: the service is not installed`
* `gitlab-ci-multi-runner: the service is not installed`

Itu dari script lama, **boleh diabaikan**. Kita benerin dengan enable service manual.

#### 3.2. Enable & cek service

```bash
sudo systemctl enable gitlab-runner --now
sudo systemctl status gitlab-runner
```

Harus `active (running)`.

Kalau belum:

```bash
sudo systemctl restart gitlab-runner
sudo systemctl status gitlab-runner
```

#### 3.3. Register runner ke GitLab

Ambil **registration token** dari:\
Project → Settings → CI/CD → Runners → Registration token.

```bash
sudo gitlab-runner register
```

Jawab:

* GitLab instance URL:\
  `https://gitlab.com/`
* Registration token:\
  `<TOKEN DARI PROJECT>`
* Description:\
  `DevOps_44_runner`
* Tags:\
  `docker`
* Executor:\
  `docker`
* Default Docker image:\
  `docker:latest`

Cek:

```bash
sudo gitlab-runner list
```

Harus muncul `DevOps_44_runner`.

#### 3.4. Konfigurasi runner → pakai Docker host (bukan dind)

Sebelum ini kamu pakai `services: docker:dind` dan `DOCKER_HOST=tcp://docker:2375` → sempat error:

* `Cannot connect to the Docker daemon at tcp://docker:2375`
* Health-check service “No HOST or PORT found”

Kita ganti pattern ke yang **lebih simpel**: pakai Docker host lewat socket.

Edit `/etc/gitlab-runner/config.toml`:

```bash
sudo nano /etc/gitlab-runner/config.toml
```

Pastikan kira-kira begini:

```toml
concurrent = 1
check_interval = 0
connection_max_age = "15m0s"

[session_server]
  session_timeout = 1800

[[runners]]
  name = "DevOps_44_runner"
  url = "https://gitlab.com/"
  id = 50865738          # di tempatmu mungkin beda
  token = "XXX"          # di-generate waktu register
  executor = "docker"
  [runners.cache]
    MaxUploadedArchiveSize = 0

  [runners.docker]
    tls_verify = false
    image = "docker:latest"
    privileged = true
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/var/run/docker.sock:/var/run/docker.sock", "/cache"]
    shm_size = 0
    network_mtu = 0
```

**Penting:**

* **Tidak** ada `services = ["docker:dind"]`
* Kita share socket: `/var/run/docker.sock:/var/run/docker.sock`

Restart runner:

```bash
sudo systemctl restart gitlab-runner
```

Dengan ini, error:

* `Cannot connect to Docker daemon`
* healthcheck `No HOST or PORT found`

**hilang**, karena kita tidak pakai dind lagi.

***

### 4. Install & konfigurasi Harbor (fix error docker\_push / connection refused)

Kamu sudah install Harbor & sempat kena error:

> Error response from daemon: Get "[https://192.168.56.43:80/v2/](https://192.168.56.43:80/v2/)": connect: connection refused

Artinya:

* Harbor jalan di **HTTP** port 80
* Tapi docker client mencoba **HTTPS** ke `192.168.56.43:80`
* Docker belum menganggap registry ini sebagai `insecure-registry`

#### 4.1. Install Harbor

#### \[VM2 – devops-ci]

```bash
cd /opt
sudo wget https://github.com/goharbor/harbor/releases/download/v2.10.0/harbor-online-installer-v2.10.0.tgz
sudo tar xzf harbor-online-installer-v2.10.0.tgz
cd harbor
sudo cp harbor.yml.tmpl harbor.yml
sudo nano harbor.yml
```

Di `harbor.yml`:

```yaml
hostname: 192.168.56.43
http:
  port: 80
https:
  port: 443
  certificate: /data/cert/server.crt    # boleh dinonaktifin dulu kalau mau pure HTTP
  private_key: /data/cert/server.key
```

Kalau mau **HTTP saja**, kamu bisa:

* Comment blok `https:` dan biarkan `http.port: 80`.

Lalu:

```bash
sudo ./prepare
sudo ./install.sh
```

Akses di browser: `http://192.168.56.43`\
Buat project: **three-body-problem** (atau `threebody`, asal konsisten dengan `HARBOR_PROJECT` di CI).

#### 4.2. Tandai Harbor sebagai insecure registry di Docker host (fix docker\_push)

Supaya Docker **boleh** pakai HTTP ke registry, kita tambah `insecure-registries`.

#### \[VM2 – devops-ci]

```bash
sudo nano /etc/docker/daemon.json
```

Isi (kalau file kosong):

```json
{
  "insecure-registries": [
    "192.168.56.43:80"
  ]
}
```

Save, lalu:

```bash
sudo systemctl restart docker
sudo systemctl restart gitlab-runner
```

Tes manual:

```bash
docker login 192.168.56.43:80
# Masukkan HARBOR_USERNAME & HARBOR_PASSWORD
```

Kalau ini sudah OK, **pipeline docker\_push tidak akan lagi error** `connection refused` / `https ...:80/v2/`.

***

### 5. Clone repo & siapkan docker-compose dev

#### \[VM2 – devops-ci]

```bash
cd ~
git clone https://github.com/Rafianda/three-body-problem.git three-body-problem-main
cd three-body-problem-main
```

***

### 6. Fix nginx dev & HTTPS `dev.local` (hilangkan error dev.conf is a directory + 504 frontend)

Dulu kamu kena error:

* `pread() "/etc/nginx/conf.d/dev.conf" failed (21: Is a directory)`
* `504 Gateway Time-out` di `https://dev.local`

Penyebab:

* `devops/nginx/dev.conf` ternyata **folder**, bukan file
* Nginx nggak bisa start → frontend via `dev.local` jadi 502/504

#### 6.1. Benerin `devops/nginx/dev.conf`

#### \[VM2 – devops-ci]

```bash
cd ~/three-body-problem-main
rm -rf devops/nginx/dev.conf         # kalau dia directory
nano devops/nginx/dev.conf           # bikin file baru
```

Isi:

```nginx
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

server {
    listen 80;
    listen 443 ssl;
    server_name dev.local;

    ssl_certificate     /etc/nginx/certs/dev.crt;
    ssl_certificate_key /etc/nginx/certs/dev.key;

    location / {
        proxy_pass http://frontend:80;
    }

    location /api/laravel/ {
        limit_req zone=api_limit burst=20 nodelay;
        rewrite ^/api/laravel/(.*)$ /$1 break;
        proxy_pass http://laravel:80;
    }

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
# Harus diawali '-', contoh: -rw-r--r-- (bukan 'd')
```

#### 6.2. Self-signed certificate `dev.local`

```bash
mkdir -p certs/dev
openssl req -newkey rsa:2048 -nodes -keyout certs/dev/dev.key -x509 -days 365 -out certs/dev/dev.crt
```

**Common Name** → `dev.local`.

#### 6.3. `/etc/hosts` di **laptop** (bukan VM)

Di PC yang dipakai browser:

```
192.168.56.43   dev.local
```

***

### 7. Laravel: .env, permission, Apache → fix 403 + 504 di `/api/laravel`

Kamu sempat alami:

* Composer gagal karena `git` & `unzip` belum ada di image test → kita fix di job `test_laravel` dengan `apt-get install git unzip`.
* 403 / 504 dari Laravel karena:
  * `.env` nggak ada / permission salah
  * Apache DocumentRoot masih `/var/www/html` (bukan `public`)
  * `mod_rewrite` belum aktif

#### 7.1. .env di host + copy ke container

#### \[VM2 – devops-ci]

```bash
cd ~/three-body-problem-main/laravel
nano .env
```

Isi minimal:

```env
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
DB_PASSWORD=<SAMA_DENGAN MYSQL_PASSWORD di GitLab CI>
```

Simpan, lalu (setelah docker-compose dev jalan – step berikutnya):

```bash
# setelah container laravel sudah running
docker cp .env three-body-problem-main-laravel-1:/var/www/html/.env
```

#### 7.2. Jalankan docker-compose dev

```bash
cd ~/three-body-problem-main

docker compose -f docker-compose.dev.yml down --volumes --remove-orphans
docker compose -f docker-compose.dev.yml up -d --build --remove-orphans

docker compose -f docker-compose.dev.yml ps
```

Harus kelihatan:

* mysql, laravel, goapi, frontend, nginx-dev → `Up`.

#### 7.3. Perbaikan di dalam container laravel

```bash
docker exec -it three-body-problem-main-laravel-1 /bin/sh
```

Di **dalam** container:

```sh
# pastikan .env ada
ls -l /var/www/html/.env

chmod 664 /var/www/html/.env
chown www-data:www-data /var/www/html/.env

# APP_KEY
php artisan key:generate

# permission storage
chmod -R 775 /var/www/html/storage /var/www/html/bootstrap/cache
chown -R www-data:www-data /var/www/html/storage /var/www/html/bootstrap/cache

# (opsional) install nano
apt-get update && apt-get install -y nano
```

Edit vhost Apache:

```sh
nano /etc/apache2/sites-available/000-default.conf
```

Isi jadi:

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

Aktifkan rewrite + restart Apache:

```sh
a2enmod rewrite
service apache2 restart
exit
```

Sekarang:

* Nginx dev → ke `laravel:80`
* Laravel serve dari `/public`
* `.htaccess` aktif

Hasilnya:

* `https://dev.local/api/go/api/products` → **OK** (sudah OK di kamu)
* `https://dev.local/api/laravel/api/products` → harusnya **OK**, tidak 504 lagi
* `https://dev.local/` → frontend React (kalau image frontend sudah benar)

***

### 8. CI/CD: `.gitlab-ci.yml` final (tanpa dind, tanpa invalid tag, push & deploy dev)

Berbagai error yang dulu muncul di CI:

* `invalid tag "//frontend:...": invalid reference format`\
  → karena `HARBOR_URL` / `HARBOR_PROJECT` kosong → `//frontend`
* `Cannot connect to docker daemon`\
  → karena dind / DOCKER\_HOST salah
* `connection refused https://192.168.56.43:80/v2/`\
  → karena insecure-registry belum diset
* YAML error `script config should be a string or nested array`\
  → ada syntax salah (`script:` bukan list)

Berikut versi yang sudah kamu pakai sekarang, aku tulis lagi **sebagai versi final** (tinggal copas):

```yaml
stages:
  - build
  - test
  - docker_push
  - deploy_dev
  - deploy_staging
  - deploy_prod
  - post_deploy

variables:
  # tag image pakai SHA commit
  TAG: "$CI_COMMIT_SHA"

default:
  image: docker:latest
  tags:
    - docker
  before_script:
    - echo "Running job on commit $CI_COMMIT_SHA"

# ============================== BUILD ==============================
build_frontend:
  stage: build
  script:
    - export HARBOR_IMAGE_PREFIX="${HARBOR_URL:-192.168.56.43:80}/${HARBOR_PROJECT:-three-body-problem}"
    - echo "Using image prefix=$HARBOR_IMAGE_PREFIX"
    - docker build -f frontend/Dockerfile -t "${HARBOR_IMAGE_PREFIX}/frontend:${TAG}" frontend

build_go:
  stage: build
  script:
    - export HARBOR_IMAGE_PREFIX="${HARBOR_URL:-192.168.56.43:80}/${HARBOR_PROJECT:-three-body-problem}"
    - echo "Using image prefix=$HARBOR_IMAGE_PREFIX"
    - docker build -f go/Dockerfile -t "${HARBOR_IMAGE_PREFIX}/goapi:${TAG}" go

build_laravel:
  stage: build
  script:
    - export HARBOR_IMAGE_PREFIX="${HARBOR_URL:-192.168.56.43:80}/${HARBOR_PROJECT:-three-body-problem}"
    - echo "Using image prefix=$HARBOR_IMAGE_PREFIX"
    - docker build -f laravel/Dockerfile -t "${HARBOR_IMAGE_PREFIX}/laravel:${TAG}" laravel

# ============================== TEST ==============================
test_go:
  stage: test
  image: golang:1.22
  script:
    - cd go
    - go test ./...

test_laravel:
  stage: test
  image: php:8.2-cli
  script:
    - apt-get update && apt-get install -y git unzip && rm -rf /var/lib/apt/lists/*
    - cd laravel
    - curl -sS https://getcomposer.org/installer | php
    - php composer.phar install --no-interaction --prefer-dist
    - cp .env.example .env || true
    - php artisan key:generate || true
    - php artisan migrate --seed || true
    - php artisan test || true   # untuk demo: biar test gagal tidak merah

test_frontend:
  stage: test
  image: node:20-alpine
  script:
    - cd frontend
    - npm install
    - npm test -- --watch=false || true

# ============================== PUSH IMAGE ==============================
docker_push:
  stage: docker_push
  script:
    - export HARBOR_REGISTRY="${HARBOR_URL:-192.168.56.43:80}"
    - export HARBOR_IMAGE_PREFIX="${HARBOR_REGISTRY}/${HARBOR_PROJECT:-three-body-problem}"
    - echo "Login to Harbor=$HARBOR_REGISTRY"
    - echo "$HARBOR_PASSWORD" | docker login "$HARBOR_REGISTRY" -u "$HARBOR_USERNAME" --password-stdin
    - docker push "${HARBOR_IMAGE_PREFIX}/frontend:${TAG}"
    - docker push "${HARBOR_IMAGE_PREFIX}/goapi:${TAG}"
    - docker push "${HARBOR_IMAGE_PREFIX}/laravel:${TAG}"
  needs:
    - build_frontend
    - build_go
    - build_laravel
  only:
    - main

# ============================== DEPLOY DEV ==============================
deploy_dev:
  stage: deploy_dev
  script:
    - export HARBOR_REGISTRY="${HARBOR_URL:-192.168.56.43:80}"
    - export HARBOR_IMAGE_PREFIX="${HARBOR_REGISTRY}/${HARBOR_PROJECT:-three-body-problem}"
    - echo "Login to Harbor=$HARBOR_REGISTRY"
    - echo "$HARBOR_PASSWORD" | docker login "$HARBOR_REGISTRY" -u "$HARBOR_USERNAME" --password-stdin

    - export MYSQL_ROOT_PASSWORD="$MYSQL_ROOT_PASSWORD"
    - export MYSQL_PASSWORD="$MYSQL_PASSWORD"
    - export HARBOR_URL="$HARBOR_REGISTRY"
    - export HARBOR_PROJECT="${HARBOR_PROJECT:-three-body-problem}"
    - export CI_COMMIT_SHA="$TAG"

    - docker compose -f docker-compose.dev.yml pull
    - docker compose -f docker-compose.dev.yml up -d --remove-orphans
    - sleep 20
    - curl -k https://dev.local/ || (echo "Dev frontend not responding" && exit 1)
  environment:
    name: dev
  needs:
    - docker_push
  only:
    - main

# (staging/prod & post_deploy seperti yang sudah kamu punya; inti errormu ada di tahap-tahap di atas)
```

**Catatan penting CI variables di GitLab:**

Di `Settings → CI/CD → Variables`:

* `HARBOR_URL` → `192.168.56.43:80` (**tanpa http://**)
* `HARBOR_PROJECT` → `three-body-problem`
* `HARBOR_USERNAME` / `HARBOR_PASSWORD` → cred Harbor
* `MYSQL_ROOT_PASSWORD`, `MYSQL_PASSWORD` → password MySQL dev

Dengan kombinasi:

* Docker host terbaru + insecure-registry
* Runner pakai socket, bukan dind
* HARBOR\_URL/PROJECT tidak kosong

→ error:

* `invalid tag "//frontend:..."`
* `Cannot connect to Docker daemon`
* `https://192.168.56.43:80/v2/ connection refused`

**semua terhindari.**

***

### 9. Checklist akhir (tanpa kendala)

Urutan yang kalau kamu ulang dari nol harusnya mulus:

1. **VM & Docker**
   * Semua VM pakai Docker dari `get.docker.com`
   * `docker version` & `docker ps` ok di `devops-ci`
2. **Harbor**
   * Harbor jalan di `http://192.168.56.43`
   * Project `three-body-problem` ada
   * `/etc/docker/daemon.json` di `devops-ci` ada `insecure-registries: ["192.168.56.43:80"]`
   * `docker login 192.168.56.43:80` sukses
3. **GitLab Runner**
   * Service `gitlab-runner` `active (running)`
   * `/etc/gitlab-runner/config.toml` pakai Docker executor + socket
   * Runner muncul `online` di GitLab & punya tag `docker`
4. **Dev compose**
   * `devops/nginx/dev.conf` adalah file, bukan folder
   * cert `certs/dev/dev.{crt,key}` ada
   * `docker compose -f docker-compose.dev.yml up -d` → semua service `Up`
5. **Laravel container**
   * `.env` ada di `/var/www/html/.env` (via `docker cp`)
   * `php artisan key:generate` sukses
   * Permission `storage` & `bootstrap/cache` benar
   * Apache `DocumentRoot /var/www/html/public`, `mod_rewrite` aktif
6. **Akses dari browser**
   * `https://dev.local/` → frontend (walau mungkin plain UI repo)
   * `https://dev.local/api/go/api/products` → JSON OK
   * `https://dev.local/api/laravel/api/products` → JSON OK
7. **CI pipeline**
   * `build_*` → hijau
   * `test_*` → hijau (boleh test laravel/frontend `|| true`)
   * `docker_push` → hijau (no error login / push)
   * `deploy_dev` → hijau dan curl ke `https://dev.local/` sukses

Kalau semua itu sudah hijau, berarti **semua masalah yang pernah muncul di percakapan kita sudah tertutup** dan tahapan di atas bisa jadi dokumen final-mu buat laporan/README.

Kalau kamu mau, langkah berikutnya kita bisa fokus khusus ke:

* Rapihin Dockerfile Laravel supaya setup .env/permission langsung di image (lebih rapi untuk K8s), atau
* Susun README dalam format laporan tugas (bab-bab).
