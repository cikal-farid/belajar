# Backup Restore Server Aplikasi di Linux

Berikut cara backup server aplikasi di OS Linux.

1. Misalnya kita akan backup server aplikasi Wordpress. Kita buat direktori backup terlebih dahulu dengan memasukkan perintah dibawah ini.

```
mkdir /home/backup
```

<figure><img src="../.gitbook/assets/image (73).png" alt=""><figcaption></figcaption></figure>

2. Selanjutnya kita masuk ke direktori /var/www/html dengan menggunakan perintah “cd /var/www/html”.

```
cd /var/www/html
```

<figure><img src="../.gitbook/assets/image (74).png" alt=""><figcaption></figcaption></figure>

Pada gambar diatas kita akan backup direktori home.

3. Selanjutnya kita backup dengan ekstensi .tar menggunakan perintah “ sudo tar –zcvf backupwp.tar home/”.

```
sudo tar -zcvf backupwp.tar home/
```

<figure><img src="../.gitbook/assets/image (75).png" alt=""><figcaption></figcaption></figure>

Pada gambar diatas tunggu sampai proses kompress selesai.

Jika proses aman akan muncul file ekstensi tar seperti gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (76).png" alt=""><figcaption></figcaption></figure>

4. Lalu kita compress kembali dengan ekstensi .gz agar berekstensi .tar.gz dengan menggunakan perintah “gzip –v backupwp.tar”.

```
sudo gzip -v backupwp.tar
```

<figure><img src="../.gitbook/assets/image (77).png" alt=""><figcaption></figcaption></figure>

5. Selanjutnya kita pindahkan file backup nya ke direktori /home/backup dengan menggunakan perintah “mv backupwp.tar.gz /home/backup”.

```
sudo mv backupwp.tar.gz /home/backup
```

<figure><img src="../.gitbook/assets/image (78).png" alt=""><figcaption></figcaption></figure>

Untuk memastikan file bisa menggunakan perintah dibawah ini

```
ll /home/backup/
```

```
ll /var/www/html/
```

Berikut cara restore file backup server aplikasi di OS Linux.

1. Jika kita ingin mengembalikan data ke keadaan semula pada saat di backup maka kita pindahkan file backup nya ke direktori tujuan lalu kita cek di direktori tujuan dengan menggunakan perintah “cp backupwp.tar.gz /var/www/html && cd /var/www/html && ls”.

```
cd /home/backup/
```

<figure><img src="../.gitbook/assets/image (79).png" alt=""><figcaption></figcaption></figure>

```
sudo cp backupwp.tar.gz /var/www/html && cd /var/www/html && ls
```

<figure><img src="../.gitbook/assets/image (80).png" alt=""><figcaption></figcaption></figure>

2. Karena disini ada 2 ekstensi yaitu .tar dan .gz maka kita perlu mengekstrak secara berurutan dari yang terakhir ke yang awal atau dari ekstensi paling kanan ke ekstensi paling kiri agar data tidak terjadi corrupt. Maka menggunakan perintah “gunzip –v backupwp.tar.gz” lalu “tar –zxvf backupwp.tar”.

```
sudo gunzip -v backupwp.tar.gz
```

<figure><img src="../.gitbook/assets/image (81).png" alt=""><figcaption></figcaption></figure>

```
sudo tar -zxvf backupwp.tar
```

<figure><img src="../.gitbook/assets/image (82).png" alt=""><figcaption></figcaption></figure>
