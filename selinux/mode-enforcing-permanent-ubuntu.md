# Mode Enforcing Permanent (Ubuntu)

Mode Enforcing Permanent

Dan jika ingin selalu dalam mode enforcing setiap server restart sebaiknya kita permissive terlebih dahulu NetworkManager karena pada saat server reboot kita dapat SSH kembali, karena secara default selinux melarang untuk SSH.

1. Perintah ini memberitahu SELinux untuk "mempercayai" NetworkManager

```
sudo semanage permissive -a NetworkManager_t
```

2. Masuk ke konfigurasi selinux

```
sudo nano /etc/selinux/config
```

3. Cari baris dibawah ini dan ubah menjadi seperti dibawah ini

```
SELINUX=enforcing
```

4. Selesai kemudian save dan restart

```
sudo reboot
```
