# Startup Service Weblogic

Mengaktifkan service weblogic pada windows server 2008 (agar setiap awal boot weblogic aktif)

1. Buka task scheduler dan klik create task

<figure><img src="../../.gitbook/assets/image (145).png" alt=""><figcaption></figcaption></figure>

2. Kemudian buat nama service seperti pada gambar dibawah ini

```
Start Service Weblogic
```

Kemudian pilih "**Run whether user is logged on or not"** dan **"Run with highest privileges"**

<figure><img src="../../.gitbook/assets/image (146).png" alt=""><figcaption></figcaption></figure>

3. Lanjut ke tab Triggers dan klik new

<figure><img src="../../.gitbook/assets/image (147).png" alt=""><figcaption></figcaption></figure>

4. Kemudian pilih At Startup dan sesuaikan waktunya seperti pada gambar dibawah ini kemudian klik OK.

<figure><img src="../../.gitbook/assets/image (148).png" alt=""><figcaption></figcaption></figure>

5. Lanjut ke menu Action klik new

<figure><img src="../../.gitbook/assets/image (151).png" alt=""><figcaption></figcaption></figure>

6. Kemudian klik browse dan cari file service web logic seperti gambar dibawah ini dan klik OK

```
C:\Oracle\Middleware\Oracle_Home\user_projects\domains\base_domain\bin\startWebLogic.cmd
```

<figure><img src="../../.gitbook/assets/image (152).png" alt=""><figcaption></figcaption></figure>

7. Kemudian lanjut ke tab conditions dan uncentang semuanya seperti pada gambar dibawah ini

<figure><img src="../../.gitbook/assets/image (153).png" alt=""><figcaption></figcaption></figure>

8. Kemudian pada tab Settings ini sesuaikan seperti pada gambar dibawah ini

<figure><img src="../../.gitbook/assets/image (154).png" alt=""><figcaption></figcaption></figure>

9. Kemudian klik OK dan masukkan password user login (Coba213)

<figure><img src="../../.gitbook/assets/image (155).png" alt=""><figcaption></figcaption></figure>

10. Sampai disini tahapan setting service weblogic telah selesai tinggal Run seperti gambar dibawah ini dan test restart Windows Server untuk memastikan

<figure><img src="../../.gitbook/assets/image (156).png" alt=""><figcaption></figcaption></figure>
