# Nginx (Ubuntu)

#### Nginx

software web server dan reverse proxy yang bertugas menerima permintaan dari pengguna (browser), memproses atau meneruskan permintaan itu ke aplikasi, lalu mengembalikan hasilnya ke pengguna dengan sangat cepat dan efisien.

Bayangkan kamu punya **Tomcat** di port 8080 dan **WebLogic** di 7001.\
Kamu tidak ingin user tahu port-port itu, karena terlalu teknis.

Maka NGINX bertugas **menyembunyikan backend**

Logika ini disebut **reverse proxy**, karena NGINX bertindak sebagai _perantara terbalik_ — dia menerima request atas nama backend server.

Instalasi dan Konfigurasi Nginx

1. Update sistem dan install NGINX:

pindah root untuk update

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
sudo apt install nginx -y
```

2. Cek status service :

```
sudo systemctl status nginx
```

3. Edit konfigurasi utama untuk ganti port karena ingin load balance

```
sudo nano /etc/nginx/sites-available/default
```

Cari baris ini:

```
listen 80 default_server;
listen [::]:80 default_server;
```

Ubah jadi :

```
listen 81 default_server;
listen [::]:81 default_server;
```

4. Cek syntax:

```
sudo nginx -t
```

Kalau “syntax is ok” → restart service:

```
sudo systemctl restart nginx
```

5. Lalu deploy apps web mu di folder /var/www/html/

```
cd /var/www/html/
```

```
sudo cp -r folder_procjet/ /var/www/html/
```

6. Cek Apps di Web Browser
