# Start Stop WebLogic (Ubuntu)

## Start Stop WebLogic

Berikut step by stepnya :

1\.       Membuat direktori seperti pada gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (369).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (370).png" alt=""><figcaption></figcaption></figure>

```
mkdir -p /u01/app/oracle/config/domains/mydomain/servers/AdminServer/security
```

&#x20;2\.       Lalu membuat file boot.properties seperti pada gambar dibawah ini

<figure><img src="../.gitbook/assets/image (371).png" alt=""><figcaption></figcaption></figure>

```
nano /u01/app/oracle/config/domains/mydomain/servers/AdminServer/security/boot.properties
```

<figure><img src="../.gitbook/assets/image (372).png" alt=""><figcaption></figcaption></figure>

```
username=weblogic
password=12345678
```

Membuat file boot.properties untuk otomatis login pada linux

&#x20;3\.       Kemudian jalankan WebLogic dengan perintah seperti gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (373).png" alt=""><figcaption></figcaption></figure>

```
cd /u01/app/oracle/config/domains/mydomain/bin
```

<figure><img src="../.gitbook/assets/image (374).png" alt=""><figcaption></figcaption></figure>

```
./startWebLogic.sh
```

&#x20;4\.       Kemudian kita akses [http://192.168.56.2:7001/console](http://192.168.56.2:7001/console) pada web browser kita dan masukkan passwordnya.

<figure><img src="../.gitbook/assets/image (375).png" alt=""><figcaption></figcaption></figure>

```
http://192.168.56.2:7001/console
```

5\.       Weblogic berhasil masuk dan dijalankan kembali

<figure><img src="../.gitbook/assets/image (376).png" alt=""><figcaption></figcaption></figure>



6\.       Jika kita ingin menghentikan service WebLogic, Langkah awal adalah buka terminal baru dan ikuti perintah berikut

<figure><img src="../.gitbook/assets/image (378).png" alt=""><figcaption></figcaption></figure>

```
cd /u01/app/oracle/config/domains/mydomain/bin
```

```
./stopWebLogic.sh
```

Maka service yang sedang berjalan pada tab terminal sebelumnya akan berhenti juga, seperti pada gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (379).png" alt=""><figcaption></figcaption></figure>

Dan jika kita refresh pada web browser maka akan muncul tampilan seperti dibawah ini.

<figure><img src="../.gitbook/assets/image (380).png" alt=""><figcaption></figcaption></figure>

&#x20;

Jika ingin stop WebLogic bisa dengan url [http://192.168.56.2:7001/console](http://192.168.56.2:7001/console), kemudian masuk ke menu Environment dan masuk submenu Servers kemudian pilih control dan pilih AdminServer(admin) dan shutdown à force shutdown now dan klik yes, seperti pada gambar dibawah ini. (JANGAN MENGGUNAKAN CTRL + C PADA TERMINAL KARENA AKAN MENYEBABKAN WEBLOGIC RUSAK)

```
http://192.168.56.2:7001/console
```

<figure><img src="../.gitbook/assets/image (382).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (383).png" alt=""><figcaption></figcaption></figure>

&#x20;7\.       Sekarang kita membuat service WebLogic berjalan di background, dan ikuti step by step sebagai berikut :

Membuat file weblogic.service

```
sudo nano /etc/systemd/system/weblogic.service
```

<figure><img src="../.gitbook/assets/image (384).png" alt=""><figcaption></figcaption></figure>

Isi skrip file tersebut seperti gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (385).png" alt=""><figcaption></figcaption></figure>

```
[Unit]
Description=WebLogic Server
After=network.target
 
[Service]
Type=simple
User=oracle
WorkingDirectory=/u01/app/oracle/config/domains/mydomain
ExecStart=/u01/app/oracle/config/domains/mydomain/bin/startWebLogic.sh
ExecStop=/u01/app/oracle/config/domains/mydomain/bin/stopWebLogic.sh
Restart=on-failure
 
[Install]
WantedBy=multi-user.target
```

Memuat skrip systemd yang baru ( reload systemd )

<figure><img src="../.gitbook/assets/image (386).png" alt=""><figcaption></figcaption></figure>

```
systemctl daemon-reload
```

Cek status weblogic.service

<figure><img src="../.gitbook/assets/image (387).png" alt=""><figcaption></figcaption></figure>

```
systemctl status weblogic.service
```

Disini status masih disable dan inactive karena kita belum enable akan tetapi systemd sudah terbaca, Langkah selanjutnya aktifkan service tersebut seperti gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (388).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (389).png" alt=""><figcaption></figcaption></figure>

```
systemctl enable weblogic.service
```

```
systemctl start weblogic.service
```

Jika ingin menghentikan WebLogic bisa dengan cara dibawah ini

<figure><img src="../.gitbook/assets/image (390).png" alt=""><figcaption></figcaption></figure>

```
systemctl stop weblogic.service
```

```
systemctl status weblogic.service
```

Bisa juga membuat shell script seperti dibawah ini.

Pertama kita buat terlebih dahulu file nya dengan menjalankan perintah seperti gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (391).png" alt=""><figcaption></figcaption></figure>

```
nano start_weblogic.sh
```

Kemudian masukkan skrip dibawah ini pada file diatas yang sebelumnya sudah kita buat.

<figure><img src="../.gitbook/assets/image (392).png" alt=""><figcaption></figcaption></figure>

```
#!/bin/bash
 
# Navigasi ke direktori domain WebLogic
cd /u01/app/oracle/config/domains/mydomain/bin
 
# Jalankan WebLogic Server di background menggunakan nohup
nohup ./startWebLogic.sh > weblogic.out 2>&1 &
 
echo "WebLogic Server berjalan di belakang layar yaa guys krn dia malu. Buat Logs bisa kamu liat di weblogic.out"
```

Kemudian ubah akses kepemilikkan seperti gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (393).png" alt=""><figcaption></figcaption></figure>

```
chmod +x start_weblogic.sh
```

Kemudian jalankan perintah berikut untuk mengaktifkan file skrip yang sudah kita buat sebelumnya.

<figure><img src="../.gitbook/assets/image (394).png" alt=""><figcaption></figcaption></figure>

```
./start_weblogic.sh
```

Untuk melihat isi log nya seperti pada gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (395).png" alt=""><figcaption></figcaption></figure>

```
cd /u01/app/oracle/config/domains/mydomain/bin
```

```
tail -f weblogic.out
```

\[ ctrl + c ] Jika sudah selesai melihat lognya.

Jika ingin menghentikannya menggunakan shell script juga bisa, dengan mengikuti Langkah berikut :

Membuat file untuk shell script tersebut seperti gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (397).png" alt=""><figcaption></figcaption></figure>

```
nano stop_weblogic.sh
```

<figure><img src="../.gitbook/assets/image (398).png" alt=""><figcaption></figcaption></figure>

```
#!/bin/bash
 
# Navigasi ke direktori domain WebLogic
cd /u01/app/oracle/config/domains/mydomain/bin
 
# Jalankan WebLogic Server di background menggunakan nohup
nohup ./stopWebLogic.sh > WeblogicStop.out 2>&1 &
 
echo "WebLogic Server kita hentikan di belakang layar yaa guys krn dia malu. Buat Logs bisa kamu liat di weblogic.out"Dan hal yang sama seperti saat membuat start_weblogic.sh kita rubah kepemelikkannya, seperti gambar dibawah ini. 
```

<figure><img src="../.gitbook/assets/image (399).png" alt=""><figcaption></figcaption></figure>

```
chmod +x stop_weblogic.sh
```

Kemudian jalankan perintah berikut untuk menghentikan WebLogic seperti gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (400).png" alt=""><figcaption></figcaption></figure>

```
./stop_weblogic.sh
```

Untuk melihat log berhentinya WebLogic, jalankan perintah berikut.

<figure><img src="../.gitbook/assets/image (401).png" alt=""><figcaption></figcaption></figure>

```
cd /u01/app/oracle/config/domains/mydomain/bin
```

```
tail -f WeblogicStop.out
```

Contoh Kasus Start WebLogic

Namun jika ingin menjalankan kembali weblogic dengan user oracle yang sudah di jalankan dengan user root sebelumnya itu tidak bisa dan akan muncul error dibawah ini

*

    <figure><img src="../.gitbook/assets/image (329).png" alt=""><figcaption></figcaption></figure>

1.  `<Error> <Log Management> <BEA-170020> <.../AdminServer.log could not be opened successfully.>`

    * Artinya: User `oracle` tidak bisa membuka file log-nya sendiri. Kenapa? Karena file `AdminServer.log` tersebut sekarang dimiliki oleh `root`.

    <figure><img src="../.gitbook/assets/image (331).png" alt=""><figcaption></figcaption></figure>
2.  `<Error> <EmbeddedLDAP> <BEA-000000> <Error opening the Transaction Log: .../EmbeddedLDAP.tran (Permission denied)>`

    * Artinya: Ini adalah error fatalnya. Server tidak bisa membuka database LDAP internalnya karena Permission Denied. File ini (dan direktorinya) sekarang juga dimiliki oleh `root`.

    <figure><img src="../.gitbook/assets/image (330).png" alt=""><figcaption></figcaption></figure>

Semua error lain yang Anda lihat (seperti `ClassCastException` dan `Critical <WebLogicServer> <BEA-000362>`) hanyalah gejala dari kegagalan LDAP. Server tidak bisa berjalan tanpa layanan LDAP-nya.

Untuk memperbaiki kendala tersebut kita perlu mengubah semua user kepemilikkan dari root menjadi user oracle semua, agar user oracle dapat menjalankan kembali weblogic.

```
sudo chown -R oracle:oinstall /u01/app/oracle/config/domains/mydomain/
```
