# APM Signoz

#### Apa itu “APM SigNoz”?

**SigNoz** adalah **platform observability open‑source** (sering dipakai sebagai **APM**) yang “OpenTelemetry‑native”. Artinya, SigNoz mengumpulkan **traces (distributed tracing/APM)**, **metrics**, dan **logs** (plus alerting) dalam satu UI, dengan data masuk lewat **OpenTelemetry Collector** dan disimpan terutama di **ClickHouse** untuk query yang cepat.

Kalau kamu fokus APM: SigNoz APM biasanya dipakai untuk:

* melihat **service map**, **latency p99**, **error rate**, dan analisis trace end‑to‑end (request mengalir ke service mana saja, bottleneck di mana).

Gambaran arsitektur sederhananya:

1. Aplikasi di‑instrument dengan **OpenTelemetry SDK/agent**
2. Kirim telemetry ke **SigNoz OTel Collector**
3. Collector menulis data ke **ClickHouse**
4. **SigNoz (backend+frontend)** men-query & menampilkan di UI + alerting

***

## RUNBOOK FINAL — SigNoz (Docker Compose) Ubuntu 22.04 (VM 192.168.56.23)

### A. Install & Deploy (sekali saja)

#### 1) Update OS + tools

```bash
sudo apt update
sudo apt -y upgrade
sudo apt -y install git curl ca-certificates gnupg
```

#### 2) (Recommended) Tambah swap 4GB (VM RAM 6GB)

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
free -h
```

#### 3) Install Docker Engine + Compose plugin

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt-get remove -y $pkg
done

sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo systemctl enable --now docker
sudo docker run --rm hello-world
docker compose version
```

#### 4) (Optional, recommended) Docker tanpa sudo untuk user `cikal`

```bash
sudo usermod -aG docker $USER
newgrp docker
```

#### 5) Clone SigNoz + start stack

```bash
cd ~
git clone -b main https://github.com/SigNoz/signoz.git
cd ~/signoz/deploy/docker
docker compose up -d --remove-orphans
```

#### 6) Cek status container

```bash
cd ~/signoz/deploy/docker
docker compose ps
```

#### 7) Akses UI

Buka di browser:

* `http://192.168.56.23:8080`

***

### B. Deploy “Generator Test” (otlp-python) (buat test trace kapan pun)

> Ini bukan wajib, tapi sangat berguna untuk test cepat.

#### 1) Jalankan otlp-python di network SigNoz + auto-start

```bash
docker rm -f otlp-python 2>/dev/null || true

docker run -d --name otlp-python --restart unless-stopped \
  --network signoz-net \
  -p 5002:5002 \
  -e OTLP_ENDPOINT="signoz-otel-collector:4317" \
  -e INSECURE="true" \
  signoz/otlp-python
```

#### 2) Tes generate trace

```bash
curl "http://127.0.0.1:5002/?user=alice"
curl "http://127.0.0.1:5002/?user=bob"
```

#### 3) Validasi di UI

UI → Explorer → Traces:

* time range **Last 15 minutes**
* filter `service.name="demo-app"` (atau pilih service `demo-app`)
* Run Query → harus muncul trace baru.

***

## C. SOP Setelah VM Restart (INI YANG KAMU BUTUH)

### 1) Pastikan Docker hidup + SigNoz container up

```bash
sudo systemctl status docker --no-pager
cd ~/signoz/deploy/docker
docker compose ps
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

### 2) Smoke test: pastikan OTLP receiver benar-benar listen

> Ini penting, karena “Hello bob” tidak menjamin trace masuk.

```bash
python3 - <<'PY'
import socket
for port in (4317,4318):
    try:
        s=socket.create_connection(("127.0.0.1", port), timeout=2)
        print("OTLP OK", port)
        s.close()
    except Exception as e:
        print("OTLP FAIL", port, e)
PY
```

**Kalau hasilnya `OTLP OK 4317` dan/atau `OTLP OK 4318` → lanjut langkah 3.**\
**Kalau `OTLP FAIL` → lompat ke bagian D (Recovery).**

### 3) Test trace (cukup curl)

```bash
curl "http://127.0.0.1:5002/?user=alice"
curl "http://127.0.0.1:5002/?user=bob"
```

### 4) Cek cepat error exporter (harusnya tidak ada “UNAVAILABLE”)

```bash
docker logs --tail=50 otlp-python
```

***

## D. Recovery (kalau setelah reboot “curl hello” tapi tidak ada trace baru)

Gejala:

* UI kosong untuk `demo-app`
* log `otlp-python` ada `StatusCode.UNAVAILABLE`
* atau smoke test `OTLP FAIL`

### 1) Restore config collector ke default repo + recreate collector

```bash
cd ~/signoz

# backup config yang sedang ada (biar aman)
cp deploy/docker/otel-collector-config.yaml ~/otel-collector-config.yaml.bak.$(date +%F-%H%M%S)

# balikin ke config default repo
git checkout -- deploy/docker/otel-collector-config.yaml

# recreate collector
cd ~/signoz/deploy/docker
docker compose up -d --force-recreate --no-deps otel-collector
```

### 2) Verify port 4317/4318 dari dalam otlp-python

```bash
docker exec otlp-python sh -lc 'python3 - <<PY
import socket
host="signoz-otel-collector"
for port in (4317,4318):
    try:
        s=socket.create_connection((host,port),timeout=3)
        print("OK",port)
        s.close()
    except Exception as e:
        print("FAIL",port,e)
PY'
```

### 3) Generate trace lagi

```bash
curl "http://127.0.0.1:5002/?user=alice"
curl "http://127.0.0.1:5002/?user=bob"
```

***

## E. Operasional (Start/Stop)

### Stop semua

```bash
cd ~/signoz/deploy/docker
docker compose down
```

### Start lagi

```bash
cd ~/signoz/deploy/docker
docker compose up -d --remove-orphans
```

***

## F. Catatan kecil

* Warning `version is obsolete` di compose **aman diabaikan**.
* Kalau VM kamu berat/sering OOM, swap 4GB sangat membantu.

***

#### Tambahan jika restart vm dan tidak ada log nya (connection refused)

Mantap, hasil debug kamu sudah **sangat jelas**:

* `otlp-python` dan `signoz-otel-collector` **sudah satu network (`signoz-net`)** ✅
* DNS resolve **beres** (ke `172.18.0.4`) ✅
* Tapi koneksi ke `4317/4318` dari `otlp-python` = **Connection refused** ❌
* Dan di collector `/proc/net/tcp*` **tidak ada LISTEN di 4317/4318** ❌ (`NOT LISTENING`)

Jadi root cause-nya: **proses di dalam container `signoz-otel-collector` tidak membuka OTLP receiver** (port 4317/4318). Port “published” di compose itu hanya mapping, tapi kalau di dalam container tidak listen, tetap akan “refused”.

Di bawah ini fix yang paling cepat dan “bersih”.

***

### 1) Restore config collector ke default dari repo (paling sering ini penyebabnya)

Kadang file `otel-collector-config.yaml` berubah/korup sehingga receiver OTLP hilang.

Jalankan:

```bash
cd ~/signoz

# backup dulu biar aman
cp deploy/docker/otel-collector-config.yaml ~/otel-collector-config.yaml.bak.$(date +%F-%H%M%S)

# balikin ke versi default repo
git checkout -- deploy/docker/otel-collector-config.yaml
```

***

### 2) Recreate container collector (biar config baru kebaca)

```bash
cd ~/signoz/deploy/docker
docker compose up -d --force-recreate --no-deps otel-collector
```

***

### 3) Tes lagi dari dalam `otlp-python` (harusnya sekarang OK)

```bash
docker exec otlp-python sh -lc 'python3 - <<PY
import socket
host="signoz-otel-collector"
for port in (4317,4318):
    try:
        s=socket.create_connection((host,port),timeout=3)
        print("OK",port)
        s.close()
    except Exception as e:
        print("FAIL",port,e)
PY'
```

Target: minimal `OK 4317` dan idealnya `OK 4318` juga.

***

### 4) Kalau sudah OK, baru test generate trace

```bash
curl "http://127.0.0.1:5002/?user=alice"
curl "http://127.0.0.1:5002/?user=bob"
```

Di UI:

* time range **Last 15 minutes**
* **Clear** query/filter yang aneh dulu (hapus `service.name not in ['compose']` sementara)
* Run Query → nanti akan muncul service demo lagi

***

### Kalau setelah recreate masih `NOT LISTENING`

Berarti collector sebenarnya jalan, tapi config yang dia pakai tetap tidak mengaktifkan OTLP. Kita cek isi config yang sedang dipakai **di dalam container**:

```bash
docker exec signoz-otel-collector sh -lc '
echo "=== otel config (search otlp receiver) ==="
grep -n "receivers:" -n /etc/otel-collector-config.yaml || true
grep -n "otlp" /etc/otel-collector-config.yaml | head -n 80 || true
'
```

Dan cek log collector (pakai `docker logs` langsung):

```bash
docker logs --tail=200 signoz-otel-collector
```

***

#### Atau ini juga bisa

Mantap, log yang kamu kirim **sudah cukup jelas** untuk nemuin akar masalahnya.

### 1) Diagnosis (kenapa UI kosong & “connection refused”)

#### A. `otlp-python` **bisa jalan**, tapi **gagal kirim OTLP ke collector**

Buktinya dari `docker logs otlp-python`:

* `Transient error StatusCode.UNAVAILABLE ... exporting logs/traces to signoz-otel-collector:4317`\
  Ini artinya **exporter OTLP di app tidak bisa konek ke endpoint collector** (biasanya TCP connect gagal / endpoint salah).

#### B. Endpoint yang dipakai demo kamu adalah **`signoz-otel-collector:4317`**

Dari log jelas targetnya: `signoz-otel-collector:4317`

Masalahnya: pada Docker Compose, nama service yang paling stabil untuk dipakai antar container adalah **nama service**, yaitu:

✅ **`otel-collector:4317`**\
bukan `signoz-otel-collector:4317` (itu container\_name, kadang bisa, tapi sering bikin masalah setelah restart / DNS internal)

#### C. Setelah reboot, Docker restart container **tanpa urutan depends\_on**

Ini menjelaskan kenapa di log collector sempat muncul:

* `dial tcp ...:9000 connect: connection refused` (ClickHouse belum siap saat collector start)
* OpAMP connect refused (tidak fatal buat OTLP, tapi tanda urutan start kacau)

Jadi **setelah reboot**, stack bisa “kelihatan up”, tapi endpoint internal bisa tidak stabil sampai kamu “compose up” ulang.

***

### 2) FIX CEPAT (yang pasti bikin trace masuk)

Kita **recreate** container demo `otlp-python` dengan endpoint yang benar.

> Jalankan dari host VM kamu (Ubuntu).

#### Step 1 — hapus demo lama

```bash
docker rm -f otlp-python 2>/dev/null || true
```

#### Step 2 — jalankan ulang demo dengan endpoint service name (REKOMENDASI)

```bash
docker run -d --name otlp-python --restart unless-stopped \
  --network signoz-net \
  -p 5002:5002 \
  -e OTLP_ENDPOINT="otel-collector:4317" \
  -e INSECURE="true" \
  signoz/otlp-python:latest
```

#### Step 3 — tes koneksi TCP dari dalam container (harus OK)

```bash
docker exec otlp-python python3 - <<'PY'
import socket
for host in ("otel-collector","signoz-otel-collector"):
  for port in (4317,4318):
    try:
      s=socket.create_connection((host,port),timeout=3); s.close()
      print("OK ",host,port)
    except Exception as e:
      print("FAIL",host,port,repr(e))
PY
```

**Target minimal yang wajib OK:** `otel-collector 4317`

#### Step 4 — generate traffic

```bash
curl "http://127.0.0.1:5002/?user=alice"
curl "http://127.0.0.1:5002/?user=bob"
```

#### Step 5 — cek log demo (warning harus hilang)

```bash
docker logs --tail=80 otlp-python
```

Kalau masih muncul `Transient error ... signoz-otel-collector:4317`, berarti container kamu belum terganti (biasanya karena command lama masih jalan). Pastikan `docker rm -f otlp-python` sukses dulu.

***

### 3) Cara pastiin muncul di UI (biar nggak “no results”)

1. Buka: `http://192.168.56.23:8080`
2. Ubah time range jadi **Last 15 minutes**
3. Masuk menu **Services**
4. Cari service name yang benar:

**Demo ini biasanya muncul sebagai `demo-service`** (di log kamu juga ada `"name": "demo-service"`).\
Jadi kalau kamu sebelumnya nge-filter **`demo-app`**, ya bakal “no results”.

✅ Solusi: **Clear filter** → cari `demo-service`.

***

### 4) Fix permanen setelah reboot (biar tidak kejadian lagi)

Karena Docker restart container **tanpa urutan**, setelah reboot lakukan salah satu:

#### Opsi A (paling simpel): setelah reboot jalankan ulang compose

```bash
cd ~/signoz/deploy/docker
docker compose up -d
```

#### Opsi B (lebih rapih): restart stack sekali saja setelah boot

Kalau setelah reboot kamu lihat trace “mati”, jalankan:

```bash
cd ~/signoz/deploy/docker
docker compose restart clickhouse
docker compose restart otel-collector
```

> Ini memastikan ClickHouse siap dulu, lalu collector.

***

### 5) Kalau masih gagal (plan B yang “anti DNS internal”)

Kalau (jarang) DNS docker internal bermasalah, pakai metode host-gateway (paling kebal):

```bash
docker rm -f otlp-python 2>/dev/null || true

docker run -d --name otlp-python --restart unless-stopped \
  -p 5002:5002 \
  --add-host signoz:host-gateway \
  -e OTLP_ENDPOINT="signoz:4317" \
  -e INSECURE="true" \
  signoz/otlp-python:latest
```

***

Kalau kamu jalankan ulang demo pakai `otel-collector:4317`, lalu kirim output dari:

* `docker exec otlp-python python3 ...` (tes socket)
* `docker logs --tail=50 otlp-python`
