# Penjelasan runbook

Di bawah ini aku jelaskan runbook kamu **dari awal sampai akhir** secara **step-by-step**, dengan fokus:

* **Apa tujuan tiap langkah**
* **Apa fungsi tiap command / flag penting**
* **Apa efeknya ke sistem**
* **Apa yang harus kamu lihat sebagai “tanda sukses” (gate/cek)**
* **Apa risiko/pitfall yang sering kejadian** (biar kamu paham kenapa runbook ini dibuat “stabil”)

Aku akan pakai struktur sama seperti runbook kamu: **A → B → C → D → E → F → (Addon restart)**.

***

### Gambaran besar arsitektur dan alur kerja

#### Target akhir

Host-only network `192.168.56.0/24`:

* **vm-docker (192.168.56.42)**\
  Role: **Docker host + Harbor registry (HTTP :8080) + Edge Nginx TLS (hit.local) + GitLab Runner**
* **vm-k8s (192.168.56.43)**\
  Role: **Kubernetes control-plane (kubeadm)** + (opsional) **Monitoring Loki+Grafana**
* **vm-worker (192.168.56.44)**\
  Role: **Kubernetes worker** tempat Pod aplikasi berjalan + NodePort (30080/30081/30082)

#### Jalur traffic aplikasi (runtime)

Kamu membangun pola “edge reverse proxy”:

```
User/Browser
   |
   | HTTPS https://hit.local
   v
Edge Nginx (vm-docker)
   |
   | proxy_pass ke NodePort di vm-worker
   v
Kubernetes Services NodePort (vm-worker)
   ├─ frontend :30080
   ├─ go       :30081
   └─ laravel  :30082
        |
        v
      MySQL (StatefulSet + hostPath PV di vm-worker)
```

#### Jalur CI/CD (build & deploy)

Kamu membangun jalur “build image → push ke registry → deploy ke cluster”:

```
Git push ke GitLab
   |
   v
GitLab Runner (vm-docker) executor shell
   |
   | docker build
   | docker push
   v
Harbor Registry (vm-docker, http://harbor.local:8080)
   |
   | K8s pull image (via imagePullSecret)
   v
Kubernetes cluster (vm-k8s + vm-worker)
```

Kenapa runner ditaruh di vm-docker? Karena vm-docker adalah tempat Docker daemon & Harbor berada, jadi build+push paling stabil (DNS lokal, jarak dekat, tidak tergantung internet untuk registry internal).

***

## A. Langkah wajib di semua VM (vm-docker, vm-k8s, vm-worker)

Tujuan bagian A: bikin **lingkungan lab stabil**. Ini penting banget, karena banyak error DevOps di lab muncul bukan dari Kubernetes/CI, tapi dari:

* auto update jalan sendiri
* service restart otomatis saat install paket
* DNS “kadang resolve kadang tidak”
* hostname/hosts tidak konsisten antar VM

***

### A1) Hardening: matikan auto update & cegah restart service otomatis

#### Command

```bash
sudo systemctl stop unattended-upgrades 2>/dev/null || true
sudo systemctl disable --now unattended-upgrades 2>/dev/null || true
```

**Penjelasan**

* `unattended-upgrades` itu service Ubuntu/Debian yang bisa **auto install update** di background.
* Bahayanya untuk lab: tiba-tiba package upgrade → versi berubah → service restart → “kok tiba-tiba error”.

**Detail flag**

* `2>/dev/null` = buang error output (biar runbook “bersih” kalau service tidak ada).
* `|| true` = walau command gagal, step tidak menghentikan runbook (fail-safe).

Lanjut:

```bash
sudo systemctl stop apt-daily.service apt-daily-upgrade.service 2>/dev/null || true
sudo systemctl disable --now apt-daily.timer apt-daily-upgrade.timer 2>/dev/null || true
```

**Penjelasan**

* `apt-daily` & `apt-daily-upgrade` adalah timer yang memicu update otomatis.
* Kamu matikan timer agar tidak ada “apt lock” pas kamu install sesuatu.

Terakhir:

```bash
sudo dpkg --configure -a || true
```

**Penjelasan**

* Kalau sebelumnya ada install yang ketahan / power off / apt ke-lock, `dpkg` bisa status “half-configured”.
* `dpkg --configure -a` merapikan supaya apt normal lagi.

#### needrestart: jangan auto restart service

```bash
if [ -f /etc/needrestart/needrestart.conf ]; then
  sudo cp -a /etc/needrestart/needrestart.conf /etc/needrestart/needrestart.conf.bak.$(date +%F-%H%M%S)
  sudo sed -i "s/^\s*\$nrconf{restart}.*/\$nrconf{restart} = 'l';/" /etc/needrestart/needrestart.conf || true
fi
```

**Penjelasan**

* `needrestart` kadang memutuskan service harus direstart setelah upgrade library.
* Mode `'l'` = **list only**: hanya kasih tahu apa yang perlu restart, tapi tidak memaksa restart.
* Ini penting agar kubelet/containerd/docker tidak restart tiba-tiba saat kamu sedang setup.

**Gate cek yang bagus (opsional)**

*   Pastikan tidak ada apt lock:

    ```bash
    ps aux | egrep 'apt|dpkg' | grep -v egrep
    ```

***

### A2) Set hostname

Contoh:

```bash
sudo hostnamectl set-hostname vm-docker
```

**Penjelasan**

* `hostname` dipakai banyak hal:
  * identitas node Kubernetes (label `kubernetes.io/hostname`)
  * `/etc/hosts` mapping
  * nodeAffinity PV MySQL kamu mengikat ke hostname `vm-worker`

Jadi hostname **harus tepat** sesuai runbook.

**Gate**

```bash
hostname
```

***

### A3) `/etc/hosts` wajib

Command:

```bash
sudo tee /etc/hosts >/dev/null <<'EOF'
127.0.0.1 localhost
192.168.56.42 vm-docker harbor.local hit.local
192.168.56.43 vm-k8s
192.168.56.44 vm-worker
EOF

getent hosts harbor.local hit.local vm-k8s vm-worker
hostname
```

**Kenapa ini penting**

* Lab host-only sering tidak punya DNS internal.
* Kamu butuh “domain konsisten” untuk:
  * registry: `harbor.local:8080`
  * edge: `hit.local`

**Makna command**

* `tee /etc/hosts` dengan heredoc = menulis file secara deterministik.
* `<<'EOF'` (quote) = isi tidak diexpand variable, jadi aman.
* `getent hosts` = cek resolusi via NSS (cara yang lebih “real” dibanding sekadar `ping`).

**Gate**

* Output `getent hosts` harus balik IP yang benar.

**Pitfall**

* Kalau kamu pakai NAT/bridged dan IP berubah, /etc/hosts harus diupdate.
* Kalau kamu menimpa `/etc/hosts` total (seperti ini), entri lama hilang. Itu oke untuk lab, tapi sadar efeknya.

***

### A4) DNS fix (systemd-resolved)

Command:

```bash
sudo tee /etc/systemd/resolved.conf >/dev/null <<'EOF'
[Resolve]
DNS=1.1.1.1 8.8.8.8
FallbackDNS=9.9.9.9 8.8.4.4
DNSStubListener=yes
EOF
```

**Penjelasan**

* Kamu memaksa DNS server yang stabil (Cloudflare + Google) dan fallback.
* `DNSStubListener=yes` artinya `systemd-resolved` tetap menyediakan stub.

Restart + flush:

```bash
sudo systemctl restart systemd-resolved
sudo resolvectl flush-caches
```

Lalu yang paling penting:

```bash
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

**Kenapa ini krusial**

* Banyak sistem default memakai `/etc/resolv.conf` menunjuk ke `127.0.0.53` (stub).
* Dalam beberapa kondisi (terutama CI runner / docker build / gitlab-runner), stub ini kadang bikin resolusi tidak stabil.
* Dengan symlink ini, `/etc/resolv.conf` berisi nameserver real (1.1.1.1, 8.8.8.8).

**Gate**

```bash
getent hosts gitlab.com altssh.gitlab.com registry-1.docker.io || true
```

Kalau ini resolve, berarti DNS sudah jalan.

**Pitfall**

* Kalau VM kamu memakai NetworkManager yang suka “rewrite resolv.conf”, kadang symlink berubah. Untuk lab biasanya aman.

***

## B. VM-DOCKER (192.168.56.42) — Docker + Harbor + Edge + GitLab Runner

Tujuan bagian B:

1. Pasang Docker untuk build image & jalankan Harbor/Edge
2. Pasang Harbor sebagai registry internal
3. Siapkan repo (manifests, Dockerfile, CI pipeline)
4. Jalankan edge nginx TLS untuk akses `https://hit.local`
5. Pasang GitLab runner agar CI/CD bisa jalan dari vm-docker

***

### B1) Install tools + Docker (+ UFW)

#### Install paket dasar

```bash
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl git nano unzip rsync openssl openssh-server
sudo apt install ufw -y
sudo apt install lsof -y
sudo systemctl enable --now ssh
```

**Kenapa paket ini dipakai**

* `ca-certificates, curl` → download script/binary via HTTPS (docker install, kubectl, helm)
* `git` → repo
* `nano` → edit file
* `unzip` → ekstrak project zip
* `rsync` → copy sync (opsional)
* `openssl` → bikin TLS self-signed untuk hit.local
* `openssh-server` → supaya bisa di-SSH & scp
* `ufw` → firewall sederhana
* `lsof` → cek port dipakai siapa

#### UFW

```bash
sudo ufw enable
...
sudo ufw allow 8080/tcp || true
sudo ufw allow ssh
sudo ufw allow OpenSSH
sudo ufw allow 22/tcp
sudo ufw reload || true
```

**Penjelasan**

* Kamu mengaktifkan firewall dan memastikan:
  * `8080/tcp` dibuka untuk Harbor UI/registry HTTP
  * SSH tetap bisa masuk

**Pitfall**

* Kalau UFW aktif tapi port NodePort/443 tidak dibuka (untuk edge), akses `https://hit.local` bisa bermasalah.
* Tapi di runbook edge memakai port 80/443 di vm-docker, biasanya aman kalau UFW mengizinkan.

#### Install Docker dari get.docker.com

```bash
curl -fsSL https://get.docker.com | sudo sh
```

**Penjelasan**

* Ini script resmi Docker untuk install cepat.
* Untuk lab, ini paling praktis.

Tambahkan user ke group docker:

```bash
sudo usermod -aG docker "$USER"
newgrp docker
```

**Kenapa**

* Tanpa ini, kamu harus `sudo docker ...` terus.
* `newgrp docker` membuat session shell baru supaya group docker langsung aktif (tanpa logout/login).

**Gate**

```bash
docker version
docker compose version
```

***

### B2) Docker allow insecure registry Harbor (HTTP)

Command:

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json >/dev/null <<'EOF'
{
  "insecure-registries": ["harbor.local:8080"]
}
EOF

sudo systemctl restart docker
docker info | egrep -i "insecure|registry" || true
```

**Konsep**

* Docker default mengharuskan registry pakai HTTPS (TLS).
* Harbor kamu jalan HTTP di `:8080`.
* Jadi kamu harus whitelist sebagai `insecure-registries`.

**Apa efeknya**

* Docker daemon akan mengizinkan `docker login harbor.local:8080` dan `docker push` via HTTP.

**Pitfall**

* Ini **hanya untuk lab**. Di production, registry wajib HTTPS.

***

### B3) Install Harbor (HTTP :8080)

#### Download offline installer

```bash
export HARBOR_VERSION="v2.14.1"
...
wget -O "harbor-offline-installer-${HARBOR_VERSION}.tgz" ...
tar -xzf ...
```

**Penjelasan**

* Offline installer berisi image-image Harbor dalam tarball, jadi tidak perlu pull banyak dari internet saat install.

Pindahkan ke /opt/harbor:

```bash
sudo rm -rf /opt/harbor
sudo mv harbor /opt/harbor
sudo chown -R "$USER:$USER" /opt/harbor
```

**Kenapa /opt**

* Praktik umum: aplikasi pihak ketiga di `/opt`.

#### Konfigurasi `harbor.yml`

Isi minimal:

```yaml
hostname: harbor.local
http:
  port: 8080

harbor_admin_password: Harbor12345
data_volume: /data/harbor
trivy:
  enabled: false
```

**Arti tiap bagian**

* `hostname: harbor.local`\
  → Harbor akan “menganggap dirinya” di domain itu (penting untuk URL yang dibuat Harbor).
* `http.port: 8080`\
  → Nginx internal Harbor listen 8080.
* “pastikan TIDAK ada https aktif”\
  → kamu sengaja memilih HTTP untuk lab, karena kamu pakai `insecure-registries`.
* `data_volume: /data/harbor`\
  → semua data Harbor (image registry, database, config) disimpan di host path ini.
* `trivy.enabled: false`\
  → Trivy scanner makan resource; untuk lab dimatikan biar ringan & stabil.

Install:

```bash
sudo mkdir -p /data/harbor
sudo ./prepare
sudo ./install.sh
```

**Kenapa ada `prepare`**

* Harbor generate config file docker compose dan nginx berdasarkan `harbor.yml`.

**Gate**

```bash
sudo docker compose -f /opt/harbor/docker-compose.yml ps
curl -fsSI http://harbor.local:8080/v2/ | head -n 1 || true
```

**Makna `/v2/`**

* Endpoint standar Docker Registry API.
* Kalau Harbor hidup, biasanya balas:
  * `HTTP/1.1 401 Unauthorized` (ini justru bagus → berarti endpoint registry hidup tapi butuh auth)

#### Di UI Harbor

Akses:

* `http://harbor.local:8080/`\
  Buat project: `threebody`

**Kenapa harus project**

* Image path kamu nanti: `harbor.local:8080/threebody/<service>:tag`

***

### B4) Deploy project + Edge Nginx (hit.local)

Bagian ini sebenarnya “menyiapkan repo” (kode, manifests, CI). Banyak step-nya adalah membuat file konfigurasi.

#### Ekstrak repo

```bash
unzip -q three-body-problem-final.zip
mv three-body-problem-final three-body-problem
cd ~/three-body-problem
```

**Penjelasan**

* `-q` quiet.
* rename folder biar pendek dan konsisten.

Cek file edge.conf:

```bash
ls -lah deploy/edge/nginx/conf.d/edge.conf
grep -n "proxy_pass" deploy/edge/nginx/conf.d/edge.conf
```

Ini “sanity check”: memastikan file ada dan ada `proxy_pass`.

#### Buat struktur folder

```bash
mkdir -p ~/three-body-problem/deploy/edge/nginx/conf.d
mkdir -p ~/three-body-problem/deploy/k8s
mkdir -p deploy/k8s/base deploy/k8s/jobs
```

**Kenapa**

* `deploy/edge` untuk edge nginx container
* `deploy/k8s/base` untuk manifest utama (service/deploy/sts)
* `deploy/k8s/jobs` untuk job migrasi

***

#### File 1: `deploy/edge/nginx/conf.d/edge.conf`

Isi:

```nginx
limit_req_zone $binary_remote_addr zone=api_rl:10m rate=5r/s;
```

**Penjelasan**

* Ini membuat “rate limit bucket”:
  * key: `$binary_remote_addr` (IP client)
  * storage: `10m` (shared memory 10MB)
  * rate: `5r/s` (5 request per detik per IP)

Server HTTP → redirect HTTPS:

```nginx
server {
  listen 80;
  server_name hit.local;
  return 301 https://$host$request_uri;
}
```

**Penjelasan**

* Semua request ke `http://hit.local` dipaksa ke HTTPS.
* `301` permanent redirect.

Server HTTPS:

```nginx
server {
  listen 443 ssl;
  server_name hit.local;

  ssl_certificate     /etc/nginx/certs/tls.crt;
  ssl_certificate_key /etc/nginx/certs/tls.key;
```

**Penjelasan**

* Edge nginx terminasi TLS menggunakan cert self-signed yang kamu generate nanti.
* Cert disimpan di container path `/etc/nginx/certs/...` lewat volume mount.

Routing:

1. Frontend

```nginx
location / {
  proxy_pass http://192.168.56.44:30080;
}
```

**Penjelasan penting**

* Ini meneruskan semua path `/...` ke NodePort frontend di worker.
* Tidak ada trailing slash di `proxy_pass`, sehingga URI diteruskan apa adanya.

2. Go API

```nginx
location /go/ {
  limit_req zone=api_rl burst=10 nodelay;
  proxy_pass http://192.168.56.44:30081/;
}
```

**Kenapa pakai trailing slash `/` di proxy\_pass**

* `location /go/` + `proxy_pass http://.../;` artinya prefix `/go/` akan “dipotong” dan diteruskan sebagai `/`.
* Jadi request:
  * `https://hit.local/go/api/products`
  * diteruskan ke:
  * `http://192.168.56.44:30081/api/products`

`limit_req ...`

* `burst=10` artinya boleh “numpuk” 10 request ekstra saat spike.
* `nodelay` artinya burst tidak di-delay, tapi langsung dilepas selama masih dalam burst.

3. Laravel API

```nginx
location /laravel/ {
  limit_req zone=api_rl burst=10 nodelay;
  proxy_pass http://192.168.56.44:30082/;
}
```

Sama konsepnya.

**Pitfall umum**

* Banyak orang salah trailing slash → path jadi dobel `/go/go/...` atau `/laravel/laravel/...`. Kamu sudah benar.

***

#### File 2: `deploy/edge/docker-compose.yml`

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
```

**Penjelasan**

* `ports` membuka port host vm-docker: 80 & 443.
* `volumes`:
  * config nginx dipasang read-only (`:ro`) supaya lebih aman dan tidak berubah dari dalam container.
  * cert TLS juga read-only.
* `restart: unless-stopped` memastikan container otomatis restart kalau Docker restart (tapi kamu juga buat systemd unit, itu lebih kuat).

Network:

```yaml
networks:
  edge-net:
    name: edge-net
```

Ini bikin network docker custom agar rapi (meski untuk 1 container saja sebenarnya opsional).

***

### K8s Manifests (deploy/k8s/base)

Sebelum masuk detail file, ingat: pipeline deploy akan:

* create namespace `threebody-prod`
* create secrets `app-secrets` (DB & APP\_KEY)
* create secret docker-registry untuk pull dari Harbor
* patch default ServiceAccount agar imagePullSecret dipakai default

Jadi file-file YAML ini sengaja **mengandalkan secret** yang dibuat dari CI, bukan hardcode.

***

#### File 3: `10-mysql.yaml` (MySQL headless + PV/PVC + StatefulSet)

**a) Service headless**

```yaml
kind: Service
spec:
  clusterIP: None
```

**Kenapa headless**

* Untuk StatefulSet, headless service memberi DNS per-pod:
  * `mysql-0.mysql` (ini kamu pakai di migrate job)
* Ini lebih stabil daripada Service biasa saat kamu butuh alamat pod spesifik.

**b) PV hostPath**

```yaml
kind: PersistentVolume
spec:
  hostPath:
    path: /data/threebody/mysql
    type: DirectoryOrCreate
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - vm-worker
```

**Penjelasan**

* `hostPath` artinya data MySQL disimpan di filesystem node worker.
* `DirectoryOrCreate` bikin folder otomatis kalau belum ada.
* `nodeAffinity` mengunci PV ini hanya valid di node `vm-worker`.\
  Ini penting karena hostPath cuma ada di node itu; kalau MySQL pindah node, datanya hilang.

**c) PVC**

PVC mengikat ke PV (`volumeName: mysql-pv-hostpath`) dan storageClass kosong supaya tidak pakai dynamic provisioning.

**d) StatefulSet**

```yaml
kind: StatefulSet
spec:
  serviceName: mysql
  replicas: 1
```

**Kenapa StatefulSet**

* MySQL butuh storage persisten dan identitas pod stabil.
* StatefulSet memberi pod bernama tetap: `mysql-0`.

**Init container fix-perms**

```yaml
initContainers:
  - name: fix-perms
    image: busybox:1.36
    ...
    chown -R 999:999 /var/lib/mysql
```

**Kenapa ini ada**

* hostPath `DirectoryOrCreate` biasanya dibuat `root:root`.
* Image MySQL official sering jalan sebagai user `999`.
* Kalau permission salah, MySQL crashloop (“Permission denied”).
* Init container jalan sebagai root untuk memperbaiki permission sebelum MySQL start.

**Liveness/Readiness probe**

* Keduanya pakai `tcpSocket: 3306`.
* Ini sederhana: kalau port 3306 terbuka, dianggap hidup/siap.
* Untuk lab cukup, meski tidak mengecek query.

***

#### File 4: `20-go.yaml` (Service NodePort + Deployment)

Service:

```yaml
type: NodePort
nodePort: 30081
```

**Penjelasan**

* NodePort bikin port terbuka di node worker (dan control-plane juga, tapi biasanya traffic masuk via worker).
* Ini yang diakses edge nginx: `192.168.56.44:30081`.

Deployment:

```yaml
replicas: 0
```

**Kenapa default 0**

* Kamu ingin “stop dulu” sampai MySQL siap & migrasi selesai.
* Ini strategi anti race condition.

Env DB:

* `DB_HOST: mysql` (service name dalam cluster)
* credentials dari secret `app-secrets`.

***

#### File 5: `30-laravel.yaml` (Nginx+PHP-FPM dalam 1 Pod)

Ini agak advanced tapi bagus untuk lab.

**a) ConfigMap `laravel-nginx-conf`**

Berisi config nginx yang serve `public/` dan meneruskan PHP ke `127.0.0.1:9000` (php-fpm).

**Kenapa `fastcgi_pass 127.0.0.1:9000`**

* Karena php-fpm container ada dalam Pod yang sama, jadi bisa akses via loopback.

**b) Service NodePort 30082**

Edge nginx akan proxy ke sini untuk path `/laravel/...`.

**c) Deployment replicas: 0 (default)**

Sama alasan: start setelah migrate sukses.

**d) Shared volume `emptyDir`**

```yaml
volumes:
  - name: app-shared
    emptyDir: {}
```

**Konsep**

* Kamu punya 2 container:
  * php-fpm (laravel code)
  * nginx (butuh file static dari public/)
* `emptyDir` jadi “folder bersama” di dalam Pod.

**e) initContainer `copy-app`**

```yaml
initContainers:
  - name: copy-app
    image: harbor.../laravel:latest
    cp -a /var/www/html/. /shared/
```

**Kenapa perlu copy**

* Nginx container `nginx:alpine` tidak punya source code laravel.
* Jadi initContainer menyalin code dari image laravel ke volume `emptyDir`, lalu:
  * php-fpm mount volume itu
  * nginx mount volume itu read-only

**Ini pattern yang bagus untuk lab** karena kamu tetap pakai image nginx standar.

**f) Nama container laravel harus “laravel”**

Kamu menulis:

> “nama HARUS "laravel" supaya kubectl set image kamu tetap jalan”

Betul. Karena di pipeline kamu:

```bash
kubectl set image deployment/laravel laravel="...:TAG" copy-app="...:TAG"
```

Kalau nama container beda, command ini gagal.

***

#### File 6: `40-frontend.yaml` (Service NodePort 30080 + Deployment)

Service NodePort `30080` adalah endpoint utama frontend.

Deployment replicas 1 (default). Kenapa frontend bisa start dulu?\
Karena frontend hanya UI; walau backend belum siap, UI masih bisa tampil (tapi tombol fetch mungkin error). Tapi di pipeline kamu justru start frontend terakhir, jadi makin aman.

***

### K8s Job: `laravel-migrate-job.yaml`

Job ini penting karena kamu mau “push pertama tidak error”.

#### Struktur job

* `ttlSecondsAfterFinished: 600`\
  Job akan auto dibersihkan 10 menit setelah selesai (biar cluster tidak kotor).
* `backoffLimit: 0`\
  Kalau gagal, tidak retry berkali-kali (fail-fast).
* `activeDeadlineSeconds: 1800`\
  Maksimum 30 menit, anti job nyangkut.

#### initContainer `wait-mysql`

Dia pakai image `mysql:8.0` untuk punya binary `mysql`.

Script:

```sh
until mysql -h "$DB_HOST"... -e "SELECT 1"; do sleep 2; done
```

**Kenapa DB\_HOST = `mysql-0.mysql`**

*   Kamu bilang:

    > lebih stabil daripada "mysql" headless: langsung ke pod statefulset
* Ini benar: kadang Service belum propagate cepat, tapi DNS pod-statefulset biasanya konsisten.

#### container migrate

* `image: __LARAVEL_IMAGE__`\
  Ini placeholder yang akan di-substitute oleh pipeline (sed).

Script migrasi:

1. Clear cache (anti config-cache nyangkut)
2. `php artisan migrate --force`
3. Seed hanya jika table `products` kosong

**Bagian seed idempotent**\
Kamu bikin logic:

* Cek `SELECT COUNT(*) FROM products`
* Kalau 0, jalankan seed
* Kalau >0, skip\
  Ini mencegah “5 jadi 10” karena seed double.

***

### ✅ Add-on Monitoring repo files (WAJIB untuk job monitoring\_install)

Kamu membuat file values untuk Helm chart.

#### 1) Loki values (SingleBinary)

Inti:

* `deploymentMode: SingleBinary` → semua komponen Loki dalam 1 deployment (ringan untuk lab)
* `auth_enabled: false` → tidak ada auth
* `persistence.enabled: false` + `emptyDir` → log hilang kalau Loki restart (kamu sudah kasih catatan ini)

#### 2) Promtail values

Promtail mengirim log ke:

```yaml
url: http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push
```

Itu endpoint internal cluster.

#### 3) Datasource Loki untuk Grafana (kps)

`kube-prometheus-stack` punya Grafana, kamu tambahkan datasource Loki.

#### 4) expose UI NodePort

Kamu buat service NodePort:

* Grafana 30030
* Prometheus 30090
* Loki gateway 30100

Di pipeline monitoring\_install kamu akhirnya “patch service yang sudah ada” (lebih rapi), tapi file ini tetap berguna sebagai referensi/manual apply.

***

### .dockerignore (frontend/go/laravel)

Tujuan `.dockerignore`:

* mempercepat build
* mengurangi “context size”
* menghindari file sensitif ikut ke image (misal `.env`)
* menghindari folder berat (`node_modules`, `vendor`)

Contoh `frontend/.dockerignore`:

* `node_modules` → jangan dibawa, karena npm install dilakukan di image stage.
* `.env` → jangan bocor ke image.

***

### Dockerfile: Frontend (React)

```dockerfile
FROM node:20 AS build
...
RUN npm ci
...
RUN npm run build
FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
```

**Konsep multi-stage**

* Stage 1 “build”: compile React jadi file static
* Stage 2 “runtime”: hanya nginx + static file (lebih kecil, lebih bersih)

**ARG/ENV**

```dockerfile
ARG REACT_APP_GO_API_BASE=/go
...
ENV REACT_APP_GO_API_BASE=$REACT_APP_GO_API_BASE
```

React CRA hanya membaca env yang prefix `REACT_APP_` saat build. Jadi kamu set base path API.

***

### Dockerfile: Go

```dockerfile
FROM golang:1.22 AS builder
...
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o server .
FROM gcr.io/distroless/static:nonroot
...
USER nonroot:nonroot
```

**Penjelasan**

* `CGO_ENABLED=0` bikin binary statically linked → cocok untuk distroless.
* `distroless` artinya image runtime sangat minimal (lebih aman, lebih kecil).
* `nonroot` meningkatkan security.

***

### Dockerfile: Laravel (PHP-FPM Alpine + Composer vendor)

Ini 3 stage:

1. `base` runtime + PHP extension
2. `vendor` install composer dependencies
3. `runtime` final image

Kelebihan:

* build deps dihapus (`apk del .build-deps`) → image lebih kecil.
* composer cache dibersihkan.
* permission storage dan cache diperbaiki.

***

### Frontend `App.js` (penyesuaian endpoint)

Tujuan perubahan file ini:

* UI bisa “hit” **dua backend** lewat edge nginx:
  * Go: `/go/api/products`
  * Laravel: `/laravel/api/products`
* Menggunakan env variable agar flexible.

Bagian penting:

```js
const GO_API_BASE = (process.env.REACT_APP_GO_API_BASE || "/go").replace(/\/$/, "");
```

* memastikan tidak ada trailing slash ganda.

Normalize payload:

* Karena response Go & Laravel bisa beda bentuk, UI tetap bisa menampilkan list produk.

Ini membuat demo kamu lebih “tahan banting”.

***

### `.gitignore`

Poin penting:

* `.bin/` karena CI download kubectl/helm di situ.
* `deploy/edge/certs/tls.*` supaya cert lokal tidak ikut repo.

***

## GitLab CI: `.gitlab-ci.yml` (ini inti automation)

Aku jelasin job satu per satu, karena ini biasanya yang paling bikin orang “ohh ngerti”.

### Struktur global

Stages:

```yaml
stages:
  - build
  - push
  - deploy
  - monitoring
```

Workflow rules:

```yaml
workflow:
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - when: never
```

**Artinya**

* Pipeline hanya berjalan kalau commit di branch `main`.
* Kalau branch lain, tidak jalan (mencegah trigger tidak sengaja).

Variables penting:

* `TAG: "$CI_COMMIT_SHORT_SHA"`\
  Ini tag unik per commit.
* `HARBOR_HOST`, `REGISTRY`\
  Membentuk path image `harbor.local:8080/threebody/...`
* `KUBECTL_VERSION` & `HELM_VERSION`\
  Pipeline download binary tertentu biar stabil.

Default tags:

```yaml
default:
  tags: ["deploy"]
```

Artinya job akan di-pick runner yang punya tag `deploy` (runner kamu).

***

### Job 1: build\_check (stage build)

Tujuan: memastikan Dockerfile valid & build sukses **tanpa push**.

Kenapa berguna?

* Kalau build gagal, kamu stop di awal, tidak buang waktu push/deploy.

***

### Job 2: push\_images (stage push)

Bagian penting:

#### Sanity check Harbor

```sh
curl -fsSI "http://$HARBOR_HOST/v2/"
```

Ini memastikan endpoint registry up.

#### docker login insecure

```sh
echo "$HARBOR_PASSWORD" | docker login "$HARBOR_HOST" -u "$HARBOR_USERNAME" --password-stdin
```

Lebih aman daripada password di CLI arg.

#### Build + push tagged commit

Kamu build 3 image: frontend, go, laravel.\
Lalu tag `:latest` juga.

**Kenapa push `latest` juga**

* Manifest K8s kamu refer `:latest` (default).
* Tapi pipeline deploy akan `kubectl set image` ke tag commit SHA. Jadi `latest` tidak wajib untuk deploy, tapi berguna untuk debugging/manual run.

***

### Job 3: deploy (stage deploy)

Ini yang paling “DevOps banget” karena kamu mengatasi semua race condition umum.

#### Validasi variables (fail-fast)

```sh
: "${KUBECONFIG_PROD:?Missing KUBECONFIG_PROD ...}"
```

**Penjelasan**

* `:${VAR:?msg}` adalah shell pattern: kalau VAR kosong, langsung error dengan pesan.
* Ini mencegah pipeline jalan “setengah” dan bikin cluster kacau.

#### Siapkan kubectl portable

Download kubectl binary ke `.bin/kubectl`.\
Ini pattern “self-contained pipeline”.

#### Set kubeconfig dari File variable

```sh
export KUBECONFIG="$KUBECONFIG_PROD"
chmod 600 "$KUBECONFIG"
```

**Kenapa File variable**

* Lebih aman, tidak perlu paste kubeconfig dalam plaintext variable.
* `chmod 600` mencegah warning kubeconfig terlalu terbuka.

#### Cek cluster

```sh
kubectl get nodes -o wide
```

Kalau ini gagal, berarti:

* network runner → cluster bermasalah
* kubeconfig salah
* API server down

#### Namespace & ServiceAccount gate

Kamu melakukan:

* create namespace jika belum ada
* tunggu `serviceaccount default` ada (anti race)

Ini penting karena kadang namespace baru dibuat tapi SA belum ready, lalu patch SA gagal.

#### Buat `app-secrets`

```sh
kubectl create secret generic app-secrets ... --dry-run=client -o yaml | kubectl apply -f -
```

**Kenapa pattern ini bagus**

* Idempotent: bisa dijalankan berulang.
* Tidak error “AlreadyExists”.

Isi secret:

* MYSQL root password
* DB name/user/pass
* Laravel APP\_KEY

#### Buat imagePullSecret Harbor

```sh
kubectl create secret docker-registry "$K8S_IMAGEPULL_SECRET" ...
```

Ini membuat secret tipe `kubernetes.io/dockerconfigjson`.

#### Patch ServiceAccount default

```sh
kubectl patch serviceaccount default -p '{"imagePullSecrets":[{"name":"..."}]}'
```

**Kenapa patch SA default**

* Supaya semua pod di namespace itu otomatis bisa pull dari Harbor tanpa perlu menambahkan `imagePullSecrets` di tiap manifest.

Kamu bahkan verifikasi:

```sh
kubectl get sa default -o jsonpath='{.imagePullSecrets[*].name}'
... | grep -wq "$K8S_IMAGEPULL_SECRET"
```

Ini gate yang bagus.

#### Apply manifests

```sh
kubectl apply -f deploy/k8s/base/
```

Ini membuat resource ada (mysql sts, svc, deploy go/laravel/frontend dll).

#### STOP semua app dulu

Kamu scale frontend/laravel/go ke 0, delete pod yang mungkin sudah jalan.

**Tujuan**

* Pastikan MySQL benar-benar siap dulu sebelum backend start.
* Mencegah crashloop karena DB belum ada.

#### Tunggu MySQL rollout + Ready + mysqladmin ping

Ini 3 layer gate:

1. `rollout status` memastikan sts update sukses
2. `kubectl wait Ready pod/mysql-0`
3. `mysqladmin ping` memastikan mysqld benar-benar siap menerima koneksi

Ini jauh lebih stabil daripada cuma “pod Running”.

#### Ensure DB + grant user (idempotent)

Kamu menjalankan SQL:

* `CREATE DATABASE IF NOT EXISTS`
* `CREATE USER IF NOT EXISTS`
* `ALTER USER` (memastikan password sesuai)
* `GRANT ...`
* `FLUSH PRIVILEGES`

**Kenapa ini bagus**

* Menjamin database benar-benar ada sebelum migrate.
* Menghindari “Access denied” karena user belum di-grant.

#### Update image ke TAG commit

```sh
kubectl set image deployment/frontend frontend="$REGISTRY/frontend:$TAG"
...
kubectl set image deployment/laravel laravel="...:$TAG" copy-app="...:$TAG"
```

**Kenapa set image setelah mysql siap**

* Karena begitu pod start, dia akan pakai image baru.
* Kalau DB belum siap, laravel/go bisa error. Kamu menghindari itu.

#### Jalankan migrate job

Kamu:

* delete job lama
* apply job baru dari template yang sudah di-substitute image tag

Fail-fast:

* tunggu condition=failed 60 detik (kalau job cepat gagal, langsung stop)
* tunggu complete max 30 menit

#### Start aplikasi urut

1. laravel
2. go
3. frontend

**Kenapa urut ini masuk akal**

* laravel sering butuh DB + migrate
* go butuh DB
* frontend terakhir agar UI “langsung berfungsi”

#### Healthcheck

Kamu cek:

* NodePort frontend `http://192.168.56.44:30080/`
* fallback ke edge `https://hit.local/` dengan `--resolve` ke 127.0.0.1

Jika gagal, kamu dump debug (pods/events/describe).

***

### Job 4: monitoring\_install (manual)

Tujuan: pasang stack:

* kube-prometheus-stack (Prometheus + Grafana + exporters)
* Loki
* Promtail

Kenapa `when: manual`

* Monitoring tidak selalu wajib untuk “deploy sukses”.
* Kamu bisa install setelah aplikasi stable.

Step penting:

* download helm binary lokal
* add repo helm
* `helm upgrade --install ... --wait --timeout 20m`\
  `--wait` ini gate: helm tunggu semua ready.

Expose NodePort via patch service:

* grafana 30030
* prometheus 30090
* loki gateway 30100

Lalu print admin password Grafana dari secret.

***

### B: TLS cert untuk edge (hit.local)

Command:

```bash
openssl req -x509 -nodes -newkey rsa:2048 -days 3650 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=hit.local"
```

**Penjelasan**

* `-x509` = buat self-signed cert
* `-nodes` = private key tidak dipassword (biar nginx bisa start otomatis)
* `-days 3650` = 10 tahun
* `CN=hit.local` harus cocok dengan domain yang dipakai.

**Pitfall**

* Browser akan warning “Not secure” karena self-signed. Untuk lab oke.

***

### B: Naikkan edge

```bash
sudo docker compose up -d
sudo docker exec -it edge-nginx nginx -t
```

**Kenapa `nginx -t` di dalam container**

* Yang jalan adalah nginx container, bukan nginx host.
* `-t` memastikan config valid (kalau salah, kamu cepat tahu).

Tes:

```bash
curl -vk --resolve hit.local:443:127.0.0.1 https://hit.local/ ...
```

**Kenapa pakai `--resolve`**

* Memaksa `hit.local` resolve ke 127.0.0.1 pada host itu sendiri (vm-docker).
* Ini bypass DNS/hosts kalau lagi bermasalah, cocok untuk troubleshooting.

***

### B5) Install GitLab Runner + akses Docker

Install:

```bash
sudo apt-get install -y gitlab-runner
sudo systemctl enable --now gitlab-runner
```

Agar runner bisa pakai docker:

```bash
sudo usermod -aG docker gitlab-runner
sudo chown root:docker /var/run/docker.sock
sudo chmod 660 /var/run/docker.sock
```

**Penjelasan**

* `gitlab-runner` user harus bisa akses `/var/run/docker.sock`.
* Kalau tidak, pipeline akan error:
  * `permission denied` saat `docker build`

Gate:

```bash
sudo -u gitlab-runner docker ps
```

Kalau ini jalan, runner sudah bisa pakai docker.

***

### B6) SSH GitLab via port 443

Kamu membuat SSH key:

```bash
ssh-keygen -t ed25519 -C "vm-docker-gitlab" ...
```

Lalu config host:

```sshconfig
Host gitlab-443
  HostName altssh.gitlab.com
  User git
  Port 443
...
```

**Kenapa**

* Banyak jaringan membatasi port 22.
* `altssh.gitlab.com:443` adalah jalur alternatif GitLab untuk SSH.

Tes:

```bash
ssh -T git@gitlab-443
```

Kalau sukses biasanya muncul pesan “Welcome to GitLab”.

***

### Systemd unit untuk Harbor & Edge compose

Tujuan: **setelah VM restart**, Harbor & edge otomatis up.

#### Harbor unit

Type `oneshot` + `RemainAfterExit=yes`:

* artinya perintah start dijalankan sekali (compose up -d), lalu dianggap “aktif”.
* cocok untuk docker compose.

`After=docker.service` memastikan docker ready dulu.

#### Edge unit

`After=docker.service harbor-compose.service`

* artinya edge start setelah docker & harbor (meski edge tidak butuh harbor, urutan ini membuat boot lebih tertib).

Pitfall penting:

* `EDGE_DIR="/home/cikal/..."` harus sesuai user kamu. Kalau user beda, path salah → service fail.

Gate:

```bash
sudo systemctl status harbor-compose.service --no-pager
sudo systemctl status edge-compose.service --no-pager
```

***

## C. VM-K8S (192.168.56.43) — Kubernetes control-plane

Tujuan bagian C:

* persiapan kernel (networking + iptables)
* swap off (wajib untuk kubelet)
* install container runtime (containerd) dengan konfigurasi yang benar
* allow pull dari Harbor HTTP
* install kubeadm/kubelet/kubectl
* init cluster
* install CNI (Calico)

***

### C1) Kernel prereq + swap off

#### Swap off

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

**Kenapa**

* Kubernetes (kubelet) mensyaratkan swap off (kecuali kamu enable fitur tertentu).
* Swap bikin scheduling dan memory management unpredictable untuk k8s.

#### Load kernel modules

```bash
overlay
br_netfilter
```

**Penjelasan**

* `overlay` dibutuhkan untuk filesystem overlay (container).
* `br_netfilter` memastikan traffic bridge (pod network) lewat iptables.

#### Sysctl

```bash
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
```

**Penjelasan**

* memastikan iptables melihat traffic pod network
* forwarding untuk routing antar interface/pod.

**Gate**

```bash
swapon --show   # harus kosong
lsmod | egrep 'overlay|br_netfilter'
sysctl net.ipv4.ip_forward
```

***

### C2) Install + konfigurasi containerd

```bash
sudo apt-get install -y containerd
```

Generate config:

```bash
containerd config default | sudo tee /etc/containerd/config.toml
```

#### WAJIB: systemd cgroup

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
```

**Kenapa**

* Kubelet default modern memakai systemd cgroup driver.
* Kalau containerd masih pakai cgroupfs, sering terjadi mismatch → node NotReady / error resource.

#### WAJIB: registry config\_path

Kamu set:

* `config_path = "/etc/containerd/certs.d"`

**Tujuan**

* containerd akan membaca konfigurasi registry (hosts.toml) dari folder itu.
* Ini yang kamu gunakan untuk allow Harbor HTTP.

Restart & aktifkan:

```bash
sudo systemctl enable --now containerd
sudo systemctl restart containerd
```

**Gate**

```bash
sudo systemctl is-active containerd
```

Cek config tidak rusak:

```bash
sudo grep -n '\\1' /etc/containerd/config.toml || echo "OK"
```

Ini semacam “lint” untuk memastikan sed tidak menghasilkan karakter aneh.

***

### C3) Allow Harbor HTTP untuk containerd

Buat folder:

```bash
sudo mkdir -p /etc/containerd/certs.d/harbor.local:8080
```

Buat `hosts.toml`:

```toml
server = "http://harbor.local:8080"

[host."http://harbor.local:8080"]
  capabilities = ["pull", "resolve", "push"]
```

**Penjelasan**

* Secara default containerd menganggap registry harus HTTPS.
* Dengan ini, kamu bilang: “registry ini HTTP dan boleh pull”.

**Catatan**

* `push` capability sebenarnya tidak dibutuhkan oleh k8s node (k8s cuma pull). Tapi tidak masalah.

Restart:

```bash
sudo systemctl restart containerd
```

***

### C4) Install kubeadm/kubelet/kubectl v1.30

Kamu menambahkan repo `pkgs.k8s.io` untuk v1.30.

Lalu:

```bash
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

**Kenapa di-hold**

* Agar tidak auto-upgrade versi minor yang bisa bikin cluster mismatch.

***

### C5) `kubeadm init` + Calico

Init:

```bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.56.43 \
  --apiserver-cert-extra-sans=192.168.56.43,vm-k8s \
  --pod-network-cidr=192.168.0.0/16
```

**Penjelasan parameter**

* `--apiserver-advertise-address`\
  IP yang dipakai node lain untuk mengakses API server.
* `--apiserver-cert-extra-sans`\
  Tambahan SAN pada sertifikat API server. Ini mencegah error TLS saat akses via IP/hostname tertentu.
* `--pod-network-cidr=192.168.0.0/16`\
  CIDR untuk jaringan pod (harus cocok dengan CNI yang kamu pakai; Calico biasanya default ini juga).

Set kubeconfig untuk user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown ... $HOME/.kube/config
```

Install Calico:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

**Kenapa butuh CNI**

* Tanpa CNI, node akan NotReady karena networking pod belum siap.

Gate:

```bash
kubectl get nodes -o wide
watch -n 2 -d 'kubectl get nodes'
```

Tunggu sampai `Ready`.

Join command:

```bash
kubeadm token create --print-join-command --ttl 24h
```

Ini yang dipakai di worker.

***

### Catatan “hapus swap permanen”

Bagian swap remove yang kamu tulis (swapoff, edit fstab, rm /swap.img, chattr -i) itu tujuannya:

* memastikan setelah reboot swap tidak balik lagi

Ini bagus, karena kalau swap hidup lagi, kubelet bisa gagal start.

***

### Monitoring gate (opsional di runbook kamu)

Kamu menaruh gate seperti:

```bash
ss -lntp | egrep ':3000|:9090|:3100'
curl -fsS http://192.168.56.43:3000/login | head -n 1
...
```

Intinya:

* memastikan Grafana/Prometheus/Loki benar-benar listen dan ready.

***

## D. VM-WORKER (192.168.56.44) — Kubernetes worker + NodePort

Tujuan bagian D:

* worker node siap jalankan workload
* bisa pull image dari Harbor HTTP
* memiliki storage hostPath untuk MySQL
* NodePort bisa diakses oleh edge

***

### D1) Ulangi langkah C1–C4

Ini penting: worker juga perlu:

* swap off
* sysctl modules
* containerd + config systemd cgroup
* hosts.toml Harbor HTTP
* install kubeadm/kubelet/kubectl

Kalau ada `/swap.img`, hapus dan pastikan `swapon --show` kosong.

***

### D2) Join cluster

Jalankan command join dari control-plane:

```bash
sudo kubeadm join 192.168.56.43:6443 --token ... --discovery-token-ca-cert-hash sha256:...
```

Gate dari vm-k8s:

```bash
kubectl get nodes -o wide
```

Pastikan `vm-worker` status `Ready`.

***

### D3) Siapkan hostPath MySQL

Karena PV pakai `/data/threebody/mysql` dan MySQL berjalan sebagai uid 999:

```bash
sudo mkdir -p /data/threebody/mysql
sudo chown -R 999:999 /data/threebody/mysql || true
sudo chmod 700 /data/threebody/mysql || true
```

**Kenapa 700**

* MySQL datadir biasanya butuh permission ketat.

***

### D4) Firewall NodePort

Untuk lab, paling aman:

```bash
sudo ufw disable || true
```

Kalau mau tetap pakai UFW:

```bash
sudo ufw allow 30080:30082/tcp
```

**Kenapa ini penting**

* Edge nginx di vm-docker akses NodePort di vm-worker.
* Kalau firewall block, maka `https://hit.local` akan error 502/bad gateway.

***

## E. GitLab CI/CD

Tujuan bagian E:

* set secrets dan akses cluster di GitLab
* push pertama memicu pipeline build/push/deploy

***

### E1) GitLab Variables

Yang wajib kamu set (ini benar):

* Harbor credential
* MySQL credential
* Laravel APP\_KEY
* `KUBECONFIG_PROD` sebagai **File variable**

Kenapa kubeconfig harus file?

* Pipeline butuh file kubeconfig untuk `kubectl`.
* Lebih aman dan lebih kompatibel.

Ambil kubeconfig dari vm-k8s:

```bash
scp cikal@192.168.56.43:~/.kube/config ./kubeconfig-prod
```

***

### E2) Push pertama

Intinya:

* init repo
* set branch main
* remote pakai `gitlab-443`
* commit dan push

Command kamu:

```bash
git init
git config --global init.defaultBranch main
git branch -M main
git remote add origin git@gitlab-443:cikalfarid/three-body-problem-lab.git
...
git push -u origin main --force
```

**Penjelasan**

* `--force` dipakai supaya repo remote mengikuti state lokal (untuk lab awal). Di production biasanya dihindari.

**Catatan penting yang kamu tulis sudah benar**

> Jangan ubah manifest jadi `:${TAG}` karena Kubernetes tidak substitusi `${TAG}`.

Betul: K8s YAML tidak otomatis “render variable”. Makanya kamu melakukan `kubectl set image` di pipeline.

***

## F. Setelah deploy sukses: migrate/seed manual (opsional)

Kamu memberi command:

```bash
LARAVEL_POD=$(kubectl -n "$NS" get pod -l app=laravel -o jsonpath='{.items[0].metadata.name}')
kubectl -n "$NS" exec -it "$LARAVEL_POD" -- php artisan migrate --force
kubectl -n "$NS" exec -it "$LARAVEL_POD" -- php artisan db:seed --force || true
```

**Penjelasan**

* Ini cara manual kalau kamu butuh re-run.
* Tapi di pipeline kamu sudah ada job migrate, jadi bagian ini sebenarnya “sekali saja” atau untuk debug.

**Masalah seed dobel**\
Kamu sudah tahu:

* `db:seed` default tidak idempotent, bisa menggandakan data.

Solusi kamu: seed policy + opsi `migrate:fresh --seed` untuk reset lab. Itu benar.

***

## Add-on: “3 VM restart → Harbor & hit.local auto pulih”

Ini bagian reliability.

### A) Pastikan service enabled

* vm-docker: docker enabled
* vm-k8s & vm-worker: containerd + kubelet enabled

### B) systemd unit untuk compose (Harbor & Edge)

Ini membuat boot lebih deterministik dibanding hanya `restart: unless-stopped`.

### C) Checklist setelah restart

Kamu melakukan smoke test:

* `/v2/` harbor
* hit.local frontend
* hit.local go api
* hit.local laravel api
* k8s nodes/pods status

Ini adalah “operational checklist” yang bagus.

**Urutan boot aman**

1. vm-docker (Harbor & edge)
2. vm-k8s
3. vm-worker

Kenapa? Karena:

* cluster butuh runtime & registries saat scheduling/pull image.
* Kalau Harbor down saat pod start, ImagePullBackOff bisa terjadi.

***

## Kenapa runbook ini terasa “stabil” (inti desainnya)

Kalau aku rangkum “pola stabil” yang kamu pakai:

1. **DNS & hosts dipaksa konsisten** → mengurangi error random resolve.
2. **Auto update dimatikan** → versi tidak berubah diam-diam.
3. **Registry insecure di-whitelist di dua sisi**:
   * Docker daemon runner (push)
   * containerd node k8s (pull)
4. **Pipeline deploy punya “gate” berlapis**:
   * SA default ada
   * secret imagePullSecret benar-benar terpasang
   * MySQL rollout + Ready + mysqladmin ping
   * DB & user dibuat idempotent
   * migrate job fail-fast
   * backend start setelah migrate sukses
   * healthcheck NodePort + edge
5. **Restart resilience** via systemd unit compose + enable service runtime

Itu kombinasi yang biasanya membedakan runbook “jalan sekali” vs “jalan terus walau diulang”.

***

## Bonus: Cara belajar cepat dari runbook ini (biar benar-benar “nempel”)

Kalau kamu ingin memahami lebih cepat, kamu bisa latihan dengan pola berikut:

### 1) Validasi setiap layer sebelum naik ke layer berikutnya

* Layer OS/networking: `getent hosts ...`
* Layer Docker: `docker version`, `docker info`
* Layer Harbor: `curl -I http://harbor.local:8080/v2/`
* Layer K8s: `kubectl get nodes`
* Layer App internal: `kubectl get pods`, `kubectl logs`
* Layer Edge: `curl -k --resolve hit.local:443:127.0.0.1 https://hit.local/...`

### 2) Pahami “nama DNS internal K8s”

* Service: `mysql` → resolve ke Service ClusterIP/headless
* Pod statefulset: `mysql-0.mysql` → resolve langsung ke pod tertentu

### 3) Pahami “kenapa replicas 0”

Itu strategi deployment yang sengaja dibuat:

* resource dibuat dulu
* MySQL siap
* migrasi
* baru app dinaikkan

***

Kalau kamu mau, aku juga bisa bikin versi “catatan belajar” yang lebih ringkas tapi tetap lengkap: semacam **glosarium** (mis. `--dry-run=client` itu apa, `imagePullSecret` itu apa, `NodePort` itu apa, `headless service` itu apa) dan “alur troubleshooting” (kalau errornya ImagePullBackOff harus cek apa dulu, kalau 502 dari edge cek apa dulu, dll).
