# Monitoring di pipeline

Bisa **secara teknis**, tapi yang kamu tulis itu **systemd service di host OS (vm-k8s)** — sedangkan pipeline GitLab kamu jalan di **runner vm-docker**. Jadi pipeline **nggak bisa langsung** `sudo tee /etc/systemd/...` di vm-k8s _kecuali_ kamu bikin pipeline **remote-exec via SSH** ke vm-k8s dan user-nya punya **passwordless sudo**.

Di lab kamu, ada 2 opsi yang paling “aman & enak dipakai dari VM kosong”:

***

### Opsi A (paling rekomendasi): Expose UI pakai NodePort (tanpa systemd, tanpa port-forward)

Ini bikin pipeline selesai → UI langsung bisa diakses dari laptop via `http://IP_NODE:NodePort`.

Kamu bisa **buat service tambahan** (lebih aman daripada patch service helm), misalnya file baru:

`deploy/monitoring/expose-ui-nodeport.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana-nodeport
  namespace: monitoring
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: kps
  ports:
    - name: http
      port: 80
      targetPort: 3000
      nodePort: 30030
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus-nodeport
  namespace: monitoring
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/instance: kps
  ports:
    - name: http
      port: 9090
      targetPort: 9090
      nodePort: 30090
---
apiVersion: v1
kind: Service
metadata:
  name: loki-gateway-nodeport
  namespace: monitoring
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: loki
    app.kubernetes.io/instance: loki
    app.kubernetes.io/component: gateway
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30100
```

Lalu di job `monitoring_install` (setelah helm install) tambahkan:

```bash
kubectl apply -f deploy/monitoring/expose-ui-nodeport.yaml
kubectl -n monitoring get svc | egrep -i 'grafana-nodeport|prometheus-nodeport|loki-gateway-nodeport'
```

Akses dari laptop:

* Grafana: `http://192.168.56.44:30030` (bisa juga pakai `192.168.56.43`)
* Prometheus: `http://192.168.56.44:30090`
* Loki gateway: `http://192.168.56.44:30100`

> Ini **tidak butuh** edit `C:\Windows\System32\drivers\etc\hosts` karena kamu akses pakai IP:port.

**Password Grafana tetap ambil dari secret:**

```bash
kubectl -n monitoring get secret kps-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo
```

✅ Keuntungan opsi A: pipeline selesai, UI langsung bisa diakses, tidak tergantung proses port-forward yang harus hidup.

***

### Opsi B: Pipeline “membuat systemd port-forward” (bisa, tapi butuh SSH + sudo)

Kalau kamu tetap mau model service systemd seperti `kpf-grafana.service`, pipeline harus **SSH ke vm-k8s** dan menjalankan perintah di sana.

Syarat minimal:

1. vm-k8s ada SSH server
2. user SSH (misal `cikal`) bisa `sudo` tanpa password untuk `tee/systemctl` (atau kamu set `NOPASSWD`)
3. GitLab CI punya SSH private key (Variable type File / masked)

Contoh konsep job (inti doang, biar kebayang):

```bash
# di pipeline job
eval $(ssh-agent -s)
ssh-add "$SSH_KEY_FILE"

ssh -o StrictHostKeyChecking=no cikal@192.168.56.43 'sudo tee /etc/systemd/system/kpf-grafana.service >/dev/null <<EOF
[Unit]
Description=Kubernetes Port-Forward Grafana (monitoring)
After=network-online.target
Wants=network-online.target
[Service]
Type=simple
Environment=KUBECONFIG=/etc/kubernetes/admin.conf
ExecStart=/usr/bin/kubectl -n monitoring port-forward svc/kps-grafana 3000:80 --address 0.0.0.0
Restart=always
RestartSec=3
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now kpf-grafana
'
```

⚠️ Catatan penting:

* Aku sarankan `--address 0.0.0.0` (biar listen di semua interface). Kalau pakai `--address 192.168.56.43` juga oke, tapi pastikan interface itu memang ada.
* Pipeline **tidak bisa** “menjaga” proses port-forward tetap hidup setelah job selesai, kecuali kamu jadikan **service systemd** di vm-k8s (yang kamu lakukan).
* Ini lebih ribet + butuh akses root via SSH → untuk lab, NodePort biasanya jauh lebih simpel.

***

### Tentang “cleanup pv ada titik-titik …”

Itu cuma **placeholder** (bukan harus ada titik-titik). Kalau kamu mau “reset total” DB MySQL (fresh dari nol), urutan aman yang kamu tulis itu benar:

* delete job migrate (biar nggak ganggu)
* scale mysql=0
* delete pod mysql-0
* delete pvc + pv
* bersihkan folder hostPath di node yang nyimpen data (`vm-worker`)

Itu memang diperlukan **kalau kamu ingin benar-benar mulai dari DB kosong**.\
Tapi untuk “start dari VM kosong” nanti, kamu **nggak perlu cleanup**—karena VM kosong = datanya belum ada.

***

### Kenapa tadi `192.168.56.43:9090 / :3000` tidak bisa?

Karena service helm default **ClusterIP** (hanya bisa diakses dari dalam cluster). Jadi harus:

* **NodePort** (opsi A), atau
* **Ingress**, atau
* **port-forward** (manual/systemd)

***

#### Saran final untuk “VM kosong + sekali push langsung jadi”

✅ Pakai **Opsi A (NodePort expose)**, lalu monitoring job selesai → kamu bisa langsung akses UI tanpa systemd dan tanpa port-forward.

Kalau kamu mau, kirim output ini (sekali aja) supaya aku pastiin selector NodePort-ku 100% match helm chart kamu:

```bash
kubectl -n monitoring get svc kps-grafana -o yaml | sed -n '1,120p'
kubectl -n monitoring get svc kps-kube-prometheus-stack-prometheus -o yaml | sed -n '1,160p'
kubectl -n monitoring get svc loki-gateway -o yaml | sed -n '1,160p'
```

Tapi kalau kamu nggak mau nambah langkah, langsung pakai manifest NodePort di atas juga biasanya sudah jalan untuk setup `kps` dan `loki` seperti log kamu.
