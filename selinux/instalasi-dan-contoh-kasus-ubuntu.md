# Instalasi dan Contoh kasus (Ubuntu)

#### Instalasi dan Contoh kasus

1. Hentikan AppArmor

```
sudo systemctl stop apparmor
```

2. Nonaktifkan agar tidak menyala saat boot

```
sudo systemctl disable apparmor
```

3. Hapus paket AppArmor

```
sudo apt purge apparmor
```

4. Install Selinux

```
sudo apt update
```

```
sudo apt install selinux-basics selinux-policy-default auditd policycoreutils
```

* `selinux-basics`: Paket dasar.
* `selinux-policy-default`: Kebijakan keamanan default.
* `auditd`: Daemon untuk mencatat log penolakan SELinux.
* `policycoreutils`: Utilitas untuk mengelola SELinux (seperti `semanage` dan `restorecon`).

5. Aktifkan Selinux

```
sudo selinux-activate
```

6. Selesai dan reboot

```
sudo reboot
```

7. Setelah server kembali online, periksa status SELinux.

```
sestatus
```

Awalnya, statusnya mungkin `permissive`. Mode `permissive` berarti SELinux akan _mencatat_ pelanggaran tetapi _tidak akan memblokirnya_. Ini bagus untuk debugging.

Untuk skenario ini, kita ingin melihat pemblokiran terjadi. Jadi, kita akan mengaturnya ke mode `enforcing`.

```
sudo setenforce 1
```

Sekarang kita siapkan Nginx untuk menyajikan file dari lokasi non-standar, yang pasti akan ditolak oleh SELinux.

```
sudo apt install nginx
```

```
sudo mkdir -p /srv/testdeny
```

```
echo "Halo dari /srv/testdeny (SELinux di Ubuntu)" | sudo tee /srv/my-website/index.html
```

```
sudo nano /etc/nginx/sites-available/default
```

Di dalam editor `nano`, ubah baris `root` dari `/var/www/html` menjadi direktori baru kita:

```
# Ganti ini:
root /var/www/html;

# Menjadi ini:
root /srv/testdeny;
```

Simpan dan restart service nginx

```
sudo systemctl restart nginx
```

```
curl localhost
```

Anda tidak akan melihat pesan "Halo...". Anda akan mendapatkan 403 Forbidden. Ini bukan masalah izin file, ini adalah blokir dari SELinux.

Mari kita buktikan. Kita akan periksa log audit SELinux (`auditd`):

Begitu SELinux aktif, jalankan:

```
sudo ausearch -m AVC -c nginx
```

Anda akan melihat output yang rumit, tetapi intinya adalah: `avc: denied { read } for pid=... comm="nginx" name="index.html" ... scontext=... tcontext=...`

Ini membuktikan SELinux (bukan izin file) memblokir Nginx (`comm="nginx"`) untuk membaca (`denied { read }`) file tersebut.

Masalahnya adalah direktori `/srv/testdeny` memiliki label SELinux yang salah. Mari kita periksa:

```
ls -Z /srv
```

Anda mungkin akan melihat label seperti `default_t` atau `var_t`. Nginx (yang berjalan dengan konteks `httpd_t`) tidak diizinkan membaca label ini. Label yang benar untuk konten web adalah `httpd_sys_content_t`.

Kita harus memberi tahu SELinux aturan baru ini:

1. Tetapkan Aturan Konteks (Policy): Beri tahu SELinux bahwa semua yang ada di `/srv/my-website` (dan di dalamnya `(/.*)?`) harus memiliki label `httpd_sys_content_t`.

```
sudo semanage fcontext -a -t httpd_sys_content_t "/srv/testdeny(/.*)?"
```

2. Terapkan Aturan (Restore Context): Perintah di atas hanya mengubah _aturan_. Perintah ini _menerapkan_ aturan tersebut ke file sistem yang sebenarnya.

```
sudo restorecon -Rv /srv/testdeny
```

Outputnya akan menunjukkan bahwa label telah diubah: `restorecon reset /srv/testdeny context ... -> ...:s0:httpd_sys_content_t`

Sekarang label file sudah benar, SELinux akan mengizinkan Nginx membacanya.

```
# Restart Nginx untuk memastikan
sudo systemctl restart nginx

# Coba akses lagi
curl localhost
```

Output Sukses:

```
Halo dari /srv/testdeny (SELinux di Ubuntu)
```

#### Kesimpulan

Anda telah berhasil:

1. Mengganti AppArmor dengan SELinux di Ubuntu (langkah yang sangat tidak biasa).
2. Mendiagnosis masalah Nginx 403 sebagai penolakan SELinux.
3. Memperbaiki masalah tersebut dengan mengubah konteks keamanan file menggunakan `semanage` dan `restorecon`.
