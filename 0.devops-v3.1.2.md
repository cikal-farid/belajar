# 0.DevOps V3.1.2

Aku sudah **cek isi ZIP yang kamu upload** (`three-body-problem-main-main-patched.zip`) supaya kita **nggak mengira-ngira**. Hasil ceknya:

### 1) Hasil pengecekan ZIP yang kamu upload (fakta dari file)

Yang paling penting:

1.  **`.gitlab-ci.yml` di ZIP kamu ada bug quoting** di bagian patch serviceaccount:\
    Di file ZIP, barisnya seperti ini (ini raw dari file, dan ini bisa bikin command rusak di shell):

    ```bash
    kubectl -n "$K8S_NS" patch serviceaccount default \
      -p "{"imagePullSecrets": [{"name": "$K8S_IMAGEPULL_SECRET"}]}" >/dev/null || true
    ```

    Ini harusnya pakai quote yang aman (escaped JSON) seperti yang kamu tulis belakangan.
2. Di ZIP kamu ada folder **`.bin/kubectl` ukuran \~50MB**.\
   Ini **bahaya kalau ikut ke-commit** (repo jadi bengkak, push lama, bisa bikin pipeline/clone berat). Di ZIP kamu tidak ada `.gitignore` yang mengabaikan `.bin/`.
3. Di ZIP kamu ada **`deploy/edge/certs/tls.key` + `tls.crt`** (private key!).\
   Ini **sebaiknya jangan pernah ikut commit**. Sertifikat edge seharusnya **digenerate di vm-docker**, bukan disimpan di repo.
4. Manifest K8s di ZIP kamu masih pakai `:latest` (benar), contohnya:
   * `harbor.local:8080/threebody/frontend:latest`
   * `harbor.local:8080/threebody/go:latest`
   * `harbor.local:8080/threebody/laravel:latest`
5. `deploy/edge/nginx/conf.d/edge.conf` di ZIP kamu **sudah benar**: proxy `hit.local` ke NodePort `30080/30081/30082` di IP worker.

***

### 2) Aku sudah bikin ZIP “FINAL” yang sudah diperbaiki (biar start dari 3 VM kosong aman)

Aku buat **full perbaikan** berdasarkan masalah-masalah yang kamu alami (DNS, dpkg lock, containerd registry, imagePullBackOff, nodeport, dan bug CI):

✅ Perubahan di **project FINAL**:

* Fix `.gitlab-ci.yml` (patch SA quoting aman)
* CI **push tag `$TAG` + juga push `:latest`** → supaya manifest `:latest` **tidak pernah “not found”** lagi
* Tambah `.gitignore` (mengabaikan `.bin/`, vendor, node\_modules, dll)
* Hapus `.bin/kubectl` dari project
* Hapus `deploy/edge/certs/tls.key/tls.crt` dari repo + `.gitignore` untuk certs
* Optimasi **Laravel Dockerfile** kecilkan sedikit (runtime tidak install git/unzip) + tambah `laravel/.dockerignore`

📦 **Download project finalnya:**\
[Download three-body-problem-final.zip](https://sandbox/mnt/data/three-body-problem-final.zip)

> Pakai ZIP ini untuk “mulai dari awal” biar runbook di bawah nyambung dan konsisten.

***

## RUNBOOK FINAL (3 VM kosong) — FULL REPLACE (copy/paste)

Target: **sekali push** → pipeline **build/push/deploy** → `https://hit.local/` jalan.

> Aku tulis runbook ini **parameter-based** supaya kamu tinggal ganti IP sekali saja kalau IP VM kamu beda.

***

### A. Tentukan IP (PASTIKAN dari awal)

Pakai ini (sama seperti rancangan runbook kamu):

* **vm-docker** = `192.168.56.42`
* **vm-k8s** (control-plane) = `192.168.56.43`
* **vm-worker** = `192.168.56.44`

Port:

* Harbor: `http://harbor.local:8080`
* Edge: `https://hit.local`
* NodePort: `30080(frontend) / 30081(go) / 30082(laravel)`

Kalau IP kamu beda, ganti di:

* `/etc/hosts` semua VM
* `deploy/edge/nginx/conf.d/edge.conf` (proxy\_pass ke IP worker)
* kubeadm init/join (pakai IP control-plane)

***

### B. LANGKAH WAJIB DI SEMUA VM (vm-docker, vm-k8s, vm-worker)

#### 1) Matikan auto update biar tidak ada `dpkg lock` / restart random

```bash
sudo systemctl stop unattended-upgrades 2>/dev/null || true
sudo systemctl disable --now unattended-upgrades 2>/dev/null || true

sudo systemctl stop apt-daily.service apt-daily-upgrade.service 2>/dev/null || true
sudo systemctl disable --now apt-daily.timer apt-daily-upgrade.timer 2>/dev/null || true

sudo dpkg --configure -a || true
```

#### 2) DNS stabil (biar GitLab/registry tidak random gagal resolve)

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
```

#### 3) Set hostname + /etc/hosts (WAJIB agar harbor.local & hit.local resolve)

Jalankan sesuai VM:

```bash
# vm-docker
sudo hostnamectl set-hostname vm-docker
```

```bash
# vm-k8s
sudo hostnamectl set-hostname vm-k8s
```

```bash
# vm-worker
sudo hostnamectl set-hostname vm-worker
```

Di **SEMUA VM**:

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

#### 4) Firewall (biar NodePort tidak ketahan)

Untuk lab dan biar “anti pusing”, aku sarankan **disable UFW** dulu:

```bash
sudo ufw disable 2>/dev/null || true
```

Kalau kamu mau UFW tetap ON, minimal di **vm-worker**:

```bash
sudo ufw allow 30080:30082/tcp
sudo ufw allow 22/tcp
sudo ufw reload
sudo ufw status verbose
```

***

## C. VM-1: vm-docker (Harbor + Edge + GitLab Runner)

### 1) Install Docker + tools

```bash
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl git nano openssh-server unzip rsync openssl
sudo systemctl enable --now ssh

curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"
newgrp docker

docker version
docker compose version
```

### 2) Docker allow insecure registry (Harbor HTTP)

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json >/dev/null <<'EOF'
{
  "insecure-registries": ["harbor.local:8080"]
}
EOF
sudo systemctl restart docker

docker info | egrep -i "insecure|registry" || true
```

### 3) Install Harbor (HTTP :8080, tanpa TLS)

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

Edit config:

```bash
cd /opt/harbor
cp harbor.yml.tmpl harbor.yml
nano harbor.yml
```

Isi minimal:

```yaml
hostname: harbor.local
http:
  port: 8080

# pastikan https tidak dipakai
# https:
#   port: 443

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

Buka UI Harbor:

* `http://harbor.local:8080/`
* Login: `admin / Harbor12345`
* **Create Project**: `threebody` (Private boleh)

### 4) Jalankan Edge Nginx (hit.local) pakai container

#### a) Extract project FINAL

Upload/ambil file: `three-body-problem-final.zip` ke vm-docker, lalu:

```bash
cd ~
unzip -q three-body-problem-final.zip
mv three-body-problem-final three-body-problem
cd ~/three-body-problem
ls -lah
```

#### b) Generate cert hit.local (jangan commit key)

```bash
cd ~/three-body-problem
mkdir -p deploy/edge/certs
cd deploy/edge/certs

openssl req -x509 -nodes -newkey rsa:2048 -days 3650 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=hit.local"
```

#### c) Pastikan edge.conf proxy ke IP worker

Cek file:

```bash
grep -n "proxy_pass" ../nginx/conf.d/edge.conf
```

Jika tidak menemukan dengan perintah diatas, bisa gunakan perintah dibawah dari root repo lokal

```bash
grep -n "proxy_pass" deploy/edge/nginx/conf.d/edge.conf
```

Harus mengarah ke:

* `http://192.168.56.44:30080`
* `http://192.168.56.44:30081`
* `http://192.168.56.44:30082`

#### d) Run edge

```bash
cd ~/three-body-problem/deploy/edge
sudo docker compose -f docker-compose.edge.yml up -d
sudo docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Ports}}'

# test config Nginx DI DALAM CONTAINER (bukan nginx host)
sudo docker exec -it edge-nginx nginx -t
```

### 5) Install GitLab Runner (shell) + akses docker

```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install -y gitlab-runner
sudo systemctl enable --now gitlab-runner

sudo usermod -aG docker gitlab-runner
sudo systemctl restart docker
sudo systemctl restart gitlab-runner

sudo -u gitlab-runner docker ps
```

### 6) Register runner (sekali)

```bash
sudo gitlab-runner register
```

Rekomendasi isi:

* executor: `shell`
* tags: `deploy`
* run untagged: `false`

***

## D. VM-2: vm-k8s (control-plane)

### 1) Base + swap off + sysctl

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
```

### 2) Install containerd + set config registry path

```bash
sudo apt-get install -y containerd
sudo systemctl stop containerd || true

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null

# systemd cgroup wajib
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# registry config_path wajib
sudo sed -i 's|^\(\s*\)\[plugins\."io\.containerd\.grpc\.v1\.cri"\.registry\]\s*$|&\n\\1  config_path = "/etc/containerd/certs.d"|' /etc/containerd/config.toml

sudo systemctl enable --now containerd
sudo systemctl restart containerd
sudo systemctl is-active containerd
```

### 3) Allow Harbor HTTP (insecure) untuk containerd

```bash
sudo mkdir -p /etc/containerd/certs.d/harbor.local:8080

sudo tee /etc/containerd/certs.d/harbor.local:8080/hosts.toml >/dev/null <<'EOF'
server = "http://harbor.local:8080"

[host."http://harbor.local:8080"]
  capabilities = ["pull", "resolve", "push"]
EOF

sudo systemctl restart containerd
```

### 4) Install Kubernetes v1.30

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

### 5) kubeadm init + Calico

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

***

## E. VM-3: vm-worker (worker node)

Ulangi langkah **vm-k8s** untuk:

* swap off + sysctl
* install containerd + set `SystemdCgroup=true` + `config_path=/etc/containerd/certs.d`
* allow Harbor HTTP (`/etc/containerd/certs.d/harbor.local:8080/hosts.toml`)
* install kubelet/kubeadm/kubectl

Lalu join:

```bash
sudo kubeadm join 192.168.56.43:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

HostPath MySQL:

```bash
sudo mkdir -p /data/threebody/mysql
sudo chown -R 999:999 /data/threebody/mysql || true
sudo chmod 700 /data/threebody/mysql || true
```

Cek dari vm-k8s:

```bash
kubectl get nodes -o wide
```

***

## F. GitLab Variables (kunci “sekali push aman”)

Project → Settings → CI/CD → Variables:

Wajib:

* `HARBOR_USERNAME` = `admin`
* `HARBOR_PASSWORD` = `Harbor12345`
* `MYSQL_ROOT_PASSWORD` = (bebas)
* `MYSQL_DATABASE` = `threebody`
* `MYSQL_USER` = `admin`
* `MYSQL_PASSWORD` = (bebas)
* `LARAVEL_APP_KEY` = `base64:...` (punya Laravel kamu)

`KUBECONFIG_PROD` **Type: File**:\
Ambil dari vm-k8s:

```bash
# jalankan dari vm-docker
scp cikal@192.168.56.43:~/.kube/config ./kubeconfig-prod
```

Upload file itu ke variable `KUBECONFIG_PROD` (Type: File).

***

## G. Push pertama (dari vm-docker)

Masuk repo:

```bash
cd ~/three-body-problem
```

Init git + set remote:

```bash
git init
git branch -M main
git config --global user.name "cikalfarid"
git config --global user.email "cikalfarid@users.noreply.gitlab.com"

git remote remove origin 2>/dev/null || true
git remote add origin <REMOTE_GITLAB_KAMU>
git remote -v
```

Commit & push:

```bash
git add -A
git commit -m "init: final runbook + ci + deploy" || true
git push -u origin main
```

***

## H. Verifikasi setelah pipeline sukses

Di vm-k8s:

```bash
NS=threebody-prod
kubectl -n "$NS" get pods -o wide
kubectl -n "$NS" get svc -o wide
```

Di vm-docker:

```bash
# nodeport langsung
curl -fsSI "http://192.168.56.44:30080/" | head
curl -fsSI "http://192.168.56.44:30082/api/products" | head

# lewat edge (self-signed)
curl -vk --resolve hit.local:443:127.0.0.1 https://hit.local/ 2>&1 | tail -n 30
```

***

### Catatan soal “Laravel image besar”

Dari data kamu sebelumnya, laravel content size \~242MB.\
Di ZIP FINAL aku sudah bikin perubahan aman:

* runtime **tidak install git/unzip**
* tambah `.dockerignore` laravel\
  Biasanya ini **mengurangi beberapa MB–puluhan MB** (tergantung dependency).

Kalau kamu mau “turun jauh” (misalnya 80–150MB), biasanya harus pindah ke **php-fpm-alpine + nginx** (lebih kompleks). Kalau nanti kamu mau, bilang saja—nanti aku bikinkan versi “advanced shrink” yang tetap rapih untuk K8s.

***

Kalau kamu pakai ZIP FINAL + runbook di atas, kamu **tidak perlu lagi** trik sed ganti `:latest` jadi `${TAG}` di manifest (yang kemarin sempat bikin bingung). CI sekarang:

* push `$TAG`
* push juga `:latest`
* deploy pakai `$TAG` (jadi update selalu pasti ketarik)

Kalau kamu mau, setelah kamu mulai dari 3 VM kosong dan push pertama, kirim **1 output ini saja** kalau ada yang nyangkut:

* log job GitLab yang gagal (bagian error paling bawah ±50 baris)

Nanti aku kunci debug-nya pas di titik itu.
