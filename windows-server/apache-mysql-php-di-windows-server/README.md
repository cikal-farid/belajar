# Apache, Mysql, PHP di Windows Server

Berikut cara memasang Apache, MariaDB dan PHP.

1. Pertama kita perlu menyiapkan file Xampp yang berisi paket-paket Apache, MariaDB dan PHP. Untuk mengunduhnya kita bisa membuka laman https://www.apachefriends.org/index.html. Lalu klik Xampp for Windows.

```
https://www.apachefriends.org/index.html
```

<figure><img src="../../.gitbook/assets/image (116).png" alt=""><figcaption></figcaption></figure>

2. Sebelum memasang kita perlu mematikan akses User Account Account karena beberapa file Xampp membutuhkan write permission di folder C:/Program Files. Klik Start > Control Panel.

<figure><img src="../../.gitbook/assets/image (117).png" alt=""><figcaption></figcaption></figure>

3. Klik User Accounts.

<figure><img src="../../.gitbook/assets/image (118).png" alt=""><figcaption></figcaption></figure>

4. Klik User Accounts lagi.

<figure><img src="../../.gitbook/assets/image (119).png" alt=""><figcaption></figcaption></figure>

5. Klik Change User Account Control Settings.

<figure><img src="../../.gitbook/assets/image (120).png" alt=""><figcaption></figcaption></figure>

6. Klik Never Notify > Ok.

<figure><img src="../../.gitbook/assets/image (121).png" alt=""><figcaption></figcaption></figure>

7. Selanjutnya kita install file yang telah di download tadi. Klik 2x.

<figure><img src="../../.gitbook/assets/image (122).png" alt=""><figcaption></figcaption></figure>

8. Klik Run.

<figure><img src="../../.gitbook/assets/image (124).png" alt=""><figcaption></figcaption></figure>

9. Klik next

<figure><img src="../../.gitbook/assets/image (126).png" alt=""><figcaption></figcaption></figure>

10. Klik Next

<figure><img src="../../.gitbook/assets/image (127).png" alt=""><figcaption></figcaption></figure>

11. Selanjutnya kita akan mengisi letak folder Xampp berada, biarkan saja terisi secara default lalu klik Next.

<figure><img src="../../.gitbook/assets/image (128).png" alt=""><figcaption></figcaption></figure>

12. Selanjutnya kita akan memilih bahasa yang digunakan untuk control panel Xampp nanti. Pilih English > Next.

<figure><img src="../../.gitbook/assets/image (129).png" alt=""><figcaption></figcaption></figure>

13. Selanjutnya matikan centang Learn more about Binami for Xampp > Next.

<figure><img src="../../.gitbook/assets/image (130).png" alt=""><figcaption></figcaption></figure>

14. Selanjutnya klik Next.

<figure><img src="../../.gitbook/assets/image (131).png" alt=""><figcaption></figcaption></figure>

15. Selanjutnya centang Do you want to start the Control Panel now?. Lalu klik Finish.

<figure><img src="../../.gitbook/assets/image (132).png" alt=""><figcaption></figcaption></figure>

16. Selanjutnya kita kembalikan settingan User Account Control ke sebelumnya.

<figure><img src="../../.gitbook/assets/image (133).png" alt=""><figcaption></figcaption></figure>

17. Jika sudah kita coba jalankan service Apache dan MySql dengan klik start.

<figure><img src="../../.gitbook/assets/image (134).png" alt=""><figcaption></figcaption></figure>

18. Jika berwarna hijau dan ada tulisan stop maka service berhasil di jalankan.

<figure><img src="../../.gitbook/assets/image (135).png" alt=""><figcaption></figcaption></figure>

19. Kita coba masukkan ip address kita di browser maka akan muncul halaman dashboard Xampp.

```
http://192.168.56.16:81/dashboard/
```

<figure><img src="../../.gitbook/assets/image (136).png" alt=""><figcaption></figcaption></figure>

20. Selanjutnya kita cek MariaDb nya apakah berjalan atau tidak dengan memasukkan ip address/phpmyadmin, jika berhasil maka seperti tampilan berikut.

```
http://192.168.56.16:81/phpmyadmin/
```

<figure><img src="../../.gitbook/assets/image (137).png" alt=""><figcaption></figcaption></figure>

21. Selanjutnya kita cek apakah service PHP nya berjalan atau tidak dengan memasukkan file .php ke folder C:Xampp/htdocs/.

<figure><img src="../../.gitbook/assets/image (139).png" alt=""><figcaption></figcaption></figure>

22. Selanjutnya kita buka link di web browser ip address/nama file php.php jika berhasil maka akan berjalan file php nya seperti berikut.

<figure><img src="../../.gitbook/assets/image (140).png" alt=""><figcaption></figcaption></figure>
