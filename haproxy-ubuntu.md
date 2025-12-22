# HAProxy (Ubuntu)

#### HAProxy

HAProxy (High Availability Proxy) yaitu software load balancer dan reverse proxy

Load Balancer ialah sebuah teknik atau proses Membagi traffic ke beberapa server. Dengan tujuan utama yaitu membantu meningkatkan daya tahan server dengan cara mendistribusikan beban secara merata. Dalam situasi di mana salah satu server mengalami masalah atau gagal, load balancer akan secara otomatis mengalihkan lalu lintas ke server lain yang masih berfungsi. Dengan demikian, pengguna tidak akan merasakan adanya gangguan dalam layanan.

Reverse Proxy ialah sebuah server yang bertindak sebagai perantara antara klien dan server tujuan. Ketika pengguna internet melakukan permintaan, reverse proxy menerima permintaan tersebut dan meneruskannya ke server tujuan. Kemudian, server tujuan memberikan respons ke reverse proxy, dan reverse proxy meneruskannya kembali ke pengguna internet.

<figure><img src=".gitbook/assets/image (333).png" alt=""><figcaption></figcaption></figure>

### Contoh Skenario Sederhana

Mari kita lihat dua skenario untuk membedakan penggunaannya.

#### Skenario A: Reverse Proxy (Fokus pada Routing)

Anda memiliki satu domain (`domain-saya.com`), tapi di belakangnya ada dua aplikasi berbeda yang berjalan di server berbeda.

* Aplikasi Blog (WordPress) di `Server 1`
* Aplikasi Toko Online di `Server 2`

Anda bisa menggunakan Nginx sebagai Reverse Proxy untuk mengatur ini:

* Jika ada yang akses `domain-saya.com/blog/` -> Nginx meneruskan ke Server 1.
* Jika ada yang akses `domain-saya.com/toko/` -> Nginx meneruskan ke Server 2.

> Hasil: Pengguna tidak tahu ada dua server. Bagi mereka, semuanya terlihat seperti satu website yang utuh. Di sini, Nginx bertindak sebagai Reverse Proxy tapi _bukan_ sebagai _Load Balancer_ (karena tujuannya memilah, bukan membagi beban ke server yang identik).

#### Skenario B: Reverse Proxy + Load Balancer (Fokus pada Skalabilitas)

Sekarang, Toko Online Anda sangat sukses dan traffic-nya meledak. Satu server tidak cukup. Anda putuskan untuk memakai 3 server agar kuat.

* Aplikasi Toko Online di `Server A`
* Aplikasi Toko Online di `Server B`
* Aplikasi Toko Online di `Server C`

Anda menggunakan HAProxy di depannya:

* Saat 1000 pengguna datang ke `domain-saya.com/toko/` secara bersamaan:
  * HAProxy mengirim Pengguna 1-333 ke Server A.
  * HAProxy mengirim Pengguna 334-666 ke Server B.
  * HAProxy mengirim Pengguna 667-1000 ke Server C.
* Tiba-tiba, Server B mati.
* HAProxy (lewat _health check_) tahu Server B mati. Ia berhenti mengirim traffic ke sana.
* Sekarang, 500 pengguna baru dibagi rata hanya ke Server A dan Server C.

> Hasil: Toko Anda tetap berjalan lancar meskipun satu server mati, dan pelanggan tidak ada yang merasakan gangguan. Di sini, HAProxy bertindak sebagai Reverse Proxy _dan_ Load Balancer.

| Fungsi                           | Penjelasan Logika                                                | Analogi Dunia Nyata                                                                    |
| -------------------------------- | ---------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| **Load Balancing**               | Membagi beban request ke beberapa server                         | Manajer restoran menyebar tamu ke beberapa pelayan agar tidak numpuk di satu orang     |
| **Reverse Proxy**                | Menjadi pintu depan semua request sebelum ke backend             | Resepsionis hotel yang menerima tamu, lalu mengarahkan ke kamar tertentu               |
| **Health Check**                 | Mengecek apakah server backend hidup atau mati                   | Manajer memastikan setiap pelayan masih di tempat dan siap kerja                       |
| **Failover / High Availability** | Jika satu server gagal, trafik dialihkan otomatis ke server lain | Kalau satu pelayan izin, pelanggan langsung dilayani pelayan cadangan                  |
| **SSL Termination**              | HAProxy menangani enkripsi HTTPS agar backend tidak terbebani    | Resepsionis yang menangani verifikasi identitas tamu, supaya pelayan fokus melayani    |
| **Logging & Monitoring**         | Mencatat semua aktivitas dan performa server                     | Buku laporan manajer: berapa banyak tamu datang, siapa yang dilayani, siapa yang absen |

Instalasi dan Konfigurasi HAProxy

Update dan Install HAProxy

1. Pindah root untuk update

```
sudo su -
```

```
sudo apt update && apt-get upgrade -y
```

atau

```
sudo apt update
```

```
sudo apt install haproxy -y
```

2. Cek versi dan status

```
haproxy -v
sudo systemctl status haproxy
```

Kalau service aktif (running), lanjut ke tahap konfigurasi.

3. Selalu amankan file konfigurasi sebelum edit:

```
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.backup
```

Buka file:

```
sudo nano /etc/haproxy/haproxy.cfg
```

ganti semua menjadi

```
global
    log /dev/log local0
    maxconn 2000
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode http
    option httplog
    option dontlognull
    timeout connect     10s
    timeout client      1m
    timeout server      1m

frontend my_frontend
    bind *:80
    default_backend my_backend

backend my_backend
    balance roundrobin
    option tcp-check
    server nginx 192.168.56.3:81 check
    server nginx2 192.168.56.4:82 check
    server tomcat 192.168.56.4:8080 check
    server weblogic 192.168.56.5:7001 check
    server php 192.168.56.6:83 check

listen monitoring
    bind *:2113
    stats enable
    stats refresh 10s
    stats auth cikal:cikal
    stats uri /
```

4. Periksa syntax:

```
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```

Kalau muncul:

```
Configuration file is valid
```

5. → lanjut restart service:

```
sudo ufw allow 2113/tcp
```

```
sudo ufw allow 80/tcp
```

```
sudo systemctl restart haproxy
```

MAINTENANCE & MONITORING

6. Cek status HAProxy

```
sudo systemctl status haproxy
```

7. Cek log error

```
sudo journalctl -u haproxy -f
```

8. Lihat koneksi aktif

```
sudo ss -tuln | grep haproxy
```

**HASIL AKHIR**

| Komponen           | IP           | Port | Fungsi                      |
| ------------------ | ------------ | ---- | --------------------------- |
| HAProxy            | 192.168.56.2 | 80   | Load balancer utama         |
| NGINX              | 192.168.56.3 | 82   | Backend static HTML         |
| Tomcat             | 192.168.56.4 | 8080 | Backend Java WAR            |
| WebLogic           | 192.168.56.5 | 7001 | Backend aplikasi enterprise |
| PHP/Apache         | 192.168.56.6 | 83   | Backend PHP app             |
| HAProxy Monitoring | 192.168.56.2 | 2113 | Dashboard admin             |
