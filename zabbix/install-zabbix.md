# Install Zabbix

Install **Zabbix Server + Web (Apache) + MariaDB + Agent** di **Ubuntu 22.04**, dengan **HTTPS** di IP **192.168.56.18** (pakai **self-signed certificate**).

> Repo + paket Zabbix ambil dari repo resmi Zabbix. ([zabbix.com](https://www.zabbix.com/documentation/current/en/manual/installation/install_from_packages?utm_source=chatgpt.com))\
> DB disarankan `utf8mb4` + `utf8mb4_bin`, dan saat import schema biasanya perlu `log_bin_trust_function_creators=1`. ([zabbix.com](https://www.zabbix.com/documentation/current/en/manual/appendix/install/db_scripts?utm_source=chatgpt.com))

***

### 0) Set variabel (ubah password DB sesuai keinginanmu)

Copy-paste ini dulu:

```bash
export ZBX_IP="192.168.56.18"
export DB_NAME="zabbix"
export DB_USER="zabbix"
export DB_PASS="zabbix"
export PHP_TZ="Asia/Jakarta"
```

<figure><img src="../.gitbook/assets/image (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

***

### 1) Update OS + set timezone OS

```bash
sudo apt update && sudo apt -y upgrade
sudo timedatectl set-timezone "$PHP_TZ"
```

<figure><img src="../.gitbook/assets/image (2) (1) (1).png" alt=""><figcaption></figcaption></figure>

Tunggu sampai proses selesai

***

### 2) Tambah repo resmi Zabbix (contoh: Zabbix 7.4) + install paket

Paket repo untuk Ubuntu 24.04 tersedia di repo resmi. ([repo.zabbix.com](https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/?utm_source=chatgpt.com))

```bash
sudo apt install -y ufw
sudo apt install -y dialog
sudo apt install -y wget ca-certificates gnupg
cd /tmp

wget -O zabbix-release.deb \
  https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu24.04_all.deb

sudo dpkg -i zabbix-release.deb
sudo apt update
```

<figure><img src="../.gitbook/assets/image (3) (1) (1).png" alt=""><figcaption></figcaption></figure>

Tunggu sampai proses selesai

Install Zabbix + Apache + MariaDB:

```bash
sudo apt install -y apache2 mariadb-server \
  zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts \
  zabbix-agent
```

<figure><img src="../.gitbook/assets/image (4) (1).png" alt=""><figcaption></figcaption></figure>

***

### 3) Buat database + user untuk Zabbix (MariaDB)

Zabbix rekomendasikan `utf8mb4` + `utf8mb4_bin`. ([zabbix.com](https://www.zabbix.com/documentation/current/en/manual/appendix/install/db_scripts?utm_source=chatgpt.com))

```bash
sudo mysql <<SQL
CREATE DATABASE ${DB_NAME} CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER '${DB_USER}'@'localhost' IDENTIFIED BY '${DB_PASS}';
GRANT ALL PRIVILEGES ON ${DB_NAME}.* TO '${DB_USER}'@'localhost';
SET GLOBAL log_bin_trust_function_creators = 1;
FLUSH PRIVILEGES;
SQL
```

<figure><img src="../.gitbook/assets/image (5) (1).png" alt=""><figcaption></figcaption></figure>

***

### 4) Import schema awal Zabbix ke database

```bash
zcat /usr/share/zabbix/sql-scripts/mysql/server.sql.gz \
 | mysql --default-character-set=utf8mb4 -u"${DB_USER}" -p"${DB_PASS}" "${DB_NAME}"
```

<figure><img src="../.gitbook/assets/image (6) (1).png" alt=""><figcaption></figcaption></figure>

Tunggu sampai proses selesai

Balikin setting `log_bin_trust_function_creators` (opsional tapi rapi):

```bash
sudo mysql -e "SET GLOBAL log_bin_trust_function_creators = 0;"
```

<figure><img src="../.gitbook/assets/image (7) (1).png" alt=""><figcaption></figcaption></figure>

(Alasan setting ini dibutuhkan saat import schema dijelaskan di dokumentasi Zabbix.) ([zabbix.com](https://www.zabbix.com/documentation/current/en/manual/appendix/install/db_scripts?utm_source=chatgpt.com))

***

### 5) Isi DB password di config Zabbix Server

```bash
sudo sed -i \
  -e "s/^#\s*DBPassword=.*/DBPassword=${DB_PASS}/" \
  -e "s/^DBPassword=.*/DBPassword=${DB_PASS}/" \
  /etc/zabbix/zabbix_server.conf

# kalau ternyata baris DBPassword belum ada, tambahkan:
sudo grep -q '^DBPassword=' /etc/zabbix/zabbix_server.conf || echo "DBPassword=${DB_PASS}" | sudo tee -a /etc/zabbix/zabbix_server.conf > /dev/null
```

***

### 6) Set timezone PHP (biar web frontend aman)

```bash
PHPVER="$(php -r 'echo PHP_MAJOR_VERSION.".".PHP_MINOR_VERSION;')"
PHPINI="/etc/php/${PHPVER}/apache2/php.ini"

# uncomment / set date.timezone
sudo sed -i "s~^;date.timezone =.*~date.timezone = ${PHP_TZ}~" "$PHPINI"
grep -q "^date.timezone" "$PHPINI" || echo "date.timezone = ${PHP_TZ}" | sudo tee -a "$PHPINI" > /dev/null

sudo systemctl restart apache2
```

<figure><img src="../.gitbook/assets/image (8) (1).png" alt=""><figcaption></figcaption></figure>

***

### 7) Enable & start semua service

```bash
sudo systemctl enable --now mariadb apache2 zabbix-server zabbix-agent
sudo systemctl status zabbix-server --no-pager
```

<figure><img src="../.gitbook/assets/image (9) (1).png" alt=""><figcaption></figcaption></figure>

***

### 8) Konfigurasi Zabbix Agent (karena server ini juga dimonitor)

```bash
sudo sed -i \
  -e "s/^Server=.*/Server=127.0.0.1,${ZBX_IP}/" \
  -e "s/^ServerActive=.*/ServerActive=${ZBX_IP}/" \
  -e "s/^Hostname=.*/Hostname=Zabbix-Server/" \
  /etc/zabbix/zabbix_agentd.conf

sudo systemctl restart zabbix-agent
```

<figure><img src="../.gitbook/assets/image (10) (1).png" alt=""><figcaption></figcaption></figure>

***

### 9) Firewall (UFW) buka port 80, 443, 10051, 10050

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 10051/tcp
sudo ufw allow 10050/tcp
sudo ufw --force enable
sudo ufw status
```

<figure><img src="../.gitbook/assets/image (11) (1).png" alt=""><figcaption></figcaption></figure>

***

### 10) Aktifkan HTTPS di Apache (Self-Signed) + redirect HTTP → HTTPS

Zabbix juga punya best-practice “redirect ke SSL URL”. ([zabbix.com](https://www.zabbix.com/documentation/current/en/manual/best_practices/security/web_server?utm_source=chatgpt.com))

#### 10.1 Buat sertifikat self-signed (sudah ada SAN untuk IP)

```bash
sudo mkdir -p /etc/ssl/zabbix

sudo tee /etc/ssl/zabbix/openssl.cnf > /dev/null <<EOF
[req]
default_bits = 4096
prompt = no
default_md = sha256
distinguished_name = dn
x509_extensions = v3_req

[dn]
C = ID
ST = Jakarta
L = Jakarta
O = Lab
OU = Zabbix
CN = ${ZBX_IP}

[v3_req]
subjectAltName = @alt_names
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

[alt_names]
IP.1 = ${ZBX_IP}
DNS.1 = zabbix.local
EOF

sudo openssl req -x509 -nodes -days 825 -newkey rsa:4096 \
  -keyout /etc/ssl/zabbix/zabbix.key \
  -out /etc/ssl/zabbix/zabbix.crt \
  -config /etc/ssl/zabbix/openssl.cnf

sudo chmod 600 /etc/ssl/zabbix/zabbix.key
```

#### 10.2 Enable module SSL + buat VirtualHost 80 redirect + VirtualHost 443

```bash
sudo a2enmod ssl rewrite headers

# optional: set ServerName biar apache ga warning
echo "ServerName ${ZBX_IP}" | sudo tee /etc/apache2/conf-available/servername.conf > /dev/null
sudo a2enconf servername
```

<figure><img src="../.gitbook/assets/image (12) (1).png" alt=""><figcaption></figcaption></figure>

Buat vhost HTTP (redirect ke HTTPS):

```bash
sudo tee /etc/apache2/sites-available/zabbix-http.conf > /dev/null <<EOF
<VirtualHost *:80>
  ServerName ${ZBX_IP}
  RewriteEngine On
  RewriteRule ^/(.*)$ https://${ZBX_IP}/\$1 [R=301,L]
</VirtualHost>
EOF
```

<figure><img src="../.gitbook/assets/image (13) (1).png" alt=""><figcaption></figcaption></figure>

Buat vhost HTTPS:

```bash
sudo tee /etc/apache2/sites-available/zabbix-ssl.conf > /dev/null <<EOF
<VirtualHost *:443>
  ServerName ${ZBX_IP}

  SSLEngine On
  SSLCertificateFile /etc/ssl/zabbix/zabbix.crt
  SSLCertificateKeyFile /etc/ssl/zabbix/zabbix.key

  # kalau buka https://IP/ langsung, lempar ke /zabbix
  RedirectMatch 302 ^/$ /zabbix

  # sedikit security header (opsional)
  Header always set X-Content-Type-Options "nosniff"
  Header always set X-Frame-Options "SAMEORIGIN"
</VirtualHost>
EOF
```

<figure><img src="../.gitbook/assets/image (14) (1).png" alt=""><figcaption></figcaption></figure>

Aktifkan site baru, matikan default, lalu reload Apache:

```bash
sudo a2dissite 000-default.conf || true
sudo a2ensite zabbix-http.conf zabbix-ssl.conf

sudo apache2ctl configtest
sudo systemctl reload apache2
```

<figure><img src="../.gitbook/assets/image (15) (1).png" alt=""><figcaption></figcaption></figure>

**generate locale `en_US.UTF-8` (dan opsional `id_ID.UTF-8`)** lalu restart Apache.

```bash
# 1) install paket locales
sudo apt update
sudo apt install -y locales

# 2) aktifkan locale yang dibutuhkan (en_US) + (opsional id_ID)
sudo sed -i 's/^# *en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
sudo sed -i 's/^# *id_ID.UTF-8 UTF-8/id_ID.UTF-8 UTF-8/' /etc/locale.gen || true

# 3) generate locales
sudo locale-gen

# 4) set default locale server
sudo update-locale LANG=en_US.UTF-8

# 5) pastikan locale sudah ada
locale -a | egrep 'en_US|id_ID' || true

# 6) restart apache
sudo systemctl restart apache2

```

Tunggu sampai proses selesai

***

### 11) Akses Zabbix via HTTPS

Buka browser:

* [**https://192.168.56.18/zabbix**](https://192.168.56.18/zabbix)

<figure><img src="../.gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure>

Lanjut klik Next Step

<figure><img src="../.gitbook/assets/image (17).png" alt=""><figcaption></figcaption></figure>

Isi Password = zabbix dan Lanjut klik Next Step

> Karena sertifikat self-signed, browser akan warning. Itu normal. Kalau mau “trusted” tanpa warning, kita butuh domain + CA/Let’s Encrypt (biasanya bukan untuk IP private).

<figure><img src="../.gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>

Isi Server Name dan tidak perlu di centang untuk checkbox web interface karena saat ini https sudah di handle oleh apache sebelumnya. Dan lanjut Next Step.

<figure><img src="../.gitbook/assets/image (19).png" alt=""><figcaption></figcaption></figure>

Lanjut klik Next Step

<figure><img src="../.gitbook/assets/image (20).png" alt=""><figcaption></figcaption></figure>

Lanjut klik Finish

Login default:

* **User:** `Admin`
* **Pass:** `zabbix` ([zabbix.com](https://www.zabbix.com/documentation/current/en/manual/quickstart/login?utm_source=chatgpt.com))

<figure><img src="../.gitbook/assets/image (21).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (22).png" alt=""><figcaption></figcaption></figure>

Jika ingin melihat detail Graphnya lanjut ikuti arahan dibawah ini

<figure><img src="../.gitbook/assets/image (23).png" alt=""><figcaption></figcaption></figure>

Klik Zabbix server

<figure><img src="../.gitbook/assets/image (24).png" alt=""><figcaption></figcaption></figure>

Klik Graphs

<figure><img src="../.gitbook/assets/image (25).png" alt=""><figcaption></figcaption></figure>

***
