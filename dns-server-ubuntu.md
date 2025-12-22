# DNS Server (Ubuntu)

## INSTALLASI DAN KONFIGURASI DNS SERVER

### A. Pengertian DNS Server

Domain Name System Server atau DNS adalah sebuah sistem yang menghubungkan Uniform Resource Locator (URL) dengan Internet Protocol Address (IP Address). Untuk mengakses internet kita perlu mengetikkan ip address sebuah website. Cara ini cukup merepotkan karena kita perlu punya daftar lengkap ip address website yang dikunjungi dan memasukkannya secara manual.\
DNS adalah system yang meringkas pekerjaan ini. Kita hanya perlu mengingat nama domain dan memasukkannya ke dalam address bar. DNS kemudian akan menerjemahkan domain tersebut ke dalam IP Address yang komputer pahami. Berikut fungsi DNS.

* Meminta informasi IP Address sebuah website berdasarkan nama domain.
* Meminta informasi URL sebuah website berdasarkan IP Address yang dimasukkan.
* Mencari server yang tepat untuk mengirimkan email.

### B. Arsitektur DNS Server

Prinsip dasar cara kerja DNS adalah dengan cara mencocokkan nama komponen URL dengan komponen IP Address. Setiap URL dan IP Address memiliki bagian-bagian yang saling menjelaskan satu dengan yang lain. Struktur DNS ini berbentuk hierarki atau juga pohon yang mempunyai beberapa cabang. Cabang-cabang tersebut yang mewakili domain, serta bisa atau dapat berupa host, subdomain, ataupun juga top level domain.

#### Struktur dan Hierarki DNS

Root (.)\
├── com\
│ └── google.com\
│ └── www.google.com\
├── org\
│ └── wikipedia.org\
└── id\
└── go.id

* **Root DNS** = level tertinggi (seperti pemerintah pusat)
* **TLD DNS (.com, .org, .id)** = level menengah (seperti provinsi)
* **Authoritative DNS (google.com)** = level akhir (seperti RT/RW yang tahu alamat rumah pasti)

#### Jenis Data di DNS (Record)

| Record    | Fungsi Logis                        | Contoh                                |
| --------- | ----------------------------------- | ------------------------------------- |
| **A**     | Nama → IP versi 4                   | google.com → 142.250.190.14           |
| **AAAA**  | Nama → IP versi 6                   | google.com → 2607:f8b0:4005:808::200e |
| **CNAME** | Alias nama                          | www → google.com                      |
| **MX**    | Alamat mail server                  | gmail.com → mail.google.com           |
| **NS**    | Server DNS resmi domain             | google.com → ns1.google.com           |
| **TXT**   | Catatan tambahan (misal verifikasi) | SPF, DKIM, dll                        |

#### Konfigurasi DNS server sederhana (pakai Bind9)

1. Instalasi BIND9 (Ubuntu)

```
sudo apt update && apt-get upgrade -y
```

atau

```
sudo apt update
```

```
sudo apt install bind9 bind9utils bind9-doc -y
```

2. Aktifkan service bind9

```
sudo systemctl status bind9
```

```
sudo systemctl enable bind9
```

```
sudo systemctl start bind9
```

```
sudo systemctl restart bind9
```

3. Cek Interface & IP

```
ip addr show enp0s8
```

4. Definisikan Zona di `named.conf.local`

```
sudo nano /etc/bind/named.conf.local
```

Tambahkan

```
zone "cinosta.local" {
    type master;
    file "/etc/bind/db.cinosta.local";
};

zone "56.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.56";
};
```

5. Buat File Zona (Forward Zone)

```
sudo cp /etc/bind/db.local /etc/bind/db.cinosta.local
```

edit

```
sudo nano /etc/bind/db.cinosta.local
```

```
$TTL 604800
@   IN  SOA cinosta.local. root.cinosta.local. (
        2025102501 ; Serial
        604800     ; Refresh
        86400      ; Retry
        2419200    ; Expire
        604800 )   ; Negative Cache TTL

; Nameserver
@       IN  NS      dns.cinosta.local.

; Host record
dns     IN  A       192.168.56.7
web1    IN  A       192.168.56.7
@       IN  A       192.168.56.7
```

6. Buat file zona _reverse_ (`db.192.168.56`)

```
sudo nano /etc/bind/db.192.168.56
```

```
$TTL    604800
@       IN      SOA     dns.cinosta.local. admin.cinosta.local. (
                        2025102501
                        604800
                        86400
                        2419200
                        604800 )

; Name Server
@       IN      NS      dns.cinosta.local.

; PTR Record (reverse lookup)
7      IN      PTR     dns.cinosta.local.
7      IN      PTR     web1.cinosta.local.
```

7. Pastikan `listen-on` di Bind9 sudah “any”

```
sudo nano /etc/bind/named.conf.options
```

```
options {
    directory "/var/cache/bind";
    recursion yes;
    allow-query { any; };
    forwarders {
        8.8.8.8;
        1.1.1.1;
    };
    listen-on { any; };
    listen-on-v6 { any; };
};
```

8. Simpan dan restart bind9

```
sudo systemctl restart bind9
```

9. cek status UFW:

```
sudo ufw status
```

Tambahkan :

```
sudo ufw allow 80/tcp
```

```
sudo ufw allow 53
```

```
sudo ufw allow 53/udp
```

```
sudo ufw reload
```

10. Cek konfigurasi virtual host di nginx, karena disini kita test langsung di nginx

```
sudo nano /etc/nginx/sites-available/default
```

Pastikan di dalamnya ada: (pastikan pada bagian server\_name disesuaikan seperti dibawah ini)

```
server {
    listen 80;
    server_name web1.cinosta.local;
    root /var/www/html;
    index index.html;
}
```

11. Reload nginx

```
sudo systemctl restart nginx
```

12. Edit `/etc/resolv.conf`: (secara default sudah ada dan kita remark terlebih dahulu ganti dengan yang dibawah)

```
sudo nano /etc/resolv.conf
```

```
nameserver 192.168.56.7
```

13. kita perlu menambahkan prefered DNS server dan alternatif DNS server dari sisi pc host (windows) di control panelnya pada bagian network VirtualBox Host-Only Network, karena server yang saat ini kita gunakan server dari vm virtual box, seperti pada gambar dibawah ini

<figure><img src=".gitbook/assets/image (476).png" alt=""><figcaption></figcaption></figure>

14. Klik OK → tutup semua jendela → buka CMD, jalankan: (windows)

```
ipconfig /flushdns
```

```
nslookup web1.cinosta.local
```

hasil dari command diatas seperti pada teks dibawah ini

Windows IP Configuration\
Successfully flushed the DNS Resolver Cache.

Server: dns.cinosta.local\
Address: 192.168.56.7

Name: web1.cinosta.local\
Address: 192.168.56.8

15. Dan test ping ip nya juga (cmd windows)

```
ping 192.168.56.7
```

```
ping web1.cinosta.local
```

16. Cek apakah Bind9 dengar di IP 192.168.56.7

```
sudo ss -tulnp | grep :53
```

hasilnya

udp UNCONN 0 0 192.168.56.7:53 _:_ users:(("named",pid=xxxx,fd=xx)) tcp LISTEN 0 10 192.168.56.7:53 _:_ users:(("named",pid=xxxx,fd=xx))

Periksa Konfigurasi

17. Cek konfigurasi utama:

```
sudo named-checkconf
```

18. Cek zona forward:

```
sudo named-checkzone cinosta.local /etc/bind/db.cinosta.local
```

19. Cek zona reverse:

```
sudo named-checkzone 56.168.192.in-addr.arpa /etc/bind/db.192.168.56
```

Kalau semuanya “OK”, artinya konfigurasi valid.

20. Uji di web browser

```
http://web1.cinosta.local/home/
```

```
http://192.168.56.7/home/
```
