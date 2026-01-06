# Bersih-bersih kuber

Boleh, itu **arahnya sudah benar** buat “retry bersih”. Tapi biar benar-benar aman (nggak ada file yang masih “dipegang” pod mysql saat kamu hapus folder), aku sarankan **urutannya sedikit dirapikan**.

### 0) Cek dulu (biar yakin yang akan dihapus memang cuma environment app)

Di **vm-k8s**:

```bash
NS=threebody-prod

kubectl get ns | egrep -n "NAME|threebody|monitoring" || true
kubectl -n $NS get all || true
kubectl -n $NS get pvc || true
kubectl get pv | egrep -n "NAME|mysql-pv-hostpath" || true
```

Kalau yang muncul memang hanya resource “threebody”, lanjut bersih.

***

### 1) Bersihkan di vm-k8s (disarankan, lebih aman dari “masih running”)

#### A. Matikan dulu workload (biar volume dilepas clean)

```bash
NS=threebody-prod

kubectl -n $NS scale deployment/frontend deployment/go deployment/laravel --replicas=0 || true
kubectl -n $NS scale statefulset/mysql --replicas=0 || true

# pastikan podnya hilang dulu (terutama mysql)
kubectl -n $NS delete pod -l app=mysql --ignore-not-found=true || true
kubectl -n $NS delete pod -l app=frontend --ignore-not-found=true || true
kubectl -n $NS delete pod -l app=go --ignore-not-found=true || true
kubectl -n $NS delete pod -l app=laravel --ignore-not-found=true || true
```

#### B. Hapus namespace + PV

```bash
kubectl delete ns $NS --wait=true || true

# PV cluster-scoped (ini yang bikin “nggak bersih” kalau dibiarkan)
kubectl delete pv mysql-pv-hostpath --ignore-not-found=true || true
```

#### Cek hasil

```bash
kubectl get ns | grep -w threebody-prod || echo "OK: namespace sudah hilang"
kubectl get pv | grep -w mysql-pv-hostpath || echo "OK: PV sudah hilang"
```

> Kalau namespace/PV **nyangkut Terminating**, baru lakukan “force remove finalizer” (kasih tahu aku outputnya, nanti aku arahkan patch finalizer yang tepat).

***

### 2) Bersihkan di vm-worker (data mysql hostPath)

Ini aman, **tapi lakukan setelah step vm-k8s di atas** (supaya pod mysql benar-benar mati dan nggak nulis lagi).

```bash
sudo rm -rf /data/threebody/mysql/*
sudo mkdir -p /data/threebody/mysql
sudo chown -R 999:999 /data/threebody/mysql
sudo chmod 700 /data/threebody/mysql
sudo ls -la /data/threebody/mysql
```

***

### Catatan soal “imagePullSecret variable”

Kamu sudah benar: **jangan hapus `spec:`** di manifest. Yang kamu lakukan sekarang (pakai variable `K8S_IMAGEPULL_SECRET` + patch ke default serviceaccount) sudah cocok, jadi manifest deployment tidak perlu `imagePullSecrets:` lagi.

***

Kalau kamu mau aku pastikan ini **100% bersih**, kirim output ini setelah kamu jalankan “cek dulu” (bagian 0):

```bash
kubectl -n threebody-prod get all 2>/dev/null || echo "ns not found"
kubectl get pv | grep mysql || echo "no mysql pv"
```

Ambil password Grafana (kube-prometheus-stack / kps)

1. Cara paling umum (sesuai NOTES helm kamu)

Di vm-k8s:

kubectl -n monitoring get secret kps-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo

Login:

user: admin

password: hasil command di atas
