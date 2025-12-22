# Cara Menemukan Baris yang Error (Ubuntu)

1. Ini adalah cara cepat untuk memaksa `vsftpd` memberitahu kita apa yang salah. Jalankan perintah ini di terminal Anda:

```
sudo /usr/sbin/vsftpd /etc/vsftpd.conf
```

(Kita menjalankan programnya secara manual, di luar `systemd`).

2. Karena konfigurasinya salah, perintah ini akan gagal dan seharusnya mencetak pesan error yang spesifik ke terminal Anda, misalnya:

`500 OOPS: unrecognized option in config file: [nama_opsi_yang_salah]`

Atau mungkin:

`500 OOPS: bad bool value: [baris_yang_salah]`

```
sudo nano /etc/vsftpd.conf
```

```
sudo systemctl restart vsftpd
```

```
sudo systemctl status vsftpd
```
