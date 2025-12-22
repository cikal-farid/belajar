# Aplikasi .NET Menggunakan IIS

IIS (Internet Information Service) merupakan sebuah komponen yang digunakan untuk mengelola website, Ghoper, File Transfer Protocol dan HTTP. IIS mengadopsi konsep GUI di dalam melakukan berbagai pengaturan atau konfigurasi sitem. IIS hanya mendukung sistem operasi yang dibuat Microsoft.\
Berikut cara installasi IIS.

1. Klik Start > Administrative Tools > Server Manager.

<figure><img src="../.gitbook/assets/image (742).png" alt=""><figcaption></figcaption></figure>

2. Klik Roles > Add Roles.

<figure><img src="../.gitbook/assets/image (743).png" alt=""><figcaption></figcaption></figure>

3. Klik Next.

<figure><img src="../.gitbook/assets/image (744).png" alt=""><figcaption></figcaption></figure>

4. Centang Web Server (IIS) lalu Next.

<figure><img src="../.gitbook/assets/image (745).png" alt=""><figcaption></figcaption></figure>

5. Klik Next.

<figure><img src="../.gitbook/assets/image (746).png" alt=""><figcaption></figcaption></figure>

6. Centang HTTP Redirection, Application Development, Health and Diagnostic, Management Tools lalu klik Next.

<figure><img src="../.gitbook/assets/image (747).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (748).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (749).png" alt=""><figcaption></figcaption></figure>

7. Klik Install.

<figure><img src="../.gitbook/assets/image (750).png" alt=""><figcaption></figcaption></figure>

8. Selanjutnya klik Close.

<figure><img src="../.gitbook/assets/image (751).png" alt=""><figcaption></figcaption></figure>

9. Selanjutnya buka web browser dan masukkan “localhost”. Jika muncul logo IIS maka berhasil.

<figure><img src="../.gitbook/assets/image (752).png" alt=""><figcaption></figcaption></figure>

10. Selanjutnya kita coba untuk membuat domain baru. Klik Start > Administrative Tools > DNS.

<figure><img src="../.gitbook/assets/image (753).png" alt=""><figcaption></figcaption></figure>

11. Pilih > Forward Lookup Zones > Klik kanan > New Domain.

<figure><img src="../.gitbook/assets/image (754).png" alt=""><figcaption></figcaption></figure>

12. Lalu masukkan nama domain nya lalu klik OK.

<figure><img src="../.gitbook/assets/image (755).png" alt=""><figcaption></figcaption></figure>

13. Selanjutnya kita edit file hots dengan membuka C:\Windows\System32>drivers>Etc>hosts, Pilih Hosts klik kanan lalu Edit with notepad lalu klik ok.

<figure><img src="../.gitbook/assets/image (756).png" alt=""><figcaption></figcaption></figure>

14. Lalu kita tambahkan ip dan dns nya seperti berikut. Lalu di save dan exit.

<figure><img src="../.gitbook/assets/image (757).png" alt=""><figcaption></figcaption></figure>

15. Selanjutnya kita coba ping domain yang telah di buat tadi menggunakan cmd. Jika Reply berarti berhasil.

<figure><img src="../.gitbook/assets/image (758).png" alt=""><figcaption></figcaption></figure>

16. Selanjutnya kita unduh .Net Framework 4.6.2 dan file deploy nya. Lalu jalankan installasi .Net Framework 4.6.2. Centang I have read and accept the license terms lalu klik Install.

<figure><img src="../.gitbook/assets/image (759).png" alt=""><figcaption></figcaption></figure>

17. Berikutnya klik Finish.

<figure><img src="../.gitbook/assets/image (760).png" alt=""><figcaption></figcaption></figure>

18. Selanjutnya extract file deploy nya dan masukkan ke folder C:inetpub/wwwroot/.

<figure><img src="../.gitbook/assets/image (761).png" alt=""><figcaption></figcaption></figure>

19. Selanjutnya buka Server Manager > Roles > Web Server (IIS) > Internet Information Services (IIS) Manager.

<figure><img src="../.gitbook/assets/image (762).png" alt=""><figcaption></figcaption></figure>

20. Lalu klik kanan Sites > Add Web Site.

<figure><img src="../.gitbook/assets/image (763).png" alt=""><figcaption></figcaption></figure>

21. Selanjutnya isi Site name, Physical path (lokasi folder file deploy), IP Address, Port, Host name, dan centang Start Web site immediately lalu klik ok.

<figure><img src="../.gitbook/assets/image (764).png" alt=""><figcaption></figcaption></figure>

22. Selanjutnya klik Application Pools > \[nama\_dns}.

<figure><img src="../.gitbook/assets/image (766).png" alt=""><figcaption></figcaption></figure>

23. Ubah menjadi .Net Framework terbaru yang telah di install lalu klik ok.

<figure><img src="../.gitbook/assets/image (765).png" alt=""><figcaption></figcaption></figure>

24. Selanjutnya buka cmd dan run as administrator lalu ketikkan iisreset.

<figure><img src="../.gitbook/assets/image (767).png" alt=""><figcaption></figcaption></figure>

25. Selanjutnya buka web browser dan masukkan dns website nya. Jika muncul maka berhasil.

<figure><img src="../.gitbook/assets/image (769).png" alt=""><figcaption></figcaption></figure>



Note : jika project tidak bisa di deploy, coba cek folder bin pada project ada atau tidak jika tidak ada berarti belum di publish, dan untuk publishnya ada di visual studio (restore nuget, kemudian copas folder packages ke project dan build -> build solution di visual studio dan kemudian baru bisa di deploy)
