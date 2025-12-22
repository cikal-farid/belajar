# Maintenance Log File (Ubuntu)

**Log file** adalah _catatan otomatis_ dari aktivitas sistem, aplikasi, atau pengguna.\
Sedangkan **maintenance log file** berarti **merawat, mengelola, dan mengontrol** log tersebut agar:

* Tidak memenuhi storage,
* Tetap mudah dibaca,
* Dapat digunakan untuk analisis masalah atau audit,
* Tidak membahayakan keamanan (karena kadang berisi data sensitif).

### Struktur Log

/var/log/\
├── syslog\
├── auth.log\
├── nginx/\
│ ├── access.log\
│ └── error.log\
└── app/\
├── app.log\
├── app.log.1\
├── app.log.2.gz

## Maintenance log file menggunakan tools linux yaitu logrotate

### install, konfigurasi, dan jalankan logrotate

1. Update sistem

```
apt update && apt-get upgrade -y
```

2. Install logrotate

```
sudo apt install logrotate -y
```

3. Cek versi logrotate

```
logrotate --version
```

4. Konfigurasi logrotate

```
sudo nano /etc/logrotate.d/nginx
```

5. Masukkan skrip dibawah ini

```
/var/log/nginx/*.log {
    daily              # rotasi setiap hari
    rotate 7           # simpan 7 file lama
    compress           # kompres file lama (.gz)
    delaycompress      # tunda kompres 1 hari
    missingok          # lanjut walau file hilang
    notifempty         # jangan rotasi jika kosong
    create 0640 www-data adm
    postrotate
        systemctl reload nginx > /dev/null 2>&1 || true
    endscript
}
```

| Baris                      | Fungsi                                                                      |
| -------------------------- | --------------------------------------------------------------------------- |
| `/var/log/nginx/*.log`     | Target log yang akan dirotasi.                                              |
| `daily`                    | Jalankan rotasi tiap hari.                                                  |
| `rotate 7`                 | Simpan 7 log lama, hapus selebihnya.                                        |
| `compress`                 | Kompres log lama agar hemat ruang.                                          |
| `delaycompress`            | Tunda kompres 1 siklus (biar log kemarin tetap terbaca).                    |
| `missingok`                | Tidak error kalau file log tidak ada.                                       |
| `notifempty`               | Jangan rotasi log yang kosong.                                              |
| `create 0640 www-data adm` | Buat file log baru dengan permission & owner ini.                           |
| `postrotate ... endscript` | Jalankan perintah setelah rotasi (reload nginx supaya pakai file log baru). |

6. Jalankan logrotate

```
sudo logrotate -d /etc/logrotate.d/nginx
```

```
sudo logrotate -f /etc/logrotate.d/nginx
```

```
ls -lh /var/log/nginx/
```

7. Jika ingin secara otomatis setiap hari bisa menggunakan cron

```
/etc/cron.daily/logrotate
```

```
systemctl status logrotate.timer
```

## Cara Manual Maintenance Log File

1. Masuk ke folder log:

```
cd /var/log/nginx
```

2. Ganti nama log aktif (rotasi):

```
sudo mv access.log access-$(date +%F).log
sudo mv error.log error-$(date +%F).log
```

3. Buat file log baru (kosong):

```
sudo touch access.log error.log
sudo chown www-data:adm access.log error.log
sudo chmod 640 access.log error.log
```

4. Reload nginx supaya pakai file baru:

```
sudo systemctl reload nginx
```

5. Kompres log lama agar hemat ruang:

```
sudo gzip access-*.log error-*.log
```

6. Hapus log lama lebih dari 14 hari:

```
sudo find /var/log/nginx -type f -name "*.gz" -mtime +14 -delete
```
