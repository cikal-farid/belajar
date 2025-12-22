# Weblogic di Windows Server

WebLogic adalah Server Aplikasi yang berjalan pada tingkat menengah, antara database back-end dan aplikasi terkait serta klien tipis berbasis browser. WebLogic Server memediasi pertukaran permintaan dari tingkat klien dengan respons dari tingkat back-end. WebLogic Server dapat digunakan sebagai server Web utama untuk aplikasi lanjutan yang mendukung Web.

<figure><img src="../../.gitbook/assets/image (799).png" alt=""><figcaption></figcaption></figure>

INSTALL WEBLOGIC DI WINDOWS Berikut cara installasi WebLogic di windows server.

1. Siapkan file jdk dan WebLogic nya. Install jdk nya terlebih dahulu.

<figure><img src="../../.gitbook/assets/image (167).png" alt=""><figcaption></figcaption></figure>

karena disini saya sudah install maka lanjut ketahap selanjutnya

<figure><img src="../../.gitbook/assets/image (168).png" alt=""><figcaption></figcaption></figure>

2. Selanjutnya pindah ke folder installasi weblogic nya dengan menggunakan perintah “cd \[lokasi\_file]”.

```
cd Documents\WEBLOGIC
```

<figure><img src="../../.gitbook/assets/image (170).png" alt=""><figcaption></figcaption></figure>

3. Jalankan file installasi nya dengan menggunakan perintah “java –jar fmw\_14.1.1.0.0\_wls\_lite\_generic.jar”.

```
java –jar fmw_14.1.1.0.0_wls_lite_generic.jar
```

<figure><img src="../../.gitbook/assets/image (171).png" alt=""><figcaption></figcaption></figure>

4. Selanjutnya klik Next.

<figure><img src="../../.gitbook/assets/image (173).png" alt=""><figcaption></figcaption></figure>

5. Klik Next.

<figure><img src="../../.gitbook/assets/image (174).png" alt=""><figcaption></figcaption></figure>

6. Klik Next.

<figure><img src="../../.gitbook/assets/image (175).png" alt=""><figcaption></figcaption></figure>

7. Pilih WebLogic Server lalu Nex

<figure><img src="../../.gitbook/assets/image (176).png" alt=""><figcaption></figcaption></figure>

8. Klik Next.

<figure><img src="../../.gitbook/assets/image (177).png" alt=""><figcaption></figcaption></figure>

9. Lalu klik Install.

<figure><img src="../../.gitbook/assets/image (178).png" alt=""><figcaption></figcaption></figure>

10. Selanjutnya klik Next.

<figure><img src="../../.gitbook/assets/image (179).png" alt=""><figcaption></figcaption></figure>

11. Lalu centang Automatically Launch the Configuration Wizard lalu klik Finish.

<figure><img src="../../.gitbook/assets/image (180).png" alt=""><figcaption></figcaption></figure>

12. Selanjutnya pilih Create a new domain, lalu atur Domain Location nya lalu klik Next.

<figure><img src="../../.gitbook/assets/image (181).png" alt=""><figcaption></figcaption></figure>

13. Berikutnya centang Include all selected templates lalu klik Next.

<figure><img src="../../.gitbook/assets/image (182).png" alt=""><figcaption></figcaption></figure>

14. Lalu buat username dan kata sandi nya lalu klik Next.

<figure><img src="../../.gitbook/assets/image (183).png" alt=""><figcaption></figcaption></figure>

untuk sandi saya buat (Coba2113)

15. Selanjutnya pilih Development lalu klik Next.

<figure><img src="../../.gitbook/assets/image (184).png" alt=""><figcaption></figcaption></figure>

16. Centang Administration Server dan Node Mnager lalu klik Next.

<figure><img src="../../.gitbook/assets/image (185).png" alt=""><figcaption></figcaption></figure>

17. Masukkan Server Name, Listen Address dan Listen Port lalu klik Next.

<figure><img src="../../.gitbook/assets/image (186).png" alt=""><figcaption></figcaption></figure>

18. Selanjutnya masukkan Username dan kata sandinya lalu klik Next.

<figure><img src="../../.gitbook/assets/image (187).png" alt=""><figcaption></figcaption></figure>

19. Selanjutnya klik Create.

<figure><img src="../../.gitbook/assets/image (188).png" alt=""><figcaption></figcaption></figure>

20. Selanjutnya klik Next.

<figure><img src="../../.gitbook/assets/image (189).png" alt=""><figcaption></figcaption></figure>

21. Selanjutnya centang Start Admin Server lalu klik Finish.

<figure><img src="../../.gitbook/assets/image (190).png" alt=""><figcaption></figcaption></figure>

22. Selanjutnya buka \[ip address] : 7001 / console dan login.

```
http://192.168.56.16:7001/console
```

<figure><img src="../../.gitbook/assets/image (192).png" alt=""><figcaption></figcaption></figure>

23. Jika berhasil login maka WebLogic siap digunakan.

<figure><img src="../../.gitbook/assets/image (193).png" alt=""><figcaption></figcaption></figure>
