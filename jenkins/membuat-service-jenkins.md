# Membuat Service Jenkins

## ✅ **1. Buat folder untuk Jenkins**

Misalnya kamu simpan di `/opt/jenkins`:

```bash
sudo mkdir -p /opt/jenkins
sudo cp /root/jenkins.war /opt/jenkins/
sudo chown -R root:root /opt/jenkins
```

> Kalau jenkins.war kamu di lokasi lain, sesuaikan path-nya.

***

## ✅ **2. Buat file systemd service**

Buat file service baru:

```bash
sudo nano /etc/systemd/system/jenkins.service
```

Isi dengan:

```
[Unit]
Description=Jenkins Server (jenkins.war)
After=network.target

[Service]
User=root
WorkingDirectory=/opt/jenkins
ExecStart=/usr/bin/java -jar /opt/jenkins/jenkins.war --httpPort=9090
Restart=always

[Install]
WantedBy=multi-user.target
```

📌 **Keterangan:**

* `--httpPort=9090` → pakai port yang kamu inginkan
* `Restart=always` → auto restart jika crash
* `After=network.target` → tunggu jaringan aktif dulu

Kalau kamu ingin Jenkins **tidak dijalankan oleh root**, kamu bisa pakai user lain (misal `jenkins`), tinggal ubah:

```
User=jenkins
```

dan pastikan jenkins.war bisa dibaca user itu.

***

## ✅ **3. Reload systemd**

```bash
sudo systemctl daemon-reload
```

***

## ✅ **4. Enable service agar aktif otomatis setelah boot**

```bash
sudo systemctl enable jenkins
```

***

## ✅ **5. Start Jenkins**

```bash
sudo systemctl start jenkins
```

***

## ✅ **6. Cek status**

```bash
sudo systemctl status jenkins
```

Harus muncul:

```
Active: active (running)
```

***

## 🛠️ Jika ingin melihat log Jenkins:

```bash
journalctl -u jenkins -f
```

***

## 🟢 Selesai! Jenkins sekarang otomatis berjalan tiap kali VM hidup.

Kalau mau pastikan benar-benar auto-start, reboot VM:

```bash
sudo reboot
```

Lalu setelah masuk lagi:

```bash
systemctl status jenkins
```
