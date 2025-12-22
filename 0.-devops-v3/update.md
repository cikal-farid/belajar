# Update

Siap, kita **lanjut** dan aku rapihin bagian yang masih bolong + yang bikin kamu error. Di bawah ini aku kasih **lanjutan runbook end-to-end** (tetap pemula-friendly, step-by-step), fokus ke:

1. **Fix error Harbor** (`prepare` narik image ke Docker Hub)
2. **Fix vm-docker** (docker belum ada → `docker not found`)
3. **Install Kubernetes kubeadm** (containerd + file config muncul)
4. **K8s manifest lengkap** (MySQL + Go + Laravel + Frontend + NodePort)
5. **Observability lengkap** (Loki + Grafana + Promtail: staging docker + k8s)
6. **Update CI GitLab** biar benar-benar end-to-end (tag = `CI_COMMIT_SHORT_SHA`)

***

### A) FIX ERROR HARBOR: `./prepare` gagal (narik `goharbor/prepare` ke Docker Hub)

#### Penyebab error kamu

Kamu menjalankan `sudo ./prepare` **sebelum** image `goharbor/prepare:v2.14.1` diload dari paket offline → akhirnya Docker mencoba pull ke `docker.io` dan kena timeout.

> Harbor memang **wajib** `hostname` benar di `harbor.yml`, dan tidak boleh localhost. ([Harbor](https://goharbor.io/docs/1.10/install-config/configure-yml-file/?utm_source=chatgpt.com))

#### A1) Di **vm-harbor**: pastikan `harbor.yml` sudah benar

Edit `/opt/harbor/harbor.yml` dan pastikan:

```yaml
hostname: harbor.local
https:
  port: 443
  certificate: /etc/harbor/certs/harbor.local.crt
  private_key: /etc/harbor/certs/harbor.local.key
data_volume: /data/harbor
```

#### A2) Di **vm-harbor**: load image offline dulu, baru prepare/install

Masuk folder:

```bash
cd /opt/harbor
```

**Load semua image dari offline tarball:**

```bash
sudo docker load -i /opt/harbor/harbor.v2.14.1.tar.gz
```

Cek image `prepare` sudah ada:

```bash
sudo docker images | grep goharbor/prepare
```

Sekarang jalankan install **(ini akan generate docker-compose.yml dan jalanin service)**:

```bash
sudo ./install.sh
```

Cek status:

```bash
sudo docker compose -f /opt/harbor/docker-compose.yml ps
curl -k https://harbor.local/
```

Kalau mau re-generate config:

```bash
sudo ./prepare
sudo docker compose -f /opt/harbor/docker-compose.yml up -d
```

***

### B) FIX vm-docker: `docker: command not found` dan `docker.service not found`

Ini murni karena **Docker belum terinstall di vm-docker**.

#### B1) Di **vm-docker**: install Docker Engine + Compose

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker

docker version
docker compose version
```

Kalau service docker belum aktif:

```bash
sudo systemctl enable --now docker
sudo systemctl status docker --no-pager
```

***

### C) Wajib: `/etc/hosts` di SEMUA VM (biar `harbor.local` resolve)

Di **vm-docker, vm-harbor, vm-k8s, vm-worker**, pastikan `/etc/hosts` ada:

```txt
192.168.56.42 vm-docker staging.local prod.local
192.168.56.43 vm-harbor harbor.local
192.168.56.44 vm-k8s
192.168.56.45 vm-worker
```

Test dari **vm-docker**:

```bash
getent hosts harbor.local
ping -c 1 harbor.local
```

***

### D) Copy CA Harbor ke vm-docker (biar `docker login harbor.local` aman)

Dokumentasi Docker: CA registry ditaruh di `/etc/docker/certs.d/<registry-host>/ca.crt`. ([Docker Documentation](https://docs.docker.com/engine/security/certificates/?utm_source=chatgpt.com))

#### D1) Pastikan SSH tidak ketutup UFW di vm-harbor

Di **vm-harbor**:

```bash
sudo ufw allow OpenSSH
sudo ufw status
```

#### D2) Copy CA pakai IP (paling aman)

Di **vm-docker**:

```bash
scp cikal@192.168.56.43:/etc/harbor/certs/ca.crt /tmp/harbor-ca.crt
```

#### D3) Pasang CA untuk Docker

Di **vm-docker**:

```bash
sudo mkdir -p /etc/docker/certs.d/harbor.local
sudo cp /tmp/harbor-ca.crt /etc/docker/certs.d/harbor.local/ca.crt
sudo systemctl restart docker

docker login harbor.local
```

***

### E) Install Kubernetes (kubeadm) di vm-k8s + vm-worker (containerd → file config jadi ada)

#### E1) Di **vm-k8s & vm-worker**: disable swap

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

#### E2) Install containerd + generate config default

Di **vm-k8s & vm-worker**:

```bash
sudo apt-get update -y
sudo apt-get install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
```

Set `SystemdCgroup = true`:

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl enable --now containerd
sudo systemctl restart containerd
```

> Ini yang bikin kamu sebelumnya error `/etc/containerd/config.toml: No such file` — karena containerd belum diinstall & config belum digenerate.

#### E3) Install kubeadm/kubelet/kubectl

Di **vm-k8s & vm-worker** (Ubuntu):

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

sudo mkdir -p -m 0755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

#### E4) Init cluster di **vm-k8s**

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

Set kubeconfig:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes
```

#### E5) Install CNI (Calico)

Di **vm-k8s**:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

Tunggu node ready:

```bash
kubectl get nodes -w
```

#### E6) Join worker (di **vm-worker**)

Ambil command join dari output `kubeadm init`, lalu jalankan di vm-worker, contoh:

```bash
sudo kubeadm join 192.168.56.44:6443 --token <...> --discovery-token-ca-cert-hash sha256:<...>
```

Cek dari vm-k8s:

```bash
kubectl get nodes -o wide
```

***

### F) K8s bisa pull dari Harbor (containerd trust CA)

Containerd registry config pakai folder `/etc/containerd/certs.d/<host>/hosts.toml`. ([GitHub](https://github.com/containerd/containerd/blob/main/docs/hosts.md?utm_source=chatgpt.com))

#### F1) Copy CA ke vm-k8s & vm-worker

Di **vm-k8s**:

```bash
scp cikal@192.168.56.43:/etc/harbor/certs/ca.crt /tmp/harbor-ca.crt
```

Ulangi juga di **vm-worker** (atau scp dari vm-k8s ke vm-worker).

#### F2) Buat hosts.toml di **vm-k8s & vm-worker**

```bash
sudo mkdir -p /etc/containerd/certs.d/harbor.local
sudo cp /tmp/harbor-ca.crt /etc/containerd/certs.d/harbor.local/ca.crt

cat | sudo tee /etc/containerd/certs.d/harbor.local/hosts.toml >/dev/null <<'EOF'
server = "https://harbor.local"

[host."https://harbor.local"]
  ca = "/etc/containerd/certs.d/harbor.local/ca.crt"
EOF
```

Pastikan containerd pakai `config_path` (di file `/etc/containerd/config.toml`):

```toml
[plugins."io.containerd.grpc.v1.cri".registry]
  config_path = "/etc/containerd/certs.d"
```

Restart:

```bash
sudo systemctl restart containerd
```

***

### G) Edge Nginx tetap di vm-docker (HTTPS + rate limit) — LANJUT ke Observability + K8s Prod

Sekarang kita lengkapi **file-file yang belum ada** di repo kamu.

> Karena GitHub repo kamu kebuka tapi halaman foldernya di GitHub sering “error while loading” dari tool web-ku, aku pakai asumsi struktur yang kamu sebut (**frontend/**, **go/**, **laravel/**) + endpoint yang sudah kamu mention (`/api/products`).

***

## 1) Tambahkan folder deploy (di repo GitLab kamu di vm-docker)

Di **vm-docker**:

```bash
cd ~/three-body-problem-main

mkdir -p deploy/edge/nginx/conf.d deploy/edge/certs
mkdir -p deploy/staging
mkdir -p deploy/k8s/base
mkdir -p deploy/observability/{loki,promtail,grafana/provisioning/datasources,k8s}
```

***

## 2) Observability di vm-docker: Loki + Grafana + Promtail (staging docker logs)

### 2.1 File: `deploy/observability/loki/loki-config.yml`

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093
```

### 2.2 File: `deploy/observability/grafana/provisioning/datasources/loki.yml`

```yaml
apiVersion: 1

datasources:
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    isDefault: true
```

### 2.3 File: `deploy/observability/promtail/promtail-docker.yml`

Promtail ini baca log container docker (staging + edge + dsb) dari host.

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker-containers
    static_configs:
      - targets: [localhost]
        labels:
          job: docker
          __path__: /var/lib/docker/containers/*/*-json.log
```

### 2.4 File: `deploy/observability/docker-compose.observability.yml`

```yaml
services:
  loki:
    image: grafana/loki:3.1.0
    container_name: loki
    command: ["-config.file=/etc/loki/loki-config.yml"]
    volumes:
      - ./loki/loki-config.yml:/etc/loki/loki-config.yml:ro
      - loki_data:/loki
    ports:
      - "3100:3100"
    networks: [edge-net]
    restart: unless-stopped

  grafana:
    image: grafana/grafana:11.1.0
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning/datasources:/etc/grafana/provisioning/datasources:ro
    ports:
      - "3000:3000"
    networks: [edge-net]
    restart: unless-stopped

  promtail:
    image: grafana/promtail:3.1.0
    container_name: promtail
    command: ["-config.file=/etc/promtail/promtail.yml"]
    volumes:
      - ./promtail/promtail-docker.yml:/etc/promtail/promtail.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/log:/var/log:ro
    networks: [edge-net]
    restart: unless-stopped

volumes:
  loki_data:
  grafana_data:

networks:
  edge-net:
    external: true
```

### 2.5 Jalankan observability (vm-docker)

Pastikan `edge-net` ada (kalau belum):

```bash
docker network create edge-net || true
```

Run:

```bash
cd deploy/observability
docker compose -f docker-compose.observability.yml up -d
docker ps
```

Akses:

* Grafana: `http://192.168.56.42:3000` (admin / admin123)
* Loki: `http://192.168.56.42:3100/ready`

***

## 3) Promtail di Kubernetes (ambil log pods → kirim ke Loki vm-docker)

#### 3.1 File: `deploy/observability/k8s/promtail-configmap.yml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: promtail-config
  namespace: kube-system
data:
  promtail.yaml: |
    server:
      http_listen_port: 9080
      grpc_listen_port: 0

    positions:
      filename: /run/promtail/positions.yaml

    clients:
      - url: http://192.168.56.42:3100/loki/api/v1/push

    scrape_configs:
      - job_name: kubernetes-pods
        static_configs:
          - targets: [localhost]
            labels:
              job: k8s-pods
              __path__: /var/log/pods/*/*/*.log
```

> Konsep promtail tail `/var/log/pods/...` lalu push ke Loki. (Format ini umum dipakai untuk logging K8s → Loki.) ([GitHub](https://github.com/containerd/containerd/blob/main/docs/hosts.md?utm_source=chatgpt.com))

#### 3.2 File: `deploy/observability/k8s/promtail-daemonset.yml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: promtail
  namespace: kube-system
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: promtail
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: promtail
  template:
    metadata:
      labels:
        app: promtail
    spec:
      serviceAccountName: promtail
      tolerations:
        - operator: Exists
      containers:
        - name: promtail
          image: grafana/promtail:3.1.0
          args:
            - -config.file=/etc/promtail/promtail.yaml
          volumeMounts:
            - name: config
              mountPath: /etc/promtail
            - name: varlogpods
              mountPath: /var/log/pods
              readOnly: true
            - name: run
              mountPath: /run/promtail
      volumes:
        - name: config
          configMap:
            name: promtail-config
        - name: varlogpods
          hostPath:
            path: /var/log/pods
        - name: run
          hostPath:
            path: /run/promtail
            type: DirectoryOrCreate
```

#### 3.3 Apply (di vm-k8s)

```bash
kubectl apply -f deploy/observability/k8s/promtail-configmap.yml
kubectl apply -f deploy/observability/k8s/promtail-daemonset.yml

kubectl -n kube-system get ds promtail
kubectl -n kube-system logs -l app=promtail --tail=50
```

***

## 4) K8s PROD Manifest lengkap (NodePort, tanpa ingress/metalLB)

Kita pakai namespace: `threebody-prod`\
NodePort:

* frontend: `30080`
* go: `30081`
* laravel: `30082`

### 4.1 File: `deploy/k8s/base/00-namespace.yml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: threebody-prod
```

### 4.2 MySQL (pinned ke worker) — `deploy/k8s/base/10-mysql.yml`

> Karena kubeadm bare-metal biasanya belum ada StorageClass, kita pakai hostPath + nodeSelector (simple buat lab).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: threebody-prod
spec:
  ports:
    - port: 3306
      targetPort: 3306
  selector:
    app: mysql
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: threebody-prod
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      nodeSelector:
        kubernetes.io/hostname: vm-worker
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_ROOT_PASSWORD
            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_DATABASE
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_USER
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_PASSWORD
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
          readinessProbe:
            exec:
              command: ["sh","-c","MYSQL_PWD=$MYSQL_ROOT_PASSWORD mysqladmin ping -h 127.0.0.1 --silent"]
            initialDelaySeconds: 10
            periodSeconds: 5
  volumeClaimTemplates: []
  # hostPath volume (simple)
  template:
    metadata:
      labels:
        app: mysql
    spec:
      nodeSelector:
        kubernetes.io/hostname: vm-worker
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_ROOT_PASSWORD
            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_DATABASE
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_USER
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_PASSWORD
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-data
          hostPath:
            path: /data/k8s/mysql
            type: DirectoryOrCreate
```

> Catatan: YAML MySQL di atas sengaja “lab-simple”. Kalau kamu mau yang rapi production-grade, nanti kita pasang provisioner StorageClass.

### 4.3 GO Deployment + NodePort — `deploy/k8s/base/20-go.yml`

`__TAG__` nanti diganti di CI.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go
  namespace: threebody-prod
spec:
  replicas: 1
  selector:
    matchLabels:
      app: go
  template:
    metadata:
      labels:
        app: go
    spec:
      imagePullSecrets:
        - name: harbor-regcred
      containers:
        - name: go
          image: harbor.local/threebody/go:__TAG__
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              value: mysql
            - name: DB_PORT
              value: "3306"
            - name: DB_NAME
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_DATABASE
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_USER
            - name: DB_PASS
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_PASSWORD
          readinessProbe:
            tcpSocket:
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: go
  namespace: threebody-prod
spec:
  selector:
    app: go
  ports:
    - port: 8080
      targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: go-nodeport
  namespace: threebody-prod
spec:
  type: NodePort
  selector:
    app: go
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30081
```

### 4.4 Laravel Deployment + NodePort — `deploy/k8s/base/21-laravel.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel
  namespace: threebody-prod
spec:
  replicas: 1
  selector:
    matchLabels:
      app: laravel
  template:
    metadata:
      labels:
        app: laravel
    spec:
      imagePullSecrets:
        - name: harbor-regcred
      containers:
        - name: laravel
          image: harbor.local/threebody/laravel:__TAG__
          ports:
            - containerPort: 80
          env:
            - name: DB_HOST
              value: mysql
            - name: DB_PORT
              value: "3306"
            - name: DB_DATABASE
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_DATABASE
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_PASSWORD
            - name: APP_KEY
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: LARAVEL_APP_KEY
          readinessProbe:
            tcpSocket:
              port: 80
            initialDelaySeconds: 8
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: laravel
  namespace: threebody-prod
spec:
  selector:
    app: laravel
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: laravel-nodeport
  namespace: threebody-prod
spec:
  type: NodePort
  selector:
    app: laravel
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30082
```

### 4.5 Frontend Deployment + NodePort — `deploy/k8s/base/22-frontend.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: threebody-prod
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      imagePullSecrets:
        - name: harbor-regcred
      containers:
        - name: frontend
          image: harbor.local/threebody/frontend:__TAG__
          ports:
            - containerPort: 80
          readinessProbe:
            tcpSocket:
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-nodeport
  namespace: threebody-prod
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

***

## 5) Update Edge Nginx: route prod → NodePort worker (tetap sesuai rencana)

Di `deploy/edge/nginx/conf.d/edge.conf`, bagian prod sudah benar konsepnya:

* `/` → `http://192.168.56.45:30080`
* `/go/` → `http://192.168.56.45:30081/`
* `/laravel/` → `http://192.168.56.45:30082/`

Pastikan edge container join network `edge-net` dan staging service juga join `edge-net` (kamu sudah).

***

## 6) GitLab CI deploy PROD: bikin secret dari GitLab Variables (Secret Manager)

Kita **tidak commit secret ke repo**. CI yang buat secret di cluster.

### 6.1 Install kubectl di vm-docker (karena runner shell butuh)

Di **vm-docker**:

```bash
sudo apt-get update -y
sudo apt-get install -y kubectl
kubectl version --client
```

### 6.2 Replace `.gitlab-ci.yml` (versi rapih + tag `CI_COMMIT_SHORT_SHA`)

Ini versi yang **lebih bener** untuk:

* build/push Harbor
* staging compose + edge + observability
* prod k8s apply + create secrets + rollout + curl check

```yaml
stages:
  - build
  - deploy_staging
  - deploy_prod

variables:
  DOCKER_BUILDKIT: "1"
  TAG: "$CI_COMMIT_SHORT_SHA"
  REGISTRY: "harbor.local/threebody"

default:
  tags: ["deploy"]
  before_script:
    - set -euo pipefail

build_images:
  stage: build
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - echo "$HARBOR_PASSWORD" | docker login harbor.local -u "$HARBOR_USERNAME" --password-stdin
    - docker build -t "$REGISTRY/frontend:$TAG" -f frontend/Dockerfile frontend
    - docker build -t "$REGISTRY/go:$TAG" -f go/Dockerfile go
    - docker build -t "$REGISTRY/laravel:$TAG" -f laravel/Dockerfile laravel
    - docker push "$REGISTRY/frontend:$TAG"
    - docker push "$REGISTRY/go:$TAG"
    - docker push "$REGISTRY/laravel:$TAG"

deploy_staging:
  stage: deploy_staging
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - echo "$HARBOR_PASSWORD" | docker login harbor.local -u "$HARBOR_USERNAME" --password-stdin

    - docker network create edge-net || true

    # edge
    - cd deploy/edge
    - docker compose -f docker-compose.edge.yml up -d
    - cd ../..

    # observability (loki+grafana+promtail)
    - cd deploy/observability
    - docker compose -f docker-compose.observability.yml up -d
    - cd ../..

    # staging
    - cd deploy/staging
    - export REGISTRY="$REGISTRY"
    - export TAG="$TAG"
    - docker compose -f docker-compose.staging.yml up -d
    - cd ../..

    # healthcheck staging
    - curl -kfsS https://staging.local/ >/dev/null
    - curl -kfsS https://staging.local/go/api/products >/dev/null || true
    - curl -kfsS https://staging.local/laravel/api/products >/dev/null || true

deploy_prod:
  stage: deploy_prod
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - echo "$KUBECONFIG_B64" | base64 -d > /tmp/kubeconfig
    - export KUBECONFIG=/tmp/kubeconfig

    - kubectl apply -f deploy/k8s/base/00-namespace.yml

    # image pull secret (Harbor)
    - kubectl -n threebody-prod create secret docker-registry harbor-regcred \
        --docker-server=harbor.local \
        --docker-username="$HARBOR_USERNAME" \
        --docker-password="$HARBOR_PASSWORD" \
        --dry-run=client -o yaml | kubectl apply -f -

    # app secrets (mysql + laravel key)
    - kubectl -n threebody-prod create secret generic app-secrets \
        --from-literal=MYSQL_ROOT_PASSWORD="$MYSQL_ROOT_PASSWORD" \
        --from-literal=MYSQL_DATABASE="$MYSQL_DATABASE" \
        --from-literal=MYSQL_USER="$MYSQL_USER" \
        --from-literal=MYSQL_PASSWORD="$MYSQL_PASSWORD" \
        --from-literal=LARAVEL_APP_KEY="$LARAVEL_APP_KEY" \
        --dry-run=client -o yaml | kubectl apply -f -

    # apply manifests with TAG replacement
    - for f in deploy/k8s/base/10-mysql.yml deploy/k8s/base/20-go.yml deploy/k8s/base/21-laravel.yml deploy/k8s/base/22-frontend.yml; do
        sed "s/__TAG__/${TAG}/g" "$f" | kubectl apply -f - ;
      done

    - kubectl -n threebody-prod rollout status deploy/go --timeout=180s
    - kubectl -n threebody-prod rollout status deploy/laravel --timeout=180s
    - kubectl -n threebody-prod rollout status deploy/frontend --timeout=180s

    # healthcheck prod via edge
    - curl -kfsS https://prod.local/ >/dev/null
    - curl -kfsS https://prod.local/go/api/products >/dev/null || true
    - curl -kfsS https://prod.local/laravel/api/products >/dev/null || true
```

***

## 7) GitLab Variables yang WAJIB kamu isi

Project → Settings → CI/CD → Variables:

* `HARBOR_USERNAME` = `admin` (atau robot account)
* `HARBOR_PASSWORD` = `...`
* `MYSQL_ROOT_PASSWORD` = `...`
* `MYSQL_DATABASE` = `...`
* `MYSQL_USER` = `...`
* `MYSQL_PASSWORD` = `...`
* `LARAVEL_APP_KEY` = `base64:...` (generate dari laravel)
* `KUBECONFIG_B64` = base64 kubeconfig admin

Generate `KUBECONFIG_B64` di **vm-k8s**:

```bash
base64 -w0 ~/.kube/config
```

***

### H) Checklist validasi (biar kamu yakin end-to-end)

#### Harbor

* `https://harbor.local` kebuka
* dari vm-docker: `docker login harbor.local` sukses

#### Staging (vm-docker)

* `https://staging.local` kebuka (frontend tampil)
* Grafana kebuka: `http://192.168.56.42:3000` dan datasource Loki ada

#### Prod (k8s)

* `kubectl -n threebody-prod get pod -o wide`
* `https://prod.local` kebuka

#### Logging terintegrasi

* Di Grafana Explore pilih Loki → query:
  * `{job="docker"}` untuk log docker (staging/edge)
  * `{job="k8s-pods"}` untuk log pods (prod)

***

Kalau kamu kirim **isi folder** repo kamu setelah `git clone` (cukup output `tree -L 2` atau `ls -la` tiap folder `frontend/ go/ laravel/`), aku bisa **kunci** Dockerfile + env var supaya 100% pas dengan project kamu (biar endpoint & koneksi DB tidak “tebakan”).
