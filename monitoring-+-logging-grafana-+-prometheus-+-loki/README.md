# Monitoring + Logging (Grafana + Prometheus + Loki)

## Monitoring + Logging (Grafana + Prometheus + Loki)

Sip — dari output kamu, prosesnya **sudah aman** dan akar masalahnya ketemu: **`baltocdn.com` memang tidak bisa di-resolve** di environment kamu, tapi **`get.helm.sh` + `github.com` bisa**. Jadi runbook harus **JANGAN bergantung ke repo APT baltocdn**, dan default install Helm pakai **binary dari get.helm.sh** (yang terbukti sukses di kamu).

Di bawah ini **runbook final (monitoring + logging) versi “anti gagal”** sesuai kendala yang kamu alami.

***

### RUNBOOK FINAL — Monitoring + Logging (2 Node Lab)

Target:

* **Monitoring**: kube-prometheus-stack (Prometheus + Grafana + Node Exporter + KSM)
* **Logging**: Loki (single binary, lab ringan) + Promtail (kirim log pod ke Loki)
* **Akses UI**: via **systemd port-forward** dari vm-k8s (IP 192.168.56.43)

> Semua perintah dijalankan di **vm-k8s** sebagai user `cikal` (kecuali kalau tertulis lain).

***

#### 0) Prasyarat cek cepat (wajib)

```bash
kubectl get nodes -o wide
kubectl get pods -A | head
```

Harusnya node **Ready**.

***

#### 1) DNS fix (wajib supaya Helm chart repo bisa diakses)

> Ini mencegah error mirip “Could not resolve host …” (seperti baltocdn.com).

```bash
sudo tee /etc/systemd/resolved.conf >/dev/null <<'EOF'
[Resolve]
DNS=1.1.1.1 8.8.8.8
FallbackDNS=9.9.9.9 8.8.4.4
DNSStubListener=yes
EOF

sudo systemctl restart systemd-resolved
sudo resolvectl flush-caches
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

Cek:

```bash
cat /etc/resolv.conf
getent hosts github.com || true
getent hosts get.helm.sh || true
```

> Catatan: `baltocdn.com` boleh saja tetap gagal — runbook ini **tidak butuh baltocdn**.

***

#### 2) Helm install (VERSI AMAN — binary dari get.helm.sh)

> Ini yang kamu sudah buktikan berhasil.

**Bersihkan sisa repo helm APT (kalau pernah coba):**

```bash
sudo rm -f /etc/apt/sources.list.d/helm-stable-debian.list
sudo rm -f /etc/apt/keyrings/helm.gpg
sudo apt-get update
```

**Install Helm binary:**

```bash
cd /tmp
VER="v3.14.4"
curl -fsSLO "https://get.helm.sh/helm-${VER}-linux-amd64.tar.gz"
tar -xzf "helm-${VER}-linux-amd64.tar.gz"
sudo install -m 0755 linux-amd64/helm /usr/local/bin/helm
helm version
which helm
```

***

#### 3) Add repo chart (Prometheus + Grafana)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

***

#### 4) Install Monitoring (kube-prometheus-stack)

```bash
kubectl get ns monitoring >/dev/null 2>&1 || kubectl create ns monitoring

helm upgrade --install kps prometheus-community/kube-prometheus-stack -n monitoring
```

Tunggu siap:

```bash
kubectl -n monitoring get pods -o wide
```

Ambil password Grafana:

```bash
kubectl -n monitoring get secret kps-grafana -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

***

#### 5) Akses Grafana + Prometheus pakai systemd (TANPA port-forward manual)

> Pakai bind ke IP host-only **192.168.56.43** (lebih aman daripada 0.0.0.0)

**Grafana**

```bash
sudo tee /etc/systemd/system/kpf-grafana.service >/dev/null <<'EOF'
[Unit]
Description=Kubernetes Port-Forward Grafana (monitoring)
After=network-online.target kubelet.service
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

**Prometheus**

```bash
sudo tee /etc/systemd/system/kpf-prometheus.service >/dev/null <<'EOF'
[Unit]
Description=Kubernetes Port-Forward Prometheus (monitoring)
After=network-online.target kubelet.service
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
sudo systemctl enable --now kpf-grafana kpf-prometheus
```

Cek:

```bash
ss -lntp | egrep ':3000|:9090'
curl -fsS http://192.168.56.43:3000/login | head -n 1
curl -fsS http://192.168.56.43:9090/-/ready
```

***

#### 6) Install Loki (VERSI LAB RINGAN + anti CrashLoop read-only)

Ini versi yang **kamu sudah buktikan stabil**:

* `test.enabled=false` (biar tidak maksa canary)
* `readOnlyRootFilesystem=false`
* mount `/var/loki` pakai `emptyDir`
* matikan cache chunks/results + canary (biar tidak Pending / makan RAM besar)

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

helm upgrade --install loki grafana/loki -n monitoring -f /tmp/loki-values-lab.yaml
```

Tunggu:

```bash
kubectl -n monitoring get pod loki-0 -w
kubectl -n monitoring get svc | egrep -i 'loki|memberlist|gateway'
```

***

#### 7) Install Promtail (kirim log pod ke Loki)

> Chart promtail memang deprecated, tapi **masih jalan** buat lab. Ini sesuai yang kamu pakai.

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

helm upgrade --install promtail grafana/promtail -n monitoring -f /tmp/promtail-values.yaml
kubectl -n monitoring get ds/promtail
kubectl -n monitoring get pods -o wide | egrep -i promtail
```

***

#### 8) Tambahkan Loki sebagai Data Source Grafana (via Helm upgrade kps)

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
kubectl -n monitoring rollout status deploy/kps-grafana
```

***

#### 9) Akses Loki gateway dari luar (systemd port-forward)

> Penting: karena yang kamu forward adalah **gateway (nginx)**, endpoint yang benar **pakai prefix `/loki`**, bukan `/ready`.

```bash
sudo tee /etc/systemd/system/kpf-loki.service >/dev/null <<'EOF'
[Unit]
Description=Kubernetes Port-Forward Loki Gateway (monitoring)
After=network-online.target kubelet.service
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
```

Cek:

```bash
ss -lntp | egrep ':3100'
curl -fsS "http://192.168.56.43:3100/loki/api/v1/status/buildinfo" | head
```

***

### 10) Cara lihat tampilan monitoring di Grafana (buat pemula)

1. Buka browser di host kamu:

* Grafana: `http://192.168.56.43:3000`
* User: `admin`
* Password: (dari secret `kps-grafana`)

2. Setelah login:

* Klik menu kiri **Dashboards → Browse**
* Cari folder **Kubernetes / Compute Resources** atau **Node Exporter**
  * contoh yang enak dilihat:
    * **Kubernetes / Compute Resources / Namespace (Pods)**
    * **Kubernetes / Compute Resources / Node (Pods)**
    * **Node Exporter / Nodes**

3. Kalau kamu mau **URL langsung buka dashboard**:

* Buka dashboard yang kamu mau
* Copy URL-nya (bookmark)
* Atau set jadi Home:
  * Klik avatar (pojok kiri bawah) → **Preferences**
  * **Home dashboard** → pilih dashboard yang kamu suka → Save\
    (mulai sekarang buka `http://192.168.56.43:3000/` akan langsung ke dashboard itu)

***

### 11) Cara lihat LOG di Grafana (Loki)

1. Menu kiri → **Explore**
2. Pilih Data source: **Loki**
3. Contoh query cepat:

*   Semua log namespace production kamu:

    ```
    {namespace="threebody-prod"}
    ```
*   Log pod laravel:

    ```
    {namespace="threebody-prod", app="laravel"}
    ```

Kalau label `app` beda, klik “Label browser” di Explore untuk lihat label yang tersedia.

***

### 12) Perlu “di-save” apa saja? (supaya restart VM aman)

✅ Wajib disimpan:

* File systemd:
  * `/etc/systemd/system/kpf-grafana.service`
  * `/etc/systemd/system/kpf-prometheus.service`
  * `/etc/systemd/system/kpf-loki.service`
* Values helm (biar bisa re-install identik):
  *   `/tmp/loki-values-lab.yaml` → pindahkan ke folder permanen, contoh:

      ```bash
      sudo mkdir -p /opt/runbook/monitoring
      sudo cp /tmp/loki-values-lab.yaml /opt/runbook/monitoring/
      sudo cp /tmp/promtail-values.yaml /opt/runbook/monitoring/
      sudo cp /tmp/kps-datasource-loki.yaml /opt/runbook/monitoring/
      ```

⚠️ Catatan penting:

* Loki kamu sekarang **tanpa PVC** → log akan **hilang kalau pod restart**. Untuk lab aman, tapi kalau kamu mau “log awet”, nanti kita upgrade pakai storageClass + persistence.

***

#### 13) Verifikasi final (sekali jalan)

```bash
# monitoring pods OK
kubectl -n monitoring get pods -o wide

# port-forward systemd OK
systemctl --no-pager status kpf-grafana kpf-prometheus kpf-loki
ss -lntp | egrep ':3000|:9090|:3100'

# endpoint OK
curl -fsS http://192.168.56.43:9090/-/ready
curl -fsS http://192.168.56.43:3100/loki/api/v1/status/buildinfo | head
```

***

Kalau kamu mau, aku bisa rapikan runbook ini jadi **format “sekali copy-paste dari awal”** (dengan blok A–Z dan check/expected output) supaya kamu tinggal ikuti tanpa mikir.
