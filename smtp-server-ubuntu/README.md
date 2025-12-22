# SMTP Server (Ubuntu)

#### 1. Pengertian SMTP Server

**SMTP** (Simple Mail Transfer Protocol) adalah **protokol standar Internet** yang digunakan untuk **mengirim dan meneruskan email** antar server.\
SMTP bekerja di **layer aplikasi (Application Layer)** dari model OSI, menggunakan **port 25**, **587**, atau **465 (SSL/TLS)**.

➡️ **SMTP Server** adalah **server yang menjalankan protokol SMTP**, berfungsi untuk:

* Mengirim email dari **klien email (MUA / Mail User Agent)** seperti Outlook, Thunderbird, atau webmail ke server tujuan.
* Meneruskan (relay) email antar **Mail Transfer Agent (MTA)** sampai mencapai **Mail Delivery Agent (MDA)** di sisi penerima.

Singkatnya:

> SMTP Server = "pos pengirim" yang bertugas **mengirim dan meneruskan surat elektronik** ke tujuan.

***

#### 2. Arsitektur SMTP Server

Arsitektur SMTP umumnya terdiri dari **3 komponen utama** dan **2 jenis agen email**:

#### 💡 Komponen Utama

| Komponen                      | Keterangan                                                                                            |
| ----------------------------- | ----------------------------------------------------------------------------------------------------- |
| **MUA (Mail User Agent)**     | Aplikasi pengguna seperti Outlook, Thunderbird, atau webmail (mengirim/menerima email).               |
| **MTA (Mail Transfer Agent)** | Server yang mengatur pengiriman email dari pengirim ke penerima (contohnya: Postfix, Sendmail, Exim). |
| **MDA (Mail Delivery Agent)** | Menyimpan dan mengantarkan email ke mailbox penerima (contohnya: Dovecot, Courier).                   |

<figure><img src="../.gitbook/assets/image (633).png" alt=""><figcaption></figcaption></figure>

#### 3. Instalasi, Konfigurasi serta testing by lokal SMTP Server

Berikut contoh pada **Ubuntu Server (Debian-based)** menggunakan **Postfix**.

Langkah-langkah Instalasi Postfix

1. Install paket

```
su -
```

```
apt update
```

```
apt install postfix -y
```

<figure><img src="../.gitbook/assets/image (551).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (552).png" alt=""><figcaption></figcaption></figure>

Selama instalasi akan muncul konfigurasi interaktif:

* Pilih **"Internet Site"**
* Masukkan nama domain (contoh: `mail.cinosta.com`)

```
mail.cinosta.com
```

<figure><img src="../.gitbook/assets/image (553).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (554).png" alt=""><figcaption></figcaption></figure>

2. Verifikasi Service

```
systemctl status postfix
```

<figure><img src="../.gitbook/assets/image (555).png" alt=""><figcaption></figcaption></figure>

Pastikan statusnya: **active (running)**

3. Cek konfigurasi utama

```
nano /etc/postfix/main.cf
```

Contoh konfigurasi minimal :

```
myhostname = mail.cinosta.local
mydomain = cinosta.local
myorigin = $mydomain
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain, mail.cinosta.com
relayhost =
inet_interfaces = all
inet_protocols = ipv4
home_mailbox = Maildir/
```

<figure><img src="../.gitbook/assets/image (556).png" alt=""><figcaption></figcaption></figure>

Ubah bagian dibawah ini

```
nano /etc/mailname
```

Isi dengan:

```
cinosta.local
```

(Sekarang Postfix akan menganggap email lokal: `cikal@cinosta.local`)

Simpan dan restart:

```
systemctl restart postfix
```

<figure><img src="../.gitbook/assets/image (557).png" alt=""><figcaption></figcaption></figure>

Install mailutils untuk mengirim email via command line:

```
apt install mailutils -y
```

<figure><img src="../.gitbook/assets/image (558).png" alt=""><figcaption></figcaption></figure>

Install rsyslog untuk menyimpan log mail di /var/log/mail.log

```
apt install rsyslog -y
```

<figure><img src="../.gitbook/assets/image (559).png" alt=""><figcaption></figcaption></figure>

Cek status rsyslog

```
systemctl status rsyslog
```

<figure><img src="../.gitbook/assets/image (560).png" alt=""><figcaption></figcaption></figure>

Dan edit konfigurasi rsyslog seperti gambar dibawah ini:

```
nano /etc/rsyslog.d/50-default.conf
```

Pastikan baris berikut **tidak dikomentari (#)**:

```
mail.*                          -/var/log/mail.log
```

<figure><img src="../.gitbook/assets/image (561).png" alt=""><figcaption></figcaption></figure>

```
sudo systemctl restart rsyslog
```

Install Mutt untuk mendukung Maildir (Mail Client)

```
apt install mutt -y
```

Buat konfigurasi Mutt agar tahu lokasi mailbox sebenarnya:

```
sudo -u cikal nano /home/cikal/.muttrc
```

Lalu tambahkan isi berikut:

```
set mbox_type=Maildir
set folder="~/Maildir"
set spoolfile="~/Maildir"
set record="+.Sent"
set postponed="+.Drafts"
set move=no
```

Simpan dan keluar

4. Uji kirim email secara lokal

Kirim email uji :

```
echo "Tes ke user lokal" | mail -s "Tes SMTP lokal" cikal
```

Setelah kita jalankan perintah diatas, kita bisa melihat hasilnya melalui log, ketikan perintah berikut :

```
tail -f /var/log/mail.log
```

<figure><img src="../.gitbook/assets/image (562).png" alt=""><figcaption></figcaption></figure>

Bisa cek menggunakan log maupun dari Mutt (Mail Client)

```
sudo -u cikal mutt
```

<figure><img src="../.gitbook/assets/image (563).png" alt=""><figcaption></figcaption></figure>

ketik q jika ingin keluar



Test jika ingin mengirim kan mail dari VM Server ke VM Client, seperti berikut :

Install postfix, mailutils dan telnet untuk keperluan VM Client

```
apt install postfix -y
```

<figure><img src="../.gitbook/assets/image (551).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (552).png" alt=""><figcaption></figcaption></figure>

Selama instalasi akan muncul konfigurasi interaktif:

* Pilih **"Internet Site"**
* Masukkan nama domain (contoh: `mail.cinosta.local`)

```
mail.cinosta.local
```

<figure><img src="../.gitbook/assets/image (553).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (554).png" alt=""><figcaption></figcaption></figure>

2. Verifikasi Service

```
systemctl status postfix
```

<figure><img src="../.gitbook/assets/image (555).png" alt=""><figcaption></figcaption></figure>

Pastikan statusnya: **active (running)**

```
apt install mailutils -y
```

```
apt install telnet -y
```

Install rsyslog untuk menyimpan log mail di /var/log/mail.log

```
apt install rsyslog -y
```

<figure><img src="../.gitbook/assets/image (559).png" alt=""><figcaption></figcaption></figure>

Cek status rsyslog

```
systemctl status rsyslog
```

<figure><img src="../.gitbook/assets/image (560).png" alt=""><figcaption></figcaption></figure>

Dan edit konfigurasi rsyslog seperti gambar dibawah ini:

```
nano /etc/rsyslog.d/50-default.conf
```

Pastikan baris berikut **tidak dikomentari (#)**:

```
mail.*                          -/var/log/mail.log
```

<figure><img src="../.gitbook/assets/image (561).png" alt=""><figcaption></figcaption></figure>

```
sudo systemctl restart rsyslog
```

Edit file /etc/hosts, dan tambahkan ip server berikut :

```
nano /etc/hosts
```

```
192.168.56.16   mail.cinosta.com mail
```

Masukkan relayhost dengan perintah berikut :

```
sudo postconf -e 'relayhost = [mail.cinosta.com]'
```

Bisa juga dengan buka file dibawah ini

```
nano /etc/postfix/main.cf
```

```
systemctl restart postfix
```

Tambahkan ip client /etc/hosts di VM Server

```
nano /etc/hosts
```

```
192.168.56.17   client.cinosta.com client
```

Buka juga firewall 25 di VM Server

```
ufw allow 25
```

Bisa test menggunakan perintah telnet jika firewall 25/tcp yg di VM Server sudah di allow

```
telnet mail.cinosta.com 25
```

Setelah semua terkonfigurasi kita bisa mengirimkan mail dari client menuju server dengan menjalankan perintah berikut

```
echo "Tes kirim dari client ke server SMTP" | mail -s "Tes SMTP antar VM" cikal@mail.cinosta.local
```

Jika berhasil, kemudian cek log dari sisi Server maka akan seperti dibawah ini :

```
tail -f /var/log/mail.log
```

<figure><img src="../.gitbook/assets/image (564).png" alt=""><figcaption></figcaption></figure>

Kalau koneksi dan konfigurasi sudah benar, di **server** nanti akan muncul log seperti:

postfix/smtpd\[xxxx]: connect from client.cinosta.local\[192.168.56.11]\
postfix/smtpd\[xxxx]: ... status=sent (delivered to maildir)

JIka ingin menggunakan iredmail, sebagai berikut panduannya :

