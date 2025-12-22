# Manage Server di WebLogic (Ubuntu)

## Manage Server di WebLogic

Merupakan komponen inti dalam arsitektur domain weblogic, dan bagian penting dari bagaimana WebLogic mengelola aplikasi enterprise.

Berikut step by step manage server :

1\.       Login terlebih dahulu ke WebLogic

<figure><img src="../.gitbook/assets/image (334).png" alt=""><figcaption></figcaption></figure>

&#x20;2\.       Masuk ke menu Environment à Servers dan klik New untuk buat baru server

<figure><img src="../.gitbook/assets/image (335).png" alt=""><figcaption></figcaption></figure>

&#x20;3\.       Disini saya buat secara default hanya ganti port nya dan klik next, seperti pada gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (336).png" alt=""><figcaption></figcaption></figure>

&#x20;4\.       Kemudian klik Finish

<figure><img src="../.gitbook/assets/image (337).png" alt=""><figcaption></figcaption></figure>

Setelah klik Finish terdapat popup pesan seperti gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (338).png" alt=""><figcaption></figcaption></figure>

&#x20;5\.       Kemudian ke menu Machine dan klik next (disini memilih default)

<figure><img src="../.gitbook/assets/image (339).png" alt=""><figcaption></figcaption></figure>

&#x20;6\.       Kemudian klik Finish (default)

<figure><img src="../.gitbook/assets/image (340).png" alt=""><figcaption></figcaption></figure>

&#x20; 7\.       Kemudian di menu Machine lalu ketuk submenu Servers dan klik Add, seperti gambar dibawah ini

<figure><img src="../.gitbook/assets/image (341).png" alt=""><figcaption></figcaption></figure>

&#x20;8\.       Kemudian pilih opsi select server yang menuju ke ke Server-0 seperti gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (342).png" alt=""><figcaption></figcaption></figure>

&#x20;9\.       Kemudian pergi ke menu Deployments dan klik App nya (sample) (disini saya sudah pernah deploy aplikasi Bernama sample.war jadi akan digunakan sebagai running di port server yang baru dibuat (7005) jika belum ada maka deploy terlebih dahulu)

<figure><img src="../.gitbook/assets/image (343).png" alt=""><figcaption></figcaption></figure>

&#x20;10\.   Kemudian pergi ke menu Targets dan pilih Server-0 yang sebelumnya dibuat dan klik Save.

<figure><img src="../.gitbook/assets/image (344).png" alt=""><figcaption></figcaption></figure>

&#x20;11\.   Setelah di klik Save akan muncul popup pesan baru seperti gambar dibawah, dan setelah semua terkonfigurasi maka tahap selanjutnya klik Activate Changes dan akan seperti tampilan dibawah terdapat popup pesan baru yang artinya semua perubahan telah aktif.

<figure><img src="../.gitbook/assets/image (345).png" alt=""><figcaption></figcaption></figure>

&#x20;12\.   Selanjutnya kita running skrip startNodeManager.sh untuk mengaktifkan server yang baru kita buat sebelumnya. Setiap running skrip yang ada folder bin jauh lebih mudah tanpa perlu mengingat path WebLogic maka ikuti Langkah berikut :&#x20;

<figure><img src="../.gitbook/assets/image (346).png" alt=""><figcaption></figcaption></figure>

```
sudo nano /etc/profile
```

<figure><img src="../.gitbook/assets/image (347).png" alt=""><figcaption></figcaption></figure>

```
export DOMAIN=/u01/app/oracle/config/domains/mydomain/
```

<figure><img src="../.gitbook/assets/image (348).png" alt=""><figcaption></figcaption></figure>

```
source /etc/profile
```

Nah sampai sini kita hanya perlu menjalankan perintah _cd $DOMAIN_, dan akan masuk otomatis ke path WebLogic seperti gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (349).png" alt=""><figcaption></figcaption></figure>

File skrip yang akan dijalankan itu berada pada folder bin dan kita perlu masuk ke bin/ dan jalankan perintah seperti gambar dibawah ini untuk running skripnya.

<figure><img src="../.gitbook/assets/image (351).png" alt=""><figcaption></figcaption></figure>

```
cd bin
```

```
./startNodeManager.sh
```

13\.   Skrip berjalan dengan baik status started seperti gambar dibawah ini

<figure><img src="../.gitbook/assets/image (352).png" alt=""><figcaption></figcaption></figure>

&#x20;14\.   Untuk memastikan proses berjalan dengan baik kita pergi ke menu Machine dan pilih nama mesinnya, seperti gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (353).png" alt=""><figcaption></figcaption></figure>

&#x20;15\.   Kemudian ketuk tab Monitoring dan cek status jika Reachable maka kita bisa lanjut ketahap selanjutnya seperti gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (354).png" alt=""><figcaption></figcaption></figure>

&#x20;16\.   Kemudian pergi ke menu Servers dan ketuk submenu Control seperti pada gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (355).png" alt=""><figcaption></figcaption></figure>

&#x20;17\.   Kemudian pilih server yang baru dibuat sebelumnya dan kita ketuk Start

<figure><img src="../.gitbook/assets/image (356).png" alt=""><figcaption></figcaption></figure>

&#x20;18\.   Kemudian ketuk Yes

<figure><img src="../.gitbook/assets/image (357).png" alt=""><figcaption></figcaption></figure>

&#x20;19\.   Kemudian jalankan perintah berikut

<figure><img src="../.gitbook/assets/image (358).png" alt=""><figcaption></figcaption></figure>

```
sudo ufw allow 7005/tcp
```

&#x20;20\.   Jalankan url `http://192.168.56.2:7005/sample/` untuk melihat hasilnya seperti gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (359).png" alt=""><figcaption></figcaption></figure>

```
http://192.168.56.2:7005/sample/
```
