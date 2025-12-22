# KuberMenetas

Di bawah ini runbook **VM1 (k8s-master | 192.168.56.42)** yang “jalan dari nol sampai cluster siap staging/prod”, **single-node** sesuai topologi kamu (VM2 = CI + Harbor HTTP 8081, VM1 = K8s, VM3 = monitoring).

> Catatan penting (biar nggak kejebak): karena host-only kamu pakai **192.168.56.0/24**, **jangan** pakai Pod CIDR `192.168.0.0/16` (overlap). Di runbook ini aku pakai **Pod CIDR Flannel `10.244.0.0/16`**. ([Stack Overflow](https://stackoverflow.com/questions/48984659/understanding-kubeadm-init-command-for-flannel?utm_source=chatgpt.com))

***

### 0) Prasyarat Network & Hostname (sekali saja)

#### 0.1 Set hostname + hosts

Di **VM1**:

```bash
sudo hostnamectl set-hostname k8s-master

sudo tee -a /etc/hosts >/dev/null <<'EOF'
192.168.56.42 k8s-master
192.168.56.43 devops-ci
192.168.56.44 monitoring
EOF
```

#### 0.2 Pastikan bisa saling ping via host-only

```bash
ping -c 2 192.168.56.43
ping -c 2 192.168.56.44
```

#### 0.3 (Opsional) Contoh netplan 2 NIC (kalau IP belum fix)

Umumnya VirtualBox: NAT=`enp0s3`, host-only=`enp0s8`.

```bash
ip a
sudo nano /etc/netplan/01-netcfg.yaml
```

Contoh:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses: [192.168.56.42/24]
```

Apply:

```bash
sudo netplan apply
```

***

### 1) Setup dasar OS (VM1)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl vim git ca-certificates gnupg lsb-release apt-transport-https
```

#### 1.1 Disable swap (wajib untuk kubeadm)

```bash
sudo swapoff -a
sudo sed -i.bak '/\sswap\s/s/^/#/' /etc/fstab
free -h | sed -n '1,2p'
```

***

### 2) Kernel module & sysctl (wajib)

```bash
cat <<'EOF' | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<'EOF' | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

Cek cepat:

```bash
sysctl net.ipv4.ip_forward
lsmod | egrep 'br_netfilter|overlay'
```

***

### 3) Install Container Runtime: containerd

```bash
sudo apt update
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
```

#### 3.1 Set cgroup driver = systemd (wajib stabil)

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

#### 3.2 Izinkan pull image dari Harbor **HTTP** (insecure registry)

Containerd mendukung konfigurasi registry via **hosts.toml** per host registry. ([GitHub](https://github.com/containerd/containerd/blob/main/docs/hosts.md?utm_source=chatgpt.com))

1. Pastikan containerd CRI baca directory ini:

```bash
sudo perl -0777 -i -pe 's/\[plugins\."io\.containerd\.grpc\.v1\.cri"\.registry\]\n(\s*)config_path = ".*?"/[plugins."io.containerd.grpc.v1.cri".registry]\n$1config_path = "\/etc\/containerd\/certs.d"/s' /etc/containerd/config.toml
```

2. Buat hosts.toml untuk Harbor:

```bash
sudo mkdir -p /etc/containerd/certs.d/192.168.56.43:8081

cat <<'EOF' | sudo tee /etc/containerd/certs.d/192.168.56.43:8081/hosts.toml
server = "http://192.168.56.43:8081"

[host."http://192.168.56.43:8081"]
  capabilities = ["pull", "resolve"]
EOF
```

Restart:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd --no-pager
```

***

### 4) Install kubeadm/kubelet/kubectl (VM1)

Ikuti repo APT baru di **pkgs.k8s.io** (panduan resmi “Installing kubeadm”). ([Kubernetes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/?utm_source=chatgpt.com))

> Di contoh ini aku pakai channel stable **v1.34** (sesuai dokumen install kubeadm saat ini). ([Kubernetes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/?utm_source=chatgpt.com))

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

***

### 5) Bootstrap cluster single-node (kubeadm init)

Karena host-only kamu **192.168.56.0/24**, kita pakai Pod CIDR **10.244.0.0/16** (Flannel default). ([Stack Overflow](https://stackoverflow.com/questions/48984659/understanding-kubeadm-init-command-for-flannel?utm_source=chatgpt.com))

```bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.56.42 \
  --pod-network-cidr=10.244.0.0/16 \
  --cri-socket=unix:///run/containerd/containerd.sock
```

#### 5.1 Setup kubeconfig untuk user kamu

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown "$(id -u)":"$(id -g)" $HOME/.kube/config

kubectl get nodes
```

***

### 6) Install CNI (Flannel)

Manifest Flannel ada di repo resmi flannel-io. ([GitHub](https://github.com/flannel-io/flannel/blob/master/Documentation/kubernetes.md?utm_source=chatgpt.com))

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

Tunggu node jadi Ready:

```bash
watch -n 2 kubectl get nodes
```

Kalau mau verifikasi lebih mantap, jalankan ini:

```bash
kubectl get pods -A
kubectl describe node k8s-master | sed -n '/Taints:/,/Conditions:/p'
kubectl -n kube-system get pods -o wide
```

Yang kamu harapkan:

* CoreDNS, kube-proxy, dsb **Running**
* Taints di node **kosong** atau tidak ada `NoSchedule` untuk control-plane.

Jalankan:

```bash
sudo grep -n "sandbox_image" -n /etc/containerd/config.toml || true
```

Kalau belum ada, tambahkan di bagian CRI (paling aman pakai command ini):

```bash
sudo mkdir -p /etc/containerd
sudo sed -i '/\[plugins\."io.containerd.grpc.v1.cri"\]/a\ \ sandbox_image = "registry.k8s.io/pause:3.10.1"' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

Cek ulang node tetap Ready:

```bash
kubectl get nodes
```

***

### 7) Single-node: izinkan pod jalan di control-plane

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane- || true
kubectl taint nodes --all node-role.kubernetes.io/master- || true
```

***

### 8) Storage untuk MySQL (wajib): Local Path Provisioner

Local Path Provisioner menyediakan dynamic PV berbasis local storage node. ([GitHub](https://github.com/rancher/local-path-provisioner?utm_source=chatgpt.com))

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
kubectl -n local-path-storage get pods -o wide
kubectl get storageclass
```

Pastikan `local-path` ada (biasanya jadi default, tapi cek):

```bash
kubectl get storageclass -o wide
```

Kalau belum default:

```bash
kubectl patch storageclass local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### (Opsional tapi sangat disarankan) Test cepat PVC benar-benar bisa provision

Buat PVC test:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 128Mi
EOF
```

Cek status PVC:

```bash
kubectl get pvc test-pvc
kubectl get pv | head
```

Kalau PVC sudah `Bound`, berarti storage provisioning beres ✅

Bersihkan test:

```bash
kubectl delete pvc test-pvc
```

### Fix cepat (aman): hapus baris tambahan + ganti yang lama

1. **Lihat ada berapa `sandbox_image`**

```bash
sudo grep -n 'sandbox_image' /etc/containerd/config.toml
```

Kalau kamu lihat **lebih dari 1 baris**, itu penyebabnya.

2. **Hapus baris tambahan yang kamu sisipkan (pause:3.10.1), lalu ubah yang lama (pause:3.8 → 3.10.1)**

```bash
# hapus semua baris yang tepat berisi pause:3.10.1 (yang hasil sisipan)
sudo sed -i '/sandbox_image = "registry\.k8s\.io\/pause:3\.10\.1"/d' /etc/containerd/config.toml

# ganti baris lama 3.8 jadi 3.10.1
sudo sed -i 's|sandbox_image = "registry.k8s.io/pause:3.8"|sandbox_image = "registry.k8s.io/pause:3.10.1"|' /etc/containerd/config.toml
```

3. **Pastikan sekarang cuma 1 baris**

```bash
sudo grep -n 'sandbox_image' /etc/containerd/config.toml
```

Target: hanya **1** hasil, nilainya `pause:3.10.1`.

4. **Restart containerd + kubelet**

```bash
sudo systemctl restart containerd
sudo systemctl restart kubelet
sudo systemctl status containerd --no-pager -l
```

Kalau masih gagal, kirim output ini:

```bash
sudo journalctl -xeu containerd.service --no-pager | tail -n 80
```

***

### 9) Install MetalLB (LoadBalancer di bare-metal VM)

MetalLB butuh install + konfigurasi **IPAddressPool** dan **L2Advertisement**. ([metallb.io](https://metallb.io/installation/?utm_source=chatgpt.com))

#### 9.1 Install MetalLB

Pakai kustomize remote (contoh official). ([metallb.universe.tf](https://metallb.universe.tf/installation/?utm_source=chatgpt.com))

```bash
kubectl apply -k "github.com/metallb/metallb/config/native?ref=v0.15.3"
kubectl -n metallb-system get pods
```

#### 9.2 Konfigurasi pool IP (pilih range yang _tidak bentrok_ dengan VM)

Aku sarankan: `192.168.56.240-192.168.56.250`

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: pool-hostonly
  namespace: metallb-system
spec:
  addresses:
  - 192.168.56.240-192.168.56.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-hostonly
  namespace: metallb-system
spec: {}
EOF
```

### Yang harus kamu lakukan (fix)

#### 1) Tunggu MetalLB pod benar-benar Running

```bash
watch -n 2 kubectl -n metallb-system get pods -o wide
```

Target:

* controller: **1/1 Running**
* speaker: **1/1 Running** (daemonset)

#### 2) Pastikan webhook service sudah punya endpoints

```bash
kubectl -n metallb-system get svc metallb-webhook-service
kubectl -n metallb-system get endpoints metallb-webhook-service -o wide
```

Kalau endpoints masih kosong / none → webhook memang belum siap.

#### 3) Kalau pod nyangkut (ContainerCreating / ImagePullBackOff), cek penyebabnya

```bash
kubectl -n metallb-system describe pod -l app=metallb,component=controller | sed -n '/Events:/,$p'
kubectl -n metallb-system describe pod -l app=metallb,component=speaker   | sed -n '/Events:/,$p'
```

Biasanya ketemu salah satu ini:

* **ImagePullBackOff** (internet/DNS ke registry image MetalLB)
* masalah CNI (tapi node kamu sudah Ready + flannel sudah apply, jadi kemungkinan kecil)

#### 4) Setelah pod Running + endpoints ada, **apply ulang** IP pool

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: pool-hostonly
  namespace: metallb-system
spec:
  addresses:
  - 192.168.56.240-192.168.56.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-hostonly
  namespace: metallb-system
spec: {}
EOF
```

MetalLB memang butuh IPAddressPool supaya bisa assign IP ke Service type LoadBalancer. [metallb.io](https://metallb.io/configuration/?utm_source=chatgpt.com)

***

#### Catatan “jangan panik”

Error ini **tidak mengganggu cluster**; hanya berarti **MetalLB belum bisa dikonfigurasi** sampai webhook siap.

Kalau kamu kirim output:

```bash
kubectl -n metallb-system get pods -o wide
kubectl -n metallb-system get endpoints metallb-webhook-service -o wide
```

Iya, **sekarang sudah aman** dan kamu **bisa lanjut**. ✅\
Buktinya:

* `controller` **Running**
* `speaker` **Running**
* `metallb-webhook-service` sudah punya **endpoint `10.244.0.5:9443`** → webhook sudah bisa di-call

Error sebelumnya murni karena webhook belum siap.

Yang perlu kamu lakukan sekarang: **apply ulang IPAddressPool + L2Advertisement** (karena tadi gagal).

***

### A) Apply ulang konfigurasi MetalLB (wajib)

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: pool-hostonly
  namespace: metallb-system
spec:
  addresses:
  - 192.168.56.240-192.168.56.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-hostonly
  namespace: metallb-system
spec: {}
EOF
```

Verifikasi:

```bash
kubectl -n metallb-system get ipaddresspool,l2advertisement
```

***

### B) Pastikan “memberlist” secret sudah ada (biar benar-benar clean)

Event speaker kamu tadi sempat bilang `secret "memberlist" not found` (itu sempat terjadi saat awal start). Sekarang speaker sudah Running, tapi kita cek saja supaya rapi:

```bash
kubectl -n metallb-system get secret memberlist
```

Kalau **ADA** → aman, lanjut.

Kalau **TIDAK ADA**, buat manual (aman dibuat kapan pun):

```bash
kubectl -n metallb-system create secret generic memberlist \
  --from-literal=secretkey="$(openssl rand -base64 128)"
kubectl -n metallb-system rollout restart ds/speaker
kubectl -n metallb-system get pods -o wide
```

***

### C) Test MetalLB benar-benar ngasih IP (opsional tapi aku sarankan)

Biar yakin pool kamu kepakai, bikin service LoadBalancer dummy:

```bash
kubectl create deploy lb-test --image=nginx --replicas=1
kubectl expose deploy lb-test --port 80 --type LoadBalancer
watch -n 2 kubectl get svc lb-test
```

Harus dapat `EXTERNAL-IP` dari range `192.168.56.240-250`.

Kalau sudah dapat, bersihkan:

```bash
kubectl delete svc lb-test
kubectl delete deploy lb-test
```

***

### 10) Install Ingress-NGINX

Ingress-NGINX punya guide instalasi resmi + catatan bare-metal. ([kubernetes.github.io](https://kubernetes.github.io/ingress-nginx/deploy/?utm_source=chatgpt.com))\
Versi terbaru (contoh saat ini) terlihat di release repo ingress-nginx. ([GitHub](https://github.com/kubernetes/ingress-nginx/releases?utm_source=chatgpt.com))

#### 10.1 Install (manifest “cloud” → Service type LoadBalancer cocok untuk MetalLB)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.14.1/deploy/static/provider/cloud/deploy.yaml
kubectl -n ingress-nginx get pods
kubectl -n ingress-nginx get svc
```

Tunggu `ingress-nginx-controller` dapat `EXTERNAL-IP` dari range MetalLB:

```bash
watch -n 2 kubectl -n ingress-nginx get svc ingress-nginx-controller
```

Catat IP-nya, misal jadi: `192.168.56.240` (contoh).

***

### 11) Buat namespace staging & prod

```bash
kubectl create ns threebody-staging
kubectl create ns threebody-prod
kubectl get ns | egrep 'threebody|ingress|metallb'
```

***

### 12) Koneksi ke Harbor (imagePullSecret)

> Ini wajib supaya Pod bisa pull image `192.168.56.43:8081/threebody/...`.

Login Docker registry secret per namespace:

```bash
# ganti USER/PASS sesuai Harbor kamu (cek harbor.yml: harbor_admin_password)
HARBOR=192.168.56.43:8081
PROJ=threebody
USER=admin
PASS='Harbor12345'   # contoh default, sesuaikan

kubectl -n threebody-staging create secret docker-registry harbor-creds \
  --docker-server="$HARBOR" --docker-username="$USER" --docker-password="$PASS"

kubectl -n threebody-prod create secret docker-registry harbor-creds \
  --docker-server="$HARBOR" --docker-username="$USER" --docker-password="$PASS"
```

Attach ke default serviceaccount:

```bash
kubectl -n threebody-staging patch serviceaccount default -p '{"imagePullSecrets":[{"name":"harbor-creds"}]}'
kubectl -n threebody-prod    patch serviceaccount default -p '{"imagePullSecrets":[{"name":"harbor-creds"}]}'
```

***

### 13) Deploy MySQL di staging & prod (di K8s)

Di bawah ini template MySQL **per namespace** (jadi staging dan prod terisolasi).

#### 13.1 MySQL (staging)

```bash
cat <<'EOF' | kubectl apply -n threebody-staging -f -
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
stringData:
  MYSQL_ROOT_PASSWORD: RootPassStaging123!
  MYSQL_DATABASE: threebody
  MYSQL_USER: threebody
  MYSQL_PASSWORD: UserPassStaging123!
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
    name: mysql
  selector:
    app: mysql
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
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
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
          name: mysql
        envFrom:
        - secretRef:
            name: mysql-secret
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        readinessProbe:
          exec:
            command: ["sh","-lc","mysqladmin ping -h 127.0.0.1 -uroot -p$MYSQL_ROOT_PASSWORD --silent"]
          initialDelaySeconds: 10
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
EOF
```

#### 13.2 MySQL (prod)

```bash
cat <<'EOF' | kubectl apply -n threebody-prod -f -
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
stringData:
  MYSQL_ROOT_PASSWORD: RootPassProd123!
  MYSQL_DATABASE: threebody
  MYSQL_USER: threebody
  MYSQL_PASSWORD: UserPassProd123!
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
    name: mysql
  selector:
    app: mysql
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
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
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
          name: mysql
        envFrom:
        - secretRef:
            name: mysql-secret
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        readinessProbe:
          exec:
            command: ["sh","-lc","mysqladmin ping -h 127.0.0.1 -uroot -p$MYSQL_ROOT_PASSWORD --silent"]
          initialDelaySeconds: 10
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
EOF
```

Cek:

```bash
kubectl -n threebody-staging get pods,pvc,svc
kubectl -n threebody-prod    get pods,pvc,svc
```

***

### 14) Set variabel yang kamu pakai (sekali saja di VM1)

Pakai tag yang sama untuk staging dan prod. Untuk awal bisa pakai `local` (hasil push manual dari VM2), nanti CI ganti ke `$CI_COMMIT_SHA`.

```bash
export HARBOR=192.168.56.43:8081
export PROJ=threebody
export TAG=local

export IMG_LARAVEL=$HARBOR/$PROJ/laravel:$TAG
export IMG_GOAPI=$HARBOR/$PROJ/goapi:$TAG
export IMG_FE=$HARBOR/$PROJ/frontend:$TAG
```

Cek cepat:

```bash
echo $IMG_LARAVEL
echo $IMG_GOAPI
echo $IMG_FE
```

***

### 14.1 – Pastikan containerd & kubelet aman (biar nggak “ImagePullBackOff”)

```bash
sudo systemctl is-active containerd
sudo systemctl is-active kubelet
```

Harusnya `active`.

***

### 14.2 – Buat imagePullSecret Harbor di **staging & prod**

Ini penting supaya Pod bisa pull dari Harbor private project.

> Ganti `PASS` sesuai password admin Harbor kamu (yang kamu set di harbor.yml / saat install).

```bash
USER=admin
PASS='Harbor12345'

for ns in threebody-staging threebody-prod; do
  kubectl -n $ns create secret docker-registry harbor-creds \
    --docker-server="$HARBOR" \
    --docker-username="$USER" \
    --docker-password="$PASS" \
    --dry-run=client -o yaml | kubectl apply -f -

  kubectl -n $ns patch serviceaccount default \
    -p '{"imagePullSecrets":[{"name":"harbor-creds"}]}'
done
```

Validasi:

```bash
kubectl -n threebody-staging get sa default -o yaml | sed -n '/imagePullSecrets/,+5p'
kubectl -n threebody-prod    get sa default -o yaml | sed -n '/imagePullSecrets/,+5p'
```

***

### 14.3 – Buat APP\_KEY untuk Laravel per namespace

Laravel **wajib** APP\_KEY valid.

```bash
for ns in threebody-staging threebody-prod; do
  KEY="$(openssl rand -base64 32)"
  kubectl -n $ns create secret generic app-secret \
    --from-literal=APP_KEY="base64:${KEY}" \
    --dry-run=client -o yaml | kubectl apply -f -
done
```

***

### 14.4 – Deploy aplikasi di STAGING

Kita bikin 3 Deployment + 3 Service:

* `laravel` (port 80)
* `goapi` (port 8080)
* `frontend` (port 80)

#### 14.4.1 Apply manifest STAGING

```bash
cat <<EOF | kubectl apply -n threebody-staging -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel
spec:
  replicas: 1
  selector:
    matchLabels: {app: laravel}
  template:
    metadata:
      labels: {app: laravel}
    spec:
      containers:
      - name: laravel
        image: ${IMG_LARAVEL}
        ports:
        - containerPort: 80
        env:
        - name: APP_ENV
          value: "staging"
        - name: APP_DEBUG
          value: "false"
        - name: LOG_CHANNEL
          value: "stderr"
        - name: APP_KEY
          valueFrom: {secretKeyRef: {name: app-secret, key: APP_KEY}}
        - name: DB_CONNECTION
          value: "mysql"
        - name: DB_HOST
          value: "mysql"
        - name: DB_PORT
          value: "3306"
        - name: DB_DATABASE
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_DATABASE}}
        - name: DB_USERNAME
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_USER}}
        - name: DB_PASSWORD
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_PASSWORD}}
---
apiVersion: v1
kind: Service
metadata:
  name: laravel
spec:
  selector: {app: laravel}
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: goapi
spec:
  replicas: 1
  selector:
    matchLabels: {app: goapi}
  template:
    metadata:
      labels: {app: goapi}
    spec:
      containers:
      - name: goapi
        image: ${IMG_GOAPI}
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          value: "mysql"
        - name: DB_PORT
          value: "3306"
        - name: DB_NAME
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_DATABASE}}
        - name: DB_USER
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_USER}}
        - name: DB_PASSWORD
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_PASSWORD}}
---
apiVersion: v1
kind: Service
metadata:
  name: goapi
spec:
  selector: {app: goapi}
  ports:
  - port: 8080
    targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels: {app: frontend}
  template:
    metadata:
      labels: {app: frontend}
    spec:
      containers:
      - name: frontend
        image: ${IMG_FE}
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector: {app: frontend}
  ports:
  - port: 80
    targetPort: 80
EOF
```

Cek:

```bash
kubectl -n threebody-staging get pods,svc
```

***

### 14.5 – Deploy aplikasi di PROD

Sama seperti staging, tapi replicas lebih banyak (contoh 2).

```bash
cat <<EOF | kubectl apply -n threebody-prod -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel
spec:
  replicas: 2
  selector:
    matchLabels: {app: laravel}
  template:
    metadata:
      labels: {app: laravel}
    spec:
      containers:
      - name: laravel
        image: ${IMG_LARAVEL}
        ports:
        - containerPort: 80
        env:
        - name: APP_ENV
          value: "production"
        - name: APP_DEBUG
          value: "false"
        - name: LOG_CHANNEL
          value: "stderr"
        - name: APP_KEY
          valueFrom: {secretKeyRef: {name: app-secret, key: APP_KEY}}
        - name: DB_CONNECTION
          value: "mysql"
        - name: DB_HOST
          value: "mysql"
        - name: DB_PORT
          value: "3306"
        - name: DB_DATABASE
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_DATABASE}}
        - name: DB_USERNAME
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_USER}}
        - name: DB_PASSWORD
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_PASSWORD}}
---
apiVersion: v1
kind: Service
metadata:
  name: laravel
spec:
  selector: {app: laravel}
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: goapi
spec:
  replicas: 2
  selector:
    matchLabels: {app: goapi}
  template:
    metadata:
      labels: {app: goapi}
    spec:
      containers:
      - name: goapi
        image: ${IMG_GOAPI}
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          value: "mysql"
        - name: DB_PORT
          value: "3306"
        - name: DB_NAME
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_DATABASE}}
        - name: DB_USER
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_USER}}
        - name: DB_PASSWORD
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_PASSWORD}}
---
apiVersion: v1
kind: Service
metadata:
  name: goapi
spec:
  selector: {app: goapi}
  ports:
  - port: 8080
    targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels: {app: frontend}
  template:
    metadata:
      labels: {app: frontend}
    spec:
      containers:
      - name: frontend
        image: ${IMG_FE}
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector: {app: frontend}
  ports:
  - port: 80
    targetPort: 80
EOF
```

Cek:

```bash
kubectl -n threebody-prod get pods,svc
```

***

### 14.6 – Ingress: routing + REWRITE prefix (ini bagian yang paling sering bikin bingung)

Karena frontend kamu call:

* `/api/laravel/api/products`
* `/api/go/api/products`

Maka backend **harus menerima**:

* laravel menerima `/api/products` (tanpa prefix `/api/laravel`)
* goapi menerima `/api/products` (tanpa prefix `/api/go`)

Di dev kamu pakai `rewrite` di nginx. Di ingress-nginx kita lakukan hal yang sama pakai **use-regex + rewrite-target**.

> Penting: **Ingress frontend (`/`) harus dipisah** dari ingress API yang pakai rewrite-target, supaya `/` tidak ikut kerewrite.

#### 14.6.1 Ambil EXTERNAL-IP ingress controller (untuk host file)

```bash
kubectl -n ingress-nginx get svc ingress-nginx-controller -o wide
```

Misal dapat `192.168.56.241`, lalu di **Host PC** tambahkan hosts:

```
192.168.56.42 staging.local
192.168.56.42 prod.local
```

#### 14.6.2 Ingress STAGING

```bash
cat <<'EOF' | kubectl apply -n threebody-staging -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: staging-frontend
spec:
  ingressClassName: nginx
  rules:
  - host: staging.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: staging-api
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: staging.local
    http:
      paths:
      - path: /api/laravel(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: laravel
            port:
              number: 80
      - path: /api/go(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: goapi
            port:
              number: 8080
EOF
```

#### 14.6.3 Ingress PROD

```bash
cat <<'EOF' | kubectl apply -n threebody-prod -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prod-frontend
spec:
  ingressClassName: nginx
  rules:
  - host: prod.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prod-api
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: prod.local
    http:
      paths:
      - path: /api/laravel(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: laravel
            port:
              number: 80
      - path: /api/go(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: goapi
            port:
              number: 8080
EOF
```

Cek:

```bash
kubectl -n threebody-staging get ingress
kubectl -n threebody-prod get ingress
```

***

### 14.7 – Jalankan migrate Laravel (wajib, seperti di VM2)

Di VM2:

```bash
ssh cikal@192.168.56.42
```

Di VM1:

STAGING:

```bash
kubectl -n threebody-staging exec -it deploy/laravel -- sh -lc 'php artisan migrate --force'
```

PROD:

```bash
kubectl -n threebody-prod exec -it deploy/laravel -- sh -lc 'php artisan migrate --force'
```

### Opsi B (recommended untuk CI/CD): install `kubectl` di VM2 + ambil kubeconfig dari VM1

Ini yang paling nyambung untuk requirement kamu (CI deploy dari VM2 ke K8s).

#### B1) Install `kubectl` di VM2 (client saja)

Di **VM2**:

```bash
sudo apt update
sudo apt install -y curl ca-certificates gnupg

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubectl
sudo apt-mark hold kubectl

kubectl version --client
```

#### B2) Copy kubeconfig dari VM1 ke VM2

Di **VM2**:

```bash
mkdir -p ~/.kube
scp cikal@192.168.56.42:~/.kube/config ~/.kube/config
chmod 600 ~/.kube/config
```

Cek alamat API server di kubeconfig (harus ke 192.168.56.42:6443):

```bash
grep -n "server:" ~/.kube/config
```

Kalau ternyata masih `127.0.0.1:6443`, fix:

```bash
sed -i 's#https://127.0.0.1:6443#https://192.168.56.42:6443#g' ~/.kube/config
```

Test koneksi:

```bash
kubectl get nodes
```

#### B3) Baru jalankan migrate dari VM2

```bash
kubectl -n threebody-staging exec -it deploy/laravel -- sh -lc 'php artisan migrate --force'
kubectl -n threebody-prod    exec -it deploy/laravel -- sh -lc 'php artisan migrate --force'
```

***

### 14.8 – Validasi end-to-end (paling jelas pakai curl Host header)

Ambil dulu IP ingress (EXTERNAL-IP):

```bash
ING_IP=$(kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $ING_IP
```

Test STAGING:

```bash
curl -H 'Host: staging.local' http://$ING_IP/ -I
curl -H 'Host: staging.local' http://$ING_IP/api/go/api/products
curl -H 'Host: staging.local' http://$ING_IP/api/laravel/api/products
```

Test PROD:

```bash
curl -H 'Host: prod.local' http://$ING_IP/ -I
curl -H 'Host: prod.local' http://$ING_IP/api/go/api/products
curl -H 'Host: prod.local' http://$ING_IP/api/laravel/api/products
```

***

## Kalau masih bingung, biasanya nyangkut di 1 dari 3 titik ini

1. **ImagePullBackOff** → secret harbor belum kepasang / containerd belum bisa HTTP registry\
   Cek: `kubectl -n threebody-staging describe pod <pod>` (lihat Events)
2. **Laravel 500** → APP\_KEY belum valid atau belum migrate\
   Cek log: `kubectl -n threebody-staging logs deploy/laravel --tail=200`
3. **API path 404** → ingress rewrite belum benar\
   Pastikan kamu pakai ingress **terpisah**: frontend tanpa rewrite, api dengan rewrite-target `/$2`.

***

Kalau kamu paste output ini, aku bisa “bedah” persis posisi bingungnya kamu di mana:

```bash
kubectl -n threebody-staging get pods,svc,ing
kubectl -n threebody-staging logs deploy/laravel --tail=80
kubectl -n ingress-nginx get svc ingress-nginx-controller -o wide
```

***

### Solusi paling cepat & stabil untuk demo: pakai IP VM1, bukan VIP MetalLB

Kalau tujuan kamu adalah “URL bisa kebuka di browser host”, cara paling aman di lab VirtualBox adalah **Inginress bisa diakses lewat IP node VM1**.

#### A. Patch ingress-nginx service supaya juga listen di IP VM1

Di **VM1 (k8s-master)**:

```bash
kubectl -n ingress-nginx patch svc ingress-nginx-controller --type merge \
  -p '{"spec":{"externalIPs":["192.168.56.42"]}}'

kubectl -n ingress-nginx get svc ingress-nginx-controller -o wide
```

#### B. Ubah hosts Windows supaya staging/prod mengarah ke VM1

Ubah jadi:

```
192.168.56.42 staging.local
192.168.56.42 prod.local
```

Lalu:

```bat
ipconfig /flushdns
```

#### 4.3 Akses dari browser

Buka:

* `http://staging.local`

```
http://staging.local
```

* `http://prod.local`

```
http://prod.local
```

Ini biasanya langsung beres karena tidak bergantung pada ARP “VIP” MetalLB.

## Kalau ternyata DB memang kosong: cara beresin

### 3) Clear cache Laravel + migrate + seed

Kadang config/cache bikin Laravel “nempel” ke nilai lama, jadi aku selalu clear dulu.

**STAGING**

```bash
kubectl -n threebody-staging exec deploy/laravel -- sh -lc '
  php artisan optimize:clear || true
  php artisan migrate --force
  php artisan db:seed --force || true
'
```

**PROD**

```bash
kubectl -n threebody-prod exec deploy/laravel -- sh -lc '
  php artisan optimize:clear || true
  php artisan migrate --force
  php artisan db:seed --force || true
'
```

> Kalau `db:seed` ternyata “tidak melakukan apa-apa” (tidak ada seeder / tidak insert data), itu normal—berarti project memang belum menyiapkan seed data.

***

### 18) Checklist “lulus setup VM1”

* `kubectl get nodes` = **Ready**
* `kubectl -n kube-system get pods` = core pods running
* `kubectl -n ingress-nginx get pods` = running & svc dapat **EXTERNAL-IP**
* `kubectl -n metallb-system get pods` = controller/speaker running
* `kubectl -n threebody-staging get pods,pvc,ing` = semua running
* `kubectl -n threebody-prod get pods,pvc,ing` = semua running
* akses `staging.local` & `prod.local` dari Host PC jalan

***

### Troubleshooting cepat (yang paling sering kejadian)

1. **Node NotReady / CoreDNS Pending** → biasanya CNI belum jalan / Pod CIDR mismatch. Pastikan `kubeadm init` pakai `10.244.0.0/16` dan Flannel ter-apply. ([Stack Overflow](https://stackoverflow.com/questions/48984659/understanding-kubeadm-init-command-for-flannel?utm_source=chatgpt.com))
2. **ImagePullBackOff dari Harbor** → containerd belum diset HTTP registry + belum ada imagePullSecret. (langkah 3.2 + 12) ([GitHub](https://github.com/containerd/containerd/blob/main/docs/hosts.md?utm_source=chatgpt.com))
3. **Ingress EXTERNAL-IP pending** → MetalLB belum terkonfigurasi IPAddressPool/L2Advertisement. ([metallb.io](https://metallb.io/configuration/?utm_source=chatgpt.com))
4. **PVC pending** → StorageClass default belum ada → pastikan local-path-provisioner terpasang. ([GitHub](https://github.com/rancher/local-path-provisioner?utm_source=chatgpt.com))

***

Kalau VM1 sudah sampai **Ingress punya EXTERNAL-IP** dan namespace + MySQL + app pod jalan, langkah berikutnya biasanya:

* **CI/CD deploy dari VM2** (GitLab Runner) ke namespace staging/prod (kubeconfig/serviceaccount + kubectl apply / helm),
* dan integrasi monitoring ke VM3.

Kalau kamu mau, aku bisa lanjutkan runbook berikutnya: **“CI deploy ke staging lalu promote ke prod (tag immutable dari Harbor)”** tanpa rebuild image, sesuai requirement kamu.
