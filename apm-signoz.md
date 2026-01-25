# APM Signoz

#### Apa itu “APM SigNoz”?

**SigNoz** adalah **platform observability open‑source** (sering dipakai sebagai **APM**) yang “OpenTelemetry‑native”. Artinya, SigNoz mengumpulkan **traces (distributed tracing/APM)**, **metrics**, dan **logs** (plus alerting) dalam satu UI, dengan data masuk lewat **OpenTelemetry Collector** dan disimpan terutama di **ClickHouse** untuk query yang cepat. ([SigNoz](https://signoz.io/?utm_source=chatgpt.com))

Kalau kamu fokus APM: SigNoz APM biasanya dipakai untuk:

* melihat **service map**, **latency p99**, **error rate**, dan analisis trace end‑to‑end (request mengalir ke service mana saja, bottleneck di mana). ([SigNoz](https://signoz.io/docs/instrumentation/overview/?utm_source=chatgpt.com))

Gambaran arsitektur sederhananya:

1. Aplikasi di‑instrument dengan **OpenTelemetry SDK/agent**
2. Kirim telemetry ke **SigNoz OTel Collector**
3. Collector menulis data ke **ClickHouse**
4. **SigNoz (backend+frontend)** men-query & menampilkan di UI + alerting ([SigNoz](https://signoz.io/docs/architecture/))

***

### Rekomendasi butuh berapa VM? Apakah 1 cukup?

**Jawaban praktisnya:**

* **1 VM cukup** untuk **POC/lab/dev** dan **workload kecil** (single-node, tanpa HA).
* Untuk **production**, “cukup” tergantung target: volume telemetry, retensi data, dan kebutuhan HA. Minimal production yang lebih nyaman biasanya **2 VM** (pisah storage ClickHouse vs komponen lain).
* Untuk **HA & scale**, bisa butuh beberapa node (ClickHouse shard/replica + ZooKeeper + beberapa collector). Di panduan capacity planning SigNoz (community), contoh deployment yang scalable memakai beberapa komponen dengan beberapa replika/node (misalnya collector 3 replika, ZooKeeper 3 node, ClickHouse multi-shard, dst). ([SigNoz](https://signoz.io/docs/setup/capacity-planning/community/resources-planning/))

#### Rekomendasi “tier” yang masuk akal

**Tier A — 1 VM (cukup untuk mulai)**

* Cocok: trial 1–2 minggu, internal kecil, atau monitoring beberapa service.
* Kelemahan: **tidak HA**; kalau VM mati, observability ikut mati.
* Minimal: SigNoz menyebut **minimal 4GB memory** untuk Docker dan pastikan port **8080/4317/4318** terbuka. ([SigNoz](https://signoz.io/docs/install/docker/))
* Saran realistis agar tidak “ngos-ngosan”:
  * **vCPU:** 4
  * **RAM:** 8–16 GB
  * **Disk:** 80–200 GB SSD (ClickHouse cepat “makan” disk kalau traces/logs banyak)

**Tier B — 2 VM (rekomendasi untuk production kecil-menengah)**

* VM#1: **ClickHouse** (disk & IOPS paling penting)
* VM#2: **SigNoz core + OTel Collector + PostgreSQL**
* Benefit: lebih mudah scaling (khususnya storage) dan performa lebih stabil.

**Tier C — HA / Growth**

* Mengarah ke multi-node: beberapa collector, ClickHouse shard/replica, ZooKeeper (atau keeper) terpisah, dsb. (lihat contoh kebutuhan resource per komponen di capacity planning). ([SigNoz](https://signoz.io/docs/setup/capacity-planning/community/resources-planning/))

> Catatan: SigNoz juga menyarankan memasang SigNoz di VM/cluster terpisah dari VM tempat aplikasi berjalan (bukan wajib, tapi direkomendasikan). ([SigNoz](https://signoz.io/docs/faqs/installation/))

***

## Runbook: Install SigNoz Self‑Hosted di VM Ubuntu 22.04 (Docker Compose)

Ini runbook paling “aman dan cepat” untuk VM tunggal. SigNoz menyediakan 2 cara: pakai **install script** atau jalankan **docker compose up** sendiri. ([SigNoz](https://signoz.io/docs/install/docker/))\
Aku tulis detail untuk opsi “manual” dulu (lebih terkontrol), lalu ada opsi “install script” sebagai alternatif.

***

### 0) Checklist sebelum mulai

#### 0.1. Kebutuhan VM & akses

* Ubuntu **22.04 LTS (Jammy)** (Docker resmi support Jammy 22.04). ([Docker Documentation](https://docs.docker.com/engine/install/ubuntu/?utm_source=chatgpt.com))
* Punya user dengan sudo.
* Disk SSD direkomendasikan (ClickHouse).

#### 0.2. Port yang perlu dibuka

Di VM tempat SigNoz berjalan, pastikan inbound:

* **8080/tcp** → UI SigNoz
* **4317/tcp** → OTLP gRPC (traces/metrics/logs dari app/collector)
* **4318/tcp** → OTLP HTTP ([SigNoz](https://signoz.io/docs/install/docker/))

> Security tip: jangan buka 4317/4318 ke internet publik. Batasi ke private network / VPN / security group internal.

***

### 1) Update OS & install tool dasar

```bash
sudo apt update
sudo apt -y upgrade
sudo apt -y install git curl ca-certificates gnupg
```

(Opsional tapi bagus) reboot:

```bash
sudo reboot
```

***

### 2) (Opsional tapi direkomendasikan) Buat user khusus “signoz”

Kalau VM masih fresh dan kamu ingin rapih:

```bash
sudo adduser signoz
sudo usermod -aG sudo signoz
sudo -iu signoz
```

SigNoz juga mengingatkan untuk **pakai non-root user** saat clone repo agar tidak kena problem permission. ([SigNoz](https://signoz.io/docs/install/docker/))

***

### 3) Install Docker Engine + Docker Compose plugin (cara resmi)

Ikuti langkah repo resmi Docker (ringkas tapi sesuai doc):

#### 3.1. Bersihkan paket Docker versi distro (kalau ada)

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt-get remove -y $pkg
done
```

#### 3.2. Add repository Docker resmi

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
```

Sumber rujukan langkah install Docker Engine di Ubuntu ada di dokumentasi resmi Docker. ([Docker Documentation](https://docs.docker.com/engine/install/ubuntu/?utm_source=chatgpt.com))

#### 3.3. Install Docker + Compose plugin

```bash
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Dokumentasi Compose plugin di Linux: ([Docker Documentation](https://docs.docker.com/compose/install/linux/?utm_source=chatgpt.com))

#### 3.4. Tes Docker jalan

```bash
sudo docker run --rm hello-world
```

#### 3.5. Supaya bisa jalanin docker tanpa sudo (opsional)

```bash
sudo usermod -aG docker $USER
newgrp docker
docker version
docker compose version
```

***

### 4) (Opsional) Set firewall UFW

Kalau kamu pakai UFW:

```bash
sudo ufw allow 22/tcp
sudo ufw allow 8080/tcp
sudo ufw allow 4317/tcp
sudo ufw allow 4318/tcp
sudo ufw enable
sudo ufw status
```

> Pastikan port SSH (22) sudah di-allow sebelum `ufw enable`, biar tidak lockout.

***

### 5) Deploy SigNoz dengan Docker Compose

#### 5.1. Clone repo SigNoz

Dari doc resmi SigNoz:

```bash
git clone -b main https://github.com/SigNoz/signoz.git
cd signoz/deploy/
```

([SigNoz](https://signoz.io/docs/install/docker/))

#### 5.2. Jalankan SigNoz (cara manual docker compose)

```bash
cd docker
docker compose up -d --remove-orphans
```

Ini sesuai instruksi “Install SigNoz Using Docker Compose”. ([SigNoz](https://signoz.io/docs/install/docker/))

> Pertama kali pull image bisa agak lama tergantung koneksi.

***

### 6) Verifikasi instalasi

#### 6.1. Pastikan container up

Jalankan dari folder `signoz/deploy/docker`:

```bash
docker compose ps
```

Kalau ada yang restart-loop, lihat log:

```bash
docker compose logs -f --tail=200
```

#### 6.2. Akses UI

Buka di browser:

* `http://<IP-VM>:8080/` ([SigNoz](https://signoz.io/docs/install/docker/))

Kalau kamu akses dari laptop:

* pastikan security group / firewall cloud mengizinkan port 8080.

***

### 7) Konfigurasi aplikasi agar mengirim traces/metrics/logs ke SigNoz (ringkas)

Dari sisi aplikasi (yang sudah instrument OpenTelemetry), endpoint umumnya:

* OTLP gRPC: `<IP-VM>:4317`
* OTLP HTTP: `http://<IP-VM>:4318`

Port ini memang diminta dibuka oleh SigNoz saat instal. ([SigNoz](https://signoz.io/docs/install/docker/))

Contoh (umum, banyak SDK support env var):

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="http://<IP-VM>:4317"
export OTEL_EXPORTER_OTLP_PROTOCOL="grpc"
```

(Detailnya beda tiap bahasa/framework, tapi konsepnya sama: arahkan exporter ke 4317/4318.)

***

### 8) Operasional dasar

#### 8.1. Stop / Start stack

Stop:

```bash
cd ~/signoz/deploy/docker
docker compose down
```

Start lagi:

```bash
cd ~/signoz/deploy/docker
docker compose up -d
```

#### 8.2. Uninstall (HATI-HATI: bisa hapus data)

Kalau kamu mau hapus container + network:

```bash
docker compose down --remove-orphans
```

Kalau mau sekalian hapus volume (data ClickHouse/Postgres ikut hilang):

```bash
docker compose down -v --remove-orphans
```

***

## Alternatif: Install pakai `install.sh` (lebih otomatis)

SigNoz juga menyediakan cara “1 command” via script yang:

* cek environment,
* install Docker Engine & Docker Compose di Linux,
* lalu menjalankan `docker compose up`. ([SigNoz](https://signoz.io/docs/install/docker/))

Langkahnya (sesuai doc):

```bash
git clone -b main https://github.com/SigNoz/signoz.git && cd signoz/deploy/
./install.sh
```

([SigNoz](https://signoz.io/docs/install/docker/))

Kalau kamu sudah install Docker sendiri, biasanya aku lebih prefer **cara manual** (bagian runbook di atas) supaya tidak ada “kejutan” dari script.

***

### Troubleshooting cepat (yang paling sering)

1. **UI tidak bisa dibuka di `<IP>:8080`**

* Cek port 8080 terbuka (SG/NACL/UFW)
*   Cek container:

    ```bash
    docker compose ps
    ```
*   Cek log:

    ```bash
    docker compose logs -f --tail=200
    ```

2. **`docker compose` tidak ada**

*   Pastikan package `docker-compose-plugin` ter-install dan cek:

    ```bash
    docker compose version
    ```

    Rujukan instal Compose plugin: ([Docker Documentation](https://docs.docker.com/compose/install/linux/?utm_source=chatgpt.com))

3. **VM sering OOM / lambat**

* Untuk 1 VM, ClickHouse + collector bisa berat. Naikkan RAM (mulai 8–16GB), pastikan SSD, dan cek disk free.
