# SAN (ISCSI) di Ubuntu

Instalasi dan Konfigurasi SCSI Target (SAN Server)

1. Contoh disini menyiapkan 1 Disk utuh (tidak mount) dan hidupkan VM Servernya

<figure><img src=".gitbook/assets/image (606).png" alt=""><figcaption></figcaption></figure>

2. Update Sistem dan Install Targetcli

```
su -
```

```
apt update
```

```
apt -y install targetcli-fb
```

<figure><img src=".gitbook/assets/image (607).png" alt=""><figcaption></figcaption></figure>

3. Cek Targetcli

```
systemctl status rtslib-fb-targetctl
```

```
which targetcli
```

<figure><img src=".gitbook/assets/image (608).png" alt=""><figcaption></figcaption></figure>

4. Cek Disk yang sudah kita siapkan sebelumnya

```
lsblk
```

<figure><img src=".gitbook/assets/image (611).png" alt=""><figcaption></figcaption></figure>

4. Running Targetcli

```
targetcli
```

<figure><img src=".gitbook/assets/image (609).png" alt=""><figcaption></figcaption></figure>

5. Buat block storage (didalam folder /backstores/block)

```
cd /backstores/block
```

```
create block1 /dev/sdb
```

<figure><img src=".gitbook/assets/image (612).png" alt=""><figcaption></figcaption></figure>

6. Buat TPG (masuk ke folder /iscsi)

```
cd /iscsi
```

```
create iqn.2025-10.com.example.serverchikz:remotedisk1
```

<figure><img src=".gitbook/assets/image (613).png" alt=""><figcaption></figcaption></figure>

7. Buat ACL di dalam TPG (masuk ke /iscsi/nama\_tpg/tpg\_number/acls) (di dalam folder /iscsi itu menjalankan perintah cd /iqn.2025-10.com.example.serverchikz:remotedisk1/tpg1/acls)

```
cd iqn.2025-10.com.example.serverchikz:remotedisk1/tpg1/acls
```

```
create iqn.2025-10.com.example.clientchikz:211301
```

<figure><img src=".gitbook/assets/image (614).png" alt=""><figcaption></figcaption></figure>

8. Buat LUN di dalam TPG (masuk ke /iscsi/nama\_tpg/tpg\_number/luns) (di dalam folder /iscsi itu menjalankan perintah cd iqn.2025-10.com.example.serverchikz:remotedisk1/tpg1/luns)

```
cd /iscsi/
```

```
cd iqn.2025-10.com.example.serverchikz:remotedisk1/tpg1/luns
```

```
create /backstores/block/block1
```

<figure><img src=".gitbook/assets/image (616).png" alt=""><figcaption></figcaption></figure>

9. Buat PORTAL di dalam TPG (masuk ke /iscsi/nama\_tpg/tpg\_number/portals) (di dalam folder /iscsi itu menjalankan perintah cd iqn.2025-10.com.example.serverchikz:remotedisk1/tpg1/portals)

```
cd /iscsi/
```

```
cd iqn.2025-10.com.example.serverchikz:remotedisk1/tpg1/portals
```

```
delete 0.0.0.0 3260
```

```
create 192.168.56.12 3260
```

<figure><img src=".gitbook/assets/image (617).png" alt=""><figcaption></figcaption></figure>

10. Buka firewall untuk port 3260

```
ufw allow 3260/tcp
```

<figure><img src=".gitbook/assets/image (618).png" alt=""><figcaption></figcaption></figure>



Konfigurasi SCSI Initiator (SAN Client)

```
su -
```

1. Ubah nama initiator seperti gambar dibawah ini

```
nano /etc/iscsi/initiatorname.iscsi
```

```
iqn.2025-10.com.example.clientchikz:211301
```

<figure><img src=".gitbook/assets/image (619).png" alt=""><figcaption></figcaption></figure>

2. Menjalankan discovery dan cek

```
iscsiadm -m discovery -t sendtargets -p 192.168.56.12
```

```
iscsiadm --mode node
```

<figure><img src=".gitbook/assets/image (621).png" alt=""><figcaption></figcaption></figure>

3. Login atau masuk Server Target

```
iscsiadm --mode node targetname iqn.2025-10.com.example.serverchikz:remotedisk1 --portal 192.168.56.12 --login
```

<figure><img src=".gitbook/assets/image (622).png" alt=""><figcaption></figcaption></figure>

Jika sudah bisa login, cek disk apakah ada update atau belum

Sebelum

<figure><img src=".gitbook/assets/image (624).png" alt=""><figcaption></figcaption></figure>

Sesudah

<figure><img src=".gitbook/assets/image (623).png" alt=""><figcaption></figcaption></figure>



Konfigurasi untuk melepas disk dari SAN

1. Sebelumnya masih dalam keadaan login, dan saat ini menjalankan perintah untuk logout dan delete user agar tidak terhubung (Client Side)

```
 iscsiadm --mode node targetname iqn.2025-10.com.example.serverchikz:remotedisk1 --portal 192.168.56.12 --logout
```

```
iscsiadm --mode node targetname iqn.2025-10.com.example.serverchikz:remotedisk1 --portal 192.168.56.12 -o delete
```

```
iscsiadm --mode node
```

```
ll /dev/sd*
```

<figure><img src=".gitbook/assets/image (625).png" alt=""><figcaption></figcaption></figure>

2. Langkah selanjutnya menjalankan perintah untuk melepas portals (Server Side)

```
cd /iscsi/iqn.2025-10.com.example.serverchikz:remoteldisk1/tpg1/portals/
```

```
delete 192.168.56.12 3260
```

<figure><img src=".gitbook/assets/image (627).png" alt=""><figcaption></figcaption></figure>

3. Langkah selanjutnya menjalankan perintah untuk melepas luns (Server Side)

```
cd ../luns/
```

```
delete lun0
```

<figure><img src=".gitbook/assets/image (628).png" alt=""><figcaption></figcaption></figure>

4. Langkah selanjutnya menjalankan perintah untuk melepas acls (Server Side)

```
cd ../acls/
```

```
delete iqn.2025-10.com.example.clientchikz:211301
```

<figure><img src=".gitbook/assets/image (629).png" alt=""><figcaption></figcaption></figure>

5. Langkah selanjutnya menjalankan perintah untuk melepas iscsi (Server Side)

```
cd /iscsi/
```

```
delete iqn.2025-10.com.example.serverchikz:remotedisk1
```

<figure><img src=".gitbook/assets/image (630).png" alt=""><figcaption></figcaption></figure>

6. Berikut Langkah terakhir adalah melepas block (Server Side)

```
cd /backstores/block/
```

```
delete block1
```

<figure><img src=".gitbook/assets/image (631).png" alt=""><figcaption></figcaption></figure>
