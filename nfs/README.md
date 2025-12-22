# NFS

## 1. Pengertian NFS Server (ringkas + detail)

**NFS (Network File System)** adalah protokol jaringan yang memungkinkan satu mesin (server) mengekspor (membagikan) direktori sehingga mesin lain (client) dapat _mount_ direktori tersebut seolah-olah berada pada filesystem lokal.\
Digunakan luas untuk berbagi file antar Linux/Unix; mendukung akses file remote dengan semantics seperti file lokal (tergantung mode).

Hal penting:

* NFS dapat berjalan dalam beberapa versi utama: **NFSv2, NFSv3, NFSv4**. NFSv4 adalah yang paling modern (stateful, firewall-friendlier — memakai port 2049 saja, mendukung namespace virtual).
* Komponen utama di Linux: `nfs-kernel-server` (server), `nfs-common` (client utilities dan lib).
* ID mapping: akses file bergantung pada UID/GID; user harus cocok antar mesin kecuali memakai mekanisme mapping atau Kerberos.
* Opsi ekspor mempengaruhi izin & keamanan: `rw/ro`, `root_squash`, `no_root_squash`, `sync/async`, `subtree_check/no_subtree_check`.

## 2. Installation NFS Server (Ubuntu)

Langkah-langkah berikut untuk **Ubuntu (server)** — contoh IP server `192.168.56.12`, clients di jaringan `192.168.56.0/24`.

Instalasi NFS

1. Masuk root

```
su -
```

2. Update Sistem

```
apt update
```

#### Dari sisi VM Server

3. Install NFS Server

```
apt install -y nfs-kernel-server
```

<figure><img src="../.gitbook/assets/image (288).png" alt=""><figcaption></figcaption></figure>

4. Buat direktori terlebih dahulu

```
mkdir -p /srv/nfs/shared
```

<figure><img src="../.gitbook/assets/image (289).png" alt=""><figcaption></figcaption></figure>

5. Buat user dan grup

```
chown nobody:nogroup /srv/nfs/shared
```

```
chmod 2775 /srv/nfs/shared
```

<figure><img src="../.gitbook/assets/image (290).png" alt=""><figcaption></figcaption></figure>

6. Konfigurasi file /etc/exports

```
nano /etc/exports
```

<figure><img src="../.gitbook/assets/image (291).png" alt=""><figcaption></figcaption></figure>

CTRL + X dan Y untuk Simpan dan keluar

7. Terapkan konfigurasi dan restart NFS:

```
exportfs -rav
```

<figure><img src="../.gitbook/assets/image (292).png" alt=""><figcaption></figcaption></figure>

```
systemctl restart nfs-server
```

8. Periksa status & ports:

```
exportfs -s
```

```
showmount -e
```

```
ss -tulpn | grep 2049
```

<figure><img src="../.gitbook/assets/image (293).png" alt=""><figcaption></figcaption></figure>

9. Firewall (jika aktif — UFW contohnya):\
   NFSv4 hanya perlu port 2049 (TCP/UDP), tapi jika kamu gunakan NFSv3 & RPC services perlu beberapa port/ rpcbind.\
   Contoh membuka untuk jaringan internal:

```
ufw allow from 192.168.56.0/24 to any port 2049 proto tcp
```

```
ufw allow from 192.168.56.0/24 to any port 2049 proto udp
```

```
ufw allow from 192.168.56.0/24 to any port 111 proto tcp
```

```
ufw allow from 192.168.56.0/24 to any port 111 proto udp
```

```
ufw status
```

<figure><img src="../.gitbook/assets/image (294).png" alt=""><figcaption></figcaption></figure>

#### Dari sisi Client

Install NFS Client (Ubuntu)

1. Masuk root

```
su -
```

2. Update Sistem

```
apt update
```

3. Install NFS

```
apt install -y nfs-common
```

4. Buat titik mount

```
mkdir -p /mnt/nfs/shared
```

<figure><img src="../.gitbook/assets/image (295).png" alt=""><figcaption></figcaption></figure>

5. Mounting NFS

```
mount -t nfs 192.168.56.12:/srv/nfs/shared /mnt/nfs/shared
```

```
mount -a
```

<figure><img src="../.gitbook/assets/image (297).png" alt=""><figcaption></figcaption></figure>

6. Mounting permanent

```
nano /etc/fstab
```

```
192.168.56.12:/srv/nfs/shared  /mnt/nfs/shared  nfs  defaults,_netdev  0  0
```

<figure><img src="../.gitbook/assets/image (296).png" alt=""><figcaption></figcaption></figure>

7. Cek apakah sudah mount atau belum

```
mount | grep nfs
```

```
df -h | grep nfs
```

<figure><img src="../.gitbook/assets/image (298).png" alt=""><figcaption></figcaption></figure>

8. Cek NFS

```
ls -la /mnt/nfs/shared
```

<figure><img src="../.gitbook/assets/image (299).png" alt=""><figcaption></figcaption></figure>

9. Test NFS buat file dari sisi server

```
echo "halo dari server" | sudo tee /srv/nfs/shared/hello.txt
```

<figure><img src="../.gitbook/assets/image (300).png" alt=""><figcaption></figcaption></figure>

10. Cek dari sisi Client

```
ls -la /mnt/nfs/shared
```

<figure><img src="../.gitbook/assets/image (301).png" alt=""><figcaption></figcaption></figure>

11. Test NFS dari sisi Client buat file

```
echo "halo dari client" | sudo tee /mnt/nfs/shared/hello1.txt
```

<figure><img src="../.gitbook/assets/image (305).png" alt=""><figcaption></figcaption></figure>

12. Cek dari sisi server

```
ls -la /srv/nfs/shared
```

<figure><img src="../.gitbook/assets/image (306).png" alt=""><figcaption></figcaption></figure>
