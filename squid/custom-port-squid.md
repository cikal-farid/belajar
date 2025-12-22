# Custom Port Squid

## ✅ **1. Cek Port Squid Saat Ini**

Biasanya default port Squid adalah **3128**.

Cek service Squid berjalan:

```bash
sudo systemctl status squid
```

<figure><img src="../.gitbook/assets/image (705).png" alt=""><figcaption></figcaption></figure>

Cek port Squid yang aktif:

```bash
sudo netstat -tulpn | grep squid
```

atau:

```bash
sudo ss -tulpn | grep squid
```

<figure><img src="../.gitbook/assets/image (706).png" alt=""><figcaption></figcaption></figure>

***

## ✅ **2. Edit Konfigurasi Squid**

File konfigurasi Squid ada di:

Ubuntu/Debian:

```
/etc/squid/squid.conf
```

CentOS/RHEL:

```
/etc/squid/squid.conf
```

Edit file:

```bash
sudo nano /etc/squid/squid.conf
```

Cari baris:

```
http_port 3128
```

Ubah menjadi port yang kamu inginkan.\
Contoh ubah port menjadi **2113**:

```
http_port 2113
```

Jika ingin menggunakan multiple port:

```
http_port 3128
http_port 8080
http_port 9000
```

Jika ingin port **HTTPS (SSL Bump)**:

```
https_port 443 accel cert=/etc/squid/ssl/squid.pem key=/etc/squid/ssl/squid.key
```

> Keterangan: Jika hanya HTTP proxy biasa, cukup `http_port`.

***

## ✅ **3. Cek Konflik Port**

Pastikan port baru tidak digunakan aplikasi lain:

```bash
sudo ss -tulpn | grep 2113
```

<figure><img src="../.gitbook/assets/image (708).png" alt=""><figcaption></figcaption></figure>

Jika ada conflict, pilih port lain.

***

## ✅ **4. Simpan & Exit**

Pada nano:

* Tekan **CTRL + O** → Enter (save)
* Tekan **CTRL + X** (keluar)

***

## ✅ **5. Restart Squid**

Setelah konfigurasi diubah, restart service:

```bash
sudo systemctl restart squid
```

Cek status:

```bash
sudo systemctl status squid
```

<figure><img src="../.gitbook/assets/image (709).png" alt=""><figcaption></figcaption></figure>

Pada bagian log status pun port squid sudah berubah

***

## ✅ **6. Verifikasi Port Baru**

Tes apakah port Squid sudah mendengarkan:

```bash
sudo ss -tulpn | grep squid
```

<figure><img src="../.gitbook/assets/image (710).png" alt=""><figcaption></figcaption></figure>

Contoh output:

```
LISTEN  0  256  *:2113  ...
```

Berarti port 2113 aktif.

***

## ✅ **7. Tes Proxy dari Client**

Dari sisi server:

```bash
curl -x http://192.168.56.14:2113 -I http://google.com
```

<figure><img src="../.gitbook/assets/image (713).png" alt=""><figcaption></figcaption></figure>

Dari PC client:

```bash
curl -x http://192.168.56.14:2113 -I http://google.com
```

Jika muncul HTTP header, berarti berhasil.

<figure><img src="../.gitbook/assets/image (711).png" alt=""><figcaption></figcaption></figure>

## **Coba test download file untuk memastikan caching aktif**

```
curl -x http://192.168.56.14:2113 -O http://speedtest.tele2.net/1MB.zip
```

Lalu unduh kedua kali (should be cached jika caching aktif).

<figure><img src="../.gitbook/assets/image (712).png" alt=""><figcaption></figcaption></figure>

***

## (Opsional) 🔥 **8. Buka Port Firewall**

Kalau server memakai firewall:

#### ● Ubuntu UFW

```bash
sudo ufw allow 2113/tcp
```

#### ● FirewallD (CentOS/RHEL)

```bash
sudo firewall-cmd --add-port=2113/tcp --permanent
sudo firewall-cmd --reload
```

***

## (Opsional) 🔥 **9. Batasi Siapa yang Boleh Pakai Port Baru**

Jika ingin mengizinkan hanya subnet 192.168.56.0/24:

```bash
acl localnet src 192.168.56.0/24
http_access allow localnet
http_access deny all
```

***

## 🎉 **Selesai – Squid Berjalan dengan Custom Port**

Jika kamu berikan port yang ingin dipakai, saya bisa buatkan konfigurasi lengkap + cek firewall + test command secara otomatis.
