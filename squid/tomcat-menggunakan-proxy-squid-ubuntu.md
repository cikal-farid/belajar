# Tomcat menggunakan proxy Squid (Ubuntu)

**Agar aplikasi Java di dalam Tomcat**, **mengakses internet atau server lain lewat proxy Squid di VM1 (192.168.56.14:2113)**

Langkah awal yaitu edit file service tomcat nya

```
sudo nano /etc/systemd/system/tomcat.service
```

```
[Unit]
Description=Apache Tomcat 10 Web Application Container
After=network.target

[Service]
Type=forking

User=tomcat
Group=tomcat

Environment="JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64"
Environment="CATALINA_HOME=/opt/tomcat/latest"
Environment="CATALINA_BASE=/opt/tomcat/latest"
Environment="CATALINA_PID=/opt/tomcat/latest/temp/tomcat.pid"
Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC -Dhttp.proxyHost=192.168.56.14 -Dhttp.proxyPort=2113 -Dhttps.proxyHost=192.168.56.14 -Dhttps.proxyPort=2113 -Dhttp.nonProxyHosts=localhost\\|127.0.0.1"

ExecStart=/opt/tomcat/latest/bin/startup.sh
ExecStop=/opt/tomcat/latest/bin/shutdown.sh

Restart=on-failure

[Install]
WantedBy=multi-user.target

```

<figure><img src="../.gitbook/assets/image (244).png" alt=""><figcaption></figcaption></figure>

```
sudo systemctl daemon-reload
```

```
sudo systemctl restart tomcat
```

```
sudo systemctl status tomcat
```

Bisa juga dengan cara dibawah ini

1. Langkah awal yaitu membuat 1 file bernama setenv.sh didalam folder bin tomcat

```
sudo nano /opt/tomcat/latest/bin/setenv.sh
```

```
#!/bin/bash
export CATALINA_OPTS="$CATALINA_OPTS \
  -Dhttp.proxyHost=192.168.56.14 \
  -Dhttp.proxyPort=3128 \
  -Dhttps.proxyHost=192.168.56.14 \
  -Dhttps.proxyPort=3128 \
  -Dhttp.nonProxyHosts=localhost\\|127.0.0.1 \
  -Xms512M \
  -Xmx1024M \
  -server \
  -XX:+UseParallelGC"
```

<figure><img src="../.gitbook/assets/image (651).png" alt=""><figcaption></figcaption></figure>

Simpan → keluar.

2. Perbaiki izin dan kepemilikan

```
sudo chown tomcat:tomcat /opt/tomcat/latest/bin/setenv.sh
sudo chmod +x /opt/tomcat/latest/bin/setenv.sh
```

3. Restart service tomcat

```
sudo systemctl restart tomcat
```

4. Kemudian bisa kita akses menggunakan perintah dibawah ini

```
curl -x http://192.168.56.14:3128 -I http://192.168.56.15:8080/home/
```

<figure><img src="../.gitbook/assets/image (652).png" alt=""><figcaption></figcaption></figure>

5. Bisa juga kita cek log apakah tomcat sudah menggunakan squid proxy atau belum

```
ps -ef | grep java
```

```
ps aux | grep java
```

<figure><img src="../.gitbook/assets/image (653).png" alt=""><figcaption></figcaption></figure>

6. Kemudian lanjutkan bisa kita cek log dari VM1 sebagai server squid proxy

```
sudo tail -f /var/log/squid/access.log
```

<figure><img src="../.gitbook/assets/image (654).png" alt=""><figcaption></figcaption></figure>

***

#### (Opsional) Mengatur Proxy Secara Global di VM2

Jika Anda ingin _semua_ perintah di terminal VM2 (seperti `apt update`, `wget`, dll.) otomatis menggunakan proxy tanpa perlu mengetik flag `-x` setiap saat, Anda bisa mengatur _environment variables_.

Tambahkan baris berikut ke file `~/.bashrc` Anda:

```
nano ~/.bashrc
```

Tambahkan di bagian paling bawah file:

```
# Set proxy untuk HTTP dan HTTPS
export http_proxy="http://192.168.56.14:3128"
export https_proxy="http://192.168.56.14:3128"

# Opsional: Jika ada IP yang tidak ingin dilewatkan proxy (misal server VM1 itu sendiri)
export no_proxy="localhost,127.0.0.1,192.168.56.14"
```

Simpan file (Ctrl+O, Enter) dan tutup (Ctrl+X). Lalu, terapkan pengaturan:

```
source ~/.bashrc
```

**Tetapi** APT _kadang_ tetap mengabaikan set environment global ini ketika dijalankan sebagai root → sehingga **cara paling konsisten** adalah tetap membuat file `/etc/apt/apt.conf.d/80proxy`.

Kita jika perlu membuat file ini agar dapat apt via http yang berjalan lewat squid di ubuntu

```
sudo nano /etc/apt/apt.conf.d/80proxy
```

```
Acquire::http::Proxy "http://192.168.56.14:3128/";
Acquire::https::Proxy "http://192.168.56.14:3128/";
Acquire::http::No-Cache "true";
Acquire::https::No-Cache "true";
```

<figure><img src="../.gitbook/assets/image (655).png" alt=""><figcaption></figcaption></figure>

Restart service

```
sudo systemctl restart systemd-resolved
```

Sekarang, Anda bisa menguji hanya dengan:

Ini akan otomatis menggunakan proxy

```
curl -x http://192.168.56.14:3128 -I http://detik.com
```

(Hasil: Access Denied)

```
curl -x http://192.168.56.14:3128 -I http://google.com
```

Perintah untuk update sistem dan sekaligus menguji apakah bisa mendapatkan inet

```
sudo apt update
```
