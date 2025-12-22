# Deploy Aplikasi PHP

Pada dasarnya, WordPress adalah Content Management System (CMS) yang dibangun menggunakan bahasa pemrograman PHP dan database MySQL. Sebagai CMS, WordPress mempermudah pembuatan website tanpa harus paham coding. Mulai dari website untuk blog pribadi, portofolio, toko online, hingga company profile, WordPress bisa menjadi pilihan terbaik. Kamu bisa menambah fitur, mengatur tampilan, dan menata konten website dengan mudah. Selain mudah digunakan, WordPress bisa dipakai gratis karena bersifat open-source. Dengan begitu, WordPress adalah platform yang cocok bagi pemula atau profesional untuk membuat website dengan mudah hanya dalam hitungan menit saja.

Berikut merupakan Langkah-langkah yang dapat dilakukan untuk installasi wordpress dengan xampp di windows server 2008 menggunakan wordpress

1. Siapkan file php nya, disini saya menggunakan wordpress dan letakkan di dalam folder C:Xampp/htdocs/.

<figure><img src="../../.gitbook/assets/image (141).png" alt=""><figcaption></figcaption></figure>

2. Berikutnya kita buat database wordpress di MariaDb dengan memasukkan perintah “CREATE DATABASE wordpress;”.

```
CREATE DATABASE wordpress;
```

<figure><img src="../../.gitbook/assets/image (83).png" alt=""><figcaption></figcaption></figure>

3. Selanjutnya kita buat user database untuk wordpress nya dengan memasukkan perintah

```
CREATE USER 'adminwp'@'192.168.56.16' IDENTIFIED BY 'cinosta';
```

<figure><img src="../../.gitbook/assets/image (84).png" alt=""><figcaption></figcaption></figure>

4. Lalu kita beri hak akses dengan memasukkan perintah

```
GRANT ALL ON wordpress.* TO 'adminwp'@'192.168.56.16' IDENTIFIED BY 'cinosta';
```

<figure><img src="../../.gitbook/assets/image (85).png" alt=""><figcaption></figcaption></figure>

5. Selanjutnya kita reload dengan memasukkan perintah

```
FLUSH PRIVILEGES;
```

<figure><img src="../../.gitbook/assets/image (86).png" alt=""><figcaption></figcaption></figure>

6. Selanjutnya kita cari file wp-config-sample.php. Kita copy dan rename menjadi wp-config.php

```
wp-config.php
```

<figure><img src="../../.gitbook/assets/image (87).png" alt=""><figcaption></figcaption></figure>

7. Lalu klik kanan wp-config.php > Edit.

<figure><img src="../../.gitbook/assets/image (88).png" alt=""><figcaption></figcaption></figure>

8. Pada script SQL Setting sesuaikan nama database, username, password dan database charset seperti berikut.

```
wordpress
```

```
adminwp
```

```
cinosta
```

```
192.168.56.16
```

```
utf8mb4
```

<figure><img src="../../.gitbook/assets/image (89).png" alt=""><figcaption></figcaption></figure>

9. Lalu scroll ke bagian Authentication unique keys and salt. Untuk mengisi bagian ini kita perlu generate secret key dari link https://api.wordpress.org/secret-key/1.1/salt/. Kita copy script dari web tersebut ke wp-config.php kita seperti berikut.

```
https://api.wordpress.org/secret-key/1.1/salt/
```

<figure><img src="../../.gitbook/assets/image (91).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (90).png" alt=""><figcaption></figcaption></figure>

10. Lalu kita save file wp-config.php yang telah kita edit dan buka link nya di web browser alamat ip/wordpress lalu klik Ayo! seperti berikut.

```
http://192.168.56.16:81/wordpress/
```

11. Pertama kali masuk kita akan memilih bahasa terlebih dahulu dan lanjut continue

<figure><img src="../../.gitbook/assets/image (92).png" alt=""><figcaption></figcaption></figure>

12. Selanjutnya masukkan nama basis data, nama pengguna, sandi, dan email lalu klik install wordpress.

```
wordpress
```

```
adminwp
```

```
cinosta
```

```
adminwp@wp.com
```

<figure><img src="../../.gitbook/assets/image (95).png" alt=""><figcaption></figcaption></figure>

13. Kemudian klik log masuk

<figure><img src="../../.gitbook/assets/image (96).png" alt=""><figcaption></figcaption></figure>

14. Kemudian lanjut masukan username dan password lalu lanjut log masuk

<figure><img src="../../.gitbook/assets/image (97).png" alt=""><figcaption></figcaption></figure>

15. Wordpress siap untuk digunakan.

<figure><img src="../../.gitbook/assets/image (98).png" alt=""><figcaption></figcaption></figure>
