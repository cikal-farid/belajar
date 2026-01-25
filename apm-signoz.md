# APM Signoz

Berikut **runbook final (lengkap & “langsung jalan”)** untuk install **SigNoz self-hosted (Docker Compose) di Ubuntu 22.04** pada VM kamu (**IP 192.168.56.23, RAM 6GB, 2 core**).

***

### 0) Asumsi & checklist sebelum mulai

#### 0.1 Resource & catatan penting

* SigNoz (single VM) **bisa jalan untuk POC/lab/dev**; minimum RAM yang disebut adalah **4GB** dan port **8080/4317/4318** harus terbuka.
* Di runbook PDF, rekomendasi “nyaman” untuk 1 VM itu lebih besar (RAM 8–16GB, vCPU 4). Kamu RAM 6GB & 2 core, jadi **bisa jalan**, tapi supaya minim risiko **OOM**, aku tambahkan langkah **SWAP** (aman, standar).

#### 0.2 Port yang dipakai

Buka inbound:

* **8080/tcp** → UI SigNoz
* **4317/tcp** → OTLP gRPC
* **4318/tcp** → OTLP HTTP\
  Catatan security: **jangan buka 4317/4318 ke internet publik**; batasi ke private network/VPN.

***

### 1) Update OS & install tools dasar (sebagai user `cikal`)

```bash
sudo apt update
sudo apt -y upgrade
sudo apt -y install git curl ca-certificates gnupg
```

#### (Sangat disarankan) Tambah SWAP 4GB biar nggak gampang OOM (RAM 6GB)

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
free -h
```

(Optional) reboot kalau kernel/upgrade besar:

```bash
sudo reboot
```

***

### 2) Install Docker Engine + Docker Compose plugin (cara repo resmi)

Langkah di bawah ini mengikuti pola yang ada di PDF: remove paket konflik, tambah repo Docker, install `docker-ce` + `docker-compose-plugin`.

#### 2.1 Bersihkan paket Docker versi distro (kalau ada)

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt-get remove -y $pkg
done
```

#### 2.2 Add repository Docker resmi

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

#### 2.3 Install Docker + Compose plugin, lalu tes

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo docker run --rm hello-world
docker version
docker compose version
```

#### 2.4 Supaya `cikal` bisa jalanin docker tanpa sudo (opsional tapi enak)

```bash
sudo usermod -aG docker $USER
newgrp docker
```

> Kalau `newgrp docker` bikin session “aneh”, cukup logout/login SSH sekali.

***

### 3) (Opsional) Firewall UFW — versi aman (private network)

Di PDF, contoh UFW: allow 22, 8080, 4317, 4318 lalu enable.\
Karena kamu pakai network 192.168.56.x, ini versi yang lebih aman: **4317/4318 hanya dari subnet lokal**.

```bash
sudo ufw allow 22/tcp
sudo ufw allow 8080/tcp

# Batasi OTLP hanya dari jaringan host-only / private (sesuaikan kalau beda subnet)
sudo ufw allow from 192.168.56.0/24 to any port 4317 proto tcp
sudo ufw allow from 192.168.56.0/24 to any port 4318 proto tcp

sudo ufw enable
sudo ufw status
```

(Pastikan port 22 sudah di-allow sebelum enable supaya tidak ke-lockout.)

***

### 4) Deploy SigNoz (Docker Compose) — pakai user `cikal` (tanpa user baru)

Di PDF, langkahnya: clone repo SigNoz, masuk `signoz/deploy/docker`, lalu `docker compose up -d --remove-orphans`.

```bash
cd ~
git clone -b main https://github.com/SigNoz/signoz.git
cd signoz/deploy/docker
docker compose up -d --remove-orphans
```

***

### 5) Verifikasi instalasi

#### 5.1 Pastikan container up

```bash
cd ~/signoz/deploy/docker
docker compose ps
```

Kalau ada yang restart-loop:

```bash
docker compose logs -f --tail=200
```

#### 5.2 Akses UI

Dari laptop/browser:

* `http://192.168.56.23:8080/`

***

### 6) Endpoint OTLP untuk aplikasi

Arahkan exporter OpenTelemetry ke:

* OTLP gRPC: `192.168.56.23:4317`
* OTLP HTTP: `http://192.168.56.23:4318`

Contoh env var (gRPC) yang dicontohkan di PDF:

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="http://192.168.56.23:4317"
export OTEL_EXPORTER_OTLP_PROTOCOL="grpc"
```

***

### 7) Tes end-to-end (tanpa instrument aplikasi dulu)

PDF punya cara “paling enak”: jalankan sample generator `signoz/otlp-python`, lalu hit endpoint untuk generate traces/logs.

#### 7.1 Jalankan generator

```bash
sudo docker rm -f otlp-python 2>/dev/null || true

sudo docker run -d --name otlp-python \
  -p 5002:5002 \
  --add-host signoz:host-gateway \
  -e OTLP_ENDPOINT="signoz:4317" \
  -e INSECURE="true" \
  signoz/otlp-python
```

#### 7.2 Generate traffic (buat muncul data)

```bash
curl "http://127.0.0.1:5002/?user=alice"
curl "http://127.0.0.1:5002/?user=bob"
```

#### 7.3 Validasi di UI

* **Traces Explorer** → time range **Last 15 mins** → harus muncul trace dari service sample
* **Logs Explorer** → time range **Last 15 mins** → harus muncul log

***

### 8) Operasional dasar

#### 8.1 Stop / Start stack

```bash
cd ~/signoz/deploy/docker
docker compose down
docker compose up -d
```

#### 8.2 Uninstall

Hapus container + network (data tetap ada di volume):

```bash
cd ~/signoz/deploy/docker
docker compose down --remove-orphans
```

⚠️ Hapus juga volume (ClickHouse/Postgres ikut hilang):

```bash
cd ~/signoz/deploy/docker
docker compose down -v --remove-orphans
```

***

### 9) Troubleshooting cepat (yang paling sering)

#### 9.1 UI tidak bisa dibuka di `http://192.168.56.23:8080`

* Pastikan port 8080 kebuka (UFW/SG)
* Cek container & log:

```bash
cd ~/signoz/deploy/docker
docker compose ps
docker compose logs -f --tail=200
```

#### 9.2 `docker compose` tidak ada

Pastikan package `docker-compose-plugin` terpasang dan cek:

```bash
docker compose version
```

#### 9.3 Lupa lokasi `docker-compose.yaml` SigNoz (tanpa tebak-tebakan)

Dari user `cikal`, jalankan:

```bash
sudo docker compose ls
```

Lihat kolom **CONFIG FILES**; lalu jalankan:

```bash
sudo docker compose -f /PATH_YANG_MUNCUL ps
```

Kalau `compose ls` kosong, cari file compose:

```bash
sudo find /home -maxdepth 5 -type f \( -name "docker-compose.yml" -o -name "docker-compose.yaml" \) | grep -i signoz
```

#### 9.4 VM sering OOM / lambat

Untuk 1 VM, kombinasi ClickHouse + collector memang bisa berat; PDF menyarankan naikkan RAM (8–16GB) dan pastikan SSD + cek disk free.\
Dengan VM kamu (6GB), **SWAP** di langkah 1 biasanya cukup membantu menghindari crash, walau performa bisa turun saat swap aktif.

***

Kalau kamu mau, kirim output 2 perintah ini dari VM (biar aku bisa pastikan semuanya “sehat” di kondisi 6GB RAM):

```bash
docker compose -f ~/signoz/deploy/docker/docker-compose.yaml ps
sudo docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}" | head -n 30
```
