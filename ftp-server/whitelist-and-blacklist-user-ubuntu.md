# Whitelist & Blacklist user (Ubuntu)

Mari kita asumsikan kita punya 2 user:

1. `user_boleh` (User yang akan di-whitelist/diizinkan).
2. `user_blokir` (User yang akan di-blacklist/dilarang).

Berikut adalah konfigurasi untuk dua skenario berbeda: Blacklist (blokir sedikit, izinkan semua) dan Whitelist (izinkan sedikit, blokir semua).

***

#### 1. Persiapan User

Pertama, buat user-nya di terminal jika belum ada: (**SERVER SIDE**)

Bash

```
sudo adduser user_boleh
```

```
sudo adduser user_blokir
```

***

#### 2. Skenario A: Blacklist (Blokir User Tertentu)

Logika: Semua orang boleh masuk, kecuali nama yang ada di daftar.

1. Edit file konfigurasi: `sudo nano /etc/vsftpd.conf`
2.  Pastikan baris berikut diatur seperti ini:

    Ini, TOML

    ```
    userlist_enable=YES
    userlist_deny=YES
    userlist_file=/etc/vsftpd.user_list
    ```

    > _Penjelasan: `userlist_deny=YES` artinya "Tolak user yang ada di dalam file list"._
3. Edit file daftar user: `sudo nano /etc/vsftpd.user_list`
4.  Masukkan nama user yang ingin diblokir:

    Plaintext

    ```
    user_blokir
    ```

    _(Jangan masukkan `user_boleh` di sini)_.
5. Restart servis: `sudo systemctl restart vsftpd`

Hasil: `user_boleh` bisa login. `user_blokir` akan ditolak langsung (biasanya error "Permission denied").

***

#### 3. Skenario B: Whitelist (Hanya Izinkan User Tertentu)

Logika: Semua orang dilarang masuk (Default Deny), hanya nama yang ada di daftar yang boleh masuk. Ini jauh lebih aman.

1. Edit file konfigurasi: `sudo nano /etc/vsftpd.conf`
2.  Ubah konfigurasinya menjadi:

    Ini, TOML

    ```
    userlist_enable=YES
    userlist_deny=NO
    userlist_file=/etc/vsftpd.user_list
    ```

    > _Penjelasan: `userlist_deny=NO` artinya "Hanya izinkan user yang ada di dalam file list"._
3. Edit file daftar user: `sudo nano /etc/vsftpd.user_list`
4.  Masukkan nama user yang ingin diizinkan:

    Plaintext

    ```
    user_boleh
    ```

    _(User lain seperti `user_blokir` atau user sistem lainnya otomatis tertolak karena tidak ada di daftar ini)._
5. Restart servis: `sudo systemctl restart vsftpd`

Hasil: `user_boleh` sukses login. `user_blokir` (dan user lainnya) akan gagal login.

Kemudian masuk client login FTP dengan user yang sudah dibuat sebelumnya

```
ftp 192.168.56.2
```

```
user_boleh
```

```
user_blokir
```

#### Ringkasan Perbedaan

| Fitur     | Setting di `vsftpd.conf` | Fungsi File `vsftpd.user_list` |
| --------- | ------------------------ | ------------------------------ |
| Blacklist | `userlist_deny=YES`      | Daftar orang yang DILARANG     |
| Whitelist | `userlist_deny=NO`       | Daftar orang yang DIIZINKAN    |
