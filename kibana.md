# Kibana

Kibana merupakan open source data visualization dan exploration tool untuk\
Elasticsearch. Log dan time series analytic, aplikasi monitoring, dan kegiatan operasional dapat\
dengan mudah dilakukan dengan Kibana karena antarmuka web dashboard nya yang rapi\
membuat pengelolaan dan visualisasi data dari Elasticsearch menjadi sangat lancer.

Berikut **runbook detail opsi 2 (2 VM)** untuk Ubuntu 24.04 dengan IP:

| VM      | IP                | Role                                                                              |
| ------- | ----------------- | --------------------------------------------------------------------------------- |
| **VM1** | **192.168.56.21** | **Elasticsearch + Kibana**                                                        |
| **VM2** | **192.168.56.22** | **Logstash + Filebeat** (kirim log → Logstash → Elasticsearch → tampil di Kibana) |

Port yang dipakai:

* **Elasticsearch:** `9200/tcp` (VM1)
* **Kibana:** `5601/tcp` (VM1)
* **Logstash Beats input:** `5044/tcp` (VM2)

> Catatan versi: pastikan **Elasticsearch, Kibana, Logstash, Filebeat = 1 major version yang sama** (misal semuanya 8.x).

***

## 0) Persiapan (dijalankan di KEDUA VM)

Set hostname (opsional tapi enak dibaca):

```bash
# di VM1
sudo hostnamectl set-hostname vm-elk

# di VM2
sudo hostnamectl set-hostname vm-ingest
```

Tambahkan `/etc/hosts` di **keduanya** (biar pakai nama):

```bash
sudo nano /etc/hosts
```

Tambahkan:

```
192.168.56.21  vm-elk
192.168.56.22  vm-ingest
```

Update paket dasar:

```bash
sudo apt-get update
sudo apt-get install -y curl wget gnupg apt-transport-https
```

***

## 1) Tambah APT repo Elastic (dijalankan di KEDUA VM)

Import GPG key:

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch \
| sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
```

Tambah repo (aku rekomendasikan **8.x** untuk lab):

```bash
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" \
| sudo tee /etc/apt/sources.list.d/elastic-8.x.list
```

Lalu:

```bash
sudo apt-get update
```

***

## 2) VM1 (192.168.56.21): Install & konfigurasi Elasticsearch

### 2.1 Install

Di **VM1**:

```bash
sudo apt-get install -y elasticsearch
```

### 2.2 Konfigurasi Elasticsearch (mode lab, tanpa password)

Edit config:

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

Isi minimal (boleh tempel di bawah, yang penting key ini ada):

```yaml
network.host: 0.0.0.0
http.port: 9200
discovery.type: single-node

# mode lab (tanpa auth/TLS) – lebih mirip runbook sederhana
xpack.security.enabled: false
xpack.security.enrollment.enabled: false
xpack.security.http.ssl.enabled: false
xpack.security.transport.ssl.enabled: false
```

Start + enable:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now elasticsearch
```

Cek service:

```bash
sudo systemctl status elasticsearch --no-pager
```

Cek Elasticsearch hidup:

```bash
curl http://localhost:9200
```

### 2.3 Firewall (VM1)

Buka port ES + Kibana:

```bash
sudo ufw allow 9200/tcp
sudo ufw allow 5601/tcp
sudo ufw enable
sudo ufw status
```

***

## 3) VM1 (192.168.56.21): Install & konfigurasi Kibana

### 3.1 Install

Di **VM1**:

```bash
sudo apt-get install -y kibana
```

### 3.2 Konfigurasi

Edit:

```bash
sudo nano /etc/kibana/kibana.yml
```

Set minimal:

```yaml
server.host: "0.0.0.0"
server.port: 5601
elasticsearch.hosts: ["http://192.168.56.21:9200"]
```

Start + enable:

```bash
sudo systemctl enable --now kibana
```

Cek:

```bash
sudo systemctl status kibana --no-pager
```

Tes port dari VM1:

```bash
curl -I http://localhost:5601
```

Akses dari laptop/host:

* `http://192.168.56.21:5601`

***

## 4) VM2 (192.168.56.22): Install & konfigurasi Logstash

### 4.1 Install

Di **VM2**:

```bash
sudo apt-get install -y logstash
```

### 4.2 Buat pipeline Logstash (Beats 5044 → Elasticsearch VM1)

Buat file config:

```bash
sudo nano /etc/logstash/conf.d/beats-to-elasticsearch.conf
```

Isi:

```conf
input {
  beats {
    port => 5044
    ssl => false
  }
}

output {
  elasticsearch {
    hosts => ["http://192.168.56.21:9200"]
    index => "%{[@metadata][beat]}-%{+YYYY.MM.dd}"
  }
  stdout { codec => rubydebug }
}
```

Start + enable:

```bash
sudo systemctl enable --now logstash
```

Cek:

```bash
sudo systemctl status logstash --no-pager
```

Cek Logstash listen port 5044:

```bash
sudo ss -lntp | grep 5044
```

### 4.3 Firewall (VM2)

Buka port 5044:

```bash
sudo ufw allow 5044/tcp
sudo ufw enable
sudo ufw status
```

***

## 5) VM2 (192.168.56.22): Install & konfigurasi Filebeat

Filebeat di VM2 akan ngirim **log system VM2** ke Logstash.

### 5.1 Install

Di **VM2**:

```bash
sudo apt-get install -y filebeat
```

### 5.2 Konfigurasi output ke Logstash

Edit:

```bash
sudo nano /etc/filebeat/filebeat.yml
```

Pastikan bagian output jadi seperti ini:

```yaml
output.logstash:
  hosts: ["192.168.56.22:5044"]

#output.elasticsearch:
#  hosts: ["http://192.168.56.21:9200"]
```

Aktifkan module system:

```bash
sudo filebeat modules enable system
sudo filebeat modules list
```

### 5.3 Load template + dashboards (penting!)

Karena output Filebeat ke Logstash, template biasanya perlu di-load manual.\
Jalankan di **VM2**:

**Load template ke Elasticsearch:**

```bash
sudo filebeat setup --template \
  -E output.logstash.enabled=false \
  -E 'output.elasticsearch.hosts=["http://192.168.56.21:9200"]'
```

**Load dashboards ke Kibana:**

```bash
sudo filebeat setup -e \
  -E output.logstash.enabled=false \
  -E 'output.elasticsearch.hosts=["http://192.168.56.21:9200"]' \
  -E setup.kibana.host=http://192.168.56.21:5601
```

### 5.4 Start Filebeat

```bash
sudo systemctl enable --now filebeat
sudo systemctl status filebeat --no-pager
```

***

## 6) Verifikasi end-to-end

### 6.1 Cek koneksi Filebeat → Logstash (VM2)

Lihat log Filebeat:

```bash
sudo journalctl -u filebeat -n 50 --no-pager
```

Lihat log Logstash:

```bash
sudo tail -n 50 /var/log/logstash/logstash-plain.log
```

### 6.2 Cek index masuk di Elasticsearch (VM1)

Di **VM1**:

```bash
curl 'http://localhost:9200/_cat/indices?v'
```

Harus muncul index seperti:

* `filebeat-YYYY.MM.DD`

Cari data:

```bash
curl 'http://localhost:9200/filebeat-*/_search?pretty' | head
```

### 6.3 Cek di Kibana (Browser)

Buka:

* `http://192.168.56.21:5601`

Langkah:

1. Masuk menu **Discover**
2. Buat **Data View**: `filebeat-*` (kalau diminta)
3. Harus terlihat log event dari VM2

***

## 7) Quick check (cek sederhana via CLI)

### VM1

```bash
sudo systemctl is-active elasticsearch kibana
sudo ss -lntp | egrep ':(9200|5601)\s'
curl -s http://localhost:9200 >/dev/null && echo "ES OK"
curl -I -s http://localhost:5601 | head -n 1
```

### VM2

```bash
sudo systemctl is-active logstash filebeat
sudo ss -lntp | egrep ':(5044)\s'
sudo journalctl -u filebeat -n 20 --no-pager
```

***

## Troubleshooting cepat (paling sering kejadian)

1. **Kibana kebuka tapi tidak ada data**

*   Pastikan index ada:

    ```bash
    curl 'http://192.168.56.21:9200/_cat/indices?v'
    ```
*   Pastikan Filebeat jalan & tidak error:

    ```bash
    sudo journalctl -u filebeat -n 100 --no-pager
    ```

2. **Filebeat gagal connect ke Logstash**

*   Pastikan Logstash listen `5044`:

    ```bash
    sudo ss -lntp | grep 5044
    ```
*   Pastikan UFW VM2 allow 5044:

    ```bash
    sudo ufw status
    ```

3. **Logstash gagal kirim ke Elasticsearch**

*   Cek Elasticsearch bisa diakses dari VM2:

    ```bash
    curl http://192.168.56.21:9200
    ```

***

Kalau kamu mau, aku bisa bikinin juga **versi yang benar-benar “copy-paste” tanpa edit manual nano** (pakai `cat <<EOF` buat langsung menimpa file config), biar runbook-nya makin gampang buat pemula.
