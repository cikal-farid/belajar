# Monitoring + Logging (Grafana + Prometheus + Loki)

Berikut **RUNBOOK FINAL – Monitoring + Logging (Grafana + Prometheus + Loki + Promtail)** yang sudah aku rapikan dari semua kendala yang kamu alami (bucketNames, schema\_config, rootfs read-only, memcached cache berat, port-forward aman, datasource Loki, dll). Ini versi **paling ringan & aman untuk lab 2 node** seperti setup kamu.

> **Target kamu:**
>
> * Metrics: Prometheus (kube-prometheus-stack)
> * Dashboard: Grafana
> * Logging: Loki (SingleBinary ringan, tanpa PVC)
> * Log Collector: Promtail (DaemonSet, ambil log semua pod/node → Loki)
> * Akses UI dari host: via **systemd port-forward** di vm-k8s (bind ke IP vm-k8s saja, bukan 0.0.0.0)

***

## 0) Konteks & Asumsi (sesuai percakapan)

* Cluster 2 node:
  * **vm-k8s** (control-plane) IP: `192.168.56.43`
  * **vm-worker** (worker) IP: `192.168.56.44`
* Kamu menjalankan `kubectl/helm` dari **vm-k8s**.
* Namespace: `monitoring`
* Release Helm:
  * `kps` = kube-prometheus-stack
  * `loki` = grafana/loki
  * `promtail` = grafana/promtail

> Catatan: kita pakai mode “lab ringan” → **tanpa PVC** (data logs akan hilang kalau pod restart / node reboot). Ini sengaja supaya stabil di resource kecil.

***

## 1) Pre-check Wajib (cek dulu sebelum install)

Jalankan di **vm-k8s**:

```bash
kubectl get nodes -o wide
kubectl get pods -A | head
```

Pastikan:

* Node **Ready**
* DNS & image pull aman (kalau sering ImagePullBackOff → bereskan DNS dulu seperti sebelumnya)

***

## 2) Install Helm Repo

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

***

## 3) Install Monitoring Metrics: kube-prometheus-stack (Prometheus + Grafana + Alertmanager)

### 3.1 Install

```bash
kubectl create ns monitoring 2>/dev/null || true

helm upgrade --install kps prometheus-community/kube-prometheus-stack \
  -n monitoring
```

### 3.2 Tunggu pod running

```bash
kubectl -n monitoring get pods -o wide -w
```

Minimal yang kamu lihat harus Running:

* `prometheus-kps-...`
* `kps-grafana-...`
* `alertmanager-kps-...`
* `kps-kube-state-metrics...`
* `kps-prometheus-node-exporter...` (di 2 node)

> Kalau grafana sempat “Readiness/Liveness failed connect refused”, itu normal saat startup plugin. Biasanya stabil setelah beberapa menit.

***

## 4) Akses UI secara Aman via systemd Port-Forward (Auto setelah reboot)

**Kenapa begini?** Karena kamu mau akses dari host tanpa bikin NodePort/Ingress, dan lebih aman karena kita bind ke IP vm-k8s saja.

### 4.1 Grafana Port-Forward (3000)

```bash
sudo tee /etc/systemd/system/kpf-grafana.service >/dev/null <<'EOF'
[Unit]
Description=Kubernetes Port-Forward Grafana (monitoring)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
Environment=KUBECONFIG=/etc/kubernetes/admin.conf
ExecStart=/usr/bin/kubectl -n monitoring port-forward svc/kps-grafana 3000:80 --address 192.168.56.43
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF
```

### 4.2 Prometheus Port-Forward (9090)

```bash
sudo tee /etc/systemd/system/kpf-prometheus.service >/dev/null <<'EOF'
[Unit]
Description=Kubernetes Port-Forward Prometheus (monitoring)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
Environment=KUBECONFIG=/etc/kubernetes/admin.conf
ExecStart=/usr/bin/kubectl -n monitoring port-forward svc/kps-kube-prometheus-stack-prometheus 9090:9090 --address 192.168.56.43
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now kpf-grafana.service
sudo systemctl enable --now kpf-prometheus.service
```

### 4.3 Cek listener + endpoint

```bash
ss -lntp | egrep ':3000|:9090' || true
curl -fsSI http://192.168.56.43:3000/login | head -n 1
curl -fsS  http://192.168.56.43:9090/-/ready | head -n 1
```

### 4.4 Login Grafana

Ambil password admin:

```bash
kubectl -n monitoring get secret kps-grafana -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

Buka:

* Grafana: `http://192.168.56.43:3000`
* Prometheus: `http://192.168.56.43:9090`

***

## 5) Install Logging: Loki (versi ringan & stabil untuk lab)

> Ini bagian yang dulu error dan sudah kita “kunci” biar nggak kejadian lagi:

* Chart minta `bucketNames`
* Chart minta `schema_config` → kita pakai `useTestSchema: true`
* Error **read-only filesystem /var/loki** → kita set `readOnlyRootFilesystem: false` + mount `emptyDir` ke `/var/loki`
* Memcached cache bikin Pending karena request memory besar → kita **disable** chunksCache & resultsCache
* Helm test memaksa canary → kita **disable test** dan **disable canary**

### 5.1 Values Loki (LAB)

```bash
cat > /tmp/loki-values-lab.yaml <<'EOF'
deploymentMode: SingleBinary

test:
  enabled: false

loki:
  auth_enabled: false
  useTestSchema: true

  containerSecurityContext:
    readOnlyRootFilesystem: false

  commonConfig:
    replication_factor: 1

  storage:
    type: filesystem
    bucketNames:
      chunks: chunks
      ruler: ruler
      admin: admin

singleBinary:
  replicas: 1
  persistence:
    enabled: false

  extraVolumes:
    - name: loki-data
      emptyDir: {}
  extraVolumeMounts:
    - name: loki-data
      mountPath: /var/loki

gateway:
  enabled: true

minio:
  enabled: false

# versi ringan: matikan komponen berat
lokiCanary:
  enabled: false

chunksCache:
  enabled: false

resultsCache:
  enabled: false

backend: { replicas: 0 }
read: { replicas: 0 }
write: { replicas: 0 }
EOF
```

### 5.2 Install/Upgrade Loki

```bash
helm upgrade --install loki grafana/loki -n monitoring -f /tmp/loki-values-lab.yaml
kubectl -n monitoring get pods -o wide | egrep -i 'loki|gateway'
```

**Wajib sukses:**

* `loki-0` harus **2/2 Running**
* `loki-gateway` **Running**

Cek cepat:

```bash
kubectl -n monitoring get pod loki-0
kubectl -n monitoring logs loki-0 -c loki --tail=50
```

***

## 6) Install Promtail (ambil log semua node/pod → push ke Loki)

> Kamu sudah pakai promtail dan berhasil. Ini versi ringan sesuai lab.

### 6.1 Values Promtail

```bash
cat > /tmp/promtail-values.yaml <<'EOF'
config:
  clients:
    - url: http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 256Mi
EOF
```

### 6.2 Install/Upgrade Promtail

```bash
helm upgrade --install promtail grafana/promtail -n monitoring -f /tmp/promtail-values.yaml
kubectl -n monitoring get pods -o wide | egrep -i 'promtail'
kubectl -n monitoring get ds/promtail
```

***

## 7) Tambahkan Datasource Loki ke Grafana (via kube-prometheus-stack)

Ini biar Grafana otomatis punya datasource Loki tanpa klik manual.

```bash
cat > /tmp/kps-datasource-loki.yaml <<'EOF'
grafana:
  additionalDataSources:
    - name: Loki
      type: loki
      access: proxy
      url: http://loki-gateway.monitoring.svc.cluster.local
      isDefault: false
      jsonData:
        maxLines: 1000
EOF

helm upgrade kps prometheus-community/kube-prometheus-stack -n monitoring -f /tmp/kps-datasource-loki.yaml
```

***

## 8) (Opsional) Port-forward Loki Gateway untuk akses dari luar cluster

Kalau kamu mau test dari host (bukan dari Grafana), bikin service systemd:

```bash
sudo tee /etc/systemd/system/kpf-loki.service >/dev/null <<'EOF'
[Unit]
Description=Kubernetes Port-Forward Loki Gateway (monitoring)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
Environment=KUBECONFIG=/etc/kubernetes/admin.conf
ExecStart=/usr/bin/kubectl -n monitoring port-forward svc/loki-gateway 3100:80 --address 192.168.56.43
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now kpf-loki.service
ss -lntp | egrep ':3100' || true
```

**Catatan penting (sesuai kendala kamu):**\
Karena ini lewat **gateway**, endpoint health `/ready` bisa **404** (normal). Untuk test, pakai endpoint Loki via gateway path `/loki/...`, contoh:

```bash
curl -fsS http://192.168.56.43:3100/loki/api/v1/status/buildinfo | head
```

***

## 9) Cara Melihat Monitoring di UI Grafana (yang kamu bingung tadi)

### 9.1 Lihat dashboard bawaan

Di Grafana:

* Menu kiri → **Dashboards**
* Pilih salah satu yang paling penting:
  * **Kubernetes / Compute Resources / Cluster**
  * **Kubernetes / Compute Resources / Namespace (Pods)**
  * **Kubernetes / API server**
  * **CoreDNS**
  * **etcd**
  * **Alertmanager / Overview**

### 9.2 Biar buka `http://IP:3000/` langsung masuk dashboard monitoring (set Home Dashboard)

* Klik user icon kanan atas → **Preferences**
* Set **Home dashboard** = misalnya **Kubernetes / Compute Resources / Cluster**
* Save

> Setelah itu, buka grafana langsung tampil monitoring.

### 9.3 Lihat Logs (Loki)

* Menu kiri → **Explore**
* Pilih datasource: **Loki**
* Coba filter:
  * `namespace="threebody-prod"`
  * pilih `pod="laravel-..."`

***

## 10) Verifikasi Final (Checklist “AMAN”)

Jalankan ini, hasilnya harus sesuai:

```bash
# Semua komponen utama running
kubectl -n monitoring get pods -o wide | egrep -i 'grafana|prometheus|alertmanager|loki|promtail'

# Loki wajib 2/2
kubectl -n monitoring get pod loki-0

# Promtail wajib 2 pod (daemonset 2 node)
kubectl -n monitoring get ds promtail

# Listener port-forward
ss -lntp | egrep ':3000|:9090|:3100' || true

# Endpoint basic
curl -fsSI http://192.168.56.43:3000/login | head -n 1
curl -fsS  http://192.168.56.43:9090/-/ready | head -n 1
```

***

## 11) Hal yang WAJIB kamu “SAVE” biar runbook nggak hilang

Supaya nggak ulang dari nol kalau lupa, **jangan simpan hanya di /tmp**.

Buat folder repo, contoh:

```bash
mkdir -p ops/monitoring
cp /tmp/loki-values-lab.yaml ops/monitoring/
cp /tmp/promtail-values.yaml ops/monitoring/
cp /tmp/kps-datasource-loki.yaml ops/monitoring/
```

Lalu commit ke Git supaya permanen.

***

## 12) Troubleshooting cepat (berdasarkan error yang kamu alami)

#### A) `loki-0 CrashLoopBackOff: mkdir /var/loki read-only file system`

✅ Fix: pastikan values punya:

* `loki.containerSecurityContext.readOnlyRootFilesystem: false`
* mount `emptyDir` ke `/var/loki`

#### B) `loki-chunks-cache Pending Insufficient memory/cpu`

✅ Fix: disable caches:

* `chunksCache.enabled: false`
* `resultsCache.enabled: false`

#### C) Helm bilang “must provide schema\_config”

✅ Fix: `loki.useTestSchema: true`

#### D) Helm bilang “Please define bucketNames”

✅ Fix: isi:

```yaml
loki:
  storage:
    bucketNames:
      chunks: chunks
      ruler: ruler
      admin: admin
```

#### E) Promtail chart deprecated

Itu warning saja, masih jalan. Untuk lab aman.

***

Kalau kamu mau, aku bisa lanjutkan runbook “final keseluruhan project” (CI/CD + K8s deploy + Monitoring + Logging) jadi 1 dokumen rapi. Tapi untuk request kamu barusan, ini sudah **final monitoring + logging** dan sudah “dikunci” supaya tidak mengulang error-error kamu sebelumnya.
