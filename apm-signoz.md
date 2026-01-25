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
