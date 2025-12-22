# Tomcat di Windows Server

Berikut cara installasi tomcat di windows server.

1. Pertama kita siapkan file installasi jdk dan apache tomcat. Disini saya menggunakan jdk versi 11 dan apache tomcat versi 10.1.49. Selanjutnya kita install file jdk nya lalu klik Next.

<figure><img src="../../.gitbook/assets/image (770).png" alt=""><figcaption></figcaption></figure>

2. Lalu klik Next.

<figure><img src="../../.gitbook/assets/image (771).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (772).png" alt=""><figcaption></figcaption></figure>

3. Jika sudah klik close.

<figure><img src="../../.gitbook/assets/image (773).png" alt=""><figcaption></figcaption></figure>

4. Selanjutnya kita akan menginstall tomcat nya. Klik next.

<figure><img src="../../.gitbook/assets/image (774).png" alt=""><figcaption></figcaption></figure>

5. Selanjutnya klik I Agree.

<figure><img src="../../.gitbook/assets/image (775).png" alt=""><figcaption></figcaption></figure>

6. Centang Manager dan Host Manager lalu klik Next.

<figure><img src="../../.gitbook/assets/image (776).png" alt=""><figcaption></figcaption></figure>

7. Kemudian isi Server Shutdown Port jika di isi “-1” maka kita tidak akan bisa shutdown service tomcat secara remote, sedangkan jika di isi “8005” maka kita bisa shutdown service tomcat secara remote karena port 8005 merupakan tcp protocol yang penting pada konfigurasi firewall. Lalu isi HTTP/1.1 Connector Port, Windows Service Name, centang Create shortcuts for all users, dan isi username login dan password serta roles nya. Lalu klik Next.

<figure><img src="../../.gitbook/assets/image (778).png" alt=""><figcaption></figcaption></figure>

8. Selanjutnya klik Next.

<figure><img src="../../.gitbook/assets/image (779).png" alt=""><figcaption></figcaption></figure>

9. Klik Install.

<figure><img src="../../.gitbook/assets/image (780).png" alt=""><figcaption></figcaption></figure>

10. Selanjutnya hapus centang Show Readme lalu klik Finish.

<figure><img src="../../.gitbook/assets/image (781).png" alt=""><figcaption></figcaption></figure>

11. Selanjutnya buka cmd > run as administrator. Lalu masukkan perintah berikut :

```
set JAVA_HOME=C:\Program Files\Java\jdk-11.0.29
```

```
set path=C:\Program Files\Java\jdk-17.0.1\bin;%path%
```

<figure><img src="../../.gitbook/assets/image (782).png" alt=""><figcaption></figcaption></figure>

12. Selanjutnya buka file context.xml di folder C:/Program Files/Apache Software Foundation/Tomcat 10.0/webapps/manager/META-INF/ dan C:/Program Files/Apache Software Foundation/Tomcat 10.0/webapps/host-manager/META-INF. Selanjutnya ubah script berikut.

<figure><img src="../../.gitbook/assets/image (783).png" alt=""><figcaption></figcaption></figure>

#### Ubah jadi untuk mengizinkan semua IP:

```xml
<Valve className="org.apache.catalina.valves.RemoteAddrValve"
       allow=".*"/>
```

Atau hanya IP kamu:

```xml
<Valve className="org.apache.catalina.valves.RemoteAddrValve"
       allow="127\.0\.0\.1|192\.168\.56\.1|192\.168\.56\.16"/>
```

13. Selanjutnya buka file tomcat-users.xml di foder C:/Program Files/Apache Software Foundation/Tomcat 10.0/conf/tomcat-users.xml Selanjutnya cari kata sandi yang sudah di buat dan ubah roles nya seperti berikut.

<figure><img src="../../.gitbook/assets/image (784).png" alt=""><figcaption></figcaption></figure>

14. Selanjutnya buka file server.xml di folder C:/ Program Files/Apache Software Foundation/Tomcat 10.0/conf/server Lalu cari “shutdown” dan ubah port nya menjadi “8005”

<figure><img src="../../.gitbook/assets/image (786).png" alt=""><figcaption></figcaption></figure>

15. Selanjutnya buka cmd > run as administrator. Masukkan perintah berikut “cd C:\Program Files\Apache Software Foundation\Tomcat 10.0\bin”. Lalu restart service tomcat dengan memasukkan perintah “shutdown.bat” dan “startup.bat”.

```
cd C:\Program Files\Apache Software Foundation\Tomcat 10.1\bin
```

```
shutdown.bat
```

```
startup.bat
```

<figure><img src="../../.gitbook/assets/image (787).png" alt=""><figcaption></figcaption></figure>

Jika ada kendala pada path java, ikuti langkah dibawah ini

## **Mengatur JAVA\_HOME di Windows (Super Detail)**

### **🟦 1. Buka pengaturan Environment Variables**

1. Klik tombol **Start** (ikon Windows).
2. Ketik:\
   **environment variables**
3. Pilih:\
   **Edit the system environment variables**\
   (biasanya muncul paling atas)

Akan terbuka jendela bernama **System Properties**.

***

### **🟦 2. Buka menu Environment Variables**

Di jendela **System Properties**, lakukan:

1. Klik tombol **Environment Variables…**\
   (letaknya di kanan bawah)

Akan muncul jendela baru bernama **Environment Variables**.

***

### **🟦 3. Membuat variabel baru JAVA\_HOME**

Pada jendela **Environment Variables**, fokus ke bagian bawah:

📌 **Bagian bawah = System variables** (Wajib di sini ya, jangan di bagian User variables)

1. Klik tombol **New…**

Muncul jendela kecil untuk membuat variable baru.

2. Isi seperti berikut:
   *   **Variable name:**

       ```
       JAVA_HOME
       ```
   *   **Variable value:**\
       Isi dengan _lokasi folder Java_ kamu, contoh:

       ```
       C:\Program Files\Java\jdk-11
       ```

       atau lokasi Java 11 kamu yang sebenarnya.
3. Klik **OK**

***

### **🟦 4. Tambahkan JAVA\_HOME ke PATH**

Masih di jendela **Environment Variables**:

1. Pada **System variables**, cari baris bernama **Path**
2. Pilih **Path**, lalu klik **Edit…**
3. Muncul jendela daftar path
4. Klik **New**
5.  Ketik (atau paste):

    ```
    %JAVA_HOME%\bin
    ```
6. Tekan **Enter**
7. Klik **OK**

***

### **🟦 5. Simpan semua**

Kamu sekarang punya 3 jendela terbuka:

* Edit Path
* Environment Variables
* System Properties

Klik **OK** di setiap jendela:

1. Klik **OK** (Edit Path)
2. Klik **OK** (Environment Variables)
3. Klik **OK** (System Properties)

***

### **🟦 6. Tes apakah berhasil**

Tutup CMD sebelumnya dan buka CMD baru.

Jalankan:

```
echo %JAVA_HOME%
```

Jika berhasil, harus tampil path seperti:

```
C:\Program Files\Java\jdk-11
```

Lalu jalankan:

```
java -version
```

Terakhir, test Tomcat:

```
shutdown.bat
```

Sekarang seharusnya **tidak ada error JAVA\_HOME**.

## **Jadikan Tomcat sebagai Windows Service (Paling Direkomendasikan)**

Jika Anda menggunakan **Apache Tomcat versi Windows Installer**, biasanya sudah termasuk _Service Installer_. Cek langkah berikut:

#### **1. Buka Tomcat Monitor**

*   Tekan **Win + R**, ketik:

    ```
    services.msc
    ```
* Cari layanan bernama:
  * **Apache Tomcat 9.0** (atau versi Anda)

#### **2. Set Startup Type ke Automatic**

* Klik layanan tersebut → **Properties**
*   Pada **Startup type**, pilih:

    ```
    Automatic
    ```
* Klik **Start** (untuk menjalankan sekarang)
* Klik **OK**

Dengan ini, Tomcat akan otomatis hidup setiap kali Windows Server (atau VM) booting.
