# Instalasi dan Konfigurasi FTP Server (Ubuntu)

Di sini kita akan menggunakan contoh perangkat lunak FTP Server yang paling populer, aman, dan ringan di Linux (Ubuntu/Debian), yaitu vsftpd (Very Secure FTP Daemon).

#### Langkah 1: Instalasi vsftpd

1. Update Repository Paket: Buka terminal Anda dan jalankan perintah ini untuk memastikan daftar paket Anda adalah yang terbaru.

```
sudo apt update
```

2. Install vsftpd: Jalankan perintah untuk menginstal paket `vsftpd`.

```
sudo apt install vsftpd
```

3. Cek Status Layanan: Setelah instalasi, layanan vsftpd seharusnya sudah berjalan. Anda bisa memeriksanya dengan:

```
sudo systemctl status vsftpd
```

Anda akan melihat status `active (running)`.

Langkah 2: Backup File Konfigurasi

Sebelum mengedit apapun, selalu buat cadangan file konfigurasi aslinya.

```
sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.bak
```

Langkah 3: Konfigurasi vsftpd

Sekarang, kita akan mengedit file konfigurasi utama menggunakan editor teks seperti `nano`.

```
sudo nano /etc/vsftpd.conf
```

Salin skrip dibawah dan pindahkan pada file vsftpd.conf

```
# --- Keamanan Dasar ---

# Memberitahu vsftpd untuk berjalan sendiri (mode standalone)
listen=YES

# Menonaktifkan login anonim (pengguna 'ftp' or 'anonymous').
anonymous_enable=NO

# Mengizinkan pengguna lokal (user yang ada di sistem Linux) untuk login.
local_enable=YES

# --- Mengizinkan Upload/Perubahan File ---

# Mengizinkan perintah untuk mengubah file (upload, hapus, ganti nama, buat folder).
write_enable=YES

# --- Chroot Jail (SANGAT PENTING) ---

# "Mengurung" semua pengguna lokal di dalam direktori home mereka masing-masing.
# Mereka tidak akan bisa pindah (cd) ke direktori di luarnya.
chroot_local_user=YES

# (PENTING!) Mengizinkan "kurungan" (home directory) pengguna untuk bisa ditulisi.
# Diperlukan jika chroot_local_user=YES, tapi sedikit mengurangi keamanan.
allow_writeable_chroot=YES

# --- Konfigurasi Passive Mode (PENTING UNTUK FIREWALL) ---

# Mengaktifkan mode pasif. Penting agar FTP bisa diakses dari belakang firewall/NAT.
pasv_enable=YES

# Port terendah yang akan digunakan untuk koneksi data pasif.
pasv_min_port=40000

# Port tertinggi yang akan digunakan untuk koneksi data pasif.
# Pastikan range port ini (40000-41000) Anda buka di firewall.
pasv_max_port=41000

# --- Pengaturan Tambahan (Opsional) ---

# Memaksa server untuk menampilkan file/folder yang diawali titik (misal: .htaccess).
force_dot_files=YES

# Menampilkan waktu (timestamp) file dalam zona waktu lokal server (bukan GMT).
use_localtime=YES

# Mengaktifkan pencatatan log untuk setiap aktivitas transfer file (upload/download).
xferlog_enable=YES

# Menentukan lokasi file log transfer (harus sesuai dengan baris di atas).
xferlog_file=/var/log/vsftpd.log

# --- Pengaturan Penting dari File Asli ---

# Menentukan lokasi direktori "penjara" kosong yang aman untuk proses internal vsftpd.
secure_chroot_dir=/var/run/vsftpd/empty

# Menentukan nama file konfigurasi PAM yang digunakan untuk autentikasi pengguna.

# (File ini biasanya /etc/pam.d/vsftpd)
pam_service_name=vsftpd

# Menonaktifkan koneksi aman (FTPS via SSL/TLS).
# Set 'YES' jika Anda ingin mengaktifkan enkripsi (disarankan).
ssl_enable=NO

```

Setelah selesai mengedit, simpan file (di `nano`: `Ctrl+O`, `Enter`) dan keluar (`Ctrl+X`).

Langkah 4: Restart Layanan vsftpd

Agar perubahan konfigurasi diterapkan, restart layanan vsftpd.

```
sudo systemctl restart vsftpd
```

Langkah 5: Konfigurasi Firewall (UFW)

Jika Anda menggunakan firewall (seperti UFW di Ubuntu), Anda harus membuka port yang diperlukan.

1. Buka port 21 (Control Channel):

```
sudo ufw allow 21/tcp
```

2. Buka port 20 (Untuk Active Mode, opsional jika hanya pasif):

```
sudo ufw allow 20/tcp
```

3. Buka rentang port Pasif (yang Anda tentukan di `.conf`):

```
sudo ufw allow 40000:41000/tcp
```

4. Cek status UFW:

```
sudo ufw status
```

Anda akan melihat port-port di atas dalam daftar `ALLOW`.

Langkah 6: Membuat Pengguna FTP

Anda tidak boleh login menggunakan user `root`. Sebaiknya buat pengguna khusus untuk FTP.

1. Buat user baru (misalnya `ftpuser`):

```
sudo adduser ftpuser
```

Anda akan diminta untuk mengatur password baru untuk pengguna ini.

Changing the user information for ftpuser\
Enter the new value, or press ENTER for the default\
Full Name \[]: <--- Tekan ENTER\
Room Number \[]: <--- Tekan ENTER\
Work Phone \[]: <--- Tekan ENTER\
Home Phone \[]: <--- Tekan ENTER\
Other \[]: <--- Tekan ENTER\
Is the information correct? \[Y/n] Y

2. (Opsional) Membatasi user hanya untuk FTP: Jika Anda ingin `ftpuser` ini _hanya_ bisa login via FTP dan tidak bisa login via SSH (login shell), Anda bisa mengubah shell mereka:

```
sudo usermod -s /sbin/nologin ftpuser
```

Direktori home pengguna ini (misal `/home/ftpuser`) adalah tempat di mana mereka akan "dikurung" (chroot) saat login FTP.

Langkah 7: Pengujian

Sekarang server FTP Anda sudah siap. Anda bisa mengujinya menggunakan FTP Client (seperti FileZilla):

* Host: Alamat IP server Anda (misal `192.168.1.10`)
* Username: `ftpuser` (atau user yang Anda buat)
* Password: Password yang Anda atur untuk user tersebut
* Port: `21`

Jika berhasil, Anda akan login dan berada di dalam direktori `/home/ftpuser` dan tidak bisa keluar dari situ. Anda bisa mencoba mengunggah atau mengunduh file.
