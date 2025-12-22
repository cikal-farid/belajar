# Instalasi dan Konfigurasi Squid (Ubuntu)

Berikut adalah perintah untuk instalasi Squid di Ubuntu Server

Tentu! Ini adalah panduan lengkap untuk mengatur Squid proxy di VM1 (Ubuntu Server) untuk memfilter koneksi dari VM2 (Ubuntu Client) berdasarkan skenario Anda.

Skenario ini mengasumsikan:

* VM1 (Server): IP `192.168.56.14` (Host-Only) & punya akses internet (via NAT).
* VM2 (Client): IP `192.168.56.15` (Host-Only).
* VM2 bisa melakukan `ping 192.168.56.14`.

***

#### Langkah 1: Konfigurasi VM1 (Server Squid - 192.168.56.14)

Di server ini, kita akan menginstal Squid, membuat daftar situs yang diblokir, dan mengonfigurasi Squid untuk menggunakan daftar tersebut.

**1. Instalasi Squid**

Buka terminal di VM1 dan jalankan perintah berikut:

Update daftar paket

```
sudo apt update
```

Install Squid

```
sudo apt install squid -y
```

**2. Buat Daftar Situs yang Akan Diblokir**

Kita akan membuat file terpisah untuk menyimpan daftar domain yang ingin diblokir.

Buat file baru bernama 'situs\_diblokir.txt'

```
sudo nano /etc/squid/situs_diblokir.txt
```

Masukkan domain yang ingin Anda blokir. Gunakan titik di depan domain (misalnya `.facebook.com`) untuk memblokir domain tersebut beserta semua subdomainnya (seperti `www.facebook.com`, `m.facebook.com`, dll.).

```
# Isi file /etc/squid/situs_diblokir.txt
.facebook.com
.detik.com
.instagram.com
```

Simpan file tersebut (Ctrl+O, Enter) dan tutup (Ctrl+X).

**3. Konfigurasi Squid (`squid.conf`)**

Sekarang kita edit file konfigurasi utama Squid. Sebaiknya backup dulu file aslinya:

Backup file konfigurasi asli

```
sudo cp /etc/squid/squid.conf /etc/squid/squid.conf.asli
```

Buka file konfigurasi dan edit

```
sudo nano /etc/squid/squid.conf
```

Di dalam file `squid.conf`, kita perlu melakukan 3 hal:

1. Membuat ACL (Access Control List) untuk jaringan klien kita (VM2).
2. Membuat ACL untuk membaca file daftar blokir kita.
3. Menerapkan aturan (Rules) untuk mengizinkan klien dan memblokir situs.

Cari bagian ACL (biasanya dimulai dengan `acl ...`). Tambahkan dua baris ini di area tersebut:

```
# ACL untuk jaringan lokal kita (mencakup IP 192.168.56.15)
acl jaringan_lokal src 192.168.56.0/24

# ACL untuk daftar situs yang diblokir (mengambil dari file)
acl situs_terlarang dstdomain "/etc/squid/situs_diblokir.txt"
```

Selanjutnya, cari bagian `http_access` (biasanya `http_access allow localhost`). Urutan sangat penting di sini. Aturan dibaca dari atas ke bawah.

Kita harus menempatkan aturan kita _sebelum_ `http_access deny all`. Konfigurasi yang aman adalah seperti ini:

```
# 1. Blokir situs terlarang UNTUK jaringan lokal
http_access deny situs_terlarang jaringan_lokal

# 2. Izinkan sisa trafik DARI jaringan lokal
http_access allow jaringan_lokal

# 3. Izinkan akses dari localhost (server itu sendiri)
http_access allow localhost

# 4. Tolak semua permintaan lain yang tidak cocok dengan aturan di atas
http_access deny all
```

Terakhir, pastikan port Squid aktif. Cari `http_port 3128`. Baris ini biasanya sudah ada. Port `3128` adalah port default yang akan didengarkan oleh Squid.

```
# Pastikan baris ini ada dan tidak ada tanda # di depannya
http_port 3128
```

Simpan file (Ctrl+O, Enter) dan tutup (Ctrl+X).

**4. Restart dan Buka Firewall**

Terapkan konfigurasi baru dengan me-restart layanan Squid:

Restart service squid agar membaca konfigurasi yang baru

```
sudo systemctl restart squid
```

Cek status untuk memastikan tidak ada error

```
sudo systemctl status squid
```

Jika Anda menggunakan UFW (Uncomplicated Firewall) di Ubuntu, Anda harus mengizinkan koneksi masuk ke port 3128:

Izinkan port 3128/tcp

```
sudo ufw allow 3128/tcp
```

Reload firewall

```
sudo ufw reload
```

Server VM1 sekarang siap menerima koneksi proxy.

***

#### Langkah Pengujian di VM2 (Client - 192.168.56.15)

Disini kita perlu mematikan jaringan NAT pada VM2 ini untuk sekaligus menguji apakah dapat koneksi inet pada VM1 atau tidak

<figure><img src="../.gitbook/assets/image (649).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (650).png" alt=""><figcaption></figcaption></figure>

Buka terminal di VM2 dan jalankan perintah-perintah berikut:

**1. Tes Situs yang Diizinkan (Misal: https://www.google.com/url?sa=E\&source=gmail\&q=google.com)**

Perintah ini akan mengarahkan `curl` untuk meminta `google.com` melalui proxy Anda di `192.168.56.14:3128`.

```
curl -x http://192.168.56.14:3128 -I http://google.com
```

* Hasil yang Diharapkan: Anda akan melihat banyak kode HTML (output dari halaman Google), yang menandakan koneksi berhasil dan diizinkan.

**2. Tes Situs yang Diblokir (Misal: detik.com)**

Sekarang, kita uji salah satu domain yang ada di file `situs_diblokir.txt`.

```
curl -x http://192.168.56.14:3128 -I http://detik.com
```

* Hasil yang Diharapkan: Anda tidak akan melihat HTML. Sebaliknya, Anda akan melihat pesan error dari Squid, yang biasanya berisi kata-kata seperti "Access Denied" atau "Forbidden". Ini menandakan Squid berhasil memblokir permintaan tersebut.

***

#### (Opsional) Mengatur Proxy Secara Global di VM2

Jika Anda ingin _semua_ perintah di terminal VM2 (seperti `apt update`, `wget`, dll.) otomatis menggunakan proxy tanpa perlu mengetik flag `-x` setiap saat, Anda bisa mengatur _environment variables_.

Tambahkan baris berikut ke file `~/.bashrc` Anda:

```
sudo nano ~/.bashrc
```

Tambahkan di bagian paling bawah file:

```
# Set proxy untuk HTTP dan HTTPS
export http_proxy="http://192.168.56.14:3128"
export https_proxy="http://192.168.56.14:3128"

# Opsional: Jika ada IP yang tidak ingin dilewatkan proxy (misal server VM1 itu sendiri)
export no_proxy="localhost,127.0.0.1,192.168.56.14"
```

Simpan file (Ctrl+O, Enter) dan tutup (Ctrl+X). Lalu, terapkan pengaturan:

```
source ~/.bashrc
```

Kita jika perlu membuat file ini agar dapat akses internet via http yang berjalan lewat squid di ubuntu

```
sudo nano /etc/apt/apt.conf.d/80proxy
```

<figure><img src="../.gitbook/assets/image (655).png" alt=""><figcaption></figcaption></figure>

Restart service DNS Resolver

```
sudo systemctl restart systemd-resolved
```

Sekarang, Anda bisa menguji hanya dengan:

Ini akan otomatis menggunakan proxy

```
curl -x http://192.168.56.14:3128 -I http://detik.com
```

(Hasil: Access Denied)

```
curl -x http://192.168.56.14:3128 -I http://google.com
```

Perintah untuk update sistem dan sekaligus menguji apakah bisa mendapatkan inet

```
sudo apt update
```
