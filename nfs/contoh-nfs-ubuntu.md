# Contoh NFS (Ubuntu)

## -------------------------------------------------------

## 🟢 CASE 1 — Membatasi folder hanya untuk user tertentu

## -------------------------------------------------------

### **1. Buat folder**

```bash
sudo mkdir -p /srv/nfs/projectA
```

<figure><img src="../.gitbook/assets/image (268).png" alt=""><figcaption></figcaption></figure>

### **2. Buat user dan group yang sama seperti di client**

Misal kita ingin user “devuser” yang bisa akses.

```bash
sudo groupadd -g 1001 devgroup
sudo useradd -m -u 1001 -g devgroup -s /bin/bash devuser
```

<figure><img src="../.gitbook/assets/image (269).png" alt=""><figcaption></figcaption></figure>

Set password (opsional):

```bash
sudo passwd devuser
```

### **3. Set pemilik folder**

```bash
sudo chown -R devuser:devgroup /srv/nfs/projectA
sudo chmod 770 /srv/nfs/projectA
```

<figure><img src="../.gitbook/assets/image (270).png" alt=""><figcaption></figcaption></figure>

Artinya:

* Owner dan group boleh rwx
* Orang lain (other) ditolak

***

### **4. Tambahkan export di `/etc/exports`**

```bash
sudo nano /etc/exports
```

Tambahkan:

```
/srv/nfs/projectA 192.168.56.13(rw,sync,no_subtree_check,root_squash)
```

<figure><img src="../.gitbook/assets/image (271).png" alt=""><figcaption></figcaption></figure>

Reload:

```bash
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
```

***

## -------------------------------------------------------

## 🟢 CASE 2 — Folder dengan read-only access

## -------------------------------------------------------

### **1. Buat folder**

```bash
sudo mkdir -p /srv/nfs/dataset
sudo chmod 755 /srv/nfs/dataset
```

<figure><img src="../.gitbook/assets/image (272).png" alt=""><figcaption></figcaption></figure>

### **2. Tambahkan ke `/etc/exports`**

```
sudo nano /etc/exports
```

Tambahkan:

```
/srv/nfs/dataset 192.168.56.13(ro,sync,no_subtree_check)
```

<figure><img src="../.gitbook/assets/image (273).png" alt=""><figcaption></figcaption></figure>

Reload:

```bash
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
```

Folder ini **tidak bisa ditulis dari client**, hanya baca.

***

## -------------------------------------------------------

## 🟢 CASE 3 — root\_squash untuk keamanan

## -------------------------------------------------------

> root\_squash = user root di client akan dipaksa menjadi “nobody” di server.

Contoh folder:

```bash
sudo mkdir -p /srv/nfs/securedata
sudo chown root:root /srv/nfs/securedata
sudo chmod 755 /srv/nfs/securedata
```

<figure><img src="../.gitbook/assets/image (274).png" alt=""><figcaption></figcaption></figure>

Tambahkan ke export:

```
sudo nano /etc/exports
```

```
/srv/nfs/securedata 192.168.56.13(rw,sync,no_subtree_check,root_squash)
```

<figure><img src="../.gitbook/assets/image (276).png" alt=""><figcaption></figcaption></figure>

Reload:

```bash
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
```

Dengan ini:

* root di client tidak bisa baca `securedata`
* hanya root server yang punya akses

***

## =======================================================

## 🟩 **VM2 (192.168.56.13) → NFS CLIENT**

## =======================================================

Install:

```bash
sudo apt update
sudo apt install nfs-common -y
```

***

## -------------------------------------------------------

## 🟢 MOUNT untuk CASE 1 (user tertentu)

## -------------------------------------------------------

### **1. Buat user yang sama di client**

**UID dan GID harus sama** dengan server agar cocok.

Cek di server:

```bash
id devuser
```

Misal:

```
uid=1001(devuser) gid=1001(devgroup)
```

Buat user client dengan UID & GID yang sama:

```bash
sudo groupadd -g 1001 devgroup
sudo useradd -m -u 1001 -g devgroup -s /bin/bash devuser
```

<figure><img src="../.gitbook/assets/image (281).png" alt=""><figcaption></figcaption></figure>

### **2. Buat mount point**

```bash
sudo mkdir -p /mnt/projectA
```

<figure><img src="../.gitbook/assets/image (278).png" alt=""><figcaption></figcaption></figure>

### 3. **Set pemilik folder**

```
sudo chown -R devuser:devgroup /mnt/projectA
sudo chmod 770 /mnt/projectA
```

<figure><img src="../.gitbook/assets/image (283).png" alt=""><figcaption></figcaption></figure>

### **4. Mount**

```bash
sudo mount 192.168.56.12:/srv/nfs/projectA /mnt/projectA
```

<figure><img src="../.gitbook/assets/image (279).png" alt=""><figcaption></figcaption></figure>

### **5. Test**

Login sebagai devuser:

```bash
sudo -u devuser touch /mnt/projectA/file1.txt
```

→ Berhasil ✔

Cek kondisi

Sisi server

<figure><img src="../.gitbook/assets/image (284).png" alt=""><figcaption></figcaption></figure>

Sisi Client

<figure><img src="../.gitbook/assets/image (286).png" alt=""><figcaption></figcaption></figure>

Sekarang coba sebagai root:

```bash
sudo touch /mnt/projectA/root_test.txt
```

→ Harusnya gagal ❌ (karena root\_squash + permission)

<figure><img src="../.gitbook/assets/image (282).png" alt=""><figcaption></figcaption></figure>

***

## -------------------------------------------------------

## 🟢 MOUNT untuk CASE 2 (read-only)

## -------------------------------------------------------

```bash
sudo mkdir -p /mnt/dataset
sudo mount 192.168.56.12:/srv/nfs/dataset /mnt/dataset
```

Test:

```bash
touch /mnt/dataset/test.txt
```

<figure><img src="../.gitbook/assets/image (287).png" alt=""><figcaption></figcaption></figure>

→ **Error** (read-only filesystem)

***

## -------------------------------------------------------

## 🟢 MOUNT untuk CASE 3 (root\_squash)

## -------------------------------------------------------

```bash
sudo mkdir -p /mnt/securedata
sudo mount 192.168.56.12:/srv/nfs/securedata /mnt/securedata
```

Test sebagai root:

```bash
sudo ls /mnt/securedata
```

→ Harusnya **permission denied**

Karena root di client dianggap sebagai nobody.

***

## =======================================================

## 🏁 **RINGKASAN PERILAKU HAK AKSES NFS**

## =======================================================

| CONTOH KASUS               | Permission               | Efek                                          |
| -------------------------- | ------------------------ | --------------------------------------------- |
| **CASE 1 – User tertentu** | chmod 770 + UID/GID sama | Hanya user & group yg sama bisa akses         |
| **CASE 2 – read-only**     | ro                       | Client hanya bisa membaca, tidak bisa menulis |
| **CASE 3 – root\_squash**  | root\_squash (default)   | Root client diubah menjadi nobody             |
