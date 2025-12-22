# Active Service

Sip—berarti root cause 401 tadi sudah beres (project `threebody` belum dibuat). Sekarang soal “kalau VM2 mati/di‑reboot, semua tetap aktif”:

> Secara fisik kalau VM2 **mati/off**, ya tidak mungkin service “tetap jalan”.\
> Yang bisa kita buat adalah: **begitu VM2 nyala lagi**, semua service **auto start** dan kembali running **tanpa kamu jalankan manual** (Docker, Harbor, GitLab Runner, dan (opsional) dev stack).

Di VM2 kita bikin **auto-start yang rapi** pakai `systemd`.

***

### 1) Pastikan service dasar auto-start

#### 1.1 Docker & containerd

Di VM2:

```bash
sudo systemctl enable --now docker
sudo systemctl enable --now containerd
sudo systemctl status docker --no-pager
```

> Biasanya Docker sudah enable by default, tapi ini memastikan.

#### 1.2 GitLab Runner (kalau kamu pakai runner di VM2)

```bash
sudo systemctl enable --now gitlab-runner
sudo systemctl status gitlab-runner --no-pager
```

***

### 2) Pastikan Harbor auto-start setelah reboot

Ada 2 pendekatan:

#### Opsi A (paling sederhana): pakai restart policy container

Harbor Compose defaultnya biasanya sudah `restart: always`, jadi saat Docker hidup, container Harbor ikut hidup.

Cek restart policy Harbor container:

```bash
sudo docker ps -a --format 'table {{.Names}}\t{{.Status}}'
sudo docker inspect -f '{{.Name}} -> {{.HostConfig.RestartPolicy.Name}}' \
  harbor-core harbor-db harbor-jobservice harbor-portal nginx redis registry registryctl harbor-log 2>/dev/null
```

Kalau restart policy kosong, kamu bisa paksa:

```bash
sudo docker update --restart unless-stopped harbor-core harbor-db harbor-jobservice harbor-portal nginx redis registry registryctl harbor-log
```

**TAPI**: Opsi A ini tidak mengontrol urutan start dan tidak “nunggu Harbor ready”.

***

#### Opsi B (rekomendasi, paling stabil): bikin `systemd service` untuk Harbor

Ini membuat Harbor pasti “di-up” saat boot dan jadi dependency untuk service lain (misalnya dev stack).

Buat unit file:

```bash
sudo tee /etc/systemd/system/harbor.service > /dev/null <<'EOF'
[Unit]
Description=Harbor Registry (Docker Compose)
Requires=docker.service network-online.target
After=docker.service network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/harbor

# tunggu docker siap
ExecStartPre=/bin/sh -c 'until /usr/bin/docker info >/dev/null 2>&1; do sleep 1; done'

ExecStart=/usr/bin/docker compose -f /opt/harbor/docker-compose.yml up -d
ExecStop=/usr/bin/docker compose -f /opt/harbor/docker-compose.yml stop
ExecReload=/usr/bin/docker compose -f /opt/harbor/docker-compose.yml restart
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
EOF
```

Aktifkan:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now harbor
sudo systemctl status harbor --no-pager
```

Tes cepat:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://192.168.56.43:8081/api/v2.0/ping
# target: 200
```

***

### 3) (Opsional tapi biasanya diminta) Dev stack auto-start juga

Kamu bisa pilih:

* **Dev stack auto-start** (mysql/laravel/goapi/frontend/nginx-dev langsung running setelah boot)
* atau **hanya Harbor + Runner** yang auto-start (dev stack kamu nyalakan manual kalau butuh)

Karena kamu bilang “semuanya aktif”, aku buatkan yang auto-start juga.

#### 3.1 Pastikan file compose dev sudah fix dan PATH benar

Asumsi struktur kamu:

* `/home/cikal/three-body-problem-main/docker-compose.dev.yml`
* `/home/cikal/three-body-problem-main/.env.dev.compose`

Cek:

```bash
ls -la /home/cikal/three-body-problem-main/docker-compose.dev.yml
ls -la /home/cikal/three-body-problem-main/.env.dev.compose
```

> Kalau file compose dev kamu namanya `docker-compose.dev.yml` (pakai tanda minus), sesuaikan di bawah. Yang penting konsisten.

***

#### 3.2 Tambahkan restart policy di `docker-compose.dev.yml` (biar survive docker restart)

Di setiap service tambahkan:

```yaml
restart: unless-stopped
```

Contoh:

```yaml
services:
  mysql:
    restart: unless-stopped
    ...

  laravel:
    restart: unless-stopped
    ...
```

Setelah edit, recreate sekali:

```bash
cd /home/cikal/three-body-problem-main
docker compose --env-file .env.dev.compose -f docker-compose.dev.yml up -d --force-recreate
```

***

#### 3.3 Bikin systemd service untuk dev stack (start setelah Harbor ready)

Ini yang bikin “boot -> harbor ready -> dev stack up”.

```bash
sudo tee /etc/systemd/system/threebody-dev.service > /dev/null <<'EOF'
[Unit]
Description=Three Body Problem Dev Stack (Docker Compose)
Requires=docker.service harbor.service network-online.target
After=docker.service harbor.service network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/cikal/three-body-problem-main

# tunggu docker siap
ExecStartPre=/bin/sh -c 'until /usr/bin/docker info >/dev/null 2>&1; do sleep 1; done'

# tunggu harbor ready (ping 200)
ExecStartPre=/bin/sh -c 'until curl -fsS http://192.168.56.43:8081/api/v2.0/ping >/dev/null; do sleep 2; done'

ExecStart=/usr/bin/docker compose --env-file /home/cikal/three-body-problem-main/.env.dev.compose -f /home/cikal/three-body-problem-main/docker-compose.dev.yml up -d
ExecStop=/usr/bin/docker compose --env-file /home/cikal/three-body-problem-main/.env.dev.compose -f /home/cikal/three-body-problem-main/docker-compose.dev.yml stop
ExecReload=/usr/bin/docker compose --env-file /home/cikal/three-body-problem-main/.env.dev.compose -f /home/cikal/three-body-problem-main/docker-compose.dev.yml restart
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
EOF
```

Aktifkan:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now threebody-dev
sudo systemctl status threebody-dev --no-pager
```

Cek container jalan:

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

***

### 4) Checklist setelah reboot (ini cara verifikasi)

Tes “beneran auto-start”:

```bash
sudo reboot
```

Setelah VM2 up lagi, cek:

```bash
systemctl status docker --no-pager
systemctl status harbor --no-pager
systemctl status gitlab-runner --no-pager
systemctl status threebody-dev --no-pager

docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
curl -s -o /dev/null -w "%{http_code}\n" http://192.168.56.43:8081/api/v2.0/ping
```

Target:

* docker: active
* harbor: active
* gitlab-runner: active
* threebody-dev: active
* `docker ps` ada container harbor + container dev stack
* ping harbor = 200

***

### 5) Catatan penting biar tidak “hilang data” / tidak bikin masalah

1. **Jangan** pakai ini kecuali kamu memang mau reset DB dev:

```bash
docker compose down --volumes
```

Karena itu menghapus volume MySQL.

2. Kalau kamu mau stop dev stack tapi tidak hapus data:

```bash
sudo systemctl stop threebody-dev
# atau:
docker compose --env-file .env.dev.compose -f docker-compose.dev.yml stop
```

3. Kalau kamu mau stop harbor:

```bash
sudo systemctl stop harbor
```

***

Kalau kamu setuju, langkah berikutnya setelah auto-start beres adalah:

* kita rapikan **struktur repo “infra”** (folder `devops/`, compose, nginx, dockerfiles) biar stabil,
* lalu baru lanjut bikin **`.gitlab-ci.yml final`** (build → test → push Harbor → deploy dev compose → deploy k8s staging/prod → healthcheck + post-deploy).
