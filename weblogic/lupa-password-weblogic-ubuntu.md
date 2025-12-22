# Lupa Password WebLogic (Ubuntu)

Jika kita lupa password login pada WebLogic, kita bisa melakukan reset user dan pass nya dengan cara berikut :

1. Masuk ke user oracle

```
su - oracle
```

<figure><img src="../.gitbook/assets/image (538).png" alt=""><figcaption></figcaption></figure>

2. Pastikan semua service WebLogic dalam keadaan off

```
sudo systemctl stop weblogic
./stopWebLogic.sh
```

3. Pastikan juga kepemilikkannya oracle

```
sudo chown -R oracle:oinstall /u01/app/oracle/config/domains/mydomain/
```

4. Masuk ke direktori WebLogic /bin (disini saya sudah set menggunakan $DOMAIN bisa dengan cara manual untuk masuk ke bin)

```
cd $DOMAIN
```

```
cd bin
```

<figure><img src="../.gitbook/assets/image (539).png" alt=""><figcaption></figcaption></figure>

5. Disini kita perlu menjalankan skrip `setDomainEnv.sh` agar _classpath_ Java Anda benar.

```
. setDomainEnv.sh
```

<figure><img src="../.gitbook/assets/image (540).png" alt=""><figcaption></figcaption></figure>

6. Kemudian masuk ke direktori security

```
cd $DOMAIN
```

```
cd security
```

<figure><img src="../.gitbook/assets/image (541).png" alt=""><figcaption></figcaption></figure>

7. Backup terlebih dahulu file dibawah ini dengan cara merubah nama file nya

```
mv DefaultAuthenticatorInit.ldift DefaultAuthenticatorInit_old.ldift
```

<figure><img src="../.gitbook/assets/image (542).png" alt=""><figcaption></figcaption></figure>

8. Kemudian jalankan utilitas dibawah untuk merubah user dan pass atau membuat baru

```
java weblogic.security.utils.AdminAccount cikal cikal1234 .
```

<figure><img src="../.gitbook/assets/image (543).png" alt=""><figcaption></figcaption></figure>

Perintah diatas setelah AdminAccount adalah membuat user dan pass atau merubah password yang lama.

9. Kemudian masuk lagi ke direktori nya security di AdminServer untuk edit file boot.properties

```
cd $DOMAIN
```

```
cd servers/AdminServer/security
```

<figure><img src="../.gitbook/assets/image (544).png" alt=""><figcaption></figcaption></figure>

10. Kemudian edit user dan pass yang sudah ada (akan tampil seperti enkripsi namun kita bisa mengubah nya tidak perlu enkripsi karena weblogic akan enkripsi user pass tersebut secara otomatis)

```
nano boot.properties
```

<figure><img src="../.gitbook/assets/image (545).png" alt=""><figcaption></figcaption></figure>

Menjadi

<figure><img src="../.gitbook/assets/image (546).png" alt=""><figcaption></figcaption></figure>

11. Kemudian pergi ke direktori data nya AdminServer untuk backup folder LDAP dengan cara merubah nama (hitungannya hapus karena akan terbuat yang baru)

```
cd $DOMAIN
```

```
cd servers/AdminServer/data
```

```
mv ldap/ ldap.bak/
```

<figure><img src="../.gitbook/assets/image (547).png" alt=""><figcaption></figcaption></figure>

12. Langkah terakhir adalah jalankan WebLogic kembali

```
cd $DOMAIN
```

```
cd bin
```

```
./startWebLogic.sh
```

<figure><img src="../.gitbook/assets/image (548).png" alt=""><figcaption></figcaption></figure>

Pada saat pertama kali menjalankan kembali WebLogicnya akan menemukan error seperti gambar dibawah itu karena WebLogic tidak menemukan user sebelumnya maupun yang baru dibuat, tapi setelah WebLogic dijalankan untuk kedua kalinya maka tidak akan terjadi kendala karena WebLogic sudah mengenali user yang baru.

<figure><img src="../.gitbook/assets/image (549).png" alt=""><figcaption></figcaption></figure>

Jika langkah demi langkah sudah dilakukan dan pada saat ingin menjalannkan tahap terakhir di weblogic dan akan error , berikut cara memperbaiki nya

<figure><img src="../.gitbook/assets/image (307).png" alt=""><figcaption></figcaption></figure>

berikut cara perbaikinya

hapus atau backup file boot.properties

```
mv boot.properties boot.properties.bak
```

Jalankan kembali WebLogic

```
cd $DOMAIN
```

```
cd bin
```

```
./startWebLogic.sh
```
