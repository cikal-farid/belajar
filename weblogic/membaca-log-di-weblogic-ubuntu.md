# Membaca log di WebLogic (Ubuntu)

## Membaca log di WebLogic

Setiap kali WebLogic melakukan sesuatu — mulai dari **start server, deploy aplikasi, error database, hingga request dari user** — semua itu dicatat dalam log file.

Jadi, **log = “jejak digital”** dari semua aktivitas WebLogic.

Berikut step by stepnya :

1\.       Login ke WebLogic terlebih dahulu dan masuk ke menu Diagnostic dan pilih submenu Log Files

<figure><img src="../.gitbook/assets/image (360).png" alt=""><figcaption></figcaption></figure>

&#x20;2\.       Pilih log nya dan sebagai contoh ingin melihat log pada bagian ServerLog dan klik view

<figure><img src="../.gitbook/assets/image (361).png" alt=""><figcaption></figcaption></figure>

&#x20;3\.       Pilih entries yang ingin dilihat dan klik View

<figure><img src="../.gitbook/assets/image (529).png" alt=""><figcaption></figcaption></figure>

&#x20;4\.       Seperti gambar dibawah ini untuk detail log nya.

<figure><img src="../.gitbook/assets/image (530).png" alt=""><figcaption></figcaption></figure>

&#x20;5\.       Jika ingin melihat log menggunakan perintah didalam linux, ikuti perintah dibawah ini

<figure><img src="../.gitbook/assets/image (531).png" alt=""><figcaption></figcaption></figure>

```
cd /u01/app/oracle/config/domains/mydomain/servers/AdminServer/logs
```

&#x20;6\.       Jika kita melihat detail isi didalam direktori yang sebelumnya kita jalankan maka hasilnya seperti gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (532).png" alt=""><figcaption></figcaption></figure>

Terdapat 3 file log yaitu :

Access.log = untuk melihat akses weblogic server yang sedang berjalan

AdminServer.log = untuk melihat aplikasi yang sedang dijalankan oleh admin di AdminServer

Mydomain.log = untuk melihat proses berjalannya service WebLogic

Untuk detailnya sebagai berikut :

&#x20;Access.log

<figure><img src="../.gitbook/assets/image (534).png" alt=""><figcaption></figcaption></figure>

Adminserver.log

<figure><img src="../.gitbook/assets/image (535).png" alt=""><figcaption></figcaption></figure>

Mydomain.log

<figure><img src="../.gitbook/assets/image (536).png" alt=""><figcaption></figcaption></figure>
