# Set Up SSL di NGINX (Ubuntu)

## ✅ **STEP 1 — Pastikan Web di Port 81 Berjalan**

Cek service nginx:

```bash
sudo systemctl status nginx
```

Test dari Windows:

```
http://192.168.56.3:81
```

Kalau ini belum muncul, HTTPS memang **tidak akan bisa**.

***

## ✅ **STEP 2 — Buat Folder untuk SSL**

```bash
sudo mkdir -p /etc/nginx/ssl
```

***

## ✅ **STEP 3 — Buat File SAN (Supaya HTTPS Tidak Error Parah)**

Nginx _self-signed_ lebih aman kalau pakai **Subject Alternative Name (SAN)**.

Buat file:

```bash
sudo nano /etc/nginx/ssl/san.cnf
```

Isi dengan:

```
[req]
default_bits       = 2048
distinguished_name = req_distinguished_name
req_extensions     = req_ext
x509_extensions    = v3_ca
prompt = no

[req_distinguished_name]
C = ID
ST = Jawa Barat
L = Bandung
O = MyServer
OU = Dev
CN = 192.168.56.3

[req_ext]
subjectAltName = @alt_names

[v3_ca]
subjectAltName = @alt_names

[alt_names]
IP.1 = 192.168.56.3
```

Simpan → CTRL+S → CTRL+X.

<figure><img src="../../.gitbook/assets/image (723).png" alt=""><figcaption></figcaption></figure>

***

## ✅ **STEP 4 — Generate SSL Certificate**

```bash
sudo openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/selfsigned.key \
  -out /etc/nginx/ssl/selfsigned.crt \
  -config /etc/nginx/ssl/san.cnf
```

<figure><img src="../../.gitbook/assets/image (721).png" alt=""><figcaption></figcaption></figure>

Hasilnya:

* `/etc/nginx/ssl/selfsigned.key`
* `/etc/nginx/ssl/selfsigned.crt`

<figure><img src="../../.gitbook/assets/image (722).png" alt=""><figcaption></figcaption></figure>

***

## ✅ **STEP 5 — Buat Konfigurasi HTTPS di Nginx**

Buat file server block HTTPS:

```bash
sudo nano /etc/nginx/sites-available/https.conf
```

Isi dengan:

```
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name _;

    ssl_certificate /etc/nginx/ssl/selfsigned.crt;
    ssl_certificate_key /etc/nginx/ssl/selfsigned.key;

    root /var/www/html;
    index index.html index.nginx-debian.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

<figure><img src="../../.gitbook/assets/image (719).png" alt=""><figcaption></figcaption></figure>

Edit file default (sebelumnya backup terlebih dahulu di home/cikal)

```
sudo cp /etc/nginx/sites-available/default /home/cikal/default.bak
```

Isi dengan:

```
sudo nano /etc/nginx/sites-available/default
```

```
server {
    listen 81 default_server;
    listen [::]:81 default_server;

    root /var/www/html;
    index index.html index.nginx-debian.html;

    server_name _;

    location / {
        try_files $uri $uri/ =404;
    }
}

```

<figure><img src="../../.gitbook/assets/image (720).png" alt=""><figcaption></figcaption></figure>

***

## 🔗 **STEP 6 — Enable Konfigurasi**

```bash
sudo ln -s /etc/nginx/sites-available/https.conf /etc/nginx/sites-enabled/
```

Disable default config (kalau perlu):

```bash
sudo rm /etc/nginx/sites-enabled/default
```

Jika ada error pada saat test nginx bisa hapus juga file .bak nya

```
sudo rm /etc/nginx/sites-enabled/default.bak
```

***

## ✅ **STEP 7 — Test Konfigurasi**

```bash
sudo nginx -t
```

<figure><img src="../../.gitbook/assets/image (717).png" alt=""><figcaption></figcaption></figure>

Kalau OK → restart:

```bash
sudo systemctl restart nginx
```

<figure><img src="../../.gitbook/assets/image (718).png" alt=""><figcaption></figcaption></figure>

***

## 🔥 **STEP 8 — Allow Firewall (Kalau pakai UFW)**

```bash
sudo ufw allow 443
```

Kalau ingin memastikan port 81 tetap buka:

```bash
sudo ufw allow 81
```

<figure><img src="../../.gitbook/assets/image (716).png" alt=""><figcaption></figcaption></figure>

***

## 🚀 **STEP 9 — Test dari Windows**

Akses:

```
https://192.168.56.3
```

<figure><img src="../../.gitbook/assets/image (715).png" alt=""><figcaption></figcaption></figure>

```
https://192.168.56.3:81/home/
```

<figure><img src="../../.gitbook/assets/image (714).png" alt=""><figcaption></figcaption></figure>

Browser pasti muncul warning _“connection is not private”_,\
klik **Advanced → Proceed**.

Jika masih error:

```
https://192.168.56.3:443
```

***

## ⚠️ Jika Tidak Bisa Diakses — Cek Ini

```
sudo ss -tulpn | grep nginx
```

dan

```
sudo nginx -T
```
