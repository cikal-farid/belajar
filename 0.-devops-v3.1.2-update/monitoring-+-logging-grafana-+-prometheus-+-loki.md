# Monitoring + Logging (Grafana + Prometheus + Loki)

Dari yang kamu kirim **(output cluster + isi file project `three-body-problem-final.zip`)**, kondisinya gini:

### 1) Cek requirement tugas vs project/runbook kamu (yang sekarang)

Yang **SUDAH terpenuhi** dari file & kondisi cluster kamu:

* **CI/CD GitLab** ada (`.gitlab-ci.yml`) dengan stage **build → push → deploy**.
* **Build semua service** (frontend React, Go, Laravel) dilakukan di pipeline.
* **Push image ke Harbor** (`harbor.local:8080/threebody/...`) dan tagging `:$CI_COMMIT_SHORT_SHA` + `:latest`.
* **Deploy ke Kubernetes** (namespace `threebody-prod`) + rollout + migrate job + healthcheck.
* **Secret manager via GitLab Variables**: `HARBOR_USERNAME/PASSWORD`, `MYSQL_*`, `LARAVEL_APP_KEY`, `KUBECONFIG_PROD` (File).
* **HTTPS + rate limiting** sudah ada di **Edge Nginx** (`deploy/edge/nginx/conf.d/edge.conf`) untuk `/go/` dan `/laravel/`.

Yang **BELUM ada** (dan ini yang kamu minta lanjut sekarang):

* Namespace `monitoring` **kosong** (“No resources found”) → artinya **belum ada monitoring/logging stack**.
* Jadi requirement **#6 “Logging Service & utilitas server + Grafana dan log semua service terintegrasi”** memang **belum terpenuhi**.

Sekarang kita masuk tahap monitoring + logging yang rapi dan “sekali pasang”.

***

## 2) RUNBOOK Monitoring + Logging (Grafana + Prometheus + Loki)

Target akhir:

* **Metrics**: node vm-k8s & vm-worker + cluster metrics masuk Prometheus, tampil di Grafana.
* **Logs**: semua pod `threebody-prod` (frontend/go/laravel/mysql) masuk Loki, dibaca di Grafana.
* (Opsional tapi bagus) **vm-docker** (Harbor/Edge/Runner) ikut termonitor & log ikut terkirim.

> Semua perintah di bawah jalankan dari **vm-k8s** (kecuali bagian opsional vm-docker).

***

### A. Prasyarat cepat (wajib cek, biar NodePort bisa diakses)

Di **vm-worker** pastikan firewall tidak blok NodePort:

**Opsi paling simpel (lab):**

```bash
sudo ufw disable || true
```

**Kalau kamu tetap mau UFW aktif**, minimal buka port monitoring yang nanti dipakai:

```bash
sudo ufw allow 30030/tcp   # grafana
sudo ufw allow 30090/tcp   # prometheus
sudo ufw allow 31010/tcp   # loki (opsional kalau mau terima log dari vm-docker)
sudo ufw reload
sudo ufw status verbose
```

***

### B. Install Helm (di vm-k8s)

Pilih salah satu cara.

**Cara 1 (paling praktis):**

```bash
sudo snap install helm --classic
helm version
```

**Cara 2 (script resmi Helm):**

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | sudo bash
helm version
```

***

### C. Install Monitoring: kube-prometheus-stack (Prometheus + Grafana + exporter)

Chart ini memang “paket lengkap” untuk monitoring Kubernetes (Prometheus Operator, Grafana, Alertmanager, node-exporter, kube-state-metrics, dsb). ([artifacthub.io](https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack?utm_source=chatgpt.com))

#### C1) Add repo Helm + buat namespace

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl get ns monitoring >/dev/null 2>&1 || kubectl create ns monitoring
```

#### C2) Install stack

Aku sarankan release name: **kps** (biar label gampang).

```bash
helm upgrade --install kps prometheus-community/kube-prometheus-stack -n monitoring
```

#### C3) Pastikan pod monitoring naik

```bash
kubectl -n monitoring get pods -o wide
kubectl -n monitoring get svc -o wide
```

Tanda sukses: ada pod semacam `kps-kube-prometheus-stack-operator`, `kps-grafana`, `...prometheus...`, `...alertmanager...`, `...node-exporter...`.

***

### D. Akses Grafana & Prometheus

#### D1) Cara paling “nggak rewel” (port-forward)

Ini tidak mengubah service type, tapi harus jalan saat kamu akses.

**Grafana:**

```bash
kubectl -n monitoring port-forward svc/kps-grafana 3000:80 --address 0.0.0.0
```

**Prometheus:**\
Cari dulu service prometheus:

```bash
kubectl -n monitoring get svc | grep -i prometheus
```

Lalu port-forward service prometheus yang ada port 9090 (biasanya namanya mengandung `prometheus`):

```bash
kubectl -n monitoring port-forward svc/<NAMA_SVC_PROMETHEUS> 9090:9090 --address 0.0.0.0
```

#### D2) (Kalau kamu mau permanen) NodePort untuk Grafana

Paling mudah: patch service Grafana jadi NodePort 30030.

1. Lihat nama service Grafana:

```bash
kubectl -n monitoring get svc | grep -i grafana
```

2. Patch (anggap nama servicenya `kps-grafana`):

```bash
kubectl -n monitoring patch svc kps-grafana --type merge -p '{
  "spec": {
    "type": "NodePort",
    "ports": [{
      "name": "service",
      "port": 80,
      "targetPort": 3000,
      "nodePort": 30030
    }]
  }
}'
```

Akses dari vm-docker / host ke:

* `http://192.168.56.44:30030` (vm-worker) atau
* `http://192.168.56.43:30030` (vm-k8s, kalau scheduler mengizinkan service diakses dari sana)

#### D3) Ambil password admin Grafana

```bash
kubectl -n monitoring get secret kps-grafana -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

User default: `admin`

***

### E. Install Logging: Loki + Promtail (log semua pod masuk Grafana)

Penting: Loki **tidak punya auth bawaan**, jadi jangan diekspos bebas tanpa reverse proxy/auth. ([Grafana Labs](https://grafana.com/docs/loki/latest/operations/authentication/))

Promtail saat ini statusnya **deprecated (LTS)**, tapi masih umum dipakai untuk lab; alternatif modernnya adalah Grafana Alloy. ([Grafana Labs](https://grafana.com/docs/loki/latest/send-data/promtail/))

#### E1) Add repo Grafana

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

#### E2) Install Loki (single instance, cocok buat lab)

Pakai contoh “monolithic/single binary” sesuai guidance Helm Loki (replication\_factor=1). ([Grafana Labs](https://grafana.com/docs/loki/latest/setup/install/helm/install-monolithic/?utm_source=chatgpt.com))

```bash
cat > /tmp/loki-values.yaml <<'EOF'
deploymentMode: SingleBinary

loki:
  commonConfig:
    replication_factor: 1

singleBinary:
  replicas: 1

# matikan mode scalable/microservices biar ringan
backend:
  replicas: 0
read:
  replicas: 0
write:
  replicas: 0
EOF

helm upgrade --install loki grafana/loki -n monitoring -f /tmp/loki-values.yaml
```

Cek:

```bash
kubectl -n monitoring get pods -o wide | grep -i loki
kubectl -n monitoring get svc -o wide | grep -i loki
```

#### E3) Install Promtail (ambil log semua pod)

Chart promtail memang menyediakan `config.clients` untuk arahkan ke Loki. ([GitHub](https://github.com/grafana/helm-charts/blob/main/charts/promtail/README.md?utm_source=chatgpt.com))

```bash
cat > /tmp/promtail-values.yaml <<'EOF'
config:
  clients:
    - url: http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push
EOF

helm upgrade --install promtail grafana/promtail -n monitoring -f /tmp/promtail-values.yaml
```

Cek:

```bash
kubectl -n monitoring get pods -o wide | grep -i promtail
```

***

### F. Integrasikan Loki ke Grafana (biar “log semua service kebaca di Grafana”)

1. Buka Grafana (port-forward atau NodePort).
2. **Connections → Data sources → Add data source → Loki**
3. URL isi:

* Kalau dari dalam cluster: `http://loki-gateway.monitoring.svc.cluster.local`
* Kalau kamu nanti expose via NodePort/reverse proxy, menyesuaikan.

4. Save & Test.

#### Contoh query LogQL yang langsung berguna

Masuk menu **Explore** → pilih datasource Loki:

*   Semua log namespace aplikasi:

    ```
    {namespace="threebody-prod"}
    ```
*   Log laravel saja (biasanya label minimal: namespace + pod/container):

    ```
    {namespace="threebody-prod", container="laravel"}
    ```
*   Cari error:

    ```
    {namespace="threebody-prod"} |= "error"
    ```

Kalau kamu bingung label apa saja yang ada, di Explore biasanya ada **label browser** untuk klik-klik.

***

## 3) (Opsional tapi direkomendasikan) Biar “3 VM” benar-benar ikut termonitor

Saat ini kube-prometheus-stack **otomatis monitor vm-k8s & vm-worker** (karena mereka node K8s).\
**vm-docker** belum.

### Opsi G1: monitor vm-docker pakai node-exporter container + ServiceMonitor

#### G1.1) Di vm-docker: jalankan node-exporter

```bash
docker run -d --name node-exporter --restart unless-stopped \
  --net=host --pid=host \
  -v "/:/host:ro,rslave" \
  quay.io/prometheus/node-exporter:v1.8.1 \
  --path.rootfs=/host
```

Cek:

```bash
curl -fsS http://127.0.0.1:9100/metrics | head
```

#### G1.2) Di vm-k8s: buat Endpoints+Service yang menunjuk ke vm-docker:9100

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Endpoints
metadata:
  name: vm-docker-node-exporter
  namespace: monitoring
subsets:
- addresses:
  - ip: 192.168.56.42
  ports:
  - name: metrics
    port: 9100
---
apiVersion: v1
kind: Service
metadata:
  name: vm-docker-node-exporter
  namespace: monitoring
  labels:
    app: vm-docker-node-exporter
spec:
  ports:
  - name: metrics
    port: 9100
    targetPort: 9100
EOF
```

#### G1.3) Buat ServiceMonitor

> Penting: label `release: kps` biasanya dibutuhkan supaya Prometheus Operator “nangkep” ServiceMonitor dari Helm release kamu.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: vm-docker-node-exporter
  namespace: monitoring
  labels:
    release: kps
spec:
  selector:
    matchLabels:
      app: vm-docker-node-exporter
  namespaceSelector:
    matchNames: ["monitoring"]
  endpoints:
  - port: metrics
    interval: 15s
EOF
```

Setelah itu, di Grafana dashboard node-exporter harus mulai muncul “vm-docker”.

***

## 4) Checklist validasi (biar kamu yakin sudah beres)

Jalankan di **vm-k8s**:

```bash
kubectl -n monitoring get pods -o wide
kubectl -n monitoring get svc -o wide
kubectl -n threebody-prod get pods -o wide
```

Lalu validasi dari Grafana:

* Dashboard Kubernetes/Nodes muncul (metrics masuk)
* Explore Loki bisa query `{namespace="threebody-prod"}` (logs masuk)

***

Kalau kamu jalankan runbook ini dan kirim balik output 2 command ini:

1. `kubectl -n monitoring get pods -o wide`
2. `kubectl -n monitoring get svc -o wide`

…aku bisa “cek cepat” apakah Loki/Promtail sudah ngirim log bener (biasanya masalahnya cuma URL gateway/service name kalau chart Loki beda bentuk).
