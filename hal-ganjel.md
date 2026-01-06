# Penjelasan runbook

Aku sudah cek dari **runbook + isi `.gitlab-ci.yml` yang kamu paste**, plus **isi project ZIP yang kamu kasih** (di ZIP itu `frontend/src/App.js` masih pakai URL localhost hardcode). Ini jawaban detail untuk 3 pertanyaanmu + audit step yang redundant.

***

### 1) `deploy/k8s/base/00-namespace.yaml` dipakai atau tidak?

**Dipakai, tapi fungsinya redundant (tidak wajib) dalam alur pipeline kamu sekarang.**

Kenapa aku bilang “dipakai”?\
Karena di job `deploy` ada step ini:

```bash
kubectl apply -f deploy/k8s/base/
```

Kalau file `00-namespace.yaml` ada di folder itu, maka **kubectl akan ikut apply file tersebut**.

Tapi… kamu juga sudah punya step ini sebelum apply:

```bash
kubectl get ns "$K8S_NS" >/dev/null 2>&1 || kubectl create ns "$K8S_NS"
```

Artinya **namespace sudah dipastikan ada** sebelum semua manifest di-apply. Jadi `00-namespace.yaml` tidak memberikan nilai tambah dalam pipeline kamu saat ini.

#### Rekomendasi paling aman & rapi

Pilih **salah satu** (jangan dua-duanya):

**Opsi A (paling rapi menurutku):**

* **Hapus `00-namespace.yaml`**
* Tetap pakai step pipeline `kubectl create ns "$K8S_NS"`

Keuntungannya:

* Nama namespace cuma “diatur” dari **1 sumber**: variable `K8S_NS` di `.gitlab-ci.yml`.
* Menghindari risiko mismatch kalau suatu hari kamu ganti `K8S_NS` tapi lupa update YAML.

**Opsi B (kalau kamu mau semua infra “pure manifest”):**

* **Pertahankan `00-namespace.yaml`**
* Hapus step pipeline `kubectl create ns ...`
* Tapi pastikan apply namespace “lebih dulu” supaya tidak race (idealnya apply namespace file dulu, baru apply sisanya), misalnya:
  * `kubectl apply -f deploy/k8s/base/00-namespace.yaml`
  * `kubectl apply -f deploy/k8s/base/`

Karena `kubectl apply -f folder/` tidak selalu “dijamin” urutan file-nya aman di semua situasi.

✅ Jadi jawaban final: **`00-namespace.yaml` terpakai karena ikut ke-apply**, tapi **tidak diperlukan** karena pipeline sudah create namespace lebih dulu.

***

### 2) `kubeconfig-prod` terpakai atau tidak? (scp dari vm-docker)

Di sini ada 2 hal yang sering ketukar:

#### A. File lokal `./kubeconfig-prod` di vm-docker

File ini kamu buat dari:

```bash
scp cikal@192.168.56.43:~/.kube/config ./kubeconfig-prod
```

File **lokal** ini **tidak dipakai oleh pipeline** secara langsung.

Dia cuma dipakai untuk **1 tujuan**: **bahan upload** ke GitLab Variable `KUBECONFIG_PROD` (Type: File).

Setelah kamu upload ke GitLab, file lokal `kubeconfig-prod` itu:

* boleh kamu simpan (buat backup/debug),
* atau **boleh kamu hapus**,
* yang penting **jangan pernah kamu commit ke repo**.

#### B. GitLab Variable `KUBECONFIG_PROD` (Type: File)

Nah ini yang **dipakai pipeline**.

Di job `deploy` dan `monitoring_install` kamu ada:

```bash
: "${KUBECONFIG_PROD:?Missing KUBECONFIG_PROD (GitLab Variables Type: File)}"
export KUBECONFIG="$KUBECONFIG_PROD"
```

Di GitLab, **File variable** itu ketika job jalan akan dibuat jadi file temporary, lalu env var-nya berisi path file itu. Jadi `kubectl` baca kubeconfig dari situ.

✅ Jadi jawaban final:

* **Yang dipakai pipeline = `KUBECONFIG_PROD` (GitLab File Variable)**
* **File `kubeconfig-prod` hasil scp = cuma “alat” untuk upload sekali** (setelah upload, tidak wajib ada lagi)

**Kapan perlu scp ulang?**

* Kalau kamu **re-init cluster kubeadm** (admin.conf berubah)
* Kalau cert/token berubah (di lab biasanya lama banget baru perlu)

***

### 3) Perlu ubah `frontend/src/App.js` atau tidak?

**Iya, perlu (minimal harus ganti endpoint-nya).**

Aku cek isi `frontend/src/App.js` dari ZIP yang kamu kirim, sekarang masih begini:

```js
const LARAVEL_API = 'http://127.0.0.1:8001/api/products';
const GO_API = 'http://localhost:8080/api/products';
```

Kalau frontend kamu jalan di browser dan kamu akses via:

* `https://hit.local`

maka:

* `127.0.0.1` dan `localhost` itu mengarah ke **laptop kamu sendiri** (client), bukan ke vm-worker / cluster.
* Itu pasti bikin fetch salah arah / CORS / tidak nyambung.

#### Minimal perubahan yang cukup (tanpa rewrite besar)

Kamu cukup ubah bagian konstanta API jadi relative base path (sesuai edge nginx config: `/go` dan `/laravel`).

Contoh patch minimal:

```js
const GO_API_BASE = (process.env.REACT_APP_GO_API_BASE || "/go").replace(/\/$/, "");
const LARAVEL_API_BASE = (process.env.REACT_APP_LARAVEL_API_BASE || "/laravel").replace(/\/$/, "");

const GO_API = `${GO_API_BASE}/api/products`;
const LARAVEL_API = `${LARAVEL_API_BASE}/api/products`;
```

Sisanya boleh tetap.

✅ Jadi jawaban final: **perlu ubah `App.js`** (minimal endpoint), dan versi App.js yang kamu tulis di runbook itu sudah “benar arah” untuk target arsitektur.

> Catatan penting kecil: kamu bikin `frontend/.env.production`, tapi kamu juga bikin `frontend/.dockerignore` yang meng-ignore `.env.*`.\
> Artinya **`.env.production` tidak ikut masuk konteks docker build**, jadi untuk pipeline kamu, file itu sebenarnya **tidak kepakai** (yang kepakai adalah `--build-arg ...` di docker build).\
> File `.env.production` masih berguna kalau kamu build React **langsung di lokal** tanpa Docker.

***

## Audit runbook: step yang bener-bener bisa dianggap “tidak perlu / redundant”

Ini yang paling aman untuk dipangkas karena **double** / **nggak kepakai** / **berpotensi bikin bingung**:

### A. Redundant yang aman dihapus

1. **`00-namespace.yaml`**\
   Redundant karena pipeline sudah create namespace. (lihat poin #1)
2. **Blok “hapus swap biar mantap” yang muncul 2x**\
   Kamu sudah swapoff + edit fstab sebelumnya. Cukup **1 versi** aja.
3. **`kubeadm token create --print-join-command` yang muncul lagi di bagian worker**\
   Ini harusnya **cukup di vm-k8s** (control-plane). Di worker, kamu cuma pakai join command-nya.
4. **Bagian git command ganda (dua blok `git init ...` dan rebase dll)**\
   Untuk push pertama, cukup 1 blok yang clean.\
   Blok yang ada `git rebase --continue` itu cuma kepake kalau memang terjadi konflik/rebase.
5. **`deploy/monitoring/expose-ui-nodeport.yaml`**\
   Di pipeline `monitoring_install`, kamu **tidak pernah `kubectl apply` file ini**.\
   Pipeline kamu pakai `kubectl patch svc ... nodePort ...`.\
   Jadi file ini **opsional** (keep kalau kamu mau cara manual apply, tapi kalau mau minimal, hapus saja).
6.  **Gate monitoring yang pakai port 3000/9090/3100 di vm-k8s**\
    Ini tidak match dengan implementasi final kamu yang expose NodePort:

    * Grafana: `30030`
    * Prometheus: `30090`
    * Loki gateway: `30100`

    Jadi “gate” itu kalau dijalankan bisa bikin bingung (bukan step wajib).

### B. Redundant “tergantung kebutuhan” (opsional)

1. **UFW enable + allow port**\
   Untuk lab internal host-only network, UFW itu tidak wajib (kalau kamu mau simple).\
   Tapi kalau kamu mau “hardening”, keep — cuma pastikan NodePort dan 8080/443 kebuka.
2. **SSH altssh GitLab port 443**\
   Wajib hanya kalau port 22 kamu keblok. Kalau port 22 aman, ini bisa di-skip.
3. **Systemd unit untuk harbor-compose & edge-compose**\
   Docker restart policy biasanya sudah cukup, tapi unit systemd bikin lebih deterministik.\
   Bukan wajib, tapi recommended.

### C. Optimasi pipeline (ini benar-benar ngurangin step)

Di pipeline kamu sekarang:

* `build_check` build image
* `push_images` build lagi (ulang) lalu push

Kalau runner kamu **cuma 1 mesin (vm-docker)** dan executor `shell`, kamu bisa bikin `push_images` **tidak perlu rebuild**, cukup login lalu `docker push` image yang sudah dibuild di `build_check`.

Ini bukan wajib, tapi ini _paling besar_ memangkas waktu dan step.

***

Kalau kamu mau, aku bisa tuliskan **versi runbook yang sudah “di-trim”** (tetap aman, tapi tanpa step duplikat dan file yang tidak kepakai), tapi dari hasil audit di atas kamu sudah bisa langsung potong bagian-bagian yang redundant tanpa mengubah arsitektur finalnya.
