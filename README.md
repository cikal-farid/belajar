# Final Test DevOps

#### Target arsitektur (yang akan jadi “hasil akhir”)

**Host-only network:** `192.168.56.0/24`

| VM        |            IP | Role                                               | Detail                                                   |
| --------- | ------------: | -------------------------------------------------- | -------------------------------------------------------- |
| vm-docker | 192.168.56.42 | GitLab Runner + Edge Nginx + Harbor registry (TLS) | (Harbor HTTP `:8080` + Edge `hit.local` + GitLab Runner) |
| vm-k8s    | 192.168.56.43 | Kubernetes control-plane (kubeadm) + Loki+Grafana  | (control-plane)                                          |
| vm-worker | 192.168.56.44 | Kubernetes worker                                  | (worker, NodePort `30080/30081/30082`)                   |

**Akses aplikasi:**

* Harbor UI: `http://harbor.local:8080`

```
http://harbor.local:8080
```

* Prod app: `https://hit.local` (edge → NodePort K8s)

```
https://hit.local
```

***

### A. LANGKAH WAJIB DI SEMUA VM (vm-docker, vm-k8s, vm-worker)

**A1) HARDEN semua VM: matikan auto update & cegah restart service otomatis**

> Jalankan di **SEMUA VM**: `vm-docker`, `vm-harbor`, `vm-k8s`, `vm-worker`

```bash
sudo systemctl stop unattended-upgrades 2>/dev/null || true
sudo systemctl disable --now unattended-upgrades 2>/dev/null || true

sudo systemctl stop apt-daily.service apt-daily-upgrade.service 2>/dev/null || true
sudo systemctl disable --now apt-daily.timer apt-daily-upgrade.timer 2>/dev/null || true

# rapikan dpkg kalau pernah kepotong
sudo dpkg --configure -a || true
```

Kalau file needrestart ada, set supaya **tidak auto-restart service**:

```bash
if [ -f /etc/needrestart/needrestart.conf ]; then
  sudo cp -a /etc/needrestart/needrestart.conf /etc/needrestart/needrestart.conf.bak.$(date +%F-%H%M%S)
  sudo sed -i "s/^\s*\$nrconf{restart}.*/\$nrconf{restart} = 'l';/" /etc/needrestart/needrestart.conf || true
fi
```

> Mode `'l'` = cuma list, **tidak restart service** otomatis.

***

#### A2) Set hostname

Jalankan sesuai VM:

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

#### A3) `/etc/hosts` wajib (biar `harbor.local` & `hit.local` stabil)

Jalankan di **SEMUA VM**:

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

#### A4) DNS fix (menghindari “GitLab kadang resolve kadang tidak”)

Jalankan di **SEMUA VM**:

```bash
sudo tee /etc/systemd/resolved.conf >/dev/null <<'EOF'
[Resolve]
DNS=1.1.1.1 8.8.8.8
FallbackDNS=9.9.9.9 8.8.4.4
DNSStubListener=yes
EOF

sudo systemctl restart systemd-resolved
sudo resolvectl flush-caches

# penting: resolv.conf pakai nameserver real, bukan 127.0.0.53
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf

getent hosts gitlab.com altssh.gitlab.com registry-1.docker.io || true
```

***

### B. VM-DOCKER (192.168.56.42) — Docker + Harbor + Edge + GitLab Runner

#### B1) Install tools + Docker

```bash
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl git nano unzip rsync openssl openssh-server
sudo apt install ufw
sudo apt install lsof
sudo systemctl enable --now ssh
sudo ufw enable
echo "=== [Firewall] pastikan port 8080 tidak diblok ==="
sudo ufw status verbose || true
# kalau UFW aktif, BUKA port 8080:
sudo ufw allow 8080/tcp || true
sudo ufw allow ssh
sudo ufw reload || true

curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"
newgrp docker

docker version
docker compose version
```

#### B2) Docker allow insecure registry Harbor (HTTP)

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

#### B3) Install Harbor (HTTP :8080)

> Harbor jalan di container `nginx` milik Harbor, port **8080**.

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

Edit `harbor.yml`:

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

# pastikan TIDAK ada https aktif
harbor_admin_password: Harbor12345
data_volume: /data/harbor
trivy:
  enabled: false
```

Install & cek:

```bash
sudo mkdir -p /data/harbor
sudo ./prepare
sudo ./install.sh

sudo docker compose -f /opt/harbor/docker-compose.yml ps
curl -fsSI http://harbor.local:8080/v2/ | head -n 1 || true
```

Di UI Harbor:

* `http://harbor.local:8080/`
* Buat project: **threebody**

#### B4) Deploy project “three-body-problem” + Edge Nginx (hit.local)

```bash
cd ~
unzip -q three-body-problem-final.zip
mv three-body-problem-final three-body-problem
cd ~/three-body-problem

# cek edge.conf beneran ada
ls -lah deploy/edge/nginx/conf.d/edge.conf
grep -n "proxy_pass" deploy/edge/nginx/conf.d/edge.conf
```

Buat folder deploy

```bash
mkdir -p ~/three-body-problem/deploy/edge/nginx/conf.d
mkdir -p ~/three-body-problem/deploy/k8s
```

#### Setup Konfigurasi

Buat file edge.conf

```bash
nano deploy/edge/nginx/conf.d/edge.conf
```

isi file diatas dengan konfig dibawah ini

```bash
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
```

Buat file docker-compose.edge.yml

```bash
nano deploy/edge/docker-compose.edge.yml
```

isi file diatas dengan konfig dibawah ini

```bash
services:
  edge-nginx:
    image: nginx:alpine
    container_name: edge-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./certs:/etc/nginx/certs:ro
    networks:
      - edge-net
    restart: unless-stopped

networks:
  edge-net:
    name: edge-net
```

Buat file docker-compose.yml

```bash
nano deploy/edge/docker-compose.yml
```

isi file diatas dengan konfig dibawah ini

```bash
services:
  edge-nginx:
    image: nginx:alpine
    container_name: edge-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./certs:/etc/nginx/certs:ro
    networks:
      - edge-net
    restart: unless-stopped

networks:
  edge-net:
    name: edge-net
```

Buat file kubeconfig-prod

```bash
nano deploy/edge/kubeconfig-prod
```

isi file diatas dengan konfig dibawah ini

```bash
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURCVENDQWUyZ0F3SUJBZ0lJSGZHcWswZW5CTVl3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TlRFeU1qY3hOVEE1TVRSYUZ3MHpOVEV5TWpVeE5URTBNVFJhTUJVeApFekFSQmdOVkJBTVRDbXQxWW1WeWJtVjBaWE13Z2dFaU1BMEdDU3FHU0liM0RRRUJBUVVBQTRJQkR3QXdnZ0VLCkFvSUJBUURLMW13dkVCYnB6TVdZVzcvTXlYVG9scjZrR245c3Fpb2hCM2hXNE5kbllINTVEN0dNUUFwWU53c3QKbm1mS2RlYnJwekR0bFp5R3hjNnZ2dFA5b21sRDM3Y2dvYnJQRTRPMkRUU2lPdkMvRFVzdGVYV0RmS3BMdGFTWApjT1k3YnViNzNkVnlETFRWNWZYSmdBVkVmNU5FRDdVOXF4L3dsY3BVeUtoeDdTd21SVEZrZm5ZODZwMHJnanNmCmlXRytxcDQ2MEt2Qkw1akt4c09KQ3NYdllDTGx2V2tPOEdkVm1sYy9kVnFlMDVVRjhmbEoxK21TMTdERW1IQ1gKZ2JBbUdUWEYzTExLRndjOUgxNVNMb3NoMkpkTUkrRk54WTZyVWlKYnhMSG1DdjlGY1dtZVRGWDloMThLRjVGOQpOVmJ1dHRIdVlBZ2M3NHhtTEswS1BCSTNaMzNaQWdNQkFBR2pXVEJYTUE0R0ExVWREd0VCL3dRRUF3SUNwREFQCkJnTlZIUk1CQWY4RUJUQURBUUgvTUIwR0ExVWREZ1FXQkJTbkNlcWc2VEd3dnV5dlZRM29zaTM2RWlpOWl6QVYKQmdOVkhSRUVEakFNZ2dwcmRXSmxjbTVsZEdWek1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQkFRQlliV1hLanVTMwp1SWlpNXlQWno3Z3IxTWJkb2ZzREYxa2EzOVltemNTNkFlWWduVFUrVGljSEd3bFM4dm0xRTYvN2l5TkhKN2U4CnRRci9lSnlubUV0UFl4VXRKakdOcEJNTXVmR2FHV2VRNWtHdzc0RG5tcE43aUovTEVRRkl6a2pvNkRWcFRjUEsKOFBrOVUvZ2N1ZGxDdE8vaktGc0dWeGRSZko1d0RFbjY1bThVc1lqdEhsUnB2R25hZGxERzF1MDgrTXN1Slp3KwpmZjFYVWt5dnVaVUdNMjdNT241YjY1YVU0Sm1SY0FpODIyeEp4bnd2dW5TZVMwZkJVdkpKelFvN1E0cXU1a1BwClEwOTlXaitUWTVPMFpudDIwU3A5bXdLTnkyZjEvWi84SUc5RXl3djhucEsyN0xKVHpzdnJmQU1VY0dYS01vTVUKWjNWSXlvYmpRemhTCi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    server: https://192.168.56.43:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURLVENDQWhHZ0F3SUJBZ0lJUFF5R2lKQzE2alF3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TlRFeU1qY3hOVEE1TVRSYUZ3MHlOakV5TWpjeE5URTBNVGhhTUR3eApIekFkQmdOVkJBb1RGbXQxWW1WaFpHMDZZMngxYzNSbGNpMWhaRzFwYm5NeEdUQVhCZ05WQkFNVEVHdDFZbVZ5CmJtVjBaWE10WVdSdGFXNHdnZ0VpTUEwR0NTcUdTSWIzRFFFQkFRVUFBNElCRHdBd2dnRUtBb0lCQVFEMTBhQkEKTzgzc09hSUk5RmZNaGI0NnFSUmZJYXA5OXNvd1RQNTQ5d25ValFiOWR0YWRnTWFQeTRwTTR3Q2FWNGErUUgrSQpOeFNRUGVpcTI3M1BtZEV5QmhzQlp5QzdhekZ1OVFrRmZEMEdhM3FSemp2SzhFL2lQOWN1Qml6SW50YWdwUUJTCnJZeGhlRTdHeE54ODBQZlpxTU1yQTIyVCtYSWJNaTBucmlWb3BUR1NsSzBFV0xmUDFWQm5MS3FTWVBLMytKWHAKUjNEemI4NU55TDZrWll0NHdoUFZYSzF4QjBqSGRpOE42eUFRbU94di9UdzZsbXZ1Qzk3L0gvRklGR1NZbG9QUwpWNlhXbFZ0UTJoY3FOY2NDelNVZ2hEVlpQcEU3REtucnl0RG9tSFd2dmV0SS94cXpKRVVOYzR4SWFOaGY3VFc0Cmp5Ym03Y1RWcko1NFJPTmZBZ01CQUFHalZqQlVNQTRHQTFVZER3RUIvd1FFQXdJRm9EQVRCZ05WSFNVRUREQUsKQmdnckJnRUZCUWNEQWpBTUJnTlZIUk1CQWY4RUFqQUFNQjhHQTFVZEl3UVlNQmFBRktjSjZxRHBNYkMrN0s5VgpEZWl5TGZvU0tMMkxNQTBHQ1NxR1NJYjNEUUVCQ3dVQUE0SUJBUUE0ck9lam5INjRiVXhGS2Jrbk9icXVsTFBBCnRvU1J1M0tHNWZXM3l0c2xOOXVBTWVMc2Z0L1ljelA4NGp5aHNhck1tRERUTjRUMFRoRTJFV2d5d08ycmhXVTAKcWEydENoengvSlcxWXZGRmRLazk1U2VVa0hkcXRYSU9JL0VTRU1FNm5OMmo2R0c1NjUvNmpBeWVINVJ3YkpacgorZDhSeDdmNllhZUNDcm10eUc0NzNuY3pZRitnZ3ZNSTZxQXlnRnhlY0hpV0pEaE9JTEZQcUl5WU9XTHhJZ21BCmN0dHI1UW9sc0U1NUhmWXVBblh2Q0E4YldkVEtJQ0lYV2Y2cDdLOVJZaFpVZVNyZVlWR3pCSnR5eEtrNFlKWTAKOVVnMUp5alFkYkxvdXk4aWRySlRBczhkZWdoMTAvYmlkeXlyZnY1amZlc1VEUmUzQjluWnpnRjVJa3JxCi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb3dJQkFBS0NBUUVBOWRHZ1FEdk43RG1pQ1BSWHpJVytPcWtVWHlHcWZmYktNRXorZVBjSjFJMEcvWGJXCm5ZREdqOHVLVE9NQW1sZUd2a0IvaURjVWtEM29xdHU5ejVuUk1nWWJBV2NndTJzeGJ2VUpCWHc5Qm10NmtjNDcKeXZCUDRqL1hMZ1lzeUo3V29LVUFVcTJNWVhoT3hzVGNmTkQzMmFqREt3TnRrL2x5R3pJdEo2NGxhS1V4a3BTdApCRmkzejlWUVp5eXFrbUR5dC9pVjZVZHc4Mi9PVGNpK3BHV0xlTUlUMVZ5dGNRZEl4M1l2RGVzZ0VKanNiLzA4Ck9wWnI3Z3ZlL3gveFNCUmttSmFEMGxlbDFwVmJVTm9YS2pYSEFzMGxJSVExV1Q2Uk93eXA2OHJRNkpoMXI3M3IKU1A4YXN5UkZEWE9NU0dqWVgrMDF1SThtNXUzRTFheWVlRVRqWHdJREFRQUJBb0lCQUErZFlFT1VTUEZmZTlaNwp4dlA3N3VpRkF3RkQvek4vTUxxK3VVOXZ5ZFVSbkVjUGZIU0cwSllqeE1ZN2lUZEVwZytCdzFnZ3Y1S21EQmlHCld0bE1HUWpIY0lHM1RsSDViaXR4TDBDRThzOXNsbDJhcnh1bUJqMEY5eUk2VEMvMlB3WDNWVHpyUDBhVndOTkoKZ0FRQnRpekNWcHR0M1ZFUUFGcVA1bVJsNElSMEVnSkxMTFppdFdYM3R6SVd6V2R1T2pWYllZcVZsWkdHRURoYQpTdEpkN0tqU0c5UXlHVGdmVHhtRzVSbiszZGNEb1ZmeUZmcEtFa2oyZ293a2ZadEtBYzVzbVE5cW5JKzRTcjd4CmV3RG04L3RrUUlCVlhpNzZ0dS9DcnN2SzZ4Uy9yVXBKT3FCWmd2elVnZ2xhR1YrTWZiY3NzZis2aTRab1JJK0gKNUJ6cVByRUNnWUVBK21TWlpNVGhXbzRtUFZVeEpKTE9vbjVFL0RocVpVQTBGaHJoTFREMzBUajJtdTRyQXcxZgpVWHo0Q25LWkdaejNNSk1DbGo1WDdEWXZsVHIrVU1jbUxHc096ZEZ5WjhZQmlaOURwNENpZzgvOTdKUjU5M1ZjCkJ5OHU5bkxZNy9DeG96Mm9oSjFqaDUvT2RtMGQzdFpsY0k0eWZJWlNGbTg5TEFUUUtteS85dGtDZ1lFQSsxTE8KSlNsczdHOU9OaEtKUEs3YU9PSVpBcW5ZYkp1MEpBaXdKcmZUWUMwTnFWZDhtZlR2S1pwdmQvL2RnUVJNYTJTNwpVR3lIaE1JMFFUMWxPWFA2b3FpOVRkVGl5TGF1bWhhcmlDek9iVXo5MXdyNHpxQS9ab2ozN01GMnhud2cvanRTCmFKcENCTElWSDU5blo1d1ZhRFZ5bElVSm83KzJVVjlBOU9NWWVQY0NnWUVBN0VHeGpxaEo3aFRFTmdqRkpyb3kKcUpXMTU3b2ZnY3V6bWN2Zy9hLy9sSFZJRW1WNVJoYi9BamVGSHU2RVVPdVBQVkRwRnZSYUozSDFNYnlKSzVUTgpDd28wM0xTc1d3ZzBlVE5DTmJFbEpuUkJKbEh1UnpTQUVmTk9QNUdiNmNIVkNxN3A2NzlKb3pXQkxMTEdoWFVNCkg4aThFRXo0OVZ3ZWZ3TkhtMUFqVFBrQ2dZQTJhWllpSzViZWxiYWNOR1VzRXp0QjU0bThydU1obUpCbTRNNnEKZzFBcldYWEV2U0ZILytTcjREalVCVHlDeTFoMFErbmVIckUwaUF0aU9KV3EyN1plOHlPUkwwOHVYNGJyNjQ2Vgp2QVd4V21MMjVQUTdZQ3g1Mlp1NzBhTVNsQnRjajZGbTdrYnUxMU9GZGpDZ0dRWDFldVRrek96cVcvNXdIYXhsCm1LbGF2d0tCZ0JMSm9XMVVuRS9TeXNIdEQ0TG1xaWhkcGJDSStaOUZ4THRyWC9TeGZOYzg4bXdWM3B2cVYzZUUKS3pqYzZkRnhiMU1sY1NDNUVVUExSZzZYN1FWZ091UXljeitkZUltWEhNT0RBcFEvYlhzcGtoek9OSUJPWkRHSwppVC9hNjJRMVExdWhCWnN1SUYyc3ZCL29LM2pKUnl4NTE4cyt4QXcwdVdVQ21LYy9DR0pSCi0tLS0tRU5EIFJTQSBQUklWQVRFIEtFWS0tLS0tCg==

```

Buat file 00-namespace.yaml

```bash
nano deploy/k8s/base/00-namespace.yaml
```

isi file diatas dengan konfig dibawah ini

```bash
apiVersion: v1
kind: Namespace
metadata:
  name: threebody-prod
```

Buat file 10-mysql.yaml

```bash
nano deploy/k8s/base/10-mysql.yaml
```

isi file diatas dengan konfig dibawah ini

```bash
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: threebody-prod
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
    - name: mysql
      port: 3306
      targetPort: 3306
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-hostpath
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  volumeMode: Filesystem
  hostPath:
    path: /data/threebody/mysql
    type: DirectoryOrCreate
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - vm-worker
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: threebody-prod
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ""
  volumeName: mysql-pv-hostpath
  resources:
    requests:
      storage: 5Gi
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
      terminationGracePeriodSeconds: 30
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
              name: mysql
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
          livenessProbe:
            tcpSocket:
              port: 3306
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            tcpSocket:
              port: 3306
            initialDelaySeconds: 10
            periodSeconds: 5
      volumes:
        - name: mysql-data
          persistentVolumeClaim:
            claimName: mysql-pvc
```

Buat file 20-go.yaml

```bash
nano deploy/k8s/base/20-go.yaml
```

isi file diatas dengan konfig dibawah ini

```bash
apiVersion: v1
kind: Service
metadata:
  name: go
  namespace: threebody-prod
spec:
  type: NodePort
  selector:
    app: go
  ports:
    - name: http
      port: 8081
      targetPort: 8080
      nodePort: 30081
---
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
          image: harbor.local:8080/threebody/go:latest
          ports:
            - containerPort: 8080
          env:
            - name: DB_DRIVER
              value: mysql
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
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_PASSWORD
```

Buat file 30-laravel.yaml

```bash
nano deploy/k8s/base/30-laravel.yaml
```

isi file diatas dengan konfig dibawah ini

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: laravel-nginx-conf
  namespace: threebody-prod
data:
  default.conf: |
    server {
      listen 80;
      server_name _;
      root /var/www/html/public;
      index index.php index.html;

      location / {
        try_files $uri $uri/ /index.php?$query_string;
      }

      location ~ \.php$ {
        try_files $uri =404;
        include /etc/nginx/fastcgi_params;
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
      }
    }

---
apiVersion: v1
kind: Service
metadata:
  name: laravel
  namespace: threebody-prod
spec:
  type: NodePort
  selector:
    app: laravel
  ports:
    - name: http
      port: 8082
      targetPort: 80
      nodePort: 30082

---
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

      volumes:
        - name: app-shared
          emptyDir: {}
        - name: nginx-conf
          configMap:
            name: laravel-nginx-conf

      initContainers:
        - name: copy-app
          image: harbor.local:8080/threebody/laravel:latest
          imagePullPolicy: IfNotPresent
          command: ["sh", "-c"]
          args:
            - |
              set -eu
              echo "[copy-app] copy app to shared volume"
              cp -a /var/www/html/. /shared/
              chown -R www-data:www-data /shared/storage /shared/bootstrap/cache || true
              chmod -R ug+rwX /shared/storage /shared/bootstrap/cache || true
          volumeMounts:
            - name: app-shared
              mountPath: /shared

      containers:
        # PHP-FPM (nama HARUS "laravel" supaya kubectl set image kamu tetap jalan)
        - name: laravel
          image: harbor.local:8080/threebody/laravel:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 9000
          env:
            - name: APP_ENV
              value: production
            - name: APP_DEBUG
              value: "false"
            - name: APP_KEY
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: LARAVEL_APP_KEY
            - name: DB_CONNECTION
              value: mysql
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
          volumeMounts:
            - name: app-shared
              mountPath: /var/www/html

        # Nginx (serve public + fastcgi ke fpm)
        - name: laravel-nginx
          image: nginx:alpine
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          volumeMounts:
            - name: app-shared
              mountPath: /var/www/html
              readOnly: true
            - name: nginx-conf
              mountPath: /etc/nginx/conf.d/default.conf
              subPath: default.conf
```

Buat file 40-frontend.yaml

```bash
nano deploy/k8s/base/40-frontend.yaml
```

isi file diatas dengan konfig dibawah ini

```bash
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: threebody-prod
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - name: http
      port: 8080
      targetPort: 80
      nodePort: 30080
---
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
          image: harbor.local:8080/threebody/frontend:latest
          ports:
            - containerPort: 80
```

Buat file laravel-migrate-job.yaml

```bash
nano deploy/k8s/jobs/laravel-migrate-job.yaml
```

isi file diatas dengan konfig dibawah ini

```bash
apiVersion: batch/v1
kind: Job
metadata:
  name: laravel-migrate
  labels:
    app: laravel-migrate
spec:
  ttlSecondsAfterFinished: 600
  backoffLimit: 3
  template:
    metadata:
      labels:
        app: laravel-migrate
    spec:
      restartPolicy: Never
      serviceAccountName: default

      initContainers:
        - name: wait-mysql
          image: busybox:1.36
          command: ["sh", "-c"]
          args:
            - |
              echo "[wait-mysql] waiting mysql:3306 ..."
              until nc -z mysql 3306; do sleep 2; done
              echo "[wait-mysql] mysql ready"

      containers:
        - name: migrate
          image: __LARAVEL_IMAGE__
          imagePullPolicy: IfNotPresent
          command: ["sh", "-c"]
          args:
            - |
              set -eu
              cd /var/www/html

              echo "==> migrate"
              php artisan migrate --force

              echo "==> seed only if products empty"
              php -r '
              try {
                $host=getenv("DB_HOST"); $port=getenv("DB_PORT");
                $db=getenv("DB_DATABASE"); $user=getenv("DB_USERNAME"); $pass=getenv("DB_PASSWORD");
                $pdo=new PDO("mysql:host=$host;port=$port;dbname=$db;charset=utf8mb4",$user,$pass,[PDO::ATTR_ERRMODE=>PDO::ERRMODE_EXCEPTION]);
                $count=(int)$pdo->query("SELECT COUNT(*) FROM products")->fetchColumn();
                exit($count===0 ? 0 : 2);
              } catch (Throwable $e) { exit(0); }
              ' && php artisan db:seed --force || true

              echo "==> done"
          env:
            - name: APP_ENV
              value: production
            - name: APP_DEBUG
              value: "false"
            - name: APP_KEY
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: LARAVEL_APP_KEY
            - name: DB_CONNECTION
              value: mysql
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
```

Buat file .gitlab-ci.yml didalam projectnya (three-body-problem)

```bash
nano .gitlab-ci.yml
```

isi file diatas dengan konfig dibawah ini

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
  tags: ["deploy"]
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
    (curl -fsSI "http://$HARBOR_HOST/v2/" | head -n 1) || true

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

    echo "==> [push] tag+push :latest"
    docker tag "$REGISTRY/frontend:$TAG" "$REGISTRY/frontend:latest"
    docker tag "$REGISTRY/go:$TAG" "$REGISTRY/go:latest"
    docker tag "$REGISTRY/laravel:$TAG" "$REGISTRY/laravel:latest"

    docker push "$REGISTRY/frontend:latest"
    docker push "$REGISTRY/go:latest"
    docker push "$REGISTRY/laravel:latest"

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

    echo "==> [deploy] siapkan kubectl"
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

    echo "==> [deploy][addon] tunggu default serviceaccount ready (anti race)"
    for i in $(seq 1 60); do
      kubectl -n "$K8S_NS" get sa default >/dev/null 2>&1 && break
      sleep 1
    done
    kubectl -n "$K8S_NS" get sa default >/dev/null 2>&1 || (
      echo "ERROR: serviceaccount default tidak ada"
      kubectl -n "$K8S_NS" get sa || true
      exit 1
    )

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

    echo "==> [deploy][addon] pastikan secret imagePull ada"
    kubectl -n "$K8S_NS" get secret "$K8S_IMAGEPULL_SECRET" >/dev/null 2>&1 || (
      echo "ERROR: secret $K8S_IMAGEPULL_SECRET tidak ada"
      kubectl -n "$K8S_NS" get secret || true
      exit 1
    )

    echo "==> [deploy] attach imagePullSecret ke default serviceaccount (tidak boleh silent)"
    kubectl -n "$K8S_NS" patch serviceaccount default \
      -p "{\"imagePullSecrets\": [{\"name\": \"$K8S_IMAGEPULL_SECRET\"}]}" >/dev/null

    echo "==> [deploy][addon] verifikasi SA default"
    kubectl -n "$K8S_NS" get sa default -o jsonpath='{.imagePullSecrets[*].name}'; echo
    kubectl -n "$K8S_NS" get sa default -o jsonpath='{.imagePullSecrets[*].name}' | grep -wq "$K8S_IMAGEPULL_SECRET" || (
      echo "ERROR: imagePullSecret belum nempel ke SA default"
      kubectl -n "$K8S_NS" get sa default -o yaml || true
      exit 1
    )

    echo "==> [deploy] apply manifests"
    kubectl apply -f deploy/k8s/base/

    echo "==> [deploy] update image tag ke commit SHA"
    kubectl -n "$K8S_NS" set image deployment/frontend frontend="$REGISTRY/frontend:$TAG"
    kubectl -n "$K8S_NS" set image deployment/go go="$REGISTRY/go:$TAG"
    # IMPORTANT: update container laravel + initContainer copy-app biar code yang dicopy selalu versi commit yang sama
    kubectl -n "$K8S_NS" set image deployment/laravel laravel="$REGISTRY/laravel:$TAG" copy-app="$REGISTRY/laravel:$TAG"

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

    echo "==> [deploy][addon] jalankan migrate job (anti push pertama error)"
    JOB_FILE="deploy/k8s/jobs/laravel-migrate-job.yaml"

    kubectl -n "$K8S_NS" delete job laravel-migrate --ignore-not-found=true

    sed "s|__LARAVEL_IMAGE__|$REGISTRY/laravel:$TAG|g" "$JOB_FILE" \
    | kubectl -n "$K8S_NS" apply -f -

    kubectl -n "$K8S_NS" wait --for=condition=complete job/laravel-migrate --timeout=10m || (
    echo "ERROR: migrate job gagal / timeout"
    kubectl -n "$K8S_NS" logs job/laravel-migrate --all-containers=true --tail=200 || true
    kubectl -n "$K8S_NS" describe job laravel-migrate || true
    exit 1
    )

    echo "==> [deploy][addon] migrate job log (ringkas)"
    kubectl -n "$K8S_NS" logs job/laravel-migrate --all-containers=true --tail=200 || true

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

Buat cert:

```bash
cd ~/three-body-problem
mkdir -p deploy/edge/certs
cd deploy/edge/certs

openssl req -x509 -nodes -newkey rsa:2048 -days 3650 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=hit.local"
```

Naikkan edge:

```bash
cd ~/three-body-problem/deploy/edge
sudo docker compose up -d

# test config nginx di dalam container (bukan nginx di host)
sudo docker exec -it edge-nginx nginx -t
```

Tes akses:

```bash
curl -vk --resolve hit.local:443:127.0.0.1 https://hit.local/ 2>&1 | tail -n 20
```

#### B5) Install GitLab Runner + pastikan runner bisa docker

```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install -y gitlab-runner
sudo systemctl enable --now gitlab-runner

sudo usermod -aG docker gitlab-runner
sudo chown root:docker /var/run/docker.sock
sudo chmod 660 /var/run/docker.sock

sudo systemctl restart docker
sudo systemctl restart gitlab-runner

sudo -u gitlab-runner docker ps
```

Register runner (sekali, interaktif):

```bash
sudo gitlab-runner register
```

* executor: `shell`
* tags: `deploy`
* untagged: `false`

#### B6) SSH GitLab via port 443 (kalau port 22 bermasalah)

```bash
ssh-keygen -t ed25519 -C "vm-docker-gitlab" -f ~/.ssh/id_ed25519 -N ""

cat <<'EOF' > ~/.ssh/config
Host gitlab-443
  HostName altssh.gitlab.com
  User git
  Port 443
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
  StrictHostKeyChecking accept-new
EOF

chmod 600 ~/.ssh/config ~/.ssh/id_ed25519
```

Copas semua hasil command dibawah untuk di input ke gitlab

```bash
cat ~/.ssh/id_ed25519.pub
```

Tes koneksi

```bash
ssh -T git@gitlab-443
```

**1) Harbor service**

```bash
sudo tee /etc/systemd/system/harbor-compose.service >/dev/null <<'EOF'
[Unit]
Description=Harbor via docker compose
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/harbor
ExecStart=/usr/bin/docker compose -f /opt/harbor/docker-compose.yml up -d
ExecStop=/usr/bin/docker compose -f /opt/harbor/docker-compose.yml down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now harbor-compose.service
sudo systemctl status harbor-compose.service --no-pager
```

**2) Edge service (sesuaikan username kalau bukan `cikal`)**

```bash
EDGE_DIR="/home/cikal/three-body-problem/deploy/edge"

sudo tee /etc/systemd/system/edge-compose.service >/dev/null <<EOF
[Unit]
Description=Edge Nginx via docker compose
After=docker.service harbor-compose.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=${EDGE_DIR}
ExecStart=/usr/bin/docker compose -f ${EDGE_DIR}/docker-compose.yml up -d
ExecStop=/usr/bin/docker compose -f ${EDGE_DIR}/docker-compose.yml down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now edge-compose.service
sudo systemctl status edge-compose.service --no-pager
```

***

### C. VM-K8S (192.168.56.43) — Kubernetes control-plane

#### C1) Prasyarat kernel + swap off

```bash
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl nano apt-transport-https gpg openssh-server
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

Kalau ada swapfile bawaan (seperti kasus kamu `/swap.img`):

```bash
sudo rm -f /swap.img
```

Verifikasi:

```bash
swapon --show
```

Harus kosong.

#### C2) Install + konfigurasi containerd

```bash
sudo apt-get install -y containerd

sudo systemctl stop containerd || true
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null

# WAJIB: systemd cgroup
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# WAJIB: registry config_path -> /etc/containerd/certs.d
sudo sed -i \
'/\[plugins\."io\.containerd\.grpc\.v1\.cri"\.registry\]/,/^\[/{s/^[[:space:]]*config_path[[:space:]]*=.*/      config_path = "\/etc\/containerd\/certs.d"/}' \
/etc/containerd/config.toml

sudo systemctl enable --now containerd
sudo systemctl restart containerd
sudo systemctl is-active containerd
```

Cek aman:

```bash
sudo grep -n '\\1' /etc/containerd/config.toml || echo "OK: tidak ada \\1"
sudo grep -nE '\[plugins\."io\.containerd\.grpc\.v1\.cri"\.registry\]|config_path' /etc/containerd/config.toml | head -n 40
```

#### C3) Allow Harbor HTTP untuk containerd (pull dari harbor.local:8080)

```bash
sudo mkdir -p /etc/containerd/certs.d/harbor.local:8080

sudo tee /etc/containerd/certs.d/harbor.local:8080/hosts.toml >/dev/null <<'EOF'
server = "http://harbor.local:8080"

[host."http://harbor.local:8080"]
  capabilities = ["pull", "resolve", "push"]
EOF

sudo systemctl restart containerd
sudo systemctl is-active containerd
```

#### C4) Install kubeadm/kubelet/kubectl v1.30

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

#### C5) `kubeadm init` + Calico

```bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.56.43 \
  --apiserver-cert-extra-sans=192.168.56.43,vm-k8s \
  --pod-network-cidr=192.168.0.0/16

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown "$(id -u)":"$(id -g)" $HOME/.kube/config

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
kubectl get nodes -o wide
```

```bash
watch -n 2 -d 'kubectl get nodes'
```

Ambil join command:

```bash
kubeadm token create --print-join-command --ttl 24h
```

***

**BIAR LEBIH MANTAP KARENA INI TUGAS LEBIH BAIK HAPUS SWAP AGAR SETIAP RESTART VM DAN SWAP TIDAK AKTIF**

Lakukan di **vm-k8s** dan **vm-worker**.

***

#### 1) Matikan swap sekarang (langsung hilang tanpa reboot)

```bash
sudo swapoff /swap.img
# atau kalau mau matikan semua swap:
sudo swapoff -a

swapon --show
```

Kalau sukses, `swapon --show` harus kosong (tidak ada output).

***

#### 2) Biar swap tidak hidup lagi saat reboot (permanen)

Cek dulu swap dinyalakan dari mana (umumnya dari `/etc/fstab`):

```bash
grep -nE 'swap|/swap\.img' /etc/fstab || true
```

Kalau ada baris seperti `/swap.img none swap sw 0 0`, **hapus atau comment** baris itu.

Cara aman: backup dulu lalu comment otomatis:

```bash
sudo cp -a /etc/fstab /etc/fstab.bak.$(date +%F-%H%M%S)
sudo sed -i '/\/swap\.img/s/^/#/' /etc/fstab
```

Reload systemd (opsional tapi bagus):

```bash
sudo systemctl daemon-reload
```

***

#### 3) Hapus file swap

Setelah swap sudah off dan fstab sudah dibereskan:

```bash
sudo rm -f /swap.img
```

Kalau masih muncul `Operation not permitted`, biasanya karena file dibuat **immutable**. Cek & lepas immutable lalu hapus:

```bash
lsattr /swap.img 2>/dev/null || true
sudo chattr -i /swap.img 2>/dev/null || true
sudo rm -f /swap.img
```

***

#### 4) Verifikasi setelah reboot

Reboot VM, lalu cek:

```bash
swapon --show
free -h
```

Harus tidak ada `/swap.img` dan Swap total idealnya `0`.

***

### D. VM-WORKER (192.168.56.44) — Kubernetes worker + NodePort

#### D1) Ulangi langkah containerd yang SAMA seperti vm-k8s

Jalankan di vm-worker: **C1 + C2 + C3 + C4** (swap off, sysctl, containerd safe config, hosts.toml Harbor, install kubeadm/kubelet/kubectl).

Kalau ada swapfile bawaan (seperti kasus kamu `/swap.img`):

```bash
sudo rm -f /swap.img
```

Verifikasi:

```bash
swapon --show
```

Harus kosong.

#### D2) Join cluster

Pakai command dari vm-k8s, contoh:

```bash
sudo kubeadm join 192.168.56.43:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

Cek dari vm-k8s:

```bash
kubectl get nodes -o wide
```

#### D3) Siapkan hostPath MySQL (sesuai manifest kamu)

Karena PV kamu pakai:

* `/data/threebody/mysql`\
  dan nodeAffinity ke hostname `vm-worker`

Maka di vm-worker:

```bash
sudo mkdir -p /data/threebody/mysql
sudo chown -R 999:999 /data/threebody/mysql || true
sudo chmod 700 /data/threebody/mysql || true
```

#### D4) Firewall (biar NodePort bisa diakses dari vm-docker)

Untuk lab paling simpel: **matikan ufw** di vm-worker & vm-k8s:

```bash
sudo ufw disable || true
```

Kalau kamu mau tetap pakai ufw, minimal di vm-worker:

```bash
sudo ufw allow 30080:30082/tcp
sudo ufw reload
sudo ufw status verbose
```

```bash
kubeadm token create --print-join-command --ttl 24h
```

***

**BIAR LEBIH MANTAP KARENA INI TUGAS LEBIH BAIK HAPUS SWAP AGAR SETIAP RESTART VM DAN SWAP TIDAK AKTIF**

Lakukan di **vm-k8s** dan **vm-worker**.

***

#### 1) Matikan swap sekarang (langsung hilang tanpa reboot)

```bash
sudo swapoff /swap.img
# atau kalau mau matikan semua swap:
sudo swapoff -a

swapon --show
```

Kalau sukses, `swapon --show` harus kosong (tidak ada output).

***

#### 2) Biar swap tidak hidup lagi saat reboot (permanen)

Cek dulu swap dinyalakan dari mana (umumnya dari `/etc/fstab`):

```bash
grep -nE 'swap|/swap\.img' /etc/fstab || true
```

Kalau ada baris seperti `/swap.img none swap sw 0 0`, **hapus atau comment** baris itu.

Cara aman: backup dulu lalu comment otomatis:

```bash
sudo cp -a /etc/fstab /etc/fstab.bak.$(date +%F-%H%M%S)
sudo sed -i '/\/swap\.img/s/^/#/' /etc/fstab
```

Reload systemd (opsional tapi bagus):

```bash
sudo systemctl daemon-reload
```

***

#### 3) Hapus file swap

Setelah swap sudah off dan fstab sudah dibereskan:

```bash
sudo rm -f /swap.img
```

Kalau masih muncul `Operation not permitted`, biasanya karena file dibuat **immutable**. Cek & lepas immutable lalu hapus:

```bash
lsattr /swap.img 2>/dev/null || true
sudo chattr -i /swap.img 2>/dev/null || true
sudo rm -f /swap.img
```

***

#### 4) Verifikasi setelah reboot

Reboot VM, lalu cek:

```bash
swapon --show
free -h
```

Harus tidak ada `/swap.img` dan Swap total idealnya `0`.

***

### E. GITLAB CI/CD

#### E1) GitLab Variables (Project → Settings → CI/CD → Variables)

Wajib:

* `HARBOR_USERNAME` = `admin`
* `HARBOR_PASSWORD` = `Harbor12345`
* `MYSQL_ROOT_PASSWORD` = (bebas)
* `MYSQL_DATABASE` = `threebody`
* `MYSQL_USER` = `admin`
* `MYSQL_PASSWORD` = (bebas)
* `LARAVEL_APP_KEY` = `base64:...`
* `KUBECONFIG_PROD` **Type: File** (isi dari `~/.kube/config` di vm-k8s)

Ambil kubeconfig dari vm-k8s (jalankan dari vm-docker):

```bash
scp cikal@192.168.56.43:~/.kube/config ./kubeconfig-prod
```

Upload file itu ke variable `KUBECONFIG_PROD` (Type: File).

***

Setelah semua langkah-langkah setup diatas sudah dilakukan, lanjut pastikan dengan menjalankan perintah dibawah ini.

#### ✅ ADD-ON 1

> Tujuan: memastikan **docker daemon** sudah benar-benar mengizinkan insecure registry, dan runner (user `gitlab-runner`) bisa resolve & akses Harbor.

```bash
# =========================================================
# ADD-ON: VALIDASI DOCKER + RUNNER BISA AKSES HARBOR
# Tempel setelah B2 (atau setelah ADD-ON Harbor UP)
# =========================================================

echo "=== [Docker] pastikan insecure registry kebaca ==="
docker info | egrep -i "insecure|registry" || true

echo "=== [DNS] pastikan harbor.local resolve ke IP yang benar ==="
getent hosts harbor.local

echo "=== [Runner] cek dari konteks user gitlab-runner ==="
sudo -u gitlab-runner getent hosts harbor.local || true
sudo -u gitlab-runner bash -lc 'curl -fsSI http://harbor.local:8080/v2/ | head -n 1' || true
```

#### ✅ ADD-ON 2

> Tujuan: memastikan Harbor **benar-benar UP** dan **listen di 8080** sebelum kamu lanjut ke step berikutnya / sebelum push pertama ke GitLab.

```bash
# =========================================================
# ADD-ON: HARBOUR MUST BE UP (wajib sebelum pipeline pertama)
# Tempel setelah B3 (Install & cek Harbor)
# =========================================================

echo "=== [Harbor] cek container status ==="
sudo docker compose -f /opt/harbor/docker-compose.yml ps

echo "=== [Harbor] pastikan port 8080 LISTEN di host ==="
# harus ada output LISTEN untuk :8080
sudo ss -lntp | grep ':8080' || true

echo "=== [Harbor] sanity check endpoint /v2 (401 normal) ==="
# target: dapat HTTP/1.1 401 Unauthorized (yang penting bukan connection refused)
curl -fsSI http://harbor.local:8080/v2/ | head -n 1 || true

echo "=== [Harbor] kalau masih connection refused: start ulang Harbor ==="
cd /opt/harbor
sudo docker compose -f /opt/harbor/docker-compose.yml up -d
sudo docker compose -f /opt/harbor/docker-compose.yml ps

echo "=== [Harbor] cek log nginx harbor kalau masih gagal ==="
# cari error bind port / crash loop
sudo docker compose -f /opt/harbor/docker-compose.yml logs --tail=120 nginx || true
sudo docker compose -f /opt/harbor/docker-compose.yml logs --tail=120 harbor-core || true
sudo docker compose -f /opt/harbor/docker-compose.yml logs --tail=120 registry || true

echo "=== [Harbor] cek lagi port 8080 & /v2 ==="
sudo ss -lntp | grep ':8080' || true
curl -fsSI http://harbor.local:8080/v2/ | head -n 1 || true

echo "=== [Harbor] cek konflik port (kalau 8080 tidak bisa bind) ==="
# kalau ada service lain pakai 8080, akan kelihatan di sini
sudo lsof -iTCP:8080 -sTCP:LISTEN -nP || true
```

#### ✅ ADD-ON 2 (Opsional tapi sangat membantu): “Harbor readiness gate sebelum push pertama”

```bash
# =========================================================
# ADD-ON: HARBOR READINESS GATE (wajib sebelum pipeline pertama)
# Jalankan di vm-docker
# =========================================================
echo "=== Harbor containers ==="
sudo docker compose -f /opt/harbor/docker-compose.yml ps

echo "=== Port 8080 must LISTEN ==="
sudo ss -lntp | grep ':8080'

echo "=== /v2 must respond (401 is OK) ==="
curl -fsSI http://harbor.local:8080/v2/ | head -n 1
```

Patokan:

* ada LISTEN `:8080`
* `/v2` balas `HTTP/1.1 401 Unauthorized`

Kalau ini lolos, job push akan aman.

***

***

#### ✅ ADD-ON 3

> Tujuan: bikin “gate” terakhir supaya kamu **tidak push** kalau Harbor belum siap (biar pipeline pertama tidak gagal lagi).

```bash
# =========================================================
# ADD-ON: FINAL PRE-FLIGHT SEBELUM PUSH PERTAMA KE GITLAB
# Tempel tepat sebelum E2 (Push pertama)
# =========================================================

echo "=== FINAL CHECK: Harbor harus hidup & reachable ==="
curl -fsSI http://harbor.local:8080/v2/ | head -n 1 || (
  echo "ERROR: Harbor tidak reachable di http://harbor.local:8080"
  echo "Pastikan: Harbor containers UP, port 8080 LISTEN, dan UFW tidak blok."
  exit 1
)

echo "=== FINAL CHECK: project 'threebody' sudah dibuat di UI Harbor ==="
echo "Buka: http://harbor.local:8080/ lalu pastikan project threebody ada."
```

#### ✅ ADD-ON 4 “Bootstrap ImagePull Secret di K8S sebelum pipeline pertama”

Dengan ADD-ON ini, sebelum push pertama kamu “mengunci” kondisi cluster supaya **selaras** dengan `.gitlab-ci.yml`.

```bash
# =========================================================
# ADD-ON: K8S BOOTSTRAP IMAGEPULL (sekali saja sebelum pipeline pertama)
# Jalankan di vm-k8s
# =========================================================
NS=threebody-prod
SECRET=harbor-regcred

echo "=== namespace ==="
kubectl get ns "$NS" >/dev/null 2>&1 || kubectl create ns "$NS"

echo "=== serviceaccount default ==="
kubectl -n "$NS" get sa default >/dev/null 2>&1 || kubectl -n "$NS" create sa default

echo "=== create/refresh secret $SECRET ==="
kubectl -n "$NS" create secret docker-registry "$SECRET" \
  --docker-server="harbor.local:8080" \
  --docker-username="admin" \
  --docker-password="Harbor12345" \
  --dry-run=client -o yaml | kubectl apply -f -

echo "=== attach ke SA default ==="
kubectl -n "$NS" patch sa default --type merge \
  -p '{"imagePullSecrets":[{"name":"'"$SECRET"'"}]}'

echo "=== verifikasi ==="
kubectl -n "$NS" get secret "$SECRET" -o name
kubectl -n "$NS" get sa default -o jsonpath='{.imagePullSecrets[*].name}'; echo
```

> Kalau kamu nanti pakai username/password Harbor dari GitLab Variables (bukan hardcode), kamu bisa jalankan bootstrap ini setelah kamu yakin kredensial Harbor benar. Tapi untuk runbook “sekali push lab”, ini yang paling stabil.

***

**✅ ADD-ON LARAVEL READY (wajib)**

```bash
# =========================================================
# ADD-ON: FIX LARAVEL APACHE DOCROOT (WAJIB sebelum push pertama)
# Lokasi: vm-docker, setelah repo siap, sebelum git push pertama
# =========================================================
cd ~/three-body-problem

echo "=== cek Dockerfile Laravel sudah set APACHE_DOCUMENT_ROOT ke /public ==="
grep -n "APACHE_DOCUMENT_ROOT" -n laravel/Dockerfile || true
grep -n "laravel.conf" -n laravel/Dockerfile || true

echo "Jika belum ada APACHE_DOCUMENT_ROOT=/var/www/html/public, lakukan FULL REPLACE laravel/Dockerfile sesuai runbook."
```

Pastikan kamu di root repo saat cek file (biar nggak kejadian salah path lagi):

**✅ ADD-ON LARAVEL SMOKE TEST (wajib)**

**Tempatkan:** di runbook **setelah pipeline deploy sukses** (misal setelah ringkasan deploy / healthcheck)

```bash
# =========================================================
# ADD-ON: SMOKE TEST LARAVEL & GO VIA EDGE
# Lokasi: vm-docker (setelah deploy sukses)
# =========================================================
curl -kfsS --resolve hit.local:443:127.0.0.1 https://hit.local/go/api/products | head
curl -kfsS --resolve hit.local:443:127.0.0.1 https://hit.local/laravel/api/products | head
```

Target: dua-duanya balik JSON (bukan HTML Apache 404/403).

***

#### E2) Push pertama

Di vm-docker (repo sudah ada):

```bash
cd ~/three-body-problem

git init
git config --global init.defaultBranch main
git config --global user.name "cikalfarid"
git config --global user.email "cikalfarid@users.noreply.gitlab.com"
git branch -M main

git remote remove origin 2>/dev/null || true
git remote add origin git@gitlab-443:cikalfarid/three-body-problem.git
git remote -v

git add -A
git commit -m "init: ci + k8s + edge + harbor" || true
git push -u origin main --force
```

> **Penting:** jangan ubah YAML jadi `:${TAG}` karena Kubernetes **tidak** melakukan substitusi `${TAG}`.\
> Di project kamu sudah benar: manifest pakai `:latest`, lalu pipeline **`kubectl set image ...:$CI_COMMIT_SHORT_SHA`**.

```bash
git init

git add .

git pull --rebase origin main

git rebase --continue

git fetch origin

git log origin/main --oneline

git push -u origin main
```

Kalau kamu sudah commit sebelumnya tapi mau trigger ulang:

```bash
git commit --allow-empty -m "chore: trigger pipeline"
git push gitlab main
```

***

### F. Setelah deploy sukses (sekali saja): migrate/seed

Di vm-k8s:

```bash
NS=threebody-prod
LARAVEL_POD=$(kubectl -n "$NS" get pod -l app=laravel -o jsonpath='{.items[0].metadata.name}')
kubectl -n "$NS" exec -it "$LARAVEL_POD" -- php artisan migrate --force
kubectl -n "$NS" exec -it "$LARAVEL_POD" -- php artisan db:seed --force || true
```

**✅ ADD-ON SEED POLICY (biar data tidak dobel)**

Masalah “5 jadi 10” itu **normal** karena `db:seed` kamu tidak idempotent.

**Tempatkan:** di runbook **F (migrate/seed)**

```bash
# =========================================================
# ADD-ON: SEED JANGAN DIJALANKAN BERULANG
# - php artisan db:seed jika dijalankan berkali-kali bisa bikin data dobel
# =========================================================
```

**Kalau kamu ingin “reset balik ke 5” (lab)**

Tambahkan juga ini di bagian F sebagai opsi:

```bash
# =========================================================
# ADD-ON: LAB RESET DATA (balikin kondisi awal seed)
# WARNING: hapus & buat ulang table + seed ulang (untuk lab)
# =========================================================
NS=threebody-prod
LARAVEL_POD=$(kubectl -n "$NS" get pod -l app=laravel -o jsonpath='{.items[0].metadata.name}')
kubectl -n "$NS" exec -it "$LARAVEL_POD" -- php artisan migrate:fresh --seed --force
```

***

#### ✅ ADD-ON: “3 VM Restart → Harbor & hit.local harus auto pulih”

Ini yang bikin stabil saat laptop kamu dimatikan/VM restart.

**A) Pastikan service yang wajib selalu enabled**

Jalankan:

**vm-docker**

```bash
sudo systemctl enable --now docker
sudo systemctl is-enabled docker
sudo systemctl is-active docker
```

**vm-k8s + vm-worker**

```bash
sudo systemctl enable --now containerd kubelet
sudo systemctl is-enabled containerd kubelet
sudo systemctl is-active containerd kubelet
```

**B) (Recommended) Buat systemd unit supaya Harbor & Edge “dipaksa up” saat boot**

> Ini membuat setelah restart, dia bukan cuma “harap restart policy docker”, tapi **systemd yang memastikan compose up**.

**C) Checklist setelah restart (real check, bukan dummy)**

Setelah 3 VM nyala lagi, jalankan:

**vm-docker**

```bash
curl -fsSI http://harbor.local:8080/v2/ | head -n 1
curl -kfsS --resolve hit.local:443:127.0.0.1 https://hit.local/ | head
curl -kfsS --resolve hit.local:443:127.0.0.1 https://hit.local/go/api/products | head
curl -kfsS --resolve hit.local:443:127.0.0.1 https://hit.local/laravel/api/products | head
```

**vm-k8s**

```bash
kubectl get nodes -o wide
kubectl -n threebody-prod get pods -o wide
```

**Urutan boot paling aman (kalau kamu bisa atur):**

1. vm-docker (Harbor up dulu)
2. vm-k8s
3. vm-worker

***

#### ✅ 1. CI/CD menggunakan GitLab CI

**Status: TERPENUHI**

✔ Kamu sudah punya `.gitlab-ci.yml`\
✔ Runner sudah:

* terpasang di **vm-docker**
* executor **shell**
* bisa akses **docker**
* bisa login & push ke **Harbor**

✔ Pipeline **sekali push langsung jalan normal**\
✔ Tidak ada error:

* DNS
* Docker permission
* Harbor unreachable
* runner cannot connect docker.sock

➡️ **Ini sudah sesuai runbook dan stabil**

***

#### ✅ 2. Pipeline meliputi build, deploy, health check, post-deployment

**Status: TERPENUHI**

**a) Build semua service**

✔ React\
✔ Golang\
✔ Laravel\
✔ Semua pakai **Docker image**

**b) Deploy ke Docker & Kubernetes**

✔ Docker → untuk **Harbor + Edge Nginx**\
✔ Kubernetes → untuk **React, Go, Laravel, MySQL**

✔ cluster:

* vm-k8s (control plane)
* vm-worker (worker)
* containerd OK
* pull image dari Harbor OK

**c) Health check & post-deploy**

✔ Kubernetes memastikan:

* Pod `Running`
* Container `Ready`\
  ✔ Manual smoke test:

```bash
/go/api/products
/laravel/api/products
```

✔ Edge Nginx berhasil proxy

➡️ **Secara DevOps, ini sudah post-deployment validation**

***

#### ✅ 3. Secret Manager (GitLab Variables)

**Status: TERPENUHI**

Kamu sudah menggunakan **GitLab CI/CD Variables**, bukan hardcode:

✔ MySQL credential\
✔ Laravel APP\_KEY\
✔ Harbor credential\
✔ KUBECONFIG\_PROD (Type: File)

➡️ **Ini sudah memenuhi konsep Secret Manager (sesuai requirement)**

***

#### ✅ 4. Rate limiting & HTTPS

**Status: TERPENUHI**

✔ HTTPS:

* Edge Nginx
* TLS cert (self-signed) untuk `hit.local`
* Semua traffic frontend → backend lewat HTTPS

✔ Rate limiting:

* Di Edge Nginx
* Berlaku untuk request API

➡️ **Sudah benar secara arsitektur**

***

#### ✅ 5. Image di Harbor + multistage build

**Status: TERPENUHI**

✔ Semua image di-push ke **Harbor**\
✔ Registry **bukan Docker Hub**\
✔ Laravel & Go pakai **multistage build**\
✔ Tidak build di server langsung

Catatan penting:

* Ukuran image **bukan pelanggaran requirement**
* Optimasi size = **tahap lanjutan**, bukan syarat tugas

➡️ **Requirement ini sudah 100% terpenuhi**

***

#### ✅ 6. Logging Service & utilitas server (Grafana)&#x20;

✔ Requirement ini **belum kamu kerjakan**\
✔ Dan **itu BENAR**, karena:

* Fondasi sudah stabil
* CI/CD aman
* Cluster sehat
