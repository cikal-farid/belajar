# Administrasi Windows Server 2008

Active Directory Services merupakan sebuah layanan di windows server yang digunakan untuk mengelola aturan, hak akses serta hal – hal yang berkaitan dengan keamanan lainnya dari user ataupun komputer pada seluruh jaringan di perusahaan. Berikut ini hal – hal yang dapat dilakukan oleh Administrator dengan adanya Active Directory Service :

* Mengatur apa saja yang boleh dan tidak boleh dilakukan oleh sebuah komputer pada jaringan milik perusahaan. Contoh : Apakah komputer A boleh digunakan untuk mengakses file pada folder tertentu.
* Mengatur bagaimana mekanisme akses jaringan komputer antara kantor pusat dan kantor cabang, induk perusahaan dengan anak perusahaan atau antar kantor lainnya yang bersifat remote. Contoh : Apakah user di kantor wilayah A dapat mengakses kantor wilayah B. Domain Controller adalah server yang mengontrol Active Directory Services. Semua data dan informasi tentang konfigurasi dari Active Directory Services disimpan di Domain Controller. Data yang disimpan dibagi menjadi 2, yaitu :

1. User Account merupakan informasi aturan, hak akses, dan hal – hal yang berkaitan dengan security dari seorang karyawan atau user.
2. Computer Account merupakan informasi aturan, hak akses, dan hal – hal yang berkaitan dengan security lainnya dari sebuah komputer yang terhubung ke jaringan perusahaan.&#x20;

Semua user account, computer account dan konfigurasi keamanannya disimpan pada sebuah database di Domain Controller. Database yang disimpan memiliki sebuah Schema, yaitu aturan yang menentukan tipe dan jenis data serta informasi apa saja yang disimpan di database.&#x20;

Group merupakan sekumpulan dari user account yang memiliki konfigurasi keamanan sejenis. User account ini dikelompokkan untuk mempermudah proses pengaturan yang dilakukan oleh Administrator. Organizational Unit adalah sekumpulan group yang dibentuk dengan tujuan adanya delegasi kegiatan administrator pada salah satu user account di group tersebut.&#x20;

Organizational unit merupakan pilihan yang sangat bagus bagi beberapa kantor dengan lokasi yang berbeda atau kantor remote.&#x20;

Domain adalah kumpulan – kumpulan organizational unit. Sebuah domain dapat memiliki beberapa sub domain yang dibagi berdasarkan kebutuhan masing – masing perusahaan. Biasanya pembagian sub domain dilakukan berdasarkan lokasi.&#x20;

Tree adalah kumpulan dari domain serta sub domain sub domain nya.&#x20;

Forest merupakan kumpulan – kumpulan dari tree. Biasanya penggabungan 2 tree dilakukan ketika ada dua bagian yang sangat berbeda, terutama dalam hal schema yang disimpan pada domain controller. Hal yang sering ditemui adalah pada 2 perusahaan yang telah memiliki tree masing – masing dan bergabung (merger) menjadi satu. Hubungan antara 2 tree dalam 1 forest bersifat one – way trust. One – way trust memungkinkan user account di salah satu tree mengakses tree lainnya, namun tidak berlaku sebaliknya.

Berikut kita coba untuk installasi Active Directory Domain Service .

1. Pertama klik Start > Administrative Tools > Server Manager.

<figure><img src="../.gitbook/assets/image (738).png" alt=""><figcaption></figcaption></figure>

2. Selanjutnya klik Roles > Add Roles.

<figure><img src="../.gitbook/assets/image (739).png" alt=""><figcaption></figcaption></figure>

3. Selanjutnya klik Next.

<figure><img src="../.gitbook/assets/image (740).png" alt=""><figcaption></figcaption></figure>

4. Centang Active Directory Domain Service lalu akan muncul permintaan untuk installasi .NET Framework 3.5.1 klik Add Required Features. Selanjutnya klik Next.

<figure><img src="../.gitbook/assets/image (741).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (194).png" alt=""><figcaption></figcaption></figure>

5. Selanjutnya klik Next.

<figure><img src="../.gitbook/assets/image (195).png" alt=""><figcaption></figcaption></figure>

6. Lalu klik Install.

<figure><img src="../.gitbook/assets/image (196).png" alt=""><figcaption></figcaption></figure>

7. Jika sudah klik Close.

<figure><img src="../.gitbook/assets/image (197).png" alt=""><figcaption></figcaption></figure>

8. Selanjutnya klik Active Directory Domain Services.

<figure><img src="../.gitbook/assets/image (198).png" alt=""><figcaption></figcaption></figure>

9. Selanjutnya klik Run the Active Directory Domain Services Installation Wizard (dcpromo.exe).

<figure><img src="../.gitbook/assets/image (199).png" alt=""><figcaption></figcaption></figure>

10. Selanjutnya klik Next.

<figure><img src="../.gitbook/assets/image (200).png" alt=""><figcaption></figcaption></figure>

11. Klik Next.

<figure><img src="../.gitbook/assets/image (201).png" alt=""><figcaption></figcaption></figure>

12. Selanjutnya kita akan membuat domain baru dan forest baru. Klik Create a new domain in a new forest lalu klik Next.

<figure><img src="../.gitbook/assets/image (202).png" alt=""><figcaption></figcaption></figure>

13. Masukkan nama forest root domain lalu klik Next. Contoh cinosta.com.

<figure><img src="../.gitbook/assets/image (203).png" alt=""><figcaption></figcaption></figure>

14. Pilih Windows Server 2008 R2 lalu klik Next.

<figure><img src="../.gitbook/assets/image (204).png" alt=""><figcaption></figcaption></figure>

15. Selanjutnya klik Next.

<figure><img src="../.gitbook/assets/image (205).png" alt=""><figcaption></figcaption></figure>

16. Klik Yes.

<figure><img src="../.gitbook/assets/image (206).png" alt=""><figcaption></figcaption></figure>

17. Klik Yes.

<figure><img src="../.gitbook/assets/image (207).png" alt=""><figcaption></figcaption></figure>

18. Selanjutnya klik Next.

<figure><img src="../.gitbook/assets/image (208).png" alt=""><figcaption></figcaption></figure>

19. Lalu masukkan kata sandinya dan klik Next. (Coba213)

<figure><img src="../.gitbook/assets/image (209).png" alt=""><figcaption></figcaption></figure>

20. Selanjutnya klik Finish.

<figure><img src="../.gitbook/assets/image (210).png" alt=""><figcaption></figcaption></figure>

21. Selanjutnya kita restart, klik Restart Now.

<figure><img src="../.gitbook/assets/image (211).png" alt=""><figcaption></figcaption></figure>
