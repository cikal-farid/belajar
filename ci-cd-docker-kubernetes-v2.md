# CI/CD Docker, Kubernetes V2

Berikut runbook yang **lebih lengkap, step-by-step, dan “anti-rancu”** untuk ngejalanin pipeline yang kamu tulis (build→push Harbor→deploy dev compose→deploy staging K8s→smoke+migrate→manual promote→deploy prod K8s→smoke+migrate). Aku buat urut dari **prasyarat**, **setup VM1/VM2**, **variables GitLab**, sampai **cara verifikasi & troubleshooting**.

***

### Gambaran arsitektur (biar nyambung)

* **VM1** = Kubernetes control-plane (cluster K8s kamu: ingress-nginx + MetalLB + storage class)
* **VM2** = GitLab Runner (executor **shell**) + Docker + kubectl\
  Semua job CI jalan **di VM2**, lalu VM2 menembak API K8s di VM1 pakai kubeconfig.

Pipeline flow (main branch):

1. Build image (laravel/goapi/frontend) → push ke Harbor dengan tag `${CI_COMMIT_SHA}`
2. Deploy dev via Docker Compose di VM2
3. Deploy staging ke K8s (VM1) → smoke+migrate
4. Manual approve “promote” → mengunci tag yang sama
5. Deploy prod ke K8s → smoke+migrate

***

## A. Prasyarat wajib sebelum ngomongin GitLab CI

### A1) VM1 (Kubernetes) harus punya 3 komponen ini

Jalankan di **VM1**:

#### 1) Ingress NGINX ada

```bash
kubectl get ns ingress-nginx
kubectl -n ingress-nginx get deploy,svc
```

#### 2) MetalLB / LoadBalancer IP siap

Pastikan service ingress dapat **EXTERNAL-IP**:

```bash
kubectl -n ingress-nginx get svc ingress-nginx-controller
```

* Kalau kolom `EXTERNAL-IP` masih `<pending>` terus, job CI kamu bakal stop di loop “wait ingress ip”.

#### 3) StorageClass default ada (ini sering jadi sumber “Pending PVC”)

```bash
kubectl get storageclass
```

Harus ada minimal 1 StorageClass, dan idealnya ada yang default (ada annotation `is-default-class: "true"`).

Kalau belum default, set salah satu jadi default:

```bash
kubectl patch storageclass <NAMA_SC> -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

> Kenapa penting? Karena YAML kamu bikin PVC untuk MySQL. Kalau gak ada provisioner/storageclass default, MySQL StatefulSet akan **Pending** selamanya.

***

### A2) VM2 (Runner) wajib bisa: docker + kubectl + akses ke VM1:6443

Di **VM2**:

#### 1) Pastikan docker jalan

```bash
docker version
docker compose version
```

#### 2) Pastikan VM2 bisa reach API server VM1

Ganti `VM1_IP` sesuai IP VM1:

```bash
VM1_IP=<IP_VM1>
nc -vz $VM1_IP 6443
```

Harus “succeeded”.

***

## B. Soal Runner tag “deploy” (biar job gak Pending)

Kalau `.gitlab-ci.yml` kamu pakai:

```yaml
default:
  tags: ["deploy"]
```

Maka di GitLab UI:

* Project → **Settings → CI/CD → Runners**
* Pilih runner `devops-ci`
* Pastikan **Tags** berisi `deploy`\
  **atau** centang “Run untagged jobs” (kalau kamu hapus tags dari YAML).

**Rule praktis:**

* Kalau kamu cuma punya 1 runner (VM2) → paling aman: **hapus tags** dari default.
* Kalau kamu punya banyak runner dan mau “ngunci” hanya VM2 yang boleh deploy → **biarkan tags**, tapi pastikan runner punya tag `deploy`.

***

## C. Install kubectl di VM2 (runner shell)

Di **VM2** jalankan:

```bash
which kubectl || echo "kubectl belum ada"
kubectl version --client || true
```

Kalau belum ada, install:

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubectl
```

Test sebagai user runner:

```bash
sudo -u gitlab-runner kubectl version --client
```

***

## D. GitLab CI/CD Variables yang harus kamu isi (lengkap + cara buat)

Masuk GitLab:\
**Project → Settings → CI/CD → Variables → Add variable**

> Saran checkbox:

* **Masked**: untuk password (kalau GitLab menolak karena format, minimal gunakan **Protected**)
* **Protected**: wajib untuk kubeconfig & prod secrets (biar cuma jalan di protected branch/tag)
* **Environment scope**: opsional (kalau mau rapi, tapi untuk awal gak wajib)

### D1) Variables Harbor (wajib)

| Variable          | Contoh nilai                           | Catatan                                           |
| ----------------- | -------------------------------------- | ------------------------------------------------- |
| `HARBOR_URL`      | `harbor.domain.local` atau `10.0.0.10` | tanpa `https://` juga boleh, tapi harus konsisten |
| `HARBOR_PROJECT`  | `threebody`                            | ini nama project di Harbor                        |
| `HARBOR_USERNAME` | `admin` / `robot$ci`                   | lebih aman pakai robot account                    |
| `HARBOR_PASSWORD` | `********`                             | password / token robot                            |

✅ Test cepat di VM2 (manual):

```bash
echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin
```

***

### D2) Variables DEV (Docker Compose) (wajib kalau deploy\_dev dipakai)

| Variable                  | Contoh            | Catatan                                    |
| ------------------------- | ----------------- | ------------------------------------------ |
| `DEV_MYSQL_ROOT_PASSWORD` | `root_dev_...`    | boleh berubah karena dev boleh wipe volume |
| `DEV_MYSQL_PASSWORD`      | `user_dev_...`    | password user mysql `threebody`            |
| `DEV_APP_KEY`             | `base64:xxxxxxxx` | format Laravel harus `base64:...`          |

#### Cara bikin `DEV_APP_KEY` (cepat)

Di mesin mana saja yang ada `php`:

```bash
php -r "echo 'base64:'.base64_encode(random_bytes(32)).PHP_EOL;"
```

***

### D3) Kubeconfig untuk akses K8s (wajib)

Pipeline kamu pakai **base64** kubeconfig: `KUBECONFIG_B64`.

#### Step 1 — cek dulu “server:” di kubeconfig (INI PENTING)

Di **VM1**:

```bash
sudo grep -n "server:" /etc/kubernetes/admin.conf
```

Kalau hasilnya:

* ✅ `server: https://<IP_VM1>:6443` → aman
* ❌ `server: https://127.0.0.1:6443` → **VM2 gak bisa pakai**, karena 127.0.0.1 itu VM2 sendiri

Kalau masih 127.0.0.1, buat copy yang khusus untuk VM2:

```bash
VM1_IP=<IP_VM1>
sudo cp /etc/kubernetes/admin.conf /tmp/admin-vm2.conf
sudo sed -i "s#server: https://127.0.0.1:6443#server: https://${VM1_IP}:6443#g" /tmp/admin-vm2.conf
sudo grep -n "server:" /tmp/admin-vm2.conf
```

#### Step 2 — encode ke base64 (tanpa newline)

Pakai file yang benar:

* Kalau server sudah IP VM1: pakai `/etc/kubernetes/admin.conf`
* Kalau sebelumnya 127.0.0.1: pakai `/tmp/admin-vm2.conf`

```bash
sudo base64 -w0 /etc/kubernetes/admin.conf
# atau:
sudo base64 -w0 /tmp/admin-vm2.conf
```

Copy outputnya → buat variable GitLab:

* `KUBECONFIG_B64` = (paste base64 panjang itu)

✅ Test di VM2 (manual, sebelum CI):

```bash
echo "$KUBECONFIG_B64" | base64 -d > /tmp/kubeconfig
KUBECONFIG=/tmp/kubeconfig kubectl get nodes
```

***

### D4) Secrets STAGING (wajib)

| Variable                      | Contoh           | Catatan penting                               |
| ----------------------------- | ---------------- | --------------------------------------------- |
| `STAGING_MYSQL_ROOT_PASSWORD` | `root_stage_...` | **jangan sering diubah** setelah PV terbentuk |
| `STAGING_MYSQL_PASSWORD`      | `user_stage_...` | password user mysql `threebody`               |
| `STAGING_APP_KEY`             | `base64:xxxx`    | sama seperti dev                              |

***

### D5) Secrets PROD (wajib, jangan berubah-ubah)

| Variable                   | Contoh          | Catatan penting                                                 |
| -------------------------- | --------------- | --------------------------------------------------------------- |
| `PROD_MYSQL_ROOT_PASSWORD` | `root_prod_...` | **jangan diubah** sembarangan (PV akan “ngunci” password lama)  |
| `PROD_MYSQL_PASSWORD`      | `user_prod_...` | password user mysql `threebody`                                 |
| `PROD_APP_KEY`             | `base64:xxxx`   | **harus stabil** (kalau berubah, Laravel session/encrypt rusak) |

> Catatan penting MySQL StatefulSet:\
> Sekali MySQL sudah bikin data di volume, env `MYSQL_ROOT_PASSWORD` **tidak otomatis mengganti** root password.\
> Kalau kamu ubah variable root password, readiness probe bisa fail dan MySQL kelihatan “rusak”.\
> Untuk staging bisa “reset” dengan delete PVC; untuk prod jangan kecuali kamu memang mau reset total.

***

### D6) Host (opsional)

Kalau kamu mau ubah host domain:

| Variable       | Default         | Contoh                    |
| -------------- | --------------- | ------------------------- |
| `STAGING_HOST` | `staging.local` | `staging.threebody.local` |
| `PROD_HOST`    | `prod.local`    | `threebody.local`         |

***

## E. Checklist file `.gitlab-ci.yml` (yang paling sering bikin error)

Pastikan di pipeline kamu:

1. Runner executor **shell** → berarti semua command harus available di VM2
2. `kubectl` sudah ada (bagian C)
3. `KUBECONFIG_B64` benar (bagian D3)
4. K8s punya ingress + metallb + storageclass (bagian A1)

***

## F. Cara menjalankan & verifikasi hasil deploy

### F1) Jalankan pipeline

* Push commit ke branch `main`
* Pipeline otomatis jalan:
  * `build_*` → push image ke Harbor
  * `deploy_dev` → docker compose (VM2)
  * `deploy_staging` → apply manifest ke K8s
  * `smoke_staging` → migrate + curl
* Setelah itu muncul tombol manual: **promote\_to\_prod**
* Klik promote → lanjut deploy prod + smoke prod

***

### F2) Cara akses staging/prod dari laptop kamu (biar bisa buka di browser)

CI smoke test kamu akses pakai:

* `http://<INGRESS_IP>/` + header `Host: staging.local` / `prod.local`

Supaya browser juga bisa, di laptop (atau VM2) tambahkan `/etc/hosts`:

```bash
<INGRESS_IP> staging.local
<INGRESS_IP> prod.local
```

Lalu buka:

* `http://staging.local/`
* `http://prod.local/`

***

## G. Troubleshooting cepat (yang paling sering kejadian)

### 1) Job “Pending” terus

Penyebab 90%:

* job minta tag `deploy` tapi runner gak punya tag itu

Solusi:

* Tambahkan tag `deploy` di runner **atau**
* Hapus `default: tags: ["deploy"]`

***

### 2) `kubectl: command not found`

Solusi:

* Install kubectl di VM2 (bagian C)

***

### 3) `The connection to the server 127.0.0.1:6443 was refused`

Artinya kubeconfig masih menunjuk 127.0.0.1.\
Solusi:

* Ikuti langkah D3 (buat `/tmp/admin-vm2.conf` dengan IP VM1)

***

### 4) Ingress EXTERNAL-IP kosong / pending

Solusi di VM1:

* cek MetalLB jalan + ada pool IP + service type LoadBalancer dapet IP

CI kamu memang akan fail kalau IP kosong (by design).

***

### 5) MySQL StatefulSet Pending (PVC pending)

Solusi:

* Pastikan ada StorageClass default (bagian A1-3)

***

### 6) ImagePullBackOff di pod laravel/goapi/frontend

Biasanya karena secret registry salah / belum terpasang.

* Pastikan `HARBOR_URL/USERNAME/PASSWORD` benar
* Di namespace staging/prod cek:

```bash
kubectl -n threebody-staging get secret harbor-creds
kubectl -n threebody-staging describe pod <pod>
```

***

Kalau kamu mau, aku bisa lanjutkan runbook ini dengan **runbook khusus VM1** (setup Kubernetes yang “pasti jadi” untuk kebutuhan kamu: container runtime, kubeadm init, CNI, MetalLB pool, ingress-nginx, storage provisioner), tapi untuk pertanyaan kamu sekarang, bagian yang paling bikin rancu (variables + kubeconfig + dependency K8s) sudah aku urutkan sampai bisa langsung kamu eksekusi tanpa nebak-nebak.
