# reboot vm

Siap—aku **ingat 4 VM kamu** dan kita tetap pakai rancangan yang sama:

* **vm-docker** = GitLab Runner + build/push image + **edge Nginx (HTTPS + rate-limit)** + **staging (docker compose)**
* **vm-harbor** = Harbor registry (`harbor.local`)
* **vm-k8s** = Kubernetes **control-plane** (API server)
* **vm-worker** = Kubernetes **worker node** (tempat pod prod jalan)

Di arsitektur ini, urutan **mati / nyala** yang paling aman itu mengikuti **dependency**: yang “dipakai orang lain” dinyalakan dulu, dimatikan paling akhir.

***

### 1) Urutan paling aman

#### A. Kalau mau MATIKAN semua VM (shutdown)

**Urutan:**

1. **vm-docker** (karena ini “client/consumer”: edge & staging bergantung ke k8s/harbor)
2. **vm-worker** (worker k8s)
3. **vm-k8s** (control-plane k8s)
4. **vm-harbor** (registry—paling akhir)

> Kenapa Harbor terakhir? Karena saat shutdown/naik lagi, k8s/staging bisa butuh pull image dari Harbor.

***

#### B. Kalau mau HIDUPKAN semua VM (boot)

**Urutan:**

1. **vm-harbor**
2. **vm-k8s**
3. **vm-worker**
4. **vm-docker**

> Ini penting supaya:
>
> * K8s bisa “sehat” dulu (API up)
> * Worker join normal
> * Edge/staging di vm-docker tidak jadi “nembak backend yang belum ada”
> * Pipeline aman karena Harbor + K8s sudah siap

***

### 2) Checklist MATIKAN (copy-paste)

#### Step 0 — Pastikan kamu nggak lagi running pipeline

Di **vm-docker**:

```bash
sudo systemctl stop gitlab-runner
sudo systemctl status gitlab-runner --no-pager -l
```

#### Step 1 — Stop edge + staging (vm-docker)

```bash
sudo systemctl stop threebody-staging threebody-edge
sudo systemctl status threebody-staging threebody-edge --no-pager -l
docker ps --format 'table {{.Names}}\t{{.Status}}' | egrep 'edge-nginx|stg-' || true
```

Lalu shutdown vm-docker:

```bash
sudo shutdown -h now
```

#### Step 2 — Shutdown worker (vm-worker)

Di **vm-worker**:

```bash
sudo systemctl status kubelet --no-pager -l
sudo shutdown -h now
```

#### Step 3 — Shutdown control-plane (vm-k8s)

Di **vm-k8s**:

```bash
sudo systemctl status kubelet --no-pager -l
sudo shutdown -h now
```

#### Step 4 — Shutdown harbor (vm-harbor)

Di **vm-harbor**:

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}' | egrep 'harbor|registry|nginx' || true
sudo shutdown -h now
```

***

### 3) Checklist HIDUPKAN (boot) + validasi per VM

#### Step 1 — Nyalakan vm-harbor, lalu cek

Di **vm-harbor**:

```bash
sudo systemctl status docker --no-pager -l
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}' | egrep 'harbor|registry|nginx' || true
curl -kI https://harbor.local/ | head -n 5
```

Kalau `docker ps` belum muncul “healthy”, tunggu 1-2 menit lalu cek lagi.

***

#### Step 2 — Nyalakan vm-k8s, lalu cek control-plane siap

Di **vm-k8s**:

```bash
sudo systemctl status kubelet --no-pager -l

kubectl get nodes -o wide
kubectl -n kube-system get pods -o wide
```

Target minimal:

* `vm-k8s` status **Ready**
* Pod core (coredns/calico/kube-proxy) **Running** (boleh ada yang “init” sebentar)

***

#### Step 3 — Nyalakan vm-worker, lalu cek join + network aman

Di **vm-worker**:

```bash
sudo systemctl status kubelet --no-pager -l
sudo ss -lntp | grep 10250 || echo "10250 tidak listen"

sudo ufw status verbose || true
sudo iptables -S | head -n 30
```

**Yang wajib kamu jaga supaya tidak kejadian laravel prod ngadat lagi:**

* `ufw` **harus disable**
* Policy FORWARD **jangan DROP**

Kalau kamu mau “mengunci” supaya aman setelah reboot, jalankan **sekali** di vm-worker & vm-k8s:

```bash
sudo systemctl disable --now ufw || true

sudo sed -i 's/^DEFAULT_FORWARD_POLICY=.*/DEFAULT_FORWARD_POLICY="ACCEPT"/' /etc/default/ufw || true
grep DEFAULT_FORWARD_POLICY /etc/default/ufw || true
```

***

#### Step 4 — Cek workload prod di k8s

Di **vm-k8s**:

```bash
kubectl -n threebody-prod get pod -o wide
kubectl -n threebody-prod get svc -o wide
kubectl -n threebody-prod get endpoints -o wide
```

Lanjut test laravel internal + DB:

```bash
POD=$(kubectl -n threebody-prod get pod -l app=laravel -o jsonpath='{.items[0].metadata.name}')
kubectl -n threebody-prod exec -it "$POD" -- sh -lc '
php -r "
$h=getenv(\"DB_HOST\"); $p=3306;
$s=@fsockopen($h,$p,$e,$m,3);
echo $s?\"OK TCP $h:$p\n\":\"FAIL TCP $h:$p errno=$e msg=$m\n\";
if($s) fclose($s);
";
curl -fsS -m 10 http://127.0.0.1/api/products | head -c 200; echo
'
```

***

#### Step 5 — Nyalakan vm-docker, lalu cek edge + staging + prod via edge

Di **vm-docker**:

```bash
sudo systemctl status docker --no-pager -l
sudo systemctl start threebody-edge threebody-staging
sudo systemctl status threebody-edge threebody-staging --no-pager -l
docker ps --format 'table {{.Names}}\t{{.Status}}' | egrep 'edge-nginx|stg-' || true
```

Healthcheck:

```bash
# staging
curl -kfsS --resolve staging.local:443:127.0.0.1 https://staging.local/ >/dev/null && echo "OK staging"

# prod homepage
curl -kfsS --resolve prod.local:443:127.0.0.1 https://prod.local/ >/dev/null && echo "OK prod"

# prod api
curl -kfsS --resolve prod.local:443:127.0.0.1 https://prod.local/go/api/products >/dev/null && echo "OK prod go"
curl -kfsS --resolve prod.local:443:127.0.0.1 https://prod.local/laravel/api/products >/dev/null && echo "OK prod laravel"
```

Kalau semua “OK”, berarti **shutdown/boot aman**.

***

### 4) Apakah aman kalau nanti jalankan pipeline setelah boot?

**Aman**, dengan syarat:

1. **Harbor sudah up** (biar push/pull nggak gagal)
2. **K8s nodes Ready** (biar deploy prod bisa rollout)
3. **vm-docker DNS internet aman** (gitlab-runner kamu sempat error `lookup gitlab.com timeout`)

Di **vm-docker**, sebelum run pipeline, cek ini:

```bash
getent hosts gitlab.com || true
curl -I --max-time 5 https://gitlab.com/ 2>/dev/null | head -n 2 || true
resolvectl status | sed -n '1,120p' || true
```

Kalau DNS sering timeout, solusi cepat paling aman:

* Pastikan network VM “NAT”/internet jalan
* Lalu restart resolver:

```bash
sudo systemctl restart systemd-resolved
```

***

### 5) Apakah aman remote SSH dari host setelah restart?

**Aman**, selama:

* IP 192.168.56.xx kamu tetap (host-only)
* `sshd` auto start

Cek di tiap VM:

```bash
sudo systemctl status ssh --no-pager -l
ip a | egrep '192\.168\.56\.' -n || true
```

Dari host:

```bash
ssh cikal@192.168.56.41   # contoh (sesuaikan IP vm-docker)
ssh cikal@192.168.56.43   # vm-harbor
ssh cikal@192.168.56.44   # vm-k8s
ssh cikal@192.168.56.45   # vm-worker
```

***

### 6) Perlu update .gitlab-ci.yml supaya “reboot-proof”?

YAML kamu sudah oke untuk build + deploy staging. Tapi supaya **lebih aman setelah semua VM mati/nyala**, aku sarankan 3 upgrade kecil:

1. **Preflight check**: stop lebih cepat kalau Harbor/K8s belum siap (biar nggak buang waktu).
2. **Deploy staging**: tambah verify `threebody-edge/staging` benar-benar up + test API staging (bukan cuma homepage).
3. **Deploy prod**: isi beneran (sekarang masih placeholder), minimal: ambil kubeconfig dari variable, apply manifest/ set image, rollout status, lalu healthcheck NodePort + via edge.

Kalau kamu mau, aku bisa kirim **full replace `.gitlab-ci.yml`** yang:

* ada `preflight` stage,
* deploy staging lebih strict,
* deploy prod beneran + healthcheck,\
  tetap **sesuai rancangan** (NodePort + edge Nginx, tanpa ingress/metalLB).

Kamu tinggal bilang: manifest prod kamu sekarang disimpan di folder mana (misal `deploy/prod/` atau `k8s/`), atau kalau belum ada aku bikinkan struktur standar yang paling simpel.
