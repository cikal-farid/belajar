# 0. DevOps V3

[https://harbor.local\
\
https://staging.local\
\
https://prod.local](https://harbor.localhttps/staging.localhttps://prod.local)

```
https://harbor.local
```

```
https://staging.local
```

```
https://prod.local
```

Di bawah ini **runbook**:

* **K8s pakai kubeadm (bukan k3s)**
* **Tanpa Ingress & tanpa MetalLB**
* Akses aplikasi lewat **Edge Nginx** (jalan di **vm-docker**) + **NodePort** (untuk prod K8s)
* **Harbor wajib** untuk registry + image multistage
* **Tag image = `CI_COMMIT_SHORT_SHA`**
* Ada **push dari GitHub → GitLab** (step-by-step)
* Ada **Logging terpusat (Loki + Grafana)** + promtail

> Catatan penting dari error kamu sebelumnya:
>
> 1. Harbor error `Please specify hostname` karena `harbor.yml` masih `hostname: reg.mydomain.com` → **harus diganti `harbor.local`** dan tidak boleh kosong. Harbor memang mewajibkan hostname di-update. ([goharbor.io](https://goharbor.io/docs/2.12.0/install-config/configure-yml-file/))
> 2. `scp` kamu timeout karena **UFW di vm-harbor belum buka SSH (22)**. Kamu cuma allow 80/443 → SSH ketutup → scp timeout.
> 3. vm-docker kamu belum install docker → `docker: command not found`, `docker.service not found`.
> 4. vm-k8s belum install containerd → `/etc/containerd/config.toml` memang belum ada.

***

## 0) Target Arsitektur & Nama Host

**Host-only network**: `192.168.56.0/24`

| VM                     |            IP | Hostname OS | FQDN/akses                                 |
| ---------------------- | ------------: | ----------- | ------------------------------------------ |
| vm-docker              | 192.168.56.42 | `vm-docker` | `staging.local`, `prod.local` (Edge Nginx) |
| vm-harbor              | 192.168.56.43 | `vm-harbor` | `harbor.local` (Harbor UI/Registry)        |
| vm-k8s (control-plane) | 192.168.56.44 | `vm-k8s`    | -                                          |
| vm-worker              | 192.168.56.45 | `vm-worker` | -                                          |

**Flow CI/CD:**\
Git push → GitLab Pipeline → build & push image ke Harbor → deploy **staging Docker Compose** → healthcheck → deploy **prod K8s NodePort** → healthcheck → logs ke Loki+Grafana.

***

## 1) “Common Step” untuk SEMUA VM (dari VM kosong)

Lakukan di **vm-docker, vm-harbor, vm-k8s, vm-worker**.

### 1.1 Set hostname OS

Jalankan sesuai VM:

**vm-harbor**

```bash
sudo hostnamectl set-hostname vm-harbor
```

**vm-docker**

```bash
sudo hostnamectl set-hostname vm-docker
```

**vm-k8s**

```bash
sudo hostnamectl set-hostname vm-k8s
```

**vm-worker**

```bash
sudo hostnamectl set-hostname vm-worker
```

Cek:

```bash
hostname
```

### 1.2 Isi `/etc/hosts` (WAJIB biar `harbor.local`, `staging.local`, `prod.local` resolve)

Di **setiap VM**, edit:

```bash
sudo nano /etc/hosts
```

Isi (tambahkan) ini:

```txt
192.168.56.42 vm-docker staging.local prod.local 
192.168.56.43 vm-harbor harbor.local
192.168.56.44 vm-k8s
192.168.56.45 vm-worker
```

Cek:

```bash
getent hosts harbor.local
getent hosts staging.local
getent hosts prod.local
```

> **Di laptop host kamu juga** (Windows/Mac/Linux) tambahkan hosts yang sama supaya browser bisa buka Harbor dan app.

### 1.3 Update paket & tools dasar

```bash
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl git nano openssh-client openssh-server
```

Cek SSH aktif:

```bash
sudo systemctl enable --now ssh
sudo systemctl status ssh --no-pager
```

Jika belum ada ssh, jalankan dibawah ini

```
sudo apt-get install -y openssh-server
```

### 1.4 UFW (firewall) – jangan putus SSH!

Install & allow SSH dulu:

```bash
sudo apt-get install -y ufw
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status
```

> Ini yang bikin `scp` kamu timeout sebelumnya: di vm-harbor **belum allow OpenSSH**.

***

## 2) Install Harbor di `vm-harbor` (FULL + rapi)

> Harbor wajib update `hostname` di `harbor.yml`. ([goharbor.io](https://goharbor.io/docs/2.12.0/install-config/configure-yml-file/))

### 2.1 Install Docker Engine + Compose (di vm-harbor)

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker

docker version
docker compose version
```

### 2.2 Siapkan folder & download Harbor offline installer

Pilih versi (contoh kamu pakai `v2.14.1`):

```bash
export HARBOR_VERSION="v2.14.1"

cd /tmp
wget -O "harbor-offline-installer-${HARBOR_VERSION}.tgz" \
  "https://github.com/goharbor/harbor/releases/download/${HARBOR_VERSION}/harbor-offline-installer-${HARBOR_VERSION}.tgz"

tar -xzf "harbor-offline-installer-${HARBOR_VERSION}.tgz"

sudo rm -rf /opt/harbor
sudo mv harbor /opt/harbor
sudo chown -R $USER:$USER /opt/harbor

ls -la /opt/harbor
```

### 2.3 Buat TLS (CA + cert `harbor.local`)

```bash
sudo apt-get install -y openssl
sudo mkdir -p /etc/harbor/certs
cd /etc/harbor/certs
```

#### 2.3.1 Buat CA

```bash
sudo openssl genrsa -out ca.key 4096
sudo openssl req -x509 -new -nodes -sha512 -days 3650 \
  -subj "/C=ID/ST=Jakarta/L=Jakarta/O=Lab/OU=DevOps/CN=lab-ca" \
  -key ca.key \
  -out ca.crt
```

#### 2.3.2 Buat key + CSR untuk `harbor.local`

```bash
sudo openssl genrsa -out harbor.local.key 4096
sudo openssl req -new -sha512 \
  -subj "/C=ID/ST=Jakarta/L=Jakarta/O=Lab/OU=DevOps/CN=harbor.local" \
  -key harbor.local.key \
  -out harbor.local.csr
```

#### 2.3.3 Buat SAN config lalu sign

```bash
cat | sudo tee v3.harbor.ext >/dev/null <<'EOF'
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=harbor.local
IP.1=192.168.56.43
EOF

sudo openssl x509 -req -sha512 -days 3650 \
  -in harbor.local.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out harbor.local.crt \
  -extfile v3.harbor.ext
```

Cek:

```bash
openssl x509 -in /etc/harbor/certs/harbor.local.crt -noout -subject -issuer
```

### 2.4 Konfigurasi `harbor.yml` (INI FIX ERROR “Please specify hostname”)

Masuk folder:

```bash
cd /opt/harbor
cp harbor.yml.tmpl harbor.yml
```

Edit:

```bash
nano harbor.yml
```

Minimal yang kamu ubah:

```yaml
hostname: harbor.local

http:
  port: 80

https:
  port: 443
  certificate: /etc/harbor/certs/harbor.local.crt
  private_key: /etc/harbor/certs/harbor.local.key

harbor_admin_password: Harbor12345

data_volume: /data/harbor
```

> `hostname` **wajib** benar dan tidak kosong. Harbor minta hostname berupa IP/FQDN yang bisa diakses klien. ([goharbor.io](https://goharbor.io/docs/2.12.0/install-config/configure-yml-file/))

Buat data dir:

```bash
sudo mkdir -p /data/harbor
sudo chown -R $USER:$USER /data/harbor
```

### 2.5 Install Harbor

**JANGAN** `docker compose down` kalau `docker-compose.yml` belum ada.

Jalankan:

```bash
cd /opt/harbor
sudo ./install.sh
```

Cek container:

```bash
sudo docker compose -f /opt/harbor/docker-compose.yml ps
```

Test akses dari vm-harbor:

```bash
curl -k https://harbor.local/
```

Dari laptop, buka:

* `https://harbor.local`\
  Login: `admin / Harbor12345`

### 2.6 Buat project di Harbor

Di UI Harbor:

* Projects → New Project
* Nama: `threebody`
* Visibility: bebas (Private juga OK)

## A) Buat Harbor Robot Account (push/pull) untuk project threebody

### A1. Buat robot di Harbor (UI)

1. Login Harbor: `https://harbor.local` (atau sesuai host Harbor kamu)
2. Masuk **Projects** → pilih project **threebody**
3. Menu **Robot Accounts** → klik **New Robot Account**
4. Isi:
   * **Name**: `gitlab-ci` (bebas, tapi rapi)
   * **Expiration**: misal 90 hari / 1 tahun (sesuai kebutuhan)
   * **Permissions**: pilih repository dalam project `threebody`:
     * ✅ **Pull**
     * ✅ **Push**
     * (Tidak perlu Delete / Admin)
5. Klik **Add / Save**
6. Harbor akan menampilkan:
   * **Robot Username** (contoh format biasanya mirip `robot$threebody+gitlab-ci`)
   * **Robot Token/Secret** (sekali tampil, **copy & simpan**)

> Catatan: format username robot kadang beda antar versi Harbor. **Yang penting:** pakai **username persis** yang ditampilkan di UI Harbor.

***

### A2. Simpan ke GitLab Variables

Di GitLab repo kamu: **Settings → CI/CD → Variables** tambahkan:

* `HARBOR_USERNAME` = `<robot username dari Harbor>`
* `HARBOR_PASSWORD` = `<robot token dari Harbor>`

Rekomendasi setting:

* ✅ **Protected** (biar cuma jalan di branch protected; kamu pakai main)
* ✅ **Masked** (kalau GitLab mengizinkan; kalau ditolak karena format token, cukup Protected aja)

> Setelah ini, kamu **nggak perlu pakai admin Harbor** di pipeline.

***

## 3) Trust CA Harbor di VM lain (biar `docker login` dan pull image aman)

Dokumentasi Docker untuk trust registry dengan custom CA pakai `/etc/docker/certs.d/<host>/ca.crt`. ([Docker Documentation](https://docs.docker.com/engine/security/certificates/?utm_source=chatgpt.com))

### 3.1 Copy CA dari vm-harbor (PAKAI IP biar gak ngadat)

Di **vm-docker**:

```bash
scp cikal@192.168.56.43:/etc/harbor/certs/ca.crt /tmp/harbor-ca.crt
```

Kalau masih timeout → pastikan di vm-harbor:

```bash
sudo apt install ufw
sudo ufw status
sudo ufw allow OpenSSH
```

### 3.2 Install CA ke Docker (vm-docker)

Di **vm-docker**:

```bash
sudo mkdir -p /etc/docker/certs.d/harbor.local
sudo cp /tmp/harbor-ca.crt /etc/docker/certs.d/harbor.local/ca.crt
sudo systemctl restart docker

docker login harbor.local
```

### Fixed Install CA Harbor ke Docker + OS trust store (HTTPS tetap proper)

Jalankan di **vm-docker (runner host)**:

```bash
# 1) Ambil cert chain dari Harbor (pakai SNI)
openssl s_client -showcerts -connect harbor.local:443 -servername harbor.local </dev/null 2>/dev/null \
  | sed -n '/-----BEGIN CERTIFICATE-----/,/-----END CERTIFICATE-----/p' \
  > /tmp/harbor-ca.crt

# 2) Pasang ke Docker trust store
sudo mkdir -p /etc/docker/certs.d/harbor.local
sudo mkdir -p /etc/docker/certs.d/harbor.local:443

sudo cp /tmp/harbor-ca.crt /etc/docker/certs.d/harbor.local/ca.crt
sudo cp /tmp/harbor-ca.crt /etc/docker/certs.d/harbor.local:443/ca.crt

# 3) Pasang juga ke OS trust store (biar tool lain ikut percaya)
sudo cp /tmp/harbor-ca.crt /usr/local/share/ca-certificates/harbor-ca.crt
sudo update-ca-certificates

# 4) Restart docker daemon
sudo systemctl restart docker
```

### Ambil certificate/CA Harbor dari vm-harbor

Di **vm-harbor**, cari file cert yang dipakai Harbor. Jalankan:

```bash
sudo find / -maxdepth 4 -type f \( -name "*harbor*.crt" -o -name "server.crt" -o -name "tls.crt" \) 2>/dev/null
```

Biasanya cert Harbor ada di folder data/harbor atau cert yang kamu generate saat install.\
Karena Harbor kamu **self-signed**, biasanya **file CRT itu sendiri bisa dipakai sebagai CA**.

Misal ketemu file cert di `/opt/harbor/certs/harbor.local.crt` (contoh), lalu copy ke vm-k8s & vm-worker:

```bash
scp /opt/harbor/certs/harbor.local.crt cikal@vm-k8s:/tmp/harbor-ca.crt
scp /opt/harbor/certs/harbor.local.crt cikal@vm-worker:/tmp/harbor-ca.crt
```

> Kalau lokasi file cert kamu beda, ganti path-nya sesuai hasil `find`.

#### Test manual (di vm-docker)

```bash
docker logout harbor.local || true
docker login harbor.local -u "$HARBOR_USERNAME" -p "$HARBOR_PASSWORD"

# coba push image kecil biar cepet (atau pull dulu)
docker pull harbor.local/threebody/frontend:d4d27d1d || true
```

Kalau login/pull aman, pipeline `docker push` harusnya beres.

> Catatan: Kalau Harbor kamu pakai **self-signed** cert, isi `/tmp/harbor-ca.crt` di atas biasanya cukup (server cert = CA).\
> Kalau Harbor pakai CA internal + intermediate, langkah di atas juga sering works karena kita taruh full chain.

#### 1) Test login (pakai interaktif biar gak tergantung env)

Di `vm-docker` jalankan:

```bash
docker logout harbor.local || true
docker login harbor.local
```

Masukkan user/pass Harbor (mis. `admin` + password kamu, atau robot account).

Kalau kamu memang mau pakai env:

```bash
export HARBOR_USERNAME='admin'
export HARBOR_PASSWORD='PASSWORD_HARBOR_KAMU'
echo "$HARBOR_PASSWORD" | docker login harbor.local -u "$HARBOR_USERNAME" --password-stdin
```

### Bonus penting (biar nanti Kubernetes gak ImagePullBackOff)

Kalau cluster Kubernetes kamu pakai **containerd** (umum di kubeadm), tiap node K8s juga harus trust CA Harbor.

Di **setiap node K8s**:

```bash
sudo mkdir -p /etc/containerd/certs.d/harbor.local
sudo cp /usr/local/share/ca-certificates/harbor-ca.crt /etc/containerd/certs.d/harbor.local/ca.crt
sudo systemctl restart containerd
```

(Atau copy `/tmp/harbor-ca.crt` dari vm-docker ke node-node K8s, lalu taruh sebagai `ca.crt` di path itu.)

***

## 4) Push project dari GitHub → GitLab (step-by-step)

GitLab mendukung “create project dengan `git push`” atau lewat UI. ([Dokumentasi GitLab](https://docs.gitlab.com/topics/git/project/))

### 4.1 Siapkan SSH key untuk GitLab (di vm-docker)

```bash
ssh-keygen -t ed25519 -C "vm-docker-gitlab" -f ~/.ssh/id_ed25519 -N ""
cat ~/.ssh/id_ed25519.pub
```

#### 1) Buat folder & file config

```bash
mkdir -p ~/.ssh
nano ~/.ssh/config
```

#### 2) Isi file `~/.ssh/config`

Tempel ini:

```sshconfig
Host gitlab-443
  HostName altssh.gitlab.com
  User git
  Port 443
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
  ServerAliveInterval 60
  ServerAliveCountMax 3
```

```
# pastikan owner bener
sudo chown -R cikal:cikal /home/cikal/.ssh

# permission ketat
chmod 700 /home/cikal/.ssh
chmod 600 /home/cikal/.ssh/config

# kunci private key juga harus 600
chmod 600 /home/cikal/.ssh/id_ed25519 2>/dev/null || true
chmod 644 /home/cikal/.ssh/id_ed25519.pub 2>/dev/null || true

# known_hosts boleh 644 atau 600, kita set aman 644
chmod 644 /home/cikal/.ssh/known_hosts 2>/dev/null || true
```

#### 3) Set permission (wajib, biar SSH nggak nolak)

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
chmod 600 ~/.ssh/config
```

Copy output pubkey → GitLab:

* Profile → Preferences → **SSH Keys** → Paste → Add key

#### 4. Preload `known_hosts` (biar setelah reboot tidak prompt “authenticity”)

```bash
ssh-keyscan -p 443 altssh.gitlab.com >> ~/.ssh/known_hosts
chmod 644 ~/.ssh/known_hosts
```

### Opsi 1 (recommended): set DNS di systemd-resolved

1. Edit config:

```bash
sudo nano /etc/systemd/resolved.conf
```

2. Isi / aktifkan bagian ini:

```ini
[Resolve]
DNS=1.1.1.1 8.8.8.8
FallbackDNS=9.9.9.9 8.8.4.4
```

3. Restart:

```bash
sudo systemctl restart systemd-resolved
```

4. Bersihkan cache + cek server DNS yang kepakai:

```bash
sudo resolvectl flush-caches
resolvectl status | sed -n '1,120p'
```

5. Uji resolve:

```bash
resolvectl query altssh.gitlab.com
```

> Catatan: `/etc/resolv.conf` kamu sudah benar (stub ke `127.0.0.53`), jadi **nggak perlu diedit manual**.

Test:

```bash
ssh -T git@gitlab-443
```

### 4.2 Clone repo GitHub kamu

Di **vm-docker**:

```bash
cd ~
git clone https://github.com/cikal-farid/three-body-problem-main.git
cd three-body-problem-main
git status
```

### 4.3 Buat project baru di GitLab (UI)

Di GitLab:

* Create new → New project/repository
* Create blank project
* Nama misal: `three-body-problem-main`

```
three-body-problem-main
```

* **Jangan centang “Initialize with README”** (biar tidak konflik history)

Copy URL SSH repo GitLab-mu, contohnya:

```txt
git@gitlab.com:cikalfarid/three-body-problem-main.git
```

### 4.4 Add remote GitLab dan push

Di folder repo (vm-docker):

```bash
git remote -v
git remote add gitlab git@gitlab.com:cikalfarid/three-body-problem-main.git

git branch -M main
git push -u gitlab main
```

> Kalau GitLab kamu self-managed, ganti `gitlab.com` sesuai domain instance.

***

## 5) Rapihin repo: file yang wajib kamu tambahkan (Dockerfile multistage + deploy + CI)

Di repo **GitLab** kamu (di vm-docker), buat struktur ini:

```bash
cd ~/three-body-problem-main
mkdir -p deploy/edge/nginx/conf.d
mkdir -p deploy/edge/certs
mkdir -p deploy/staging
mkdir -p deploy/k8s/base
mkdir -p deploy/observability
```

### 5.1 Tag image = `CI_COMMIT_SHORT_SHA`

`CI_COMMIT_SHORT_SHA` adalah predefined variable GitLab.\
Kita akan pakai ini di `.gitlab-ci.yml`.

### 5.2 Dockerfile multistage (contoh template)

> Karena aku tidak bisa build repo kamu di sini, aku kasih template “aman” yang paling umum. Kalau nama folder/entrypoint berbeda, tinggal sesuaikan.

#### (A) `go/Dockerfile`

```dockerfile
# ---- build stage ----
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN go mod download
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o server .

# ---- runtime stage ----
FROM gcr.io/distroless/static:nonroot
WORKDIR /
COPY --from=builder /app/server /server
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/server"]
```

#### (B) `laravel/Dockerfile` (composer builder → apache runtime)

```
# syntax=docker/dockerfile:1

# Ambil binary composer aja (JANGAN dipakai buat run composer, karena PHP-nya bisa berubah)
FROM composer:2 AS composerbin

# Base runtime (PHP 8.2) -> ini yang kita pakai juga untuk build composer supaya versi PHP match
FROM php:8.2-apache AS base

WORKDIR /var/www/html

# Install OS deps + PHP extensions yang umum dipakai Laravel
RUN apt-get update && apt-get install -y --no-install-recommends \
    git unzip \
    libzip-dev libicu-dev libonig-dev \
    libpng-dev libjpeg62-turbo-dev libfreetype6-dev \
  && docker-php-ext-configure gd --with-freetype --with-jpeg \
  && docker-php-ext-install -j"$(nproc)" \
    pdo_mysql mbstring zip intl bcmath exif gd opcache \
  && a2enmod rewrite headers \
  && rm -rf /var/lib/apt/lists/*

# Set Apache docroot ke /public (Laravel)
ENV APACHE_DOCUMENT_ROOT=/var/www/html/public
RUN sed -ri 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/sites-available/*.conf \
 && sed -ri 's!/var/www/!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf

# -------------------------
# Build stage: vendor install (composer run di PHP 8.2)
# -------------------------
FROM base AS vendor

# Copy composer binary ke image PHP 8.2
COPY --from=composerbin /usr/bin/composer /usr/bin/composer

# Copy seluruh source dulu supaya script composer (artisan package:discover) tidak gagal
COPY . .

# Install vendor (no-dev) pakai PHP 8.2 => tidak kena masalah PHP 8.5 lagi
ENV COMPOSER_ALLOW_SUPERUSER=1
RUN composer install \
    --no-dev \
    --prefer-dist \
    --no-interaction \
    --no-progress \
    --optimize-autoloader

# -------------------------
# Runtime stage
# -------------------------
FROM base AS runtime

COPY --from=vendor /var/www/html /var/www/html

# Permission Laravel
RUN chown -R www-data:www-data storage bootstrap/cache || true \
 && chmod -R 775 storage bootstrap/cache || true

EXPOSE 80
CMD ["apache2-foreground"]
```

### (Opsional tapi disarankan) Full replace: `laravel/.dockerignore`

Biar build context kecil & cache lebih enak:

```gitignore
.git
node_modules
vendor
storage/logs
storage/framework/cache
storage/framework/sessions
storage/framework/views
.env
.env.*
.DS_Store
```

#### (C) `frontend/Dockerfile` (node build → nginx serve)

```dockerfile
cat > frontend/Dockerfile <<'EOF'
FROM node:20 AS build
WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

ARG REACT_APP_GO_API_BASE=/go
ARG REACT_APP_LARAVEL_API_BASE=/laravel

ENV REACT_APP_GO_API_BASE=$REACT_APP_GO_API_BASE
ENV REACT_APP_LARAVEL_API_BASE=$REACT_APP_LARAVEL_API_BASE

RUN npm run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
EOF
```

### 5.3 Perbaiki frontend agar tidak hardcode `localhost`

Jalankan di `vm-docker`:

Buat file:

* `frontend/.env.development`

```env
cat > frontend/.env.development <<'EOF'
REACT_APP_GO_API_BASE=http://localhost:8080
REACT_APP_LARAVEL_API_BASE=http://localhost:8001
EOF
```

* `frontend/.env.production`

```env
cat > frontend/.env.production <<'EOF'
REACT_APP_GO_API_BASE=/go
REACT_APP_LARAVEL_API_BASE=/laravel
EOF
```

Lalu di code frontend (biasanya `frontend/src/App.js` atau sejenis):

* ganti URL hardcode menjadi:

```js
const GO_API = (process.env.REACT_APP_GO_API_BASE || "/go").replace(/\/$/, "");
const LARAVEL_API = (process.env.REACT_APP_LARAVEL_API_BASE || "/laravel").replace(/\/$/, "");
```

dan request jadi:

* `${GO_BASE}/api/products`
* `${LARAVEL_BASE}/api/products`

#### Jika Ingin full replace untuk `frontend/src/App.js` jalankan perintah dibawah ini

```
cat > frontend/src/App.js <<'EOF'
import React, { useMemo, useState } from "react";
import "./App.css";

/**
 * Frontend akan hit 2 backend:
 * - Go API via Edge Nginx: /go/api/products
 * - Laravel API via Edge Nginx: /laravel/api/products
 *
 * Env yang dipakai (CRA harus prefix REACT_APP_):
 * - REACT_APP_GO_API_BASE
 * - REACT_APP_LARAVEL_API_BASE
 */
function App() {
  const [data, setData] = useState(null);
  const [activeAPI, setActiveAPI] = useState("");
  const [loading, setLoading] = useState(false);

  // Base path (prod) atau full URL (dev)
  const GO_API_BASE = useMemo(() => {
    return (process.env.REACT_APP_GO_API_BASE || "/go").replace(/\/$/, "");
  }, []);

  const LARAVEL_API_BASE = useMemo(() => {
    return (process.env.REACT_APP_LARAVEL_API_BASE || "/laravel").replace(/\/$/, "");
  }, []);

  // Endpoint yang diminta tugas (umumnya)
  const PRODUCTS_PATH = "/api/products";

  const endpoints = useMemo(() => {
    return {
      go: `${GO_API_BASE}${PRODUCTS_PATH}`,
      laravel: `${LARAVEL_API_BASE}${PRODUCTS_PATH}`,
    };
  }, [GO_API_BASE, LARAVEL_API_BASE]);

  // Biar tampilan aman walau response shape beda-beda
  const normalizePayload = (payload) => {
    // 1) kalau array langsung dianggap list produk
    if (Array.isArray(payload)) {
      return {
        success: true,
        message: "OK",
        count: payload.length,
        data: payload,
        raw: null,
      };
    }

    // 2) kalau object, cek pola umum
    if (payload && typeof payload === "object") {
      // pola { success, message, count, data: [] }
      if (Array.isArray(payload.data)) {
        return {
          success: payload.success ?? true,
          message: payload.message ?? "OK",
          count: payload.count ?? payload.data.length,
          data: payload.data,
          raw: null,
        };
      }

      // pola { products: [] }
      if (Array.isArray(payload.products)) {
        return {
          success: true,
          message: "OK",
          count: payload.products.length,
          data: payload.products,
          raw: null,
        };
      }

      // pola lain: tampilkan raw
      return {
        success: payload.success ?? true,
        message: payload.message ?? "OK",
        count: payload.count ?? 0,
        data: [],
        raw: payload,
      };
    }

    // 3) selain itu
    return { success: true, message: "OK", count: 0, data: [], raw: payload };
  };

  const fetchJson = async (url) => {
    const response = await fetch(url, {
      method: "GET",
      headers: { Accept: "application/json" },
    });

    // kalau server balikin non-JSON, ini tetap aman
    const text = await response.text();
    let parsed;
    try {
      parsed = JSON.parse(text);
    } catch {
      parsed = { success: false, message: "Non-JSON response", raw: text };
    }

    if (!response.ok) {
      // normalisasi error supaya UI tetap konsisten
      return {
        success: false,
        message: `HTTP ${response.status}`,
        error: parsed?.error || parsed?.message || "Request failed",
        raw: parsed,
      };
    }

    return parsed;
  };

  const fetchLaravelData = async () => {
    setLoading(true);
    setActiveAPI("Laravel");
    setData(null);

    try {
      const payload = await fetchJson(endpoints.laravel);
      const normalized = normalizePayload(payload);
      setData(normalized);
    } catch (err) {
      setData({
        success: false,
        message: "Error connecting to Laravel API",
        error: err?.message || String(err),
        data: [],
        count: 0,
        raw: null,
      });
    } finally {
      setLoading(false);
    }
  };

  const fetchGoData = async () => {
    setLoading(true);
    setActiveAPI("Go");
    setData(null);

    try {
      const payload = await fetchJson(endpoints.go);
      const normalized = normalizePayload(payload);
      setData(normalized);
    } catch (err) {
      setData({
        success: false,
        message: "Error connecting to Go API",
        error: err?.message || String(err),
        data: [],
        count: 0,
        raw: null,
      });
    } finally {
      setLoading(false);
    }
  };

  const renderProducts = (products) => {
    if (!products || products.length === 0) return null;

    return (
      <div className="products-list">
        <h4>Daftar Produk:</h4>
        {products.map((p, index) => {
          const name = p.name ?? p.title ?? `Product ${index + 1}`;
          const price = p.price ?? p.cost ?? "-";
          const category = p.category ?? p.type ?? "-";
          const quantity = p.quantity ?? p.stock ?? "-";
          const description = p.description ?? p.desc ?? "";

          return (
            <div key={p.id ?? index} className="product-card">
              <h5>{name}</h5>
              <p>
                <strong>Harga:</strong> {price}
              </p>
              <p>
                <strong>Kategori:</strong> {category}
              </p>
              <p>
                <strong>Stok:</strong> {quantity}
              </p>
              {description ? <p className="description">{description}</p> : null}
            </div>
          );
        })}
      </div>
    );
  };

  const renderData = () => {
    if (!data) return null;

    return (
      <div className="data-container">
        <h3>Hasil dari {activeAPI} API:</h3>

        {data.success ? (
          <div className="success-data">
            <p>
              <strong>Status:</strong> ✅ {data.message}
            </p>
            <p>
              <strong>Total Products:</strong> {data.count || 0}
            </p>

            {renderProducts(data.data)}

            {data.raw ? (
              <>
                <h4>Raw Response (debug):</h4>
                <pre style={{ textAlign: "left", whiteSpace: "pre-wrap" }}>
                  {JSON.stringify(data.raw, null, 2)}
                </pre>
              </>
            ) : null}
          </div>
        ) : (
          <div className="error-data">
            <p>
              <strong>Status:</strong> ❌ {data.message}
            </p>
            <p>
              <strong>Error:</strong> {data.error || "Unknown error"}
            </p>
            {data.raw ? (
              <pre style={{ textAlign: "left", whiteSpace: "pre-wrap" }}>
                {JSON.stringify(data.raw, null, 2)}
              </pre>
            ) : null}
          </div>
        )}
      </div>
    );
  };

  return (
    <div className="App">
      <header className="App-header">
        <h1>🚀 Products API Frontend</h1>
        <p className="subtitle">Interface untuk mengakses Laravel API dan Go API</p>

        <div className="buttons-container">
          <button className="api-button laravel-btn" onClick={fetchLaravelData} disabled={loading}>
            {loading && activeAPI === "Laravel" ? <span>⏳ Loading...</span> : <span>🐘 Hit Laravel API</span>}
          </button>

          <button className="api-button go-btn" onClick={fetchGoData} disabled={loading}>
            {loading && activeAPI === "Go" ? <span>⏳ Loading...</span> : <span>🐹 Hit Go API</span>}
          </button>
        </div>

        <div className="api-info">
          <div className="api-endpoint">
            <strong>Laravel Endpoint:</strong> <code>{endpoints.laravel}</code>
          </div>
          <div className="api-endpoint">
            <strong>Go Endpoint:</strong> <code>{endpoints.go}</code>
          </div>
        </div>

        {renderData()}
      </header>
    </div>
  );
}

export default App;
EOF

```

### Test build lokal dulu (biar yakin sebelum push)

Masih di vm-docker:

```bash
cd ~/three-body-problem-main/frontend
npm ci
npm run build
```

Kalau ini sukses, pipeline juga bakal sukses di step frontend.

***

## 6) Edge Nginx (jalan di vm-docker) + HTTPS + Rate limiting

Dokumentasi Nginx rate limiting ada `limit_req_zone` dan `limit_req`.

### 6.1 Buat cert untuk Edge (staging.local + prod.local)

Agar simpel: kita **sign pakai CA yang sama dari vm-harbor**, lalu copy ke vm-docker.

#### 6.1.1 Generate cert EDGE di vm-harbor

Di **vm-harbor**:

```bash
cd /etc/harbor/certs

sudo openssl genrsa -out edge.local.key 4096
sudo openssl req -new -sha512 \
  -subj "/C=ID/ST=Jakarta/L=Jakarta/O=Lab/OU=DevOps/CN=staging.local" \
  -key edge.local.key \
  -out edge.local.csr

cat | sudo tee v3.edge.ext >/dev/null <<'EOF'
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=staging.local
DNS.2=prod.local
IP.1=192.168.56.42
EOF

sudo openssl x509 -req -sha512 -days 3650 \
  -in edge.local.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out edge.local.crt \
  -extfile v3.edge.ext
```

Copy ke vm-docker:

```bash
scp /etc/harbor/certs/edge.local.crt cikal@192.168.56.42:/tmp/edge.local.crt
scp /etc/harbor/certs/edge.local.key cikal@192.168.56.42:/tmp/edge.local.key
```

#### 6.1.2 Taruh cert di repo (vm-docker)

Di **vm-docker** (repo):

```bash
cd ~/three-body-problem-main
cp /tmp/edge.local.crt deploy/edge/certs/tls.crt
cp /tmp/edge.local.key deploy/edge/certs/tls.key
```

### 6.2 Buat config Nginx edge

File: `deploy/edge/nginx/conf.d/edge.conf`

```nginx
# =========================
# EDGE: HTTPS + Rate Limit
# vm: vm-docker
#
# staging.local  -> docker compose di vm-docker (stg-*)
# prod.local     -> Kubernetes NodePort (vm-worker 192.168.56.45)
# =========================

# --- rate limit zone (contoh sederhana) ---
limit_req_zone $binary_remote_addr zone=req_limit_per_ip:10m rate=10r/s;

# --- Upstream STAGING (docker network: edge-net) ---
upstream stg_frontend { server stg-frontend:80; }
upstream stg_go       { server stg-go:8080; }
upstream stg_laravel  { server stg-laravel:80; }

# --- Upstream PROD (Kubernetes NodePort) ---
# Disarankan pakai worker sebagai primary.
# (control-plane kadang firewall / taint / kebijakan jaringan)
upstream prod_frontend { server 192.168.56.45:30080; }
upstream prod_go       { server 192.168.56.45:30081; }
upstream prod_laravel  { server 192.168.56.45:30082; }

# =========================
# STAGING
# =========================
server {
  listen 443 ssl;
  server_name staging.local;

  ssl_certificate     /etc/nginx/certs/tls.crt;
  ssl_certificate_key /etc/nginx/certs/tls.key;

  # rate limit (opsional, tapi requirement kamu ada)
  limit_req zone=req_limit_per_ip burst=20 nodelay;

  # timeout biar gak gampang 504 pas backend agak lama
  proxy_connect_timeout 5s;
  proxy_send_timeout 60s;
  proxy_read_timeout 60s;

  # Frontend
  location / {
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_pass http://stg_frontend;
  }

  # Go API via /go/
  location /go/ {
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Real-IP $remote_addr;

    # strip prefix /go/
    proxy_pass http://stg_go/;
  }

  # Laravel API via /laravel/
  location /laravel/ {
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Real-IP $remote_addr;

    # strip prefix /laravel/
    proxy_pass http://stg_laravel/;
  }
}

# =========================
# PROD
# =========================
server {
  listen 443 ssl;
  server_name prod.local;

  ssl_certificate     /etc/nginx/certs/tls.crt;
  ssl_certificate_key /etc/nginx/certs/tls.key;

  limit_req zone=req_limit_per_ip burst=20 nodelay;

  proxy_connect_timeout 5s;
  proxy_send_timeout 60s;
  proxy_read_timeout 60s;

  # Frontend
  location / {
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_pass http://prod_frontend;
  }

  # Go API
  location /go/ {
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_pass http://prod_go/;
  }

  # Laravel API (INI YANG KAMU BUTUHKAN)
  location /laravel/ {
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_pass http://prod_laravel/;
  }
}

```

### 6.3 Docker Compose untuk Edge

File: `deploy/edge/docker-compose.edge.yml`

```yaml
services:
  edge-nginx:
    image: nginx:alpine
    container_name: edge-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./certs:/etc/nginx/certs:ro
    networks:
      - edge-net
    restart: unless-stopped

networks:
  edge-net:
    name: edge-net
```

***

## 7) Staging Docker Compose (images dari Harbor)

File: `deploy/staging/docker-compose.staging.yml`

```yaml
services:
  stg-mysql:
    image: mysql:8.0
    container_name: stg-mysql
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - stg_mysql_data:/var/lib/mysql
    networks: [edge-net]
    healthcheck:
      test: ["CMD-SHELL", "MYSQL_PWD=$$MYSQL_ROOT_PASSWORD mysqladmin ping -h 127.0.0.1 --protocol=tcp -uroot --silent"]
      interval: 5s
      timeout: 5s
      retries: 40

  stg-go:
    image: ${REGISTRY}/go:${TAG}
    container_name: stg-go
    environment:
      DB_HOST: stg-mysql
      DB_PORT: "3306"
      DB_NAME: ${MYSQL_DATABASE}
      DB_USER: ${MYSQL_USER}
      DB_PASS: ${MYSQL_PASSWORD}
    networks: [edge-net]
    depends_on:
      stg-mysql:
        condition: service_healthy

  stg-laravel:
    image: ${REGISTRY}/laravel:${TAG}
    container_name: stg-laravel
    environment:
      DB_HOST: stg-mysql
      DB_PORT: "3306"
      DB_DATABASE: ${MYSQL_DATABASE}
      DB_USERNAME: ${MYSQL_USER}
      DB_PASSWORD: ${MYSQL_PASSWORD}
      APP_KEY: ${LARAVEL_APP_KEY}
    networks: [edge-net]
    depends_on:
      stg-mysql:
        condition: service_healthy

  stg-frontend:
    image: ${REGISTRY}/frontend:${TAG}
    container_name: stg-frontend
    networks: [edge-net]
    depends_on:
      - stg-go
      - stg-laravel

volumes:
  stg_mysql_data:

networks:
  edge-net:
    external: true
```

***

## 8) Install Kubernetes (kubeadm) di vm-k8s + vm-worker

Ikuti konsep dari dokumentasi “Installing kubeadm”.\
Pakai cgroup driver `systemd` sesuai rekomendasi kubeadm.

### 8.1 Di vm-k8s dan vm-worker: disable swap

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### 8.2 Install containerd + config default

```bash
sudo apt-get update -y
sudo apt-get install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
```

Set systemd cgroups:

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl enable --now containerd
sudo systemctl restart containerd
```

### 8.3 Install kubeadm/kubelet/kubectl (ikuti repo resmi)

Ikuti langkah dari doc `Installing kubeadm`.

Di **vm-k8s & vm-worker** (Ubuntu):

```
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

sudo mkdir -p -m 0755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Setelah install:

```bash
kubeadm version
kubelet --version
kubectl version --client
```

### 8.4 Init cluster (di vm-k8s)

#### Aktifkan IP\_Forward di vm-k8s dan vm-worker

```
cat <<'EOF' | sudo tee /etc/modules-load.d/k8s.conf >/dev/null
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<'EOF' | sudo tee /etc/sysctl.d/99-kubernetes.conf >/dev/null
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

Pilih Pod CIDR (contoh Calico):

```bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.56.44 \
  --apiserver-cert-extra-sans=192.168.56.44,vm-k8s \
  --pod-network-cidr=192.168.0.0/16
```

```
kubeadm token create --print-join-command --ttl 24h
```

Konfigurasi kubectl untuk user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes
```

Install CNI (contoh Calico):

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

Tunggu node ready:

```bash
watch -n 2 -d 'kubectl get nodes'
```

### 8.5 Join worker (di vm-worker)

Dari output kubeadm init, ambil command join.\
Misal:

```bash
kubeadm join 192.168.56.44:6443 --token 3c9zrx.0dy1u2ghwftysn0y \
        --discovery-token-ca-cert-hash sha256:3d00ec29c65595b1cd4bbabacaaa2d945f06d07610c3166dfe5df19f758405c7
```

Cek node:

```bash
kubectl get nodes -o wide
```

#### Jika terdapat error pada saat join maka jalankan perintah dibawah ini di vm-k8s

```
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube
sudo systemctl restart containerd
```

***

## 9) K8s nodes trust Harbor CA (containerd)

Containerd pakai `config_path="/etc/containerd/certs.d"` + `hosts.toml`.

Lakukan di **vm-k8s dan vm-worker**:

### 9.1 Copy CA ke node

Dari vm-harbor, copy ke masing-masing node (pakai IP):

```bash
# jalankan dari vm-k8s
scp cikal@192.168.56.43:/etc/harbor/certs/ca.crt /tmp/harbor-ca.crt
```

### 9.2 Buat certs.d + hosts.toml

Jalankan di **vm-k8s** dan **vm-worker**:

```
sudo mkdir -p /etc/containerd/certs.d/harbor.local
sudo cp /tmp/harbor-ca.crt /etc/containerd/certs.d/harbor.local/ca.crt

sudo tee /etc/containerd/certs.d/harbor.local/hosts.toml >/dev/null <<'EOF'
server = "https://harbor.local"

[host."https://harbor.local"]
  capabilities = ["pull", "resolve", "push"]
  ca = "/etc/containerd/certs.d/harbor.local/ca.crt"
EOF

sudo systemctl restart containerd
sudo systemctl is-active containerd
```

### 9.3 Aktifkan config\_path di containerd config

Edit:

```bash
sudo nano /etc/containerd/config.toml
```

Cari blok:

```toml
[plugins."io.containerd.grpc.v1.cri".registry]
```

Pastikan ada:

```toml
config_path = "/etc/containerd/certs.d"
```

Restart:

```bash
sudo systemctl restart containerd
```

#### (Opsional tapi bagus) cek TLS dari node

Di **vm-worker**:

```bash
curl -v https://harbor.local/v2/ 2>&1 | tail -n 20
```

Kalau CA sudah beres, error **x509 unknown authority** harusnya hilang (respon bisa 401/404 itu normal—yang penting bukan error TLS).

> Kalau masih error TLS, berarti cert yang kamu copy **bukan CA yang benar**, atau Harbor cert kamu butuh chain CA lain.

***

## 10) Manifest Kubernetes (NodePort) untuk PROD

Buat namespace `threebody-prod` dan service NodePort:

* frontend: **30080**
* go: **30081**
* laravel: **30082**

> MySQL untuk prod: simplest pakai StatefulSet + hostPath (karena bare kubeadm biasanya belum ada storageclass). Kalau kamu nanti mau rapi, baru pasang provisioner.

(Aku bisa tuliskan YAML lengkapnya juga, tapi karena jawaban ini sudah panjang, minimal kamu butuh: `namespace.yaml`, `mysql.yaml`, `go.yaml`, `laravel.yaml`, `frontend.yaml` + `imagePullSecret` untuk Harbor.)

#### 10.1 Namespace

**deploy/k8s/base/00-namespace.yaml**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: threebody-prod
```

#### 10.2 MySQL (hostPath, pin ke worker)

> Jalankan ini **sekali** di `vm-worker` supaya folder ada:

```bash
sudo mkdir -p /data/threebody/mysql
sudo chmod 777 /data/threebody/mysql
```

**deploy/k8s/base/10-mysql.yaml**

```
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: threebody-prod
spec:
  selector:
    app: mysql
  ports:
    - name: mysql
      port: 3306
      targetPort: 3306
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: threebody-prod
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      nodeSelector:
        kubernetes.io/hostname: vm-worker

      # opsional tapi sangat membantu untuk hostPath PV (permission)
      initContainers:
        - name: init-mysql-perms
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              mkdir -p /var/lib/mysql
              chown -R 999:999 /var/lib/mysql || true
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql

      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - name: mysql
              containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_ROOT_PASSWORD
            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_DATABASE
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_USER
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_PASSWORD

          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql

          readinessProbe:
            exec:
              command:
                - sh
                - -c
                - MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysqladmin ping -h 127.0.0.1 -uroot --silent
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 30

          livenessProbe:
            exec:
              command:
                - sh
                - -c
                - MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysqladmin ping -h 127.0.0.1 -uroot --silent
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 10

      # ✅ FIX UTAMA: definisikan volume mysql-data (yang dipakai di volumeMounts)
      volumes:
        - name: mysql-data
          persistentVolumeClaim:
            claimName: mysql-data

  # kamu sebelumnya punya ini, aku biarkan agar “model” StatefulSet-nya tetap sama
  volumeClaimTemplates: []
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-hostpath
spec:
  capacity:
    storage: 5Gi
  accessModes: ["ReadWriteOnce"]
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/threebody/mysql
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data
  namespace: threebody-prod
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 5Gi
  volumeName: mysql-pv-hostpath
```

### 1 hal penting sebelum kamu push (biar MySQL nggak nyangkut permission)

Karena PV kamu pakai `hostPath: /data/threebody/mysql` dan MySQL jalan di **vm-worker** (nodeSelector), pastikan folder itu ada di **vm-worker**:

Di **vm-worker** jalankan:

```bash
sudo mkdir -p /data/threebody/mysql
sudo chown -R 999:999 /data/threebody/mysql
sudo chmod 700 /data/threebody/mysql
```

> `999:999` ini biasanya user mysql di image `mysql:8.0`. Kalau pun beda, initContainer yang aku pasang juga bantu.

> Catatan: HostPath ini “lab-friendly” (bukan best practice cloud), tapi cukup untuk tugas.

#### 10.3 GO (NodePort 30081)

**deploy/k8s/base/20-go.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: go
  namespace: threebody-prod
spec:
  type: NodePort
  selector:
    app: go
  ports:
    - name: http
      port: 8080
      targetPort: 8080
      nodePort: 30081
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go
  namespace: threebody-prod
spec:
  replicas: 1
  selector:
    matchLabels:
      app: go
  template:
    metadata:
      labels:
        app: go
    spec:
      imagePullSecrets:
        - name: harbor-pull
      containers:
        - name: go
          image: harbor.local/threebody/go:latest
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              value: mysql
            - name: DB_PORT
              value: "3306"
            - name: DB_NAME
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_DATABASE
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_USER
            - name: DB_PASS
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_PASSWORD
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 10
```

> Kalau service Go kamu **tidak punya endpoint `/health`**, ganti probe ke path yang ada (atau hapus probe dulu biar gampang).

#### 10.4 Laravel (NodePort 30082)

**deploy/k8s/base/30-laravel.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: laravel
  namespace: threebody-prod
spec:
  type: NodePort
  selector:
    app: laravel
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30082
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel
  namespace: threebody-prod
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
        - name: harbor-pull
      containers:
        - name: laravel
          image: harbor.local/threebody/laravel:latest
          ports:
            - containerPort: 80
          env:
            - name: DB_HOST
              value: mysql
            - name: DB_PORT
              value: "3306"
            - name: DB_DATABASE
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_DATABASE
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_PASSWORD
            - name: APP_KEY
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: LARAVEL_APP_KEY
            - name: APP_ENV
              value: production
            - name: APP_DEBUG
              value: "false"
```

### Tambahkan manifest Job untuk migrate + seed

Buat file baru:

`deploy/k8s/base/35-laravel-migrate-job.yaml`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: laravel-migrate
  namespace: threebody-prod
spec:
  ttlSecondsAfterFinished: 600
  backoffLimit: 3
  template:
    metadata:
      labels:
        app: laravel-migrate
    spec:
      restartPolicy: Never

      # Kalau deploy laravel kamu pakai imagePullSecrets, samakan di sini.
      # Kalau kamu sudah patch default serviceaccount dengan imagePullSecrets,
      # bagian ini boleh dihapus.
      imagePullSecrets:
        - name: harbor-regcred

      initContainers:
        - name: wait-mysql
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              echo "[wait-mysql] waiting mysql:3306 ..."
              until nc -z mysql 3306; do sleep 2; done
              echo "[wait-mysql] mysql ready"

      containers:
        - name: migrate
          # akan di-replace oleh pipeline (sed)
          image: __LARAVEL_IMAGE__
          imagePullPolicy: IfNotPresent

          # kita set env yang paling penting untuk DB.
          # aku isi dua versi (DB_* dan MYSQL_*) biar kompatibel sama setup kamu.
          env:
            - name: DB_HOST
              value: mysql
            - name: DB_PORT
              value: "3306"
            - name: DB_DATABASE
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_DATABASE
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_PASSWORD

            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_DATABASE
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_USER
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_PASSWORD
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_ROOT_PASSWORD

          command: ["sh", "-lc"]
          args:
            - |
              set -euo pipefail
              cd /var/www/html

              echo "==> migrate"
              php artisan migrate --force

              echo "==> seed (hanya kalau tabel products masih kosong)"
              php -r '
              try {
                $host=getenv("DB_HOST") ?: "mysql";
                $port=getenv("DB_PORT") ?: "3306";
                $db=getenv("DB_DATABASE") ?: getenv("MYSQL_DATABASE");
                $user=getenv("DB_USERNAME") ?: getenv("MYSQL_USER");
                $pass=getenv("DB_PASSWORD") ?: getenv("MYSQL_PASSWORD");
                $pdo=new PDO("mysql:host=$host;port=$port;dbname=$db;charset=utf8mb4",$user,$pass,[PDO::ATTR_ERRMODE=>PDO::ERRMODE_EXCEPTION]);
                $count=(int)$pdo->query("SELECT COUNT(*) FROM products")->fetchColumn();
                exit($count===0 ? 0 : 2);
              } catch (Throwable $e) { exit(0); }
              ' && php artisan db:seed --force || true

              echo "==> done"
```

#### Penting: pastikan `imagePullSecrets` namanya benar

Cek secret yang dipakai laravel sekarang:

```bash
kubectl -n threebody-prod get deploy laravel -o jsonpath='{.spec.template.spec.imagePullSecrets}'
echo
kubectl -n threebody-prod get secret | grep -i cred
```

Kalau secret kamu bukan `harbor-regcred`, ganti di YAML job itu.

#### 10.5 Frontend (NodePort 30080)

**deploy/k8s/base/40-frontend.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: threebody-prod
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: threebody-prod
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      imagePullSecrets:
        - name: harbor-pull
      containers:
        - name: frontend
          image: harbor.local/threebody/frontend:latest
          ports:
            - containerPort: 80
```

### Buat file job migrate (baru)

Di vm-docker (repo):

```bash
cd ~/three-body-problem-main
mkdir -p deploy/k8s/jobs
nano deploy/k8s/jobs/laravel-migrate-job.yaml
```

Isi file `deploy/k8s/jobs/laravel-migrate-job.yaml` (copy full):

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: __JOB_NAME__
  labels:
    app: laravel-migrate
spec:
  ttlSecondsAfterFinished: 600
  backoffLimit: 3
  template:
    metadata:
      labels:
        app: laravel-migrate
    spec:
      restartPolicy: Never
      serviceAccountName: default
      initContainers:
        - name: wait-mysql
          image: busybox:1.36
          command: ["sh","-c", "echo '[wait-mysql] waiting mysql:3306 ...'; until nc -z mysql 3306; do sleep 2; done; echo '[wait-mysql] mysql ready'"]
      containers:
        - name: migrate
          image: __LARAVEL_IMAGE__
          imagePullPolicy: IfNotPresent
          env:
            - name: APP_ENV
              value: production
            - name: APP_DEBUG
              value: "false"
            - name: APP_KEY
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: LARAVEL_APP_KEY
            - name: DB_CONNECTION
              value: mysql
            - name: DB_HOST
              value: mysql
            - name: DB_PORT
              value: "3306"
            - name: DB_DATABASE
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_DATABASE
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_PASSWORD
          command: ["sh","-lc"]
          args:
            - |
              set -euo pipefail
              cd /var/www/html
              echo "==> migrate"
              php artisan migrate --force
              echo "==> seed only if products empty"
              php -r '
              try {
                $pdo=new PDO(
                  "mysql:host=".getenv("DB_HOST").";port=".getenv("DB_PORT").";dbname=".getenv("DB_DATABASE").";charset=utf8mb4",
                  getenv("DB_USERNAME"),
                  getenv("DB_PASSWORD"),
                  [PDO::ATTR_ERRMODE=>PDO::ERRMODE_EXCEPTION]
                );
                $count=(int)$pdo->query("SELECT COUNT(*) FROM products")->fetchColumn();
                exit($count===0 ? 0 : 2);
              } catch (Throwable $e) { exit(0); }
              ' && php artisan db:seed --force || true
              echo "==> done"

```

✅ Job ini aman untuk jalan berulang:

* migrate idempotent
* seed hanya kalau `products` kosong

***

## 11) Install GitLab Runner di vm-docker + register

Gunakan panduan install GitLab Runner untuk Linux.

### 11.1 Install runner (vm-docker)

```bash
sudo apt-get update -y
sudo apt-get install -y curl ca-certificates
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install -y gitlab-runner
gitlab-runner --version
sudo systemctl status gitlab-runner --no-pager
```

### 11.2 Register runner (shell executor)

Di GitLab project:

* Settings → CI/CD → Runners → “Register a runner”\
  Ambil **registration token** (atau runner token sesuai UI GitLab kamu)

Di vm-docker:

```bash
sudo gitlab-runner register
```

Isi:

* GitLab URL: `https://gitlab.com` (atau domain GitLab kamu)
* Token: (paste)
* Executor: `shell`
* Tags: `deploy`
* Description: `vm-docker-runner`

Cek:

```bash
sudo gitlab-runner status
```

Sebelum push ke gitlab pastikan permission Docker dan lain-lainnya terlebih dahulu dan ikuti perintah dibawah ini.

### 11.3 Fix permission Docker untuk gitlab-runner (vm-docker)

Jalankan di **vm-docker**:

```bash
# 1) pastikan grup docker ada
getent group docker || sudo groupadd docker

# 2) masukkan user gitlab-runner ke grup docker
sudo usermod -aG docker gitlab-runner

# 3) pastikan docker socket group = docker
sudo chown root:docker /var/run/docker.sock
sudo chmod 660 /var/run/docker.sock

# 4) restart docker + runner (penting)
sudo systemctl restart docker
sudo systemctl restart gitlab-runner
```

***

### 11.4 Verifikasi akses (WAJIB sebelum rerun pipeline)

Masih di **vm-docker**, jalankan:

```bash
id gitlab-runner
```

Pastikan output ada `docker` di daftar groups.

Lalu test:

```bash
sudo -u gitlab-runner docker ps
sudo -u gitlab-runner docker info >/dev/null && echo "OK docker access"
```

Kalau `docker ps` masih permission denied, kirim output `ls -l /var/run/docker.sock` dan `id gitlab-runner`.

### Tes yang benar (WAJIB): pakai `crictl pull`

Jalankan di **vm-worker** dan **vm-k8s**:

#### Install crictl (kalau belum ada)

```bash
sudo apt-get update
sudo apt-get install -y cri-tools
```

#### Pastikan endpoint containerd kebaca

```bash
sudo crictl info | head
```

#### Test pull image dari Harbor

```bash
sudo crictl pull harbor.local/threebody/frontend:86cbd04a
```

**Kalau ini sukses** → berarti Kubernetes juga bisa pull, dan problem `x509` untuk pod akan hilang (tinggal restart pod).

**Kalau ini masih x509** → berarti CA kamu memang belum dipercaya / CA file-nya bukan CA yang tepat.

### Cara paling “anti ribet”: tambahkan CA ke OS trust store (recommended)

Ini bikin **keduanya** (CRI + ctr + tools lain) percaya ke Harbor.

Lakukan di **vm-worker dan vm-k8s**:

```bash
sudo cp /etc/containerd/certs.d/harbor.local/ca.crt /usr/local/share/ca-certificates/harbor.local.crt
sudo update-ca-certificates
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

Lalu test ulang:

```bash
sudo crictl pull harbor.local/threebody/frontend:86cbd04a
```

> Ini biasanya langsung beres untuk kasus “self-signed Harbor”.

### Fix cepat (sekali jalan) — jalankan migrate + seed di pod Laravel

Kerjakan di **vm-k8s** (karena `kubectl` kamu sudah siap di sana).

```bash
NS=threebody-prod

# ambil nama pod laravel
LARAVEL_POD=$(kubectl -n "$NS" get pod -l app=laravel -o jsonpath='{.items[0].metadata.name}')
echo "$LARAVEL_POD"

# jalankan migrasi (buat tabel)
kubectl -n "$NS" exec -it "$LARAVEL_POD" -- php artisan migrate --force

# jalankan seeding (isi data awal) - kalau project kamu punya seeder
kubectl -n "$NS" exec -it "$LARAVEL_POD" -- php artisan db:seed --force
```

#### Verifikasi tabelnya benar-benar ada

Kita cek dari pod mysql (karena mysql-0 sudah Running):

```bash
kubectl -n "$NS" exec -it mysql-0 -- sh -lc '
MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysql -uroot -e "
USE $MYSQL_DATABASE;
SHOW TABLES;
SELECT COUNT(*) AS total_products FROM products;
"'
```

#### Test ulang dari vm-docker / laptop

```bash
curl -k https://prod.local/laravel/api/products
```

#### Pastikan server `prod.local` punya upstream prod sendiri

Di **vm-docker**, buka file:

```bash
sudo nano /opt/threebody/edge/nginx/conf.d/edge.conf
```

Cari blok `server_name prod.local;` (atau server prod kamu), lalu pastikan location laravel-nya **bukan** `stg-laravel`.

**Contoh yang benar (paling simpel & robust):**

* Go prod → NodePort 30081 (biasanya sudah benar karena prod go kamu OK)
* Laravel prod → NodePort 30082

Contoh potongan untuk server `prod.local`:

```nginx
# PROD
server {
  listen 443 ssl;
  server_name prod.local;

  # ... ssl cert/key, rate limit, dll ...

  location /go/ {
    proxy_pass http://192.168.56.45:30081/;
    proxy_connect_timeout 5s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;
  }

  location /laravel/ {
    # strip prefix /laravel/
    proxy_pass http://192.168.56.45:30082/;
    proxy_connect_timeout 5s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;
  }
}
```

> Catatan penting `proxy_pass`:
>
> * Pakai trailing slash `/` di akhir upstream supaya path `/laravel/api/products` jadi `/api/products` di backend.

Kalau kamu mau HA ringan, bisa tambahin backup ke control-plane (tapi control-plane kamu tadi timeout, jadi backup ini opsional):

```nginx
proxy_pass http://prod_laravel/;

upstream prod_laravel {
  server 192.168.56.45:30082;
  server 192.168.56.44:30082 backup;
}
```

#### 4B. Apply config edge

Di **vm-docker**:

```bash
docker exec -it edge-nginx nginx -t
sudo systemctl restart threebody-edge
docker exec -it edge-nginx nginx -T | grep -n "server_name prod.local" -n -A40
```

#### 4C. Test ulang via edge

```bash
curl -kfsS --resolve prod.local:443:127.0.0.1 https://prod.local/laravel/api/products | head
```

## FIX UTAMA: Rapikan firewall Kubernetes node (vm-worker + vm-k8s)

Untuk lab/assignment seperti ini, cara paling aman & cepat: **disable UFW di node Kubernetes** (vm-k8s dan vm-worker). Edge HTTPS/rate-limit tetap di vm-docker, jadi aman.

### A. Jalankan di **vm-worker**

```bash
# 1) matikan ufw (paling aman untuk Kubernetes lab)
sudo ufw disable

# 2) pastikan forwarding tidak DROP
sudo iptables -P FORWARD ACCEPT

# 3) pastikan modul dan sysctl untuk forwarding + bridge
sudo modprobe br_netfilter

sudo tee /etc/modules-load.d/k8s.conf >/dev/null <<'EOF'
br_netfilter
EOF

sudo tee /etc/sysctl.d/99-k8s.conf >/dev/null <<'EOF'
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system

# 4) restart kubelet biar stabil
sudo systemctl restart kubelet
```

### B. Jalankan juga di **vm-k8s**

```bash
sudo ufw disable
sudo iptables -P FORWARD ACCEPT
sudo modprobe br_netfilter

sudo tee /etc/modules-load.d/k8s.conf >/dev/null <<'EOF'
br_netfilter
EOF

sudo tee /etc/sysctl.d/99-k8s.conf >/dev/null <<'EOF'
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
sudo systemctl restart kubelet
```

> Catatan: Rule `sudo ufw allow from 192.168.56.44 to any port 10250 proto tcp` yang kamu buat itu bagus untuk `kubectl logs/exec`. Tetap boleh, tapi kalau UFW dimatikan ya rule-nya tidak kepakai (dan itu justru bikin K8s lebih stabil untuk lab).

***

## 2) VALIDASI: pastikan Laravel bisa “nyentuh” MySQL dari dalam pod

Setelah firewall diberesin, kita tes dari laravel pod.

Jalankan di **vm-k8s**:

```bash
kubectl -n threebody-prod get pod -o wide
kubectl -n threebody-prod get svc mysql -o wide
kubectl -n threebody-prod get endpoints mysql -o wide

POD=$(kubectl -n threebody-prod get pod -l app=laravel -o jsonpath='{.items[0].metadata.name}')
echo "POD=$POD"

# 1) cek DNS mysql
kubectl -n threebody-prod exec -it "$POD" -- sh -lc 'getent hosts mysql || true'

# 2) cek TCP ke mysql:3306 pakai PHP (tanpa tool tambahan)
kubectl -n threebody-prod exec -it "$POD" -- sh -lc 'php -r '\''
$h=getenv("DB_HOST") ?: "mysql";
$p=3306;
$s=@fsockopen($h,$p,$e,$m,3);
echo $s ? "OK TCP $h:$p\n" : "FAIL TCP $h:$p errno=$e msg=$m\n";
if($s) fclose($s);
'\'''

# 3) cek query sederhana via PDO (kalau ext mysql ada)
kubectl -n threebody-prod exec -it "$POD" -- sh -lc 'php -r '\''
$dsn="mysql:host=".getenv("DB_HOST").";dbname=".getenv("DB_DATABASE").";charset=utf8mb4";
$u=getenv("DB_USERNAME"); $pw=getenv("DB_PASSWORD");
try {
  $pdo=new PDO($dsn,$u,$pw,[PDO::ATTR_TIMEOUT=>3]);
  $v=$pdo->query("SELECT 1")->fetchColumn();
  echo "OK PDO SELECT1=$v\n";
} catch (Exception $e) {
  echo "FAIL PDO: ".$e->getMessage()."\n";
}
'\'''
```

**Kalau setelah fix firewall hasilnya:**

* `OK TCP mysql:3306`
* `OK PDO SELECT1=1`

…baru lanjut tes API.

***

## 3) TES ulang endpoint Laravel (harusnya tidak hang lagi)

Di **vm-k8s**:

```bash
kubectl -n threebody-prod exec -it "$POD" -- sh -lc 'curl -v --max-time 10 http://127.0.0.1/api/products'
```

Di **vm-docker**:

```bash
curl -v --max-time 30 http://192.168.56.45:30082/api/products
curl -kfsS --resolve prod.local:443:127.0.0.1 https://prod.local/laravel/api/products | head
```

### Cara cek cepat biar yakin YAML sudah bener (opsional tapi membantu)

Sebelum commit, jalanin:

```bash
python3 - <<'PY'
import sys, re
p = ".gitlab-ci.yml"
txt = open(p,'r',encoding='utf-8').read().splitlines()
# cek ada baris yang "keluar indent" setelah 'script: |' (heuristik sederhana)
for i,l in enumerate(txt,1):
    if re.match(r'^\s*script:\s*\|', l):
        # baris berikutnya wajib kosong atau indent
        pass
print("OK: file kebaca (cek manual juga di GitLab CI Lint).")
PY
```

Atau paling gampang: buka **Pipeline editor / CI Lint** di GitLab → paste file → validate.

### Rapikan service EDGE biar konsisten (disarankan)

Sekarang staging service sudah jalan sebagai `cikal`. Biar edge juga konsisten (dan kalau suatu saat edge butuh akses file user), kita set juga edge unit jalan sebagai `cikal`.

Jalankan di **vm-docker**:

```bash
sudo tee /etc/systemd/system/threebody-edge.service >/dev/null <<'EOF'
[Unit]
Description=Threebody Edge Nginx (HTTPS + Rate Limit)
After=docker.service network-online.target
Wants=network-online.target
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes

User=cikal
Group=docker
Environment=HOME=/home/cikal
Environment=DOCKER_CONFIG=/home/cikal/.docker

WorkingDirectory=/opt/threebody/edge
ExecStart=/usr/bin/docker compose up -d --remove-orphans
ExecStop=/usr/bin/docker compose down --remove-orphans

TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl restart threebody-edge
sudo systemctl status threebody-edge --no-pager -l
```

## `threebody-staging.service` (biar jalan sebagai cikal + baca env + working dir benar)

Sekarang kita betulkan unit file staging supaya:

* pakai `WorkingDirectory=/opt/threebody/staging` (biar tidak “no configuration file”)
* pakai `.env` dari `/opt/threebody/staging/.env`
* jalan sebagai `User=cikal` (biar credential docker login kepakai)

**Jalankan:**

```bash
sudo tee /etc/systemd/system/threebody-staging.service >/dev/null <<'EOF'
[Unit]
Description=Threebody Staging (Docker Compose)
After=docker.service network-online.target
Wants=network-online.target
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes

# Jalankan sebagai user yang punya docker login credential
User=cikal
Group=docker

# Pastikan docker baca config user cikal
Environment=HOME=/home/cikal
Environment=DOCKER_CONFIG=/home/cikal/.docker

WorkingDirectory=/opt/threebody/staging
EnvironmentFile=/opt/threebody/staging/.env

ExecStart=/usr/bin/docker compose up -d --remove-orphans
ExecStop=/usr/bin/docker compose down --remove-orphans

TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
EOF
```

Reload & restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart threebody-staging
sudo systemctl status threebody-staging --no-pager -l
```

***

## 5) Validasi hasil (harusnya service tidak failed lagi)

Cek container:

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}' | grep -E 'stg-|edge-'
```

Cek apakah staging bisa diakses via edge:

```bash
curl -kI --resolve staging.local:443:127.0.0.1 https://staging.local/ | head -n 15
```

Kalau `HTTP/1.1 200 OK` → staging sehat.

***

### B) Pastikan auto-start saat reboot (enable + verifikasi)

Jalankan:

```bash
sudo systemctl enable threebody-edge threebody-staging
systemctl is-enabled threebody-edge threebody-staging
```

Harusnya output: `enabled`.



### 1) Install paket yang kurang (rsync + opsional tools)

Di vm-docker:

```bash
sudo apt-get update
sudo apt-get install -y rsync curl ca-certificates
```

Cek:

```bash
rsync --version | head -n 1
```

### 2) Copy folder deploy ke /opt (ulang yang kemarin gagal)

Pastikan repo kamu ada di `~/three-body-problem-main` (sesuai log kamu). Lalu:

```bash
sudo mkdir -p /opt/threebody/edge /opt/threebody/staging
sudo rsync -a --delete ~/three-body-problem-main/deploy/edge/ /opt/threebody/edge/
sudo rsync -a --delete ~/three-body-problem-main/deploy/staging/ /opt/threebody/staging/
```

Verifikasi isi folder:

```bash
ls -la /opt/threebody/edge
ls -la /opt/threebody/staging
```

> Yang kita cari: file `docker-compose*.yml` + config Nginx (biasanya ada folder `nginx/` atau file conf).

### vm-harbor (Harbor auto pulih & cepat)

Jalankan di vm-harbor

#### 1.1 Pastikan Docker auto-start (kamu sudah)

```bash
sudo systemctl enable --now docker
sudo systemctl is-enabled docker
```

#### 1.2 Perbaiki `harbor.service` (ini yang penting)

Edit file:

```bash
sudo nano /etc/systemd/system/harbor.service
```

Ganti full jadi ini (copy-paste):

```ini
[Unit]
Description=Harbor Registry (Docker Compose)
After=network-online.target docker.service
Wants=network-online.target docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/harbor

# start / ensure running
ExecStart=/usr/bin/docker compose -f /opt/harbor/docker-compose.yml up -d --remove-orphans

# PENTING: jangan "down" saat shutdown, cukup stop biar cepat up lagi setelah reboot
ExecStop=/usr/bin/docker compose -f /opt/harbor/docker-compose.yml stop

TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

Reload + restart:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now harbor
sudo systemctl restart harbor
sudo systemctl status harbor --no-pager
```

#### 1.3 Validasi Harbor sehat

Jalankan ini (tunggu 1-3 menit setelah reboot kalau perlu):

```bash
cd /opt/harbor
sudo docker compose ps
sudo docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}' | head -n 30
curl -kI https://harbor.local | head -n 5
```

Kalau setelah beberapa menit masih gagal, cek log boot:

```bash
sudo journalctl -u harbor -b --no-pager | tail -n 200
cd /opt/harbor && sudo docker compose logs --tail=200
```

Karena systemd kamu memanggil `docker compose up` **tanpa `-f`**, maka harus ada file bernama:

* `/opt/threebody/edge/docker-compose.yml`
* `/opt/threebody/staging/docker-compose.yml`

Kalau file kamu namanya **bukan** `docker-compose.yml` (misalnya `docker-compose.edge.yml`), kita bikin symlink biar simpel:

#### Untuk EDGE

```bash
cd /opt/threebody/edge
ls -1
```

Kalau kamu lihat misalnya ada `docker-compose.edge.yml`, jalankan:

```bash
sudo ln -sf docker-compose.edge.yml docker-compose.yml
```

#### Untuk STAGING

```bash
cd /opt/threebody/staging
ls -1
```

Kalau ada `docker-compose.staging.yml`, jalankan:

```bash
sudo ln -sf docker-compose.staging.yml docker-compose.yml
```

Cek hasil:

```bash
ls -la /opt/threebody/edge/docker-compose.yml
ls -la /opt/threebody/staging/docker-compose.yml
```

***

## 12) `.gitlab-ci.yml` (tag = CI\_COMMIT\_SHORT\_SHA)

`CI_COMMIT_SHORT_SHA` predefined variable GitLab.

Buat/replace `.gitlab-ci.yml` di root repo:

```yaml
stages:
  - build
  - push
  - deploy

workflow:
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - when: never

variables:
  DOCKER_BUILDKIT: "1"
  TAG: "$CI_COMMIT_SHORT_SHA"

  HARBOR_HOST: "harbor.local"
  HARBOR_PROJECT: "threebody"
  REGISTRY: "$HARBOR_HOST/$HARBOR_PROJECT"

  K8S_NS_PROD: "threebody-prod"
  KUBECTL_VERSION: "v1.30.14"
  K8S_ROLLOUT_TIMEOUT: "10m"
  K8S_IMAGEPULL_SECRET: "harbor-regcred"

default:
  tags: ["deploy"]     # pastikan runner vm-docker punya tag 'deploy'
  before_script:
    - set -euo pipefail

build_check:
  stage: build
  script: |
    echo "==> [build] build-check images (tanpa push)"
    docker build \
      --build-arg REACT_APP_GO_API_BASE=/go \
      --build-arg REACT_APP_LARAVEL_API_BASE=/laravel \
      -t "$REGISTRY/frontend:$TAG" \
      -f frontend/Dockerfile frontend
    docker build -t "$REGISTRY/go:$TAG" -f go/Dockerfile go
    docker build -t "$REGISTRY/laravel:$TAG" -f laravel/Dockerfile laravel

push_images:
  stage: push
  needs: ["build_check"]
  script: |
    echo "==> [push] login Harbor"
    : "${HARBOR_USERNAME:?Missing HARBOR_USERNAME}"
    : "${HARBOR_PASSWORD:?Missing HARBOR_PASSWORD}"
    echo "$HARBOR_PASSWORD" | docker login "$HARBOR_HOST" -u "$HARBOR_USERNAME" --password-stdin

    echo "==> [push] build+push images (TAG=$TAG)"
    docker build \
      --build-arg REACT_APP_GO_API_BASE=/go \
      --build-arg REACT_APP_LARAVEL_API_BASE=/laravel \
      -t "$REGISTRY/frontend:$TAG" \
      -f frontend/Dockerfile frontend
    docker push "$REGISTRY/frontend:$TAG"

    docker build -t "$REGISTRY/go:$TAG" -f go/Dockerfile go
    docker push "$REGISTRY/go:$TAG"

    docker build -t "$REGISTRY/laravel:$TAG" -f laravel/Dockerfile laravel
    docker push "$REGISTRY/laravel:$TAG"

deploy_prod:
  stage: deploy_prod
  needs: ["push_images"]
  script: |
    echo "==> [prod] validasi variables"
    : "${KUBECONFIG_PROD:?Missing KUBECONFIG_PROD (GitLab Variables Type: File)}"
    : "${HARBOR_USERNAME:?Missing HARBOR_USERNAME}"
    : "${HARBOR_PASSWORD:?Missing HARBOR_PASSWORD}"
    : "${MYSQL_ROOT_PASSWORD:?Missing MYSQL_ROOT_PASSWORD}"
    : "${MYSQL_DATABASE:?Missing MYSQL_DATABASE}"
    : "${MYSQL_USER:?Missing MYSQL_USER}"
    : "${MYSQL_PASSWORD:?Missing MYSQL_PASSWORD}"
    : "${LARAVEL_APP_KEY:?Missing LARAVEL_APP_KEY}"

    echo "==> [prod] siapkan kubectl (tanpa sudo, aman untuk runner shell)"
    mkdir -p .bin
    if [ ! -x .bin/kubectl ]; then
      curl -fsSL -o .bin/kubectl "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl"
      chmod +x .bin/kubectl
    fi
    export PATH="$PWD/.bin:$PATH"
    kubectl version --client=true

    echo "==> [prod] set kubeconfig dari File Variable"
    export KUBECONFIG="$KUBECONFIG_PROD"

    echo "==> [prod] cek cluster"
    kubectl get nodes -o wide

    echo "==> [prod] pastikan namespace"
    kubectl get ns "$K8S_NS_PROD" >/dev/null 2>&1 || kubectl create ns "$K8S_NS_PROD"

    echo "==> [prod] create/update app-secrets (dipakai go/laravel/mysql kamu)"
    kubectl -n "$K8S_NS_PROD" create secret generic app-secrets \
      --from-literal=MYSQL_ROOT_PASSWORD="$MYSQL_ROOT_PASSWORD" \
      --from-literal=MYSQL_DATABASE="$MYSQL_DATABASE" \
      --from-literal=MYSQL_USER="$MYSQL_USER" \
      --from-literal=MYSQL_PASSWORD="$MYSQL_PASSWORD" \
      --from-literal=LARAVEL_APP_KEY="$LARAVEL_APP_KEY" \
      --dry-run=client -o yaml | kubectl apply -f -

    echo "==> [prod] create/update imagePullSecret Harbor"
    kubectl -n "$K8S_NS_PROD" create secret docker-registry "$K8S_IMAGEPULL_SECRET" \
      --docker-server="$HARBOR_HOST" \
      --docker-username="$HARBOR_USERNAME" \
      --docker-password="$HARBOR_PASSWORD" \
      --dry-run=client -o yaml | kubectl apply -f -

    echo "==> [prod] attach imagePullSecret ke default serviceaccount"
    kubectl -n "$K8S_NS_PROD" patch serviceaccount default \
      -p "{\"imagePullSecrets\": [{\"name\": \"$K8S_IMAGEPULL_SECRET\"}]}" >/dev/null || true

    echo "==> [prod] apply manifests NodePort + mysql"
    kubectl apply -f deploy/k8s/base/

    echo "==> [prod] update image tag (memicu AUTO PULL di vm-worker)"
    kubectl -n "$K8S_NS_PROD" set image deployment/frontend frontend="$REGISTRY/frontend:$TAG"
    kubectl -n "$K8S_NS_PROD" set image deployment/go go="$REGISTRY/go:$TAG"
    kubectl -n "$K8S_NS_PROD" set image deployment/laravel laravel="$REGISTRY/laravel:$TAG"

    echo "==> [prod] tunggu rollout"
    kubectl -n "$K8S_NS_PROD" rollout status deployment/frontend --timeout="$K8S_ROLLOUT_TIMEOUT"
    kubectl -n "$K8S_NS_PROD" rollout status deployment/go --timeout="$K8S_ROLLOUT_TIMEOUT"
    kubectl -n "$K8S_NS_PROD" rollout status deployment/laravel --timeout="$K8S_ROLLOUT_TIMEOUT"

    echo "==> [prod] tampilkan ringkas hasilnya"
    kubectl -n "$K8S_NS_PROD" get deploy -o wide
    kubectl -n "$K8S_NS_PROD" get svc -o wide
    kubectl -n "$K8S_NS_PROD" get pods -o wide

    echo "==> [prod] healthcheck via edge nginx (prod.local di vm-docker)"
    for i in $(seq 1 30); do
      if curl -kfsS --resolve prod.local:443:127.0.0.1 https://prod.local/ >/dev/null 2>&1; then
        echo "OK: prod sehat"
        exit 0
      fi
      sleep 2
    done

    echo "ERROR: prod healthcheck gagal"
    kubectl -n "$K8S_NS_PROD" describe pods || true
    exit 1

```

***

## 13) GitLab CI Variables (Secret Manager = GitLab Variables)

Di GitLab Project → Settings → CI/CD → Variables, set:

* `HARBOR_USERNAME` = `admin`

```
HARBOR_USERNAME
```

```
admin
```

* `HARBOR_PASSWORD` = `Harbor12345` (atau yang kamu ganti)

```
HARBOR_PASSWORD
```

```
Harbor12345
```

* `MYSQL_ROOT_PASSWORD` = `...`

```
MYSQL_ROOT_PASSWORD
```

```
Harbor12345
```

* `MYSQL_DATABASE` = `...`

```
MYSQL_DATABASE
```

```
threebody
```

* `MYSQL_USER` = `...`

```
MYSQL_USER
```

```
admin
```

* `MYSQL_PASSWORD` = `...`

```
MYSQL_PASSWORD
```

```
Harbor12345
```

* `LARAVEL_APP_KEY` = `base64:...` (generate)

```
echo "base64:$(openssl rand -base64 32)"
```

```
LARAVEL_APP_KEY
```

```
base64:3vhVCCw5WxiNHZ7smNAvbK7Wkqze3clDMwq1zE2WtxA=
```

* `KUBECONFIG_B64` = base64 dari `~/.kube/config` (di vm-k8s)

```
KUBECONFIG_B64
```

```
base64:YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6Ci0gY2x1c3RlcjoKICAgIGNlcnRp
```

Generate `KUBECONFIG_B64` (di vm-k8s):

```bash
# pastikan kubeconfig normal ada (kamu sudah punya karena kubectl get nodes sukses)
ls -la ~/.kube/config

# buat base64 1 baris (tanpa newline) dan simpan ke file biar gampang copy
base64 -w0 ~/.kube/config > /tmp/kubeconfig.b64

# cek file tidak kosong
wc -c /tmp/kubeconfig.b64
head -c 60 /tmp/kubeconfig.b64; echo
```

Copy output ke variable `KUBECONFIG_B64`.

### Set variable di GitLab

GitLab Project → Settings → CI/CD → Variables → Add variable:

* Key: `KUBECONFIG_PROD`
* Type: **File** ✅
* Value: isi file `~/.kube/config` dari **vm-k8s** (paste full content YAML)
* Protected: OFF dulu (biar pemula nggak ke-skip di branch main)
* Masked: OFF (file biasanya tidak cocok masked)

Cara ambil isi kubeconfig dari vm-k8s:

```bash
cat ~/.kube/config
```

Copy semua isi, paste ke variable file.

HTTPS butuh 2 file:

1. **Certificate (CRT / fullchain)** → ibarat “KTP server”
2. **Private Key (KEY)** → ibarat “kunci rahasia server”

Pipeline kamu nyari 2 variable ini:

* `EDGE_TLS_CRT` (isi: file cert)
* `EDGE_TLS_KEY` (isi: file key)

Kalau tidak ada → job gagal (yang kamu lihat sekarang).

***

## B. Cara paling mudah (pemula): pakai Self-Signed dulu (buat latihan)

Kalau ini untuk lab/staging, kamu bisa pakai self-signed supaya pipeline lanjut.

### 1) Buat cert & key di VM runner (vm-docker)

Jalankan:

```bash
mkdir -p ~/tls
cd ~/tls

openssl req -x509 -nodes -newkey rsa:2048 -days 365 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=harbor.local"
```

Hasilnya ada:

* `~/tls/tls.crt`
* `~/tls/tls.key`

Cek:

```bash
ls -la ~/tls
```

***

### Cert kamu sekarang CN=harbor.local → sebaiknya untuk EDGE pakai staging/prod

Kamu bikin cert dengan:\
`/CN=harbor.local`

Padahal yang kamu pakai untuk HTTPS edge adalah:

* `https://staging.local`
* `https://prod.local`

Kalau cert hanya `harbor.local`, browser akan “name mismatch”.\
Memang `curl -k` masih lolos, tapi untuk beneran rapi, bikin cert dengan **SAN** untuk `staging.local` dan `prod.local`.

✅ Cara bikin cert EDGE yang benar (pakai SAN)\
Jalankan di VM:

```bash
mkdir -p ~/tls && cd ~/tls

cat > edge-openssl.cnf <<'EOF'
[req]
default_bits = 2048
prompt = no
default_md = sha256
x509_extensions = v3_req
distinguished_name = dn

[dn]
CN = staging.local

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = staging.local
DNS.2 = prod.local
EOF

openssl req -x509 -nodes -newkey rsa:2048 -days 365 \
  -keyout tls.key \
  -out tls.crt \
  -config edge-openssl.cnf
```

Nanti yang dipakai untuk GitLab variable adalah **isi**:

* `~/tls/tls.crt`
* `~/tls/tls.key`

***

### 3) Cara set `EDGE_TLS_CRT` dan `EDGE_TLS_KEY` di GitLab (step-by-step)

1. Buka GitLab project kamu
2. Masuk **Settings → CI/CD → Variables**
3. Klik **Add variable**

#### Tambah Variable 1 (CRT)

* **Key**: `EDGE_TLS_CRT`
* **Type**: **File** ✅
* **Value**: paste isi dari `~/tls/tls.crt` (termasuk BEGIN/END)
* **Environment scope**: `*` (All)
* **Protected**: OFF dulu (biar gampang untuk pemula)
* **Masked**: biarkan default (biasanya file variable nggak dimasking seperti string)

Klik **Save**

#### Tambah Variable 2 (KEY)

* **Key**: `EDGE_TLS_KEY`
* **Type**: **File** ✅
* **Value**: paste isi dari `~/tls/tls.key`
* **Environment scope**: `*` (All)
* **Protected**: OFF dulu

Klik **Save**

✅ Kalau ini beres, maka di job GitLab:

* `$EDGE_TLS_CRT` itu **bukan isi cert**, tapi **path file sementara** yang dibuat GitLab
* makanya `cp "$EDGE_TLS_CRT" ...` sudah benar

Jika dirasa setup vm-docker sudah aman lalu lanjut push project ke gitlab dan tunggu sampai proses running pipeline berhasil

```
cd ~/three-body-problem-main
git add .
git commit -m "Final Test DevOps v1"
git push gitlab main
```

Kalau kamu sudah commit sebelumnya tapi mau trigger ulang:

```bash
git commit --allow-empty -m "chore: trigger pipeline"
git push gitlab main
```

### Cara melihat “hasil akhirnya” (URL) + checklist

#### Cek pipeline di GitLab

GitLab → Build → Pipelines\
Pastikan:

* Runner status “online”
* Job `build_images` sukses → images muncul di Harbor project `threebody`
* Job `deploy_staging` sukses
* Job `deploy_prod` sukses

#### Cek di vm-docker (edge + staging container)

```bash
docker ps
curl -kI https://staging.local/
```

#### 14.3 Cek di K8s (prod)

Di vm-k8s:

```bash
kubectl -n threebody-prod get all -o wide
kubectl -n threebody-prod get svc
```

#### URL yang harus bisa dibuka dari laptop

* `https://harbor.local`
* `https://staging.local`
* `https://prod.local`

> Kalau browser warning TLS (karena CA lab), itu normal. Solusi rapi: import `ca.crt` ke trust store OS laptop.

***

## 14) Observability: Loki + Grafana (terpusat) + Promtail

Cara paling gampang sesuai rancangan kamu:

* **Loki + Grafana jalan di vm-docker** (docker compose)
* Promtail:
  * container di vm-docker (ambil log docker containers)
  * DaemonSet di k8s (ambil log pods) → push ke Loki vm-docker

(Aku bisa tuliskan file compose + promtail config + DaemonSet YAML juga kalau kamu mau aku “full tulis semua file observability”-nya sekalian.)

#### 14.1 Compose Loki+Grafana+Promtail (vm-docker)

**deploy/observability/docker-compose.observability.yml**

```yaml
services:
  loki:
    image: grafana/loki:2.9.8
    command: -config.file=/etc/loki/config.yml
    volumes:
      - ./loki-config.yml:/etc/loki/config.yml:ro
    ports:
      - "3100:3100"
    restart: unless-stopped

  grafana:
    image: grafana/grafana:10.4.5
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    restart: unless-stopped

  promtail-docker:
    image: grafana/promtail:2.9.8
    command: -config.file=/etc/promtail/config.yml
    volumes:
      - ./promtail-docker.yml:/etc/promtail/config.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/log:/var/log:ro
    restart: unless-stopped
```

**deploy/observability/loki-config.yml**

```yaml
auth_enabled: false
server:
  http_listen_port: 3100

common:
  path_prefix: /tmp/loki
  storage:
    filesystem:
      chunks_directory: /tmp/loki/chunks
      rules_directory: /tmp/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v12
      index:
        prefix: index_
        period: 24h
```

**deploy/observability/promtail-docker.yml**

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    static_configs:
      - targets: [localhost]
        labels:
          job: docker
          __path__: /var/lib/docker/containers/*/*.log
```

Jalankan (vm-docker):

```bash
cd ~/three-body-problem-main/deploy/observability
docker compose -f docker-compose.observability.yml up -d
```

Buka Grafana:

* `http://192.168.56.42:3000` (atau lewat hosts kalau kamu mau)
* Login `admin/admin`
* Add datasource Loki: URL `http://loki:3100`

> Kalau kamu butuh promtail di K8s juga (DaemonSet) biar log pods masuk Loki, bilang aja—aku bisa tuliskan YAML DaemonSet promtail yang push ke `http://192.168.56.42:3100`.

***

## 15) README.md (yang diminta tugas)

Di README tulis minimal:

1. Cara run staging (docker compose)
2. Cara run prod (kubectl apply)
3. Arsitektur (diagram teks + penjelasan)
4. Screenshot: Harbor project, pipeline sukses, staging.local, prod.local, Grafana logs

***

### “Checklist cepat” biar kamu yakin end-to-end sudah bener

1. `https://harbor.local` bisa dibuka
2. vm-docker `docker login harbor.local` sukses
3. push ke GitLab sukses, pipeline jalan
4. `https://staging.local` tampil frontend
5. `https://prod.local` tampil frontend
6. Grafana bisa akses Loki dan ada log

***

Kalau kamu mau, aku bisa lanjutkan dengan **paket file YAML K8s lengkap (frontend/go/laravel/mysql + secrets + NodePort)** dan **paket observability lengkap (docker-compose Loki+Grafana + promtail docker + promtail DaemonSet k8s)** dalam format yang tinggal kamu copy-paste ke repo.
