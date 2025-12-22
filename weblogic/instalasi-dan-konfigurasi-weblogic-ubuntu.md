# Instalasi dan Konfigurasi WebLogic (Ubuntu)

## Weblogic

Berikut cara install dan konfigurasi weblogic di linux ubuntu

1\.       Download file jdk dan weblogic oracle di situs resmi oracle [https://www.oracle.com/java/technologies/downloads/](https://www.oracle.com/java/technologies/downloads/) dan [https://www.oracle.com/middleware/technologies/weblogic-server-installers-downloads.html](https://www.oracle.com/middleware/technologies/weblogic-server-installers-downloads.html)

&#x20;

2\.       Kemudian buat direktori untuk file jdk yang sudah kita download seperti gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (477).png" alt=""><figcaption></figcaption></figure>

```
mkdir /software
```

&#x20;

3\.       Selanjutnya jalankan perintah berikut

<figure><img src="../.gitbook/assets/image (478).png" alt=""><figcaption></figcaption></figure>

```
chmod 777 /software/
```

&#x20;

4\.       Kemudian pindahkan file jdk yang sudah di download kedalam direktori software, seperti pada gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (479).png" alt=""><figcaption></figcaption></figure>

&#x20;

5\.       Lalu jalankan perintah berikut untuk install file jdk yang sudah di download

<figure><img src="../.gitbook/assets/image (480).png" alt=""><figcaption></figcaption></figure>

```
sudo apt install ./jdk-11.0.27_linux-x64_bin.deb
```

tunggu sampai proses installasi selesai

&#x20;

6\.       Jalankan perintah berikut untuk mengetahui versi java

<figure><img src="../.gitbook/assets/image (481).png" alt=""><figcaption></figcaption></figure>

```
java -version
```

&#x20;

7\.       Langkah selanjutnya adalah proses install dan konfigurasi oracle weblogic server, pindahkan file nya kedalam diretori software seperti gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (482).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (483).png" alt=""><figcaption></figcaption></figure>

```
cp -r /home/cikal/fmw_14.1.1.0.0_wls_lite_generic.jar /software/ 
```

&#x20;

8\.       Selanjutnya kita membuat group seperti pada gambar dibawah ini

<figure><img src="../.gitbook/assets/image (484).png" alt=""><figcaption></figcaption></figure>

```
groupadd -g 1001 oinstall
```

&#x20;

9\.       Lalu membuat user seperti pada gambar dibawah ini

<figure><img src="../.gitbook/assets/image (485).png" alt=""><figcaption></figcaption></figure>

```
sudo useradd -u 1001 -g oinstall -m -d /home/oracle -s /bin/bash oracle
```

&#x20;

10\.   Langkah selanjutnya adalah membuat password untuk user oracle, seperti pada gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (486).png" alt=""><figcaption></figcaption></figure>

```
passwd oracle
```

Note\* saat mengisi password memang tidak tampil dan lanjut prosesnya sampai selesai.

&#x20;

11\.   Selanjutnya kita membuat ip domain weblogic di /etc/hosts, seperti pada gambar dibawah ini, sebelum nya kita cek terlebih dahulu ip yang sedang digunakan dan untuk memastikan semuanya sesuai

<figure><img src="../.gitbook/assets/image (487).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (488).png" alt=""><figcaption></figcaption></figure>

```
nano /etc/hosts
```

```
10.0.2.15    weblogic    weblogic.test
192.168.56.2    weblogic    weblogic.test
```

&#x20;

12\.   Langkah selanjutnya adalah merubah kepemilikkan hak akses seperti pada gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (489).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (490).png" alt=""><figcaption></figcaption></figure>

```
chmod -R 755 /software | chown -R oracle:oinstall /software
```

&#x20;

13\.   Dan membuat direktori seperti perintah pada gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (491).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (492).png" alt=""><figcaption></figcaption></figure>

```
mkdir /weblogic
```

```
chmod -R 755 /weblogic
```

&#x20;

14\.   Langkah selanjutnya ikuti pada gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (493).png" alt=""><figcaption></figcaption></figure>

```
mkdir -p /u01/app/oracle
```

```
mkdir -p /u01/app/oracle/config/domains
```

```
mkdir -p /u01/app/oracle/config/applications
```

```
chown -R oracle:oinstall /u01 | chmod -R 755 /u01
```

&#x20;

15\.   Langkah selanjutnya kita membuat skrip agar dapat berjalan di aplikasi remote yang Bernama MobaXterm dengan menjalankan perintah seperti gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (494).png" alt=""><figcaption></figcaption></figure>

```
cd /home/oracle
```

```
nano .bash_profile
```

<figure><img src="../.gitbook/assets/image (402).png" alt=""><figcaption></figcaption></figure>

```
# ~/.bash_profile for WebLogic and JDK 11
 
# Load .bashrc if exists
if [ -f ~/.bashrc ]; then
    . ~/.bashrc
fi
 
# User-specific environment and startup programs
# Home bin
PATH=$PATH:$HOME/.local/bin:$HOME/bin
 
# Oracle and WebLogic base
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/14.1.1.0.0
export MW_HOME=$ORACLE_HOME
export WLS_HOME=$MW_HOME/wlserver
 
# Domain
export DOMAIN_BASE=$ORACLE_BASE/config/domains
export DOMAIN_HOME=$DOMAIN_BASE/mydomain
 
# Java
export JAVA_HOME=/usr/lib/jvm/jdk-11.0.27
export PATH=$JAVA_HOME/bin:$PATH
 
# Optionally, add WebLogic scripts to PATH
export PATH=$WLS_HOME/server/bin:$PATH
```

&#x20;

16\.   Langkah selanjutnya adalah jalankan perintah dibawah ini, untuk mengaktifkan environment dan memastikan variabelnya seperti pada gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (496).png" alt=""><figcaption></figcaption></figure>

```
source .bash_profile
```

```
echo $ORACLE_HOME
```

```
echo $JAVA_HOME
```

```
echo $WLS_HOME
```

&#x20;

17\.   Selanjutnya kita install beberapa komponen seperti install X11 server dan tools dasar serta install library libXtst guna untuk installasi GUI dapat berjalan diaplikasi remote MobaXterm

<figure><img src="../.gitbook/assets/image (497).png" alt=""><figcaption></figcaption></figure>

```
sudo apt update
```

<figure><img src="../.gitbook/assets/image (498).png" alt=""><figcaption></figcaption></figure>

```
sudo apt install -y xorg xauth x11-apps 
```

tunggu sampai proses installasi selesai

<figure><img src="../.gitbook/assets/image (499).png" alt=""><figcaption></figcaption></figure>

```
sudo apt install -y libxtst6 libxtst-dev
```

tunggu sampai proses installasi selesai

&#x20;

18\.   Kemudian kita export display dengan menjalankan perintah dibawah ini.

<figure><img src="../.gitbook/assets/image (500).png" alt=""><figcaption></figcaption></figure>

```
export DISPLAY='192.168.56.2:0.0'
```

sesuaikan ip yang sedang digunakan

&#x20;

19\.   Selanjutnya menambahkan skrip serta enable beberapa skrip yang tidak aktif seperti pada gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (501).png" alt=""><figcaption></figcaption></figure>

( AddressFamily inet )

<figure><img src="../.gitbook/assets/image (504).png" alt=""><figcaption></figcaption></figure>

Hilangkan tanda pagar untuk enable skrip tersebut seperti pada gambar diatas ini.

&#x20;

20\.   Selanjutnya adalah restart service sshd dengan menjalankan perintah seperti pada gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (505).png" alt=""><figcaption></figcaption></figure>

```
systemctl restart ssh
```

```
systemctl status ssh
```

&#x20;

21\.   Langkah selanjutnya adalah merubah kepemilikkan hak akses file weblogic seperti pada gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (506).png" alt=""><figcaption></figcaption></figure>

```
chown oracle:oinstall fmw_14.1.1.0.0_wls_lite_generic.jar
```

&#x20;

22\.   Lanjut login ssh menggunakan aplikasi remote MobaXterm dengan opsi X seperti pada gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (507).png" alt=""><figcaption></figcaption></figure>

Terlihat pada gambar dibawah ini bahwa file .Xauthority does not exist

<figure><img src="../.gitbook/assets/image (508).png" alt=""><figcaption></figcaption></figure>

&#x20;Tandanya X11 forwading sudah aktif namun file .Xauthority belum ada untuk user oracle yang artinya tanpa file tersebut X11 forwading tidak bisa autentikasi ke X server dan GUI installer weblogic tidak dapat berjalan dengan sesuai.

&#x20;Untuk menangani permasalahan diatas, kita bisa membuat file .Xauthority yang baru dengan menjalankan perintah dibawah ini.

<figure><img src="../.gitbook/assets/image (509).png" alt=""><figcaption></figcaption></figure>

```
touch ~/.Xauthority
```

```
xauth generate $DISPLAY . trusted
```

```
xauth list
```

Cek permission dan rubah hak akses

```
ls -la ~/.Xauthority
```

```
chmod 600 ~/.Xauthority
```

&#x20;Jika berhasil coba jalankan perintah “xclock” adalah aplikasi untuk membuka GUI jam pada linux seperti gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (510).png" alt=""><figcaption></figcaption></figure>

&#x20;

23\.   Langkah selanjutnya adalah jalankan perintah berikut untuk menjalankan aplikasi weblogicnya

<figure><img src="../.gitbook/assets/image (511).png" alt=""><figcaption></figcaption></figure>

```
cd /software
```

```
java -jar /software/fwm_14.1.1.0.0_wls_lite_generic.jar
```



<figure><img src="../.gitbook/assets/image (512).png" alt=""><figcaption></figcaption></figure>

Dan akan muncul popup aplikasi java nya seperti pada Digambar bawah ini

&#x20;

24\.   Untuk Inventory Directory pindahkan seperti pada gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (403).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (514).png" alt=""><figcaption></figcaption></figure>

&#x20;

25\.   Kemudian klik OK dan Next.

&#x20;![](<../.gitbook/assets/image (515).png>)

<figure><img src="../.gitbook/assets/image (516).png" alt=""><figcaption></figcaption></figure>

&#x20;&#x20;

26\.   Kemudian pilih Skip Auto Updates dan klik next.

<figure><img src="../.gitbook/assets/image (404).png" alt=""><figcaption></figcaption></figure>

&#x20;

27\.   Kemudian pilih folder seperti pada gambar dibawah ini, jika belum ada maka buatlah terlebih dahulu sesuaikan pada gambar dibawah ini dan klik next.

<figure><img src="../.gitbook/assets/image (405).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (519).png" alt=""><figcaption></figcaption></figure>

&#x20;

28\.   Pilih Weblogic Server dan klik next.

<figure><img src="../.gitbook/assets/image (520).png" alt=""><figcaption></figcaption></figure>

&#x20;

29\.   Tunggu proses cek selesai dan klik next

<figure><img src="../.gitbook/assets/image (521).png" alt=""><figcaption></figcaption></figure>

&#x20;

30\.   Pilih install

<figure><img src="../.gitbook/assets/image (522).png" alt=""><figcaption></figcaption></figure>

&#x20;

Dan tunggu sampai proses installasi selesai

<figure><img src="../.gitbook/assets/image (523).png" alt=""><figcaption></figcaption></figure>

&#x20;

31\.   Proses installasi selesai dan klik next

<figure><img src="../.gitbook/assets/image (406).png" alt=""><figcaption></figcaption></figure>

&#x20;

32\.   Selanjutnya hilang pada pada Automaically launch dan klik Finish

<figure><img src="../.gitbook/assets/image (525).png" alt=""><figcaption></figcaption></figure>

&#x20;

33\.   Sampai disini proses installasi WebLogic Server selesai dan selanjutnya adalah konfigurasi dengan membuka port 7001 dari firewall, jalankan perintah seperti pada gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (526).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (527).png" alt=""><figcaption></figcaption></figure>

```
sudo ufw allow 7001/tcp
```

```
sudo ufw status
```

&#x20;&#x20;

34\.   Langkah selanjutnya adalah mengatur konfigurasi weblogic seperti pada gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (407).png" alt=""><figcaption></figcaption></figure>

```
cd /u01/app/oracle/product/14.1.1.0.0/oracle_common/common/bin
```

```
./config.sh
```

&#x20;

35\.   Selanjutnya pergi ke popup Fusion Middleware Configuration Wizard dan pilih create a new domain kemudian arahkan direktori ke tujuan /u01/app/oracle/config/domains/mydomain dan klik next, seperti pada gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (408).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (503).png" alt=""><figcaption></figcaption></figure>

&#x20;

36\.   Kemudian pilih Create Domain Using Product Template dan klik Next.

<figure><img src="../.gitbook/assets/image (443).png" alt=""><figcaption></figcaption></figure>

&#x20;

37\.   Kemudian buat Account dan pilih Next.

<figure><img src="../.gitbook/assets/image (444).png" alt=""><figcaption></figcaption></figure>



38\.   Kemudian pilih Production pada opsi pilihan Domain Mode dan klik Next.

<figure><img src="../.gitbook/assets/image (409).png" alt=""><figcaption></figcaption></figure>



39\.   Kemudian centang pada Administration Server & Node Manager dan klik Next.

<figure><img src="../.gitbook/assets/image (448).png" alt=""><figcaption></figcaption></figure>

&#x20;

40\.   Kemudian klik next

<figure><img src="../.gitbook/assets/image (410).png" alt=""><figcaption></figcaption></figure>

&#x20;

41\.   Kemudian plih Per Domain Default Location pada opsi Node Manager Type dan isi Account  pada Node Manager Credentials.

<figure><img src="../.gitbook/assets/image (411).png" alt=""><figcaption></figcaption></figure>



42\.   Selanjutnya klik Create.

<figure><img src="../.gitbook/assets/image (412).png" alt=""><figcaption></figcaption></figure>

&#x20;

43\.   Tunggu proses konfigurasi selesai dan klik next.

<figure><img src="../.gitbook/assets/image (414).png" alt=""><figcaption></figcaption></figure>

44\.   Domain berhasil dibuat, catat Domain Location dan URL Admin Server, kemudian klik Finish

<figure><img src="../.gitbook/assets/image (415).png" alt=""><figcaption></figcaption></figure>

```
/u01/app/oracle/config/domains/mydomain
```

```
http://cinosta:7001/console
```

&#x20;

45\.   Kemudian running weblogic dengan menjalankan perintah sebagai berikut

<figure><img src="../.gitbook/assets/image (416).png" alt=""><figcaption></figcaption></figure>

```
cd /u01/app/oracle/config/domains/mydomain
```

```
./startWebLogic.sh
```

&#x20;

46\.   Service weblogic sudah running dan masukkan user dan pass yang sudah dibuat sebelumnya

<figure><img src="../.gitbook/assets/image (455).png" alt=""><figcaption></figcaption></figure>

&#x20;

47\.   Kemudian kita akses [http://192.168.56.2:7001/console](http://192.168.56.2:7001/console) pada web browser kita dan masukkan passwordnya.

<figure><img src="../.gitbook/assets/image (417).png" alt=""><figcaption></figcaption></figure>

```
http://192.168.56.2:7001/console
```

&#x20;

48\.   Weblogic berhasil masuk dan dijalankan

<figure><img src="../.gitbook/assets/image (418).png" alt=""><figcaption></figcaption></figure>
