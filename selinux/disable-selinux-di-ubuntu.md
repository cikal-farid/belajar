# Disable Selinux di Ubuntu

Langkah men-_disable_ Selinux di Ubuntu

1. Edit file konfigurasi

```
sudo nano /etc/selinux/config
```

2. Ubah baris ini

```
SELINUX=enforcing
```

3. Menjadi baris ini

```
SELINUX=disabled
```

4. Selesai kemudian save dan restart

```
sudo reboot
```
