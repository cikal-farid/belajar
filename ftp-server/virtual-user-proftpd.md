# Virtual User ProFTPD

## 🟦 **A. KONFIGURASI VM1 (192.168.56.8) — SERVER PROFTPD**

### 1. Install ProFTPD

```
sudo apt update
sudo apt install proftpd -y
```

***

### 2. Buat database user virtual

```
sudo touch /etc/proftpd/ftpd.passwd
sudo chmod 600 /etc/proftpd/ftpd.passwd
```

<figure><img src="../.gitbook/assets/image (248).png" alt=""><figcaption></figcaption></figure>

***

### 3. Buat folder home root untuk FTP

```
sudo mkdir -p /home/ftp
sudo chmod 755 /home/ftp
```

<figure><img src="../.gitbook/assets/image (249).png" alt=""><figcaption></figcaption></figure>

***

### 4. Konfigurasi ProFTPD

Edit file:

```
sudo nano /etc/proftpd/proftpd.conf
```

Tambahkan atau pastikan baris berikut ada:

```
AuthOrder mod_auth_file.c

AuthUserFile /etc/proftpd/ftpd.passwd

RequireValidShell off
DefaultRoot ~
```

<figure><img src="../.gitbook/assets/image (250).png" alt=""><figcaption></figcaption></figure>

**Tujuan:**

* Virtual user disimpan di `/etc/proftpd/ftpd.passwd`
* User terkunci di home directory-nya

## **Tentukan Passive Ports di ProFTPD**

**Menentukan passive port karena jika tidak server akan membuka random port dan jika client ingin konek dan tidak akan terhubung karena belum dibuka di firewall**

Edit:

```
sudo nano /etc/proftpd/proftpd.conf
```

Tambahkan:

```
PassivePorts 50000 51000
```

Sekarang server akan memakai port **50000–51000** saja, bukan random.

<figure><img src="../.gitbook/assets/image (256).png" alt=""><figcaption></figcaption></figure>

Simpan.

Restart ProFTPD:

```
sudo systemctl restart proftpd
```

Cek status:

```
sudo systemctl status proftpd
```

<figure><img src="../.gitbook/assets/image (251).png" alt=""><figcaption></figcaption></figure>

## **Buka port passive di firewall (di VM1)**

Jika memakai UFW:

```
sudo ufw allow 21/tcp
sudo ufw allow 50000:51000/tcp
sudo ufw reload
```

<figure><img src="../.gitbook/assets/image (257).png" alt=""><figcaption></figcaption></figure>

## **(Opsional) Tambahkan MasqueradeAddress (jika pakai NAT)**

Jika VM1 di NAT (bukan Host-Only):

```
MasqueradeAddress <IP_VirtualBox>
```

Contoh:

```
MasqueradeAddress 192.168.56.8
PassivePorts 50000 51000
```

***

### 5. Buat Virtual User (contoh: userftp1)

#### 5.1 Buat Group Khusus Virtual User dan home folder user

Buat group khusus untuk virtual user (disarankan)

```
sudo groupadd ftpusers
```

GID grup bisa dilihat dengan:

```
getent group ftpusers
```

<figure><img src="../.gitbook/assets/image (259).png" alt=""><figcaption></figcaption></figure>

```
sudo mkdir -p /home/ftp/ftpcikal
sudo chmod 755 /home/ftp/ftpcikal
sudo chown -R 2001:1009 /home/ftp/ftpcikal
```

<figure><img src="../.gitbook/assets/image (261).png" alt=""><figcaption></figcaption></figure>

#### 5.2 Buat user virtual dengan ftpasswd

```
sudo ftpasswd \
  --passwd \
  --name=ftpcikal \
  --file=/etc/proftpd/ftpd.passwd \
  --home=/home/ftp/ftpcikal \
  --shell=/bin/false \
  --uid=2001 \
  --gid=$(getent group ftpusers | cut -d: -f3)
```

Masukkan password ketika diminta.

Akan muncul prompt:

```
Password: ;
Re-enter password: ;
```

<figure><img src="../.gitbook/assets/image (260).png" alt=""><figcaption></figcaption></figure>

Sukses!

Selesai — user virtual aktif.

***

### 6. Test lokal di VM1

Test login FTP dari VM1 ke dirinya sendiri:

```
ftp 192.168.56.8
```

Isi:

```
Name: ftpcikal
Password: ; (password tadi)
```

Lalu tes:

```
pwd
ls -l
```

Jika masuk ke `/home/ftp/user1ftp` → sukses.

<figure><img src="../.gitbook/assets/image (262).png" alt=""><figcaption></figcaption></figure>

***

## 🟩 **B. KONFIGURASI VM2 (192.168.56.9) — CLIENT TEST**

### 1. Install FTP client

```
sudo apt update
sudo apt install ftp -y
```

***

### 2. Test koneksi ke server VM1

```
ping 192.168.56.8
```

Kalau reply → lanjut.

***

### 3. Test login FTP ke VM1

```
ftp 192.168.56.8
```

Masukkan:

```
Name: ftpcikal
Password: ; (password)
```

Sukses jika muncul:

```
230 User user1ftp logged in
```

<figure><img src="../.gitbook/assets/image (263).png" alt=""><figcaption></figcaption></figure>

***

### 4. Perintah-perintah test

#### Cek current directory:

```
pwd
```

<figure><img src="../.gitbook/assets/image (255).png" alt=""><figcaption></figcaption></figure>

#### List file:

```
ls -l
```

#### Upload file ke FTP server:

Buat file dummy dulu di VM2:

```
echo "halo dari VM2" > halo.txt
```

<figure><img src="../.gitbook/assets/image (264).png" alt=""><figcaption></figcaption></figure>

Upload:

```
put halo.txt
```

<figure><img src="../.gitbook/assets/image (265).png" alt=""><figcaption></figcaption></figure>

#### Download dari FTP server:

```
get halo.txt
```

<figure><img src="../.gitbook/assets/image (266).png" alt=""><figcaption></figcaption></figure>

#### Buat folder di server:

```
mkdir folder1
```

Semua operasi harus terjadi di `/home/ftp/user1ftp`.

***

***

## 🟦 **D. TEST MENGUBAH PASSWORD VIRTUAL USER (DI VM1)**

### ✅ Perintah lengkap untuk mengubah password user `ftpcikal`

Gunakan:

```bash
sudo ftpasswd \
  --passwd \
  --name=ftpcikal \
  --uid=2001 \
  --gid=1009 \
  --home=/home/ftp/ftpcikal \
  --shell=/bin/false \
  --file=/etc/proftpd/ftpd.passwd
```

Output yang benar:

```
Password:
Re-type password:
ftpasswd: entry changed
```

<figure><img src="../.gitbook/assets/image (267).png" alt=""><figcaption></figcaption></figure>

***

## 🟩 **G. Checklist Ringkas**

| Item                                | Status |
| ----------------------------------- | ------ |
| ProFTPD terinstall                  | ✔      |
| Virtual user aktif                  | ✔      |
| Home folder per-user                | ✔      |
| DefaultRoot aktif                   | ✔      |
| VM2 bisa login FTP                  | ✔      |
| Upload/Download berfungsi           | ✔      |
| Password bisa diubah pakai ftpasswd | ✔      |
