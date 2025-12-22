# Setup Load Balancer NGINX

Contoh kasus saya punya 4 VM dengan detail sebagai berikut :

1. ip 192.168.56.3:81 yang sudah terinstall nginx
2. ip 192.168.56.4:8080 yang sudah terinstall tomcat
3. ip 192.168.56.5:7001 yang sudah terinstall weblogic
4. ip 192.168.56.6:83 yang sudah terinstall php

Dan saya ingin ip 192.168.56.3 menjadi sebagai load balance dengan menggunakan NGINX, dan berikut step by stepnya

1. Jalankan perintah dibawah ini untuk membuat file

```
sudo nano /etc/nginx/conf.d/loadbalance.conf
```

2. Lalu isikan file tersebut isi seperti dibawah ini

```
upstream aplikasi_backend {
    least_conn;
    server 192.168.56.5:7001;      # WebLogic
    server 192.168.56.4:8080;    # Tomcat
    server 192.168.56.6:83;    # PHP
}

server {

        listen 8888;

    server_name _;

    location / {
        proxy_pass http://aplikasi_backend;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

<figure><img src=".gitbook/assets/image (34).png" alt=""><figcaption></figcaption></figure>

3. Test konfigurasi

```
sudo nginx -t
```

4. Jika OK:

```bash
sudo systemctl reload nginx
```

5. Buka firewall

```
sudo ufw allow 8888
sudo ufw allow 7001
sudo ufw allow 8080
sudo ufw allow 83
```

<figure><img src=".gitbook/assets/image (35).png" alt=""><figcaption></figcaption></figure>

Buka browser dan akses url dibawah ini (Jika berhasil project akan muncul dan jika di refresh berkali-kali project akan berganti-ganti)

```
http://192.168.56.3:8888/home/
```

<figure><img src=".gitbook/assets/image (36).png" alt=""><figcaption></figcaption></figure>

atau bisa dengan jalankan perintah ini di VM NGINX sebagai Load Balancer

```
curl http://192.168.56.5:7001
curl http://192.168.56.4:8080
curl http://192.168.56.6:83
```
