# RUNBOOK V4 (3 VM)

Siap. Aku bikinkan **RUNBOOK FINAL 3 VM** yang **anti-error push pertama** (menghindari kendala yang kamu alami: `dpkg lock`, DNS GitLab, docker permission runner, `6443 refused`, worker `NotReady`, containerd registry, dan edge nginx “host not found”). Sekalian aku rapihkan **nama VM + IP** sesuai yang kamu mau:

* **vm-docker** → `192.168.56.42` (**Harbor HTTP :8080** + **Edge hit.local :443** + **GitLab Runner**)
* **vm-k8s** → `192.168.56.43` (control-plane)
* **vm-worker** → `192.168.56.44` (worker + NodePort `30080/30081/30082`)

Aku juga sudah **patch project kamu** biar file K8s YAML-nya valid (di ZIP kamu ada placeholder `...` yang bikin apply gagal).\
➡️ **Download project yang sudah dipatch:** [three-body-problem-main-main-patched.zip](https://sandbox/mnt/data/three-body-problem-main-main-patched.zip)

***

## RUNBOOK V4 (3 VM) — FULL COPY/PASTE (anti error push pertama)

### 0) Rekomendasi resource (biar vm-k8s nggak “restart random”)

Karena laptop host kamu 16GB:

* **vm-docker (Harbor+Runner+Edge)**: **3–4GB RAM**, **2 core** (kalau 1 core boleh, tapi build/push lebih lama)
* **vm-k8s**: **4–6GB RAM**, **2 core**
* **vm-worker**: **3–4GB RAM**, **2 core**

> `kubeadm join` **tidak seharusnya** bikin control-plane restart. Kalau itu terjadi, hampir selalu karena **host kehabisan RAM/CPU**, hypervisor “reset”, atau VM crash. Jadi kunci “anti restart”: RAM cukup + disable auto-upgrades + jangan terlalu ketat core.

***

### 1) HARDEN SEMUA VM (wajib) — cegah dpkg lock + restart service random

Jalankan ini di **vm-docker, vm-k8s, vm-worker**:

```bash
sudo systemctl stop unattended-upgrades 2>/dev/null || true
sudo systemctl disable --now unattended-upgrades 2>/dev/null || true

sudo systemctl stop apt-daily.service apt-daily-upgrade.service 2>/dev/null || true
sudo systemctl disable --now apt-daily.timer apt-daily-upgrade.timer 2>/dev/null || true

sudo dpkg --configure -a || true

# needrestart jangan auto-restart service
if [ -f /etc/needrestart/needrestart.conf ]; then
  sudo cp -a /etc/needrestart/needrestart.conf /etc/needrestart/needrestart.conf.bak.$(date +%F-%H%M%S)
  sudo sed -i "s/^\s*\$nrconf{restart}.*/\$nrconf{restart} = 'l';/" /etc/needrestart/needrestart.conf || true
fi
```

#### ADD-ON: kalau apt ngunci (dpkg lock)

```bash
sudo systemctl stop unattended-upgrades apt-daily.service apt-daily-upgrade.service 2>/dev/null || true
sudo rm -f /var/lib/dpkg/lock-frontend /var/lib/dpkg/lock /var/cache/apt/archives/lock
sudo dpkg --configure -a || true
sudo apt-get -f install -y || true
sudo apt-get update -y
```

***

### 2) Set hostname + `/etc/hosts` (wajib, biar harbor.local & hit.local resolve stabil)

#### 2.1 Set hostname (jalankan sesuai VM)

**vm-docker**

```bash
sudo hostnamectl set-hostname vm-docker
```

**vm-k8s**

```bash
sudo hostnamectl set-hostname vm-k8s
```

**vm-worker**

```bash
sudo hostnamectl set-hostname vm-worker
```

#### 2.2 `/etc/hosts` (jalankan di SEMUA VM)

```bash
sudo tee /etc/hosts >/dev/null <<'EOF'
127.0.0.1 localhost
192.168.56.42 vm-docker harbor.local hit.local
192.168.56.43 vm-k8s
192.168.56.44 vm-worker
EOF

getent hosts harbor.local hit.local vm-k8s vm-worker
hostname
```

***

## VM-1: vm-docker (192.168.56.42)

**Fungsi:** Harbor HTTP `harbor.local:8080` + Edge `hit.local` + GitLab Runner (shell)

### A) Fix DNS (biar GitLab nggak random gagal resolve)

```bash
sudo tee /etc/systemd/resolved.conf >/dev/null <<'EOF'
[Resolve]
DNS=1.1.1.1 8.8.8.8
FallbackDNS=9.9.9.9 8.8.4.4
DNSStubListener=yes
EOF

sudo systemctl restart systemd-resolved
sudo resolvectl flush-caches

# pakai resolv.conf real (bukan 127.0.0.53)
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf

getent hosts gitlab.com altssh.gitlab.com || true
```

### B) Install Docker + tools

```bash
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl git nano openssh-server ufw unzip rsync openssl
sudo systemctl enable --now ssh

curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"
newgrp docker

docker version
docker compose version
```

### C) Docker: allow **insecure registry** Harbor (HTTP) `harbor.local:8080`

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json >/dev/null <<'EOF'
{
  "insecure-registries": ["harbor.local:8080"]
}
EOF

sudo systemctl restart docker
docker info | egrep -i "insecure registr|registry" || true
```

***

### D) Install Harbor (HTTP port 8080, TANPA TLS) — sesuai yang kamu minta

> Harbor & Edge sama-sama di vm-docker, jadi Harbor kita set **8080** supaya nggak konflik dengan port **80/443** milik edge.

```bash
export HARBOR_VERSION="v2.14.1"

cd /tmp
wget -O "harbor-offline-installer-${HARBOR_VERSION}.tgz" \
  "https://github.com/goharbor/harbor/releases/download/${HARBOR_VERSION}/harbor-offline-installer-${HARBOR_VERSION}.tgz"

tar -xzf "harbor-offline-installer-${HARBOR_VERSION}.tgz"

sudo rm -rf /opt/harbor
sudo mv harbor /opt/harbor
sudo chown -R "$USER:$USER" /opt/harbor
```

#### Edit `harbor.yml` (COPY/PASTE ini)

```bash
cd /opt/harbor
cp harbor.yml.tmpl harbor.yml
sudo nano harbor.yml
```

Isi minimal (sesuai arahanmu):

```yaml
hostname: harbor.local
http:
  port: 8080

# https:   <-- pastikan bagian ini DIHAPUS/DIKOMENTARI
#   port: 443
#   certificate: ...
#   private_key: ...

harbor_admin_password: Harbor12345
data_volume: /data/harbor
trivy:
  enabled: false
```

Install:

```bash
sudo mkdir -p /data/harbor
sudo ./prepare
sudo ./install.sh
sudo docker compose -f /opt/harbor/docker-compose.yml ps
```

#### Autostart Harbor (systemd)

```bash
sudo tee /etc/systemd/system/harbor.service >/dev/null <<'EOF'
[Unit]
Description=Harbor Registry (Docker Compose)
After=network-online.target docker.service
Wants=network-online.target docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/harbor
ExecStart=/usr/bin/docker compose -f /opt/harbor/docker-compose.yml up -d --remove-orphans
ExecStop=/usr/bin/docker compose -f /opt/harbor/docker-compose.yml stop
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now harbor
sudo systemctl status harbor --no-pager
```

#### Buat Project + Robot (wajib untuk pipeline)

Buka UI:

* `http://harbor.local:8080/`
* Create Project: `threebody`

***

### E) Edge Nginx (hit.local) — proxy ke NodePort langsung

> Kamu sudah kasih `docker-compose.edge.yml`. Yang penting: **edge.conf harus hit.local saja** dan NodePort menuju **vm-worker (192.168.56.44)**.

#### 1) Taruh edge config ke `/opt/threebody/edge`

```bash
sudo mkdir -p /opt/threebody
sudo rsync -a --delete ~/three-body-problem-main/ /opt/threebody/app/ 2>/dev/null || true

sudo mkdir -p /opt/threebody/edge
sudo rsync -a --delete /opt/threebody/app/deploy/edge/ /opt/threebody/edge/

cd /opt/threebody/edge
sudo ln -sf docker-compose.edge.yml docker-compose.yml
```

#### 2) Replace `nginx/conf.d/edge.conf` (COPY/PASTE)

```bash
sudo tee /opt/threebody/edge/nginx/conf.d/edge.conf >/dev/null <<'EOF'
# rate limiting zone
limit_req_zone $binary_remote_addr zone=api_rl:10m rate=5r/s;

server {
  listen 80;
  server_name hit.local;
  return 301 https://$host$request_uri;
}

# ---------- PROD (K8s NodePort) ----------
server {
  listen 443 ssl;
  server_name hit.local;

  ssl_certificate     /etc/nginx/certs/tls.crt;
  ssl_certificate_key /etc/nginx/certs/tls.key;

  # frontend
  location / {
    proxy_pass http://192.168.56.44:30080;
  }

  # go
  location /go/ {
    limit_req zone=api_rl burst=10 nodelay;
    proxy_pass http://192.168.56.44:30081/;
  }

  # laravel
  location /laravel/ {
    limit_req zone=api_rl burst=10 nodelay;
    proxy_pass http://192.168.56.44:30082/;
  }
}
EOF
```

#### 3) Buat TLS self-signed untuk `hit.local`

```bash
sudo mkdir -p /opt/threebody/edge/certs
cd /opt/threebody/edge/certs

sudo openssl req -x509 -nodes -newkey rsa:2048 -days 3650 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=hit.local"
```

#### 4) Jalankan edge

```bash
cd /opt/threebody/edge
sudo docker compose up -d
sudo docker exec -it edge-nginx nginx -t
```

***

### F) GitLab Runner (shell) + fix permission Docker (anti error push stage)

```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install -y gitlab-runner
sudo systemctl enable --now gitlab-runner

getent group docker || sudo groupadd docker
sudo usermod -aG docker gitlab-runner

sudo chown root:docker /var/run/docker.sock
sudo chmod 660 /var/run/docker.sock

sudo systemctl restart docker
sudo systemctl restart gitlab-runner

sudo -u gitlab-runner docker ps
```

#### Register runner (sekali, interaktif)

```bash
sudo gitlab-runner register
```

Isi yang aman:

* GitLab instance URL: `https://gitlab.com/` (atau URL GitLab kamu)
* Token: (ambil dari Project → Settings → CI/CD → Runners)
* Executor: `shell`
* Tags: `deploy`
* “Run untagged jobs?” → `false` (biar job kamu only tag deploy)
* “Lock to current project?” → `true`

***

### G) SSH GitLab via port 443 (anti masalah jaringan port 22)

Di vm-docker (atau mesin yang kamu pakai buat push):

```bash
ssh-keygen -t ed25519 -C "vm-docker-gitlab" -f ~/.ssh/id_ed25519 -N ""
mkdir -p ~/.ssh

cat <<'EOF' > ~/.ssh/config
Host gitlab-443
  HostName altssh.gitlab.com
  User git
  Port 443
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
  StrictHostKeyChecking accept-new
EOF

chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519 ~/.ssh/config
```

> Copy `~/.ssh/id_ed25519.pub` ke GitLab → SSH Keys.

Tes koneksi :

```bash
ssh -T git@gitlab-443
```

***

## VM-2: vm-k8s (192.168.56.43) — control-plane

### A) Base + swap off + sysctl + containerd

```bash
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl nano openssh-server apt-transport-https gpg
sudo systemctl enable --now ssh

sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

cat <<'EOF' | sudo tee /etc/modules-load.d/k8s.conf >/dev/null
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<'EOF' | sudo tee /etc/sysctl.d/99-kubernetes.conf >/dev/null
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

sudo apt-get install -y containerd
```

### B) Reset containerd config (SystemdCgroup + config\_path) — wajib

```bash
sudo systemctl stop containerd || true
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null

sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo awk '
/^\[plugins\."io\.containerd\.grpc\.v1\.cri"\.registry\]$/ && !done {
  print
  print "  config_path = \"/etc/containerd/certs.d\""
  done=1
  next
}
{print}
' /etc/containerd/config.toml | sudo tee /etc/containerd/config.toml.new >/dev/null
sudo mv /etc/containerd/config.toml.new /etc/containerd/config.toml

sudo systemctl enable --now containerd
sudo systemctl restart containerd
sudo systemctl is-active containerd
```

### C) Containerd: allow Harbor HTTP `harbor.local:8080` (tanpa TLS)

```bash
sudo mkdir -p /etc/containerd/certs.d/harbor.local:8080

sudo tee /etc/containerd/certs.d/harbor.local:8080/hosts.toml >/dev/null <<'EOF'
server = "http://harbor.local:8080"

[host."http://harbor.local:8080"]
  capabilities = ["pull", "resolve", "push"]
EOF

sudo systemctl restart containerd
```

### D) Install kubelet/kubeadm/kubectl v1.30

```bash
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

### E) kubeadm init + Calico

```bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.56.43 \
  --apiserver-cert-extra-sans=192.168.56.43,vm-k8s \
  --pod-network-cidr=192.168.0.0/16

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
kubectl get nodes -o wide
```

Ambil join command:

```bash
kubeadm token create --print-join-command --ttl 24h
```

```bash
watch -n 2 -d 'kubectl get nodes'
```

***

## VM-3: vm-worker (192.168.56.44) — worker node

### A) Base + swap off + sysctl + containerd (SAMA seperti vm-k8s)

Copy/paste persis langkah vm-k8s bagian:

* **A) Base + swap + sysctl + containerd**
* **B) Reset containerd (SystemdCgroup + config\_path)**
* **C) allow Harbor HTTP harbor.local:8080**
* **D) install kubelet/kubeadm/kubectl**

### B) Join cluster (pakai command dari vm-k8s)

```bash
sudo kubeadm join 192.168.56.43:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

### C) HostPath untuk MySQL (wajib, karena PV hostPath diarahkan ke sini)

```bash
sudo mkdir -p /data/threebody/mysql
sudo chown -R 999:999 /data/threebody/mysql || true
sudo chmod 700 /data/threebody/mysql || true
```

***

## 3) PATCH PROJECT (WAJIB agar deploy pertama tidak gagal)

Di ZIP project kamu, file K8s YAML ada placeholder `...` → itu bikin `kubectl apply` gagal.

✅ Paling gampang: pakai ZIP yang sudah aku patch:

* [three-body-problem-main-main-patched.zip](https://sandbox/mnt/data/three-body-problem-main-main-patched.zip)

Kalau kamu mau manual copy/paste, minimal replace file ini di repo:

* `.gitlab-ci.yml` (sudah pakai **build/push/deploy**, Harbor `harbor.local:8080`, healthcheck `hit.local`)
* `deploy/edge/nginx/conf.d/edge.conf` (hanya `hit.local`, proxy ke `192.168.56.44:30080/30081/30082`)
* `deploy/k8s/base/{10,20,30,40}-*.yaml` (valid & lengkap)

> Semua itu sudah ada di ZIP patched.

***

## 4) GitLab Variables (ini kunci “anti error push pertama”)

Project → Settings → CI/CD → Variables:

**Wajib:**

* `HARBOR_USERNAME` = admin
* `HARBOR_PASSWORD` = Harbor12345
* `MYSQL_ROOT_PASSWORD` = Harbor12345&#x20;
* `MYSQL_DATABASE` = threebody
* `MYSQL_USER` = admin
* `MYSQL_PASSWORD` = Harbor12345
* `LARAVEL_APP_KEY` (format `base64:...`)
* `KUBECONFIG_PROD` **Type: File** (isi dari `~/.kube/config` di vm-k8s)

#### Cara ambil kubeconfig dari vm-k8s → untuk variable File

Dari vm-docker (atau laptop kamu):

```bash
scp cikal@192.168.56.43:~/.kube/config ./kubeconfig-prod
```

Lalu upload `kubeconfig-prod` ke variable `KUBECONFIG_PROD` (Type: File).

#### Sebelum melakukan push pertama wajib mengikuti dibawah ini :

Edit file .gitlab-ci.yml

```bash
nano ~/three-body-problem/.gitlab-ci.yml
```

Paste ke .gitlab-ci.yml

```bash
stages:
  - build
  - push
  - deploy

workflow:
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - when: never

variables:
  DOCKER_BUILDKIT: "1"
  TAG: "$CI_COMMIT_SHORT_SHA"

  # Harbor HTTP (insecure) di vm-docker
  # IMPORTANT: docker daemon runner WAJIB sudah allow insecure registry harbor.local:8080
  HARBOR_HOST: "harbor.local:8080"
  HARBOR_PROJECT: "threebody"
  REGISTRY: "$HARBOR_HOST/$HARBOR_PROJECT"

  K8S_NS: "threebody-prod"
  KUBECTL_VERSION: "v1.30.14"
  K8S_ROLLOUT_TIMEOUT: "15m"
  K8S_IMAGEPULL_SECRET: "harbor-regcred"

default:
  tags: ["deploy"]   # pastikan runner vm-docker punya tag 'deploy'
  before_script:
    - set -euo pipefail

build_check:
  stage: build
  script: |
    echo "==> [build] build-check images (tanpa push)"
    docker build \
      --build-arg REACT_APP_GO_API_BASE=/go \
      --build-arg REACT_APP_LARAVEL_API_BASE=/laravel \
      -t "$REGISTRY/frontend:$TAG" \
      -f frontend/Dockerfile frontend

    docker build -t "$REGISTRY/go:$TAG" -f go/Dockerfile go
    docker build -t "$REGISTRY/laravel:$TAG" -f laravel/Dockerfile laravel

push_images:
  stage: push
  needs: ["build_check"]
  script: |
    echo "==> [push] sanity check harbor registry"
    # 401 itu normal artinya /v2 hidup
    (curl -fsSI "http://harbor.local:8080/v2/" | head -n 1) || true

    echo "==> [push] login Harbor (HTTP insecure lab)"
    : "${HARBOR_USERNAME:?Missing HARBOR_USERNAME}"
    : "${HARBOR_PASSWORD:?Missing HARBOR_PASSWORD}"
    echo "$HARBOR_PASSWORD" | docker login "$HARBOR_HOST" -u "$HARBOR_USERNAME" --password-stdin

    echo "==> [push] build+push images (TAG=$TAG)"
    docker build \
      --build-arg REACT_APP_GO_API_BASE=/go \
      --build-arg REACT_APP_LARAVEL_API_BASE=/laravel \
      -t "$REGISTRY/frontend:$TAG" \
      -f frontend/Dockerfile frontend
    docker push "$REGISTRY/frontend:$TAG"

    docker build -t "$REGISTRY/go:$TAG" -f go/Dockerfile go
    docker push "$REGISTRY/go:$TAG"

    docker build -t "$REGISTRY/laravel:$TAG" -f laravel/Dockerfile laravel
    docker push "$REGISTRY/laravel:$TAG"

deploy:
  stage: deploy
  needs: ["push_images"]
  script: |
    echo "==> [deploy] validasi variables"
    : "${KUBECONFIG_PROD:?Missing KUBECONFIG_PROD (GitLab Variables Type: File)}"
    : "${HARBOR_USERNAME:?Missing HARBOR_USERNAME}"
    : "${HARBOR_PASSWORD:?Missing HARBOR_PASSWORD}"
    : "${MYSQL_ROOT_PASSWORD:?Missing MYSQL_ROOT_PASSWORD}"
    : "${MYSQL_DATABASE:?Missing MYSQL_DATABASE}"
    : "${MYSQL_USER:?Missing MYSQL_USER}"
    : "${MYSQL_PASSWORD:?Missing MYSQL_PASSWORD}"
    : "${LARAVEL_APP_KEY:?Missing LARAVEL_APP_KEY}"

    echo "==> [deploy] siapkan kubectl (tanpa sudo, aman untuk runner shell)"
    mkdir -p .bin
    if [ ! -x .bin/kubectl ]; then
      curl -fsSL -o .bin/kubectl "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl"
      chmod +x .bin/kubectl
    fi
    export PATH="$PWD/.bin:$PATH"
    kubectl version --client=true

    echo "==> [deploy] set kubeconfig dari File Variable"
    export KUBECONFIG="$KUBECONFIG_PROD"

    echo "==> [deploy] cek cluster"
    kubectl get nodes -o wide

    echo "==> [deploy] pastikan namespace"
    kubectl get ns "$K8S_NS" >/dev/null 2>&1 || kubectl create ns "$K8S_NS"

    echo "==> [deploy] create/update app-secrets"
    kubectl -n "$K8S_NS" create secret generic app-secrets \
      --from-literal=MYSQL_ROOT_PASSWORD="$MYSQL_ROOT_PASSWORD" \
      --from-literal=MYSQL_DATABASE="$MYSQL_DATABASE" \
      --from-literal=MYSQL_USER="$MYSQL_USER" \
      --from-literal=MYSQL_PASSWORD="$MYSQL_PASSWORD" \
      --from-literal=LARAVEL_APP_KEY="$LARAVEL_APP_KEY" \
      --dry-run=client -o yaml | kubectl apply -f -

    echo "==> [deploy] create/update imagePullSecret Harbor"
    kubectl -n "$K8S_NS" create secret docker-registry "$K8S_IMAGEPULL_SECRET" \
      --docker-server="$HARBOR_HOST" \
      --docker-username="$HARBOR_USERNAME" \
      --docker-password="$HARBOR_PASSWORD" \
      --dry-run=client -o yaml | kubectl apply -f -

    echo "==> [deploy] attach imagePullSecret ke default serviceaccount (FIX QUOTE)"
    kubectl -n "$K8S_NS" patch serviceaccount default \
      -p "{\"imagePullSecrets\": [{\"name\": \"$K8S_IMAGEPULL_SECRET\"}]}" >/dev/null || true

    echo "==> [deploy] apply manifests (NodePort + mysql + deployments)"
    kubectl apply -f deploy/k8s/base/

    echo "==> [deploy] update image tag"
    kubectl -n "$K8S_NS" set image deployment/frontend frontend="$REGISTRY/frontend:$TAG"
    kubectl -n "$K8S_NS" set image deployment/go go="$REGISTRY/go:$TAG"
    kubectl -n "$K8S_NS" set image deployment/laravel laravel="$REGISTRY/laravel:$TAG"

    echo "==> [deploy] rollout"
    kubectl -n "$K8S_NS" rollout status deployment/frontend --timeout="$K8S_ROLLOUT_TIMEOUT" || (
      echo "---- DEBUG (frontend rollout gagal) ----"
      kubectl -n "$K8S_NS" get pods -o wide || true
      kubectl -n "$K8S_NS" get events --sort-by=.lastTimestamp | tail -n 120 || true
      kubectl -n "$K8S_NS" describe deploy frontend || true
      kubectl -n "$K8S_NS" describe pod -l app=frontend || true
      exit 1
    )
    kubectl -n "$K8S_NS" rollout status deployment/go --timeout="$K8S_ROLLOUT_TIMEOUT"
    kubectl -n "$K8S_NS" rollout status deployment/laravel --timeout="$K8S_ROLLOUT_TIMEOUT"
    kubectl -n "$K8S_NS" rollout status statefulset/mysql --timeout="$K8S_ROLLOUT_TIMEOUT" || true

    echo "==> [deploy] ringkasan"
    kubectl -n "$K8S_NS" get deploy -o wide
    kubectl -n "$K8S_NS" get svc -o wide
    kubectl -n "$K8S_NS" get pods -o wide

    echo "==> [deploy] healthcheck via edge nginx (hit.local di vm-docker)"
    for i in $(seq 1 60); do
      if curl -kfsS --resolve hit.local:443:127.0.0.1 https://hit.local/ >/dev/null 2>&1; then
        echo "OK: hit.local sehat"
        exit 0
      fi
      sleep 2
    done

    echo "ERROR: hit.local healthcheck gagal"
    kubectl -n "$K8S_NS" get pods -o wide || true
    kubectl -n "$K8S_NS" describe pods || true
    exit 1

```

Edit file deploy/edge/nginx/conf.d/edge.conf

```bash
nano deploy/edge/nginx/conf.d/edge.conf
```

Paste ke deploy/edge/nginx/conf.d/edge.conf

```bash
# rate limiting zone
limit_req_zone $binary_remote_addr zone=api_rl:10m rate=5r/s;

server {
  listen 80;
  server_name hit.local;
  return 301 https://$host$request_uri;
}

server {
  listen 443 ssl;
  server_name hit.local;

  ssl_certificate     /etc/nginx/certs/tls.crt;
  ssl_certificate_key /etc/nginx/certs/tls.key;

  # frontend (NodePort)
  location / {
    proxy_pass http://192.168.56.44:30080;
  }

  # go (NodePort)
  location /go/ {
    limit_req zone=api_rl burst=10 nodelay;
    proxy_pass http://192.168.56.44:30081/;
  }

  # laravel (NodePort)
  location /laravel/ {
    limit_req zone=api_rl burst=10 nodelay;
    proxy_pass http://192.168.56.44:30082/;
  }
}
```

***

## 5) Alur PUSH pertama (biar pipeline langsung aman)

1. Pastikan runner sudah **online** dan tag **deploy**.
2. Pastikan Harbor project `threebody` dan robot account sudah dibuat.
3. Push ke GitLab (pakai altssh 443 kalau perlu):

## A) JALANKAN DI vm-docker (lokasi: `~/three-body-problem`)

### 1) Pastikan repo lokal jadi Git repo (INIT)

```bash
cd ~/three-body-problem

# cek file ada
ls -lah

# init git repo (karena belum ada .git)
git init

# set default branch git biar ke depan nggak muncul warning
git config --global init.defaultBranch main

# set identitas git biar commit tidak gagal
git config --global user.name "cikalfarid"
git config --global user.email "cikalfarid@users.noreply.gitlab.com"

# pastikan branch main
git branch -M main
```

## 3) Tes koneksi

```bash
ssh -T git@gitlab-443
```

## 3) Set remote ke repo kamu (pakai 443)

Repo kamu: `git@gitlab.com:cikalfarid/three-body-problem.git`

Untuk 443 jadi:\
`git@gitlab-443:cikalfarid/three-body-problem.git`

```bash
cd ~/three-body-problem
git remote remove origin 2>/dev/null || true
git remote add origin git@gitlab-443:cikalfarid/three-body-problem.git
git remote -v
```

***

## 4) Commit + push (overwrite remote biar “kosong lalu diisi dari lokal”)

Ini akan membuat isi branch `main` di GitLab **jadi sama persis** seperti folder lokal kamu.

```bash
cd ~/three-body-problem

git add -A
git commit -m "initial: runbook V4 + ci build/push/deploy" || true

git push -u origin main --force
```

***

## Kalau `--force` ditolak (Protected Branch)

Kalau muncul error “not allowed to force push / protected branch”, buka GitLab:\
Project → Settings → Repository → Protected Branches → `main`

* Unprotect dulu, atau izinkan Maintainer force push\
  Lalu ulang:

```bash
git push -u origin main --force
```

***

## 6) Setelah deploy pertama (sekali saja): migrate/seed

Di vm-k8s:

```bash
NS=threebody-prod
LARAVEL_POD=$(kubectl -n "$NS" get pod -l app=laravel -o jsonpath='{.items[0].metadata.name}')
kubectl -n "$NS" exec -it "$LARAVEL_POD" -- php artisan migrate --force
kubectl -n "$NS" exec -it "$LARAVEL_POD" -- php artisan db:seed --force || true
```

***

## 7) Cara akses

* Harbor: `http://harbor.local:8080/`
* Aplikasi: `https://hit.local/` (lewat edge nginx di vm-docker)

***

### Kalau kamu mau, kirimkan:

* **log pipeline pertama yang gagal** (kalau masih ada error)\
  biar aku kunci lagi bagian yang masih bocor (misal: runner tag salah, variable belum kebaca, atau imagePull secret mismatch).
