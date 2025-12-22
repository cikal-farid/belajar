# Tomcat (Ubuntu)

Merupakan implementasi dari spesifikasi Java Servlet dan JSP, yang menyediakan lingkungan (container) agar kode Java tersebut bisa berjalan, mengelola siklus hidupnya, dan terhubung ke jaringan (HTTP) untuk melayani permintaan pengguna.

Jadi Tomcat ialah aplikasi server yang digunakan untuk menjalankan aplikasi web berbasis Java.

Tomcat dikembangkan oleh Apache Software Foundation, dan biasanya digunakan untuk menjalankan file .war, yaitu format paket aplikasi web Java.

Instalasi dan Konfigurasi Tomcat

Jalankan semua perintah dengan hak `sudo` atau user root.

1. Pindah root untuk update

```
sudo su -
```

```
sudo apt update && apt-get upgrade -y
```

atau

```
sudo apt update
```

```
sudo apt install openjdk-11-jdk -y
```

2. Cek versi Java:

```
java -version
```

Minimal harus ada Java 11 untuk Tomcat 10.

3. Masuk folder /opt

```
cd /opt
```

4. Buat direktori tomcat

```
sudo mkdir tomcat
```

5. Masuk ke direktori Tomcat

```
cd tomcat
```

6. Download Tomcat

```
sudo wget https://downloads.apache.org/tomcat/tomcat-10/v10.1.48/bin/apache-tomcat-10.1.48.tar.gz
```

7. Ekstrak Tomcat

```
sudo tar -xvzf apache-tomcat-10.1.48.tar.gz
```

8. Pindahkan isi ekstrakan ke folder latest

```
sudo mv apache-tomcat-10.1.48 latest
```

Set permission:

9. buat group dan user

```
sudo useradd -r -m -U -d /opt/tomcat -s /bin/false tomcat
```

```
sudo chown -R tomcat:tomcat /opt/tomcat
```

```
sudo chmod -R 755 /opt/tomcat
```

10. Buat file service untuk tomcat

```
sudo nano /etc/systemd/system/tomcat.service
```

isi dengan skrip dibawah ini

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
Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"

ExecStart=/opt/tomcat/latest/bin/startup.sh
ExecStop=/opt/tomcat/latest/bin/shutdown.sh

Restart=on-failure

[Install]
WantedBy=multi-user.target

```

11. Simpan, lalu aktifkan:

```
sudo systemctl daemon-reload
```

```
sudo systemctl enable tomcat
```

```
sudo systemctl start tomcat
```

```
sudo systemctl status tomcat
```

12. buka akses firewall jika belum, seperti jalankan perintah dibawh ini

```
sudo ufw allow 8080/tcp
```

13. Deploy Aplikasi ke Tomcat

Salin file `.war` kamu ke folder:

```
/opt/tomcat/latest/webapps/
```

```
sudo cp home.war /opt/tomcat/latest/webapps/
```

```
sudo chown tomcat:tomcat /opt/tomcat/latest/webapps/home.war
```

14. Config tomcatnya

```
sudo nano /opt/tomcat/latest/conf/tomcat-users.xml
```

```
    <role rolename="manager-gui"/>
    <role rolename="manager-script"/>
    <role rolename="manager-jmx"/>
    <role rolename="manager-status"/>
    <role rolename="admin-gui"/>
    <user password="cikal" roles="manager-gui,manager-script,manager-jmx,manager-status,admin-gui" username="cikal"/>
```

```
sudo nano /opt/tomcat/latest/webapps/manager/WEB-INF/web.xml
```

```
<max-file-size>524288000</max-file-size>
<max-request-size>524288000</max-request-size>

untuk atur besaran file yang di upload 
```

15. Buat File manager.xml

```
sudo nano /opt/tomcat/latest/conf/Catalina/localhost/manager.xml
```

Lalu isi

```
<Context privileged="true" antiResourceLocking="false"
        docBase="${catalina.home}/webapps/manager">
    <Valve className="org.apache.catalina.valves.RemoteAddrValve" allow="^.*$" />
</Context>
```

16. Buat File host-manager.xml

```
sudo nano /opt/tomcat/latest/conf/Catalina/localhost/host-manager.xml
```

Lalu isi

```
<Context privileged="true" antiResourceLocking="false"
        docBase="${catalina.home}/webapps/host-manager">
    <Valve className="org.apache.catalina.valves.RemoteAddrValve" allow="^.*$" />
</Context>
```

17. Restart service tomcat

```
sudo systemctl restart tomcat
```

```
http://192.168.56.15:8080/
```

```
http://192.168.56.15:8080/home/
```
