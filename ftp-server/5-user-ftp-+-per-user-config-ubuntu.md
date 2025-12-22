# 5 USER FTP + PER-USER CONFIG (Ubuntu)

## **LANGKAH LENGKAP MEMBUAT 5 USER FTP + PER-USER CONFIG**

### **1. Install vsftpd (di VM1)**

```bash
sudo apt update
sudo apt install vsftpd -y
```

Backup config default:

```bash
sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.backup
```

<figure><img src="../.gitbook/assets/image (635).png" alt=""><figcaption></figcaption></figure>

***

## **2. Aktifkan Userlist dan Per-user Config Directory**

Edit config:

```bash
sudo nano /etc/vsftpd.conf
```

Pastikan isi berikut **ditambahkan atau diaktifkan**:

```
local_enable=YES
write_enable=YES
chroot_local_user=YES
allow_writeable_chroot=YES

# Folder untuk konfigurasi per user
user_config_dir=/etc/vsftpd_user_conf

# Hanya user dalam userlist yg boleh login
userlist_enable=YES
userlist_file=/etc/vsftpd.user_list
userlist_deny=NO
```

<figure><img src="../.gitbook/assets/image (634).png" alt=""><figcaption></figcaption></figure>

Simpan → keluar.

***

## **3. Buat folder untuk konfigurasi per-user**

```bash
sudo mkdir /etc/vsftpd_user_conf
```

<figure><img src="../.gitbook/assets/image (636).png" alt=""><figcaption></figcaption></figure>

***

## **4. Buat 5 user FTP**

Kita buat 5 user:

```
user1ftp  
user2ftp  
user3ftp  
user4ftp  
user5ftp
```

Perintah buat semua user:

```bash
for u in user1ftp user2ftp user3ftp user4ftp user5ftp; do
    sudo adduser --home /home/$u --shell /usr/sbin/nologin $u
done
```

<figure><img src="../.gitbook/assets/image (637).png" alt=""><figcaption></figcaption></figure>

**Perintah diatas langsung membuat 5 user sekaligus dan untuk sekaligus mengisi password serta data diri, disini saya buat passwordnya " ; " dan data diri kosong langsung enter, serta lakukan hal yang sama untuk user berikutnya**

**Catatan:**

* `--shell /usr/sbin/nologin` → user hanya boleh FTP, tidak bisa shell login.
* Home directory otomatis dibuat.

Set password untuk tiap user:

```bash
sudo passwd user1ftp
sudo passwd user2ftp
sudo passwd user3ftp
sudo passwd user4ftp
sudo passwd user5ftp
```

***

## **5. Tambahkan semua user ke userlist**

```bash
sudo tee -a /etc/vsftpd.user_list <<EOF
user1ftp
user2ftp
user3ftp
user4ftp
user5ftp
EOF
```

<figure><img src="../.gitbook/assets/image (638).png" alt=""><figcaption></figcaption></figure>

***

## **6. Buat konfigurasi FTP untuk masing-masing user**

Di folder: `/etc/vsftpd_user_conf/`\
Setiap file harus menggunakan **nama user EXACT**.

#### **Contoh konfigurasi per user**

***

#### **User1**

```bash
sudo nano /etc/vsftpd_user_conf/user1ftp
```

Isi:

```
local_root=/home/user1ftp
write_enable=YES
```

<figure><img src="../.gitbook/assets/image (639).png" alt=""><figcaption></figcaption></figure>

***

#### **User2**

```bash
sudo nano /etc/vsftpd_user_conf/user2ftp
```

Isi:

```
local_root=/home/user2ftp
write_enable=YES
```

<figure><img src="../.gitbook/assets/image (640).png" alt=""><figcaption></figcaption></figure>

***

#### **User3**

```bash
sudo nano /etc/vsftpd_user_conf/user3ftp
```

Isi:

```
local_root=/home/user3ftp
write_enable=YES
```

<figure><img src="../.gitbook/assets/image (641).png" alt=""><figcaption></figcaption></figure>

***

#### **User4**

```bash
sudo nano /etc/vsftpd_user_conf/user4ftp
```

Isi:

```
local_root=/home/user4ftp
write_enable=YES
```

<figure><img src="../.gitbook/assets/image (642).png" alt=""><figcaption></figcaption></figure>

***

#### **User5**

```bash
sudo nano /etc/vsftpd_user_conf/user5ftp
```

Isi:

```
local_root=/home/user5ftp
write_enable=YES
```

<figure><img src="../.gitbook/assets/image (643).png" alt=""><figcaption></figcaption></figure>

Pada gambar dibawah ini terlihat file config yang sudah dibuat sebelumnya

<figure><img src="../.gitbook/assets/image (644).png" alt=""><figcaption></figcaption></figure>

***

## **7. Set ownership & permission home directory**

```bash
for u in user1ftp user2ftp user3ftp user4ftp user5ftp; do
    sudo chown $u:$u /home/$u
    sudo chmod 755 /home/$u
done
```

<figure><img src="../.gitbook/assets/image (646).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (645).png" alt=""><figcaption></figcaption></figure>

***

## **8. Restart vsftpd**

```bash
sudo systemctl restart vsftpd
sudo systemctl status vsftpd
```

<figure><img src="../.gitbook/assets/image (647).png" alt=""><figcaption></figcaption></figure>

***

## **9. TEST LOGIN FTP dari VM2 (client)**

Di VM2:

```bash
ftp 192.168.56.8
```

Login sebagai user1ftp:

```
Name: user1ftp
Password: ****
```

Jika berhasil, prompt akan menunjukkan bahwa dia masuk ke:

<figure><img src="../.gitbook/assets/image (648).png" alt=""><figcaption></figcaption></figure>

***

## ✅ HASIL AKHIR

* Kamu memiliki **5 user FTP**, masing-masing punya home directory sendiri.
* Setiap user diarahkan otomatis ke home-nya.
* Setiap user punya **konfigurasi FTP berbeda**, tanpa konfigurasi global yang memaksa semua sama.
* Tidak perlu membuat virtual users, semuanya real system users.
