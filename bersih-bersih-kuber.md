# Bersih-bersih kuber

Boleh, itu **arahnya sudah benar** buat “retry bersih”. Tapi biar benar-benar aman (nggak ada file yang masih “dipegang” pod mysql saat kamu hapus folder), aku sarankan **urutannya sedikit dirapikan**.

### 0) Cek dulu (biar yakin yang akan dihapus memang cuma environment app)

Di **vm-k8s**:

```bash
NS=threebody-prod

kubectl get ns | egrep -n "NAME|threebody|monitoring" || true
kubectl -n $NS get all || true
kubectl -n $NS get pvc || true
kubectl get pv | egrep -n "NAME|mysql-pv-hostpath" || true
```

Kalau yang muncul memang hanya resource “threebody”, lanjut bersih.

***

### 1) Bersihkan di vm-k8s (disarankan, lebih aman dari “masih running”)

#### A. Matikan dulu workload (biar volume dilepas clean)

```bash
NS=threebody-prod

kubectl -n $NS scale deployment/frontend deployment/go deployment/laravel --replicas=0 || true
kubectl -n $NS scale statefulset/mysql --replicas=0 || true

# pastikan podnya hilang dulu (terutama mysql)
kubectl -n $NS delete pod -l app=mysql --ignore-not-found=true || true
kubectl -n $NS delete pod -l app=frontend --ignore-not-found=true || true
kubectl -n $NS delete pod -l app=go --ignore-not-found=true || true
kubectl -n $NS delete pod -l app=laravel --ignore-not-found=true || true
```

#### B. Hapus namespace + PV

```bash
kubectl delete ns $NS --wait=true || true

# PV cluster-scoped (ini yang bikin “nggak bersih” kalau dibiarkan)
kubectl delete pv mysql-pv-hostpath --ignore-not-found=true || true
```

#### Cek hasil

```bash
kubectl get ns | grep -w threebody-prod || echo "OK: namespace sudah hilang"
kubectl get pv | grep -w mysql-pv-hostpath || echo "OK: PV sudah hilang"
```

> Kalau namespace/PV **nyangkut Terminating**, baru lakukan “force remove finalizer” (kasih tahu aku outputnya, nanti aku arahkan patch finalizer yang tepat).

***

### 2) Bersihkan di vm-worker (data mysql hostPath)

Ini aman, **tapi lakukan setelah step vm-k8s di atas** (supaya pod mysql benar-benar mati dan nggak nulis lagi).

```bash
sudo rm -rf /data/threebody/mysql/*
sudo mkdir -p /data/threebody/mysql
sudo chown -R 999:999 /data/threebody/mysql
sudo chmod 700 /data/threebody/mysql
sudo ls -la /data/threebody/mysql
```

***

### Catatan soal “imagePullSecret variable”

Kamu sudah benar: **jangan hapus `spec:`** di manifest. Yang kamu lakukan sekarang (pakai variable `K8S_IMAGEPULL_SECRET` + patch ke default serviceaccount) sudah cocok, jadi manifest deployment tidak perlu `imagePullSecrets:` lagi.

***

Kalau kamu mau aku pastikan ini **100% bersih**, kirim output ini setelah kamu jalankan “cek dulu” (bagian 0):

```bash
kubectl -n threebody-prod get all 2>/dev/null || echo "ns not found"
kubectl get pv | grep mysql || echo "no mysql pv"
```

Ambil password Grafana (kube-prometheus-stack / kps)

1. Cara paling umum (sesuai NOTES helm kamu)

Di vm-k8s:

kubectl -n monitoring get secret kps-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo

Login:

user: admin

password: hasil command di atas







Sip. Ini aku bikinin versi **“catatan belajar + glosarium + troubleshooting flow”** supaya kamu bisa belajar runbook ini dengan cepat, tapi tetap **paham detail fungsinya**.

Aku buat dalam format yang enak dipakai sehari-hari: **cheatsheet + decision tree** + kumpulan command “cek cepat”.

***

## Catatan Belajar Runbook Three-Body-Problem

### 0) “Peta mental” dulu (biar nggak nyasar)

Bayangin sistem kamu itu punya **5 lapisan**:

1. **DNS & hostname** (A2–A4)\
   → memastikan `harbor.local` dan `hit.local` selalu resolve benar
2. **Container runtime**
   * vm-docker pakai **Docker**
   * vm-k8s & vm-worker pakai **containerd**
3. **Registry (Harbor)**\
   → tempat pipeline push image, tempat K8s pull image
4. **Kubernetes**\
   → menjalankan MySQL + 3 service app (frontend/go/laravel)
5. **Edge Nginx (hit.local)**\
   → reverse proxy HTTPS dari user ke NodePort K8s

Kalau ada error, hampir selalu bisa dilokalisasi ke salah satu layer itu.

***

## 1) Glosarium yang wajib kamu kuasai

### A. Konsep jaringan & akses

#### Host-only network `192.168.56.0/24`

* Jaringan lokal antar VM (biasanya VirtualBox host-only).
* Stabil, tidak berubah-ubah seperti NAT.

#### `/etc/hosts`

* “DNS manual” di masing-masing VM.
* Kamu pakai supaya:
  * `harbor.local` selalu = `192.168.56.42`
  * `hit.local` selalu = `192.168.56.42`

**Kenapa penting:** CI runner dan K8s node sering butuh resolve domain ini. Kalau DNS random, pipeline bisa “kadang jalan kadang gagal”.

#### NodePort

* Service Kubernetes yang “membuka port di node”.
* Di runbook:
  * frontend: `30080`
  * go: `30081`
  * laravel: `30082`

**Artinya:** request ke `http://192.168.56.44:30081` akan diteruskan ke Pod service Go.

***

### B. Docker / Harbor

#### Harbor Registry

* Registry Docker private (di vm-docker).
* UI: `http://harbor.local:8080`
* Registry endpoint: `http://harbor.local:8080/v2/`

#### Insecure registry (HTTP)

* Normalnya registry harus HTTPS.
* Kamu lab → pakai HTTP, jadi:
  * Docker daemon runner harus allow insecure registry
  * containerd node K8s harus allow HTTP registry lewat `hosts.toml`

**Tanda sukses**:

* `curl -I http://harbor.local:8080/v2/` balas **401 Unauthorized** (itu bagus—berarti hidup).

***

### C. Kubernetes object

#### Namespace

* “Folder” untuk resource K8s.
* Kamu pakai `threebody-prod` (app) dan `monitoring` (opsional monitoring).

#### Deployment

* Menjalankan Pod stateless (frontend/go/laravel).
* Bisa scale replicas cepat.

#### StatefulSet

* Untuk workload stateful (MySQL).
* Memberi nama pod stabil: `mysql-0`.

#### Service (ClusterIP/Headless/NodePort)

* “DNS + loadbalancing” internal K8s
* Headless (`clusterIP: None`) memberi DNS pod spesifik: `mysql-0.mysql`

#### PV/PVC

* PV: resource storage “nyata”
* PVC: request storage oleh Pod

#### hostPath PV

* Storage menempel ke filesystem node.
* Kamu pakai `/data/threebody/mysql` di vm-worker

#### nodeAffinity PV

* Mengunci PV hanya bisa dipakai pada node tertentu (di sini: `vm-worker`).

***

### D. CI/CD & secrets

#### GitLab Runner (executor shell)

* Runner menjalankan script di VM langsung (bukan docker-in-docker).
* Karena shell, dia pakai Docker engine yang terinstal di vm-docker.

#### GitLab Variables (secret manager)

* Semua credential disimpan di GitLab, tidak hardcode.
* `KUBECONFIG_PROD` Type: File → kubectl bisa akses cluster.

#### imagePullSecret + ServiceAccount default

* Secret docker-registry dipasang ke ServiceAccount default namespace.
* Supaya semua Pod bisa pull image dari Harbor tanpa setting manual per manifest.

***

## 2) Cheatsheet: “Kalimat sederhana” fungsi tiap file YAML

### `deploy/k8s/base/10-mysql.yaml`

Membuat:

* Service headless `mysql` (DNS: `mysql`, pod DNS: `mysql-0.mysql`)
* PV hostPath `/data/threebody/mysql` (di node `vm-worker`)
* PVC yang bind ke PV
* StatefulSet MySQL (pod `mysql-0`)

**Kenapa ada initContainer fix-perms:**\
folder hostPath biasanya `root:root`, MySQL jalan sebagai uid `999`, kalau tidak diubah → MySQL crash.

***

### `deploy/k8s/base/20-go.yaml`

Membuat:

* Service NodePort `30081`
* Deployment go (default `replicas: 0`)

**Kenapa replicas 0:** agar go tidak start sebelum MySQL siap + migrasi done.

***

### `deploy/k8s/base/30-laravel.yaml`

Membuat:

* ConfigMap nginx config untuk laravel
* Service NodePort `30082`
* Deployment laravel (default `replicas: 0`)
* Pod berisi:
  * container `laravel` (php-fpm)
  * container `laravel-nginx` (nginx)
  * initContainer `copy-app` (copy source ke shared volume)

**Kenapa initContainer copy-app:** nginx container tidak punya source laravel, jadi source “disalin” ke volume bersama.

***

### `deploy/k8s/base/40-frontend.yaml`

Membuat:

* Service NodePort `30080`
* Deployment frontend (replicas 1)

***

### `deploy/k8s/jobs/laravel-migrate-job.yaml`

Menjalankan:

* wait mysql ready (SELECT 1)
* migrate
* seed _jika table products kosong_

**Kenapa dibuat Job terpisah:** supaya migrasi bisa “dipaksa sukses dulu” sebelum laravel service dinaikkan.

***

## 3) Cheatsheet: Edge Nginx (yang paling sering bikin bingung)

### redirect http → https

```nginx
server { listen 80; return 301 https://$host$request_uri; }
```

Tujuan: semua akses dipaksa HTTPS.

### TLS cert self-signed

Nginx pakai:

* `/etc/nginx/certs/tls.crt`
* `/etc/nginx/certs/tls.key`

### proxy\_pass dan aturan trailing slash (WAJIB PAHAM)

Ini aturan penting:

#### A) location `/go/` + proxy\_pass `http://.../`

```nginx
location /go/ { proxy_pass http://192.168.56.44:30081/; }
```

→ prefix `/go/` akan “dipotong”.

Contoh:

* `/go/api/products` → diteruskan ke `http://node:30081/api/products`

#### B) location `/` + proxy\_pass tanpa slash akhir

```nginx
location / { proxy_pass http://192.168.56.44:30080; }
```

→ URI diteruskan apa adanya.

***

### Rate limiting

```nginx
limit_req_zone $binary_remote_addr zone=api_rl:10m rate=5r/s;
...
limit_req zone=api_rl burst=10 nodelay;
```

* `rate=5r/s` = per IP max 5 req/detik
* `burst=10` = boleh spike 10 request
* `nodelay` = tidak antre; selama masih burst, langsung tembus

***

## 4) Cheatsheet: Pipeline `.gitlab-ci.yml` (apa yang terjadi saat push)

### Stage build: `build_check`

* build semua image (frontend/go/laravel) tapi tidak push
* tujuan: fail-fast jika Dockerfile error

### Stage push: `push_images`

* docker login ke Harbor (HTTP insecure)
* build + push tag commit (SHA)
* tag `latest` juga

### Stage deploy: `deploy`

Urutan “super stabil”:

1. validasi variables
2. download kubectl
3. pakai kubeconfig dari file variable
4. pastikan namespace ada
5. create secret app-secrets (DB, APP\_KEY)
6. create docker-registry secret (imagePullSecret)
7. patch default SA supaya imagePullSecret nempel
8. apply manifests
9. scale down semua app (mysql dulu)
10. tunggu mysql rollout + Ready + mysqladmin ping
11. ensure DB + grant user
12. set image ke tag commit
13. jalankan migrate job (fail-fast)
14. scale up laravel → go → frontend
15. healthcheck NodePort / edge

### Stage monitoring: `monitoring_install` (manual)

* install kps + loki + promtail via helm
* expose grafana/prom/loki via NodePort

***

## 5) Checklist “Cek cepat” sebelum push pertama

Jalankan ini biar push pertama benar-benar minim error.

### Di semua VM

```bash
getent hosts harbor.local hit.local vm-k8s vm-worker
```

### Di vm-docker (Harbor + Docker + Edge)

```bash
docker version
curl -fsSI http://harbor.local:8080/v2/ | head -n 1
sudo docker ps
curl -kfsS --resolve hit.local:443:127.0.0.1 https://hit.local/ | head
```

### Di vm-k8s (control-plane)

```bash
kubectl get nodes -o wide
kubectl get pods -A | head
```

### Di vm-worker (nodeport reachability dari vm-docker)

Dari vm-docker:

```bash
curl -fsSI http://192.168.56.44:30080/ | head -n 1
```

Kalau ini gagal, biasanya UFW di vm-worker block NodePort.

***

## 6) Troubleshooting Playbook (Decision Tree)

Ini bagian paling berguna untuk “kalau error muncul, cek apa dulu”.

***

### Kasus 1: `harbor.local` / `hit.local` tidak resolve

**Gejala:**

* `docker login harbor.local:8080` gagal resolve
* `curl http://harbor.local:8080` timeout

**Cek & perbaiki:**

```bash
cat /etc/hosts
getent hosts harbor.local hit.local
```

Kalau salah → ulangi A3.

Lanjut cek resolv.conf:

```bash
ls -l /etc/resolv.conf
cat /etc/resolv.conf
resolvectl status | head -n 80
```

Kalau `resolv.conf` bukan symlink ke `/run/systemd/resolve/resolv.conf` → ulangi A4.

***

### Kasus 2: `curl http://harbor.local:8080/v2/` tidak balas 401

**Gejala:**

* Timeout / connection refused

**Cek di vm-docker:**

```bash
sudo docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}' | egrep -i 'harbor|nginx'
sudo ss -lntp | egrep ':8080'
sudo docker compose -f /opt/harbor/docker-compose.yml ps
```

Kalau port 8080 tidak listen:

* harbor container belum naik
* atau UFW block

Cek UFW:

```bash
sudo ufw status verbose
sudo ufw allow 8080/tcp
sudo ufw reload
```

***

### Kasus 3: Pipeline error `permission denied /var/run/docker.sock`

**Gejala:**

* `docker build` gagal di runner
* `Got permission denied while trying to connect to the Docker daemon socket`

**Fix di vm-docker:**

```bash
sudo usermod -aG docker gitlab-runner
sudo chown root:docker /var/run/docker.sock
sudo chmod 660 /var/run/docker.sock
sudo systemctl restart docker gitlab-runner
sudo -u gitlab-runner docker ps
```

Kalau `sudo -u gitlab-runner docker ps` berhasil → aman.

***

### Kasus 4: K8s Pod `ImagePullBackOff` dari Harbor

**Penyebab umum (urut dari paling sering):**

1. imagePullSecret belum nempel ke ServiceAccount default
2. secret docker-registry salah (username/password/server)
3. containerd belum allow Harbor HTTP (hosts.toml)
4. DNS cluster/VM tidak resolve `harbor.local`

**Cek dari vm-k8s:**

```bash
NS=threebody-prod
kubectl -n $NS get pod -o wide
kubectl -n $NS describe pod <POD_NAME> | egrep -i 'image|pull|back-off|failed|unauthorized' -n
```

Cek SA default:

```bash
kubectl -n $NS get sa default -o yaml | egrep -n 'imagePullSecrets|name:'
```

Cek secret ada:

```bash
kubectl -n $NS get secret | egrep -i 'docker|pull|harbor'
```

Kalau SA belum ada imagePullSecrets → berarti step patch SA gagal.\
Kalau secret ada tapi unauthorized → credential salah.

**Cek containerd allow HTTP di node (vm-worker/vm-k8s):**

```bash
sudo cat /etc/containerd/certs.d/harbor.local:8080/hosts.toml
sudo systemctl restart containerd
```

***

### Kasus 5: MySQL Pod crashloop / not ready

**Cek cepat:**

```bash
kubectl -n threebody-prod get pod -l app=mysql -o wide
kubectl -n threebody-prod describe pod mysql-0 | tail -n 80
kubectl -n threebody-prod logs mysql-0 -c mysql --tail=200
```

**Penyebab umum: permission datadir**

* hostPath dibuat root, MySQL uid 999 tidak bisa tulis

Fix di vm-worker:

```bash
sudo mkdir -p /data/threebody/mysql
sudo chown -R 999:999 /data/threebody/mysql
sudo chmod 700 /data/threebody/mysql
```

Kalau PV/PVC pending:

```bash
kubectl -n threebody-prod get pvc
kubectl get pv
kubectl describe pv mysql-pv-hostpath | tail -n 80
```

Penyebab umum:

* nodeAffinity tidak cocok (hostname node tidak `vm-worker`)
* hostname node berbeda dari runbook

Cek hostname worker:

```bash
kubectl get nodes -o wide
# lihat kolom NAME harus vm-worker
```

***

### Kasus 6: migrate Job gagal

**Cek job:**

```bash
kubectl -n threebody-prod get job
kubectl -n threebody-prod describe job laravel-migrate | tail -n 120
kubectl -n threebody-prod logs -l job-name=laravel-migrate --all-containers=true --tail=200
```

**Penyebab umum:**

* DB credential salah
* DB belum dibuat/grant
* APP\_KEY invalid (kadang Laravel error)
* MySQL belum benar-benar ready

Kalau error koneksi DB, test manual:

```bash
kubectl -n threebody-prod exec -it mysql-0 -c mysql -- \
  mysqladmin ping -h 127.0.0.1 -uroot -p"$MYSQL_ROOT_PASSWORD" --silent
```

***

### Kasus 7: `https://hit.local` 502 Bad Gateway

**Ini hampir selalu salah satu dari:**

1. Pod backend belum ready / service belum punya endpoint
2. NodePort diblok firewall UFW
3. edge nginx config salah (proxy\_pass)
4. port NodePort salah

**Langkah cek:**

#### 1) Cek edge nginx

Di vm-docker:

```bash
sudo docker exec -it edge-nginx nginx -t
sudo docker logs edge-nginx --tail=100
```

#### 2) Cek NodePort dari vm-docker

```bash
curl -fsSI http://192.168.56.44:30080/ | head -n 1
curl -fsSI http://192.168.56.44:30081/api/products | head -n 1
curl -fsSI http://192.168.56.44:30082/api/products | head -n 1
```

Kalau NodePort tidak bisa diakses → cek UFW di vm-worker:

```bash
sudo ufw status verbose
sudo ufw allow 30080:30082/tcp
# atau ufw disable untuk lab
```

#### 3) Cek endpoint service K8s

Di vm-k8s:

```bash
kubectl -n threebody-prod get svc -o wide
kubectl -n threebody-prod get endpoints -o wide
kubectl -n threebody-prod get pod -o wide
```

Kalau endpoints kosong → berarti Pod belum Ready / label selector mismatch.

***

## 7) “Debug Komponen” paling cepat (per komponen)

### Harbor

```bash
curl -fsSI http://harbor.local:8080/v2/ | head -n 1
sudo docker compose -f /opt/harbor/docker-compose.yml ps
```

### Edge

```bash
sudo docker exec -it edge-nginx nginx -t
curl -kvs --resolve hit.local:443:127.0.0.1 https://hit.local/ 2>&1 | tail -n 30
```

### K8s cluster

```bash
kubectl get nodes -o wide
kubectl -n threebody-prod get pods -o wide
kubectl -n threebody-prod get svc -o wide
```

### MySQL

```bash
kubectl -n threebody-prod logs mysql-0 -c mysql --tail=200
kubectl -n threebody-prod exec -it mysql-0 -c mysql -- mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW DATABASES;"
```

***

## 8) Pola belajar “step-by-step” (biar kamu cepat hafal)

Kalau kamu latihan dari nol, ulangi dengan urutan ini:

1. **A (semua VM)**: hosts + DNS fix → `getent hosts` harus benar
2. **B (vm-docker)**: docker → harbor → `curl /v2` = 401
3. **C (vm-k8s)**: init cluster → nodes Ready
4. **D (vm-worker)**: join → node Ready → folder `/data/threebody/mysql` siap
5. **E (push)**: pipeline harus lewat build → push → deploy → healthcheck
6. **Edge test**: `https://hit.local` harus tembus frontend + API

***

Kalau kamu mau, langkah berikutnya aku bisa bikinin **“runbook debugging 1 halaman”** (super ringkas) khusus buat kamu, formatnya seperti:

* ✅ Target output yang harus terlihat
* ❌ Jika error X → jalankan 3 command ini → interpretasi outputnya

Tapi yang di atas ini sudah jadi “materi belajar lengkap” yang bisa kamu pakai kapan pun.
