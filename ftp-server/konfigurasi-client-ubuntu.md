# Konfigurasi Client (Ubuntu)

Untuk akses FTP Server yang sebelumnya kita sudah konfigurasi, nah sekarang kita perlu konfigurasi dari sisi client nya. ada beberapa cara sebagai berikut :

Akses FTP Server di Client menggunakan file explorer

#### 1. Cara Cepat (Akses Langsung via Address Bar)

Ini adalah cara termudah untuk sekadar melihat atau mengunduh file dengan cepat.

1. Buka File Explorer (shortcut: `Tombol Windows + E`).
2. Di bagian _address bar_ (tempat Anda biasanya melihat `C:\Users\...`), klik dan hapus teksnya.
3. Ketik alamat server FTP Anda diawali dengan `ftp://`.
   * Contoh: `ftp://192.168.56.2` atau `ftp://ftp.example.com`
4. Tekan Enter.
5. Akan muncul kotak dialog _pop-up_ yang meminta Username dan Password.
   * Masukkan username dan password yang Anda buat (misalnya, `ftpuser` dari contoh kita sebelumnya).
   * Klik "Log On".

Anda sekarang akan melihat file dan folder di server Anda, sama seperti Anda menjelajahi folder biasa di komputer Anda. Anda bisa _copy-paste_ atau _drag-and-drop_ file dari dan ke server.

#### 2. Cara Permanen (Menambahkan "Network Location")

Cara ini akan membuat _shortcut_ permanen ke server FTP Anda di bawah "This PC", sehingga Anda tidak perlu mengetik alamatnya berulang kali.

1. Buka File Explorer.
2. Di panel sebelah kiri, klik kanan pada "This PC" (atau "Computer").
3. Pilih "Add a network location".

<figure><img src="../.gitbook/assets/image (308).png" alt=""><figcaption></figcaption></figure>

4. Sebuah _wizard_ (panduan) akan muncul. Klik Next.

<figure><img src="../.gitbook/assets/image (309).png" alt=""><figcaption></figcaption></figure>

5. Pilih "Choose a custom network location" dan klik Next.

<figure><img src="../.gitbook/assets/image (310).png" alt=""><figcaption></figcaption></figure>

6. Di kotak "Internet or network address", ketik alamat FTP Anda (misal: `ftp://192.168.56.2`). Klik Next.

<figure><img src="../.gitbook/assets/image (311).png" alt=""><figcaption></figcaption></figure>

7. Di layar berikutnya, hilangkan centang pada "Log on anonymously".

<figure><img src="../.gitbook/assets/image (312).png" alt=""><figcaption></figcaption></figure>

8. Ketik Username Anda (misal: `ftpuser`). Klik Next.

<figure><img src="../.gitbook/assets/image (315).png" alt=""><figcaption></figcaption></figure>

9. Beri nama yang deskriptif untuk koneksi ini (misal: "Server FTP Kantor" atau "Web Server Saya") disini saya pilih default karena untuk mengingat IP nya. Klik Next.

<figure><img src="../.gitbook/assets/image (316).png" alt=""><figcaption></figcaption></figure>

10. Klik Finish.

<figure><img src="../.gitbook/assets/image (317).png" alt=""><figcaption></figcaption></figure>

Sekarang, setiap kali Anda membuka "This PC", Anda akan melihat _shortcut_ baru dengan nama yang Anda buat. Mengkliknya akan langsung menghubungkan Anda ke server (mungkin akan meminta password lagi, yang bisa Anda pilih untuk "Save password").

<figure><img src="../.gitbook/assets/image (318).png" alt=""><figcaption></figcaption></figure>

Jika ingin menggunakan Aplikasi pun juga bisa dan saat ini saya menggunakan FileZilla, seperti pada gambar dibawah ini :

Masukkan Host, Username, User, Password, dan Port kemudian di login Quick Connet

<figure><img src="../.gitbook/assets/image (319).png" alt=""><figcaption></figcaption></figure>

Jika sebelumnya menggunakan windows dan saat ini saya ingin menggunakan client dengan OS Linux Ubuntu

Ini adalah cara klasik di Linux dan sangat cepat jika Anda sudah terbiasa dengan terminal.

1. Anda mungkin perlu menginstal klien `ftp` standar (seringkali belum terinstal):

```
apt update
```

```
apt install ftp
```

2. Jalankan perintah untuk terhubung:

```
ftp 192.168.56.2
```

3. Terminal akan merespons (misal: `Connected to 192.168.56.2.`)
4. Ia akan meminta nama. Ketik: `ftpuser` lalu `Enter`.
5. Ia akan meminta password. Ketik password Anda (tidak akan terlihat) lalu `Enter`.
6. Jika berhasil, Anda akan melihat `ftp>`.

Beberapa perintah dasar di dalam `ftp>`:

* `ls`: Melihat file di server.
* `get namafile.txt`: Mengunduh file dari server ke komputer Anda.
* `put file_lokal.txt`: Mengunggah file dari komputer Anda ke server.
* `bye`: Keluar dari sesi FTP.

<figure><img src="../.gitbook/assets/image (321).png" alt=""><figcaption></figcaption></figure>

#### ⭐ Rekomendasi Terbaik: Gunakan SFTP

Karena Anda sekarang menghubungkan Linux ke Linux, Anda seharusnya TIDAK menggunakan FTP.

Gunakan SFTP (SSH File Transfer Protocol).

* Mengapa? Ini jauh lebih aman (semua, termasuk password, terenkripsi) dan menggunakan port SSH (port 22) yang kemungkinan besar sudah terbuka.
* Server: Anda tidak perlu mengubah apa pun di server. Jika SSH server Anda aktif, SFTP 99% sudah aktif secara otomatis.
* Klien: Semua metode di atas mendukung SFTP.

Cara Menggunakan SFTP:

* Di Nautilus (Files): Di "Connect to Server", ketik: `sftp://192.168.56.2`
* Di FileZilla: Di kolom "Host", ketik: `sftp://192.168.56.2` (FileZilla juga akan otomatis menggunakan SFTP jika Anda hanya memasukkan IP dan mengganti Port ke `22`).
*   Di Terminal: Gunakan perintah `sftp` (sudah pasti terinstal jika ada SSH):

    Bash

    ```
    sftp ftpuser@192.168.56.2
    ```

    (Ini akan menggunakan user dan password yang sama dengan FTP Anda).

<figure><img src="../.gitbook/assets/image (322).png" alt=""><figcaption></figcaption></figure>



Selamat sampai tahap ini kita sudah berhasil saling terhubung antara client dan server nya bisa saling sharing file maupun folder, dan berikut sebagai contohnya dibawah ini :

Windows dengan file explorer

<figure><img src="../.gitbook/assets/image (323).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (324).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (325).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (326).png" alt=""><figcaption></figcaption></figure>

Jika sudah saling terhubung disini saya mencoba salin folder test di server ke client windows menggunakan file explorer dengan drag and drop bisa, salin atau pindah file maupun folder bisa

Linux Ubuntu dengan ftp

<figure><img src="../.gitbook/assets/image (327).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (328).png" alt=""><figcaption></figcaption></figure>

Jika sudah saling terhubung antara server dengan os ubuntu dan client dengan os ubuntu, dan client ubuntu sudah terinstall ftp, maka akan seperti gambar diatas yang dimana saya melakukan pengambilan file cik.txt menggunakan perintah get dari sisi client ubuntu
