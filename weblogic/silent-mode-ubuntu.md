# Silent Mode (Ubuntu)

Silent Mode

Adalah mode instalasi atau konfigurasi otomatis tanpa antarmuka grafis (GUI) seperti pada tahapan step by step sebelumnya. Yang artinya WebLogic akan membaca semua jawaban (seperti lokasi instalasi, password admin, nama domain, dll) dari file _response file (.rsp)_ dan menjalankan sepenuhnya di latar belakang.

Berikut Langkah-langkah menjalankan silent mode :

1\.       Pertama kita akan membuat sebuah file _shell script_ yang nantinya akan menjalankan secara otomatis keperluan instalasi atau konfigurasi WebLogic, dengan mengikuti gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (419).png" alt=""><figcaption></figcaption></figure>

```
sudo nano install_konfigurasi_weblogic.sh
```

<figure><img src="../.gitbook/assets/image (460).png" alt=""><figcaption></figcaption></figure>

```
#!/bin/bash
# ==========================================================
#  Oracle WebLogic 14.1.1 Silent Install + Domain Creation
#  Newbie_Jadoel
# ==========================================================
 
set -e
 
# --- Konfigurasi variabel utama ---
INSTALLER_JAR="/home/cikal/fmw_14.1.1.0.0_wls_lite_generic.jar"
JAVA_PATH=$(readlink -f $(which java))
JAVA_HOME=$(dirname $(dirname "$JAVA_PATH"))
ORACLE_BASE="/opt/oracle"
ORACLE_HOME="$ORACLE_BASE/weblogic"
DOMAIN_HOME="$ORACLE_HOME/user_projects/domains/base_domain"
RESPONSE_FILE="/home/cikal/silent.rsp"
ORAINST_LOC="/home/cikal/oraInst.loc"
CREATE_DOMAIN_PY="/home/cikal/create_domain.py"
 
echo "=== [1/8] Menyiapkan direktori instalasi WebLogic ==="
sudo mkdir -p "$ORACLE_BASE"
sudo chown -R $(whoami):$(whoami) "$ORACLE_BASE"
 
echo "=== [2/8] Membuat file oraInst.loc ==="
sudo rm -f "$ORAINST_LOC"
cat > "$ORAINST_LOC" <<EOF
inventory_loc=$ORACLE_BASE/oraInventory
inst_group=$(whoami)
EOF
sudo chown $(whoami):$(whoami) "$ORAINST_LOC"
 
echo "=== [3/8] Membuat file silent.rsp ==="
cat > "$RESPONSE_FILE" <<EOF
[ENGINE]
Response File Version=1.0.0.0.0
BEAHOME=$ORACLE_HOME
INSTALL_TYPE=WebLogic Server
 
[GENERIC]
DECLINE_SECURITY_UPDATES=true
SECURITY_UPDATES_VIA_MYORACLESUPPORT=false
ORACLE_BASE=$ORACLE_BASE
ORACLE_HOME=$ORACLE_HOME
EOF
 
echo "=== [4/8] Menjalankan instalasi WebLogic Server (silent mode) ==="
"$JAVA_HOME/bin/java" -jar "$INSTALLER_JAR" -silent -responseFile "$RESPONSE_FILE" -invPtrLoc "$ORAINST_LOC" || {
  echo "❌ Instalasi WebLogic gagal."
  exit 1
}
echo "✅ Instalasi WebLogic selesai!"
 
echo "=== [5/8] Membuat file create_domain.py ==="
cat > "$CREATE_DOMAIN_PY" <<'EOF'
# -*- coding: utf-8 -*-
readTemplate("/opt/oracle/weblogic/wlserver/common/templates/wls/wls.jar")
 
cd('Servers/AdminServer')
set('ListenAddress','')
set('ListenPort',7001)
 
cd('/')
cd('Security/base_domain/User/weblogic')
cmo.setPassword('Welcome123')
 
setOption('OverwriteDomain', 'true')
setOption('ServerStartMode', 'dev')
writeDomain('/opt/oracle/weblogic/user_projects/domains/base_domain')
closeTemplate()
 
print('✅ Domain berhasil dibuat di /opt/oracle/weblogic/user_projects/domains/base_domain')
EOF
 
echo "=== [6/8] Membuat domain WebLogic (WLST silent mode) ==="
/opt/oracle/weblogic/oracle_common/common/bin/wlst.sh "$CREATE_DOMAIN_PY"
 
if [ $? -ne 0 ]; then
  echo "❌ Domain gagal dibuat."
  exit 1
fi
echo "✅ Domain berhasil dibuat!"
 
echo "=== [7/8] Memberi izin eksekusi script start/stop ==="
chmod +x "$DOMAIN_HOME/bin/startWebLogic.sh" 2>/dev/null || true
chmod +x "$DOMAIN_HOME/bin/stopWebLogic.sh" 2>/dev/null || true
 
echo "=== [8/8] Selesai! Untuk menjalankan WebLogic, gunakan: ==="
echo "cd $DOMAIN_HOME/bin && ./startWebLogic.sh"
echo "cd $DOMAIN_HOME/bin && ./stopWebLogic.sh"
 
echo "🎉 Semua langkah instalasi & konfigurasi Oracle WebLogic Server selesai tanpa error!"
```

2\.       Kemudian kita rubah kepemilikan hak akses seperti gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (420).png" alt=""><figcaption></figcaption></figure>

```
chmod +x /home/cikal/install_konfigurasi_weblogic.sh
```

&#x20;

3\.       Selanjutnya adalah memindahkan 2 file instalasi seperti java 11 dan WebLogic, disini saya sudah menyiapkan 2 file tersebut dari laptop host dan akan saya pindahkan ke VM Ubuntu Server saya menggunakan WinSCP, ikuti step by step seperti gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (421).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (422).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (465).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (423).png" alt=""><figcaption></figcaption></figure>

Jika ip vm dan host sudah terhubung di WinSCP, sangatlah mudah hanya tinggal _drag and drop_ file yang kita inginkan ke tujuan, seperti gambar diatas ini.

&#x20;

4\.       Setelah itu kita install terlebih dahulu java 11 nya yang sudah kita siapkan sebelumnya dengan menjalankan perintah seperti gambar dibawah ini.

<figure><img src="../.gitbook/assets/image (424).png" alt=""><figcaption></figcaption></figure>

```
sudo dpkg -i jdk-11.0.27_linux-x64_bin.deb
```

Dan tunggu sampai proses instalasi selesai.

Cek versi

<figure><img src="../.gitbook/assets/image (468).png" alt=""><figcaption></figcaption></figure>

```
java -version
```



5\.       Setelah itu kita bisa install rpm Langkah ini opsional jika tidak pun tidak akan terjadi kendala apapun akan tetapi pada saat menjalankan file _shell script_ nanti akan ada warning.

```
sudo apt install rpm -y
```



6\.       Kemudian buka port 7001 pada ufw dengan menjalankan perintah dibawah ini.

```
sudo ufw allow 7001/tcp
```

&#x20;

7\.       Setelah semua dipersiapkan dan kita mulai running file _shell script_ yang sudah kita siapkan sebelumnya, dengan menjalankan perintah dibawah ini.

<figure><img src="../.gitbook/assets/image (474).png" alt=""><figcaption></figcaption></figure>

```
/home/cikal/install_konfigurasi_weblogic.sh
```

Tunggu sampai proses instalasi dan konfigurasi selesai.

<figure><img src="../.gitbook/assets/image (425).png" alt=""><figcaption></figcaption></figure>

Sampai disini tahapan menjalankan silent mode berhasil, pada gambar diatas di terminal terdapat penjelasan untuk menjalankan maupun menghentikan WebLogic.

```
cd /opt/oracle/weblogic/user_projects/domains/base_domain/bin && ./startWebLogic.sh
```

```
cd /opt/oracle/weblogic/user_projects/domains/base_domain/bin && ./stopWebLogic.sh

```
