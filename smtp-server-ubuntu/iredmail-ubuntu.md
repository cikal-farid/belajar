# iRedMail (Ubuntu)

Test Jika ingin menggunakan dashboard seperti iRedMail, berikut panduannya :

Instalasi dan Konfigurasi iRedMail

iRedMail membutuhkan speksifikasi RAM minimal 4 GB jika pada saat instalasi terdapat kegagalan, maka ada tips sebagai berikut :

Tips Optimasi untuk RAM 2 GB

```
sudo fallocate -l 2G /swapfile
```

```
sudo chmod 600 /swapfile
```

```
sudo mkswap /swapfile
```

```
sudo swapon /swapfile
```

```
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Jika di cek akan berubah menjadi 4 GB

```
free -h
```

1. Sebelum install iRedMail, kita perlu set domain local kita terlebih dahulu, masukkan perintah berikut :

Masuk root agar mudah proses instalasi serta konfigurasinya

```
su -
```

```
hostnamectl set-hostname mail.cinosta.com
```

<figure><img src="../.gitbook/assets/image (565).png" alt=""><figcaption></figcaption></figure>

2. Masuk file /etc/hosts dan tambahkan domainnya

```
nano /etc/hosts
```

```
192.168.56.10 mail.cinosta.com
```

<figure><img src="../.gitbook/assets/image (566).png" alt=""><figcaption></figcaption></figure>

CTRL + X dan Y untuk simpan dan keluar

3. Untuk melihat hostname kita masukkan perintah dibawah ini

```
hostname -f
```

<figure><img src="../.gitbook/assets/image (567).png" alt=""><figcaption></figcaption></figure>

4. Masuk folder opt untuk menyimpan iRedMail

```
cd /opt
```

5. Download iRedMail dengan perintah dibawah ini

```
wget https://github.com/iredmail/iRedMail/archive/refs/tags/1.7.4.tar.gz
```

<figure><img src="../.gitbook/assets/image (568).png" alt=""><figcaption></figcaption></figure>

6. Kemudian ekstrak file yang sudah kita download (iRedMail)

```
tar xzf 1.7.4.tar.gz
```

<figure><img src="../.gitbook/assets/image (569).png" alt=""><figcaption></figcaption></figure>

7. Masuk folder yang sudah kita ekstrak sebelumnya

```
cd iRedMail-1.7.4
```

<figure><img src="../.gitbook/assets/image (570).png" alt=""><figcaption></figcaption></figure>

8. Jalankan file .sh iRedMail.sh

```
bash iRedMail.sh
```

<figure><img src="../.gitbook/assets/image (572).png" alt=""><figcaption></figcaption></figure>

Ketika sudah menjalankan file iRedMail.sh secara otomatis akan download kebutuhan iRedMail, sekaligus kita akan mengatur konfigurasi iRedMail dan menunggu proses installasi

9. Pada gambar dibawah ini diperintahkan enter untuk lanjut dan cancel untuk membatalkan dengan cara menekan ctrl + c

<figure><img src="../.gitbook/assets/image (573).png" alt=""><figcaption></figcaption></figure>

Setelah kita enter sebanyak 2x untuk lanjut memasang atau menambahkan repositories kebutuhan, kita akan menunggu cukup lumayan waktu untuk proses downloadnya

<figure><img src="../.gitbook/assets/image (574).png" alt=""><figcaption></figcaption></figure>

10. Lanjut klik Yes untuk masuk ke setup konfigurasi iRedMail

<figure><img src="../.gitbook/assets/image (575).png" alt=""><figcaption></figcaption></figure>

11. Mengatur path storage, disini klik Next atau Enter karena saya memilih default

<figure><img src="../.gitbook/assets/image (576).png" alt=""><figcaption></figcaption></figure>

12. Memilih web server, disini pilih NGINX dan Next (default) jika tidak ada bintang pada Nginx untuk memilih item ketuk space

<figure><img src="../.gitbook/assets/image (577).png" alt=""><figcaption></figcaption></figure>

13. Kemudian, pilih backend penyimpanan untuk akun email. Pilih salah satu yang Anda kenal. disini memilih MariaDB. Tekan tombol panah atas dan bawah, lalu tekan spasi untuk memilih.

<figure><img src="../.gitbook/assets/image (578).png" alt=""><figcaption></figcaption></figure>

14. Jika memilih MariaDB atau MySQL, maka Anda perlu mengatur kata sandi root MySQL. Disini saya input "maildb123" (bebas sesuai ketentuan)

<figure><img src="../.gitbook/assets/image (579).png" alt=""><figcaption></figcaption></figure>

15. Selanjutnya, masukkan domain email pertama Anda. Anda dapat menambahkan domain email tambahan nanti di panel admin berbasis web. Disini mengasumsikan saya menginginkan akun email seperti **cikal@cinosta.com** . Dalam hal ini, Anda perlu memasukkan **your-domain.com** di sini, tanpa subdomain. Jangan tekan spasi setelah nama domain Anda. Saya rasa iRedMail akan menyalin karakter spasi bersama dengan nama domain Anda, yang dapat mengakibatkan kegagalan instalasi. (cinosta.com)

<figure><img src="../.gitbook/assets/image (580).png" alt=""><figcaption></figcaption></figure>

16. Berikutnya, tetapkan kata sandi untuk administrator domain email. (mailadmin123)

<figure><img src="../.gitbook/assets/image (581).png" alt=""><figcaption></figcaption></figure>

17. Pilih komponen opsional. Secara default, 4 item dipilih. Jika ingin menggunakan groupware SOGo (webmail, kalender, buku alamat, ActiveSync), tekan tombol panah bawah dan spasi untuk memilih. Tekan untuk melanjutkan `Enter`ke layar berikutnya.

<figure><img src="../.gitbook/assets/image (582).png" alt=""><figcaption></figcaption></figure>

18. Sekarang kita dapat meninjau konfigurasi Anda. Ketik `Y`untuk memulai instalasi semua komponen server email dan harap menunggu hingga selesai karena proses cukup lama dan lanjut masuk ke tahap berikut nya.

<figure><img src="../.gitbook/assets/image (583).png" alt=""><figcaption></figcaption></figure>

19. Di akhir instalasi, pilih `y`untuk menggunakan aturan firewall yang disediakan oleh iRedMail dan mulai ulang firewall.

<figure><img src="../.gitbook/assets/image (585).png" alt=""><figcaption></figcaption></figure>

20. Instalasi iRedMail kini telah selesai. Kita akan menerima pemberitahuan URL webmail, panel admin web, dan kredensial login. `iRedMail.tips`Berkas ini berisi informasi penting tentang server iRedMail Anda.

<figure><img src="../.gitbook/assets/image (586).png" alt=""><figcaption></figcaption></figure>

21. Nyalakan ulang server Ubuntu

<figure><img src="../.gitbook/assets/image (588).png" alt=""><figcaption></figcaption></figure>

Gambar diatas terlihat username berubah menjadi cikal@mail ketika proses instalasi dan konfigurasi iRedMail telah selesai.

22. Setelah server Anda kembali online, Anda dapat mengunjungi panel admin web.

Web admin panel (iRedAdmin):

```
https://mail.cinosta.com/iredadmin/
```

Roundcube webmail:

```
https://mail.cinosta.com/mail/
```

netdata (monitor):

```
https://mail.cinosta.com/netdata/
```

Username:&#x20;

```
postmaster@cinosta.com
```

Password:&#x20;

```
mailadmin123
```

23. Sebelum kita akses web iRedMail, karena disini saya menggunakan VMware sebagai Server Ubuntu dan sudah terhubung dengan Laptop saya yang menjadi Host dan sudah 1 ip, saya ingin akses web iRedMail melalui browser Host saya atau laptop saya dengan cara menambahkan domain server ubuntu saya ke host saya di C:\Windows\System32\drivers\etc\hosts

```
C:\Windows\System32\drivers\etc\hosts
```

```
192.168.56.10   mail.cinosta.com localhost
```

24. Sekarang saya sudah bisa akses browser Host saya (Laptop) dengan tampilan seperti gambar dibawah ini

<figure><img src="../.gitbook/assets/image (589).png" alt=""><figcaption></figcaption></figure>

Karena server email menggunakan sertifikat TLS yang ditandatangani sendiri, baik pengguna klien email desktop maupun klien webmail akan melihat peringatan. Untuk mengatasinya, kita bisa mendapatkan dan memasang sertifikat TLS Let's Encrypt gratis.

25. Atau bisa klik Advance dan klik [Proceed to mail.cinosta.com (unsafe)](https://chrome-error/chromewebdata/) untuk tetap lanjut akses panel web admin iRedMail

<figure><img src="../.gitbook/assets/image (590).png" alt=""><figcaption></figcaption></figure>

26. Berikut tampilan login Web Panel Admin iRedMail

<figure><img src="../.gitbook/assets/image (591).png" alt=""><figcaption></figcaption></figure>

Masukkan user untuk login

Username:&#x20;

```
postmaster@cinosta.com
```

Password:&#x20;

```
mailadmin123
```

Berikut tampilan dashboard menu Web Admin Panel iRedMail

<figure><img src="../.gitbook/assets/image (592).png" alt=""><figcaption></figcaption></figure>

Menambahkan user mail agar dapat saling berkomunikasi sesama pengguna iRedMail (seperti email biasanya)

1. Masuk ke menu Add

<figure><img src="../.gitbook/assets/image (597).png" alt=""><figcaption></figcaption></figure>

2. Isi data diri user

<figure><img src="../.gitbook/assets/image (593).png" alt=""><figcaption></figcaption></figure>

3. Lanjut klik save changes karena default sudah bisa atau bisa mengatur zone dan sebagainya

<figure><img src="../.gitbook/assets/image (594).png" alt=""><figcaption></figcaption></figure>

Output setelah save changes

<figure><img src="../.gitbook/assets/image (595).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (596).png" alt=""><figcaption></figcaption></figure>

4. Lakukan cara yang sama untuk user yang lainnya

<figure><img src="../.gitbook/assets/image (598).png" alt=""><figcaption></figcaption></figure>

Setelah 2 user terbuat kita coba testing komunikasi antar sesama user di Roundcube Webmail, dan berikut tampilan loginnya dan login terlebih dahulu untuk masuk dashboard Roundcube Webmail

<figure><img src="../.gitbook/assets/image (599).png" alt=""><figcaption></figcaption></figure>

1. Berikut tampilan Roundcube Webmail

<figure><img src="../.gitbook/assets/image (600).png" alt=""><figcaption></figcaption></figure>

2. Test kirim mail berupa teks dan file ke user yang lainnya

<figure><img src="../.gitbook/assets/image (601).png" alt=""><figcaption></figcaption></figure>

3. Berhasil mengirimkan mail

<figure><img src="../.gitbook/assets/image (602).png" alt=""><figcaption></figcaption></figure>

4. Tampilan ada mail masuk dari user novita@cinosta.com

<figure><img src="../.gitbook/assets/image (603).png" alt=""><figcaption></figcaption></figure>

Perintah mengirim mail dari VM 2 (pengirim) ke VM 1 (penerima) dan masuk ke Roundcube Webmail

```
echo "Tes kirim email antar VM" | mail -s "Test Mail" cikal@cinosta.com
```

<figure><img src="../.gitbook/assets/image (605).png" alt=""><figcaption></figcaption></figure>

iRedMail juga di lengkap dengan aplikasi monitoring server seperti gambar dibawah ini

<figure><img src="../.gitbook/assets/image (604).png" alt=""><figcaption></figcaption></figure>
