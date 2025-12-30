# Monitoring + Logging (Grafana + Prometheus + Loki)

Oke, **cek dulu sebelum perbaikan**: dari output kamu, cluster **sudah sehat** (2 node Ready, core pods Running). Error yang kamu temui **bukan dari K8s/monitoring-nya**, tapi karena **Helm belum terinstall** di `vm-k8s`:

> `-bash: helm: command not found`

Jadi runbook sebelumnya gagal di langkah repo karena tool `helm` belum ada.

Di bawah ini aku tulis ulang **RUNBOOK FINAL (super detail, pemula-friendly, anti melenceng)** khusus untuk **Monitoring + Logging + utilitas akses UI**, dan aku mulai dari **install Helm dulu** karena itu yang sekarang bikin kamu stuck.

***

## RUNBOOK FINAL — Monitoring + Logging (K8s 2 node, ringan & aman)

### A) Pre-check (wajib, kamu sudah OK)

Di `vm-k8s`:

```bash
kubectl get nodes -o wide
kubectl get pods -A | head
```

✅ Lanjut kalau node **Ready** (punyamu sudah Ready).

***

### B) Install Helm di vm-k8s (INI YANG KURANG)

Kamu pakai Ubuntu 24.04, cara paling aman: **pakai apt**.

#### B1) Install helm via apt (recommended)

```bash
sudo apt-get update
sudo apt-get install -y helm
helm version
```

Kalau `apt install helm` ternyata tidak menemukan paket (kadang repo OS beda), pakai cara resmi Helm:

#### B2) Install helm via script resmi (fallback, aman juga)

```bash
sudo apt-get update
sudo apt-get install -y curl ca-certificates gnupg

curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

✅ Kalau `helm version` keluar, berarti **Helm sudah beres**.

***

### C) Tambahkan Helm repo (prometheus + grafana)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

***

### D) Buat namespace monitoring

```bash
kubectl create ns monitoring 2>/dev/null || true
kubectl get ns | grep monitoring
```

***

### E) Install kube-prometheus-stack (Prometheus + Grafana + Alertmanager)

> Ini yang bikin dashboard Kubernetes otomatis banyak seperti screenshot kamu.

#### E1) Install

```bash
helm upgrade --install kps prometheus-community/kube-prometheus-stack -n monitoring
```

#### E2) Tunggu sampai Running

```bash
kubectl -n monitoring get pods -o wide -w
```

Minimal yang harus Running:

* `kps-grafana-...`
* `prometheus-kps-...`
* `alertmanager-kps-...`
* `kps-kube-state-metrics...`
* `kps-prometheus-node-exporter...` (2 pod, tiap node 1)

***

### F) Akses Grafana & Prometheus dari host (tanpa NodePort/Ingress)

Kita pakai **systemd port-forward**, dan **bind ke IP vm-k8s** (lebih aman daripada 0.0.0.0).

#### F1) Service Port-forward Grafana (3000)

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

#### F2) Service Port-forward Prometheus (9090)

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
```

#### F3) Aktifkan

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now kpf-grafana
sudo systemctl enable --now kpf-prometheus
```

#### F4) Cek listener & endpoint

```bash
ss -lntp | egrep ':3000|:9090' || true
curl -fsSI http://192.168.56.43:3000/login | head -n 1
curl -fsS  http://192.168.56.43:9090/-/ready | head -n 1
```

#### F5) Login Grafana

Password admin:

```bash
kubectl -n monitoring get secret kps-grafana -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

Akses:

* Grafana: `http://192.168.56.43:3000`
* Prometheus: `http://192.168.56.43:9090`

***

### G) Install Loki (Logging) — versi ringan & stabil (sesuai masalah kamu sebelumnya)

**Ini versi yang sudah terbukti menghindari error yang kamu alami:**

* wajib `bucketNames`
* wajib `useTestSchema: true`
* fix read-only `/var/loki`
* disable caches berat + disable canary + disable helm test

#### G1) Buat values Loki lab

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

#### G2) Install Loki

```bash
helm upgrade --install loki grafana/loki -n monitoring -f /tmp/loki-values-lab.yaml
```

#### G3) Validasi Loki

```bash
kubectl -n monitoring get pods -o wide | egrep -i 'loki|gateway'
kubectl -n monitoring get pod loki-0
```

✅ Sukses kalau:

* `loki-0` = **2/2 Running**
* `loki-gateway` = **Running**

***

### H) Install Promtail (collector log) — dorong log ke Loki

#### H1) Values promtail ringan

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

#### H2) Install promtail

```bash
helm upgrade --install promtail grafana/promtail -n monitoring -f /tmp/promtail-values.yaml
kubectl -n monitoring get pods -o wide | egrep -i 'promtail'
kubectl -n monitoring get ds promtail
```

✅ Harus ada 2 pod promtail (karena 2 node).

***

### I) Tambahkan DataSource Loki ke Grafana (biar otomatis muncul di Explore)

#### I1) Buat values tambahan untuk kps

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
```

#### I2) Upgrade kps

```bash
helm upgrade kps prometheus-community/kube-prometheus-stack -n monitoring -f /tmp/kps-datasource-loki.yaml
```

***

### J) (Opsional) Akses Loki gateway dari host (buat test manual)

> Ini opsional. Grafana sebenarnya cukup akses internal service.

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
sudo systemctl enable --now kpf-loki
ss -lntp | egrep ':3100' || true
```

Test:

```bash
curl -fsS http://192.168.56.43:3100/loki/api/v1/status/buildinfo | head
```

***

## K) Cara “sekali akses URL langsung muncul monitoring”

Ini yang kamu tanya di UI.

Grafana default masuk halaman home, tapi kamu bisa set **Home Dashboard** supaya begitu buka `http://IP:3000` langsung muncul dashboard monitoring.

#### K1) Set Home Dashboard dari UI

1. Grafana → menu kiri **Dashboards**
2. Pilih dashboard yang kamu mau jadi default (contoh:\
   **Kubernetes / Compute Resources / Cluster**)
3. Klik **⭐ Star** (biar gampang dicari)
4. Klik icon user kanan atas → **Preferences**
5. Set **Home dashboard** → pilih dashboard yang kamu bintangin
6. Save

✅ Hasil: buka `http://192.168.56.43:3000/` langsung tampil dashboard itu.

***

## L) Checklist Final (harus hijau semua)

Jalankan:

```bash
kubectl -n monitoring get pods -o wide | egrep -i 'grafana|prometheus|alertmanager|loki|promtail'
kubectl -n monitoring get pod loki-0
kubectl -n monitoring get ds promtail
ss -lntp | egrep ':3000|:9090|:3100' || true
curl -fsSI http://192.168.56.43:3000/login | head -n 1
curl -fsS  http://192.168.56.43:9090/-/ready | head -n 1
```

***

## M) WAJIB “SAVE” (supaya tidak hilang & tidak ngulang)

Jangan taruh hanya di `/tmp`, pindahkan ke folder permanen:

```bash
mkdir -p ~/ops/monitoring
cp /tmp/loki-values-lab.yaml ~/ops/monitoring/
cp /tmp/promtail-values.yaml ~/ops/monitoring/
cp /tmp/kps-datasource-loki.yaml ~/ops/monitoring/
```

Kalau repo git kamu ada, commit file ini.

***

### Sekarang langkah kamu berikutnya (tanpa tanya ulang)

Karena error kamu saat ini **helm belum ada**, lakukan ini dulu:

```bash
sudo apt-get update
sudo apt-get install -y helm
helm version
```

Kalau `helm version` sudah keluar, lanjutkan ke bagian **C**.

Kalau setelah install helm ada error lain (misal image pull, pod pending, dsb), kirim output step yang error itu, dan aku akan perbaiki sesuai pola kendala yang sudah terjadi sebelumnya.
