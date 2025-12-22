# Kendala iRedMail (Ubuntu)

Karena kondisi saat ini saya sudah punya VM Server yang sudah ter install iRedMail (penerima) dan ingin test kirim mail dari VM Client lainnya (pengirim) dan terjadi error tidak dapat mengirim dari VM Client ke VM Server (penerima)

Berikut kesimpulan eksperimen percobaan saya mengenai kendala VM Client (pengirim) tidak ada mengirim mail hingga VM Client (pengirim) dapat mengirimkan mail dari VM Server yang sudah ada iRedMail :

## Dari sisi VM Client

1. Edit konfigurasi /etc/postfix/main.cf dan fokus pada bagian bawah karena hanya disitu adanya perubahan

```
nano /etc/postfix/main.cf
```

Fokus pada bagian `"myhostname = client.cinosta.com"` , `"mynetworks = 127.0.0.0/8 192.168.56.0/24"` , dan `'relayhost = [mail.cinosta.com]'`

```
root@client:~# cat /etc/postfix/main.cf
# See /usr/share/postfix/main.cf.dist for a commented, more complete version


# Debian specific:  Specifying a file name will cause the first
# line of that file to be used as the name.  The Debian default
# is /etc/mailname.
#myorigin = /etc/mailname

smtpd_banner = $myhostname ESMTP $mail_name (Ubuntu)
biff = no

# appending .domain is the MUA's job.
append_dot_mydomain = no

# Uncomment the next line to generate "delayed mail" warnings
#delay_warning_time = 4h

readme_directory = no

# See http://www.postfix.org/COMPATIBILITY_README.html -- default to 3.6 on
# fresh installs.
compatibility_level = 3.6



# TLS parameters
smtpd_tls_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
smtpd_tls_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
smtpd_tls_security_level=may

smtp_tls_CApath=/etc/ssl/certs
smtp_tls_security_level=may
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache


smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination
myhostname = client.cinosta.com
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
myorigin = /etc/mailname
mydestination = $myhostname, client.cinosta.local, cinosta, localhost.localdomain, localhost
relayhost = [mail.cinosta.com]
mynetworks = 127.0.0.0/8 192.168.56.0/24
mailbox_size_limit = 0
recipient_delimiter = +
inet_interfaces = all
inet_protocols = all
root@client:~#

```

2. Kemudian cek /etc/hosts/

```
nano /etc/hosts
```

```
root@client:~# cat /etc/hosts
127.0.0.1 localhost
#127.0.1.1 cinosta
192.168.56.11   client.cinosta.com client
192.168.56.10   mail.cinosta.com mail

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
root@client:~#
```

3. Kemudian restart service postfix

```
systemctl restart postfix
```

## Dari sisi VM Server iRedMail (penerima)

1. Edit konfigurasi /etc/postfix/main.cf dan fokus pada bagian bawah karena hanya disitu adanya perubahan

```
nano /etc/postfix/main.cf
```

Fokus pada bagian `"mynetworks = 127.0.0.0/8 192.168.56.0/24"` , dan `'relayhost = [mail.cinosta.com]'`

```
postconf -e "mynetworks = 127.0.0.0/8 192.168.56.0/24"
```

```
postconf -e 'relayhost = [mail.cinosta.com]'
```

```
systemctl restart postfix
```

```
ping -c 2 client.cinosta.com
```

```
nano /opt/iredapd/settings.py
```

```
ALLOWED_LOGIN_MISMATCH_SENDERS = ['root@client.cinosta.com']
ALLOWED_LOGIN_MISMATCH_SENDERS = ['@cinosta.com']
```

```
systemctl restart iredapd
```

2. Jika service PHP mati cek seperti perintah dibawah

```
sudo systemctl status nginx
sudo systemctl status php8.1-fpm
sudo systemctl status uwsgi
sudo apt update
sudo apt install php8.2-fpm -y
sudo systemctl enable --now php8.2-fpm
sudo add-apt-repository universe
sudo apt update
apt search php | grep fpm
sudo apt install php8.3-fpm -y
sudo systemctl enable --now php8.3-fpm
sudo systemctl status php8.3-fpm
sudo systemctl restart nginx uwsgi

```

Sesuaikan juga file dibawah ini

```
nano /etc/nginx/sites-enabled/00-default-ssl.conf
```

```
root@mail:~# cat /etc/nginx/sites-enabled/00-default-ssl.conf
#
# Note: This file must be loaded before other virtual host config files,
#
# HTTPS
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name _;

    root /var/www/html;
    index index.php index.html;

    include /etc/nginx/templates/misc.tmpl;
    include /etc/nginx/templates/ssl.tmpl;
    include /etc/nginx/templates/iredadmin.tmpl;
    include /etc/nginx/templates/roundcube.tmpl;
    include /etc/nginx/templates/sogo.tmpl;
    include /etc/nginx/templates/netdata.tmpl;
    include /etc/nginx/templates/php-catchall.tmpl;
    include /etc/nginx/templates/stub_status.tmpl;
}
root@mail:~#

```

3. Fokus pada bagian file dibawah ini (# Handle PHP files with upstream handler)

```
nano /etc/nginx/templates/fastcgi_php.tmpl
```

```
root@mail:~# cat /etc/nginx/templates/fastcgi_php.tmpl
#
# Template used to handle PHP fastcgi applications
#
# You still need to define `SCRIPT_FILENAME` for your PHP application, and
# probably `fastcgi_index` if your application use different index file.
#
include fastcgi_params;

# Directory index file
fastcgi_index index.php;

# Handle PHP files with upstream handler
#fastcgi_pass php_workers;
#fastcgi_pass unix:/run/php/php8.3-fpm.sock;
fastcgi_pass 127.0.0.1:9999;

# Fix the HTTPROXY issue.
# Reference: https://httpoxy.org/
fastcgi_param HTTP_PROXY '';
root@mail:~#

```

```
systemctl restart php8.3-fpm nginx
```

```
echo -e "GET /index.php HTTP/1.0\r\n\r\n" | sudo socat - UNIX-CONNECT:/run/php/php8.3-fpm.sock
```

```
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw reload
sudo ufw status
```

4. Untuk Cek

```
curl -I http://localhost/
```

```
curl -I http://127.0.0.1
```

```
curl -v 127.0.0.1:9999
```

```
ss -tulpn | grep nginx
```



```
sudo tail -n 20 /var/log/nginx/access.log
```

```
sudo tail -n 20 /var/log/nginx/error.log
```

```
sudo grep -E '^listen' /etc/php/*/fpm/pool.d/www.conf
```



Dibawah ini hasil diskusi dengan CHATGPT

Bagus — berikut ringkasan kronologis **lengkap & terperinci** dari semua kendala yang kamu temui sejak awal sampai kondisi final, termasuk diagnosa, perintah yang dijalankan, perubahan konfigurasi, dan hasilnya. Bisa kamu pakai langsung untuk dokumentasi atau catatan runbook.

## Ringkasan singkat (1–2 baris)

Mulai dari masalah SMTP (Postfix menolak HELO non-FQDN), lalu masalah logging, resolusi nama, kebijakan iRedMail (iRedAPD) yang menolak, sampai akhirnya memperbaiki Nginx↔PHP-FPM + firewall sehingga iRedMail web/roundcube bisa diakses. Semua issue diatasi langkah demi langkah.

***

## Urutan masalah → penyebab → tindakan (detail kronologis)

### 1) SMTP: `504 5.5.2 Helo command rejected: need fully-qualified hostname`

* **Symptom / log**\
  `NOQUEUE: reject: RCPT ... 504 5.5.2 <cinosta>: Helo command rejected: need fully-qualified hostname; helo=<cinosta>`
* **Penyebab**\
  Hostname client pendek (`cinosta`) — Postfix di penerima menolak HELO yang bukan FQDN.
*   **Tindakan & perintah**\
    Di VM client:

    ```bash
    sudo hostnamectl set-hostname vmdua.cinosta.com   # (kemudian ke client.cinosta.com)
    hostname; hostname -f
    sudo postconf -e "myhostname = vmdua.cinosta.com" # lalu diganti menjadi client.cinosta.com
    sudo systemctl restart postfix
    ```

    Memperbaiki `/etc/hosts` pada client (hapus `127.0.1.1 cinosta`, tambahkan FQDN):

    ```
    127.0.0.1 localhost
    192.168.56.11 vmdua.cinosta.com vmdua
    192.168.56.10 mail.cinosta.com mail
    ```
* **Hasil**\
  HELO berubah jadi `client.cinosta.com` — tapi muncul penolakan baru terkait domain `.local` atau resolusi.

***

### 2) Penolakan karena domain `.local` / Host not found

* **Symptom / log**\
  `554 5.7.1 <client.cinosta.local>: ... does not identify itself correctly (.local)` → atau `450 4.7.1 ... Helo command rejected: Host not found`
* **Penyebab**\
  Server penerima melakukan reverse/forward lookup dan tidak menemukan `client.cinosta.com` (pada awalnya client menggunakan `.local` dan server hosts berisi `client.cinosta.local`).
*   **Tindakan**\
    Perbaiki `/etc/hosts` di server penerima:

    ```
    192.168.56.11 vmdua.cinosta.com vmdua
    192.168.56.10 mail.cinosta.com mail
    ```

    Verifikasi dari server penerima:

    ```bash
    ping -c 2 client.cinosta.com
    ```
* **Hasil**\
  Server sekarang bisa resolve hostname client → HELO diterima (atau berkurang error terkait HELO).

***

### 3) Kebijakan iRedMail / iRedAPD — `451/451 4.7.1 Intentional policy rejection`

* **Symptom / log**\
  `451 4.7.1 <cikal@mail.cinosta.com>: Recipient address rejected: Intentional policy rejection, please try again later`
* **Penyebab**\
  iRedAPD (policy daemon) atau plugin seperti `greylisting`, `reject_sender_login_mismatch`, atau throttle menolak pengiriman dari IP/host yang belum dianggap trusted.
* **Tindakan yang diambil**
  *   Buat perubahan di sisi **Postfix (server)**: tandai jaringan internal sebagai trusted:

      ```bash
      sudo postconf -e "mynetworks = 127.0.0.0/8 192.168.56.0/24"
      sudo systemctl restart postfix
      ```
  *   Edit iRedAPD conf jika perlu (`/opt/iredapd/settings.py`) — tambahkan jaringan ke `MYNETWORKS` atau tweak `plugins`.\
      Contoh perubahan:

      ```python
      MYNETWORKS = ['127.0.0.1', '192.168.56.0/24']
      # atau hapus/adjust greylisting/plugin jika untuk lab
      ```
  *   Restart iredapd:

      ```bash
      sudo systemctl restart iredapd
      ```
* **Hasil**\
  Setelah menambahkan network ke `mynetworks` dan/atau `MYNETWORKS`, `postfix` menerima koneksi dan Postfix log menunjukkan `ALLOWLISTED` dan sesi SMTP normal.

***

### 4) Logging: `/var/log/mail.log` tidak ada

* **Symptom**\
  `tail: cannot open '/var/log/mail.log' for reading: No such file or directory`
* **Penyebab**\
  rsyslog tidak mengarahkan log, atau distro menggunakan journalctl saja. Or iRedMail/Ubuntu setup belum menulis ke file.
* **Tindakan**
  *   Periksa `rsyslog`:

      ```bash
      sudo systemctl status rsyslog
      sudo nano /etc/rsyslog.d/50-default.conf
      ```
  *   Pastikan baris mail.\* diarahkan ke `/var/log/mail.log`:

      ```
      mail.info    -/var/log/mail.info
      mail.*       -/var/log/mail.log
      ```
  * Restart `rsyslog` dan coba kirim mail untuk membangkitkan file.
  *   Alternatif baca via journal:

      ```bash
      sudo journalctl -u postfix -f
      ```
* **Hasil**\
  `mail.log` tersedia dan berisi events Postfix.

***

### 5) Mail delivery checks & mailbox

*   **Perintah yang digunakan untuk tes**\
    Di client:

    ```bash
    echo "Tes kirim email antar VM" | mail -s "Test Mail" cikal@mail.cinosta.com
    tail -f /var/log/mail.log   # di client dan di server
    ```
*   **Perintah verifikasi di server**

    ```bash
    sudo postqueue -p        # atau mailq
    ls /var/vmail/mail.cinosta.com/cikal/Maildir/new/
    sudo mysql vmail -e "SELECT username FROM mailbox;"
    ```
*   **Tindakan**\
    Jika mailbox belum ada → buat mailbox `cikal`:

    ```bash
    sudo iredmailcli --create-mailbox cikal@mail.cinosta.com --password 'Password123' --name 'Cikal User'
    ```
* **Hasil**\
  Mail diterima ke queue, maildir dibuat bila mailbox ada; jika mailbox dibuat, email masuk ke `/var/vmail/.../Maildir/new/`.

***

### 6) Web UI (iRedAdmin / Roundcube) tidak bisa dibuka — masalah Nginx↔PHP

* **Symptom awal**\
  `https://mail.cinosta.com/mail/` dan `.../iredadmin/` tidak dapat buka; php-fpm unit tidak ditemukan (php8.1-fpm missing).
* **Penyebab berlapis**
  * PHP-FPM versi yang diperlukan tidak terpasang / socket/config salah.
  * Template Nginx mengarah ke upstream yang tak ada (`php_workers`) atau socket versi lama.
  * `fastcgi_pass` tidak menunjuk ke socket/port yang aktif.
* **Tindakan berurutan yang dilakukan**
  1.  Pasang versi PHP-FPM yang ada di repos (Ubuntu Noble → php8.3):

      ```bash
      sudo add-apt-repository universe
      sudo apt update
      sudo apt install php8.3-fpm -y
      sudo systemctl enable --now php8.3-fpm
      ```
  2. Periksa `pool.d/www.conf` untuk `listen`:
     * Awalnya ditemukan: `listen = 127.0.0.1:9999` (TCP), bukan unix socket.
  3. Sesuaikan Nginx template:
     *   Edit `/etc/nginx/templates/fastcgi_php.tmpl` — ubah `fastcgi_pass` agar cocok:

         ```nginx
         include fastcgi_params;
         fastcgi_index index.php;
         fastcgi_pass 127.0.0.1:9999;
         fastcgi_param HTTP_PROXY '';
         ```
     * `php-catchall.tmpl` menyertakan `fastcgi_php.tmpl` dan men-`fastcgi_param SCRIPT_FILENAME` agar PHP tahu path.
  4.  Restart layanan:

      ```bash
      sudo systemctl restart php8.3-fpm nginx
      ```
  5.  Buat file test `info.php`:

      ```bash
      echo "<?php phpinfo(); ?>" | sudo tee /var/www/html/info.php
      ```

      -> tes via `http://localhost/info.php` atau `http://192.168.56.10/info.php`.
* **Hasil**\
  Setelah menyelaraskan `fastcgi_pass` dengan `listen` FPM (127.0.0.1:9999), Nginx memproses PHP dengan benar dan Roundcube/iRedAdmin bisa diakses (setelah masalah jaringan/firewall diselesaikan).

***

### 7) Nginx templates detail yang diperbaiki

* File diperiksa & diubah:
  * `/etc/nginx/templates/php-catchall.tmpl` — memastikan `include /etc/nginx/templates/fastcgi_php.tmpl;` dan `SCRIPT_FILENAME` ada.
  * `/etc/nginx/templates/fastcgi_php.tmpl` — ganti `fastcgi_pass` dari `php_workers` atau `unix:...sock` ke `127.0.0.1:9999` (atau ke socket jika FPM listen socket).
*   Pernyataan utama yang disimpan:

    ```nginx
    fastcgi_pass 127.0.0.1:9999;
    include fastcgi_params;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    ```

***

### 8) Firewall / jaringan host → VM

* **Masalah**: browser host tidak bisa akses web karena UFW belum mengizinkan port 80/443.
*   **Tindakan**:

    ```bash
    sudo ufw allow 80/tcp
    sudo ufw allow 443/tcp
    sudo ufw reload
    ```
* **Hasil**: setelah membuka port 80 (dan 443), host Windows bisa mengakses `http://192.168.56.10` dan `https://mail.cinosta.com`.

***

## Perintah penting yang kamu jalankan (ringkas untuk catatan)

(Susun ini sebagai checklist langkah yang sudah kamu pakai)

#### Hostname / SMTP

```bash
sudo hostnamectl set-hostname vmdua.cinosta.com
sudo postconf -e "myhostname = vmdua.cinosta.com"
sudo postconf -e "mynetworks = 127.0.0.0/8 192.168.56.0/24"   # di mail server
sudo systemctl restart postfix
```

#### /etc/hosts (client & server)

Client `/etc/hosts`:

```
127.0.0.1 localhost
192.168.56.11 vmdua.cinosta.com vmdua
192.168.56.10 mail.cinosta.com mail
```

Server `/etc/hosts`:

```
127.0.0.1 localhost
192.168.56.10 mail.cinosta.com mail
192.168.56.11 vmdua.cinosta.com vmdua
```

#### iRedAPD (policy)

```bash
sudo nano /opt/iredapd/settings.py
# tambahkan/ubah: MYNETWORKS = ['127.0.0.1', '192.168.56.0/24']
sudo systemctl restart iredapd
```

#### Mail tests

```bash
echo "Tes kirim email antar VM" | mail -s "Test Mail" cikal@mail.cinosta.com
sudo tail -f /var/log/mail.log
sudo postqueue -p
sudo ls /var/vmail/mail.cinosta.com/cikal/Maildir/new/
```

#### PHP-FPM & Nginx

```bash
sudo add-apt-repository universe
sudo apt update
sudo apt install php8.3-fpm -y
sudo systemctl enable --now php8.3-fpm
# edit /etc/nginx/templates/fastcgi_php.tmpl -> fastcgi_pass 127.0.0.1:9999;
sudo systemctl restart php8.3-fpm nginx
echo "<?php phpinfo(); ?>" | sudo tee /var/www/html/info.php
```

#### Firewall

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw reload
```

***

## Status akhir (ringkas, lengkap)

* Postfix (iRedMail) menerima email antar-VM (client → mail server) tanpa error HELO/Policy. ✔
* iRedAPD diatur untuk mengenali jaringan internal (greylisting/whitelist diatur). ✔
* Mailbox `cikal` & `novita` ada di DB `vmail` (atau dibuat bila belum ada). ✔
* Nginx dan PHP-FPM (php8.3-fpm listening 127.0.0.1:9999) terhubung; PHP berjalan. ✔
* Roundcube / iRedAdmin (web UI) dapat diakses dari host Windows melalui `http[s]://mail.cinosta.com` atau `http://192.168.56.10`. ✔
* Firewall (UFW) mengizinkan port 22, 25, 80, 443. ✔

***

## Rekomendasi & next steps (opsional tapi disarankan)

1. **Hapus file `info.php`** setelah verifikasi (keamanan):\
   `sudo rm /var/www/html/info.php`
2. **Backup konfigurasi** yang sudah kamu ubah (`/etc/nginx/templates/*`, `/opt/iredapd/settings.py`, `/etc/hosts`, `/etc/postfix/main.cf`) ke repos atau tarball.
3. **Kembalikan kebijakan iRedAPD/greylisting** bila kamu mematikan plugin untuk pengujian — aktifkan kembali untuk environment production.
4. **Atur DNS / PTR** jika server ini akan di-expose publik — jangan pakai `.local` di production.
5. **Document** password akun yang dibuat, dan put user creation steps di runbook.
6. Pertimbangkan pasang `logwatch` / monitoring untuk memantau deliverability.
