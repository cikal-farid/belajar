# Penjelasan

Itu **dashboard bawaan kube-prometheus-stack** (tag biasanya `kubernetes-mixin`) dengan nama:

**Kubernetes / Compute Resources / Cluster**

Artinya: ini adalah **monitoring pemakaian resource komputasi level cluster** (CPU & RAM) + breakdown per **namespace**.

### 1) Baris atas (kotak-kotak ringkas)

Ini ringkasan kondisi **seluruh cluster** (gabungan vm-k8s + vm-worker):

* **CPU Utilisation (29.1%)**\
  Pemakaian CPU **aktual saat ini** dibanding total CPU cluster yang “allocatable”.\
  Makin tinggi → cluster makin sibuk.
* **CPU Requests Commitment (48.3%)**\
  Total **CPU requests** semua Pod dibanding kapasitas CPU cluster.\
  Ini penting untuk “seberapa penuh” cluster secara scheduling.\
  &#xNAN;_&#x52;equests_ = “jatah minimum” yang diminta Pod untuk bisa dijadwalkan.
* **CPU Limits Commitment (13.3%)**\
  Total **CPU limits** semua Pod dibanding kapasitas cluster.\
  &#xNAN;_&#x4C;imits_ = “batas maksimum” CPU yang boleh dipakai Pod.
* **Memory Utilisation (37.9%)**\
  Pemakaian RAM **aktual** dibanding kapasitas RAM cluster.
* **Memory Requests Commitment (5.89%)** dan **Memory Limits Commitment (8.84%)**\
  Sama konsepnya seperti CPU, tapi untuk RAM.

**Cara baca cepat:**

* _Utilisation_ = real pemakaian sekarang (beban nyata).
* _Requests commitment_ = seberapa banyak kapasitas “sudah dipesan” oleh Pod (pengaruh ke scheduling).
* _Limits commitment_ = seberapa besar potensi puncak (overcommit risk).

### 2) Grafik “CPU Usage” (area chart)

Grafik ini menunjukkan **CPU cores yang dipakai dari waktu ke waktu**, dipecah per **namespace** (contoh: `kube-system`, `monitoring`, `threebody-prod`).

Di kanan ada legend “Last” (nilai terakhir).\
Contoh yang kamu lihat:

* `kube-system` paling besar → wajar karena komponen Kubernetes.
* `monitoring` juga lumayan (Prometheus/Grafana/Loki).
* `threebody-prod` kecil → aplikasi kamu lagi ringan saat itu.

Kalau kamu lihat spike besar di `threebody-prod`, itu sinyal aplikasi lagi kerja berat (atau ada loop/error/traffic naik).

### 3) Tabel “CPU Quota”

Bagian ini merangkum per namespace:

* **Pods**: jumlah Pod
* **Workloads**: jumlah workload (Deployment/StatefulSet/DaemonSet, dsb)
* **CPU Usage**: pemakaian CPU aktual
* **CPU Requests / Limits**: total requests/limits yang diset di Pod
* **CPU Requests % / CPU Limits %**: biasanya **dibandingkan kuota** kalau namespace punya **ResourceQuota**

Catatan penting: kamu sempat lihat `CPU Requests %` bisa **> 100% (mis. 158%)**. Itu biasanya terjadi kalau:

* Namespace punya **ResourceQuota CPU** yang kecil, sementara total requests melebihi kuota (bisa karena kuota diubah setelah Pod jalan / metrik belum “sinkron”), atau
* Panelnya memang membaca metrik kuota tertentu.

Kalau kamu mau pastikan ini “bener kuota atau bukan”, cek:

```bash
kubectl get resourcequota -A
kubectl -n monitoring describe resourcequota
```

Kalau tidak ada ResourceQuota, angka persen itu biasanya tidak relevan buat kamu.

### Ini monitoring apa, singkatnya?

Ini **dashboard kapasitas & beban cluster**:

* Apakah cluster “masih longgar” atau sudah “penuh”
* Namespace mana yang paling makan CPU/RAM
* Apakah requests/limits kamu sehat (tidak kebesaran/kekecilan)
* Jadi pintu masuk sebelum kamu drill down ke level **Namespace/Pod/Workload**

### Perlu “Save” nggak?

* **Tidak perlu save** kalau kamu cuma melihat.
* Kalau kamu **edit panel** dashboard bawaan, biasanya **lebih aman “Save As”** jadi dashboard kamu sendiri, karena dashboard bawaan bisa “ketimpa” saat helm upgrade.

### Biar “sekali akses URL langsung muncul monitoring ini”

Yang kamu mau itu set **Home Dashboard** di Grafana:

1. Buka dashboard yang kamu suka (**Kubernetes / Compute Resources / Cluster**)
2. Klik **⭐ Star** (opsional tapi enak)
3. Klik ikon user (pojok kiri bawah / kanan atas tergantung versi) → **Preferences**
4. Set **Home Dashboard** = dashboard ini

Setelah itu, buka `http://192.168.56.43:3000/` akan langsung landing ke dashboard pilihanmu.

Kalau kamu mau, kirim output:

```bash
kubectl get resourcequota -A
```

nanti aku bantu bacain apakah “158%” itu aman/sekadar efek kuota, tanpa ngubah-ngubah dulu.
