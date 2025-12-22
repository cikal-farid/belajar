# PHP (Ubuntu)

**PHP (Hypertext Preprocessor)** adalah **bahasa pemrograman server-side** yang digunakan untuk **membangun dan memproses halaman web dinamis** sebelum dikirim ke browser.

* Kode PHP **tidak langsung terlihat oleh pengguna**.
* PHP **dijalankan di sisi server**, bukan di browser seperti JavaScript.
* Hasil akhirnya berupa **HTML** (atau JSON, XML, dsb) yang dikirim ke browser.

Instalasi dan Konfigurasi PHP

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
sudo apt install whiptail -y
```

```
sudo apt install apache2 php libapache2-mod-php -y
```

2. Cek status Apache:

```
sudo systemctl status apache2
```

3. Edit file konfigurasi port:

```
sudo nano /etc/apache2/ports.conf
```

Ubah atau tambahkan:

```
Listen 83
```

4. Kemudian edit file default virtual host:

```
sudo nano /etc/apache2/sites-available/000-default.conf
```

Ubah baris:

```
<VirtualHost *:83>
```

menjadi:

```
<VirtualHost *:83>
```

5. Buat file test PHP: (optional karena disini ingin load balance haproxy)

```
mkdir -p /var/www/html/home/
```

```
sudo nano /etc/apache2/apache2.conf
```

```
ServerName localhost
```

6. edit file utama : (jangan lupa backup)

```
<Directory /var/www/html/home>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
        DirectoryIndex info.php
</Directory>
```

```
echo "<?php phpinfo(); ?>" | sudo tee /var/www/html/home/info.php
```

7. Cek syntax dulu:

```
sudo apache2ctl configtest
```

8. Kalau muncul `Syntax OK`, restart Apache:

```
sudo systemctl restart apache2
```

9. Lalu buka di browser
